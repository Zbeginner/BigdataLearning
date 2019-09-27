# HIve1.x升级到2.x，且保留元数据

1. 下载源码包或者编译过的二进制包都无所谓(源码的就自己编译一下)。

2. 先备份原来的hive

   ```shell
   mv hive-1.2.1/ hive-1.2.1-back/
   ```

3. 解压新版的hive到相同的目录并更名为hive-1.2.1<font color="red">(原来hive的名称，为了不去修改环境变量什么的，这样方便点)</font>

4. 升级元数据，hive已经为我们准备好了升级元数据的脚本了，我们只需要运行这些脚本就好，没有直接跳的只能一步步来。

   ```
   scripts/
   └── metastore
       └── upgrade
           ├── mysql
           │   ├── upgrade-0.10.0-to-0.11.0.mysql.sql
           │   ├── upgrade-0.11.0-to-0.12.0.mysql.sql
           │   ├── upgrade-0.12.0-to-0.13.0.mysql.sql
           │   ├── upgrade-0.13.0-to-0.14.0.mysql.sql
           │   ├── upgrade-0.14.0-to-1.1.0.mysql.sql
           │   ├── upgrade-0.5.0-to-0.6.0.mysql.sql
           │   ├── upgrade-0.6.0-to-0.7.0.mysql.sql
           │   ├── upgrade-0.7.0-to-0.8.0.mysql.sql
           │   ├── upgrade-0.8.0-to-0.9.0.mysql.sql
           │   ├── upgrade-0.9.0-to-0.10.0.mysql.sql
           │   ├── upgrade-1.1.0-to-1.2.0.mysql.sql
           │   ├── upgrade-1.2.0-to-1.3.0.mysql.sql
           │   ├── upgrade-1.2.0-to-2.0.0.mysql.sql  将1.2.x的元数据升级到2.0.0
           │   ├── upgrade-2.0.0-to-2.1.0.mysql.sql  将2.0.0的升级到2.1.0
           │   ├── upgrade-2.1.0-to-2.2.0.mysql.sql 		.
           │   ├── upgrade-2.2.0-to-2.3.0.mysql.sql		.
   没用的信息我删了，我是mysql管理的元数据，这里面还有别的数据库的升级脚本。
   ```

   1. 进入到这个目录下   $HIVE_HOME/scripts/metastore/upgrade/mysql

   2. 登录自己的mysql,进去你原来那个元数据库   我的是use hive;

   3. 因为我的是1.2.1--->2.3.6,分别运行了以下命令

      ```
      source /opt/moudle/hive-1.2.1/scripts/metastore/upgrade/mysql/upgrade-1.2.0-to-2.0.0.mysql.sql;
      source /opt/moudle/hive-1.2.1/scripts/metastore/upgrade/mysql/upgrade-2.0.0-to-2.1.0.mysql.sql;
      source /opt/moudle/hive-1.2.1/scripts/metastore/upgrade/mysql/upgrade-2.1.0-to-2.2.0.mysql.sql;
      source /opt/moudle/hive-1.2.1/scripts/metastore/upgrade/mysql/uupgrade-2.2.0-to-2.3.0.mysql.sql;
      
      如果不报错就问题不大了，可能会有某些表找不到，没事。
      ```

5. 将原版本的hive的jdbc驱动和配置文件拷贝到新的hive里就行了

6. 启动元数据服务 hive --service metastore &

7. 启动hive

8. 查看是否元数据成功升级

9. 查看hive是否可用

10. 都没事就成了~祝你好运