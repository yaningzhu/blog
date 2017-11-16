# Playing with Redshift Permissions

This exploratory exercise comes from a simple question: **How do we create read-only users in Redshift that can read all tables?**

Turns out it's not as simple as a DBA initially thinks. With a database like MySQL, you can do `grant select on *.* to user`. Redshift, unfortunately, you would need to do it individually for each schema.

## Chapter 1: Create User Login, the Logic Way

Before we even create an user, keep in mind that it's bad practice to have everyone sharing the same readonly user. Everyone should have their own login even if they all share the same readonly permissions.

Redshift makes this easy by allowing the concept of `group`. You can grant a set of permissions to a group, and then assign users to that group. Then in the future if you need to change permissions, you can simply change the group permission, and it will apply to every user who belongs to the same group.

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

Then I can just assign it to my group:

```sql
alter group readonly add user yzhu;
```

We can add more users if we need to. But this is the most straightforward way of doing it.

## Chapter 2: Wait, The Previous Steps Actually Doesn't Work

In the previous chapter, we created a `readonly` group, granted some select privileges to it, and assigned user `yzhu` to the group. So now `yzhu` has select on all the tables.

Here comes an interesting catch: **What happens if we create a new table?**

### Verifying the Current State

Save this query somewhere, it's super useful to see what kind of permissions an user has:

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

Wait a minute, doesn't that result look exactly the same as before? I granted select on all tables in the schema, why isn't the second table showing up.

## Chapter 3: Fixing the problem

In the previous chapter, we discovered that any new table we create does not automatically get granted permission.

### Temporary Fix

We can run the grant statement again:

```
grant select on all tables in schema report_yzhu to group readonly;
```

And then run the giant query again to see result:

```
schemaname   objectname  usename  sel  ins  upd  del  ref
report_yzhu  table1      yzhu     t    f    f    f    f
report_yzhu  table2      yzhu     t    f    f    f    f
```

NOW it shows up. Great, which means we have to do this for every new table we add. That's obviously not sustainable.

### Real Fix

Redshift behaves much like Postgres in a way that it offers the option to assign default privileges. You can actually assign default permissions to users, schemas, or whatever, so that they automatically take effect whenever new tables get added.


The easiest way to make this work:

```sql
alter default privileges grant select on tables to group readonly;
```

This automatically grants select on all future tables to the group `readonly`. Notice when we were trying to grant select on tables previously, we had to do it schema by schema. This statement covers all schemas. Neat, eh?

### Extra credit

So now let's see what we have to do when we create a new schema.

```sql
create schema report_test;
grant usage on schema report_test to group readonly;
```

And that should be it! Nothing else needs to be done thanks to the all-covering default privilege in the previous section.

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

## Chapter 4: No longer readonly

Since we're a very open company, we don't mind everyone having read access to all tables, especially to all tables in a data warehouse where the data isn't sensitive.

However, we definitely don't want people to have insert/delete/update privileges to all tables. Let's create a specific group to do that.

### Setting up

```sql
create group readwrite;
