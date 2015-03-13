Meeting minutes for Meeting#3.

# Design #

### MySQL ###
  * no user defined data type
  * can explicitly lock and unlock tables, but one session must acquire all locks it needs in one single "LOCK TABLES" statement

#### Tables ####
  * T(PK, inf, val, sup)
  * T'(TxID, PK, low, high, timestamp, status)

### Sequences ###
  1. Check whether new request satisfies constraints of T' or not.
  1. Lock T; update 'inf', 'val', and 'sup' on T; release lock; and then immediately S lock T.
  1. Insert into T' with new TxID.
  1. Commit/Abort.
    * Get X lock on T and T'.
    * Update 'inf' and 'sup' on T, and as well as 'status' on T'.
    * Release all locks.


# Meeting Notes #

Discussed the functionality supported by MySQL and decided to switch to Oracle first because MySQL doesn't support AUTONOMOUS TRANSACTION.

We will explore the possibility using different store engines in MySQL.


# Action Points #

How to implement Escrow transaction in Oracle using existing features.