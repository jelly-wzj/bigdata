---
layout: post
title:  "hostname(centos7)的几种修改方式"
date:   2021-04-08 08:00
categories: linux
permalink: /archivers/hostname(centos7)的几种修改方式
---

在centos7下，修改hostname的集中方式．

1. hostname命令

   ```shell
   hostname xx
   #临时有效
   #立即生效
   ```


2. hostnamectl命令

   ```shell
   hostnamectl xx
   #永久有效
   #立即生效
   hostnamectl --transient set-hostname xx #临时
   hostnamectl --static set-hostname xx #永久
   ```


3. sysctl kernel.hostname命令

   ```shell
   sysctl kernel.hostname=xx
   #永久有效
   #立即生效
   ```


4. 修改/etc/hostname文件

   ```shell
   echo xx >/etc/hostname
   #重启系统后生效
   #级别最低
   ```


5. 修改/proc/sys/kernel/hostname文件

   ```shell
   echo xx >/proc/sys/kernel/hostname
   #永久有效
   #立即生效
   ```


6. 修改/etc/sysconfig/network文件

   ```shell
   echo HOSTNAME=xx >> /etc/sysconfig/network
   #永久有效
   #重启系统后生效
   ```

   

