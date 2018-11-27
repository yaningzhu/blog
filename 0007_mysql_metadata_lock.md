# Finding MySQL Metadata Lock

Have you ever been hit by this dreaded error?

```
1661    Waiting for table metadata lock    ALTER TABLE blahblahblah...
```

Here we were trying to perform an ALTER TABLE. The table only has 1000 rows; it should've been a very quick operation.

However we got contacted by our developer that the migration was stuck. It's apparently been stuck in that state for over 1600 seconds.

## Quick Overview on Metadata Lock

It is exactly how it sounds. In order to change the metadata, MySQL will give the metadata lock to a process. From a Operations perspective, most of time we don't need to know under what conditions MySQL will require metadata lock. We just need to know that, okay something is holding the metadata lock and it's blocking a lot of stuff, we need to resolve it ASAP.

Unfortunately, metadata lock is not a "normal" lock by MySQL. So if you search for queries on the internet that will show you current locks, none of them will work for metadata lock!

Fun, isn't it?

## How to Actually Find Metadata Lock

In MySQL 5.7, there are settings in the performance_schema that you can turn on in order to see metadata locks. See [Percona's blog here](https://www.percona.com/blog/2016/12/28/quickly-troubleshooting-metadata-locks-mysql-5-7/). However this requires you to turn it on beforehand.

So what if, like the majority of us, don't have 5.7 or don't have it turned on?

Sometimes you can tell, because you can have a long running transaction on the same table, then you can be pretty damn sure that it is blocking everything else. The worst kinds are when everything else appears to be idle, but your transaction still can't go through due to something else holding metadata lock.

Short of killing every existing connection on the server, there is actually a way to (mostly) pinpoint the exact connection that is holding the lock. We go to our trusty INNODB STATUS!

### INNODB STATUS

1. Run `SHOW ENGINE INNODB STATUS`. It will give you an inside look into InnoDB.

2. The section that we're looking for is `TRANSACTIONS`. It lists out all current open/pending transactions, plus any idle connections.

    Each connection will look something like this:
    ```
    ---TRANSACTION xxxxxxxx, <status>
    MySQL thread id xxxxx, OS thread handle xxxxxxxxxxxxxxxx, <other stuff>
    <query here if it is attempting to run a query>
    ```

3. We want to look for any transaction that has a status of `ACTIVE`. Only active connections can hold locks. In my cases, I often see things like this:

    ```
    ---TRANSACTION 72997019, ACTIVE 6047 sec
    MySQL thread id 38407, OS thread handle 0x2b72dc185700, query id 127165531 192.168.10.241 user1
    ```

4. This particular connection is literally doing nothing. But the transaction is open for 6047 seconds. In this case, the problem is resolved by killing the connection. Pass the "thread id" into the kill command.

5. Typically this happens when a client doesn't close the transaction properly. I've seen it from human errors; I've also seen it from improperly coded client programs.
