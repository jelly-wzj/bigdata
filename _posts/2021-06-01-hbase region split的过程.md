---
layout: post
title:  "hbase region split的过程"
date:   2021-06-01 08:48
categories: bigdata
permalink: /archivers/bigdata-hbase-region
---



> 分裂策略

region中存储的是一张表的数据，当region中的数据条数过多的时候，会直接影响查询效率.
当region过大的时候，hbase会将region拆分为两个region , 这也是Hbase的一个优点 .
HBase的region split策略一共有以下6种：

* **ConstantSizeRegionSplitPolicy**

  * 0.94版本前，HBase region的默认切分策略

  * 当region中最大的store大小超过某个阈值(hbase.hregion.max.filesize=10G)之后就会触发切分，一个region等分为2个region。

  * 但是在生产线上这种切分策略却有相当大的弊端：

    * 切分策略对于大表和小表没有明显的区分。

    *  阈值(hbase.hregion.max.filesize)设置较大对大表比较友好，但是小表就有可能不会触发分裂，极端情况下可能就1个，形成热点，这对业务来说并不是什么好事。

    * 如果设置较小则对小表友好，但一个大表就会在整个集群产生大量的region，这对于集群的管理、资源使用、failover来说都不是一件好事。

      

* **IncreasingToUpperBoundRegionSplitPolicy**

  * 0.94版本~2.0版本默认切分策略

  * 总体看和ConstantSizeRegionSplitPolicy思路相同

    * 一个region中最大的store大小大于设置阈值就会触发切分。

    * 但是这个阈值并不像ConstantSizeRegionSplitPolicy是一个固定的值，而是会在一定条件下不断调整，调整规则和region所属表在当前regionserver上的region个数有关系.

      

  * region split阈值的计算公式是：

    设regioncount：是region所属表在当前regionserver上的region的个数

    阈值 = regioncount^3 * 128M * 2，当然阈值并不会无限增长，最大不超过MaxRegionFileSize（10G）；当region中最大的store的大小达到该阈值的时候进行region split

    例如：
    第一次split阈值 = 1^3 * 256 = 256MB
    第二次split阈值 = 2^3 * 256 = 2048MB
    第三次split阈值 = 3^3 * 256 = 6912MB
    第四次split阈值 = 4^3 * 256 = 16384MB > 10GB，因此取较小的值10GB
    后面每次split的size都是10GB了

  * 特点

    * 相比ConstantSizeRegionSplitPolicy，可以自适应大表、小表；

    * 在集群规模比较大的情况下，对大表的表现比较优秀

    * 对小表不友好，小表可能产生大量的小region，分散在各regionserver上

      小表达不到多次切分条件，导致每个split都很小，所以分散在各个regionServer上

  

* **SteppingSplitPolicy**
  * 2.0版本默认切分策略
  * 相比 IncreasingToUpperBoundRegionSplitPolicy 简单了一些
  * region切分的阈值依然和待分裂region所属表在当前regionserver上的region个数有关系
    * 如果region个数等于1，切分阈值为flush size :128M * 2
    * 否则为MaxRegionFileSize。
  * 这种切分策略对于大集群中的大表、小表会比 IncreasingToUpperBoundRegionSplitPolicy 更加友好，小表不会再产生大量的小region，而是适可而止。



* **KeyPrefixRegionSplitPolicy**

  根据rowKey的前缀对数据进行分区，这里是指定rowKey的前多少位作为前缀，比如rowKey都是16位的，指定前5位是前缀，那么前5位相同的rowKey在相同的region中。



* **DelimitedKeyPrefixRegionSplitPolicy**

  保证相同前缀的数据在同一个region中，例如rowKey的格式为：userid_eventtype_eventid，指定的delimiter为 _ ，则split的的时候会确保userid相同的数据在同一个region中。
  按照分隔符进行切分，而KeyPrefixRegionSplitPolicy是按照指定位数切分。



* **DisabledRegionSplitPolicy**

  不启用自动拆分, 需要指定手动拆分



------

#### 使用方式

1). 在hbase-site.xml中配置，例如：

```java
<property> 
    <name>hbase.regionserver.region.split.policy</name>  
    <value>org.apache.hadoop.hbase.regionserver.IncreasingToUpperBoundRegionSplitPolicy</value> 
</property>
```

