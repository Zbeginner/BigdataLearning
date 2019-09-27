# DataX 阿里离线数据同步工具

下载地址: [datax下载地址](http://datax-opensource.oss-cn-hangzhou.aliyuncs.com/datax.tar.gz)

官方指南:[Quick Start](https://github.com/alibaba/DataX/blob/master/userGuid.md)

## 介绍

```
DataX 是阿里巴巴集团内被广泛使用的离线数据同步工具/平台,实现包括 MySQL、SQL Server、Oracle、PostgreSQL、HDFS、Hive、HBase、OTS、ODPS 等各种异构数据源之间高效的数据同步功能。
```

datax其实就像Flume一样~，它们两个的架构都一样。总体一个思想我通过我的自定义输入读取数据，然后统一格式，在通过自定义输出输入数据。一个完美的中间件，通过我们自定义的格式达到不同数据源之间的数据同步。

相对于Flume，datax的上手难度和用户体验以及效率都大大提升,这是最主要的。

### datax的架构图

![1](img\datax的架构.png)

从上图，我们能简单的了解一下DataX的运行逻辑：

​	`MySQL`的数据经过我们的`MysqlReader`(前面提到的自定义输入，采集数据)，格式化成中间格式，然后暂存在`FrameWork`(缓冲，流控，并发，数据转换等核心技术区)上，`HDFSWriter`(自定义输出，写出数据)从FrameWork那里拉数据然后写到输出源`HDFS`上，完成了数据的同步。

#### 核心架构

![2](img\DataX核心架构.png)

这张图就能更清晰的理解DataX的运行逻辑了：(前提要知道：DataX把一次同步的任务叫做一个<font color="red">Job</font>)

一个Job会被切分(如果有设置的话)成若干个Task，多个Task并行执行。根据Task的数量，重组成一个个TaskGroup。每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，默认单个任务组的并发数量为5。

### Flume架构图

![Flume架构](img\Flume架构.png)

## DataX的安装

以下部分直接抄的DataX项目的Quick Start

- 工具部署

  - 方法一、直接下载DataX工具包：[DataX下载地址](http://datax-opensource.oss-cn-hangzhou.aliyuncs.com/datax.tar.gz)

    下载后解压至本地某个目录，进入bin目录，即可运行同步作业：

    ```
    $ cd  {YOUR_DATAX_HOME}/bin
    $ python datax.py {YOUR_JOB.json}
    ```

    自检脚本：    python {YOUR_DATAX_HOME}/bin/datax.py {YOUR_DATAX_HOME}/job/job.json

  - 方法二、下载DataX源码，自己编译：[DataX源码](https://github.com/alibaba/DataX)

    (1)、下载DataX源码：

    ```
    $ git clone git@github.com:alibaba/DataX.git
    ```

    (2)、通过maven打包：

    ```
    $ cd  {DataX_source_code_home}
    $ mvn -U clean package assembly:assembly -Dmaven.test.skip=true
    ```

    打包成功，日志显示如下：

    ```
    [INFO] BUILD SUCCESS
    [INFO] -----------------------------------------------------------------
    [INFO] Total time: 08:12 min
    [INFO] Finished at: 2015-12-13T16:26:48+08:00
    [INFO] Final Memory: 133M/960M
    [INFO] -----------------------------------------------------------------
    ```

    打包成功后的DataX包位于 {DataX_source_code_home}/target/datax/datax/ ，结构如下：

    ```
    $ cd  {DataX_source_code_home}
    $ ls ./target/datax/datax/
    bin		conf		job		lib		log		log_perf	plugin		script		tmp
    ```

    把最后这个datax目录cp到任意你想放的地方就能用了，不需要任何配置。

## DataX使用

```
说datax好用，其实就是它这种写json同步数据的方式很方便
我们只需要按照它的规范(还是中文的)，描述出我们的输入源和输出源就能很方便达到数据同步的效果。
```

#### [插件体系](https://github.com/alibaba/DataX/blob/master/introduction.md)(里面是目前支持的一些输入输出插件)

### HDFS数据同步到Mysql

```json
{
    "job": {
        "setting": {
            "speed": {
                "channel": 3 //并发数 
            }
        },
        "content": [
            {
                "reader": {
                    "name": "hdfsreader",  //输入端事HDFS,所以用hdfsreader
                    "parameter": {
                        "path": "/user/hive/warehouse/zzy.db/l1/*",  //采集目录
                        "defaultFS": "hdfs://hadoop001:9000",  //HDFS地址
                        "column": [
                            {
                                "index": 0,        //如果全取   "column":["*"],
                                "type": "string"
                            },
                            {
                                "index": 1,
                                "type": "string"
                            },
                            {
                                "index": 2,
                                "type": "long"
                            }
                        ],
                        "fileType": "text",   //文件类型
                        "encoding": "UTF-8",  //编码格式
                        "fieldDelimiter": ","  //字段切分符号
                    }
                },
                "writer": {    
                    "name": "mysqlwriter",   //输出端是mysql，所以用mysqlwriter
                    "parameter": {
                        "writeMode": "insert",   //写入数据库时的方式。
                        "username": "root",   //连接信息
                        "password": "123456",  //连接信息
                        "column": [
                            "id",
                            "login",
                            "cnt"
                        ],
                        "session": [
                            "set session sql_mode='ANSI'" //DataX在获取Mysql连接时，执行session指定的SQL语句，修改当前connection session属性
                        ],
                        "preSql": [
                            "truncate table l1"   //在执行写之前，可以先执行这里的sql语句  还有postSql，对应执行了写操作以后的sql
                        ],
                        "connection": [
                            {
                                "jdbcUrl": "jdbc:mysql://127.0.0.1:3306/learn?useUnicode=true&characterEncoding=utf8",  //连接信息
                                "table": [
                                    "l1" //对那张表操作，可以是多个表
                                ]
                            }
                        ]
                    }
                }
            }
        ]
    }
}
```

```shell
python ./bin/datax.py ./job/hdfs2mysql.json   #调用方式(当前在datax安装目录下)
```

### Mysql写入Hive

```json
{
    "job": {
        "setting": {
            "speed": {
                "channel": 1
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.02
            }
        },
        "content": [
            {
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "root",
                        "password": "123456",
                        "column": [
                            "id",
                            "login",
                            "cnt"
                        ],
                        "connection": [
                            {
                                "table": [
                                    "l1"
                                ],
                                "jdbcUrl": [
                                    "jdbc:mysql://127.0.0.1:3306/learn"
                                ]
                            }
                        ]
                    }
                },
                "writer": {
                    "name": "hdfswriter",
                    "parameter": {
                        "defaultFS": "hdfs://hadoop001:9000",
                        "fileType": "text",
                        "path": "/user/hive/warehouse/zzy.db/l1",
                        "fileName": "mysql_table_l1",
                        "column": [
                            {
                                "name": "id",
                                "type": "string"
                            },
                            {
                                "name": "login",
                                "type": "string"
                            },
                            {
                                "name": "cnt",
                                "type": "int"  //唯一要区分的就是这里，这个类型是写到hive中的数据类型，而不是dataX的数据类型
                            }
                        ],
                        "writeMode": "append",
                        "fieldDelimiter": ","
                    }
                }
            }
        ]
    }
}
```

