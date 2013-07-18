---
layout: post
---

First we need to create a Hive tar to be deployed on Windows. This can be done
using ```ant tar```, either on Linux or Windows using cygwin. Note, however,
hive does not run on Cygwin, it requires a Windows-native cmd terminal.
Vanilla hive does not contain cmd scripts yet which is subject of
<a href="https://issues.apache.org/jira/browse/HIVE-3129">HIVE-3129</a>. 
A modified version is incorporated in
<a href="https://github.com/gerashegalov/hive/tree/branch-0.11-mapr">
this fork of branch-0.11-mapr</a>. So youn can either extract and patch the
 structure from a Linux Hive package or build Hive from scratch.

{% highlight bash %}
$ git clone https://github.com/gerashegalov/hive.git
$ git checkout branch-0.11-mapr
$ ant tar
{% endhighlight %}


The tar is located under ```build/hive-0.11-mapr.tar.gz```. We need to create
```MAPR_HOME``` directory ```C:\opt\mapr``` and extract the content into 
```%MAPR_HOME%\hive```

If not already done, extract Windows MapR client into ```%MAPR_HOME``` and 
configure it according to the
<a href="http://www.mapr.com/doc/display/MapR/Setting+Up+the+Client#SettingUptheClient-windowsclient">doc</a>.

Next obstacle is that the shipped ```hadoop.bat``` does not accept 
the ```HADOOP_CLASSPATH``` passed by hive and uses explicit list of jars which '
may result in exceeding the input line size limit of the cmd processor.
The necessary change is summararized in the following unified diff:
{% highlight diff %}
diff -r -u /cygdrive/c/opt/mapr-client-2.1.3.19871GA-1.amd64/hadoop/hadoop-0.20.2/bin/hadoop.bat /cygdrive/c/opt/mapr/hadoop/hadoop-0.20.2/bin/hadoop.bat
--- /cygdrive/c/opt/mapr-client-2.1.3.19871GA-1.amd64/hadoop/hadoop-0.20.2/bin/hadoop.bat       2013-05-08 18:01:22.000000000 -0700
+++ /cygdrive/c/opt/mapr/hadoop/hadoop-0.20.2/bin/hadoop.bat    2013-07-17 03:43:57.964723200 -0700
@@ -98,12 +98,12 @@

 for %%i in ("%HADOOP_HOME%\hadoop-*-core.jar") do set CLASSPATH=!CLASSPATH!;"%%i"

-for %%i in ("%HADOOP_HOME%\lib\*.jar") do set CLASSPATH=!CLASSPATH!;"%%i"
+set CLASSPATH=!CLASSPATH!;%HADOOP_HOME%\lib\*

-if exist "%HADOOP_HOME%\build\ivy\lib\Hadoop\common" for %%i in ("%HADOOP_HOME%\build\ivy\lib\Hadoop\common\*.jar") do set CLASSPATH=!CLASSPATH!;"%%i"
-
-for %%i in ("%HADOOP_HOME%\lib\jsp-2.1\*.jar") do set CLASSPATH=!CLASSPATH!;"%%i"
+if exist "%HADOOP_HOME%\build\ivy\lib\Hadoop\common" set CLASSPATH=!CLASSPATH!;%HADOOP_HOME%\build\ivy\lib\Hadoop\common\*

+set CLASSPATH=!CLASSPATH!;%HADOOP_HOME%\lib\jsp-2.1\*
+set CLASSPATH=!CLASSPATH!;%HADOOP_CLASSPATH%
 set TOOL_PATH=
 for %%i in ("%HADOOP_HOME%\hadoop-*-tools*.jar") do set TOOL_PATH=!TOOL_PATH!;%%i
 for %%i in ("%HADOOP_HOME%\build\hadoop-*-tools*.jar") do set TOOL_PATH=!TOOL_PATH!;%%i
{% endhighlight %}

Now if we try to run hive cli, we run into the following exception:
<pre>
C:\> C:\opt\mapr\hive\hive-0.11\bin\hive
...
Illegal Hadoop Version: Unknown (expected A.B.* format)
</pre>

Indeed if we execute ```hadoop version```, we see the problem right away:
<pre>
C:\> C:\opt\mapr\hadoop\hadoop-0.20.2\bin\hadoop version
Hadoop Unknown
Source Unknown -r Unknown
Compiled by Unknown on Unknown
From source with checksum Unknownc:\cygwin\home\gshegalo\dev\hive\build\dist>bin\hive
</pre>

This means that hadoop-core jar in Windows build lacks the version info class
that Hive depends on. Luckily this is not the case with the jar provided in
Linux builds. So we can replace the Windows jar ```C:\opt\mapr\hadoop\hadoop-0.20.2\lib\hadoop-0.20.2-dev-core.jar``` with
<a href="http://repository.mapr.com/nexus/content/groups/mapr-public/org/apache/hadoop/hadoop-core/1.0.3-mapr-2.1.3.1/hadoop-core-1.0.3-mapr-2.1.3.1.jar">a correct copy</a> in MapR's maven repo.

After this last step, Hive works fine even on Windows:

<pre>
C:\> C:\opt\mapr\hive\hive-0.11\bin\hive
...
hive> dfs -ls;
Found 1 items
drwxrwxr-t   - root root          1 2013-05-19 12:34 /user/root/wcout
hive> select count(*) from g;
Total MapReduce jobs = 1
Launching Job 1 out of 1
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapred.reduce.tasks=<number>
Starting Job = job_201307170100_0002, Tracking URL = http://gera-VirtualBox:50030/jobdetails.jsp?job
id=job_201307170100_0002
Kill Command = c:\opt\mapr\hadoop\hadoop-0.20.2\bin\hadoop.cmd job  -kill job_201307170100_0002
Hadoop job information for Stage-1: number of mappers: 0; number of reducers: 1
2013-07-17 05:19:47,237 Stage-1 map = 0%,  reduce = 0%
2013-07-17 05:20:48,076 Stage-1 map = 0%,  reduce = 0%
2013-07-17 05:21:48,898 Stage-1 map = 0%,  reduce = 0%
2013-07-17 05:21:51,932 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 0.64 sec
MapReduce Total cumulative CPU time: 640 msec
Ended Job = job_201307170100_0002
MapReduce Jobs Launched:
Job 0: Reduce: 1   Cumulative CPU: 0.64 sec   MAPRFS Read: 0 MAPRFS Write: 2 SUCCESS
Total MapReduce CPU Time Spent: 640 msec
OK
0
Time taken: 131.65 seconds, Fetched: 1 row(s)
</pre>
