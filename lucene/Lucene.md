## Lucene

#### Scalable, High-Performance Indexing

- over [150GB/hour on modern hardware](http://home.apache.org/~mikemccand/lucenebench/indexing.html)
  - 每秒可以索引40M
- small RAM requirements -- only 1MB heap
  - 参考 对象重用
- incremental indexing as fast as batch indexing
- index size roughly 20-30% the size of text indexed



#### 内存

- 添加文档 -> 内存 -> 文档到达一定数量或内存满(默认16M) 会触发flash到磁盘->  flash会触发多Segment合并(后台异步合并)
- DocComsumer在处理文档时直接将数据持久化到文件，只有在处理中时需要使用RAM
  - 其他的Comsumer：FreqProxTermsWriter和NormsWriter会缓存在内存中，当有新的Segment制造出的时候才会flash到磁盘
- 查询时，需要将磁盘上的索引信息读到内存





#### 索引创建过程

参考：Annotated-Lucene源码剖析





#### Lucene数据存储

##### lucene基本概念

- **segment** : lucene内部的数据是由一个个segment组成的，写入lucene的数据并不直接落盘，而是先写在内存中，经过了refresh间隔，lucene才将该时间段写入的全部数据refresh成一个segment，segment多了之后会进行merge成更大的segment。lucene查询时会遍历每个segment完成。由于lucene* 写入的数据是在内存中完成，所以写入效率非常高。但是也存在丢失数据的风险，所以Elasticsearch基于此现象实现了translog，只有在segment数据落盘后，Elasticsearch才会删除对应的translog。

- **doc** : doc表示lucene中的一条记录
- **field** ：field表示记录中的字段概念，一个doc由若干个field组成。
- **term** ：term是lucene中索引的最小单位，某个field对应的内容如果是全文检索类型，会将内容进行分词，分词的结果就是由term组成的。如果是不分词的字段，那么该字段的内容就是一个term。
- **倒排索引**（inverted index）: lucene索引的通用叫法，即实现了term到doc list的映射。
- **正排数据**：搜索引擎的通用叫法，即原始数据，可以理解为一个doc list。
- **docvalues** :Elasticsearch中的列式存储的名称，Elasticsearch除了存储原始存储、倒排索引，还存储了一份docvalues，用作分析和排序。



##### 磁盘缓存

- IndexWriter添加的Document在内存中,此时Lucene查询不可见
  - ES同时添加translog，用于保证异常发送时数据不丢失，ES每5秒或每次请求操作结束前，会强制刷新translog日志到磁盘：(会牺牲一定性能以换去一致性)
    - index.translog.durability=async
  - ES默认30分钟进行一次主动flash或translog文档大小大于512M时
    - flash_threshold_period
    - flash_threshold_size

- ES默认1秒refresh内存到文件系统缓存，这样lucene就可实时可搜索了

  - 如果对写入性能要求高，而对查询实时要求相对低，可选择增加refresh时间 10s

- 这样会在文件系统缓存中存在很多Segment，大小不等

  - Segment合并，后台线程运行(归并线程)
    - 合并多个小Segment到大Segment，大的Segment合并期间，Lucene查询不可见，合并完成Commit后，会逐渐将被合并的几个小Segment无读取和查询打开后，删除掉，替换为从大的Segment读取
    - 归并线程数 Math.min(3, Runtime.getRuntime().availableRrocessors() / 2)

  



##### 语言处理组件(linguistic processor)

​	主要是对得到的词元(Token)做一些同语言相关的处理。

​	对于英语，语言处理组件(Linguistic Processor)一般做以下几点：

- 变为小写(Lowercase)。
- 将单词缩减为词根形式，如“cars”到“car”等。这种操作称为：stemming。
- 将单词转变为词根形式，如“drove”到“drive”等。这种操作称为：lemmatization





##### 源码核心类

- IndexWriter  只能开启一个

- DocumentWriter

- DocumentsWriterPerThreadPool
  - ThreadState
    - DocumentsWriterPerThread  并发处理索引
- DefaultIndexingChain
  - FreqProxTermsWriter  词
  - TermVectorsConsumer 词向量
  - StoredFieldsWriter 存储Field中的值
  - CompressingStoredFieldsWriter：构造函数会创建对应的.fdt和.fdx文件，并写入相应的头信息(参考：Lucene存储参考)
  - processField() 方法处理字段
    - Invert indexed fields
      - invert() 方法 处理 反向索引
    - Add stored fields
      - writeField() 方法 处理 字段存储
    - DocValues
      - indexDocValue() 方法
  - flash() 



##### Integer的问题

Lucene6后加入的点(point)的概念：来表示数值类型

```
IndexableFieldType:

/**
 * If this is positive, the field is indexed as a point.
 * 	返回点的维数		
 */
public int pointDimensionCount();

/**
 * The number of bytes in each dimension's values.
 *  返回点中数值类型的字节数
 */
public int pointNumBytes();
```

point提供了精确、范围查询的便捷方法

点：有空间概念：维度；一维 一个值，二维 两个值...

可以表示地理空间



##### 预定义的字段子类

- TextField
- StringField
- IntPoint
- LongPoint
- FloatPoint
- DoublePoint
- SortedDocValuesField
- SortedSetDocValuesField
- NumericDocValuesField
- SortedNumericDocValuesField
- StoredField

如果上述不能满足，可自行声明 Field 和 FieldType



##### 对象重用

> Field提供setXXXValue()方法
>
> 多个文档，只需要不断修改值就可以了：体现了1MB heap



> 加入索引时，每个数据记录可重用 Document对象



##### 索引更新删除

> 删除
>
> ​	根据Term、Query找到相应文档ID，同时删除索引信息，再删除对应文档
>
> 更新
>
> ​	先删除，在新增doc
>
> 只可以根据索引的字段进行更新(要基于Term和Query找ID)



##### 相似度

​	BM25和Vector Space Model算法



##### 词法分析、语言处理、语法分析



##### 问题

Lucene严格限制：

​	单个索引中最大文档数：约 21.4亿



#### 参考

[Lucene原理细讲](https://blog.csdn.net/liuhaiabc/article/details/52346493)

[Lucene存储原理](https://www.cnblogs.com/zxf330301/articles/8728369.html)

[Lucene存储参考](https://elasticsearch.cn/article/6178)

[Lucene列式存储格式DocValues](https://mp.weixin.qq.com/s?__biz=MzI4Njk3NjU1OQ==&mid=2247484096&idx=1&sn=d613665e9efcb3ec708b866571e66a90&chksm=ebd5fd80dca27496f03c5449c63064fab2ebde19860e95219e946f46e60206b18947f41d8c1d&scene=21#wechat_redirect)

[lucene字典实现原理倒排索引实现——FST](http://www.cnblogs.com/bonelee/p/6226185.html)

[FST测试]([http://examples.mikemccandless.com/fst.py?terms=&cmd=Build+it](http://examples.mikemccandless.com/fst.py?terms=&cmd=Build+it))

[Lucene源码分析](https://blog.csdn.net/conansonic/article/details/51886014)