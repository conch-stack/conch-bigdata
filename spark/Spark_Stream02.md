## Spark Streaming 优化

- 自己维护Kafka的offect

- 现在Kafka的分区消费速率，防止高峰期增加Spark集群资源压力

  - ```java
    sparkConf.set("spark.streaming.kafka.maxRatePerPartition", "10000");
    ```

- 注册序列化Kryo，提速网络传输、减少内存消耗

  - ```java
    sparkConf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer");
    sparkConf.set("spark.kryo.registrator", "com.djt.stream.registrator.MyKryoRegistrator");
    ```

