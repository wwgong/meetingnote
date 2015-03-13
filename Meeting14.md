#Meeting minutes of Meeting#14.



# Meeting Notes #
### 1. Point out design flaws and code bugs ###
  * struct to store all escrow update request
    * change to a linked list
    * modify "escrow\_req\_struct", delete "escrow\_req\_list" to use standard list definition in MySQL added in "trx\_struct".
```
typedef struct escrow_req_struct {
	ibool is_pending;
	upd_node_t`*` upd_node;
        UT_LIST_NODE_T(escrow_req_struct) escrow_reqs;
}escrow_req;

...

ibool is_escorw;
ibool redo_upd;
UT_LIST_BASE_NODE_T(escrow_req) escrow_req_list;


```

  * Lock mechanism in MySQL
    * Shouldn't place a S\_LOCK on table

  * When to initialize the shared data (val/inf/sup)?
    * No need to initialize them when server starting
    * Still be server global variables
    * Initialize them at the first time accessing escrow field data

### 2. Rewrite previous design in pseudo code ###
  * 2 versions
    * one at high level pseudo code
    * the other one with more details



# Action Points #


### 1. When to create struct prebuilt() ###
  * Command: use DBNAME
  * create one object for each table in INFOMATION\_SCHEMA.table\_names
  * Lifetime: all time while connecting to this storage/database
  * Function stacks:
```
row_create_prebuilt()
ha_innobase::open()
handler::ha_open()
open_table_from_share()
open_unireg_entry()
open_table()
open_tables()
open_normal_and_derived_tables()
mysqld_list_fields()
dispath_command()
```

### 2. How to commit a transaction ###
  * autocommit = ON
```
trx_commit_for_mysql()
innobase_commit_low()
innobase_commit()
ha_commit_one_phase()
ha_commit_trans()
ha_autocommit_or_rollback()
dispath_command()
do_command()
handle_one_connection()
```
  * autocommit = OFF
```
...
ha_commit_trans()
end_trans()
mysql_execute_command()
mysql_parse()
dispath_command()
do_command()
handle_one_connection()
```

### 3. Which level to add redo update? ###
  * Escrow is rarely applied to autocommit ON case since if use commit trx immediately after update, we don't need to delay update; moreover, both cases shared the same routine to commit transaction. Thus, we could try to add it at innobase\_commit\_low().
  * No need to restore struct prebuilt because it's accessible one innobase handler level in ha\_innobase class


