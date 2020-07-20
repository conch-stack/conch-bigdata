### 序列化工具

- FackBook Thrift  语言支持多

- Google Protobuf 压缩比大、解析速度快

- Apache Avro 
  - 动态类型： 不需要生成代码，将数据与Schema存放一起，方便构建通用的数据处理系统
  - 未标记的数据：读取Avro数据时 Schema是已知的，使得编码到数据中的类型信息变少，使得序列化后的数据量变少
  - 不需要显式指定域编号：处理数据时新旧Schema都是已知的，通过使用字段名即可解决兼容性问题



### IDL(Interface description language)

- TODO



### 文件存储格式

- 行式存储 
- 列式存储

| 对比     | 行式存储                                                 | 列式存储                                                     |
| :------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| 写性能   | 写入一次完成，性能高                                     | 将行记录拆分为单列存储，写次数多，时间花费大                 |
| 读性能   | 读取整行数据顺序读，性能高，读取几列数据时，需遍历无关列 | 读取整行数据需进行拼装，性能低，读取几列数据时，无需遍历无关列 |
| 数据压缩 | 每行数据存储在一起，压缩比低                             | 以列为单位存储数据，使得类型相同的数据存放在一起，对与压缩算法友好，压缩比高 |
| 典型代表 | Text File、Sequence File                                 | ORC(Hortonworks)、Parquet(Twitter)、CarbonData(华为)         |
| 描述     | 已行为单位，连续读取或写入同一行的所有列                 | 写数据时将数据拆分为列，以列为单位存储(相同列存储在一起)，读数据时，分别读取对应列，并**拼装成行** |



#### Hadoop文件读取写入模块抽象

- InputFormat
- OutputFormat

| 文件格式 | InputFormat             | OutputFormat             |
| -------- | ----------------------- | ------------------------ |
| Text     | TextInputFormat         | TextOutputFormat         |
| Sequence | SequenceFileInputFormat | SequenceFileOutputFormat |
| ORC      | OrcInputFormat          | OrcOutputFormat          |
| Parquet  | ParquetInputFormat      | ParquetOutputFormat      |



#### 行式存储

##### Sequence File

​	Hadoop中提供的简单Key/Value二进制行式存储格式，用于存储文本格式无法存储的数据：二进制对象、图片、视频等

- 物理存储分块
- 未压缩的Sequence File格式
- 行级压缩的Sequence File格式
- 块级压缩的Sequence File格式



#### 列式存储

##### Parquet

​	为支持复杂的嵌套类型数据，配和高效压缩和编码技术，降低存储空间，提供IO效率

​	先按行切分数据，形成Row Group，在每个Row Group中以列为单位存储数据

- Row Group：一组行数据，内部以列为单位存储这些行。当往Parquet中写数据时，Row Group中的数据会缓冲到内存中，直到达到预定大小，之后刷新到磁盘；当从Parquet中读取数据时，每个Row Group可作为一个独立的数据单元由一个任务处理，通常大小在10MB到1G之间。
- Column Chunk：Row Group中包含多个Column Chunk，Column Chunk由若干Page构成，读取数据时，可选择性跳过不感兴趣的Page
- Data Page：数据压缩的基本单元，数据读取的最小单元，通常一个Page大小在 8KB 到 100KB之间。

相比ORC File，Parquet能更好的适配各种查询引擎(Hive, Impala)、计算框架(MapReduce，Spark)、序列化框架



###### 1. 存储格式

​	定义Parquet内部存储格式、数据类型等，提供通用的和语言无关的列式存储格式：由 parquet-format实现

###### 2. 对象模型转换器

​	将外部对象模型映射为Parquet内部类型：由 parquet-mr实现；目前支持Hive、Pig、MapReduce、Cascading、Crunch、Impala、Thrift、Protobuf、Avro

###### 3. 对象模型

​	内存中数据表现的形式：Thrift、Protobuf、Avro均属于对象模型

![image-20190327162651005](../../assets/image-20190327162651005.png)



#### 压缩：见Hadoop篇MapReduce

Snappy压缩

​	

​	

​	