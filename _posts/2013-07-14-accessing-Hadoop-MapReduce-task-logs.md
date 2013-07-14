---
layout: post
---

Once debugging of a MapReduce job on a single node (in the local and pseudo-
distributed mode) is over, developers are likely to move to a fully-distributed
development hadoop cluster. It's likely that there are still some things that
need polishing and tweaks until most obvious bugs are sorted out.

In the fully-distributed mode logs of a single node are spread all over the 
cluster nodes in ```${hadoop.log.dir}/userlogs```. So they can't be easily
grepped 
for some debug output from a single machine at once. Even on a single machine,
it's complicated by the fact that multiple task attempts write to the same
 physical log file due to the JVM reuse. Though when the logs are viewed via
 the webUI it's correctly separated by means of the log.index file. MapR
addresses this problem with the feature called
<a href="http://www.mapr.com/doc/display/MapR/Centralized+Logging">Central
Logging</a>. When it is enabled task logs are streamed to MapR-FS instead of to
the local filesystems of individual nodes. The logs are then accessible 
throughout the cluster. It seems expensive but it does not actually involve
any cross-node operations because the log volumes just like MapReduce shuffle 
volumes are local to the node. Once the job has finished, you can set up a
job-centric view on the job log dirs accross the cluster using 
```maprcli job linklogs``` as explained in the documentation. It makes easy to
grep only mappers, reducers, or just tasks on a specific node. It works even 
easier when the cluster is mounted via NFS.

Even with an Apache Hadoop distribution other than MapR there is a tool
called <a href="https://github.com/cbeav/hadoosh">HadooSh</a> (an interactive
Hadoop Shell). HadooSh provides sensible hadoop command completions (file
names, job/task attempt ids). The ```tlog``` command allows grepping
task logs easily in moderate-size clusters. It utilizes the fact that user logs
 are accessible via the TaskTracker-embedded Jetty web server. 
<pre>
# Show all logs for a job:
gera > tlog -job job_201306131712_0004

# Show all mapper logs for a teragen job:
gera > tlog -dir tgen -taskpattern *_m_*

# grep logs for job tasks run on certain nodes
gera > tlog -job job_201306131712_0004 -hostpattern *.rack.company.com | grep needle
</pre>
