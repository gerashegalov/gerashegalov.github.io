---
layout: post
title: How to get a heapdump of a task getting OutOfMemoryError
---

One can enable an Hprof profiler `mapreduce.task.profile.params`. [Hprof][1] 
supports heap profiles. If the tasks throw OOM less deterministically, that 
method might not yield the desired result while affecting performance and 
creating large profile files for nothing.

Getting a heap dump only on OOM
--
On Hadoop 2, it's relatively easy to generate a heap dump into the container
log directory such that it even will be aggregated to HDFS with all other logs.
Add to the child jvm opts `mapreduce.map.java.opts` the following:
`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=<LOG_DIR>/@taskid@.hprof`
The macro <LOG_DIR> will be expanded to the container log directory that 
usually contains standard logs such as syslog, stdout/err. `@taskid@` will be 
the task attempt id.

The painful part is extracting the heap dump log. Once gets the logs for a container using:

{% highlight bash %}
$ yarn logs -applicationId application_1391047358335_0041 -containerId container_1391047358335_0041_01_000002 -nodeAddress nmhost:8041 > all.task.logs
{% endhighlight %}
 
The App id can be found either in the jobclient output or on the RM page. 
Then one navigates via WebUI to the OOM'd task attempt on the history server. 
When one click on `logs`, the URL will contain the node address and the container id such as 
http://rm:8080/jobhistory/logs/container_1391047358335_0041_01_000002/attempt_1391047358335_0041_m_000000_0/user

In all.task.logs one will need to cut the file such that it contains only the 
lines from JAVA PROFILE until and excluding the subsequent "LogType:" line. Theni
one can load it in heap dump analyzer of your choice.

[1]: http://docs.oracle.com/javase/7/docs/technotes/samples/hprof.html
