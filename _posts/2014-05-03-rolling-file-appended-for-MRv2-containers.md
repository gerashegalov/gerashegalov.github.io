---
layout: post
title: MRv2 Task log throttling using Rolling File Appender 
---

We can use mapreduse.userlog.limit.kb to limit the size of runaway MapReduce tasks. However, prior to 
[MAPREDUCE-5672](https://issues.apache.org/jira/browse/MAPREDUCE-5672), important log information leading to the failure
and a task crash might be missing because last mapred.userlog.limit.kb are cached in the JVM. 

Starting with Hadoop 2.3 we can use a more reliable method,
[RollingFileAppender](https://logging.apache.org/log4j/1.2/apidocs/org/apache/log4j/RollingFileAppender.html)
that is traditionally used to keep only maxBackupIndex of the log file and each file is at most maxFileSize. 

Now if we want to keep at most 2 megabyte of the container log we could specify mapreduce.task.userlog.limit.kb=512 and
yarn.app.mapreduce.task.container.log.backups=3 yielding the current file "syslog", and three rolled files (backups) "syslog.1",
"syslog.2", "syslog.3". The current file will have at most 512KB, and the backup files almost exactly 512KB.

MRAppMaster is a special container tracking MRv2 jobs. There only relatively few AM containers in comparison to the 
regular MR tasks in the job. We might be interested in avoiding trimming the AM log for better diagnostics (by setting
yarn.app.mapreduce.am.container.log.limit.kb=0), or keeping much longer tail of the AM log by using 
yarn.app.mapreduce.am.container.log.backups.

Here is the relevant entries in mapred-default.xml
{% highlight xml %}
<property>
  <name>mapreduce.task.userlog.limit.kb</name>
  <value>0</value>
  <description>The maximum size of user-logs of each task in KB. 0 disables the cap.
  </description>
</property>

<property>
  <name>yarn.app.mapreduce.am.container.log.limit.kb</name>
  <value>0</value>
  <description>The maximum size of the MRAppMaster attempt container logs in KB.
    0 disables the cap.
  </description>
</property>

<property>
  <name>yarn.app.mapreduce.task.container.log.backups</name>
  <value>0</value>
  <description>Number of backup files for task logs when using
    ContainerRollingLogAppender (CRLA). See
    org.apache.log4j.RollingFileAppender.maxBackupIndex. By default,
    ContainerLogAppender (CLA) is used, and container logs are not rolled. CRLA
    is enabled for tasks when both mapreduce.task.userlog.limit.kb and
    yarn.app.mapreduce.task.container.log.backups are greater than zero.
  </description>
</property>

<property>
  <name>yarn.app.mapreduce.am.container.log.backups</name>
  <value>0</value>
  <description>Number of backup files for the ApplicationMaster logs when using
    ContainerRollingLogAppender (CRLA). See
    org.apache.log4j.RollingFileAppender.maxBackupIndex. By default,
    ContainerLogAppender (CLA) is used, and container logs are not rolled. CRLA
    is enabled for the ApplicationMaster when both
    mapreduce.task.userlog.limit.kb and
    yarn.app.mapreduce.am.container.log.backups are greater than zero.
  </description>
</property>
{% endhighlight %}
