# It all started from a CannotAcquireLockException...

One day a team member was working on a data migration service and when he triggered the migration he met the following error
```
org.springframework.dao.CannotAcquireLockException: Could not execute JDBC batch update; SQL [insert into abc_invoice (col1, col2) values ...
```
It wasn't a hard question but a good one for us to reflect and review the concepts of deadlocks, transaction isolations, and how to troubleshoot related problems in MySQL.

## Where does this error occur?
Structure of the migration service looks like this:

controller --[calls]--> migration scheduler --[sends]--> listener --[calls]--> migration helper service --[calls]--> other services

The exception occurs in the following method in migration helper service
```java
@Transactional
public void migrateOneDocument(Boolean reMigration, String documentId, String userId) {
    if (reMigration) {
        clearExistingRecords(documentId, userId);
    }
    doMigration(documentId, userId);
}
```

## What does this error mean?
```
org.springframework.dao.CannotAcquireLockException: Could not execute JDBC batch update; SQL [insert into abc_invoice (col1, col2) values (?, ?)]; nested exception is org.hibernate.exception.LockAcquisitionException: Could not execute JDBC batch update
at org.springframework.orm.hibernate3.SessionFactoryUtils.convertHibernateAccessException(SessionFactoryUtils.java:651) ~[spring-orm-4.3.30.RELEASE.jar:4.3.30.RELEASE]
at org.springframework.orm.hibernate3.HibernateTransactionManager.convertHibernateAccessException(HibernateTransactionManager.java:800) ~[spring-orm-4.3.30.RELEASE.jar:4.3.30.RELEASE]
... more
...Caused by: org.hibernate.exception.LockAcquisitionException: Could not execute JDBC batch update
at org.hibernate.exception.SQLStateConverter.convert(SQLStateConverter.java:107) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.hibernate.exception.JDBCExceptionHelper.convert(JDBCExceptionHelper.java:66) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.hibernate.jdbc.AbstractBatcher.executeBatch(AbstractBatcher.java:275) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.hibernate.engine.ActionQueue.executeActions(ActionQueue.java:268) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.hibernate.engine.ActionQueue.executeActions(ActionQueue.java:184) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.hibernate.event.def.AbstractFlushingEventListener.performExecutions(AbstractFlushingEventListener.java:321) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.hibernate.event.def.DefaultFlushEventListener.onFlush(DefaultFlushEventListener.java:51) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.hibernate.impl.SessionImpl.flush(SessionImpl.java:1216) ~[hibernate-core-3.6.10.Final.jar:3.6.10.Final]
at org.springframework.orm.hibernate3.HibernateTransactionManager$HibernateTransactionObject.flush(HibernateTransactionManager.java:900) ~[spring-orm-4.3.30.RELEASE.jar:4.3.30.RELEASE]
... 45 more
Caused by: java.sql.BatchUpdateException: Lock wait timeout exceeded; try restarting transaction
at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[?:1.8.0_292]
at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62) ~[?:1.8.0_292]
at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[?:1.8.0_292]
at java.lang.reflect.Constructor.newInstance(Constructor.java:423) ~[?:1.8.0_292]
... 45 more
Caused by: com.mysql.cj.jdbc.exceptions.MySQLTransactionRollbackException: Lock wait timeout exceeded; try restarting transaction
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:123) ~[mysql-connector-java-8.0.24.jar:8.0.24]
at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122) ~[mysql-connector-java-8.0.24.jar:8.0.24]
at com.mysql.cj.jdbc.ClientPreparedStatement.executeInternal(ClientPreparedStatement.java:953) ~[mysql-connector-java-8.0.24.jar:8.0.24]
at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1092) ~[mysql-connector-java-8.0.24.jar:8.0.24]
at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1040) ~[mysql-connector-java-8.0.24.jar:8.0.24]
... 45 more
```
From the stack trace we know that it's an exception thrown in Spring where it cannot acquire the lock it needs to insert data. In addition, there's an error from MySQL JDBC level indicating that the lock wait time out exceeded, meaning this thread had been waiting until timeout but still didn't get the lock.  


## How to verify the cause?
We can find information of every currently executing transaction inside InnoDB using `INNODB_TRX` table., such as if the transaction is started, if it is waiting for a lock, etc.
It's more useful to join `INNODB_LOCKS` on `trx_requested_lock_id` and `lock_id`, to get details of the lock held and requested

```
mysql> select tnx.*, lck.* from INFORMATION_SCHEMA.INNODB_TRX tnx join INFORMATION_SCHEMA.INNODB_LOCKS lck on tnx.trx_requested_lock_id = lck.lock_id\G
*************************** 1. row ***************************
                    trx_id: 819311
                 trx_state: LOCK WAIT
               trx_started: 2023-03-20 09:47:09
     trx_requested_lock_id: 819311:4975:6:6
          trx_wait_started: 2023-03-20 09:47:28
                trx_weight: 3
       trx_mysql_thread_id: 12656
                 trx_query: insert into abc_invoice (col1, col2 ...
       trx_operation_state: inserting
         trx_tables_in_use: 1
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
                   lock_id: 819311:4975:6:6
               lock_trx_id: 819311
                 lock_mode: X,GAP
                 lock_type: RECORD
                lock_table: `repos`.`abc_invoice`
                lock_index: IDX_DOCUMENT_ID
                lock_space: 4975
                 lock_page: 6
                  lock_rec: 6
                 lock_data: '4028b2e986e9b1620186e9cc7351111', '4028b2e986fd478c0186fd54768a2222'     # id, uuid, primary key
1 row in set, 1 warning (0.00 sec)
```

From the query result above, we can see that our transaction is indeed waiting for the lock needed to insert data:
`trx_state: LOCK WAIT`, `trx_query: insert into abc_invoice (col1, col2...`, `trx_operation_state: inserting`

The next step is to find out where the other transaction is and where it holds the lock for so long. With further analysis we realized the problem is the method call `doMigration(documentId, userId)` inside `migrateOneDocument(Boolean reMigration, String documentId, String userId)` because it actually starts a new transaction when it handles the actual migration logics. It reads something like
```java
private void doMigration(String documentId, String userId) {
        ...
        handler.handle(documentId, userId);
}
```

Code in handler.java
```java
public void handle(String documentId, String userId) {
        ...
        transactionHelper.runInNewTx(s -> {
            processItems(context, itemId, temItemIds);
        })
}
```

Recall `migrateOneDocument()` is annotated with `@Transactional`. At the beginning of this transaction is a delete statement like `delete from abc_invoice where user_id=xx and document_id=xx` (with index on (user_id, document_id)) where it locks one row (Step A). However, in the next step (Step B), `INSERT`, a new transaction is started. Step A is waiting Step B to finish so that the transaction Step A is in can be committed; while Step B is waiting for Step A to release the lock to proceed. Hence, the deadlock occurs.


## How to fix?
Either remove the `@Transactional` or don't start a new transaction inside an existing transaction, as MySQL doesn't support nested transactions.

## Follow-up questions
### The data I'm trying to insert has different primary keys than the records that are being locked to delete. Why won't MySQL allow me to insert?
[TODO]
