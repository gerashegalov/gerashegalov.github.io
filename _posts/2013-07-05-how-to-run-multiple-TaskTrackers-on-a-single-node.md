---
layout: post
---

This is a somewhat exotic setup. There are use cases where Hadoop cluster
consists of very large scale up machines where we have capacity for 3-digit
numbers of map and reduce slots. This results in considerable overhead incurred
by TT for launching and tracking task attempts. To mitigate this, we need
multiple TT's on the same node. This is achieved easiest using some sort of
virtualization, e.g., creating multiple VM nodes. However, this leads to
unnecessary resource fragmentation and introducing another layer in the I/O
path. There is another way around this: run multiple tasktrackers on the same
OS.

To run multiple tasktrackers simultaneously, we need some customization in
configuration. Therefore for each TT X, with X=1..N, we need an extra directory
confX. The following is the minimum configuration. 

First we need to set a dedicated ```hadoop.tmp.dir``` to 
```/tmp/hadoop-${user.name}-X``` 

{% highlight bash %}
$ cd ${HADOOP_HOME}
$ cat confX/core-site.xml
{% endhighlight %}
{% highlight xml %}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://localhost:9000</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/tmp/hadoop-${user.name}-X</value>
  </property>
</configuration>
{% endhighlight %}

We also need to change ```mapred.task.tracker.http.address``` because it 
normally uses a static http port 50060. We can either assign manually different
port to each X or we let the OS do it by using ```0.0.0.0:0```. We don't really
need a well defined http port for TT because it is sent with heartbeats to JT.
So we can have a single ```mapred-site.xml``` for all X as follows:

{% highlight bash %}
$ cat confX/mapred-site.xml
{% endhighlight %}
{% highlight xml %}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>mapred.job.tracker</name>
    <value>localhost:9001</value>
  </property>
  <property>
    <name>mapred.task.tracker.http.address</name>
    <value>0.0.0.0:0</value>
  </property>
</configuration>
{% endhighlight %}

Then to start a TT X, we need to fix ```${HADOOP_IDENT_STRING}``` and specify
a proper confX

{% highlight bash %}
$ export HADOOP_IDENT_STRING=${USER}-X
$ bin/hadoop-daemon.sh --config confX start tasktracker
{% endhighlight %}
