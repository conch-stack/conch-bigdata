## HBase-Spark

##### HBase and Spark Streaming

- 使用HBase存储Spark Streaming的中读取Kafka的offect信息

- 使用HBase累加实现流式窗口数据的累加

  - ```java
    table.incrementColumnValue(rowkey,cf,cq,value)
    ```

- 动态表名
  - 使用脚本提前创建HBase的表，半年、一年的创建

