## 8/23/2017

Some preparation work for the quad servers' system updates.

Firstly did the IPMI firmware update 

Motherboard model: Supermicro X10SLE-DF

IPMI IP: 172.20.36.249

## 8/29/2017

Cleaned up disk space on quad201-204, outdated directories and files and accounts removed.

Removing cloudera-manager stuff on the cluster should follow this link:

https://www.cloudera.com/documentation/enterprise/5-6-x/topics/cm_ig_uninstall_cm.html

iptables quick ref:

```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Use alternatives command utility to manage /etc/alternatives

For example, do

```
alternative --config java
```

to update links to the desired java binary

## 8/30/2017

Cleaned disk space on my laptop. Removed mongodb built binaries entirely.

```
scons MONGO_VERSION=3.5.10 -c install
```

## 9/12/2017

Reading from a ByteBuffer wrapped array is at least 2.5x slow than reading from the array directly.

## 9/21/2017

Tricky parts of building Spark inside Intellij: generated code won't be installed in the source path automatically. Rule of thumb: do a first round sbt package, then add the source path to the generated source in the project structure tab.

## 9/26/2017

When diagnosing which address and port a network service is bounded to and listening on, use this command

```
netstat -nlt 
```

When postgres is not able to be connected from outside local host, check that the bound address is correctly configured in ``<data>/postgresql.conf''

## 9/28/2017

JDBC keeps failing to connect remote server. it turns out that JVM can detect network proxy at start up and sets the socksProxyHost system property accordingly.
The JDBC implementation refuses to resolve host address if this property is set.

## 10/6/2017

On ubuntu, use `dpkg -L <package>` to list the locations where the package files saved.

To use apt-get with http proxy, create `/etc/apt/apt.conf` and include one line of proxy setting  as follows 
```
Acquire::http::Proxy "http://172.20.110.182:1087";
```

Set the following environment for running jvm with http proxy settings.
```
export JAVA_OPTS="$JAVA_OPTS -Dhttp.proxyHost=yourserver -Dhttp.proxyPort=8080 -Dhttp.proxyUser=username -Dhttp.proxyPassword=password"
```

Replace the above http with https if the application relies on https connection. 

## 10/8/2017

Plan for today and tomorrow: 

install and set up Spark/Impala/Myria on 172.16.101.115

Update postgres to release 10 on 172.16.101.115

Set up tpc-ds on 172.16.101.115
 
To upgrade existing postgres database cluster to a newer major version, use pg_upgrade. See https://www.postgresql.org/docs/current/static/pgupgrade.html for reference.

## 11/8/2017

CSV data source for Spark SQL 

https://github.com/databricks/spark-csv

```
$SPARK_HOME/bin/spark-shell --packages com.databricks:spark-csv_2.11:1.5.0

