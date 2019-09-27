# Hive2.3.6更换TEZ引擎

前提环境

```
1. hadoop  我的是2.7.1
2. hive   我的是2.3.6
```

## Tez环境准备

1. 下载Tez的安装包解压 [下载路径](https://mirrors.tuna.tsinghua.edu.cn/apache/tez/0.9.0/)

2. 环境准备

   1. 进去tez安装目录下

   2. [root@hadoop001 conf]# vi tez-site.xml 

      ```xml
      <?xml version="1.0" encoding="UTF-8"?>
      <configuration>
          <!-- hdfs上tez的压缩包 基础使用配这一个属性就行 -->
          <property>
              <name>tez.lib.uris</name>
              <value>${fs.defaultFS}/tez-0.9.0/tez.tar.gz</value>
          </property>
      
          <property>
              <name>tez.use.cluster.hadoop-libs</name>
              <value>true</value>
          </property>
      
          <property>
              <name>tez.history.logging.service.class</name>
              <value>org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService</value>
          </property>
      </configuration>
      
      ```

   3. 上传tez压缩包到HDFS上

      1. hdfs dfs -mkdir /tez-0.9.0
      2. hdfs dfs -put /usr/local/tez-0.9.0/share/tez.tar.gz     /tez-0.9.0

   4. tez下的lib目录中的hadoop包的版本和真实安装的hadoop版本不一致，需要将其jar包换成一致。

      ```shell
      #删除不符合版本的jar:
      [root@hadoop01 tez-0.9.0]# rm -rf ./lib/hadoop-mapreduce-client-core-2.7.0.jar ./lib/hadoop-mapreduce-client-common-2.7.0.jar
      #重新再hadoop目录中拷贝:
      [root@hadoop01 tez-0.9.0]# cp /usr/local/hadoop-2.7.1/share/hadoop/mapreduce/hadoop-mapreduce-client-common-2.7.1.jar /usr/local/hadoop-2.7.1/share/hadoop/mapreduce/hadoop-mapreduce-client-core-2.7.1.jar /usr/local/tez-0.9.0/lib/
      
      ```

## 安装Tez在Hive上

这种方式侵入性不强，配置完以后只有hive能随意切换引擎。而别的运行在hadoop的mr程序则还是只能走原来的yarn引擎。

   1. 配置hive-env-sh  新增以下内容

      ```shell
      #TEZ_HOME根据你实际情况来
      export TEZ_HOME=/opt/moudle/tez
      export TEZ_JARS=""
      for jar in `ls $TEZ_HOME |grep jar`; do
          export TEZ_JARS=$TEZ_JARS:$TEZ_HOME/$jar
      done
      for jar in `ls $TEZ_HOME/lib`; do
       export TEZ_JARS=$TEZ_JARS:$TEZ_HOME/lib/$jar
      done
      #这个lzo的包去自己hadoop下找对应的
      export HIVE_AUX_JARS_PATH=/opt/moudle/hadoop-2.7.2/share/hadoop/common/hadoop-lzo-0.4.21-SNAPSHOT.jar$TEZ_JARS
      ```

   2. <font color ="blue">如果嫌弃上面增加这么多代码的过程很麻烦。。就可以直接将tez下的jar和tez/lib下的jar拷贝到$hive_home/lib下</font>

   3. 启动hive测试

      ```shell
      hive (default)> set hive.execution.engine=tez;
      
      
      hive (default)> select
                    > id,
                    > count(id)
                    > from zzy.l1
                    > group by id;
      03:38:10.961 [59fd501b-ea06-4ce3-83e3-24b71a218eb8 main] ERROR org.apache.hadoop.hdfs.KeyProviderCache - Could not find uri with key [dfs.encryption.key.provider.uri] to create a keyProvider !!
      Query ID = root_20190919033809_a5d0a892-78d7-4c21-ae26-234fc6429664
      Total jobs = 1
      Launching Job 1 out of 1
      Status: Running (Executing on YARN cluster with App id application_1568834471830_0001)
      
      ----------------------------------------------------------------------------------------------
              VERTICES      MODE        STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED  
      ----------------------------------------------------------------------------------------------
      Map 1 .......... container     SUCCEEDED      1          1        0        0       0       0  
         Reducer 2 ...... container     SUCCEEDED      1          1        0        0       0       0  
      ----------------------------------------------------------------------------------------------
      VERTICES: 02/02  [==========================>>] 100%  ELAPSED TIME: 6.99 s     
      ----------------------------------------------------------------------------------------------
      ```

      出现类似的结果就代表成功了。

   4. 但是这样在hive如果和hadoop的master在同一个结点上是没问题的，如果两个任务不在同一个机器上(考虑两个都很费资源，保证稳定的时候)

      ```
      FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.tez.TezTask
      ```

      查询日志

      ```
      File does not exist: hdfs:/user/zzy   #zzy是我hive运行的用户
      ```

      知道问题就好解决了，说我在HDFS没有zzy这个用户的目录，创建一个就好。在运行tez引擎就没有错误了。

## 安装Tez在Hadoop上

这种方式侵入性很强，对原hadoop集群有一定影响。会使得所有在yarn上运行的mapreduce都只能走tez引擎，所有hive运行的时候自然也是tez。

1. 将 tez-site.xml 复制一份到hadoop配置目录下($HADOOP_HOME/etc/hadoop)

2. 修改hadoop-env.sh

   ```shell
   # 新增以下内容
   export TEZ_HOME=/opt/moudle/tez  #是你的tez的解压安装目录
   for jar in `ls $TEZ_HOME |grep jar`; do
       export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$TEZ_HOME/$jar
   done
   for jar in `ls $TEZ_HOME/lib`; do
       export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$TEZ_HOME/lib/$jar
   done
   ```

3. 修改mapred-site.xml

   ```
   <property>
           <name>mapreduce.framework.name</name>
           <value>yarn-tez</value>
   </property>
   ```

4. 同步集群

5. 随便上传一个文件到HDFS上

6. 进去tez的安装目录

   ```
   hadoop jar ./tez-examples-0.9.0.jar orderedwordcount /LICENSE /out  #/LICENSE是我上传的问题
   ```

7. 这时候去看8088，会看到跑的是给TEZ的任务，就成功了

8. 测试Hive是否跑的直接就是Tez任务

   1. 直接执行一个语句

      ```
      hive (default)> select
                    > id,
                    > count(id)·
                    > from zzy.l1
                    > group by id;
      Hadoop job information for Stage-1: number of mappers: 0; number of reducers: 0
      2019-09-19 04:10:56,905 Stage-1 map = 0%,  reduce = 0%
      2019-09-19 04:11:01,094 Stage-1 map = 100%,  reduce = 100%
      Ended Job = job_1568837085850_0002
      MapReduce Jobs Launched: 
      Stage-Stage-1:  HDFS Read: 0 HDFS Write: 0 SUCCESS
      Total MapReduce CPU Time Spent: 0 msec
      OK
      A	8
      B	7
      Time taken: 17.341 seconds, Fetched: 2 row(s)
      ```

   2. 感觉好像还是走的MR?    但是去看8088可以看出确实走的TEZ，只是没出那炫酷的进度条~

   3. set hive.execution.engine=tez;  加上这个就能显示进度条了

