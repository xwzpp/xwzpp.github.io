---
layout: post
title:   Tachyon如何解决spark中的数据共享和缓存问题
description: "Tachyon如何解决spark中的数据共享和缓存问题"
category: project
avatarimg: "/img/touxiang.jpg"
tags : [Tachyon,Spark]
duoshuo: true
---
Tachyon是一款基于内存的分布式文件系统，主要致力于解决不同计算框架之间的数据共享问题。

基于内存的特性减少了不同计算框架依赖底层存储系统，如HDFS，带来的磁盘和网络IO问题。而且起初的目标主要是在spark生态系统中，解决不同application之间的RDD共享问题，后来演化成一个独立的项目，解决不同类型应用间数据共享问题。

<!-- more -->

##spark中存在的问题
spark是基于内存的计算框架，在计算方面大大领先mapreduce。但是由于spark是基于jvm，jvm的限制使得spark在数据的共享和缓存方面存在瓶颈。

####数据共享
同一集群中，可能会运行着多个不同的计算框架，比如MapReduce和Spark，在现有条件下，两个框架如果需要共享数据必须通过共享存储实现（HDFS）。而HDFS的访问则会引起磁盘和网络的IO，因此这是一个很低效的过程。

####缓存数据丢失
Spark运行在JVM上，现有的缓存机制使用block manager把数据缓存在JVM堆内存中。一旦执行引擎崩溃，缓存数据丢失。

####GC开销
数据缓存在JVM堆内存空间中，GC开销随应用程序运行时间和堆内存空间中缓存数据量的增长而迅速增长。

而之所以产生这些问题，其根本原因在于缺乏独立的脱离JVM管理的内存管理模块。首先，缺乏独立的内存管理模块，从而造成了数据共享上的问题，这主要是因为现有Spark的内存管理处于执行引擎中，所有的数据管理都在一个application（或者框架）内部，因此脱离了这个application，在内存中缓存的数据就无法被访问。其次，JVM管理，当下所有数据都储存于JVM的堆内存空间中，因此无法避免GC的开销；那么，想避免GC开销，数据必须要脱离JVM。而这些正是Tachyon的设计初衷，一个脱离JVM管理的内存管理模块。

##tachyon的解决方案
针对上述问题，使用tachyon可以很好的进行解决。

####数据共享
通过Tachyon来储存中间结果，避免了数据落到磁盘上，以实现内存数据共享。同时，绕过了HDFS可以减少因此造成的磁盘和网络IO。

####缓存数据丢失
因为所有缓存数据都储存在Tachyon的OffHeap空间中，由Spark任务异常造成的JVM崩溃将不会引发数据丢失。

####GC开销
因为所有缓存数据储存于Tachyon，从而就会解决GC的问题。需要注意的是，Tachyon解决的是缓存数据带来的GC，而非Spark任务执行过程中的GC。

##总结
Tachyon的兴起，带来了基于内存的分布式文件系统，这为不同计算框架的数据共享提供了很好的解决方案，而且基于内存的特性，使得其避免了磁盘和网络IO带来的开销，对计算速度有了一定的保证。除此之外，其脱离jvm的数据缓存功能，减少了jvm中的GC消耗，并根据数据的lineage提供了数据的容错。

