## Flume整合Kafka

- 分区优化
  - 相同的用户，进入Kafka的同一分区，后续聚合等组件可提高效率（Spark Shuffle）
  - 官方Kafka Sink支持读取header中topic、key作为kafka的主题和分区key值，且会覆盖配置文件中的topic配置
  - 实现：添加一个filter往Flume Event中添加key header
    - 官方支持的拦截器
      - Search and Replace Interceptor： 查询，替换
      - Regex Extractor Interceptor：正在匹配，匹配到的值作为header
    - 自定义拦截器
      - 动手实现，可应对后续复杂的数据场景（深度过滤）