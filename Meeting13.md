#Meeting minutes of Meeting#13.


# Meeting Note #
1. Pseudo code.

  * srv0srv.h
```
// GWW: add to support escrow trx
// store the shared data for one escrow type field

typedef struct escorw_struct{
	int val;
	int inf;
	int sup;
}escrow_struct;

escrow_struct escrow_shd;
```

  * srv0srv.c
```
// GWW: initilize the common data for one escrow type field
// TODO: for now, initialize with fixed value, should dynamically get latest val from real table
escrow_shd.val = 1000000;
escrow_shd.inf = 1000000;
escrow_shd.sup = 1000000;
```


  * trx0trx.h -- add new structs
```
// GWW: add to support escrow trx
typedef struct escrow_req {
	ibool is_pending;
	upd_node_t* upd_node;
        que_thr_t* thr;
}escrow_req;

typedef struct escrow_req_list {
	int pos;
	escrow_req* list[];
}escrow_req_list;
```

  * trx0trx.h -- add new fields in trx\_struct
```
ibool is_escorw;
ibool redo_upd;
escrow_req_list* escrow_list;
```

  * trx0trx.c -- trx\_create()
```
// GWW: initialize escrow fields
trx->is_escrow = FALSE;
trx->redo_upd = FALSE;
trx->escrow_list = mem_alloc(sizeof(trx->escrow_list));;

```



  * row0upd.c -- row\_upd\_step()
```
// GWW: 1st iteration, if this trx contains escrow updates, turn on is_escrow, and add this upd_node to redo list
if(!thr->trx->redo_upd && strstr(node->table->name, "escrow") != null)
{
	upd_field_t* upd_field;
	for(int i = 0; i < upd_get_n_fields(node->update); i++){
		upd_field = upd_get_nth_field(node->update, i);
		if(strcmp(dict_table_get_col_name(node->table, upd_field->field_no), "val") == 0) {
			// set type of current trx to escrow
			if (!thr->trx->is_escrow)
				thr->trx->is_escrow = TRUE;
			// add this node to pending escrow request list in order to redo at the end of trx ending
			thr->trx->escrow_list.pos++;
			thr->trx->escrow_list[pos].is_pending = TRUE;
			memcpy(thr->trx->escrow_list[pos]->upd_node, node, sizeof(node));
			break;
		}
	}
}

// GWW: 1st iteration, if it's escrow update, place a Shared lock on the table
// return for now, we will do the real update at the 2nd iteration at the end of trx
if(!thr->trx->redo_upd && thr->trx->is_escrow){
   	node->state == UPD_NODE_SET_S_LOCK;
    	err = lock_table(0, node->table, LOCK_S, thr);
	if (err != DB_SUCCESS) {
		goto error_handling;
	}

	// modify shared data
	mutex_enter(&kernel_mutex);

        // TODO: get val from express, and update val/inf/sup

        if (inf < LOWER_BOUND || sup > UPPER_BOUND) {
            undo val/inf/sup modification;
            goto error_handling;
        }
        

	mutex_exit(&kernel_mutex);


	thr->run_node = parent;
	return(thr);
}

```

  * trx0trx.c -- trx\_commit\_for\_mysql()
```
...
	// GWW: for escrow trx, redo all upd_req
	if(trx->is_escorw) {
		trx->redo_upd = true;
		
		int i = 0;
		for(i = 0; i < trx->escrow_list->pos; i++)
			row_upd_step(trx->escrow_list->list[pos]->upd_node, 
                                     trx->escrow_list->list[pos]->thr);
	}
...

```




2. TODO list
  * how to get update amount from expression
    * problem: can't watch parsed variable exp in eclipse
    * we can get query statement from THD->query\_string->str
  * we have to use que\_thr\_t at the end of trx to redo real update, but whether it's okay to use stored que\_thr\_t
    * que\_thr\_t is the query graph for one query, doesn't hold for whole session
    * since we didn't do real update, does it affect following statement execution?
  * how to get latest val from real table when initializing server
    * can't call corresponding function using query "select val from escrow\_table" because right now server hasn't been initialized can't process query request
    * when and where to get the real val?