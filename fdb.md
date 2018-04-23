Following is a comment by voidmain on HN explaining fdb's transaction implemention.

A FDB transaction roughly works like this, from the client's perspective:

1. Ask the distributed database for an appropriate (externally consistent) read version for the transaction

2. Do reads from a consistent MVCC snapshot at that read version. No matter what other activity is happening you see an unchanging snapshot of the database. Keep track of what (ranges of) data you have read

3. Keep track of the writes you would like to do locally.

4. If you read something that you have written in the same transaction, use the write to satisfy the read, providing the illusion of ordering within the transaction

5. When and if you decide to commit the transaction, send the read version, a list of ranges read and writes that you would like to do to the distributed database.

6. The distributed database assigns a write version to the transaction and determines if, between the read and write versions, any other transaction wrote anything that this transaction read. If so there is a conflict and this transaction is aborted (the writes are simply not performed). If not then all the writes happen atomically.

7. When the transaction is sufficiently durable the database tells the client and the client can consider the transaction committed (from an external consistency standpoint)

The implementations of 1 and 6 are not trivial, of course :-)

So a sufficiently "slow client" doing a read write transaction in a database with lots of contention might wind up retrying its own transaction indefinitely, but it can't stop other readers or writers from making progress.
