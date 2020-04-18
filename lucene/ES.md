## ElasticSearch

#### 更新

##### 原理？？？

​	Elasticsearch 已将旧文档标记为已删除，并增加一个全新的文档。尽管你不能再对旧版本的文档进行访问，但它并不会立即消失。当继续索引更多的数据，Elasticsearch 会在后台清理这些已删除文档

##### 处理冲突？？？

- 使用乐观并发控制
- 利用 _version 号来确保 应用中相互冲突的变更不会导致数据丢失
- 通过外部系统使用版本控制



#### 分片文档路由

shard = hash(routing) % number_of_primary_shards

- shard 		哪个分片， 也就是分片id
- routing      一个可变值，默认是文档的id
- hash          一个哈希函数，对rounting字段进行哈希计算生成一个数字
- number_of_primary_shards   主分片的数量，routing字段经过hash函数计算之后的值，将对 主分片的数量也就是 number_of_primary_shards 进行取与，之后得到的shard就是我们文档所在的分片的位置



##### 文档索引 _index



##### 文档ID生成 _id

- 手动
  - 结合业务id
- ES自动
  - 基于GUID，生成分布式不重复id

##### 文档类型 _type

​	ES6之后，一个索引下只能有一个类型

##### 文档源 _source



#### 分片与副本

##### 新建索引和删除单个文档 时的流程

![image-20190517173734892](assets/image-20190517173734892.png)

```
1. 先向 node 1 来一个请求这个请求可能是发送新建，索引或者删除文档等。
2. node 1 节点根据文档的_id 确定文档属于分片0， 请求被转发到node3 节点
3. node 3 在主分片执行了请求，如果主分片执行成功了，它将请求转发给node1 和node 2 节点。当所有的副分片都执行成功，node 3 将协调节点报告成功，并向客户端报告完成
```

- consistency 参数的值可以设为 one （只要主分片状态 ok 就允许执行写操作）,all（必须要主分片和所有副本分片的状态没问题才允许执行_写_操作）, 或quorum 。

- 默认值为 quorum , 即大多数的分片副本状态没问题就允许执行写操作



#### 读取文档

![image-20190517173934148](assets/image-20190517173934148.png)

```
1. 某个请求向node 1 发送获取请求 节点使用
2. 节点使用节点文档_ID来确定文档属于分片0， 分片0 的副本分片存在于所有的三个节点上，在这种情况下，他将请求转发到node 2
3. node 2 将文档返回给node 1 ，然后将文档返回给客户端
```

- 在每次处理读取请求时，协调结点在每次请求的时候都会轮训所有的副本片来达到负载均衡



#### 更新

![image-20190517174124012](assets/image-20190517174124012.png)

```
1. 客户端向node1 发送一个请求
2. 它将请求转发到主分片这个文档所在的Node 3
3. node 3从主分片检索文档，修改_Source json ，并且尝试重新索引主分片的文档。如果文档被另一个进程修改，他会重复步骤3 知道超过retry_on_conflict 次后放弃
4. node 3 成功更新文档，它将新版本的文档并行转发到node 1 和node 2 的副本分片，重新建立索引。所有副本分片都返回成功，node 3 向协调节点也返回成功，协调节点向客户端返回成功
5. update 操作也支持 新建索引的时的那些参数 routing 、 replication 、 consistency 和 timeout
```



#### 批量操作

mget 和 bulk API 的 模式类似于单文档模式。 协调节点知道每个文档的位置，将多个文档分解成每个文档的的多文档请求，并且将这些请求并行的转发到每个参与节点中

##### mget

![image-20190517174415306](assets/image-20190517174415306.png)

```
1. 客户端向node 1 发送一个mget请求
2. node 1 向每个分片构建多文档请求，并行的转发这些请求到托管在每个所需的主分盘或者副分片的节点上一旦收到所有的额回复，node 1 构建响应并将其返回给客户端
```

##### Bulk？？？

![image-20190517174457072](assets/image-20190517174457072.png)

```
1. 一个bulk请求请求到node 1
2. node 1 为每个节点创建一个批量请求，并将这些请求并行转发到每个包含主分片的节点
3. 主分片一个接一个按顺序执行每个操作。当每个操作成功时，主分片并行转发新文档（或删除）到副本分片，然后执行下一个操作。 一旦所有的副本分片报告所有操作成功，该节点将向协调节点报告成功，协调节点将这些响应收集整理并返回给客户端
```



#### 相关度评分原理

> Lucene 使用布尔模型 （Boolean Model）查找匹配文档，并主要借鉴了 词频（Term Frequency）、逆向文档评率 (Inverse Document Frequency) 和向量空间模型 (Vector Space Model)，同时加入协调因子、字段长度归一化、以及词或查询语句权重提升

