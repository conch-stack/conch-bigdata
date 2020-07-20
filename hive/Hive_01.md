## Hive01



hdp defaultl warehouse:  /warehouse/tablespace/managed/hive



```sql
create database zjj;

use zjj;

create table article(
	sentence string
);

insert into article values('this is test one'),('what are you talking about!');

select sentence from article limit 1;

-- word count
select word, count(1) as count 
from (
select explode(split(sentence, ' ')) as word 
from article) t 
group by word;

-- TOP 10
select word, count(1) as count from (select explode(split(sentence, ' ')) as word from article) t group by word sort by count DESC limit 10;

-- 问题
-- order by 会将所有数据放到一个ReduceTask中
	-- 在strict模式(hive.mapred.mode=strict)下，order by子句后面必须有limit子句。如果设置hive.mapred.mode=nonstrict，limit子句不一定需要。原因是为了对所有结果进行整体的排序，必须使用一个reducer来对最后的结果进行排序。如果结果的总行数太大，单个reducer可能需要很长的时间完成。
	-- 在Hive 2.1.0及以上版本，支持对null值进行排序，升序排序是null值被排在第一位，降序时null值被排在最后。
-- sort by 在每个reduceTask中进行排序:结果是部分有序

-- 外部表： 可以直接获取到你上传的文件的内容
sudo -u hdfs hdfs dfs -put article.txt /data/external
create external table article_external(sentence string) location '/data/external';
select * from article_external;
sudo -u hdfs hdfs dfs -put article1.txt /data/external
select * from article_external;

-- 分区 按照时间天
create table article_partition (
	sentence string
)
partitioned by(dt string)

-- 添加分区 20190613
insert into table article_partition partition(dt='20190613') values('this is the first partition of artitle');

-- 查看
select * from article_partition;
show partitions article_partition;

-- 分区可设置多字段

-- 导入udata数据
use zjj;
create table udata(
user_id string,
item_id string,
rating string,
`timestamp` string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
stored AS textfile 
;
load data local inpath '/home/hdfs/csvs/hivedata/u.data' into table udata;
-- 问题：待解决
-- 	load 本地文件报错 No files matching path file 
-- 修改文件的权限，无法解决
-- 上传到server所在服务器，无法解决
-- 尝试使用beelin连接，依旧不能解决：待解决
beeline jdbc:hive2://apollo01.dev.zjz:2181,apollo02.dev.zjz:2181,apollo03.dev.zjz:2181/default;password=hdfs;serviceDiscoveryMode=zooKeeper;user=hdfs;zooKeeperNamespace=hiveserver2
Enter username : hive
Enter password : hive

-- 上传HDFS
sudo -u hdfs hdfs dfs -mkdir /data/udata
sudo -u hdfs hdfs dfs -put /home/hdfs/csvs/hivedata/u.data /data/udata
load data inpath '/data/udata/u.data' into table udata;

-- 分桶 TODO数据倾斜
-- 强制分桶
set hive.enforce.bucketing=true;  
create table bucket_udata(
user_id int,
item_id string,
rating string,
`timestamp` string
)
row format delimited fields terminated by '\t'
clustered by(user_id) into 3 buckets
stored as textfile;

insert overwrite table bucket_udata
select cast(user_id as int) as user_id, item_id, rating, `timestamp` from udata;

-- 采样 1/3
select * from bucket_udata tablesample(bucket 1 out of 3 on user_id);

-- (bucket x out of y)
-- y 需要是table的总bucket数的倍数或因子
-- x 表示从那个bucket开始取，计算规则：1%16 取余得1

-- 采样创表
create table tmp as select * from bucket_udata tablesample(bucket 1 out of 3 on user_id);

-- 采样90%
select * from udata where user_id%10>0;

-- 添加订单，产品数据
-- order_id, product_id, add_to_cart_order, reordered
create table order_product_prior (
order_id string,
product_id string, 
add_to_cart_order string, 
reordered string  
)
row format delimited fields terminated by ','
stored as textfile;

load data inpath '/data/order/order_products__prior.csv' into table order_product_prior;

-- orders 订单表：订单和用户的信息  dow=day of week
-- order_id,user_id,eval_set,order_number(购买订单的顺序),order_dow(星期几),order_hour_of_day（一天中的哪个小时）,days_since_prior_order(距离上一个订单的时间) 
create table orders(
order_id string,
user_id string,
eval_set string,
order_number string,
order_dow string,
order_hour_of_day string,
days_since_prior_order string
)
row format delimited fields terminated by ','
stored as textfile;

load data inpath '/data/order/orders.csv' into table orders;

-- 查询每个用户购买的商品数量 --left outer join
select o.user_id,count(opp.product_id) as prod_cnt
from order_product_prior opp
left outer join orders o on o.order_id == opp.order_id  
group by o.user_id
order by prod_cnt desc
limit 20;
/* rs1
+------------+-----------+
| o.user_id  | prod_cnt  |
+------------+-----------+
| 201268     | 3725      |
| 129928     | 3638      |
| 164055     | 3061      |
| 186704     | 2936      |
| 176478     | 2921      |
| 182401     | 2907      |
| 137629     | 2901      |
| 33731      | 2888      |
| 108187     | 2760      |
| 4694       | 2735      |
| 79106      | 2631      |
| 17738      | 2596      |
| 60694      | 2579      |
| 5360       | 2561      |
| 13701      | 2547      |
| 72136      | 2517      |
| 181991     | 2463      |
| 57546      | 2445      |
| 23832      | 2429      |
| 52008      | 2386      |
+------------+-----------+
*/
select o.user_id,count(opp.product_id) as prod_cnt
from order_product_prior opp
left outer join orders o on o.order_id == opp.order_id
group by o.user_id
distribute by user_id
sort by prod_cnt desc
limit 20;
/*
+------------+-----------+
| o.user_id  | prod_cnt  |
+------------+-----------+
| 201268     | 3725      |
| 129928     | 3638      |
| 164055     | 3061      |
| 186704     | 2936      |
| 176478     | 2921      |
| 182401     | 2907      |
| 137629     | 2901      |
| 33731      | 2888      |
| 108187     | 2760      |
| 4694       | 2735      |
| 79106      | 2631      |
| 17738      | 2596      |
| 60694      | 2579      |
| 5360       | 2561      |
| 13701      | 2547      |
| 72136      | 2517      |
| 181991     | 2463      |
| 57546      | 2445      |
| 23832      | 2429      |
| 52008      | 2386      |
+------------+-----------+
*/
-- 一模一样，但是第二种在大数据量情况下，效果更好

-- 笛卡尔积
create table cartesian_product_01(
user_id string
);
insert into table cartesian_product_01 values('1'), ('2'), ('3'), ('4'), ('5'); 

-- 没有 on 的 join 就是笛卡尔积 
select * 
from (select * from cartesian_product_01) t1
join 
(select * from cartesian_product_01) t2

-- reduce优化
	-- 制一个job会有多少个reducer来处理，依据多少的是文件的总大小，默认1g
	set hive.exec.reducers.bytes.per.reducer=<number>
	set hive.exec.reducers.max=<number>
	set mapreduce.job.reduces=<number>
	-- 全局的，和顺序无关，只在我当前cli生效
	
-- 列转行
select user_id,
concat_ws(',',collect_list(order_id)) as order_value 
from col_lie
group by user_id
limit 10;

-- 行转列
select user_id,order_value,order_id
from lie_col
lateral view explode(split(order_value,',')) num as order_id
limit 10;
```



