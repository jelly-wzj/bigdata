---
layout: post
title:  "SecondaryNamenode"
date:   2022-05-02 10:14
categories: bigdata
permalink: /archivers/bigdata-hadoop-SecondaryNamenode
---



# 作用

------

在[Hadoop](https://so.csdn.net/so/search?q=Hadoop&spm=1001.2101.3001.7020)中，有一些命名不好的模块，Secondary NameNode是其中之一。从它的名字上看，它给人的感觉就像是NameNode的备份。但它实际上却不是。很多Hadoop的初学者都很疑惑，Secondary NameNode究竟是做什么的，而且它为什么会出现在HDFS中。因此，在这篇文章中，我想要解释下Secondary NameNode在HDFS中所扮演的角色。

从它的名字来看，你可能认为它跟NameNode有点关系。没错，你猜对了。因此在我们深入了解Secondary NameNode之前，我们先来看看NameNode是做什么的。

# NameNode

------

NameNode主要是用来保存[HDFS](https://so.csdn.net/so/search?q=HDFS&spm=1001.2101.3001.7020)的元数据信息，比如命名空间信息，块信息等。当它运行的时候，这些信息是存在内存中的。但是这些信息也可以持久化到磁盘上。

![NameNode保存元数据信息](https://img-blog.csdn.net/20170422101444974?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaXRfZHg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 
上面的这张图片展示了NameNode怎么把元数据保存到磁盘上的。这里有两个不同的文件：

fsimage - 它是在NameNode启动时对整个文件系统的快照 
edit logs - 它是在NameNode启动后，对文件系统的改动序列

只有在NameNode重启时，edit logs才会合并到fsimage文件中，从而得到一个文件系统的最新快照。但是在产品集群中NameNode是很少重启的，这也意味着当NameNode运行了很长时间后，edit logs文件会变得很大。在这种情况下就会出现下面一些问题： 
\- edit logs文件会变的很大，怎么去管理这个文件是一个挑战。 
\- NameNode的重启会花费很长时间，因为在edit log中有很多改动要合并到fsimage文件上。如果NameNode挂掉了，那我们就需要大量时间将edit log与fsimage进行合并。[会将还在内存中但是没有写到edit logs的这部分。] 
因此为了克服这个问题，我们需要一个易于管理的机制来帮助我们减小edit logs文件的大小和得到一个最新的fsimage文件，这样也会减小在NameNode上的压力。这跟Windows的恢复点是非常像的，Windows的恢复点机制允许我们对OS进行快照，这样当系统发生问题时，我们能够回滚到最新的一次恢复点上。

现在我们明白了NameNode的功能和所面临的挑战 - 保持文件系统最新的元数据。那么，这些跟Secondary NameNode又有什么关系呢？

# Secondary NameNode

Secondary NameNode就是来帮助解决上述问题的，它的职责是合并NameNode的edit logs到fsimage文件中。

![这里写图片描述](https://img-blog.csdn.net/20170422102009087?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaXRfZHg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上面的图片展示了Secondary NameNode是怎么工作的。

- 它定时到NameNode去获取edit logs，并更新到自己的fsimage上。
- 一旦它有了新的fsimage文件，它将其拷贝回NameNode中。
- NameNode在下次重启时会使用这个新的fsimage文件，从而减少重启的时间。

Secondary NameNode的整个目的是在HDFS中提供一个检查点。它只是NameNode的一个助手节点。这也是它在社区内被认为是检查点节点的原因。

现在，我们明白了Secondary NameNode所做的不过是在文件系统中设置一个检查点来帮助NameNode更好的工作。它不是要取代掉NameNode也不是NameNode的备份。所以从现在起，让我们养成一个习惯，称呼它为检查点节点吧。

注：关于NameNode是什么时候将改动写到edit logs中的？这个操作实际上是由DataNode的写操作触发的，当我们往DataNode写文件时，DataNode会跟NameNode通信，告诉NameNode什么文件的第几个block放在它那里，NameNode这个时候会将这些元数据信息写到edit logs文件中。

# Secondary NameNode 作用

SecondaryNameNode有两个作用：

1. 镜像备份备份fsimage,(fsimage是元数据发送检查点时写入文件)
2. 日志与镜像的定期合并将Namenode中edits日志和fsimage合并,防止(如果Namenode节点故障，namenode下次启动的时候，会把fsimage加载到内存中，**应用**edit log,edit log往往很大，导致操作往往很耗时。)

# Secondary NameNodeode 工作原理

日志与镜像的定期合并总共分五步：

1. SecondaryNameNode通知NameNode准备提交edits文件，此时主节点产生edits.new。
2. SecondaryNameNode通过http get方式获取NameNode的fsimage与edits文件（在SecondaryNameNode的current同级目录下可见到 temp.check-point或者previous-checkpoint目录，这些目录中存储着从namenode拷贝来的镜像文件）。
3. SecondaryNameNode开始合并获取的上述两个文件，产生一个新的fsimage文件fsimage.ckpt。
4. SecondaryNameNode用http post方式发送fsimage.ckpt至NameNode 
   NameNode将fsimage.ckpt与edits.new文件分别重命名为fsimage与edits，然后更新fstime，整个checkpoint过程到此结束。

SecondaryNameNode备份由三个参数控制fs.checkpoint.period控制周期，fs.checkpoint.size控制日志文件超过多少大小时合并， dfs.http.address表示http地址，这个参数在SecondaryNameNode为单独节点时需要设置。

# 相关配置文件设置

core-site.xml：这里有2个参数可配置，但一般来说我们不做修改。fs.checkpoint.period表示多长时间记录一次hdfs的镜像。默认是1小时。fs.checkpoint.size表示一次记录多大的size，默认64M。

```
<property><name>fs.checkpoint.period</name>
<value>3600</value>
<description>The number of seconds between two periodic checkpoints.
</description>
</property>

<property>
<name>fs.checkpoint.size</name>
<value>67108864</value>
<description>The size of the current edit log (in bytes) that triggers a periodic checkpoint even if the fs.checkpoint.period hasn’t expired.
</description>
</property>123456789101112
```

镜像备份的周期时间是可以修改的，如果不想一个小时备份一次，可以改的时间短点，修改core-site.xml中的fs.checkpoint.period值。

## Import Checkpoint（恢复数据）

如果主节点namenode挂掉了，硬盘数据需要时间恢复或者不能恢复了，现在又想立刻恢复HDFS，这个时候就可以import checkpoint。步骤如下：

准备原来机器一样的机器，包括配置和文件，创建一个空的文件夹，该文件夹就是配置文件中dfs.name.dir所指向的文件夹。拷贝你的secondary NameNode checkpoint出来的文件，到某个文件夹，该文件夹为fs.checkpoint.dir指向的文件夹（如：/home/hadadm/clusterdir/tmp/dfs/namesecondary） 
执行命令bin/hadoop namenode –importCheckpoint这样NameNode会读取checkpoint文件，保存到dfs.name.dir。但是如果你的dfs.name.dir包含合法的 fsimage，是会执行失败的。因为NameNode会检查fs.checkpoint.dir目录下镜像的一致性，但是不会去改动它。

一般建议给maste配置多台机器，让namesecondary与namenode不在同一台机器上值得推荐的是，你要注意备份你的dfs.name.dir和 ${hadoop.tmp.dir}/dfs/namesecondary。