##### 布尔模型

​	就是在查询中使用 **AND 、 OR 和 NOT （与、或和非）** 来匹配文档

##### 词频/逆向文档频率（TF/IDF）

​	一个文档的相关度评分部分取决于每个查询词在文档中的 权重*

- 词频 ： 词在文档中出现的频度是多少？ 频度越高，权重 越高
  - **tf(t in d) = √frequency 词 t 在文档 d 的词频（ tf ）是该词在文档中出现次数的平方根**

- 逆向文档频率： 词在集合所有文档里出现的频率是多少？频次越高，权重 越低
  - **vidf(t) = 1 + log ( numDocs / (docFreq + 1)) 词 t 的逆向文档频率（ idf ）是：索引中文档数量除以所有包含该词的文档数，然后求其对数**

- 字段长度归一值： 字段的长度是多少？ 字段越短，字段的权重 越高 
  - **norm(d) = 1 / √numTerms 字段长度归一值（ norm ）是字段中词数平方根的倒数**



#### 节点与分片规划

1. 控制每个分片占用的硬盘容量**不超过ES的最大JVM的堆空间设置**（一般设置不超过32G，参加下文的JVM设置原则），因此，如果索引的总容量在500G左右，那分片大小在16个左右即可；当然，最好同时考虑原则2。
2. 考虑一下node数量，一般一个节点有时候就是一台物理机，如果分片数过多，大大超过了节点数，很可能会导致一个节点上存在多个分片，一旦该节点故障，即使保持了1个以上的副本，同样有可能会导致数据丢失，集群无法恢复。所以， **一般都设置分片数不超过节点数的3倍**。
3. 主分片，副本和节点最大数之间数量，我们分配的时候可以参考以下关系：
   `节点数<=主分片数*（副本数+1）`

如果后期索引所对应的数据越来越多，我们还可以通过***索引别名***等其他方式解决。



##### 取消_all设置

_all是可以传入整个JSON用于查询，现在已经没必要了

如何设置禁用_all ？？？





##### 对比Solr

- 比Solr快

- 支持数据类型没有solr多
- 不依赖ZK
- 插件多



##### 嵌套类型

```
// Nested
// 创建mapping
{
  "mappings": {
    "_doc": {
      "properties": {
        "dynasty": {
          "type": "keyword"
        },
        "emperor": {
          "type": "text"
        },
        "woman": {
          "type": "nested",
          "properties": {
            "age": {
              "type": "integer"
            },
            "name": {
              "type": "text"
            },
            "address": {
              "type": "text"
            }
          }
        }
      }
    }
  }
}

// NESTED类型有限制的，嵌套数量不能多于50个
// 参考 https://www.cnblogs.com/ljhdo/p/4904430.html
```



##### 分词器

- 内置
  - standard 默认 lowcase、去除停用词，标点，支持中文，但是以单字为切割
  - simple  通过非字母字符分词，lowcase
  - whitesace 去重空格，不lowcase，不支持中文，不标准化
  - language 特定语言的分词器，不支持中文

- ik
  - https://github.com/medcl/elasticsearch-analysis-ik
  - 安装  ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.2.4/elasticsearch-analysis-ik-6.2.4.zip



##### 优化策略

```
# 选择合适的分片数和副本数
	节点数<=主分片数*（副本数+1）

# 控制分片分配行为
	调整分片分配器的类型，具体是在elasticsearch.yml中设置cluster.routing.allocation.type属性，共有两种分片器even_shard,balanced（默认）。even_shard是尽量保证每个节点都具有相同数量的分片，balanced是基于可控制的权重进行分配，相对于前一个分配器，它更暴漏了一些参数而引入调整分配过程的能力。
	ES内置十一个裁决者来决定是否触发分片调整
	
# 路由优化

# GC调优
	我们使用JVM的Xms和Xmx参数来提供指定内存大小
		1. 开启ElasticSearch上的GC日志
		2. 使用jstat命令
		3. 生成内存Dump
		
# 避免内存交换
	可以通过在elasticsearch.yml文件中的bootstrap.mlockall设置为true来实现，但是需要管理员权限，需要修改操作系统的相关配置文件。

# 控制索引合并
	ES还是提供了三种合并的策略tiered，log_byte_size，log_doc
	ES也提供了两种Lucene索引段合并的调度器：concurrent和serial
	
# 最佳字段查询： 
https://my.oschina.net/LucasZhu/blog/1501141
tie_breaker参数会让dis_max查询的行为更像是dis_max和bool的一种折中

# 多字段搜索：
https://my.oschina.net/LucasZhu/blog/1501150
multi_match:
	1. best_fields   最佳字段
	2. most_fields   多数字段
	3. cross_fields  跨字段
	4. phrase
	5. phrase_prefix

```