优化：

```sql
-- join优化、并行度优化、MR优化、倾斜

1. 分区裁剪 partition
	where中的分区条件，会提前生效，不必特意做子查询，直接join和groupby
2. 笛卡尔积
	join的时候不添加on条件或者无效的on条件，Hive只能使用一个Reducer来完成笛卡尔积
3. Map join
	MAPJOIN(tablelist) 必须是小表，不要超过1G，或者50万条记录：会将数据放入内存，分发到所有节点？？？
4. Union all/distinct
	先做union all，在做join或者groupby等操作，可以有效减少MR过程，尽管是多个select，最终只有一个mr
5. set hive.groupby.skewindata=true; 一个map reduce拆成两个MR：可处理数据倾斜问题
	-- 一个mapreduce 推荐
		a.order_id = b.ord_id
		a.order_id = c.ord_id
	-- 多个mapreduce
		a.order_id = b.ord_id 
		c.product_id = b.prod_id
6. 广播
	大表提速：小表会加载如内存，Join时减速reduce
	/*+streamtable(a)*/ 大表声明
	/*+mapjoin(b)*/ 小表声明
	一个select只能指定一个
7. join时，在on里面添加where条件，条件会先生效，减少join数据量
```



练习1：

```sql
--4. 每个用户在一周中的购买订单的分布 --列转行
set hive.cli.print.header=true;
-- 转换值 case xxx when x then y else z end
select 
user_id,
sum(case order_dow when '0' then 1 else 0 end) as dow_0,
sum(case order_dow when '1' then 1 else 0 end) as dow_1,
sum(case order_dow when '2' then 1 else 0 end) as dow_2,
sum(case order_dow when '3' then 1 else 0 end) as dow_3,
sum(case order_dow when '4' then 1 else 0 end) as dow_4,
sum(case order_dow when '5' then 1 else 0 end) as dow_5,
sum(case order_dow when '6' then 1 else 0 end) as dow_6
from orders
group by user_id
limit 20;

--5. 每个用户平均每个购买天中，购买的商品数量 
-- 异常值处理 if判断
select if(days_since_prior_order='','0',days_since_prior_order) as days_since_prior_order
-- count(distinct xxx)
select user_id,count(distinct days_since_prior_order) as day_cnt
from orders
group by user_id

6. 每个用户最喜爱购买的三个product是什么，最最终表结构可以是3个列，或者一个字符串
user_id "prod_top1,prod_top2,prod_top3"

-- 查询 每个用户每个产品的购买数量
select o.user_id, opp.product_id, count(1) as prod_cnt
from order_product_prior opp
left outer join orders o on (o.order_id == opp.order_id)
group by o.user_id,opp.product_id
limit 20;
-- 排序
select o.user_id, opp.product_id, count(1) as prod_cnt
from order_product_prior opp
left outer join orders o on (o.order_id == opp.order_id)
group by o.user_id,opp.product_id
order by o.user_id asc, opp.product_id desc
limit 20;

-- 查询 每个用户每个产品购买数超过3个的所有产品
select t1.user_id, concat_ws(',',collect_list(t1.product_id)) as top3_prod 
from (select o.user_id user_id, opp.product_id product_id, count(1) as prod_cnt
from order_product_prior opp
left outer join orders o on (o.order_id == opp.order_id)
group by o.user_id,opp.product_id) t1 
where t1.prod_cnt>=3
group by t1.user_id
limit 10;

-- 查询 每个用户最喜爱购买的3个产品   -- 加排序，只为看效果
select t1.user_id, t1.product_id, t1.prod_cnt, t1.rank
from(
select t.user_id user_id, t.product_id product_id, t.prod_cnt prod_cnt, row_number() over (partition by user_id order by prod_cnt desc) rank
from(
select o.user_id user_id, opp.product_id product_id, count(1) as prod_cnt
from order_product_prior opp
left outer join orders o on (o.order_id == opp.order_id)
group by o.user_id,opp.product_id
) t
) t1
where t1.rank<=3
order by t1.user_id asc 
limit 20;

-- 最终
select t1.user_id, concat_ws(',',collect_list(t1.product_id)) as top3_prod 
from(
select t.user_id user_id, t.product_id product_id, t.prod_cnt prod_cnt, row_number() over (partition by user_id order by prod_cnt desc) rank
from(
select o.user_id user_id, opp.product_id product_id, count(1) as prod_cnt
from order_product_prior opp
left outer join orders o on (o.order_id == opp.order_id)
group by o.user_id,opp.product_id
) t
) t1
where t1.rank<=3
group by t1.user_id
limit 20;

/* 原始情况
| o.user_id  | opp.product_id  | prod_cnt  |
+------------+-----------------+-----------+
| 1          | 49235           | 2         |
| 1          | 46149           | 3         |
| 1          | 41787           | 1         |
| 1          | 39657           | 1         |
| 1          | 38928           | 1         |
| 1          | 35951           | 1         |
| 1          | 30450           | 1         |
| 1          | 26405           | 2         |
| 1          | 26088           | 2         |
| 1          | 25133           | 8         |
| 1          | 196             | 10        |
| 1          | 17122           | 1         |
| 1          | 14084           | 1         |
| 1          | 13176           | 2         |
| 1          | 13032           | 3         |
| 1          | 12427           | 10        |
| 1          | 10326           | 1         |
| 1          | 10258           | 9         |
| 10         | 9871            | 2         |
| 10         | 9339            | 3         |
+------------+-----------------+-----------+
*/
/* 得到每个人的topN  看user_id=1的，没有任何毛病
+-------------+----------------+--------------+----------+
| t1.user_id  | t1.product_id  | t1.prod_cnt  | t1.rank  |
+-------------+----------------+--------------+----------+
| 1           | 196            | 10           | 2        |
| 1           | 12427          | 10           | 1        |
| 1           | 10258          | 9            | 3        |
| 10          | 30489          | 4            | 3        |
| 10          | 28535          | 4            | 2        |
| 10          | 16797          | 4            | 1        |
| 100         | 27344          | 3            | 2        |
| 100         | 21616          | 3            | 1        |
| 100         | 24852          | 2            | 3        |
| 1000        | 28465          | 7            | 3        |
| 1000        | 30492          | 7            | 2        |
| 1000        | 49683          | 7            | 1        |
| 10000       | 21137          | 44           | 1        |
| 10000       | 42828          | 39           | 3        |
| 10000       | 5077           | 41           | 2        |
| 100000      | 19348          | 5            | 1        |
| 100000      | 16797          | 5            | 3        |
| 100000      | 3318           | 5            | 2        |
| 100001      | 21137          | 41           | 2        |
| 100001      | 13176          | 41           | 1        |
+-------------+----------------+--------------+----------+
*/
```



