## Problems

HDFS 

```
问题：
	Connection failed: [Errno 111] Connection refused to apollo03.dev.zjz:2049
解决：
	HDFS NFS Gateway工作需要依附 rpcbind 服务，所以启动前需要确定rpcbind服务正常开启。 
	service rpcbind status
	service rpcbind start
	
	sudo systemctl status rpcbind
	sudo vim /etc/systemd/system/sockets.target.wants/rpcbind.socket
	systemctl status rpcbind.socket
	
	● rpcbind.socket - RPCbind Server Activation Socket
   Loaded: loaded (/usr/lib/systemd/system/rpcbind.socket; enabled; vendor preset: enabled)
   Active: failed (Result: resources)
   Listen: /var/run/rpcbind.sock (Stream)
           0.0.0.0:111 (Stream)
           0.0.0.0:111 (Datagram)
           [::]:111 (Stream)
           [::]:111 (Datagram)
	Warning: rpcbind.socket changed on disk. Run 'systemctl daemon-reload' to reload units.

	sudo systemctl daemon-reload
	sudo systemctl start rpcbind.socket
	解决！

问题：
	hadoop集群安全模式，导致hbase无法读取，yarn无法运行
	
	在安全模式下输入指令：
  hadoop dfsadmin -safemode leave
  hdfs dfsadmin -safemode leave
  sudo -u hdfs hdfs dfsadmin -safemode leave
  即可退出安全模式

  格式：Usage: java DFSAdmin [-safemode enter | leave | get |wait]
  用户可以通过dfsadmin -safemode value 来操作安全模式，参数value的说明如下：
  enter - 进入安全模式
  leave - 强制NameNode离开安全模式
  get   - 返回安全模式是否开启的信息
  wait  - 等待，一直到安全模式结束
  
问题：权限
	change dfs.permissions.enabled to false
```



Hive

```
修改Hive Interactive  的 HiveServer2 Port 和 thrift.http.port 的地址
以匹配 HiveServer2服务

修改hive.server2.thrift.http.port=10002

待定
```



HBase:

```
// 检查HBase集群状态: 
hbase hbck
// https://blog.csdn.net/xiao_jun_0820/article/details/28602213
// https://www.cnblogs.com/quchunhui/p/9583746.html
```



TimeLine:

```
问题：
	内嵌HBase问题
// 参考，不用这个 https://www.cnblogs.com/langfanyun/p/10821415.html
// 官方：
https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.1.0/data-operating-system/content/configure_hbase_for_timeline_service_2.0.html

// {hdp-dir} = /usr/hdp/3.1.0.0-78
export HBASE_CLASSPATH_PREFIX=/usr/hdp/3.1.0.0-78/hadoop-yarn/timelineservice/*; /usr/hdp/3.1.0.0-78/hbase/bin/hbase org.apache.hadoop.yarn.server.timelineservice.storage.TimelineSchemaCreator -Dhbase.client.retries.number=35 -create -s

// 日志 
INFO  [main] storage.TimelineSchemaCreator: Successfully created HBase schema. 
INFO  [main] storage.TimelineSchemaCreator: Schema creation finished successfully

// HBase授权
grant 'yarn', 'RWXCA'

错误：ERROR: DISABLED: Security features are not available
search keywords：HDP HBase acl
参考：https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.1.0/security-reference/content/kerberos_nonambari_configure_hbase_for_access_control_lists_acl_.html
解决：因为目前集群未开启安全kerberos，所以此参考暂时无法处理，先看看不开启acl是不是任意用户都可访问
直接 scan timelineservice相关的表，发现里面已经有数据了，说明yarn可以正常读取
```



Ambari:

```
问题：
	Cannot create /var/run/ambari-server/stack-recommendations
原因：文件权限
解决：
	sudo chown -R ambari /var/run/ambari-server

参考：
	https://blog.csdn.net/qq_19968255/article/details/72881989
```



HDP 3.1.0 HIVE使用tez 长时间无反应 成功解决

```
问题：
	HDP 3.1.0 安装的HIVE使用tez，执行任务需要用到tez session时会找不到
解决：
1. 在打开后增加以下设置
		set hive.server2.tez.initialize.default.sessions=true;
2. 如需一直生效，在hive的配置文件hive-site.xml中添加
	<property>
     <name>hive.server2.tez.initialize.default.sessions</name>
     <value>true</value>
	</property>
3. 在ambari的管理界面点开hive advance 设置
		Start Tez session at Initialization 为 true
```



Spark (2.x问题汇总：https://blog.csdn.net/xwc35047/article/details/53933265)

```
问题：
	HDP 上安装了 Hive3.1 和 Spark2, 提交 Spark 作业时，报找不到 Hive 中表的问题
解决：
网上的另外几种方法：
1. 把 hive-site.xml 复制到 Spark 的 conf 目录下。
    我看了一下 spark 的 conf 目录，有 hive-site.xml 这个表的，而且从日志中也可以看到 spark 能找到 hive 的 thrift://datacenter2:9083 这个地址，说明没问题。

2. 创建 spark session 的时候要启用 hive。
val ss = SparkSession.builder().appName("统计").enableHiveSupport().getOrCreate()
我的程序里有启用的，所以也不是原因。

3. 关闭 Hive 3 中的默认的 ACID 功能，修改如下几个参数
hive.strict.managed.tables=false 
hive.create.as.insert.only=false 
metastore.create.as.acid=false
   试过之后，问题依旧。
   崩溃了，找不到其它解决方法了。先记录一下。
 ================================================
      有别的事，先做别的了。过了2天，抱着试试看的态度，在 /etc/spark2/3.1.0.0-78/0 下建了个软链接到  /etc/hive/conf 下的 hive-site.xml ，竟然找得到表了。通过比较，发现原 spark 下的 hive-site.xml 里多了一个 metastore.catalog.default 的配置，值是 spark。在网上搜了一下，才知道要改成 hive 才可以读 hive 下创建的表。这个值我理解的是表示hive仓库的命名空间。为什么 Spark 没有默认设置成 hive 的 catalog 的呢？ 因为 HDP 3.1 中的 hive 会默认开启 ACID，spark 读取 ACID 的 表时，会出错，所以设置了一个 spark 的 catalog。
      
      
问题：spark-shell
scala> spark.sql("select * from orders")  
res7: org.apache.spark.sql.DataFrame = [order_id: string, user_id: string ... 5 more fields]
scala> res7.show() 
错误：java.io.IOException: Not a file: hdfs://hdftofuat/warehouse/tablespace/managed/hive/zjj.db/orders/delta_0000001_0000001_0000
解决：
Unlike Hive, spark can not traverse through sub folders in HDFS.(与Hive不同，spark不能遍历HDFS中的子文件夹)
网上很多让设置这个的解决方案：尝试成功
scala> sc.hadoopConfiguration.setBoolean("mapreduce.input.fileinputformat.input.dir.recursive", true)
深入分析：
存在子文件是因为，ACID的事务
INSERT 语句会在一个事务中运行。它会创建名为 delta 的目录，存放事务的信息和表的数据。

metastore.catalog.default:hive
这个选项默认为Spark， 即读取SparkSQL自己的metastore_db，修改完后，Spark Shell会去读取Hive的metastore，这样就可以实现以Spark Shell方式访问Hive SQL方式创建的databases/tables.

统一解决：
给mapReduce2添加mapreduce.input.fileinputformat.input.dir.recursive=true的custom设置
```



全新Cloudera：https://www.cloudera.com/downloads.html

Doc：

https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.4/index.html

https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.4/release-notes/content/comp_versions.html