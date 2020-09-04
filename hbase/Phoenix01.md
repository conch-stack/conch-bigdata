## Phoenix

### Quick Start

us_population.sql

```sql
CREATE TABLE IF NOT EXISTS us_population (
      state CHAR(2) NOT NULL,
      city VARCHAR NOT NULL,
      population BIGINT
      CONSTRAINT my_pk PRIMARY KEY (state, city));
```

us_population.csv

```
NY,New York,8143197
CA,Los Angeles,3844829
IL,Chicago,2842518
TX,Houston,2016582
PA,Philadelphia,1463281
AZ,Phoenix,1461575
TX,San Antonio,1256509
CA,San Diego,1255540
TX,Dallas,1213825
CA,San Jose,912332
```

us_population_queries.sql

```sql
SELECT state as "State",count(city) as "City Count",sum(population) as "Population Sum"
FROM us_population
GROUP BY state
ORDER BY sum(population) DESC;
```

run

```shell
cd /usr/hdp/3.1.0.0-78/phoenix/bin

# 执行SQL脚本
./psql.py <your_zookeeper_quorum> us_population.sql us_population.csv us_population_queries.sql

# 登录本地
./sqlline.py localhost

./sqlline.py localhost ../examples/stock_symbol.sql
```



映射HBase：

```
映射前：
hbase(main):013:0> scan 'mytable'
ROW                        COLUMN+CELL                                                                                                                                                 
 row1                      column=mycf:name, timestamp=1557403023523, value=billyWangpaul                                                                                              
 row2                      column=mycf:name, timestamp=1557403057529, value=test1                                                                                                      
 row3                      column=mycf:name, timestamp=1557403063542, value=test2 
 
映射：
CREATE TABLE "mytable" (
        "ROW" VARCHAR NOT NULL PRIMARY KEY,
        "mycf"."name" VARCHAR
);

映射后：
	Phoenix：
    0: jdbc:phoenix:localhost> select * from "mytable";
    +-------+----------------+
    |  ROW  |      name      |
    +-------+----------------+
    | row1  | billyWangpaul  |
    | row2  | test1          |
    | row3  | test2          |
    +-------+----------------+
    
  HBase：
  	hbase(main):014:0> scan 'mytable'
    ROW                      COLUMN+CELL                                                                                                                                                 
     row1                    column=mycf:_0, timestamp=1557403023523, value=                                                                                                             
     row1                    column=mycf:name, timestamp=1557403023523, value=billyWangpaul                                                                                              
     row2                    column=mycf:_0, timestamp=1557403057529, value=                                                                                                             
     row2                    column=mycf:name, timestamp=1557403057529, value=test1                                                                                                      
     row3                    column=mycf:_0, timestamp=1557403063542, value=                                                                                                             
     row3                    column=mycf:name, timestamp=1557403063542, value=test2
```



### 语法学习：

#### 建表

Phoenix中的表必须有主键   

区分大小写 小写用 "my_case_sensitive_table"

主键不能有 colunm Family (Primary key columns must not have a family name)

非主键字段 不定长字段不能定义为not null (A non primary key column may only be declared as not null on tables with immutable rows)

中文的字段可设置为VARCHAR类型

> CREATE TABLE IF NOT EXISTS MYTABLE (ID INTEGER PRIMARY KEY, NAME VARCHAR, SEX VARCHAR, ADDRESS VARCHAR);

```
CREATE TABLE my_schema.my_table ( 
	id BIGINT not null primary key, 
	date
)

CREATE TABLE my_table ( 
	id INTEGER not null primary key desc, 
	date DATE not null,
  m.db_utilization DECIMAL, 
  i.db_utilization
)
m.DATA_BLOCK_ENCODING='DIFF'

CREATE TABLE stats.prod_metrics ( 
	host char(50) not null, 
	created_date date not null,
  txn_count bigint 
  CONSTRAINT pk PRIMARY KEY (host, created_date)
)
    
CREATE TABLE IF NOT EXISTS "my_case_sensitive_table"( 
	"id" char(10) not null primary key, 
	"value" integer
)
DATA_BLOCK_ENCODING='NONE',
VERSIONS=5,
MAX_FILESIZE=2000000 split on (?, ?, ?)

CREATE TABLE IF NOT EXISTS my_schema.my_table (
	org_id CHAR(15), 
  entity_id CHAR(15), 
  payload binary(1000),
  CONSTRAINT pk PRIMARY KEY (org_id, entity_id) 
)
TTL=86400
```



#### 删除表

> DROP TABLE IF EXISTS MYTABLE;

#### 插入数据

> UPSERT INTO MYTABLE VALUES (1, 'WXB', 'MALE', '010-22222222');

#### 删除数据

> DELETE FROM MYTABLE WHERE ID = 1;

#### 查询数据

> SELECT * FROM MYTABLE WHERE ID=1;

#### 修改数据

> UPSERT INTO MYTABLE VALUES (1, 'WXB', 'MALE', '010-22222222');



#### 索引

##### IMMutable Indexing

​	不可变索引主要创建在不可变表上，适用于数据只写一次不会有Update等操作，在什么场景下会用到不可变索引呢，很经典的时序数据:`write once read many times`。在这种场景下，所有索引数据（primary和index)要么全部写成功，要么一个失败全都失败返回错误给客户端。不可变索引用到场景比较少，下面是创建不可变索引的方式：

```
create table test (pk VARCHAR primary key,v1 VARCHAR, v2 VARCHAR) IMMUTABLE_ROWS=true;

即在创建表时指定IMMUTABLE_ROWS参数为true，默认这个参数为false。如果想把不可变索引改为可变索引，可用alter修改：
alter table test set IMMUTABLE_ROWS=false;
```



##### Mutable Indexing

​	可变索引意思是在修改数据如Insert、Update或Delete数据时会同时更新索引。这里的索引更新涉及WAL，即主表数据更新时，会把索引数据也同步更新到WAL，只有当WAL同步到磁盘时才会去更新实际的primary/index数据，以保证当中间任何一个环节异常时可通过WAL来恢复主表和索引表数据。



https://yq.aliyun.com/articles/688618 索引

### 参考

[博客1](https://zhuanlan.zhihu.com/p/21584994)

[入门到精通](https://yq.aliyun.com/articles/574090?spm=a2c4e.11163080.searchblog.154.4f172ec1vxmdeh)

[视频分享](https://yq.aliyun.com/teams/382/type_blog-cid_414-page_1)

[用例](https://mp.weixin.qq.com/s/xh8NE8MpmahUtClViCdEIg)

[Phoenix关于时区的处理方式说明](https://mp.weixin.qq.com/s/EL1Rfj2WbdXJj3AkjhS4ZA)

[百度智能监控场景下的HBase实践](https://mp.weixin.qq.com/s/GIdzt1Zn_wOaiDKoHNITfw)

[阿里云HBase+Spark+Phoenix，提供复杂数据分析支持](https://help.aliyun.com/document_detail/95046.html?spm=a2c4g.11186623.6.601.b81978f0GCM62z)

