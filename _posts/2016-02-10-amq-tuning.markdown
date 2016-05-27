---
layout: post
title: JBoss A-MQ 6.2 Performance Tuning Adventures 
date: 2016-02-10
comments: True
categories:
---

2008 was the last time I embarked on a similar exercise, so I was a little rusty at tuning high-performance applications. But it was refreshing to revisit the process again with JBoss A-MQ.  I recently had a customer who presented a performance issue with an unusual use case: "Publish only to a composite topic (forwarding from topics to queues) with no subscribers, no use of transactions and must have JMS durability“.  I quizzed the customer whether this was a valid real-word use case and they assured me it was.  They explained they have many applications publishing notification messages to topics which may (or may not) have any subscribers, and it was likely the topic could get backed up with millions of messages at a time.  Their initial performance benchmark looked like this:

#### <center>Pub only versus Pub/Sub, zero tuning, persistence, 13M messages<center> ####

|------+--------|
| Test | Result |
|:------:|:------:|
|Pub only | 1276 msgs/sec|
|Pub / Sub | 8832 msgs/sec|
|------+--------|

Pathetic and embarrassing to say the least.  But hey - zero tuning so you can't really expect much right?  The first question we raised was let's take a look at your hardware:

#### <center>Test Broker VM Specification</center> ####

|------+--------|
| Item | Value |
|------|------:|
| OS  | RHEL 7.1|
| Architecture | amd64|
| Processors | 4|
| Memory | 8GB|
| JVM | Oracle 64bit 1.7.0_09|
|------+--------|

Not too shabby in terms of test machine, so what about storage:

#### <center>Test Broker Storage</center> ####

|------+--------|
| Item | Value |
|------|------:|
| Type | HDS VSP array|
| Connectivity | Fiber attached, 8Gbp/s HBAs & SAN switches|
| NFS | No|
|------+--------|

