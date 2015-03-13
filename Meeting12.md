#Meeting minutes of Meeting#12.


# Meeting Note #
1. Spinlock.
  * There are spinlock and rw-lock supported in InnoDB storage of MySQL. Spinlock represent exclusive access lock to internal in-memory data structures, and rw-lock allows shared access.
  * Implementation of spinlock and rw-lock is in:
    * spinlock: .../storage/innobase/sync0sync.c
    * rw-lock: .../storage/innobase/sync0rw.c
  * How many mutexes exist in InnoDB?
    * There are lots of mutexes used in InnoDB, some of them are used in one module, some are crossing multiple modules, here are some important mutexes
    * srv0srv.h
      * kernel\_mutex: mutex protecting the server, trx structs, query threads, and lock table
      * XX-mutex: mutex to protect server object
    * sync0sync.c
      * mutex\_list: global list of database mutexes created
  * If a mutex exists, how to use the mutex:
    * request the mutex: **mutex\_enter(MUTEX)**
    * release the mutex: **mutex\_exit(MUTEX)**
  * Since we plan to use the biggest and most important mutex "kernel\_mutex", so we don't need to manage our own mutex.



2. Where to store (inf, sup, val) triplet?
  * most global variables of server are stored in srv0srv.h and initialized in srv\_init()
  * to simplify our design, we can created another structure with (inf, val, sup) in srv0srv.h and initialized it in srv\_init()
  * later, we should introduce another method to manage this structure for each escrow type field
    * because we can't keep each one for every escrow type field while server running
    * the better way is to create it when one transaction coming in to update this field and keep it until all update transactions relating to it end.



Useful course website for MIPS:
  * [Mississippi College](http://sandbox.mc.edu/~bennet/cs314/index.html)
  * [UIC](http://logos.cs.uic.edu/366/index.html)