#Meeting minutes of Meeting#15.


# Group Commit in MySQL #

At first, I thought MySQL doesn't support group commit in its storage engines, but it turns out its statically-linked storage engine InnoDB doesn't support it, but additional upgraded
InnoDB Plugin does support it from 1.0.4.([Introduction in MySQL Docs](http://dev.mysql.com/doc/innodb-plugin/1.0/en/innodb-performance-group_commit.html))

_"InnoDB, like any other ACID compliant database engine, is required to flush the redo log of a transaction before it is committed. Historically InnoDB used group commit functionality to group multiple such flush requests together to avoid one flush for each commit. With group commit, InnoDB can issue a single write to the log file to effectuate the commit action for multiple user transactions that commit at about the same time, significantly improving throughput._

_Group commit in InnoDB worked until MySQL 4.x. With the introduction of support for the distributed transactions and Two Phase Commit (2PC) in MySQL 5.0, group commit functionality inside InnoDB was broken._

_Beginning with InnoDB Plugin 1.0.4, the group commit functionality inside InnoDB works with the Two Phase Commit protocol in MySQL. Re-enabling of the group commit functionality fully ensures that the ordering of commit in the MySQL binlog and the InnoDB logfile is the same as it was before. It means it is totally safe to use InnoDB Hot Backup with InnoDB Plugin 1.0.4._

_Group commit is transparent to the user and nothing needs to be done by the user to take advantage of this significant performance improvement."_

I found this when I was trying to get understand how group commit works in PostgreSQL, there is post mentioned comparison between PostgreSQL and InnoDB. And here is a list of new features introduced in InnoDB Plugin. [The new features of InnoDB Plugin](http://dev.mysql.com/doc/innodb-plugin/1.0/en/innodb-plugin-introduction-features.html).
```
    *  Viewing the InnoDB Plugin version number
    *  Fast index creation: add or drop indexes without copying the data
    *  Data compression: shrink tables, to significantly reduce storage and i/o
    *  New row format: fully off-page storage of long BLOB, TEXT, and VARCHAR columns
    *  File format management: protects upward and downward compatibility
    *  INFORMATION_SCHEMA tables: information about compression and locking
    *  Performance and scalability enhancements:
          o Faster locking for improved scalability
          o Using operating system memory allocators
          o Controlling InnoDB insert buffering
          o Controlling adaptive hash indexing
          o Changes regarding thread concurrency
          o Changes in read ahead algorithm
          o Multiple background I/O threads
          o Group Commit
          o Controlling master thread I/O rate
          o Controlling flushing rate of dirty pages
          o Using a portable PAUSE to InnoDB spin loop
          o Control Over Spin Lock Polling
          o Changing defaults of parameters
          o Making Buffer Cache Scan Resistant
          o Improvements to Crash Recovery Performance
    *  Other changes for flexibility, ease of use and reliability:
          o Dynamic control of system configuration parameters
          o TRUNCATE TABLE reclaims space
          o InnoDB “strict mode”
          o Control over statistics estimation
          o Better error handling when dropping indexes
          o More compact output of SHOW ENGINE INNODB MUTEX
          o More Read Ahead Statistics
```


# Useful Sites #
Here are three very useful sites(blogs), their authors all work in Facebook, probably Database team.

1. [High Availability](http://mysqlha.blogspot.com/) Author: Mark Callaghan

2. [domas mituzas](http://dom.as/) Author: Domas Mituzas

3. [MySQL on Facebook](http://www.facebook.com/MySQLatFacebook)

4. [Kristian Nielsen](http://kristiannielsen.livejournal.com/)

5. [Transactions on InnoDB](http://blogs.innodb.com/wp/)

6. [MySQL Musing](http://mysqlmusings.blogspot.com/)


Also, the source code of PostgreSQL can be found here, and read online very clearly. [Source of PG](http://doxygen.postgresql.org/main.html)