Again, no alarm bells yet in terms of test VM and storage.  My colleague, Christian Posta, who published an amazing post on tuning A-MQ [here](http://blog.christianposta.com/activemq/speeding-up-activemq-persistent-messaging-performance-by-25x/), gave us some hints for checking persistent storage, specifically by running the activemq perf-test benchmarking tool.  This was the output it gave:

~~~ Text
Disk Benchmark:
Benchmarking: /opt/app/jbossamq/csst/data/amq/kahadb/disk-benchmark.dat
Writes:
  1919986 writes of size 4096 written in 10.042 seconds.
  191195.58 writes/second.
  746.8577 megs/second.

Sync Writes:
  24857 writes of size 4096 written in 10.001 seconds.
  2485.4514 writes/second.
  9.708795 megs/second.

Reads:
  8539871 reads of size 4096 read in 10.001 seconds.
  853901.7 writes/second.
  3335.5535 megs/second
~~~

The Disk Writes looked fine at 746 megs/sec.  But as Christian mentioned in his post, the item to watch was the sync writes, which in our case were abysmally slow at 9.7 megs/sec.  Christian's suggestion was to run disk-benchmark again but increase the block size to 4MB (the default block size of Apache ActiveMQ):

~~~ Text
Benchmarking: /opt/app/jbossamq/csst/extras/apache-activemq-5.11.0.redhat-62013/disk-benchmark.dat
Writes:
  1576 writes of size 4194304 written in 10.251 seconds.
  153.7411 writes/second.
  614.9644 megs/second.

Sync Writes:
  927 writes of size 4194304 written in 10.004 seconds.
  92.66293 writes/second.
  370.65173 megs/second.

Reads:
  3858 reads of size 4194304 read in 10.003 seconds.
  385.6843 writes/second.
  1542.7372 megs/second.
~~~

This time around we saw a massive increase from 9.7 megs/second to 370 megs/second, telling us to be mindful of this when tuning JBoss A-MQ.  We had a smoking gun though: storage.  But how best to proceed?  We decided to momentarily take storage out of the equation and run the tests with zero persistence, using the “memoryPersistenceAdapter” plus publish to queues (and not composite topics).  That way, everything runs in memory and we’re not worrying about disk I/O or storage holding us up.  We tried running the same tests but this time with a 4M message sample.  Unfortunately, the test broker kept bombing out with this ghastly error: 

~~~ Java
java.lang.OutOfMemoryError: GC overhead limit exceeded
~~~
A bit of Googling and we soon found out we can fix this by specifying a different Garbage Collector in the bin/setenv file:

~~~ Text
export JAVA_MIN_MEM=6G # Minimum memory for the JVM
export JAVA_MAX_MEM=6G # Maximum memory for the JVM
export KARAF_OPTS="-XX:+UseConcMarkSweepGC"
~~~

While we were at it, we noticed the customer had set the JVM min/max heap size to 8G - the total memory size of the VM which certainly wasn’t helping matters.  So we knocked that down to 6G (both min and max) giving the OS a bit of breathing space (2G of memory).  We ran the tests again and this was the new result:

#### <center>Pub only versus Pub/Sub, No persistence, New memory / GC settings<center> ####

|------+--------|
| Test | Result |
|------|------:|
| Pub only | 57,743 msgs/sec |
| Pub / Sub | 34,198 msgs/sec |
|------+--------|

Wow! What a MASSIVE difference adjusting the memory, GC and removing storage makes. We’ve opened the flood gates, so now was a good time to add back persistence (kahadb) and re-test.  Instead of going through every modification, here is a summary of tweaks we made to the default persistence store, kahadb (in the activemq.xml file):


### <center>Queues, Producer Only, Persistence On</center> ###

|------+--------|
| Enhancement | Throughput (msg/sec) |
|------|------:|
| UseConcMarkSweepGC | 1,236 |
| enableJournalDiskSyncs="true"<br/>preallocationStrategy="zeros" | 1,424 |
| preallocationStrategy="os_kernel_copy" | 1,456 |
| preallocationStrategy="zeros"<br/>concurrentStoreAndDispatchQueues="false" | 6,056 |
| enableJournalDiskSyncs="true"<br/>preallocationStrategy="zeros"<br/> concurrentStoreAndDispatchQueues="false" <br/> cleanupInterval="300000" <br/> checkpointInterval="50000" <br/> journalMaxWriteBatchSize="62k" <br/> journalMaxFileLength="1g" <br/> indexCacheSize="100000" | 5,688 |
| enableJournalDiskSyncs="true"<br/>preallocationStrategy="zeros"<br/>concurrentStoreAndDispatchQueues="false"<br/>skipMetadataUpdate=true | 7,828 |
| enableJournalDiskSyncs="true"<br/>preallocationStrategy="zeros"<br/>concurrentStoreAndDispatchQueues="false"<br/>mKahaDb (10 instances, cold database) | 11,936 |
| enableJournalDiskSyncs="true"<br/>preallocationStrategy="zeros"<br/>concurrentStoreAndDispatchQueues="false"<br/>mKahaDb (10 instances, hot database) | 12,693 |
|------+--------|

A technique suggested to us by Red Hat Engineering was the use of destination sharding using mKahaDB (or multiple kahadb’s).  The idea is to segregate topic / queue destinations into their own kahadb instance make access quicker, which obviously gave us some performance gain.  

~~~ XML
 <persistenceAdapter>
	<mKahaDB directory="${data}/kahadb">
          <filteredPersistenceAdapters>
            <filteredKahaDB queue="amqThroughPutTest.1">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.2">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.3">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.4">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.5">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.6">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.7">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.8">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.9">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
            <filteredKahaDB queue="amqThroughPutTest.10">
        <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
        </persistenceAdapter>
       </filteredKahaDB>
       <filteredKahaDB>
          <persistenceAdapter>
            <kahaDB enableJournalDiskSyncs="true" preallocationStrategy="zeros" concurrentStoreAndDispatchQueues="false"/>
          </persistenceAdapter>
       </filteredKahaDB>
    </filteredPersistenceAdapters>
   </mKahaDB>
 </persistenceAdapter> 
~~~

At this point the customer was getting pretty excited at seeing a 10x performance gain, but their expectation was 17,000-19,000 msgs/sec.  They asked whether we had anymore tricks up our sleeves, which we did of course.  I told the customer to add the following line to their etc/system.properties file:

~~~ Java
org.apache.activemq.kahaDB.files.skipMetadataUpdate=true
~~~

On some operating systems, you can obtain better performance by enabling skipMetadataUpdate to call the fdatasync() system call instead of the fsync() system call (the default), when writing to a file. The difference between these system calls is that fdatasync() updates only the file data, whereas fsync() updates both the file data and the file metadata (for example, the access time).  Keep in mind that this setting does not work on all Operating Systems or JVM’s, thus it’s not a default setting.

The customer retested this configuration using mKahadb destination sharding but reverting back to composite topics instead of queues.  This was the final result:


#### <center>Pub only versus Pub/Sub, Composite Topics, Tuned, 4M messages<center> ####

|------+--------|
| Test | Result |
|------|------:|
| Pub only | 12,197 msgs/sec |
| Pub / Sub | 16,783 msgs/sec |
|------+--------|

Not quite “high teens” but close enough.  After further analysis by Red Hat Engineering, we noticed that there were fsync bottlenecks occurring which could be optimized through a code change, so a fix was released to the community version of Apache ActiveMQ [here](https://issues.apache.org/jira/browse/AMQ-6164).  We’re all eagerly waiting for this fix to be included in a future release of JBoss A-MQ, so stay tuned to see whether we can finally hit our performance target of “high teens” once and for all.

