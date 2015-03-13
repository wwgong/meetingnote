#Meeting minutes of Meeting#17.

Attendees: Pat, Betty, Steve, Weiwei

# Meeting Note #
1. Escrow
  * long term
  * all escrow data must be memory resident(no I/O)
  * escrow data not only includes primary key, all update field

2. Infrequent update
  * thin table
  * convert fat table into thin tables
  * eliminate unnecessary data

3. Installation of Plugin.
  * static: Replace all files in "/storage/innobase" with files under "/storage/innodb\_plugin" and follow normal installation instructions from source code. Instructions from MySQLhttp://dev.mysql.com/doc/innodb-plugin/1.0/en/innodb-plugin-installation-source-unix.html
  * dynamic: not recommended if installing from source code

4. Difference between Innodb and Innodb-Plugin
  * Fast index creation: use secondary index when possible instead of clustered index to avoid data copying
  * Data compression: B-tree index and big object
  * New file format: new Barracuda file format controls how much column data is stored in the clustered index, and how much is placed on overflow pages
  * New row format: now row formates contain COMPRESS, DYNAMIC, REDUNDANT.
  * Algorithm changes:
    * Faster locking for improved scalability
    * Using operating system memory allocators
    * Controlling InnoDB insert buffering
    * Controlling adaptive hash indexing
    * Changes regarding thread concurrency
    * Changes in read ahead algorithm
    * Multiple background I/O threads
    * Group Commit
    * Controlling master thread I/O rate
    * Controlling flushing rate of dirty pages
    * Using a portable PAUSE to InnoDB spin loop
    * Control Over Spin Lock Polling
    * Changing defaults of parameters
    * Making Buffer Cache Scan Resistant
    * Improvements to Crash Recovery Performance



# Action Points #
1. Doublewrite buffer.
  * Before writing pages to a data file, InnoDB first writes them to a contiguous area -- the doublewrite buffer. After the write and the flush to the doublewrite buffer have completed, InnoDB write the pages to the data file. If the operating system crashes in the middle of a page write, InnoDB can later find a good copy of the page from the doublewrite buffer during crash recovery.
  * Turn off doublewrite buffer: SET GLOBAL innodb\_doublewrite=0
  * Advantage: protect data from partial page write; some people think saving I/O under certain workload.

2. Where to find the log when restart system after a crash?
  * Breakpoints setting:
```
srv0start.c:836  innobase_start_or_create_for_mysql()
log0log.c:1283  log_write_up_to()
os0file.c:1861  os_file_fsync()
log0recv.c:2491 recv_recovery_from_checkpoint_start()
log0recv.c:2448  recv_init_crash_recovery()
```

  * Produce crash recovery
    * start eclipse
    * run server -- mysqld -- in the debugger
    * set autocommit to false
    * do operation(create table, insert data)
    * commit
    * insider the debugger, before system write data out to file, kill the eclipse and mysqld processes
    * restart eclipse and debug the mysqld again
    * debugger stops at innobase\_start\_or\_create\_for\_mysql()

  * Stack snapshot:
```
innobase_init()
innobase_start_or_create_for_mysql()
open_or_create_data_files()
fil_read_flushed_lsn_and_arch_log_no()
```
```
innobase_init()
innobase_start_or_create_for_mysql()
recv_recv_from_checkpoint_start()
recv_init_crash_system()
trx_sys_doublewrite_init_or_restore_pages()
```

  * Important information
    * When need to do crash recovery?
```
The system are always trying to do crash recovery even after shutdown normally, 
but only when the latest lsn > last committed lsn, it will perform real crash recovery.
```
    * The last flushed(committed) lsn(log sequence number) is stored in max\_flushed\_lsn.
```
In function innobase_start_or_create_for_mysql(), 
it calls open_or_create_data_files() which calls 
fil_read_flushed_lsn_and_arch_log_no() 
opening data files to get min_flushed_lsn and max_flushed_lsn.
```
    * The lastest lsn(might not committed) is stored in recv\_sys->scanned\_lsn when doing crash recovery.
```
This number is obtained from mach_read_from_8(log_sys->checkpoint_buf + LOG_CHECKPOINT_LSN), 
and log_sys->checkpoint_buf is obtained from reading the data file corresponding to its space_id.
```