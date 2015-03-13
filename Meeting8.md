Meeting minutes of Meeting#8.

# Escrow prototype #
1. Implemented escrow in Java.

Main thread creates worker threads and also applies order confirming operation to database directly.

Each worker thread contains 1 application request, including several application transactions(insert orders, get transaction info, etc).

The insert order transaction will fire the trigger which creates another autonomous transaction to do real work.


2. Problems encountered during first tests.

  * Over limited database connection when mpl >= 90.
    * Will not be a problem in modified model

  * Failed to create trigger properly using script embedded in Java.
    * Need to recreate trigger after first data load.

# Meeting Notes #
1. Modify the prototype to make the test follows "warmup", "measurement", and "cooldown" phases. In "warmup" phases, let database start to work, the time measures during "measurement" phase, and at last reduce workload and finish all work in "cooldown" phase.

2. Modify worker thread, let each worker thread keep accepting application requests, so that we have some worker threads(~20), but can produce as many requests as we want.

3. In later work, encapsulate all database operations in PL/SQL procedures to separate application level code and database access code.

4. Add comparative test executing without autonomous transaction.


# Action Points #
1. Finish above 4 points.