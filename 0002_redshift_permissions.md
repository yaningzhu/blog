# Playing with Redshift Permissions

This exploratory exercise comes from a simple question: **How do we create read-only users in Redshift that can read all tables?**

Turns out it's not as simple as a DBA initially thinks. With a database like MySQL, you can do `grant select on *.* to user`. Redshift, unfortunately, you would need to do it individually for each schema.

## Chapter 1: Create User Login, the Logic Way

Before we even create an user, keep in mind that it's bad practice to have everyone sharing the same readonly user. Everyone should have their own login even if they all share the same readonly permissions.

Redshift makes this easy by allowing the concept of `group`. You can grant a set of permissions to a group, and then assign users to that group. In the future if you need to change permissions, you can simply change the group permissions, and it will apply to every user who belongs to that same group.

### Create a Read-only Group

```sql
create group readonly;
```

Easy right? Now we have an empty group that we can do stuff with.

Like I mentioned before, Redshift does not allow you to grant select to everything; you must do it one schema at a time. So let's say we have a couple schemas that are named:

```
android
ios
report_yzhu
```

I want to make sure that my `readonly` group has access to read from all of these schemas. We would need to do the following:

```sql
grant usage on schema android to group readonly;
grant select on all tables in schema android to group readonly;

grant usage on schema ios to group readonly;
grant select on all tables in schema ios to group readonly;

grant usage on schema report_yzhu to group readonly;
grant select on all tables in schema report_yzhu to group readonly;
```

Now my `readonly` group has read access to all tables in my schemas.

### Assign User to Group

Last step is to simply assign some users to the group, so they all get the same permissions.

Let me first create my user:

```sql
create user yzhu password 'Mysuperaw3somepAssword';
```

Then I can assign it to the `readonly` group:

```sql
alter group readonly add user yzhu;
```

We can add more users if we need to.

## Chapter 2: Wait, The Previous Steps Actually Doesn't Work

In the previous chapter, we created a `readonly` group, granted some select privileges to it, and assigned user `yzhu` to the group. So now `yzhu` has select on all the tables.

Here comes an interesting catch: **What happens if we create a new table?**

### Verifying the Current State

Save the query below, it's super useful to see what kind of permissions an user has (credit to user `drtf` from this [Stack OverFlow link](https://stackoverflow.com/questions/18741334/how-do-i-view-grants-on-redshift))

```sql
SELECT *
FROM
    (
    SELECT
        schemaname
        ,objectname
        ,usename
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'select') AS sel
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'insert') AS ins
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'update') AS upd
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'delete') AS del
        ,HAS_TABLE_PRIVILEGE(usrs.usename, fullobj, 'references') AS ref
    FROM
        (
        SELECT schemaname, 't' AS obj_type, tablename AS objectname, schemaname + '.' + tablename AS fullobj FROM pg_tables
        WHERE schemaname not in ('pg_internal')
        UNION
        SELECT schemaname, 'v' AS obj_type, viewname AS objectname, schemaname + '.' + viewname AS fullobj FROM pg_views
        WHERE schemaname not in ('pg_internal')
        ) AS objs
        ,(SELECT * FROM pg_user) AS usrs
    ORDER BY fullobj
    )
WHERE (sel = true or ins = true or upd = true or del = true or ref = true)
and schemaname='<schema name>'  -- replace schema name here
and usename = '<user name>'     -- replace user name here
;
```

For my example, I had a table in my schema `report_yzhu`. I can just check the privileges for the user `yzhu` on my schema:

```sql
<rest of the query>
...
and schemaname='report_yzhu'
and usename = 'yzhu'
```

I get a result like this:

```
schemaname   objectname  usename  sel  ins  upd  del  ref
report_yzhu  table1      yzhu     t    f    f    f    f
```

So I got select privilege on a table called `table1`. Okay, no problem there.

### Adding a New Table

Now we add a new table to my schema and see what happens:

```sql
create table report_yzhu.table2 (
  id integer primary key,
  name varchar(100)
);
```

We have two tables now, `table1` and `table2`. We can run the giant query again:

```
schemaname   objectname  usename  sel  ins  upd  del  ref
report_yzhu  table1      yzhu     t    f    f    f    f
```

Wait a minute, doesn't that result look exactly the same as before? I granted select on all tables in the schema, why isn't the second table showing up?!

## Chapter 3: Fixing the problem

