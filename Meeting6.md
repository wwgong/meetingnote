Meeting minutes of Meeting#6.

# Test Model on Oracle #

### Scenario ###
Generally, based on our schema design, the test model simulates this scenario:

  * continuous coming in requests(simulate random number requests for each second), each request having a random waiting time until they finally decide check out or discard

  * the ending of current test is when there is nothing left

  * the gaps of waiting time between placing order and making decision have normal distribution


### Measurement ###

  1. finish\_time: the total time until all stock are sold out

> 2. success\_nowait: # of Tx that don't wait when placing orders

> 3. success\_wait: # of Tx that waited when placing but eventually checked out successfully

> 4. failed\_wait: # of Tx that waited but turned out of out of stock

> 5. failed\_nowait: # of Tx that out of stock info came out before placing order


### Revised Schema Design ###

  * base table "T":
```
create table t
(id integer not null primary key, 
val integer check (val >= 0 and val <= 1000000);
```

  * helper table "TMP":
```
create table tmp
(txid integer not null, 
id integer not null, 
inf integer check (inf >= 0), 
sup integer check (sup <= 1000000), 
updated_time timestamp default SYSTIMESTAMP,
primary key(txid, id));
```

  * helper view "V":
```
create view v 
as select id, val from t;
```

  * log table "JOURNAL":
```
create table journal
(txid integer not null primary key, 
id integer not null, 
amount integer not null, 
tx_status integer not null check (tx_status in (-1, 0, 1)), 
confirmed integer not null check (confirmed in (0, 1)));
```

  * trigger "UPDATE\_VAL" insert new record into table "TMP" and "JOURNAL":
```
create or replace trigger update_val 
instead of update on v 
for each row 
declare 
	pragma autonomous_transaction; 
	txid	int; 
	amount  int; 
	min 	int; 
	max	int; 
begin 
	amount := :new.val - :old.val; 
	select inf, sup into min, max from tmp 
	where id = :new.id and updated_time = (select max(updated_time) from tmp);
	select SYS_CONTEXT ( 'USERENV', 'SESSIONID') into txid from dual;
	if(:new.val > :old.val) then 
		insert into tmp values (txid, :new.id, min, max + amount, SYSTIMESTAMP);
	else 
		insert into tmp values (txid, :new.id, min + amount, max, SYSTIMESTAMP); 
	end if; 
insert into journal values(txid, :new.id, amount, 0, 0);
commit; 
end; 
/
```


### simple app level pseudo code ###
  * on top level:
```
runner()
{
	i is for calculate when to apply net gain and update base table "T";
	while(1) {
	    requests_count is a random number less than 1000;
	    ram is new random number generator;
	    create thread factory with requests_count workers;
	
	    for each worker
	    {
	        waittime = ram.nextGaussian();
	        exeucte(prodID, ranNum, waittime);
	    }

	    // calculate net gain each 100 seconds
	    if(!i%100) {
	        try{
	            get all committed but not confirmed tx's net value from table "JOURNAL";
	            update base table "T";
	            update "JOURNAL" setting their field "confirmed" to 1
	        } catch (SQLException) {
	            abort the transaction
	        }
	        if(out of stock)
	            break;
	    }

	    sleep(1);
	}

	getResult();
}
```

  * on worker level:
```
failed = true;
waited = false;
placed = true;

execute(int prodID, int orders, int waittime)
{
    place_order(prodID, orders);

    if(placed)
        during waittime
            for each 10 secinds
                try to place order again;

    try{
        update "JOURNAL" setting its tx_status to 1
        failed = false;
    } catch (SQLException) {
        update "JOURNAL" setting its tx_status to -1
    }
}

placeOrder(int prodID, int orders)
{
    try{
        update view "V"
    } catch (SQLException) {
        // must have violated constraints on inf/sup of "TMP"
        abort current transaction, but still need to insert the record to "JOURNAL"
        start another new transaction
        update "JOURNAL" setting its tx_status to 0
        waited = true;
        placed = false;    }
}
```




# Autonomous transaction on PostgreSQL and MySQL #
There is no autonomous transaction supported in PostgreSQL and MySQL.

However, here is an example emulating it in PostgreSQL: [Autonomous transaction in PostgreSQL](http://www.postgres.cz/index.php/PostgreSQL_SQL_Tricks#Autonomous_transaction_in_PostgreSQL), another example might be useful, [Port PL/SQL from Oracle to PostgreSQL](http://www.postgresql.org/docs/8.0/static/plpgsql-porting.html#PLPGSQL-PORTING-EX4).

The trick in the first example uses 2 different databases, similar to 2 different data engines in MySQL.



# Meeting Notes #
1. why do we need table "TMP" as well as "JOURNAL", can we combine them into one single table?
Yes we can, but we want to clean up table "TMP" time and again so that it's small enough to speed up our process. Thus, we place basic info in the "JOURNAL" table and details in "TMP".

2. rewrite the app level suedo code
  * on top level:
```
void runner()
{
	i is used to calculate the interval when should apply net gain and update base table "T";
	oos indicates the status of all stock, if 0 products left, end the test, intialized to 0;

	while(1) {

	    requests_count is a random number less than 1000;
	    ram is a random number generator;

	    create thread factory with requests_count workers;
	
	    for each worker
	    {
	        waittime = ram.nextGaussian();
	        exeucte(prodID, ranNum, waittime);
	    }

	    // calculate net gain each 100 seconds
	    if(!i%100)
	        oos = CONFIRM_ORDER();

	    if(oos)
		break;
	    else
		sleep(1);
	}

	getResult();
}

int CONFIRM_ORDER()
{
    left is the stock of products left after this update, initialized to 1;

    try{
	get all committed but not comfirmed tx's net value from table "JOURNAL";
	update base table "T" and get stock of left products;
	update table "JOURNAL" setting their field "confirmed" to TRUE;
    } catch(SQLException) {
	abort current transaction;
	start a new transaction to try again;
    } finally {
	if(left > 0)
	    return 0;
	else
	    return 1;
    }
}

```



  * on worker level:
```

failed = true;
waited = false;
placed = false;

void execute(int prodID, int orders, int waittime)
{
    PLACE_ORDER(prodID, orders);

    during waittime
	for each 10 seconds
	    if(!placed)
		try PLACE_ORDER(prodID, orders) again;

    // when finish, update the status of field "tx_status"
    try{
        update "JOURNAL" setting its tx_status to 1
        failed = false;
    } catch (SQLException) {
        update "JOURNAL" setting its tx_status to -1
    }

    storeResult(failed, waited);
}

void PLACE_ORDER(int prodID, int orders)
{
    try{
        update view "V" with prodID and orders;
	placed = true;
    } catch (SQLException) {
	abort current transaction;

	// there are two kinds of error, one is constraint violation error and the other one is real error
        // if violated constraints of "val" on "T" which is also implemented on inf/sup of "TMP"
	//autonomous transaction will automatically roll back, but current transaction can be executed successfully, we need to handle this
	// if real error happened, autonomous transaction must have finished, that's just what we want 
	// so we need to check if there is a record in "TMP"

	if(there is a record in "TMP")
	    placed = true;
	else
	    waited = true;

    }
}

void SUBMIT_ORDER()
{
    try {
	if(placed) {
	    update "JOURNAL" setting its tx_status to 1;
	    failed = false;
	}
    } catch(SQLException){
	try to submit order again, if this fails too, tell user error happens,
	and update "JOURNAL" setting its tx_status to -1;
    }

}


```



# Action Points #
Write client code based on Steve's test framework.