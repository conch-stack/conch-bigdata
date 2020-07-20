## Hive权限

```
sudo -u hdfs hdfs dfs -mkdir /user/admin
sudo -u hdfs hdfs dfs -chown admin:hadoop /user/admin
```



```sql
-- TODO 尝试使用 hive 用户登录shell
sudo -u hive hive
load data local inpath '/xxx' into table xxx;
```



https://blog.csdn.net/yrg5101/article/details/88837468

https://www.jianshu.com/p/dcef793cf395



##### hive.server2.enable.doAs

```
默认情况下，HiveServer2以提交查询的用户身份执行查询处理。但是，如果以下参数设置为false，则查询将以运行hiveserver2进程的用户身份运行。
hive.server2.enable.doAs - 模拟连接的用户，默认为true。
hive.server2.enable.doAs设置成false则，yarn作业获取到的hiveserver2用户都为hive用户。
设置成true则为实际的用户名

要防止在不安全模式下发生内存泄漏，请通过将以下参数设置为true来禁用文件系统缓存（请参阅  HIVE-4501）：
fs.hdfs.impl.disable.cache - 禁用HDFS文件系统缓存，默认为false。
fs.file.impl.disable.cache - 禁用本地文件系统缓存，默认为false。
```



https://blog.csdn.net/zqqnancy/article/details/51852794 

```xml
<property>
<name>hive.security.authorization.enabled</name>
<value>true</value>
</property>
 
<property>
    <name>hive.server2.enable.doAs</name>
    <value>false</value>
</property>
 
<property>
    <name>hive.users.in.admin.role</name>
    <value>hive</value>
</property>
 
<property>
<name>hive.security.authorization.manager</name> 
<value>org.apache.hadoop.hive.ql.security.authorization.plugin.sqlstd.SQLStdHiveAuthorizerFactory</value>
</property>
 
<property>
<name>hive.security.authenticator.manager</name> 
<value>org.apache.hadoop.hive.ql.security.SessionStateUserAuthenticator</value>
```



##### 我的修改

```
-- 1. 在Ambari中hive里面设置custom hive-site下添加
hive.users.in.admin.role=hdfs
-- 2. 修改Advanced hiveserver2-site中
hive.security.authorization.manager=org.apache.hadoop.hive.ql.security.authorization.plugin.sqlstd.SQLStdHiveAuthorizerFactory
原本为：
hive.security.authorization.manager=org.apache.hadoop.hive.ql.security.authorization.plugin.fallback.FallbackHiveAuthorizerFactory

-- 3.
sudo -u hdfs hive
jdbc:hive2://xxx>set hive.users.in.admin.role;
+--------------------------------+
|              set               |
+--------------------------------+
| hive.users.in.admin.role=hdfs  |
+--------------------------------+
jdbc:hive2://xxx>show current roles;
+---------+
|  role   |
+---------+
| public  |
+---------+
jdbc:hive2://xxx>set role admin;
INFO  : OK
jdbc:hive2://xxx>show roles;
+---------+
|  role   |
+---------+
| admin   |
| public  |
+---------+
jdbc:hive2://xxx>show current roles;
+--------+
|  role  |
+--------+
| admin  |
+--------+
```

