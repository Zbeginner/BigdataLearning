

# 完全分布式HDFS搭建

## 同步脚本

```shell
#!/bin/bash

pcount=$#
if((pcount==0));then
echo no args;
exit;
fi

p1=$1
fname=`basename $p1`
echo fname=$fname

pdir=`cd -P $(dirname $p1);pwd`
ehco pdir=$pdir

user=`whoami`

for((host=101;host<104;host++));do
	echo -------------hadoop$host---------
	rsync -rvl $pdir/$fname $user@hadoop$host:$pdir
done
```

```xml
#Hadoop101
#core-site.xml
<configuration>
	<!-- 指定HDFS中NameNode的地址  -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop100:9000</value>
    </property>
	<!-- 指定Hadoop运行的临时文件的存储目录  -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/moudle/hadoop/data/tmp</value>
    </property>
</configuration>

```

```xml
#Hadoop101
#hdfs-site.xml
<configuration>
	<!-- 备份数  -->
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <!-- secondaryNamenode节点地址  -->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop103:50090</value>
    </property>
</configuration>

```

```txt
#/opt/module/hadoop-2.7.2/etc/hadoop/slaves
hadoop101
hadoop102
hadoop103
```

```xml
#Hadoop101
#yarn-site.xml
<configuration>
<!-- Site specific YARN configuration properties -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop102</value>
    </property>
</configuration>
```

## 时间同步(root用户)

1. /etc/ntp.conf

```
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst


server 127.127.1.0
fudge 127.127.1.0 stratum 10
```

2. vi /etc/sysconfig/ntpd

   添加  SYNC_HWCLOCK=yes

3. 重启ntpd

4. 配置定时任务

   其它机器上root用户 

   crontab -e 添加

   */10 * * * * /usr/sbin/ntpdate hadoop101

# 新节点服役

## 前期准备

1. 准备一台新的虚拟机环境和集群机器环境相同
2. 修改主机IP和主机名称
3. 删除hadoop目录下的data 和log文件夹
4. 配置hdfs和yarn对于新节点的SSH免密登陆

## 新节点配置

**在namenode节点上**  etc/hadoop/目录下

1. 新建dfs.hosts文件(文件名任意)   输入以下内容。hadoop104是新服务节点

   ```shell
   hadoop101
   hadoop102
   hadoop103
   hadoop104
   ```

2. vi hdfs-site.xml

   ```xml
   <property>
       <name>dfs.hosts</name>
       <!-- dfs.hosts文件目录 -->
       <value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts</value>
   </property>
   ```

3. vi slaves

   ```txt
   hadoop101
   hadoop102
   hadoop103
   #hadoop104新增
   hadoop104
   ```

4. 同步配置文件到集群上的机器上去

5. 刷新namenode和resourseManager节点

   1. hdfs dfsadmin -refreshNode
   2. yarn rmadmin -refreshNode

   正常情况下此时在集群页面就能看到一个dead的hadoop104 datanode节点

6. 在hadoop104上单独启动datanode和nodemanager

## 遇到的问题

hadoop104上单独启动datanode和nodemanager后  集群中的节点并没有正常工作。

解决思路：

​	查询104的datanode的日志信息发现：一直在尝试连接hadoop100:9000

​	但是正常的namenode节点应该是hadoop101:9000

​	在克隆机器100后只同步了hdfs-site.xml

​	修改core-site.xml 以及 yarn-site.xml后再重启datanode和nodemanager  一切正常

# 旧节点退役

1. 在namenode节点上新增dfs.hosts.exclude

   ```txt
   hadoop104  #表示要退役的节点
   ```

2. vi hdfs-site.xml

   ```xml
   <property>
       <name>dfs.hosts.exclude</name>
       <value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts.exclude</value>
   </property>
   ```

3. 刷新namenode和resourceManager

   1. hdfs dfsadmin -refreshNode
   2. yarn rmadmin -refreshNode

   这时候会发现hadoop104状态是Decommission In Progress表示正在向其它节点备份文件等状态变为Decommissioned再进行下一步。

4. 在hadoop104上单独关闭datanode和nodemanager

5. 在namenode节点上修改salves和dfs.hosts文件，删除其中关于hadoop104的信息

6. 刷新namenode和resourceManager ,节点成功退役

# 高可用HDFS 

# HDFS其它功能

## 归档

1. yarn必须打开  因为归档底层跑的是个MapReduce程序

2. 归档文件

   ```shell
   bin/hadoop archive -archiveName myhar.har -p /user/zzy/input /user.zzy/optput
                                 归档文件名称 固定参数  输入文件路径	输出文件路径
   ```

3. 查看归档文件

   ```shel
   hadoop fs -lsr har:///uer/zzy/output/myhar.har
   ```

4. 解锁归档文件

   ```shell
   hadoop fs -cp har:///uer/zzy/output/myhar.har /user/output
   ```

## 快照

1. 基本语法
   1. hdfs dfsadmin -allowSnapshot 路径    **开启指定目录的快照功能**
   2. hdfs dfsadmin -disallowSnapshot 路径   禁止指定目录的快照功能，**默认是禁止的**
   3. hdfs dfs -createSnapshot 路径  **对目录创建快照**
   4. hdfs dfs -createSnapshot 路径 名称  **指定名称创建快照**
   5. hdfs dfs -renameSnapshot 路径  旧名称 新名称  **重命名快照**
   6. hdfs lsSnapshottableDir  **列出当前用户所有可快照的目录**
   7. hdfs snapshotDiff   路径1 路径2   **比较两个快照目录的不同之处**
   8. hdfs dfs -deleteSnapshot\<path\>\<snapshotName\>   **删除快照**
   9. hdfs dfs -cp /user/zzy/input/.snapshot/xxxx  恢复快照

