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

## 10/8/2017

Plan for today and tomorrow: 

install and set up Spark/Impala/Myria on 172.16.101.115

Update postgres to release 10 on 172.16.101.115

Set up tpc-ds on 172.16.101.115
 