2). 在HBase Configuration中配置

```java
private static Configuration conf = HBaseConfiguration.create();
conf.set("hbase.regionserver.region.split.policy", "org.apache.hadoop.hbase.regionserver.SteppingSplitPolicy");
```

3). 在创建表的时候配置 Region的拆分策略需要根据表的属性来合理的配置，所以建议不要使用前两种方法来配置拆分策略，关于在建表的时候怎么配置，会在下面解释每种策略的时候说明。

```javascript
HTableDescriptor tableDesc = new HTableDescriptor(TableName.valueOf("tableName"));
tableDesc.setRegionSplitPolicyClassName("org.apache.hadoop.hbase.regionserver.ConstantSizeRegionSplitPolicy");

create ’table’, {NAME => ‘cf’, SPLIT_POLICY => ‘org.apache.hadoop.hbase.regionserver. ConstantSizeRegionSplitPolicy'}
```

​	

> 分裂点

​	整个Region中最大store中的最大文件中最中心的一个block的首个rowkey - 分裂点。如果rowkey是整个文件的首个或者最后一个rowkey的话，则不存在分裂点。例如整个表就只存在一个block的时候，就不存在分裂点。



> 分裂过程

1. 准备阶段

   在内存中初始化两个子region，具体是生成两个HRegionInfo对象，包含tableName,regionName,startKey,endKey等。同时会生成一个transaction journal，这个对象记录分裂的过程。

   

2. 执行阶段

   - 1、RegionServer决定本地的region分裂，并准备分裂工作。第一步是，在zookeeper的/hbase/region-in-reansition/region-name下创建一个znode，并设为SPLITTING状态。
   - 2、Master通过父region-in-transition znode的watcher监测到刚刚创建的znode。
   - 3、RegionServer在HDFS中父region的目录下创建名为“.split”的子目录。
   - 4、RegionServer关闭父region，并强制刷新缓存内的数据，之后在本地数据结构中将标识为下线状态。此时来自Client的对父region的请求会抛出NotServingRegionException ，Client将重新尝试向其他的region发送请求。
   - 5、RegionServer在.split目录下为子regionA和B创建目录和相关的数据结构。然后RegionServer分割store文件，这种分割是指，为父region的每个store文件创建两个Reference文件。这些Reference文件将指向父region中的文件。
   - 6、RegionServer在HDFS中创建实际的region目录，并移动每个子region的Reference文件。
   - 7、RegionServer向.META.表发送Put请求，并在.META.中将父region改为下线状态，添加子region的信息。此时表中并单独存储没有子region信息的条目。Client扫描.META.时回看到父region为分裂状态，但直到子region信息出现在表中，Client才直到他们的存在。如果Put请求成功，那么父region将被有效地分割。如果在这条RPC成功之前RegionServer死掉了，那么Master和打开region的下一个RegionServer会清理关于该region分裂的脏状态。在.META.更新之后，region的分裂将被Master回滚到之前的状态。
   - 8、RegionServer打开子region，并行地接受写请求。
   - 9、RegionServer将子region A和B的相关信息写入.META.。此后，Client便可以扫描到新的region，并且可以向其发送请求。Client会在本地缓存.META.的条目，但当她们向RegionServer或.META.发送请求时，这些缓存便无效了，他们竟重新学习.META.中新region的信息。
   - 10、RegionServer将zookeeper中的znode /hbase/region-in-transition/region-name更改为SPLIT状态，以便Master可以监测到。如果子Region被选中了，Balancer可以自由地将子region分派到其他RegionServer上。
   - 11、分裂之后，元数据和HDFS中依然包含着指向父region的Reference文件。这些Reference文件将在子region发生紧缩操作重写数据文件时被删除掉。Master的垃圾回收工会周期性地检测是否还有指向父region的Reference，如果没有，将删除父region。

   

