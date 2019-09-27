# Azkaban 运行DataX脚本

.project

```
azkaban-flow-version: 2.0
```

.flow

```
nodes:
    - name: job_mysql_to_hive
      type: command
      config:
        command: /usr/local/bin/python /opt/moudle/datax/bin/datax.py mysql2hive.json
```

调度方式要找到python的安装路径和datax的安装路径

mysql2hive.json

```
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
                            "*"
                        ],
                        "connection": [
                            {
                                "table": [
                                    "sales_order"
                                ],
                                "jdbcUrl": [
                                    "jdbc:mysql://127.0.0.1:3306/sales_source"
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
                        "path": "/user/hive/warehouse/sales_rds.db/sales_order",
                        "fileName": "sales_order",
                        "column": [
                            {
                                "name": "order_number",
                                "type": "int"
                            },
                            {
                                "name": "customer_number",
                                "type": "int"
                            },
                            {
                                "name": "product_code",
                                "type": "int"
                            },
                            {
                                "name": "order_date",
                                "type": "timestamp"
                            },
                            {
                                "name": "entry_date",
                                "type": "timestamp"
                            },
                            {
                                "name": "order_amount",
                                "type": "double"
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

mysql表结构

```sql
CREATE TABLE `sales_source` (
  `order_number` int(11) NOT NULL AUTO_INCREMENT,
  `customer_number` int(11) NOT NULL,
  `product_code` int(11) NOT NULL,
  `order_date` datetime(0) NOT NULL,
  `entry_date` datetime(0) NOT NULL,
  `order_amount` decimal(18, 2) NOT NULL,
  PRIMARY KEY (`order_number`) USING BTREE
) ;
```

hive表结构

```sql
CREATE TABLE `sales_order`(
  `order_number` int, 
  `customer_number` int, 
  `product_code` int, 
  `order_date` timestamp, 
  `entry_date` timestamp, 
  `order_amount` decimal(18,2)
)
```

