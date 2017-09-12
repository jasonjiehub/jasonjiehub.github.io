---
layout:     post
title:      我是如何解决jobtracker.info could only be replicated to 0 nodes, instead of 1这个问题的
subtitle:   
date:       2017-09-08
author:  Jason
header-img: 
catalog: true
tags:
    - hadoop
    - Java
---


我按照慕课网上的教程来学习搭建hadoop-1.2.1环境，可是在start-all.sh这步的时候一直通不过，命令行报如下错误
```
[@soguo /home/denglinjie/hadoop]# start-all.sh   
Warning: $HADOOP_HOME is deprecated.  
  
starting namenode, logging to /home/denglinjie/hadoop-1.2.1/logs/hadoop-denglinjie-namenode-bjzw_30_4.out  
localhost: ssh_exchange_identification: Connection closed by remote host  
localhost: ssh_exchange_identification: Connection closed by remote host  
starting jobtracker, logging to /home/denglinjie/hadoop-1.2.1/logs/hadoop-denglinjie-jobtracker-bjzw_30_4.out  
localhost: ssh_exchange_identification: Connection closed by remote host 
```
日志文件报如下错误：
```
2017-03-02 19:28:00,951 WARN org.apache.hadoop.hdfs.DFSClient: Error Recovery for null bad datanode[0] nodes == null  
2017-03-02 19:28:00,952 WARN org.apache.hadoop.hdfs.DFSClient: Could not get block locations. Source file "/home/denglinjie/hadoop/tmp/mapred/system/jobtracker.info" - Aborting  
...  
2017-03-02 19:28:00,952 WARN org.apache.hadoop.mapred.JobTracker: Writing to file hdfs://localhost:9063/home/denglinjie/hadoop/tmp/mapred/system/jobtracker.info failed!  
2017-03-02 19:28:00,952 WARN org.apache.hadoop.mapred.JobTracker: FileSystem is not ready yet!  
2017-03-02 19:28:00,954 WARN org.apache.hadoop.mapred.JobTracker: Failed to initialize recovery manager.   
org.apache.hadoop.ipc.RemoteException: java.io.IOException: File /home/denglinjie/hadoop/tmp/mapred/system/jobtracker.info could only be replicated to 0 nodes, instead of 1  
    at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getAdditionalBlock(FSNamesystem.java:1920)  
    at org.apache.hadoop.hdfs.server.namenode.NameNode.addBlock(NameNode.java:783)  
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)  
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)  
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)  
    at java.lang.reflect.Method.invoke(Method.java:597)  
    at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:587)  
    at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:1432)  
    at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:1428)  
    at java.security.AccessController.doPrivileged(Native Method)  
    at javax.security.auth.Subject.doAs(Subject.java:396)  
    at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1190)  
    at org.apache.hadoop.ipc.Server$Handler.run(Server.java:1426)  
  
    at org.apache.hadoop.ipc.Client.call(Client.java:1113)  
    at org.apache.hadoop.ipc.RPC$Invoker.invoke(RPC.java:229)  
    at com.sun.proxy.$Proxy7.addBlock(Unknown Source)  
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)  
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)  
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)  
    at java.lang.reflect.Method.invoke(Method.java:597)  
    at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:85)  
    at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:62)  
    at com.sun.proxy.$Proxy7.addBlock(Unknown Source)  
    at org.apache.hadoop.hdfs.DFSClient$DFSOutputStream.locateFollowingBlock(DFSClient.java:3720)  
    at org.apache.hadoop.hdfs.DFSClient$DFSOutputStream.nextBlockOutputStream(DFSClient.java:3580)  
    at org.apache.hadoop.hdfs.DFSClient$DFSOutputStream.access$2600(DFSClient.java:2783)  
    at org.apache.hadoop.hdfs.DFSClient$DFSOutputStream$DataStreamer.run(DFSClient.java:3023)  
```
首先解决命令行报的错误：
```
localhost: ssh_exchange_identification: Connection closed by remote host  
```
从google中找问题，发现网上的解决办法千奇百怪，有的说要修改slaves和masters文件中的主机名为ip地址，有的说是因为主机名不能有下划线，但是其实我的主机名用的默认的localhost，总之尝试了网上的各种解决办法，都失败，网上大多数办法都是集中在如下几个配置文件配置的问题上：
core-site.xml,  mapred-site.xml,  hdfs-site.xml
不过无论我怎么配置都不行，于是一下午都没有找到原因
第二天来之后，我想了下也许慕课网那个视频教程是精简版的呢，于是我索性自己从google上搜索了一篇hadoop-1.2.1搭建本地伪分布式安装的教程，按照别人的教程来，文章地址如下：
https://hexo2hexo.github.io/hadoop1.2.1%E5%AE%89%E8%A3%85/
这篇文章中的步骤其实和慕课网的视频教程步骤基本雷同，但是多了如下步骤：
```
1.生成秘钥  
ssh-keygen -t rsa  
2.一直回车即可,然后进入.ssh目录,执行命令  
cd ~/.ssh  
cp id_rsa.pub authorized_keys  
3.检查是否需要密码  
ssh localhost 
```
这很显然是设置免密码登陆啊，于是我按照它的步骤操作了一遍，发现执行到ssh localhost这步的时候，报了如下错误：
```
ssh_exchange_identification: Connection closed by remote host 
```
我一看，这步跟我启动hadoop的时候报的错误是一样的吗？感情启动hadoop的时候就是在执行ssh  localhost这步呢，于是原因也找到了，
```
jobtracker.info could only be replicated to 0 nodes, instead of 1 
```
上面这个应该只是ssh执行失败的结果，而不是造成问题的主要原因，既然这样那只要保证ssh  localhost成功登陆本地主机不就OK了吗
于是又在网上一通找，