```

## 12/21/2017

After laptop updated to High Sierra (10.13.2), trying ssh connection to local server all failed due to unknown identification issues. It turns out the culprit is the local socks proxy. When the socks is diabled, ssh works fine. 


## 1/23/2018

Postgres performance tuning.

To exploit the best tps and latency requires deep understanding of how different server configurations impact db performance and how os interferes with the db server when theres resource constraints and contention.

First, we should know the underlying hardware. Some figures are mostly importmant, such as the disk speed, memory size, cpu frequency and number of processors.

Standard tools for measuring disk IO performance.

Have cache activated using hdparm -W1, then measure write streaming performance with dd. Make sure that results are averaged from multiple runs.

```
bai@ccrfox248:~$ sudo hdparm -W1 /dev/sda1
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 12.4466 s, 86.3 MB/s
```

With cache deactivated(hdparm -W0), measure the write streaming again.

```
bai@ccrfox248:~$ sudo hdparm -W0 /dev/sda1
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 18.0897 s, 59.4 MB/s
```

Cache on, small writes.
```
bai@ccrfox248:~$ sudo hdparm -W1 /dev/sda1
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=512 count=1000 oflag=direct1000+0 records in
1000+0 records out
512000 bytes (512 kB) copied, 0.250835 s, 2.0 MB/s
```

Cache off, small writes.
```
bai@ccrfox248:~$ sudo hdparm -W0 /dev/sda1
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=512 count=1000 oflag=direct1000+0 records in
1000+0 records out
512000 bytes (512 kB) copied, 15.6599 s, 32.7 kB/s
```

Cache on, streaming, synchronized IO
```
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=1G count=1 oflag=dsync
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 14.3929 s, 74.6 MB/s
```

Cache off, streaming, synchronized IO
```
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=1G count=1 oflag=dsync
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 15.7831 s, 68.0 MB/s
```

Cache on, small writes, synchronized IO, VERY SLOW
```
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=512 count=1000 oflag=dsync
1000+0 records in
1000+0 records out
512000 bytes (512 kB) copied, 57.6155 s, 8.9 kB/s
```

Cache off, small writes, synchronized IO, VERY SLOW
```
bai@ccrfox248:~$ dd if=/dev/zero of=/tmp/testfile bs=512 count=1000 oflag=dsync
1000+0 records in
1000+0 records out
512000 bytes (512 kB) copied, 66.9303 s, 7.6 kB/s
```

What are direct IO, synchronized IO and nonblocking IO anyways?

Direct IO avoids caching in the kernel and sends IO requests directly to disk.

In Linux terminology, synchronized IO means writes won't return until the data is actually persisted to the storage whereas blocking IO simply means the writes return when the data is copied to the kernel.

Use `hdparm` command on Linux to set SATA/IDE device parameters.

Use `iostat` command to measure IO usage.

On the test machine in our lab, the measured IOPS are ~250 for streaming direct write, ~170 for streaming synchronized write.

Postgres performance test is done using pgbench. A quick web search reveals that people have done a nice comparison study using benchmarks for both Postgres and MySQL. Here we follow a blog post from Percona. The source is downloaded from https://github.com/postgrespro/pg_oltp_bench.git

## 1/25/2018

Postgres tuning.

https://github.com/jfcoz/postgresqltuner

postgresqltuner is simple configuration advice tool for postgres.

By running a quick screening in one of our machines, the tool sugggested several system parameter changes:

```
=====  Configuration advices  =====
-----  checkpoint  -----
[MEDIUM] Your checkpoint completion target is too low. Put something nearest from 0.8/0.9 to balance your writes better during the checkpoint interval
-----  extension  -----
[LOW] Enable pg_stat_statements to collect statistics on all queries (not only queries longer than log_min_duration_statement in logs)
-----  sysctl  -----
[URGENT] set vm.overcommit_memory=2 in /etc/sysctl.conf and run sysctl -p to reload it. This will disable memory overcommitment and avoid postgresql killed by OOM killer.
```

OS memory management settings that affect writeout performance:

vm.dirty_expire_centisecs
vm.overcommit_memory
vm.overcommit_ratio

Some pg parameters that matter most to performance:

shared_buffers

effective_cache_size

checkpoint_timeout

checkpoint_completion_target

work_mem

wal_sync_method

wal_buffers

...

Running pgbench suggests the RW workload only achieves ~400 tps, consuming about 40MB of write bandwidth. The bandwidth usage is similar to writing 1MB blocks continuously, which can be simulated by running following dd command:

```
dd if=/dev/zero of=/tmp/testfile bs=1M count=10000 oflag=dsync
```

## 4/11/2018

Postgres 10 hot standby setup.

On master, edit postgresql.conf to enable replication and archiving. Most importantly, set the following parameters to the desired values. 

```
wal_level = replica
synchronous_standby_names = 'standby1,standby2'
synchronous_commit = on   # for synchronous streaming
archive_mode = on
archive_command = 'test ! -f /usr/local/pgsql/data/archive/%f && cp %p /usr/local/pgsql/data/archive/%f'
```

How synchronously the master waits for the standbys when committing a transaction is determined by the `synchronous_commit` setting. Details refer to [Postgresql 10's document](https://www.postgresql.org/docs/10/static/warm-standby.html). Briefly, remote_write requires transaction committing to wait for the replies that the standbys have written the commit record to os, remote_apply for the replies that the standbys have replayed the transaction on their own databases.

Next, we create a replication slot for each standby.

```
psql -c 'SELECT pg_create_physical_replication_slot("standby1");'
```

Restart the master server and now switch to the standby machine.

The first thing to do is backup the master database. We use the standard pg backup tool pg_basebackup. 

With pg_basebackup, the data of the master database cluster is copied to the standby machine where the standby server can start streaming right away against what's copied over. 

```
pg_basebackup -D standby -Xs -S standby1 -c fast -P -R -l 'initial backup' -h 172.16.101.115 -p 5432
```

Remember to set hot_standby to on if not yet.

Start the standby postgres server.

## 4/16/2018

Some useful system performance monitoring tools.

sysstat utilities: iostat, pidstat, sar

dstat, iotop

## 9/3/2018

Use ``parted'' on linux for GPT partitioning. Other useful fs/disk commands: ``lsblk'', for showing block device information, ``blkid'', for printing UUIDs for each block device, and ``df -h'', for listing file systems displayed in a human readable fashion.

## 9/8/2018

Use ``--noreloda'' command line option when debugging Django server app if the debugger doesnt support following child processes.

## 9/12/2018

Ooops, I accidentally removed all the entire directory of /usr/local/lib and fortunately was saved by ``extundelete --restore-all /dev/sda4''.

## 10/28/2018

A few examples of wrapping sql queries in fab commands.

```
from fabric.api import (env, hide, settings, local, task, lcd)
from fabric.state import output



def build_psql(sql):
    return 'psql -p 5433 -d xsbench -c "{}"'.format(sql)

@task
def add_node(host, user, passwd):
    sql = "SELECT xschema.add_node('host={} dbname=xsbench port=5433 user={} password={}', repl_group=>'rg_0');".format(host, user, passwd)
    cmd = build_psql(sql)
    with hide('running'):
        local(cmd)

@task
def show_nodes():
    sql = "select id as node_id, connection_string from xschema.nodes;"
    cmd = build_psql(sql)
    with hide('running'):
        local(cmd)
@task
def shard_table(table_name, sharding_key, total_partitions):
    sql = "select xschema.create_hash_partitions('{}', '{}', '{}', 0)".format(table_name, sharding_key, total_partitions)
    cmd = build_psql(sql)
    with hide('running'):
        local(cmd)

@task
def show_partitions(table_name):
    sql = "select part_name as partition, node_id, relation as table from xschema.partitions where relation = {}".format(table_name)
    cmd = build_psql(sql)
    with hide('running'):
        local(cmd)

@task
def move_partition(partition, node):
    sql = "select xschema.mv_partition('{}', '{}')".format(partition, node)
    cmd = build_psql(sql)
    with hide('running'):
        local(cmd)

@task
def show_standbys():
    sql = "select client_addr as standby_addr, client_port as port, backend_start, state, sent_lsn, write_lsn, flush_lsn, replay_lsn, sync_state from pg_stat_replication;"
    cmd = build_psql(sql)
    with hide('running'):
        local(cmd)

@task
def throttle_connections(ratio):
    sql = "select count(*) as removed from (select pid, pg_terminate_backend(pid) as terminated from pg_stat_activity where pid <> pg_backend_pid() and random() < {}) a;".format(ratio)
    cmd = build_psql(sql)
    with hide('running'):
        local(cmd)

```
