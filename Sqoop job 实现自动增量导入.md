# sqoop job 实现自动增量导入

### 普通的增量导入

```
# 这个问题在于我们每次再增量导入的时候就要手动去更改--last-value  \的值。
# 否则就每次都是全量导入。显得不灵活
bin/sqoop import \
--connect jdbc:mysql://hadoop001:3306/learn \
--username root --password 123456 \
--table user \
-m 1 \
--incremental append \
--check-column user_id \
--last-value 0 \
--fields-terminated-by ',' \
--hive-partition-key dt \
--hive-partition-value '2019-09-19' \
--hive-import \
--hive-table zzy.test3
```

### sqoop job 增量导入

sqoop job每次会为我们维护last-value的值，达到自动增量导入的目的

#### 创建job

```
## -- import 中间有个空格 
bin/sqoop job --create mysql_hive_append -- import --connect jdbc:mysql://hadoop001:3306/learn \
--username root --password 123456 \
--table user \
-m 1 \
--incremental append \
--check-column user_id \
--last-value 0 \
--fields-terminated-by ',' \
--hive-partition-key dt \
--hive-partition-value '2019-09-19' \
--hive-import \
--hive-table zzy.test3
```

<font color="red" size=5>sqoop.Sqoop: Got exception running Sqoop: 
java.lang.NullPointerException，没遇到可以跳过</font>

```
19/09/20 09:57:47 ERROR sqoop.Sqoop: Got exception running Sqoop: 
java.lang.NullPointerException
	at org.json.JSONObject.<init>(JSONObject.java:144)  ##  缺少的东西
	at org.apache.sqoop.util.SqoopJsonUtil.getJsonStringforMap(SqoopJsonUtil.java:43)
	at org.apache.sqoop.SqoopOptions.writeProperties(SqoopOptions.java:785)
	at org.apache.sqoop.metastore.hsqldb.HsqldbJobStorage.createInternal(HsqldbJobStorage.java:399)
	at org.apache.sqoop.metastore.hsqldb.HsqldbJobStorage.create(HsqldbJobStorage.java:379)
	at org.apache.sqoop.tool.JobTool.createJob(JobTool.java:181)
	at org.apache.sqoop.tool.JobTool.run(JobTool.java:294)
	at org.apache.sqoop.Sqoop.run(Sqoop.java:147)
	at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
	at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:183)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:234)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:243)
	at org.apache.sqoop.Sqoop.main(Sqoop.java:252)
```

查了半天是缺少java-json.jar这么一个jar包。找了半天大部分CSDN都要钱。下面整理了一些可下载的地址。

[下载地址(需要翻墙)](http://www.java2s.com/Code/JarDownload/java-json/java-json.jar.zip)

[百度网盘](https://pan.baidu.com/s/1bnfXsHjW_e4FRk5aBUuybw)

#### 运行job

```
bin/sqoop job --exec mysql_hive_append
```

我这里明明设置了密码。但是还是要求我再输入一次mysql的连接密码。暂时没解决，输入就是了。

```
[zzy@hadoop001 sqoop-1.4.7]$bin/sqoop job --exec mysql_hive_append
19/09/20 10:20:29 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/moudle/hadoop-2.7.2/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/moudle/hive-1.2.1/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
Enter password:
```

导入一次，向Mysql表中加几条数据，然后再导入，发现只增加了几条新增的数据就成功了。