首先开启了/etc/ssh/sshd_config配置文件中的如下几个选项
```
RSAAuthentication yes  
PubkeyAuthentication yes  
AuthorizedKeysFile  .ssh/authorized_key 
```
然后重启ssh服务
# service  sshd   restart
发现不好使，还是登陆不上，然后又把/etc/hosts.deny文件中的
# sshd  ALL
这行注释打开，重启sshd服务，再次登陆成功了！
于是迫不及待的重新执行
```
# stop-all.sh  
# hadoop namenode -format  
# start-all.sh
```
成功了！！！
执行jps命令，成功看到了如下输出：
```
[@sohuo ~/hadoop-1.2.1]$ jps  
30999 Jps  
30510 DataNode  
30651 SecondaryNameNode  
30885 TaskTracker  
30395 NameNode  
30760 JobTracker  
```
然后说下日志文件报的错误：
```
jobtracker.info could only be replicated to 0 nodes, instead of 1  
```
这个问题，网上的解决办法也是多种多样，有的说是防火墙没关闭，但是其实我的机器根本没有防火墙。有的说是目录没有删除干净等等。
当然最后解决我问题的还是磁盘空间不足的问题。
在core-site.xml文件和hdfs-site.xml文件中配置的有namenode和datanode放置的目录，我的机器上，这个目录所在盘已经满了，于是我修改这两个文件配置到了另外一个盘上，问题就解决了。
其实一开始，我把hadoop-1.2.1.tar.gz文件解压到我自己的用户家目录下的时候就发现莫名其妙的异常，就是加压后的一些脚本文件内容都是空的，我以为是解压的过程中丢失了，于是我重新解压，才发现日志中说磁盘空间不足。
但是我又想在自己用户下工作，怎么办呢？于是我将hadoop-1.2.1.tar.gz文件移动到了一个剩余空间充裕的磁盘目录下，并解压，然后在我自己的家目录下为解压后的hadoop目录创建了软的符号链接，这样就可以了。
但是我没有意识到，我在core-site.xml文件和hdfs-site.xml文件中为hadoop指定的namenode和datanode存放的目录还是在我自己的家目录下，而这些目录就不是符号链接了，导致空间不足，所以报了上述错误，终于圆满解决。