##### Store

```
哪些情形下需要显式的指定store属性呢？大多数情况并不是必须的。从_source中获取值是快速而且高效的。如果你的文档长度很长，存储 _source或者从_source中获取field的代价很大，你可以显式的将某些field的store属性设置为yes。缺点如上边所说：假设你存 储了10个field，而如果想获取这10个field的值，则需要多次的io，如果从_source中获取则只需要一次，而且_source是被压缩过 的。 
 
还有一种情形：reindex from some field，对某些字段重建索引的时候。从source中读取数据然后reindex，和从某些field中读取数据相比，显然后者代价更低一些。这些字段store设置为yes比较合适。

总结：
 如果对某个field做了索引，则可以查询。如果store：yes，则可以展示该field的值。
 但是如果你存储了这个doc的数据（_source enable），即使store为no，仍然可以得到field的值（client去解析）。
 所以一个store设置为no 的field，如果_source被disable，则只能检索不能展示。
```



##### 预加载 fielddata

```java
https://www.elastic.co/guide/cn/elasticsearch/guide/current/preload-fielddata.html

聚合的字段可以添加该属性

@Data
@Document(indexName = "goods", type = "product")   //要加,不然报空指针异常
public class Product implements Serializable {

    @Id
    private String id;

    private String name;

    private String desc;

    private int price;

    private String producer;

    /**
     * 聚合这些操作用单独的数据结构(fielddata)缓存到内存里了，需要单独开启
     * 要么这里设置
     * 要着通过rest配置
     * PUT goods/_mapping/product
     * {
     *   "properties": {
     *     "tags": {
     *       "type":     "text",
     *       "fielddata": true
     *     }
     *   }
     * }
     */
    @Field(fielddata = true, type = FieldType.Text)
    private List<String> tags;

}
```



##### 日期处理

```
参考：https://www.elastic.co/guide/en/elasticsearch/reference/7.2/mapping-date-format.html
问题：DateFormat.date_hour_minute_second -> yyyy-MM-dd'T'HH:mm:ss   多了一个 T

常用：
// 指定存储格式 自定义方式
@Field(type = FieldType.Date, format = DateFormat.custom,pattern ="yyyy-MM-dd HH:mm:ss")  
private String createTime;

其他：
// 数据格式转换，并加上8小时进行存储
@JsonFormat(shape =JsonFormat.Shape.STRING,pattern ="yyyy-MM-dd HH:mm:ss",timezone ="GMT+8")
```



##### SearchType

```
1. query and fetch
	5.3之前，现已废弃
	
2. query then fetch（ es 默认的搜索方式）
	第一步， 先向所有的 shard 发出请求， 各分片只返回文档 id(注意， 不包括文档 document)和排名相关的信息(也就是文档对应的分值)， 然后按照各分片返回的文档的分数进行重新排序和排名， 取前 size 个文档。
　第二步， 根据文档 id 去相关的 shard 取 document。 这种方式返回的 document 数量与用户要求的大小是相等的。
　　优点：
　　　　返回的数据量是准确的。
　　缺点：
　　　　性能一般，并且数据排名不准确。
　　翻译：
  		对所有分片执行查询，但仅返回足够的信息（而不是文档内容）。 然后对结果进行排序和排序，并根据它，仅询问相关分片的实际文档内容。返回的命中数与size中指定的完全相同，因为它们是唯一获取的命中数。当索引具有大量分片（不是副本，分片ID组）时，这非常方便。
  		
3. DFS query then fetch
	比第 2 种方式多了一个 DFS 步骤。
　　也就是在进行查询之前， 先对所有分片发送请求， 把所有分片中的词频和文档频率等打分依据全部汇总到一块， 再执行后面的操作、
　　优点：
　　　　返回的数据量是准确的
　　　　数据排名准确
　　缺点：
　　　　性能最差【 这个最差只是表示在这四种查询方式中性能最慢， 也不至于不能忍受，如果对查询性能要求不是非常高， 而对查询准确度要求比较高的时候可以考虑这个】
　　　　
DFS:
	DFS 其实就是在进行真正的查询之前， 先把各个分片的词频率和文档频率收集一下， 然后进行词搜索的时候， 各分片依据全局的词频率和文档频率进行搜索和排名
```



##### scroll 游标查询