### 4. Lock types ###
  * From [InnoDB Lock Types](http://dev.mysql.com/doc/refman/5.1/en/innodb-record-level-locks.html)
  * 3 types:
    * Record lock: This is a lock on an index record.
    * Gap lock: This is a lock on a gap between index records, or a lock on the gap before the first or after the last index record.
    * Next-key lock: This is a combination of a record lock on the index record and a gap lock on the gap before the index record.


### 5. Latching Order(from sync0sync.h) ###
  * User transaction locks
  * Dictionary
    * Dictionary mutex
    * Dictionary head
  * Secondary index(if any)
    * Secondary index tree latch
    * Secondary index non-leaf
    * Secondary index leaf -- hold
  * Clustered index
    * Clustered index tree latch -- release immediately after get its leaf latch
    * Clustered index non-leaf
    * Clustered index leaf
  * Transaction (trx\_struct)
    * Transaction system header
    * Transaction undo mutex -- mutex\_t undo\_mutex
    * Rollback segment -- trx\_rseg\_t	rseg
      * Rollback segment mutex
      * Rollback segment header
  * Purge system latch
  * Undo log pages
  * File space management latch
  * File system pages
  * Kernel mutex
  * Search system mutex
  * Buffer pool mutex
  * Log mutex
  * Any other latch
  * Memory pool mutex




### 6. Lock a row in update ###
  * Lock a clustered index node -- necessary
```
lock_rec_lock() with LOCK_X
lock_clust_rec_modify_check_and_lock()
btr_cur_update_in_place()/btr_cur_optimistic_update()
row_upd_clust_rec()
row_upd_clust_step()
row_upd()
row_upd_step()
```

  * Lock a secondary index node -- if the update changes the secondary index, optional
```
lock_rec_lock() with LOCK_X
lock_sec_rec_modify_check_and_lock()
btr_cur_del_mark_set_sec_rec()
row_upd_sec_index_entry()
row_upd_sec_step()
row_upd()
row_upd_step()
```



### 7. Pseudo Code ###

  * Shared data to store val/inf/sup
    * Add a new server global variable
```
typedef struct{
        int val;
        int inf;
        int sup;
}escrow_shd_struct;

struct{
      ibool inited = false;
      escrow_shd_struct data;
}escrow_key_data;

escrow_key_data escrow_data;
```
    * Must be guarded by kernel mutex while accessing it
```
mutex_enter(kernel_mutex)
ACCESS ESCROW_DATA;
mutex_exit(kernel_mutex) 
```
    * Initialize it when first time accessing it
```
if(!escrow_data.inited) {
    escrow_data.data.val = QUERY_TABLE_GET_VALUE(table, column, key);
    escrow_data.data.inf = escrow_data.data.sup = escrow_data.data.val;
    escrow_data.inited = TRUE;
}
```

  * List to store data of all escrow-type update queries
    * Create new struct type and add it to transaction struct (trx\_struct)
```
typedef struct escrow_req_struct {
	ibool is_pending = TRUE;
	upd_node_t'*' upd_node;  /* copy from original update node */
        /* think que_thr_t no more needed, because 
         * we can modify commit at innobase handler level */
        UT_LIST_NODE_T(escrow_req_struct) neighbor_reqs;
}escrow_req;

...

ibool is_escorw = FALSE;
ibool redo_upd = FALSE;
UT_LIST_BASE_NODE_T(escrow_req) escrow_req_list;
```
    * Do real update at the end of trx commit
```
if(trx.is_escrow)
{
   trx.redo_upd = TRUE;
   escrow_req'*' cur = escrow_req_list.first;

   while(cur != escrow_req_list.last) {
       ESCROW_ROW_UPDATE(cur.upd_node, prebuild);
       cur.is_pending = FALSE;
       cur = cur.next;
   }
```

  * Routine of ESCROW\_ROW\_UPDATE()
```
/* If it's escrow-type update, 1st time, place a S lock on all row
 * original query may access, then store needed info in trx_struct;
 * 2nd time(end of committing trx), redo update using stored info. 
 * If it's regular update, just do it as regular way. */

ESCROW_ROW_UPDATE(upd_node_t(*) upd_node, que_thr_t(*) prebuilt ) {

/* escrow-type update at 1st time */
if(!trx.redo_upd && UPDATE_ON_ESCROW_TABLE_AND_FIELD)
{   
    trx.is_escrow = TRUE;
    UT_LIST_ADD_FIRST(trx.escrow_req_list, upd_node);
}



}

if(!trx.is_escrow || trx.redo_upd)
    DO_REGULAR_UPDATE;
else {
    err = lock_table(table, LOCK_IS);

    if(err != DB_SUCCESS)
        goto error_handling;

    btr_cur(*) cur = get_btr_cur(node->pcur);

    while(cur.next != NULL) {

        lock_rec_lock(cur, LOCK_S);

	if (err != DB_SUCCESS)
	      goto function_exit;

	cur = cur.next;
    }
       

    while(node->index != NULL) {

        lock_rec_lock(node->index, LOCK_S);

	if (err != DB_SUCCESS)
	      goto function_exit;

	node->index = dict_table_get_next_index(node->index);
    }

    return;
}
```

### Other ###
  * updated fields -- thd->lex->select\_lex->item\_list
  * update vals -- thd->lex->value\_list