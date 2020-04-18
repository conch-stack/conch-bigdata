## Spark_01

```sql
-- 运行spark例子
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-cluster examples/jars/spark-examples_2.11-2.3.2.3.1.0.0-78.jar 10

-- RDD依赖
1. 宽依赖：存在shuffle的操作
2. 窄依赖：一对一的操作
```



##### 测试Jieba UDF分词

```sql
-- 添加数据
use zjj;
create table news_seg(sentence string) stored as textfile;
load data inpath '/data/allfile/allfiles.txt' into table news_seg;

-- 测试
select split(regexp_replace(sentence, ' ', ''), '##@@##')[0] as sentence, split(regexp_replace(sentence, ' ', ''), '##@@##')[1] as label from news_seg limit 1;

create table news_noseg as
select split(regexp_replace(sentence, ' ', ''), '##@@##')[0] as sentence, split(regexp_replace(sentence, ' ', ''), '##@@##')[1] as label from news_seg;

select * from news_noseg limit 10;
```



##### HDP3.0 中Hive和Spark打通模式变化

```
参考：
https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.1.0/integrating-hive/content/hive_hivewarehouseconnector_for_handling_apache_spark_data.html
```



##### 练习1

```scala
/* 
-- 表信息
-- order_product_prior 订单产品信息表
-- order_id, product_id, add_to_cart_order, reordered

-- orders 订单表：订单和用户的信息  dow=day of week
-- order_id,user_id,eval_set,order_number(购买订单的顺序),order_dow(星期几),order_hour_of_day（一天中的哪个小时）,days_since_prior_order(距离上一个订单的时间) 
*/

package com.dbzjz.demo.sparkdemo

import org.apache.spark.sql.DataFrame
import org.apache.spark.sql.functions._

object SimpleFeature {

  def Feat(priors:DataFrame, orders:DataFrame): DataFrame = {

    // 每个商品的商品销量；每个商品再次购买商品的销量；每个商品再次购买占总购买的比率 avg([0,1,0,1,1,...])
    val priorsCnt = priors
      .selectExpr("product_id", "cast(reordered as int)")
      .groupBy("product_id")
      .agg(
        count("product_id").as("prod_cnt"),
        sum("reordered").as("prod_sum_rod"),
        avg("reordered").as("prod_rod_rate"))


    // 每个用户平均购买订单的间隔周期；每个用户的总订单数量 todo 处理空值
    val tmp = orders
      .selectExpr("*", "if(days_since_prior_order == '',0,days_since_prior_order) as dspo")
      .drop("days_since_prior_order") // 删除一列

    val tmp1 = tmp
      .selectExpr("user_id", "order_id", "cast(dspo as int)")
      .groupBy("user_id")
      .agg(
        avg("dspo").as("order_cycle"),
        count("order_id").as("order_cnt")
      )

    // 每个用户购买的product商品去重后的集合数据
    val joinOn = priors.col("order_id") === orders.col("order_id")
    val tmp2 = orders
    //  .join(priors, "order_id")
      .join(priors, joinOn)

    // 法一 hivesql
    val tmp3 = tmp2
      .selectExpr("user_id", "concat_ws(',', collect_set(product_id))")
      .groupBy("user_id")

    // 法二 rdd
    import priors.sparkSession.implicits._
    val tmp4 = tmp2.select("user_id", "product_id")
      .rdd
      .map(x => (x(0).toString, x(1).toString))
      .groupByKey()
      .mapValues(_.toSet.mkString(","))
      .toDF("user_id", "prods")

    // 每个用户总商品数量以及去重后的商品数量
    val tmp5 = tmp2.select("user_id", "product_id")
      .rdd
      .map(x => (x(0).toString, x(1).toString))
      .groupByKey()
      .mapValues{values=>
        val valuesSet = values.toSet
        (values.size, valuesSet.size)
      }.toDF("user_id", "tuple")
      .selectExpr("user_id", "tuple._1 as prod_cnt", "tuple._2 as prod_dist_cnt")

    // 每个用户购买的平均每个订单的商品数量(hive已做过)
    // 1）每个订单的商品数量
    val tmp6 = priors.groupBy("order_id").count()
    // 2) 每个用户，订单的avg(商品数量)
    val tmp7 = tmp6.join(orders, "order_id")
      .groupBy("user_id")
      .avg("count")
      .withColumnRenamed("avg(count)", "per_order_avg_prod_cnt")

    val userFeature = tmp1.join(tmp4, "user_id")
      .join(tmp5, "user_id")
      .join(tmp6, "user_id")
      .selectExpr("user_id",
        "order_cycle as u_order_cycle", "order_cnt as u_order_cnt",
        "prods as u_dist_prod_set",
        "prod_cnt as u_prod_cnt", "prod_dist_cnt as u_prod_dist_cnt",
        "per_order_avg_prod_cnt as u_per_order_avg_prod_cnt"
      )

    /**
      * user and product Feature : cross feature 交叉特征
      *   特征分类：count、rate(比率)、avg(mean)、std(方差)、max、min
      *
      *
      * */
    userFeature
  }
}

```



##### Submit param

```shell
./bin/spark-submit \
--class com.hadoop123.example.SparkInvertedIndex \
--master yarn-cluster \
--deploy-mode cluster \
--driver-memory 3g \
--num-executors 3 \
--executor-memory 4g \
--executor-cores 4 \
--queue spark \
SparkInvertedIndex.jar
```



##### 优化：

```shell
1. SparkConf和SparkSubmit中可设置

--driver-memory：1-4G：collect时会将数据缓存到AM中
--num-executors： Executor的数量：一般在5，10，20个之间
--executor-memory：建议4-8G，（4*10总内存）可参考数据量的10倍配置
--executor-cores：根据实际情况：1，3（建议），5个
--queue spark
--spark.default.parallelism：每个Stage的默认Task数量：建议500-1000之间，默认一个HDFS的Block对应一个Task，默认值偏少，不能有效利用资源
--spark.storage.memoryFraction：设置RDD持久化数据在executor内存中能占的比例，默认0.6，即executro内存的60%的内存可用于持久化RDD数据；建议有较多持久化操作时，可设置高一些，超出内存会导致频繁GC
--spark.shuffle.memoryFraction：聚合操作exector内存的比例，默认0.2；建议若持久化操作较少，但是shuffle较多时，可降低持久化内存占比，提高shuffle操作的内存占比

2. RDD复用

3. 避免重复创建相同数据的RDD

4. 缓存中间计算结果，加速故障恢复cache() persist() 对应缓存持久化级别：内存、磁盘、内存-磁盘等等

5. 尽量减少Shuffle，很难避免
	Broadcast+map的join操作不会导致shuffle，但是前提是广播的RDD数据量较小？2G内？
	
6. 尽量使用reduceByKey或aggregateByKey代替groupByKey，他们会预聚合	

7. 使用Kryo优化序列化性能(10倍)
```