```
GET /old_index/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], 
    "size":  1000
}

// 保持游标查询窗口一分钟。
// 关键字 _doc 是最有效的排序顺序。

// 这个查询的返回结果包括一个字段 _scroll_id`， 它是一个base64编码的长字符串 ((("scroll_id"))) 。 现在我们能传递字段 `_scroll_id 到 _search/scroll 查询接口获取下一批结果：

GET /_search/scroll
{
    "scroll": "1m",  // 注意再次设置游标查询过期时间为一分钟。
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs="
}
// 这个游标查询返回的下一批结果。 尽管我们指定字段 size 的值为1000，我们有可能取到超过这个值数量的文档。 当查询的时候， 字段 size 作用于单个分片，所以每个批次实际返回的文档数量最大为 size * number_of_primary_shards 。

// 当没有更多的结果返回的时候，我们就处理完所有匹配的文档了。 仍会返回200成功，只是hits裡的hits会是空list

特点：
	1. 查询过程中，数据（索引数据或者文档data）变化时，对游标没影响
	2. 在当前游标方式查询过程中，这部分变化的数据不能够查询出来，因为它就是一个快照信息
	
优化：
	在一般场景下，scroll通常用来取得需要排序过后的大笔数据，但是有时候数据之间的排序性对我们而言是没有关系的，只要所有数据都能取出来就好，这时能够对scroll进行优化
  1. 使用_doc去sort得出来的结果，这个执行的效率最快，但是数据就不会有排序，适合用在只想取得所有数据的场景
    POST 127.0.0.1:9200/my_index/_search?scroll=1m
    {
        "query": {
            "match_all" : {}
        },
        "sort": [
            "_doc"
            ]
        }
    }
  2. 清除scroll
  虽然我们在设置开启scroll时，设置了一个scroll的存活时间，但是如果能够在使用完顺手关闭，可以提早释放资源，降低ES的负担
    DELETE 127.0.0.1:9200/_search/scroll
    {
        "scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAdsMqFmVkZTBJalJWUmp5UmI3V0FYc2lQbVEAAAAAAHbDKRZlZGUwSWpSVlJqeVJiN1dBWHNpUG1RAAAAAABpX2sWclBEekhiRVpSRktHWXFudnVaQ3dIQQAAAAAAaV9qFnJQRHpIYkVaUkZLR1lxbnZ1WkN3SEEAAAAAAGlfaRZyUER6SGJFWlJGS0dZcW52dVpDd0hB"
    }
```



##### 问题：

主分片在所有创建时就已经确定了，如果无法水平扩容主分片：主分片的数量决定了能存储数据的大小(实际大小取决于你的数据、硬件和使用场景)



##### 删除

官方推荐两个做法：

- 按索引删除（索引按时间生成）

  - 查询就得模糊匹配了
  - GET  cl_user_daily_*/cl_user/_search
    GET  cl_user_daily_20171105/cl_user/_search

- 使用删除API Delete By Query API

  - 定时任务+delete_by_query

  - ```
    POST twitter/_delete_by_query
    {
      "query": { 
        "match": {
          "message": "some message"
        }
      }
    }
    
    // 批量删除的时候，可能会发生版本冲突 强制
    POST twitter/_doc/_delete_by_query?conflicts=proceed
    {
      "query": {
        "match_all": {}
      }
    }
    ```

- 1）删除索引是会立即释放空间的，不存在所谓的“标记”逻辑。

- 2）删除文档的时候，是将新文档写入，同时将旧文档标记为已删除。 磁盘空间是否释放取决于新旧文档是否在同一个segment file里面，因此ES后台的segment merge在**合并segment file的过程中有可能触发旧文档的物理删除**。

```
# 如何仅保存最近100天的数据
- 1）delete_by_query设置检索近100天数据； 
- 2）执行forcemerge操作，手动释放磁盘空间。


# 脚本：
#!/bin/sh
curl -H'Content-Type:application/json' -d'{
    "query": {
        "range": {
            "pt": {
                "lt": "now-100d",
                "format": "epoch_millis"
            }
        }
    }
}
' -XPOST "http://192.168.1.101:9200/logstash_*/
_delete_by_query?conflicts=proceed"

# merge脚本如下
#!/bin/sh
curl -XPOST 'http://192.168.1.101:9200/_forcemerge?
only_expunge_deletes=true&max_num_segments=1'
```

更好的方式：

```
Curator：https://github.com/elastic/curator

```



### 参考

[入门与原理](http://www.cnblogs.com/wangshouchang/p/8049492.html)

[相关读](https://www.cnblogs.com/wangshouchang/p/8667123.html)

[索引，压缩，快速](https://www.infoq.cn/article/database-timestamp-02)

[优化](https://blog.51cto.com/13527416/2132270?source=dra)

[官方压测](https://elasticsearch-benchmarks.elastic.co)

[写入优化](https://www.easyice.cn/archives/207)

[Solr对比ES](http://solr-vs-elasticsearch.com/)

