# How to replicate one schema from your MySQL database to Amazon RDS/Aurora

Replicating your own database to Amazon can be a daunting task. Thankfully, Amazon has extensive documentation about this matter. However, let's say your database has multiple schemas, and you only want one schema replicated. That becomes a little tricky.

Sure you can `mysqldump` the schema to RDS/Aurora. But then you can't continuously replicate it from your current production, because Amazon does not support options such as `replicate-do-db` or `replicate-ignore-db`.

What we want to do is set up an intermediary server, so that such replication can take place.

Essentially, we need a replication setup that goes like this: A -> B -> C, where:

- A: production
- B: intermediary
- C: RDS/Aurora

## Set up intermediary server

Boot up a server of your liking and install MySQL along with all the other necessary software. Next we want to take a copy of your current production and put it onto this intermediary server.

How we do this next part depends on your schemas size. There is no hard cutoff. If you think taking X amount of time to import a mysqldump is reasonable, then we can just mysqldump it. Otherwise we'll need to go with Percona's online backup. Since the process is a little different, we'll just cover mysqldump method in this post. We'll do Percona online backup separately in a future posting.

### mysqldump

1. Dump the desired schema from your current production server:

    ```
    mysqldump --master-data=2 -h <prod server> -u <mysql login> -p <schema> | pigz > schema.sql.gz
    ```

    `--master-data=2` ensures the binlog position is captured into the dump. `pigz` is a multi-threaded version of `gzip` that should be much faster than gzip when you are working with large data sets.

    It will prompt you for the mysql login password and proceed to dump afterwards.

2. Before importing this schema dump, make sure to grab binlog position:

    ```
    gzip -cd schema.sql.gz | head -n 30
    ```

    This command will get the first 30 lines from the .gz archive without fully decompressing it. You should see a line similar to `-- CHANGE MASTER TO MASTER_LOG_FILE='master-bin.043014', MASTER_LOG_POS=40565562;`. That's the binlog position we'll need to continue to replicate from production. WRITE DOWN THIS LOG FILE AND POSITION.

3. Next we can import the dump to our intermediary server through normal means.

    ```
    gzip -d < schema.sql.gz | mysql -h <intermediary server> -u <mysql login> -p <schema>
    ```

    It will prompt you for password. Enter the password and proceed.

    Note that you might want to do this in tmux/screen, because the import process could be rather long.

4. At the same time, you can import the dump to the RDS/Aurora server also. Similar command as the previous step, with a different hostname.

5. Now we patiently wait for the import process to complete. The end result is that you should have an identical copy of the schema on both the intermediary server and the RDS/Aurora server.

## Set up replication between intermediary server and RDS/Aurora

Before we set up replication though, we need to make sure we are only replicating the schema that we are interested in.

### Adjust my.cnf to only replicate one schema

Modify your intermediary server's MySQL configuration file, typically located at `/etc/my.cnf` or `/etc/mysql/my.cnf`.

You'll want to add the options below to your existing cnf file, if the options didn't already exist. Add them under the `[mysqld]` section

```
## my.cnf

[mysqld]
...
...
...

# assign a non-conflicting server_id. server_id is required to identify each server in a replicated setup
server-id = XXXXX

# turn on binary log, this setting should be adjusted based on your own environment
log_bin          = /log/master-bin.log
expire_logs_days = 5
max_binlog_size  = 100M

# binary log format. ROW format is important because it allows replicate-do-db to be used
binlog_format    = ROW
binlog_checksum  = none

# slave update because this intermediary server is server B in a A-B-C chain
log-slave-updates

# replicate only the schema you are interested in
replicate-do-db  = schema                  # this option should work for most setup
#replicate-wild-do-tables = schema.%       # this option is required if you do cross-schema updates. This is the also the option you should use if you update user permissions, since grant statements would fail if user does not exist
```

Restart MySQL service for new setting to take effect. Now the intermediary server will only replicate the schema change that you are interested in. This ensures the binary log only contains these changes, and replication won't break when RDS/Aurora stars replicating off of the intermediary server.

### Set up Replication

Now the intermediary server is ready, we can set RDS/Aurora to replicate off of it.

Log on to your RDS/Aurora instance. At the time of this writing, you have to use (`mysql.rds_set_external_master`)[https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql_rds_set_external_master.html] to do that. Then start replication with (`mysql.rds_start_replication()`)[https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql_rds_start_replication.html]. Consult AWS documentation for details.

Now you have completed the B->C part of the A->B->C chain.

## Set up replication between production and intermediary server

Last step, we need to complete A->B. We can just do this with standard MySQL replication. Log onto the intermediary server and run the standard `CHANGE MASTER` command:

```
CHANGE MASTER TO MASTER_LOG_FILE='master-bin.043014', MASTER_LOG_POS=40565562,
  MASTER_HOST='<production mysql host>',
  MASTER_PORT=<production mysql port>,
  MASTER_USER='<replication user login>',
  MASTER_PASSWORD='<replication user password';
```

MASTER_LOG_FILE and MASTER_LOG_POS are from the steps where you wrote down!

Then just `start slave` to start the replication. Now the A->B->C chain is complete.
