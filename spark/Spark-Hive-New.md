**参考：spark-demo 中的 JieBaKryHive.java**

1. Hive Warehouse Connector - HWC

　　在hdp3.0，默认spark和hive各使用自己的metastore catalog，即在hive中创建的表spark中是看不到的。

　　原因是新版hive支持ACID，默认也启用了ACID Manager，而SparkSQL还不支持ACID，所以它们两个各自使用不同的catalog。

　　如果要让sparksql直接使用hive元数据，有两种方案：

　　1.hive禁用ACID，spark使用hive的catalog

　　2.spark通过HWC访问hive元数据;

　　建议使用HWC，spark可以通过hwc使用hive元数据，并且也支持ranger,但只支持如下3类应用：

　　· Spark shell

　　· PySpark

　　· The spark-submit script

　　也就是还不支持spark-sql?

　　Updates for HDP-3.0:

　　· Hive uses the "hive" catalog, and Spark uses the "spark" catalog. No extra configuration steps are required – these catalogs are created automatically when you install or upgrade to HDP-3.0 (in Ambari the Spark metastore.catalog.default property is set to spark in "Advanced spark2-hive-site-override").

　　· You can use the Hive Warehouse Connector to read and write Spark DataFrames and Streaming DataFrames to and from Apache Hive using low-latency, analytical processing (LLAP). Apache Ranger and the Hive Warehouse Connector now provide fine-grained row and column access control to Spark data stored in Hive.

　　2. spark hwc配置



　　3.使用

　　启动shell：

spark-shell --jars /usr/hdp/current/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.0.0-1634.jar

　　注：因为hive-warehouse-connector-assembly-1.0.0.3.0.0.0-1634.jar 没有s3 api，在访问s3文件时会报错，但最终会查出结果。



网上资料：

#创建hive会话

scala> val hive = com.hortonworks.spark.sql.hive.llap.HiveWarehouseBuilder.session(spark).build()   #或val hive = com.hortonworks.hwc.HiveWarehouseSession.session(spark).build()

#如果使用用户名密码

val hive = com.hortonworks.hwc.HiveWarehouseSession.session(spark).userPassword("hive","passw").build()

#查看数据库

scala> hive.showDatabases().show

+------------------+

|     database_name|

+------------------+

|           default|

|information_schema|

|               iot|

|               sys|

+------------------+

#查看数据表

scala>  hive.showTables().show

+--------+

|tab_name|

+--------+

| invites|

|   pokes|

+--------+

#查询数据表内容

hive.executeQuery("SELECT * FROM pokes limit 3").show()

#切换数据库

hive.setDatabase("iot")

　　4.SparkSQL 使用HWC

　　需要配置Custom spark-thrift-sparkconf：

- spark.sql.hive.hiveserver2.url=jdbc:hive2://{hiveserver-interactive-hostname}:10500

- spark.jars=/usr/hdp/current/hive_warehouse_connector/hive-warehouse-connector-assembly-1.0.0.3.0.0.0-1634.jar

- spark.hadoop.hive.zookeeper.quorum={some-or-all-zookeeper-hostnames}:2181

- spark.hadoop.hive.llap.daemon.service.hosts=@llap0

　　属性spark_thrift_cmd_opts设置值：--jars /usr/hdp/2.6.3.0-235/spark_llap/spark-llap-assembly-1.0.0.2.6.3.0-235.jar --conf spark.sql.hive.llap=true

　　连接spark jdbc：

bin/beeline -u jdbc:hive2://slave5.cluster.local:10016 -n hive



0: jdbc:hive2://slave5.cluster.local:10016> show databases;

+---------------+--+

| databaseName  |

+---------------+--+

| default       |

| spdb          |

+---------------+--+

　　结果不对，还是spark catalog 。

　　另附: shell下访问spark catalog的结果：

scala>  spark.catalog.listDatabases.show

Hive Session ID = 1ea54a87-9df0-4ca0-b7a4-f741bb091e6f

+-------+----------------+--------------------+

|   name|     description|         locationUri|

+-------+----------------+--------------------+

|default|default database|hdfs://slave5.clu...|

|   spdb|                |hdfs://slave5.clu...|

+-------+----------------+--------------------+