Following is a comment by voidmain on HN explaining fdb's transaction implemention.

voidmain:

```
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
```

```
Q: Can you elaborate on how 6 is actually accomplished? Various earlier comments have hinted that the transactional authority (conflict checking) can actually scale 'horizontally' beyond the check-throughput that can be archived by a single node. Is that the case? and whats the magic sauce for doing that for multi-object transactions? :)
```

voidmain:

```
Yes, conflict resolution is for most workloads a pretty small fraction of total resource use so you usually don't need a ton of resolvers (I think out of the box it still comes configured with just one?), but it can scale conflict resolution horizontally.
The basic approach isn't super hard to understand, though the details are tricky. The resolvers partition the keyspace; a write ordering is imposed on transactions and then the conflict ranges of each transaction are divided among the resolvers; each resolver returns whether each transaction conflicts and transactions are aborted if there are any conflicts.

(In general the resolution is sound, but not exact - it is possible for a transaction C to be aborted because it conflicts with another transaction B, but transaction B is also aborted because it conflicts with A (on another resolver), so C "could have" been committed. When Alec Grieser was an intern at FoundationDB he did some simulations showing that in horrible worst cases this inaccuracy could significantly hurt performance. But in practice I don't think there have been a lot of complaints about it.)
```

About number of replicas

```
FoundationDB stores 2N+1 copies of some "coordination state" and does a consensus algorithm whenever it is updated. But this state doesn't contain a copy of your data; basically think of it as storing a replication configuration. It's very small and rarely changes.
In the happy case, replication takes place using the replicas and quorum rules specified by this configuration. For example, you might require writes to succeed synchronously against all N+1 replicas of some transaction log. After N failures, there will still be 1 replica remaining with the latest transactions. But in order to proceed after any failures, you have to do a consensus transaction against a majority of replicas of the coordination state, to specify the new set of N+1 replicas you will be using. And you also make sure that the 1 replica you are recovering from knows you are doing it, so that it won't continue to accept writes under the old replication configuration.

There can't be two partitions capable of committing transactions, because (in this case) you need either

(a) All N+1 replicas of the log, so that you can commit synchronously, or (b) A majority (N+1 out of 2N+1) of the replicas of the coordination state, AND 1 replica of the log

Sorry if this isn't a great explanation. Anyway it does work. I expect that you could rephrase this as an optimization of a consensus protocol, though I think it would be hard to build a performant and realistically featureful implementation that way.

...

You only have to write to the coordination state when there is a failure. You can commit millions of transactions in the happy case without ever doing such a write. And failure detector performance and other engineering concerns are usually more of a limitation, in practice, on the performance of recovery than the latency of the coordination state consensus, even when the coordinators are geographically distributed.

```

More on serializability and linearizability.

```
Yes. Linearizable means both serializable and externally consistent (or is sometimes used as just a synonym for the latter), and FDB has these properties with respect to transactions.

Q from polskibus: That would mean that SSI is not used to provide Serializable isolation. level. If so, what is used instead? 2 phase locking? I thought it's not very scalable ?

I guess textbook SSI is willing to "reorder" conflicting transactions if the result is still serializable, which could violate external consistency if you don't have any other bounds on the order. In the language of SSI, fdb simply aborts the later of any pair of read/write transactions with an rw-conflict, in accordance with a fixed ordering which is externally consistent.

I guess it could also be that your book uses an idiosyncratic definition of linearizable, like trying to apply it to individual operations within transactions, which might rule out any optimistic concurrency method. It might just be better to delete this word from your vocabulary in the database field because there is no wide agreement on what it means. The first two hits on Google for me are Wikipedia and Peter Bailis, and they give clearly conflicting definitions, though I think fdb satisfies both!

Followup from polskibus: 

Thanks, I'd love to have a bit more of your attention, foundationdb seems very interesting but I need to know a bit more :)
Let me expand the definition in Kleppmann's book then.I think it is important because it creates a difference between SSI and typical Serializable level based on 2PL. The below is paraphrasing the definitions on p. 324-329. The book references http://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf. (I must admit, I read the book, not the paper).

Basic idea - make a system appear as if there were only one copy of the data and ALL operations on it are atomic. In this model, there may be replicas, but we don't care about them. As soon as a client completes a write to the db, all clients reading the db must be able to see the value just written.

In SSI this is not true, because you may the snapshot may not include writes more recent than the snapshot -> reads from the snapshot are not lineraizable.

Linearizable CAS register is equivalent to consensus, and can provide total order. It is therefore what most developers would love to have (if cost was not an issue :) )

voidmain:

From the paper you link:
"A history is serializable if it is equivalent to one in which transactions appear to execute sequentially, i.e., without interleaving... A history is strictly serializable if the transactions’ order in the sequential history is compatible with their precedence order... Linearizability can be viewed as a special case of strict serializability where transactions are restricted to consist of a single operation applied to a single object."

In these terms, FoundationDB has the strict serializability property, and thus if you do exactly one operation in each FoundationDB transaction then that is linearizable.

But that kind of linearizability is much less powerful than what FoundationDB actually gives you. You cannot efficiently maintain global invariants, like indexes, with single-operational linearizability. I don't think this definition is very useful! I think strict serializability (which is to say serializability & external consistency) is what you actually want.

A linearizable CAS register can be implemented in FDB as simply as this:

  @fdb.transactional
  def compare_and_set( tr, key, vold, vnew ):
    if tr[key] == vold:
      tr[key] = vnew

but this is not the limit of what you can do.
```

