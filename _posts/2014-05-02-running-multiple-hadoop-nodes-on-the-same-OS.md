---
layout: post
title: Running multiple hadoop nodes on the same physical OS/node
---

Sometimes we need to reproduce a complex bug that occurs in our production environment. It might be cumbersome to debug on the cluster itself, and running multiple VM's on the same laptop isn't always possible, and always inconvenient. Our issues may include NameNode (NN) High Availability (HA), NN Federation with ViewFs, DataNodes (DN), and YARN (ResourceManager and NodeManagers). In this post we describe how to run multiple hadoop daemons that are usually assigned dedicated hardware without any VM's. Unlike MiniCluster in hadoop unit tests, each daemon will be run realistically in a dedicated JVM each.

The following setup emulates two nodes NODE1 and NODE2. 
- NODE1 runs NN1 for namespace ns1, the YARN RM, and worker daemons, a DN and an NM.
- NODE2 runs NN1 for namespace ns2, and worker daemons, DN and NM.

The configuration for all nodes is stored in the directory ```${HADOOP_CONF_DIR}```.
When you build hadoop from scratch using 
{% highlight bash %}
$ mvn clean package -Pdist -DskipTests -Dmaven.javadoc.skip
$ export G_HADOOP_HOME=${PWD}/hadoop-dist/target/hadoop-2.3.1-SNAPSHOT
$ export HADOOP_CONF_DIR=${G_HADOOP_HOME}/etc/hadoop
{% highlight %}

{% highlight bash %}
[devbox hadoop-common (branch-2.3)]$ cd ${HADOOP_CONF_DIR}
[devbox hadoop]$ ls -al .
total 112
drwxr-xr-x  19 osuser  staff    646 May  2 01:48 .
drwxr-xr-x   6 osuser  staff    204 May  2 01:42 ..
-rw-r--r--   1 osuser  staff    496 May  1 17:02 core-site.xml
-rw-r--r--   1 osuser  staff   3290 May  1 23:56 hadoop-env.sh
-rw-r--r--   1 osuser  staff   2490 May  1 18:41 hadoop-metrics.properties
-rw-r--r--   1 osuser  staff   1774 May  1 18:41 hadoop-metrics2.properties
-rw-r--r--   1 osuser  staff   2691 May  1 17:02 hdfs-site.xml
drwxr-xr-x   4 osuser  staff    136 May  1 23:59 hdfs1
drwxr-xr-x   4 osuser  staff    136 May  1 23:59 hdfs2
-rw-r--r--   1 osuser  staff   1449 May  1 18:33 httpfs-env.sh
-rw-r--r--   1 osuser  staff   1657 May  1 18:41 httpfs-log4j.properties
-rw-r--r--   1 osuser  staff  11169 May  1 18:41 log4j.properties
drwxr-xr-x  15 osuser  staff    510 May  2 00:32 logs
-rw-r--r--   1 osuser  staff   1383 May  1 18:33 mapred-env.sh
-rw-r--r--   1 osuser  staff    242 May  1 17:02 mapred-site.xml
drwxr-xr-x   3 osuser  staff    102 May  1 17:21 tmp1
drwxr-xr-x   3 osuser  staff    102 May  1 17:22 tmp2
-rw-r--r--   1 osuser  staff   4084 May  1 18:33 yarn-env.sh
-rw-r--r--   1 osuser  staff   2153 May  1 17:02 yarn-site.xml
{% highlight %}

