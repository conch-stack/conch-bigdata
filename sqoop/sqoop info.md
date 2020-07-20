### Sqoop1:(全量同步)

A tool to transfer and sync data between relational databases to hdfs.

use sqoop client instead of starting a server

baseed on Map() to create MapReduce job

submit sqoop job with shell command, sqoop will read meta data from RDBMS, 并根据并发度和数据表大小将数据分成若干片，每片交给一个Map Task处理，多个Map Task同时去取数据库中数据，并行写入目标存储系统如HDFS、HBase、和Hive

- 并发度
- 数据源
- 超时时间

使用:

```sqoop
// HDP中sqoop默认使用mysql的驱动，无需单独配置
// 校验
sqoop list-tables --connect jdbc:mysql://192.168.16.212:3306/hive --username root --password 123456
sqoop list-databases --connect jdbc:mysql://192.168.16.212:3306/hive --username root --password 123456

// MySQL -> HDFS
sqoop import --connect jdbc:mysql://localhost:3306/test --username root --password 123456 --table t_test --fields-terminated-by ',' --null-string '**' -m 2 --append --target-dir '/data/sqoop/test.db

// Hive -> MySQL
sqoop export -D sqoop.export.records.per.statement=100 --connect jdbc:mysql://localhost:3306/test --username root --password 123456 --table t_test1 --fields-terminated-by ',' --export-dir '/usr/hive/....' --batch  --update-key uid --update-mode allowinsert

// MySQL -> HBase
sqoop import --connect jdbc:mysql://localhost:3306/test --username root --password 123456 --table user --hbase-table user --column-family behavior --hbase-create-table

$ sqoop import 
    --connect jdbc:mysql://localhost/serviceorderdb 
    --username root -P 
    --table customercontactinfo 
    --columns "customernum,customername" 
    --hbase-table customercontactinfo 
    --column-family CustomerName 
    --hbase-row-key customernum -m 1
Enter password:
...
13/08/17 16:53:01 INFO mapreduce.ImportJobBase: Retrieved 5 records.
$ sqoop import 
    --connect jdbc:mysql://localhost/serviceorderdb 
    --username root -P 
    --table customercontactinfo 
    --columns "customernum,contactinfo" 
    --hbase-table customercontactinfo 
    --column-family ContactInfo 
    --hbase-row-key customernum -m 1
Enter password:
...
13/08/17 17:00:59 INFO mapreduce.ImportJobBase: Retrieved 5 records.
$ sqoop import 
    --connect jdbc:mysql://localhost/serviceorderdb 
    --username root -P 
    --table customercontactinfo 
    --columns "customernum,productnums" 
    --hbase-table customercontactinfo 
    --column-family ProductNums 
    --hbase-row-key customernum -m 1
Enter password:
```

常用参数：

- -config 指定应用程序配置文件
- -D 指定属性和属性值
- -fs  local or namenode:port 指定namenode地址
- -jt local or resourcemanager:port 指定ResourceManager地址
- -files 逗号分隔的文件列表，需要分发到集群中各个节点的文件列表
- --connect <jdbc-uri>
- --driver 驱动类 com.mysql.jdbc.Driver
- --username  --password
- --table <tablename>
- --target-dir <dir>  HDFS目录
- --as-textfile  默认文件格式
- --as-parquetfile
- --as-avrodatafile
- --as-sequencefile
- -m  <n> 并发启动的Map Task数目 并行度
- -e <sql> 只将指定sql返回的数据导出到HDFS中
- --export-dir <dir> 导出的目录
- --batch 批量处理
- --update-key <属性名> 根据主键或者若干列更新数据，默认未设置，HDFS中的数据会新增到尾部
-  --update-mode <mode> 更新的模式  updateonly只做更新 or allowinsert更新且运行（不更新的数据）新增
- --fields-terminated-by <char> 按char分隔  默认 逗号
- --lines-terminated-by <char> 按char换行 默认 \n
- --append 追加模式

优点：

​	简单方便

缺点：

​	依赖多，开发逻辑难



TODO： sqoop如何保证容错，机器或网络故障后如何完成同步工作？



>sqoop是如何根据--split-by进行分区的？
> 假设有一张表test，sqoop命令中--split-by id --m 10。首先，sqoop会去查表的元数据，sqoop会向关系型数据库比如mysql发送一个命令:select max(id),min(id) from test。然后会把max、min之间的区间平均分为10分，最后10个并行的map去找数据库，导数据就正式开始了。





### Sqoop2：

提供Server与Client，将复杂操作抽象到Server端，提供REST接口供Client调用，简化客户端逻辑，提供更加灵活便捷的操作

缺点：

​	部署维护困难



### 增量同步：

#### CDC (change data capture)

> **Low的方法**
>
> * 定时扫描整表
>
>   sqoop增量实现方式：周期性的扫描整表，把变化的数据找出来，发送给数据搜集器
>
> * 写双份
>
>   修改业务代码，将数据修改同时发送给数据库和数据搜集器
>
> * 利用触发机制
>
>   利用数据库触发器



方案：基于事务或提交日志解析

#### Canal(阿里)

主要针对MySQL

模拟数据库主备复制协议，接收数据库的binlog，捕获数据更改

* Canal实现MySQL主备复制协议，向MySQL Server发送dump协议
* MySQL接收到dump请求，向Canal推送binlog
* Canal解析binlog对象，并发送给各个消费者

#### DataBus(LinkedIn)

支持MySQL与Oracle

架构可扩展



### 多机房数据同步

#### Otter(阿里)

定位：分布式数据库同步系统













