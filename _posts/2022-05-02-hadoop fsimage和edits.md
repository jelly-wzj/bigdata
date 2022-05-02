---
layout: post
title:  "hadoop fsimage和edits"
date:   2022-05-01 10:06
categories: bigdata
permalink: /archivers/bigdata-fsedit
---



**Fsimage：镜像文件**
**Edits：编辑日志**

首先，当集群format之后，将在目录(/opt/hadoop/tmp/dfs/name/current)下产生如下内容：

![](https://img-blog.csdnimg.cn/20200629201345918.png)


（1）Fsimage文件：HDFS文件系统元数据的一个永久性的检查点，其中包含HDFS文件系统的所有目录和文件inode的序列化信息（id、类型、目录、所属用户、用户权限、时间戳……）。
（2）Edits文件：存放HDFS文件系统的所有更新操作的路径，文件系统客户端执行的所有写操作首先会被记录到edits文件中。
（3）seen_txid文件保存的是一个数字，就是最后一个edits_的数字
（4）每次Namenode启动的时候都会将fsimage文件读入内存，并从00001开始到seen_txid中记录的数字依次执行每个edits里面的更新操作，保证内存中的元数据信息是最新的、同步的，可以看成Namenode启动的时候就将fsimage和edits文件进行了合并。
下面我们来举个示例：
1、在hdfs上创建文件夹并上传一个文件

```shell
[root@hadoop101 /]# hadoop fs -mkdir -p /user/input
[root@hadoop101 /]# hadoop fs -put a.txt /user/input/
```


2、接着我们看一下生成了哪些fsimage和edits

![](https://img-blog.csdnimg.cn/20200629201350819.png)

- edits为1-3,其中edits_inprogress001-002为滚动后，给2NN合并的，edits_inprogress_003用来接受客户端的更新请求;当前生成了两个fsimage

- 此时我们在2nn节点下查看文件，可以发现edits_inprogress001-002是用来和fsimage合并的，并且已经拷贝到了2nn节点下

  ![](https://img-blog.csdnimg.cn/20200629201355627.png)

**oev查看Edits文件**

基本语法：

```shell
hdfs oev -p 文件类型 -i编辑日志名称 -o 转换后文件输出路径
```

通过以下命令来查看edits：

```shell
hdfs oev -p XML -i edits_0000000000000000001-0000000000000000002 -o e.XML
```

下面来简单看一下e.xml中的内容:
1、先创建/user文件夹

![](https://img-blog.csdnimg.cn/20200629201403946.png)

2、再创建user下的input文件夹

![](https://img-blog.csdnimg.cn/20200629201406448.png)

3、先增加一个a.txt._copying,而不是直接创建a.txt

![](https://img-blog.csdnimg.cn/20200629201411880.png)


4、分配块id

…

最后进行改名，变成了a.txt

![](https://img-blog.csdnimg.cn/20200629201415259.png)


**oev查看fsimage文件**
基本语法：

```shell
hdfs oiv -p 文件类型 -i编辑日志名称 -o 转换后文件输出路径
```

通过以下命令来查看fsimage：

```shell
hdfs oiv -p XML -i fsimage_0000000000000000000 -o fs.XML
```

![](https://img-blog.csdnimg.cn/20200629201418391.png)

**大致存储的就是文件夹和文件名称**
最后，我们可以看一下seen_txid中的内容：

seen_txid存放的就是edits的最新一个数字，即

![](https://img-blog.csdnimg.cn/20200629201425526.png)


所以每次nn启动的时候，都会从edits_001开始与fsimage，一直到seen_txid存放的edits数字结束