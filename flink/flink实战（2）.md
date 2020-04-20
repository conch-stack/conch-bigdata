## Flink实战（2）

> 算子

- map

- flatMap

- filter

- keyBy:

  - POJO对象或者Tuple2以上对象
  - 容易出现数据倾斜
  - key类型限制：
    - 不能是没有覆盖hashcode方法的POJO
    - 不能是数组

- reduce

- fold  !! 已废弃

- aggregations

- sum/min/minBy/max/maxBy  reduce 特例

  - min(返回指定字段的最大值，其他不变)
  - minBy(返回整个元素)

- Interval join

  - KeyedStream,KeyedStream -> DataStream

  - 在给定周期内，按照指定的key对两个KeyedStream进行join操作

  - 把一定时间范围内的相关的分组数组拉成一个宽表

    ```java
    keyedStream1.intervalJoin(keyedStream2)
                     .between(Time.milliseconds(-2), Time.milliseconds(2))
                     .upperBoundExclus()
                     .lowerBoundExclus()
                     .process(new ProcessJoinFunction<A a,B b, Context ctx, Colector<C> out) {
                         out.collect(new C());
                     }
    ```

- connect & union

  - connect后生成ConnectedStreams
    - 对两个流的数据应用不同处理方式，并且双流之间可以共享状态（比如计数）
    - 这在第一个流的输入会影响第二个流时，会非常有用
    - 只能连接两个流
    -  CoMap
    - CoFlatMap  类似map和flatMap，只不过在ConnectedStream上用
    - 一起运算，节约资源？？？
  - union合并多个流，新流会包含所有流的数据
    - DataStream* -> DataStream
    - 可合并多个流
  - !! connect连接的两个流类型可以不一致，而union连接的流的类型必须一致

- split & select  !! 已废弃

  - 先拆，再选
  - split:
    - DataStream -> SplitStream
    - 按照指定标准将DataStream拆分成多个流
  - select:
    - SplitStream -> DataStream
    - 搭配split使用，从SplitStream中选择一个或多个流

- project

  - 从Tuple中选择属性子集 类似 mongodb聚合的project操作
  - 从Tuple中选择属性子集 类似 mongodb聚合的project操作
  -  只支持Tuple类型的event数据
  - 仅限JavaAPI
  - intput.project(0,2): 只要Tuple的 第一个和第三个元素









