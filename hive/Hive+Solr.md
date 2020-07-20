## Hive integrate with Solr



>注意：
>
>​	安装第三方jar包时，需设置hive和interactive的 HIVE_AUX_JARS_PATH
>
>​	参考Ambari的配置：在HDPS安装文档中有链接
>
>​	Jar放入${HIVE_HOME}/auxlib目录
>
>​	HDP的 HIVE_HOME 是/usr/hdp/current/hive-server2
>
>​	完整路径为 ： /usr/hdp/current/hive-server2/auxlib
>
>- 问题
>  - 后续发现在Ambari中的hive-env中去掉export的auxlib相关配置后，系统重启能够自动识别第三方jar包
>  - 将hive-interactive-env中相应的配置也一起去掉
>  - 这样就解决了Tez在执行时，前一两次可以，后面报错：***auxlib. Failing because I am unlikely to write too
>  - Tez Session中可查看资源信息：hdfs:/tmp/hive/hdfs/_tez_session_dir/



#### 创建Hive表

```
CREATE EXTERNAL TABLE solr (id string, field1_s string, field2_i int) 
      STORED BY 'com.lucidworks.hadoop.hive.LWStorageHandler' 
      LOCATION '/tmp/solr' 
      TBLPROPERTIES('solr.zkhost' = '192.168.20.68:2181,192.168.20.65:2181,192.168.20.174:2181/solr', 
                    'solr.collection' = 'collection1',
                    'solr.query' = '*:*');
                    
SHOW tables;
DESC solr;

SELECT id, field1_s, field2_i FROM solr;
# 报错：提示collection1不存在 
# Error: java.io.IOException: shaded.org.apache.solr.common.SolrException: Collection not found: collection1 (state=,code=0)
# 解决：需手动在solr中创建collection1

# 模板
SELECT id, field1_s, field2_i FROM solr left
      JOIN sometable right
      WHERE left.id = right.id;
INSERT INTO solr
      SELECT id, field1_s, field2_i FROM sometable;      

# 测试
sudo -u hdfs hive
INSERT INTO solr VALUES('0001','xiaodou',28);
INSERT INTO solr SELECT id, title as field1_s, seq as field2_i FROM books;

# 报错
Unable to find class: com.lucidworks.hadoop.hive.LWHiveInputFormat
java.lang.ClassNotFoundException: com.lucidworks.hadoop.hive.LWHiveInputFormat
# 解决：请按照文章开头，正确配置第三方包地址

# 将books.csv上传的hdfs上
sudo -u hdfs hdfs dfs -mkdir -p /tmp/csv
sudo -u hdfs hdfs dfs -ls /tmp
sudo -u hdfs hdfs dfs -chown root:hdfs /tmp/csv
sudo hdfs dfs -put books.csv /tmp/csv
sudo -u hdfs hdfs dfs -ls /tmp/csv

CREATE TABLE books (id STRING, cat STRING, title STRING, price FLOAT, in_stock BOOLEAN, author STRING, series STRING, seq INT, genre STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' lines terminated BY '\n' stored AS textfile TBLPROPERTIES("skip.header.line.count"="1");

LOAD DATA  INPATH 'hdfs:/tmp/csv/books.csv' OVERWRITE INTO TABLE books;
```

##### Defining Fields for Solr

- dynamic fields
- create all of your fields in solr before indexing any content from hive



##### Table Properties

> - solr.zkhost
>
>   The location of the ZooKeeper quorum if using LucidWorks in SolrCloud mode. If this property is set along with the `solr.server.url` property, the `solr.server.url` property will take precedence.
>
> - solr.server.url
>
>   The location of the Solr instance if not using LucidWorks in SolrCloud mode. If this property is set along with the `solr.zkhost`property, this property will take precedence.
>
> - solr.collection
>
>   The Solr collection for this table. If not defined, an exception will be thrown.
>
> - solr.query
>
>   The specific Solr query to execute to read this table. If not defined, a default of `*:*` will be used. This property is not needed when loading data to a table, but is needed when defining the table so Hive can later read the table.
>
> - lww.commit.on.close
>
>   If true, inserts will be automatically committed when the connection is closed. True is the default.
>
> - lww.jaas.file
>
>   Used only when indexing to or reading from a Solr cluster secured with Kerberos.
>
>   This property defines the path to a JAAS file that contains a service principal and keytab location for a user who is authorized to read from and write to Solr and Hive.
>
>   The JAAS configuration file **must** be copied to the same path on every node where a Node Manager is running (i.e., every node where map/reduce tasks are executed). Here is a sample section of a JAAS file:
>
>   ```
>   Client { 
>     com.sun.security.auth.module.Krb5LoginModule required
>     useKeyTab=true
>     keyTab="/data/solr-indexer.keytab" 
>     storeKey=true
>     useTicketCache=true
>     debug=true
>     principal="solr-indexer@SOLRSERVER.COM"; 
>   };
>   ```
>
>   - The name of this section of the JAAS file. This name will be used with the `lww.jaas.appname` parameter.
>   - The location of the keytab file.
>   - The service principal name. This should be a different principal than the one used for Solr, but must have access to both Solr and Hive.
>
> - lww.jaas.appname
>
>   Used only when indexing to or reading from a Solr cluster secured with Kerberos.
>
>   This property provides the name of the section in the JAAS file that includes the correct service principal and keytab path.



```sql
// 失败
// Map类型 shell中一行一行复制进去
# 创建一个内部表
CREATE TABLE solr_map_inner (name string, movie map<string,string>) row format delimited fields terminated by '\t' collection items terminated by ',' map keys terminated by ':' lines terminated BY '\n' stored AS textfile;

# load 只能load到内部表内
load data inpath 'hdfs:/tmp/csv/film1.csv' overwrite into table solr_map_inner; 

CREATE EXTERNAL TABLE solr_map (name string, movie map<string,string>) 
 			row format delimited fields terminated by '\t'
 			collection items terminated by ','
 			map keys terminated by ':'
      STORED BY 'com.lucidworks.hadoop.hive.LWStorageHandler' 
      LOCATION '/tmp/solr' 
      TBLPROPERTIES('solr.zkhost' = '192.168.20.68:2181,192.168.20.65:2181,192.168.20.174:2181/solr', 
                    'solr.collection' = 'collection2',
                    'solr.query' = '*:*');
                    
INSERT INTO solr_map SELECT * FROM solr_map_inner; 

// 失败


// Struct类型
create table movie_score_inner(id string,name_s string,info struct<number_i:int,score_f:float>)row format delimited fields terminated by "\t" collection items terminated by ":" stored AS textfile;

load data inpath 'hdfs:/tmp/csv/movie.csv' overwrite into table movie_score_inner;

CREATE EXTERNAL TABLE solr_movie_score (id string,name_s string,info struct<number_i:int,score_f:float>) 
STORED BY 'com.lucidworks.hadoop.hive.LWStorageHandler'
LOCATION '/tmp/solr' 
TBLPROPERTIES('solr.zkhost' = '192.168.20.68:2181,192.168.20.65:2181,192.168.20.174:2181/solr', 
                    'solr.collection' = 'collection3',
                    'solr.query' = '*:*');
                    
 INSERT OVERWRITE TABLE solr_movie_score SELECT ms.* FROM movie_score_inner ms;                   
```



参考：

[官方文档](https://doc.lucidworks.com/lucidworks-hdpsearch/4.0.0/Guide-Jobs.html#hive-serde)