Since we run multiple daemons of the same kind on the same node, we need to make sure that HADOOP/YARN_IDENT_STRING can be passed externally to the launch scripts. The line ```export HADOOP_IDENT_STRING``` in hadoop-env.sh has to be removed or modified according to [HADOOP-9979](https://issues.apache.org/jira/browse/HADOOP-9979).

core-site.xml contains a ViewFs mounttable. ns1 NN contains /user and ns2 NN contains /tmp

##### core-site.xml
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>${my.hadoop.tmp.dir}</value>
  </property>
  <property>
    <name>fs.defaultFS</name>
    <value>viewfs:///</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.default.link./user</name>
    <value>hdfs://ns1/user</value> 
  </property>
  <property>
    <name>fs.viewfs.mounttable.default.link./tmp</name>
    <value>hdfs://ns2/tmp</value> 
  </property>
</configuration>

{% highlight %}

In *-site.xml, we make sure that all listener ports are distinct, and the directory locations can be passed as System properties.

##### hdfs-site.xml
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <!-- specific to node -->
  <property>
    <name>dfs.nameservice.id</name>
    <value>${my.dfs.nameservice.id}</value>
  </property>
  <property>
    <name>dfs.name.dir</name>
    <value>${my.hdfs.home.dir}/name</value>
  </property>
  <property>
    <name>dfs.checkpoint.dir</name>
    <value>${my.hdfs.home.dir}/namesecondary</value>
  </property>
  <property>
    <name>dfs.data.dir</name>
    <value>${my.hdfs.home.dir}/data</value>
  </property>
  <property>
    <name>dfs.datanode.address</name>
    <value>${my.dfs.datanode.address}</value>
  </property>
  <property>
    <name>dfs.datanode.http.address</name>
    <value>${my.dfs.datanode.http.address}</value>
  </property>
  <property>
    <name>dfs.datanode.ipc.address</name>
    <value>${my.dfs.datanode.ipc.address}</value>
  </property>
  <!-- end: specific to node -->

  <property>
    <name>dfs.nameservices</name>
    <value>ns1,ns2</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.ns1</name>
    <value>nn-ns1</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.ns2</name>
    <value>nn-ns2</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.ns1.nn-ns1</name>
    <value>localhost:8020</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.ns1.nn-ns1</name>
    <value>localhost:50070</value>
  </property>
  <property>
    <name>dfs.namenode.secondaryhttp-address.ns1.nn-ns1</name>
    <value>localhost:50090</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.ns2.nn-ns2</name>
    <value>localhost:9020</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.ns2.nn-ns2</name>
    <value>localhost:60070</value>
  </property>
  <property>
    <name>dfs.namenode.secondaryhttp-address.ns2.nn-ns2</name>
    <value>localhost:60090</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.enable.federation.redirect</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.client.failover.proxy.provider.ns1</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <name>dfs.client.failover.proxy.provider.ns2</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
</configuration>
{% highlight %}

##### yarn-site.xml 
{% highlight xml %}
<?xml version="1.0"?>
<configuration>
  <!-- node specific: begin -->
  <property>
    <name>yarn.nodemanager.localizer.address</name>
    <value>${my.yarn.nodemanager.localizer.address}</value>
  </property> 
  <property>
    <name>yarn.nodemanager.address</name>
    <value>${my.yarn.nodemanager.address}</value>
  </property>
  <property>
    <name>yarn.nodemanager.webapp.address</name>
    <value>${my.yarn.nodemanager.webapp.address}</value>
  </property>
  <property>
    <name>yarn.nodemanager.health-checker.script.opts</name>
    <value>${my.decommission.file}</value>
  </property>
  <!-- node specific: end -->

  <property>
    <name>yarn.nm.liveness-monitor.expiry-interval-ms</name>
    <value>60000</value>
  </property>
  <property>
    <name>yarn.am.liveness-monitor.expiry-interval-ms</name>
    <value>60000</value>
  </property>
  <property>
    <name>yarn.log.server.url</name>
    <value>http://${mapreduce.jobhistory.webapp.address}/jobhistory/logs</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
  </property>
  <property>
    <name>yarn.nodemanager.unheathy.drain.containers</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.nodes.exclude-path</name>
    <value>/tmp/yarn-exclude</value>
  </property>
  <property>
    <name>yarn.nodemanager.health-checker.interval-ms</name>
    <value>10000</value>
  </property>
  <property>
    <name>yarn.nodemanager.health-checker.script.timeout-ms</name>
    <value>10000</value>
  </property>
  <property>
    <name>yarn.nodemanager.health-checker.script.path</name>
    <value>/Users/osuser/bin/health.sh</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
  </property>
</configuration>
{% highlight %}

##### mapred-site.xml
{% highlight xml %}
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.shuffle.port</name>
    <value>${my.mapreduce.shuffle.port}</value>
  </property>
</configuration>
{% highlight %}

Now we can start all the daemons using the following script:

##### pseudo.sh
{% highlight bash %}
#!/bin/bash

if [ "$1" == "-debug" ]; then
  shift
  set -x
fi

cd ${HADOOP_CONF_DIR}
export HADOOP_LOG_DIR=${PWD}/logs
CMD=${1}

#
# node specific opts
#

HDFS_NODE1_OPTS="-Dmy.dfs.nameservice.id=ns1
  -Dmy.hadoop.tmp.dir=${PWD}/tmp1
  -Dmy.hdfs.home.dir=${PWD}/hdfs1
  -Dmy.dfs.datanode.address=0.0.0.0:50010
  -Dmy.dfs.datanode.http.address=0.0.0.0:50075
  -Dmy.dfs.datanode.ipc.address=0.0.0.0:50020"

HDFS_NODE2_OPTS="-Dmy.dfs.nameservice.id=ns2
  -Dmy.hadoop.tmp.dir=${PWD}/tmp2
  -Dmy.hdfs.home.dir=${PWD}/hdfs2
  -Dmy.dfs.datanode.address=0.0.0.0:50110
  -Dmy.dfs.datanode.http.address=0.0.0.0:50175
  -Dmy.dfs.datanode.ipc.address=0.0.0.0:50120"

YARN_NODE1_OPTS="
  -Dmy.hadoop.tmp.dir=${PWD}/tmp1
  -Dmy.yarn.nodemanager.localizer.address=0.0.0.0:8040
  -Dmy.yarn.nodemanager.address=0.0.0.0:8041
  -Dmy.yarn.nodemanager.webapp.address=0.0.0.0:8042
  -Dmy.mapreduce.shuffle.port=13562
  -Dmy.decommission.file=/tmp/decommission1"

YARN_NODE2_OPTS="
  -Dmy.hadoop.tmp.dir=${PWD}/tmp2
  -Dmy.yarn.nodemanager.localizer.address=0.0.0.0:8140
  -Dmy.yarn.nodemanager.address=0.0.0.0:8141
  -Dmy.yarn.nodemanager.webapp.address=0.0.0.0:8142
  -Dmy.mapreduce.shuffle.port=13563
  -Dmy.decommission.file=/tmp/decommission2"

if [ "$1" == "format" ]; then
  clid="MY.CID-$(date +%s)"

  export HADOOP_IDENT_STRING=${USER}-node1
  export HADOOP_NAMENODE_OPTS="${HDFS_NODE1_OPTS}"
  ${G_HADOOP_HOME}/bin/hdfs namenode -format -clusterId ${clid}

  export HADOOP_IDENT_STRING=${USER}-node2
  export HADOOP_NAMENODE_OPTS="${HDFS_NODE2_OPTS}"
  ${G_HADOOP_HOME}/bin/hdfs namenode -format -clusterId ${clid} 

  exit 0
fi

echo "${CMD} node1 daemons"
export HADOOP_IDENT_STRING=${USER}-node1

export HADOOP_NAMENODE_OPTS="${HDFS_NODE1_OPTS}"
${G_HADOOP_HOME}/sbin/hadoop-daemon.sh --config ${PWD} ${CMD} namenode   
export HADOOP_DATANODE_OPTS="${HDFS_NODE1_OPTS}" 
${G_HADOOP_HOME}/sbin/hadoop-daemon.sh --config ${PWD} ${CMD} datanode   

export YARN_IDENT_STRING=${HADOOP_IDENT_STRING}

export YARN_RESOURCEMANAGER_OPTS="-Dmy.hadoop.tmp.dir=${PWD}/tmp1"
${G_HADOOP_HOME}/sbin/yarn-daemon.sh --config ${PWD} ${CMD} resourcemanager

export YARN_NODEMANAGER_OPTS="${YARN_NODE1_OPTS}"
${G_HADOOP_HOME}/sbin/yarn-daemon.sh --config ${PWD} ${CMD} nodemanager   

echo "$CMD node2 daemons"
export HADOOP_IDENT_STRING=${USER}-node2
  
export HADOOP_NAMENODE_OPTS="${HDFS_NODE2_OPTS}" 
${G_HADOOP_HOME}/sbin/hadoop-daemon.sh --config ${PWD} ${CMD} namenode   
export HADOOP_DATANODE_OPTS="${HDFS_NODE2_OPTS}"
${G_HADOOP_HOME}/sbin/hadoop-daemon.sh --config ${PWD} ${CMD} datanode   

export YARN_IDENT_STRING=${HADOOP_IDENT_STRING}
export YARN_NODEMANAGER_OPTS="${YARN_NODE2_OPTS}"
${G_HADOOP_HOME}/sbin/yarn-daemon.sh --config ${PWD} ${CMD} nodemanager 

{% highlight %}

Before we launch the cluster, we need to format namespaces by executing
{% highlight bash %}
$ pseudo.sh format
# launch ihe cluster
$ pseudo.sh start
# stop the cluster
$ pseudo.sh stop
{% highlight %}