练习2：

```sql
-- 表信息
-- order_product_prior 订单产品信息表
-- order_id, product_id, add_to_cart_order, reordered

-- orders 订单表：订单和用户的信息  dow=day of week
-- order_id,user_id,eval_set,order_number(购买订单的顺序),order_dow(星期几),order_hour_of_day（一天中的哪个小时）,days_since_prior_order(距离上一个订单的时间) 

-- 1. 每个用户最喜爱的top10%的商品
	-- 如果一个用户购买一共购买了10个，返回top1，购买 了20个返回top2
	-- 如果一个用户购买了3个，3*10%=0.3 返回top1，也 就是不够的至少一个

-- 计算每个用户购买的商品数量
-- 向上取整
select o.user_id, ceil(count(distinct opp.product_id)/10)
from order_product_prior opp
left outer join orders o on (opp.order_id == o.order_id)
group by o.user_id
limit 10;
/*
+------------+------+
| o.user_id  | _c1  |
+------------+------+
| 1          | 2    |
| 10         | 10   |
| 100000     | 7    |
| 100001     | 21   |
| 100005     | 7    |
| 100007     | 3    |
| 100011     | 8    |
| 100015     | 4    |
| 100019     | 3    |
| 10002      | 2    |
+------------+------+
*/
-- 每个用户购买过的产品数量排序
-- concat_ws(',', collect_list(t.product_id)) as topN
select t.user_id,t.product_id, t.prod_cnt, t.topN, t.rank
from 
(
select t1.user_id user_id, t1.product_id product_id, t1.prod_cnt prod_cnt, t2.topN topN, row_number() over(partition by t1.user_id order by t1.prod_cnt desc) rank
from (
select o.user_id as user_id, opp.product_id as product_id,count(1) as prod_cnt
from order_product_prior opp
left outer join orders o on (opp.order_id == o.order_id)
group by o.user_id, opp.product_id
) t1 
left outer join 
(
select o.user_id user_id, ceil(count(distinct opp.product_id)/10) topN
from order_product_prior opp
left join orders o on (opp.order_id == o.order_id)
group by o.user_id
) t2 on (t1.user_id == t2.user_id)
) t
where t.rank <= t.topN
limit 30;
/*
+------------+---------------+-------------+---------+---------+
| t.user_id  | t.product_id  | t.prod_cnt  | t.topn  | t.rank  |
+------------+---------------+-------------+---------+---------+
| 1          | 196           | 10          | 2       | 1       |
| 1          | 12427         | 10          | 2       | 2       |
| 10         | 16797         | 4           | 10      | 1       |
| 10         | 30489         | 4           | 10      | 2       |
| 10         | 47526         | 4           | 10      | 3       |
| 10         | 46979         | 4           | 10      | 4       |
| 10         | 28535         | 4           | 10      | 5       |
| 10         | 9339          | 3           | 10      | 6       |
| 10         | 40604         | 3           | 10      | 7       |
| 10         | 40706         | 3           | 10      | 8       |
| 10         | 1529          | 3           | 10      | 9       |
| 10         | 25931         | 3           | 10      | 10      |
| 100000     | 19348         | 5           | 7       | 1       |
| 100000     | 10151         | 5           | 7       | 2       |
| 100000     | 3318          | 5           | 7       | 3       |
| 100000     | 16797         | 5           | 7       | 4       |
| 100000     | 1062          | 4           | 7       | 5       |
| 100000     | 42625         | 4           | 7       | 6       |
| 100000     | 8424          | 3           | 7       | 7       |
| 100001     | 13176         | 41          | 21      | 1       |
| 100001     | 21137         | 41          | 21      | 2       |
| 100001     | 35951         | 29          | 21      | 3       |
| 100001     | 38383         | 26          | 21      | 4       |
| 100001     | 47209         | 26          | 21      | 5       |
| 100001     | 42445         | 23          | 21      | 6       |
| 100001     | 3957          | 20          | 21      | 7       |
| 100001     | 39877         | 20          | 21      | 8       |
| 100001     | 41220         | 18          | 21      | 9       |
| 100001     | 11422         | 18          | 21      | 10      |
| 100001     | 27104         | 15          | 21      | 11      |
+------------+---------------+-------------+---------+---------+
*/
-- 法二
select t.user_id,t.product_id, t.prod_cnt, t.topN, t.rank
from 
(
select t1.user_id user_id, t1.product_id product_id, t1.prod_cnt prod_cnt, 
ceil(count(1) over(partition by t1.user_id)*0.1) as topN,   -- 重点
row_number() over(partition by t1.user_id order by t1.prod_cnt desc) rank
from (
select o.user_id as user_id, opp.product_id as product_id,count(1) as prod_cnt
from order_product_prior opp
left outer join orders o on (opp.order_id == o.order_id)
group by o.user_id, opp.product_id
) t1 
) t
where t.rank <= t.topN
limit 30;
/*
+------------+---------------+-------------+---------+---------+
| t.user_id  | t.product_id  | t.prod_cnt  | t.topn  | t.rank  |
+------------+---------------+-------------+---------+---------+
| 1          | 12427         | 10          | 2       | 1       |
| 1          | 196           | 10          | 2       | 2       |
| 10         | 30489         | 4           | 10      | 1       |
| 10         | 16797         | 4           | 10      | 2       |
| 10         | 47526         | 4           | 10      | 3       |
| 10         | 46979         | 4           | 10      | 4       |
| 10         | 28535         | 4           | 10      | 5       |
| 10         | 28842         | 3           | 10      | 6       |
| 10         | 1529          | 3           | 10      | 7       |
| 10         | 9339          | 3           | 10      | 8       |
| 10         | 40706         | 3           | 10      | 9       |
| 10         | 31506         | 3           | 10      | 10      |
| 100000     | 10151         | 5           | 7       | 1       |
| 100000     | 19348         | 5           | 7       | 2       |
| 100000     | 3318          | 5           | 7       | 3       |
| 100000     | 16797         | 5           | 7       | 4       |
| 100000     | 1062          | 4           | 7       | 5       |
| 100000     | 42625         | 4           | 7       | 6       |
| 100000     | 8424          | 3           | 7       | 7       |
| 100001     | 13176         | 41          | 21      | 1       |
| 100001     | 21137         | 41          | 21      | 2       |
| 100001     | 35951         | 29          | 21      | 3       |
| 100001     | 38383         | 26          | 21      | 4       |
| 100001     | 47209         | 26          | 21      | 5       |
| 100001     | 42445         | 23          | 21      | 6       |
| 100001     | 3957          | 20          | 21      | 7       |
| 100001     | 39877         | 20          | 21      | 8       |
| 100001     | 11422         | 18          | 21      | 9       |
| 100001     | 41220         | 18          | 21      | 10      |
| 100001     | 27104         | 15          | 21      | 11      |
+------------+---------------+-------------+---------+---------+
*/

-- 2. 建分区表，orders表按照order_dow建立分区表 orders_part，
-- 然后从hive查询orders动态插入 orders_part表中。
-- 动态插入
create table orders_part (
order_id string,
user_id string,
eval_set string,
order_number string,
order_hour_of_day string,
days_since_prior_order string
)
partitioned by(order_dow string)
row format delimited fields terminated by ','
stored as textfile;

-- 覆盖分区表
-- 实际项目中，partition(order_dow='1') 为  partition(dt=变量) ， 变量为'201907'这种
insert overwrite table orders_part partition(order_dow='1') 
select order_id,user_id,eval_set,order_number,order_hour_of_day,days_since_prior_order from orders where order_dow = '1' limit 10;

-- 动态
set hive.exec.dynamic.partition=true; -- 使用动态分区模式
set hive.exec.dynamic.partition.mode=nonstrict; -- 非严格模式
-- 注意order_dow需要写最后
insert overwrite table orders_part partition(order_dow) 
select order_id,user_id,eval_set,order_number,order_hour_of_day,days_since_prior_order,order_dow from orders limit 10;

--  每个用户购买的product商品去重后的集合数据 (用字符串表示以‘，’分割)
select o.user_id, concat_ws(',', collect_set(opp.product_id)) as prods
from order_product_prior opp
left outer join orders o on (opp.order_id == o.order_id)
group by o.user_id
limit 10;
```



