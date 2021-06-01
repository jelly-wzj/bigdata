---
layout: post
title:  "hbase region split的过程"
date:   2021-06-01 08:48
categories: bigdata
permalink: /archivers/bigdata
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

```javascript
<property> 
    <name>hbase.regionserver.region.split.policy</name>  
    <value>org.apache.hadoop.hbase.regionserver.IncreasingToUpperBoundRegionSplitPolicy</value> 
</property>
```

2). 在HBase Configuration中配置

```javascript
private static Configuration conf = HBaseConfiguration.create();
conf.set("hbase.regionserver.region.split.policy", "org.apache.hadoop.hbase.regionserver.SteppingSplitPolicy");
```

3). 在创建表的时候配置 Region的拆分策略需要根据表的属性来合理的配置，所以建议不要使用前两种方法来配置拆分策略，关于在建表的时候怎么配置，会在下面解释每种策略的时候说明。

​	

> 分裂点



> 分裂过程



> 分裂问题