3. 回滚阶段

   如果execute阶段出现异常，则执行rollback操作。为了实现回滚，整个切分过程被分为很多子阶段，回滚程序会根据当前进展到哪个子阶段清理对应的垃圾数据。代码中使用 JournalEntryType 来表征各个子阶段，具体见下图：![201809101204082b2a2244-f7f7-49d5-9111-1727ed83e04c.png](https://nos.netease.com/cloud-website-bucket/201809101204082b2a2244-f7f7-49d5-9111-1727ed83e04c.png)



> 分裂问题

通过region切分流程的了解，我们知道整个region切分过程并没有涉及数据的移动，所以切分成本本身并不是很高，可以很快完成。切分后子region的文件实际没有任何用户数据，文件中存储的仅是一些元数据信息－切分点rowkey等，那通过引用文件如何查找数据呢？子region的数据实际在什么时候完成真正迁移？数据迁移完成之后父region什么时候会被删掉？

1. 通过reference文件如何查找数据？
这里就会看到reference文件名、文件内容的实际意义啦。整个流程如下图所示：

  ![201809101204244637cbc7-9978-47e5-8984-4328ddba8456.png](https://nos.netease.com/cloud-website-bucket/201809101204244637cbc7-9978-47e5-8984-4328ddba8456.png)

（1）根据reference文件名（region名+真实文件名）定位到真实数据所在文件路径

（2）定位到真实数据文件就可以在整个文件中扫描待查KV了么？非也。因为reference文件通常都只引用了数据文件的一半数据，以切分点为界，要么上半部分文件数据，要么下半部分数据。那到底哪部分数据？切分点又是哪个点？还记得上文又提到reference文件的文件内容吧，没错，就记录在文件中。

2. 父region的数据什么时候会迁移到子region目录？
答案是子region发生major_compaction时。我们知道compaction的执行实际上是将store中所有小文件一个KV一个KV从小到大读出来之后再顺序写入一个大文件，完成之后再将小文件删掉，因此compaction本身就需要读取并写入大量数据。子region执行major_compaction后会将父目录中属于该子region的所有数据读出来并写入子region目录数据文件中。可见将数据迁移放到compaction这个阶段来做，是一件顺便的事。

3. 父region什么时候会被删除？
实际上HMaster会启动一个线程定期遍历检查所有处于splitting状态的父region，确定检查父region是否可以被清理。检测线程首先会在meta表中揪出所有split列为true的region，并加载出其分裂后生成的两个子region（meta表中splitA列和splitB列），只需要检查此两个子region是否还存在引用文件，如果都不存在引用文件就可以认为该父region对应的文件可以被删除。现在再来看看上文中父目录在meta表中的信息，就大概可以理解为什么会存储这些信息了：

4. split模块在生产线的一些坑？
  有些时候会有同学反馈说集群中部分region处于长时间RIT，region状态为spliting。通常情况下都会建议使用hbck看下什么报错，然后再根据hbck提供的一些工具进行修复，hbck提供了部分命令对处于split状态的rit region进行修复，主要的命令如下：

  ```
    -fixSplitParents  Try to force offline split parents to be online.
    -removeParents    Try to offline and sideline lingering parents and keep daughter regions.
    -fixReferenceFiles  Try to offline lingering reference store files
  ```

  

  其中最常见的问题是 ：

  ```
  ERROR: Found lingering reference file hdfs://mycluster/hbase/news_user_actions/3b3ae24c65fc5094bc2acfebaa7a56de/meta/0f47cda55fa44cf9aa2599079894aed6.b7b3faab86527b88a92f2a248a54d3dc”
  ```

  

  简单解释一下，这个错误是说reference文件所引用的父region文件不存在了，如果查看日志的话有可能看到如下异常：

  ```
  java.io.IOException: java.io.IOException: java.io.FileNotFoundException: File does not exist:/hbase/news_user_actions/b7b3faab86527b88a92f2a248a54d3dc/meta/0f47cda55fa44cf9aa2599079894aed
  ```

  父region文件为什么会莫名其妙不存在？经过和朋友的讨论，确认有可能是因为官方bug导致，详见HBASE-13331。这个jira是说HMaster在确认父目录是否可以被删除时，如果检查引用文件（检查是否存在、检查是否可以正常打开）抛出IOException异常，函数就会返回没有引用文件，导致父region被删掉。正常情况下应该保险起见返回存在引用文件，保留父region，并打印日志手工介入查看。如果大家也遇到类似的问题，可以看看这个问题，也可以将修复patch打到线上版本或者升级版本。
 