# Restore Percona Backup to Amazon Aurora

Many people these days have huge MySQL databases where mysqldump is no longer viable as backup/restore methodology. At our organization, we uses [Percona XtraBackup](https://www.percona.com/doc/percona-xtrabackup/LATEST/index.html).

The main advantage of Percona backup is that, it copies the actual data files along with transaction logs. When it's time to restore, it will use the transaction logs to perform crash recovery on the data files. This results in a (much) faster restore time than mysqldump, which is just a bunch of .sql statements that takes forever to be replayed.

## Why Aurora?

Our setup worked for a long time. We have a relatively simple 1 master + N slaves setup. However, our HA setup is flimsy. Our database usage prevented us from having something straightforward such as Percona Cluster or Galera Cluster. We had to partially make our own HA solution for the database, which caused us numerous headaches in the past. A wise person will tell you adding a second server is more like 10X the effort rather than 2X; and he would be absolutely correct.

We decided as an organization to move our database to Amazon Aurora. It is fully managed. No more scrambling to read documentation from six months ago for a bad failover. No more manually restoring from backups when trying to add another slave.

## Methodology

For a successful transition, we must take a production database backup and restore it to an Aurora cluster. MySQLDump is simply outclassed here; no one sane would try to mysqldump a multi-terabyte database, and then somehow restore it within a reasonable amount of time. We are already taking a Percona backup every night. It would be great if we can somehow use it...

...and it turns out we can!

Amazon provides [some decent instructions here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/AuroraMySQL.Migrating.ExtMySQL.html#AuroraMySQL.Migrating.ExtMySQL.S3). The gist of it is to take a Percona Backup of your existing database. Dump it out to S3. And then tell Aurora to restore from S3.

While all the steps make sense, I will go over some specific "gotcha"s.

## Prep the Database

There are a couple things that the Aurora database system simply does not support.

### Remove Compression

Aurora does not support compressed tables. Because Percona backup is literally a file backup, a compressed table will also have a compressed file on disk, and therefore cannot be restored to Aurora. So you first make sure to undo any compression on your tables.

You can find all compressed tables with this query:

```sql
SELECT *
FROM information_schema.tables
WHERE row_format = 'Compressed'
  AND table_schema NOT IN ('information_schema', 'mysql', 'performance_schema')
```

For any table that is compressed, use the following to un-compress it:

```sql
ALTER TABLE <table_name> row_format=DEFAULT
```

### Convert MyISAM Tables

Aurora also does not support MyISAM tables. You can issue `CREATE TABLE ... format=MyISAM` to Aurora, but the table will get created as InnoDB anyway. Again, because Percona XtraBackup will take your MyISAM table as is, it cannot be restored to Aurora.

Find all your MyISAM tables with the following query:

```sql
SELECT *
FROM information_schema.tables
WHERE engine = 'MyISAM'
  AND table_schema NOT IN ('information_schema', 'mysql', 'performance_schema')
```

If possible, you can convert them to InnoDB:

```sql
ALTER TABLE <table_name> engine=InnoDB;
```

Because MyISAM and InnoDB are inheritly different. You'll want to consult [MySQL's official documentation](https://dev.mysql.com/doc/refman/8.0/en/converting-tables-to-innodb.html) when converting.

### Clean Up Leftover Permissions

#### Why?

Something interesting with MySQL is that `DROP <object>` does not automatically clean up the permissions properly. That goes for `DROP TABLE` or drop anything.

To illustrate my point, consider the following queries:

```sql
create schema test_schema;
create table test_schema.table1 (
  id integer primary key
);

create user yzhu@localhost;
grant select on test_schema.table1 to yzhu@localhost;
```

Nothing special here. I just created a table, an user, and granted SELECT to that user. We can verify that the grant is there by selecting from the system view:

```sql
select * from mysql.tables_priv;
```

So far so good, right? Now something strange will happen. Let's drop that table:

```sql
drop table test_schema.table1;

select * from mysql.tables_priv;
```

And we select from the tables_priv system view again. What do we see here? That's right, the privilege is still there, even if the original table is gone.

Throughout my 20+ hours of trying to restore random things to Aurora, I discover that Aurora REALLY hates leftover permissions like these. Aurora maintains its own `mysql` system schema, so it tries to recreate it by replaying the contents in your Percona backup during restore. And if your backup contains a `mysql` schema with bad entries, have fun having the restore process exits abruptly with an error that no one knows how to troubleshoot.

#### Find All Leftover Permissions

So we have to look through at the schema/table/column level.

* Schema level:

    ```sql
    select concat('revoke all privileges on `',a.db,'`.* from `',a.user,'`@`',a.host,'`;')
    from
      mysql.db a
    where not exists (select 1 from information_schema.schemata b
                      where a.db = b.schema_name)
    ```

    The query above gives you the statements you need to run for revoking.

* Table level:

    ```sql
    select concat('revoke all privileges on `',a.db,'`.`',a.table_name,'` from `',a.user,'`@`',a.host,'`;')
    from
      mysql.tables_priv a
      left join
      (
        select distinct
          table_schema, table_name
        from
          information_schema.tables
      ) b
      on a.db = b.table_schema and a.table_name = b.table_name
    where
      b.table_schema is null;
    ```

    Run the statements to revoke.

* Column level:

    ```sql
    select concat('revoke all privileges on `',a.db,'`.`',a.table_name,'` from `',a.user,'`@`',a.host,'`;')
    from
      mysql.columns_priv a
      left join
      (
        select distinct
          table_schema, table_name, column_name
        from
          information_schema.columns
      ) b
      on a.db = b.table_schema and a.table_name = b.table_name and a.column_name = b.column_name
    where
      b.table_schema is null;
    ```

    Run the statements to revoke.

## Take Backup and Move Backup to S3

The Amazon documentation linked earlier gives you pretty good guidance on this process. I just want to highlight some syntaxes that will help you not only in this endeavor, but also for future times when you have to work with MySQL.

* When doing Percona backups, the `--stream=tar` option is super helpful for doing large backups. You can even pipe that into an compress process such as gzip to make the backup smaller. Include `--slave-info` if you are taking the backup from a slave.

    ```
    innobackupex --user=<myuser> --password=<redacted> \
    --stream=tar \
    --no-timestamp \
    --slave-info \
    /backup/dir 2>/backup/dir/logfile.log | pigz > /backup/dir/backup.tar.gz"
    ```

    We're piping the whole thing into `pigz`, which is gzip parallelized.

* Upload the backup to S3. Note you can use a multi-part splitter for faster uploads. But note that S3 only allows each file to be split to a maximum of 10,000 parts for multi-upload. So don't make your chunk too small. For reference, the minimum chunk size for a 1TB file is 100MB.

## Restore Aurora from S3
