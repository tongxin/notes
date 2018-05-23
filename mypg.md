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

To shard a table, we have to specify how the distribution is to be carried out
at the table creation time. This typically requires information about 
distribution type and partition column. 

```
CREATE DISTRIBUTED TABLE table (...) BY HASH (col)
CREATE REPLICATED TABLE table (...)
DROP DISTRIBUTED TABLE table  
```

All the distributed tables are described in the system catalog pg_distributed_tables. 