##### TODO

```
-- HDFS合并小文件

-- 导入数据
from log_169_searchd_pro_20141122 insert into table searchd_pro1 PARTITION (query_date) ？？？
```



##### Jieba Transform

```python
import sys
reload(sys)
sys.setdefaultencoding('utf8')
import jieba
for line in sys.stdin:
    ss = ' '.join(jieba.cut(line,cut_all=True))
    print('%s'%ss)
```

```sql
-- Hive sql
add file /home/hdfs/sub/jieba_udf.py;
-- Permission denied: Principal [name=hdfs, type=USER] does not have following privileges for operation ADD [ADMIN] (state=,code=1)
-- 配置hive权限，参考Hive权限.md
-- 再次报错：找不到文件
-- 暂时将他上传到hdfs上：/data/jieba/jieba_udf.py
add file hdfs:/data/jieba/jieba_udf.py;
-- 解决文件找不到问题：将文件放入hive的目录下
add file /usr/hdp/3.1.0.0-78/hive/auxlib/jieba_udf.py;

select transform(sentence) using 'python jieba_udf.py' as (seg) from news_noseg limit 10;
-- org.apache.hadoop.hive.ql.security.authorization.plugin.HiveAccessControlException: Query with transform clause is disallowed in current configuration.
-- 暂时不解决这个问题
-- Transform和UDF对比：选UDF好(UDF的jar包需放到hive的auxlib目录下，随hive启动而启动，可永久使用)
```

