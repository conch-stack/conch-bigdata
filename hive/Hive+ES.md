## Hive integrate with ElasticSearch

下载jar包，导入Hive的auxlib中，需要重启Hive么？ - 需要



#### 注意

​	目前官方的ElasticSearch-hadoop不支持hadoop3.0版本  - 2019.06.11



```
// 建表
CREATE EXTERNAL TABLE es (id string, field1_s string, field2_i int) 
      STORED BY 'org.elasticsearch.hadoop.hive.EsStorageHandler' 
      LOCATION '/tmp/es' 
      TBLPROPERTIES('es.nodes' = '10.76.95.124:31483', 
                    'es.index.auto.create' = 'true',
                    'es.resource' = 'es/hive',
                    'es.read.metadata' = 'true',
                    'es.mapping.names' = 'id:_metadata._id');
 
# 如果只需要mapping es 的 _id 可使用如下 
'es.mapping.id' = 'goods_order_id', 

// 问题：
1. MetaException(message:java.lang.NoClassDefFoundError org/apache/commons/httpclient/HttpConnectionManager)
2. message:java.lang.NoClassDefFoundError org/apache/commons/httpclient/Credentials

// 目前官方的ElasticSearch-hadoop不支持hadoop3.0版本  - 2019.06.11
```



Hive 集成：https://www.elastic.co/guide/en/elasticsearch/hadoop/current/hive.html

建表语句的在es中的配置：https://www.elastic.co/guide/en/elasticsearch/hadoop/current/configuration.html



### Configuration

####  Required settings

- `es.resource`

  Elasticsearch resource location, where data is read *and* written to. Requires the format `<index>/<type>` (relative to the Elasticsearch host/port (see [below](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/configuration.html#cfg-network)))).

  ```
  es.resource = twitter/tweet   # index 'twitter', type 'tweet'
  
  # 查询所有索引所有的type
  'es.resource' = '_all/types'
  # 查询index索引下所有的type
  'es.resource' = 'index/'
  ```

  

#### Querying

- `es.query` (default none)

  Holds the query used for reading data from the specified `es.resource`. By default it is not set/empty, meaning the entire data under the specified index/type is returned. `es.query` can have three forms:

  - uri query
    - using the form `?uri_query`, one can specify a [query string](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/search-uri-request.html). Notice the leading `?`.
  - query dsl
    - using the form `query_dsl` - note the query dsl needs to start with `{` and end with `}` as mentioned [here](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/search-request-body.html)
  - external resource
    - if none of the two above do match, elasticsearch-hadoop will try to interpret the parameter as a path within the HDFS file-system. If that is not the case, it will try to load the resource from the classpath or, if that fails, from the Hadoop `DistributedCache`. The resource should contain either a `uri query` or a `query dsl`.

  ```
  1）以uri 方式查询
  es.query = ?q=98E5D2DE059F1D563D8565
  2）以dsl 方式查询
  es.query = { "query" : { "term" : { "user" : "costinl" } } }
  'es.query'='{"query": {"match_all": { }}}',
  3) external resource
  es.query = org/mypackage/myquery.json
  ```

  









####  Serialization

`es.batch.size.bytes` (default 1mb)

Size (in bytes) for batch writes using Elasticsearch [bulk](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/docs-bulk.html) API. Note the bulk size is allocated *per task*instance. Always multiply by the number of tasks within a Hadoop job to get the total bulk size at runtime hitting Elasticsearch.

`es.batch.size.entries` (default 1000)

Size (in entries) for batch writes using Elasticsearch [bulk](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/docs-bulk.html) API - (0 disables it). Companion to `es.batch.size.bytes`, once one matches, the batch update is executed. Similar to the size, this setting is *per task* instance; it gets multiplied at runtime by the total number of Hadoop tasks running.

`es.batch.write.refresh` (default true)

Whether to invoke an [index refresh](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/indices-refresh.html) or not after a bulk update has been completed. Note this is called only after the entire write (meaning multiple bulk updates) have been executed.

`es.batch.write.retry.count` (default 3)

Number of retries for a given batch in case Elasticsearch is overloaded and data is rejected. Note that only the rejected data is retried. If there is still data rejected after the retries have been performed, the Hadoop job is cancelled (and fails). A negative value indicates infinite retries; be careful in setting this value as it can have unwanted side effects.

`es.batch.write.retry.wait` (default 10s)

Time to wait between batch write retries that are caused by bulk rejections.

`es.ser.reader.value.class` (default *depends on the library used*)

Name of the `ValueReader` implementation for converting JSON to objects. This is set by the framework depending on the library (Map/Reduce, Hive, Pig, etc…) used.

`es.ser.writer.value.class` (default *depends on the library used*)

Name of the `ValueWriter` implementation for converting objects to JSON. This is set by the framework depending on the library (Map/Reduce, Hive, Pig, etc…) used.