In the previous chapter, we discovered that any new table we create does not automatically get granted permission.

### Temporary Fix

We can run the grant statement again:

```sql
grant select on all tables in schema report_yzhu to group readonly;
```

And then run the giant query again to see results:

```
schemaname   objectname  usename  sel  ins  upd  del  ref
report_yzhu  table1      yzhu     t    f    f    f    f
report_yzhu  table2      yzhu     t    f    f    f    f
```

Now it shows up. Great, which means we have to do this for every new table we add. That's obviously not sustainable.

### Real Fix

Redshift behaves much like Postgres in a way that it offers the option to assign default privileges. You can actually assign default permissions to users, schemas, or whatever, so that they automatically take effect whenever new tables get added.


The easiest way to make this work:

```sql
alter default privileges grant select on tables to group readonly;
```

This automatically grants select on all future tables to the group `readonly` for the user running this statement. If I run this statement on my admin user, then any table the admin user creates will be readable by the group readonly.

### Consideration

But not everyone will be creating table on the admin user. Redshift has a very strong philosophy of user ownership. Basically, you create the table, you own it. It can be up the table-creator themselves to grant permissions to everyone.

Not everyone is familiar enough with permissions to do this though. So as the database admin, everytime you create an user, you can run the default privilege for that user:

```
create user "app_user";

alter default privileges for user app_user grant select on tables to group readonly;
```

The statement above will make sure any new tables that the `app_user` creates will be available to the `readonly` group.

### Extra Credit

So now let's see what we have to do when we create a new schema.

```sql
create schema report_test;
grant usage on schema report_test to group readonly;
```

And that should be it! Nothing else needs to be done.

We can check this by creating a table first:

```sql
create table report_test.table3 (
  id integer primary key,
  name varchar(100)
);
```

Then we modify the giant query that we had that shows all privileges:

```sql
...
and schemaname in ('report_yzhu', 'report_test')
and usename = 'yzhu'
```

Now run it to get results:

```
schemaname   objectname  usename  sel  ins  upd  del  ref
report_yzhu  table1      yzhu     t    f    f    f    f
report_yzhu  table2      yzhu     t    f    f    f    f
report_test  table3      yzhu     t    f    f    f    f
```

### Extra Extra Credit

To sanity-check our group privileges, we can remove my user `yzhu` from the group:

```sql
alter group readonly drop user yzhu;
```

Now you can run the giant query to check permissions again, you'll see that my user `yzhu` has no read privilege to anything in `report_yzhu` or `report_test`.

## Chapter 4: No longer readonly

Since we're a very open company, we don't mind everyone having read access to all tables in a data warehouse, where the data isn't sensitive.

However, we definitely don't want people to have insert/delete/update privileges to all tables in all schemas. Let's cover a couple different scenarios.

### Setup

Following best practice, if it's not a personal permission to a specific user account, let's first create a group:

```sql
create group readwrite;

grant usage on schema android to group readwrite;
grant select on all tables in schema android to group readwrite;

grant usage on schema ios to group readwrite;
grant select on all tables in schema ios to group readwrite;

grant usage on schema report_yzhu to group readwrite;
grant select on all tables in schema report_yzhu to group readwrite;

grant usage on schema report_test to group readwrite;
grant select on all tables in schema report_test to group readwrite;

alter default privileges grant select on tables to group readwrite;
```

The above set of queries will allow the group `readwrite` to read everything, just like how we did the privileges for the group `readonly`.

Remember how we previously remove my user from the `readonly` group in the Extra Extra Credit section? Now we sanity-check again by adding my user to the `readwrite` group:

```sql
alter group readwrite add user yzhu;
```

If we can run the giant permissions query right now, we should get the same results as if our group were readonly. Makes sense, since we haven't assigned any write privileges yet.

### Allow Write on Certain Schemas

Time to add we can add some write privileges. Let's say the schemas `android` and `ios` are sacred and cannot be touched. So we can only write to `report_yzhu` and `report_test`. Let's do that:

```sql
grant all privileges on schema report_yzhu to group readwrite;
grant all privileges on all tables in schema report_yzhu to group readwrite;

grant all privileges on schema report_test to group readwrite;
grant all privileges on all tables in schema report_test to group readwrite;
```

Note the tiered level permissions. Privileges on the table-level are self-explanatory; privileges on the schema-level allows creating the tables in the first place, among other things.
