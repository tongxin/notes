## Overview

A group of postgres servers join together to provide a distributed database.
Primary features include automatic table sharding, distributed transactions, 
and intelligent query planning.

- Users and applications can query and modify data cluster wide on any member
server if the target relation is distributed across the cluster. 
- The cluster state and sharding metadata are HA and consistency guaranteed, 
protected by reliable state machine replication mechanisms.


## Cluster management

To simplify design and usage, we don't distinguish between clusters. A database
server is either standalone or a member of a database cluster. The configuration
of the cluster that's visible to its members is maintained consistent using
Paxos-like protocol. Any membership changes, group acknowledged failures and
other cluster-wide configuration changes will be logged and applied to the
metadata on all the cluster members. 

Membership information is stored in pg_cluster system catalog, in which each
row represents a member server.

pg_cluster is defined with following columns.

| Name     | Type          | Description              |
|----------|---------------|--------------------------|
| oid      | oid           | Row identifier           |
| name     | name          | Name of the member server|
| conn     | text          | Connection string        |
| state    | text          | Current known state      |

Use following commands to discover, join and leave a cluster.

```
DISCOVER CLUSTER ON server

JOIN CLUSTER ON server

LEAVE CLUSTER
 
```

## DDL for replicated and distributed tables

To shard a table, we have to specify how the relation is going to be
distributed at the table creation time. This typically requires information
about the distribution type and the key column. 

```
CREATE DISTRIBUTED TABLE table (...) BY HASH (col)
CREATE REPLICATED TABLE table (...)
DROP DISTRIBUTED TABLE table  
```
The metadata about all the distributed tables are maintained in the system
catalog pg_distributed_tables. An update of this catalog happens if and only
if a distributed table command is successfully committed, which requires 
going throug paxos.

## Distributed execution model

Previous sharding solutions such as Postgres-xl and postgrespro are mostly
built on an execution model in which a remote query is decomposed to subqueries
that each directly relates to a particular remote node. They may adopt different
remote querying methods through fdw or other custom interfaces out of efficiency
concern but the planning complexity is inevitably high. We resort to a layered
execution model in which the global query interface defines a clear abstraction
to the client query system. The global query interface is like foreign data
service exposed through fdw except that it's not associated with a particular
server address.     

## Query planning and execution


## Cluster wide communication and reliable broadcast


## Session management

