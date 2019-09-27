# Zookeeper

## 环境搭建

1. 导入安装包、解压

2. 修改配置文件

   ```shell
   # 1. 进入配置文件目录
   cd /opt/module/zookeeper-3.4.10/conf
   # 2.修改配置文件名称使其生效
   mv zoo_sample.cfg zoo.cfg
   ```

   ```txt
   # The number of milliseconds of each tick
   tickTime=2000
   # The number of ticks that the initial 
   # synchronization phase can take
   initLimit=10
   # The number of ticks that can pass between 
   # sending a request and getting an acknowledgement
   syncLimit=5
   # the directory where the snapshot is stored.
   # do not use /tmp for storage, /tmp here is just 
   # example sakes.
   
   # 主要修改这里，添加zookeeper的数据目录  主要里面会放zookeeper的myid
   dataDir=/opt/module/zookeeper-3.4.10/zkData
   # 配置zookeeper集群信息   2888通信端口  3888选举端口
   server.2=hadoop100:2888:3888
   server.3=hadoop101:2888:3888
   server.4=hadoop102:2888:3888
   
   
   # the port at which the clients will connect
   clientPort=2181
   # the maximum number of client connections.
   # increase this if you need to handle more clients
   #maxClientCnxns=60
   #
   # Be sure to read the maintenance section of the 
   # administrator guide before turning on autopurge.
   #
   # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
   #
   # The number of snapshots to retain in dataDir
   #autopurge.snapRetainCount=3
   # Purge task interval in hours
   # Set to "0" to disable auto purge feature
   #autopurge.purgeInterval=1
   ```

3. 新建数据目录

   ```shell
   cd /opt/module/zookeeper-3.4.10
   mkdir zkData
   cd zkData
   echo 1 >> myid
   ```

4. 集群同步：在集群上安装zookeeper  然后修改/opt/module/zookeeper-3.4.10/zkData/myid下的id

## Zookeeper使用

1. 启动Zookeeper

   ```shell
   #路径 /opt/module/zookeeper-3.4.10/bin
   ./zkServer.sh start
   ```

2. 查看Zookeeper状态

   ```shell
   ./zkServer.sh status
   #ZooKeeper JMX enabled by default
   #Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
   #Mode: follower
   ```

3. Zookeeper停止

   1. 直接杀掉进程  

      ```shell
      [root@hadoop100 bin]# jps
      2880 Jps
      2624 QuorumPeerMain
      [root@hadoop100 bin]# kill -9 2624
      [root@hadoop100 bin]# jps
      2897 Jps
      [root@hadoop100 bin]# 
      ```

   2. 通过zkServer.sh stop关闭

      ```shell
      [root@hadoop100 bin]# ./zkServer.sh stop
      ZooKeeper JMX enabled by default
      Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
      Stopping zookeeper ... STOPPED
      ```

4. 集群自动启动脚本

   ```shell
   #!/bin/bash
   for i in 0 1 2
   do
   ssh hadoop10$i "source /etc/profile;/opt/module/zookeeper-3.4.10/bin/zkServer.sh start"
   done
   ```

   

### 命令行客户端使用

1. 登录客户端

   ```shell
   # 路径/opt/module/zookeeper-3.4.10/bin
   ./zkCli.sh
   ```

2. 新建结点

   ```shell
   create [-s] [-e] path data acl
   ```

   ​	-s 是否带序号(类似于自增id)

   ​	-e 不加表示持久的结点，加表示临时的结点(客户端重启后消失),且临时结点不能有任何子节点

   ​	path 结点的路径

   ​	data 结点的数据

3. 获取结点的数据

   ```
   get path [watch] 
   ```

   【watch】 添加一次数据监听(如果结点发生变化，会出现一次变更信息提示)

   eg:

   ```shell
   [zk: localhost:2181(CONNECTED) 17] get /app1
   this is create test
   cZxid = 0x400000006
   ctime = Mon Aug 26 00:22:04 CST 2019
   mZxid = 0x400000006
   mtime = Mon Aug 26 00:22:04 CST 2019
   pZxid = 0x400000007
   cversion = 1
   dataVersion = 0
   aclVersion = 0
   ephemeralOwner = 0x0
   dataLength = 19
   numChildren = 1
   ```

4. 更新结点数据

   ```shell
   set path data [version]
   ```

   eg:

   ```shell
   [zk: localhost:2181(CONNECTED) 21] set /app1 "this is set test"
   cZxid = 0x400000006
   ctime = Mon Aug 26 00:22:04 CST 2019
   mZxid = 0x400000009
   mtime = Mon Aug 26 00:32:45 CST 2019
   pZxid = 0x400000007
   cversion = 1
   ## 这里变了(版本号)
   dataVersion = 1
   aclVersion = 0
   ephemeralOwner = 0x0
   dataLength = 16
   numChildren = 1
   
   [zk: localhost:2181(CONNECTED) 22] get /app1
   this is set test
   cZxid = 0x400000006
   ctime = Mon Aug 26 00:22:04 CST 2019
   mZxid = 0x400000009
   mtime = Mon Aug 26 00:32:45 CST 2019
   pZxid = 0x400000007
   cversion = 1
   dataVersion = 1
   aclVersion = 0
   ephemeralOwner = 0x0
   dataLength = 16
   numChildren = 1
   
   ```

5. 退出客户端  :quit

