##### DStream

 一组RDD流

将连续的数据 持久化、离散化、进行批量处理

- 持久化：接收的数据暂存于executor中
- 离散化：按时间分片，形成处理单元
- 分片处理：分批处理



##### DStream Graph

反向**依赖关系**：保存DStream的依赖关系

对后期生成RDD Graph起关键作用



##### Transformation & output

- Transformation
  - reduce
  - count
  - map
  - flatMap
  - join
  - reduceByKey
- Output
  - print
  - saveAsObjectFile
  - saveAsTextFile
  - saveAsHadoopFiles: 将一批数据输出到hadoop的文件系统中，用批量数据的开始时间戳来命名
  - forEachRDD：允许用户对DStream的每一批数据对应的RDD本身做任意操作



### 架构

Master：优化，生成DStream Graph

Worker：执行job

Client：发送数据



### API

两种方式接受kafka数据

- Direct

- Receiver

### Structured Streaming

