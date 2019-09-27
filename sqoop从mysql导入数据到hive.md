#  sqoop从mysql导入数据到hive

环境：

```
hadoop 2.7.2
hive 2.3.6
sqoop 1.4.7
```

## 安装Sqoop

[sqoop-1.4.7下载地址](http://www.apache.org/dyn/closer.lua/sqoop/1.4.7)

1. 下载下来解压后配置

```
cd $SQOOP_HOME/conf
mv sqoop-env-template.sh sqoop-env.sh 
vi sqoop-env.sh
```

```
#根据你的实际情况配置
#Set path to where bin/hadoop is available
export HADOOP_COMMON_HOME=/opt/moudle/hadoop-2.7.2

#Set path to where hadoop-*-core.jar is available
export HADOOP_MAPRED_HOME=/opt/moudle/hadoop-2.7.2

#set the path to where bin/hbase is available
#export HBASE_HOME=  我没有所有没配

#Set the path to where bin/hive is available
export HIVE_HOME=/opt/moudle/hive-1.2.1

#Set the path for where zookeper config dir is
#export ZOOCFGDIR=
export HIVE_CONF_DIR=/opt/moudle/hive-1.2.1/conf
```

2. 然后弄一个mysql的驱动包([下载地址 5.1.47](https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar))放入sqoop的lib目录下

## 测试使用

为了方便你可以将sqoop_home/bin写入/etc/profile。<font color="red">后面sqoop命令都指的是sqoop安装目录下bin目录下的sqoop脚本</font>

### 读取mysql数据库目录

```shell
# \表示换行  这样写主要是为了看得清晰点
# \后面不能有空格
# 任何语句和\之间都需要至少有一个空格
# 最后一行可以没有\ 
bin/sqoop list-databases \
--connect jdbc:mysql://hadoop001:3306 \
--username root   \
--password 123456
```

### 从Mysql导入数据到Hive

#### 建表(hive端)

- 方式一：create-hive-table语句

  ```shell
  sqoop create-hive-table \
  --connect jdbc:mysql://hadoop001:3306/learn \
  --username root \
  --password 123456 \
  --table user \  #mysql 的user表
  --hive-table zzy.user  # 自动在 hive 下 zzy库下创建一个对应的user表
  ```

- 方式二：手动建一张和mysql字段完全匹配的hive表

  ```sql
  create table if not exists zzy.test2(
      user_id bigint,
      user_name string,
      trade_time string
  )
  row format delimited
  FIELDS TERMINATED by ',';
  ```

#### 导入数据

```shell
bin/sqoop import \
--connect jdbc:mysql://hadoop001:3306/learn \
--username root --password 123456 \
--table user \
--hive-import \
--fields-terminated-by ',' \  #如果是自动建的表分隔符不用填。默认的分隔符是'\001'
--hive-table zzy.test2
```

```
如果不覆盖，多次添加的话会自动排序。
0	zzy	2019-09-18 00:00:00.0
0	zzy	2019-09-18 00:00:00.0
0	zzy	2019-09-18 00:00:00.0
1	zzy2	2019-09-20 15:25:17.0
1	zzy2	2019-09-20 15:25:17.0
1	zzy2	2019-09-20 15:25:17.0
2	zzy3	2019-09-10 15:25:32.0
2	zzy3	2019-09-10 15:25:32.0
2	zzy3	2019-09-10 15:25:32.0
```

#### 别的导入方式可以参考官方文档，接下来说一下过程中遇到的问题。

**Exception1：**

```
Sqoop:Import failed:java.lang.ClassNotFoundException:org.apache.hadoop.hive.conf.HiveConf
```

**解决办法：**

```
在/etc/profile中添加下面一行
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$HIVE_HOME/lib/*
```

**Exception2：**

```
INFO conf.HiveConf: Found configuration file null
WARN common.LogUtils: hive-site.xml not found on CLASSPATH
```

**解决办法：**

```
在sqoop-env.sh中添加export HIVE_CONF_DIR=$HIVE_HOME(这里填你自己的hive安装目录)/conf
```

**Exception3：**

```
main ERROR Could not register mbeans java.security.AccessControlException: access denied ("javax.management.MBeanTrustPermission" "register")
```

**解决办法：**

```
进入你的jdk-->jre-->lib-->security下
vi java.policy

grant {
	# 在grant这个标签里添加下面这句
	permission javax.management.MBeanTrustPermission "register";
};

```

**Exception4：**

```
ERROR exec.DDLTask:java.lang.NoSuchMethodError:com.fasterxml.jackson.databind.ObjectMapper.readerFor(Ljava/lang/Class;)Lcom/fasterxml/jackson/databind/ObjectReader;
```

**解决办法：**

```
删除你sqoop/lib下所有以jackson开头的jar包。
然后复制你hive/lib下所有以jackson开头的jar包到sqoop/lib下。
```

**Exception5：**

```
Sqoop:Import failed:java.lang.ClassNotFoundException:org.apache.hadoop.hive.conf.HiveConf
```

**解决办法：**

```
在/etc/profile中添加下面一行
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$HIVE_HOME/lib/*
```



