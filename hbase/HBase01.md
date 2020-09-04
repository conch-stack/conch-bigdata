# Getting Started

### Shell

##### 连接：

```shell
$ hbase shell
# help "COMMAND"
# help "COMMAND_GROUP"
```

##### 帮助：

```shell
hbase(main):003:0> help

COMMAND GROUPS:
  Group name: general
  Commands: processlist, status, table_help, version, whoami

  Group name: ddl
  Commands: alter, alter_async, alter_status, create, describe, disable, disable_all, drop, drop_all, enable, enable_all, exists, get_table, is_disabled, is_enabled, list, list_regions, locate_region, show_filters

  Group name: namespace
  Commands: alter_namespace, create_namespace, describe_namespace, drop_namespace, list_namespace, list_namespace_tables

  Group name: dml
  Commands: append, count, delete, deleteall, get, get_counter, get_splits, incr, put, scan, truncate, truncate_preserve

  Group name: tools
  Commands: assign, balance_switch, balancer, balancer_enabled, catalogjanitor_enabled, catalogjanitor_run, catalogjanitor_switch, cleaner_chore_enabled, cleaner_chore_run, cleaner_chore_switch, clear_block_cache, clear_compaction_queues, clear_deadservers, close_region, compact, compact_rs, compaction_state, flush, is_in_maintenance_mode, list_deadservers, major_compact, merge_region, move, normalize, normalizer_enabled, normalizer_switch, split, splitormerge_enabled, splitormerge_switch, trace, unassign, wal_roll, zk_dump

  Group name: replication
  Commands: add_peer, append_peer_namespaces, append_peer_tableCFs, disable_peer, disable_table_replication, enable_peer, enable_table_replication, get_peer_config, list_peer_configs, list_peers, list_replicated_tables, remove_peer, remove_peer_namespaces, remove_peer_tableCFs, set_peer_bandwidth, set_peer_exclude_namespaces, set_peer_exclude_tableCFs, set_peer_namespaces, set_peer_replicate_all, set_peer_tableCFs, show_peer_tableCFs, update_peer_config
  
hbase(main):004:0> help "create"  

```



##### 创建表：

```shell
hbase(main):033:0> create 'mytable', 'mycf'
Created table mytable
Took 2.7219 seconds                                                                                                                                                                                               
=> Hbase::Table - mytable

hbase(main):035:0> list
TABLE                                                                                                                                                                                                             
mytable                                                                                                                                                                                                           
1 row(s)
Took 0.0096 seconds                                                                                                                                                                                               
=> ["mytable"]

hbase(main):036:0> describe 'mytable'
Table mytable is ENABLED                                                                                                                                                                                          
mytable                                                                                                                                                                                                           
COLUMN FAMILIES DESCRIPTION                                                                                                                                                                                       
{
	NAME => 'mycf', 
	VERSIONS => '1', 
	EVICT_BLOCKS_ON_CLOSE => 'false', 
	NEW_VERSION_BEHAVIOR => 'false', 
	KEEP_DELETED_CELLS => 'FALSE', 
	CACHE_DATA_ON_WRITE => 'false', 
	DATA_BLOCK_ENCODING => 'NONE', 
	TTL => 'FOREVER', 
	MIN_VERSIONS => '0', 
	REPLICATION_SCOPE => '0', 
	BLOOMFILTER => 'ROW', 
	CACHE_INDEX_ON_WRITE => 'false', 
	IN_MEMORY => 'false', 
	CACHE_BLOOMS_ON_WRITE => 'false', 
	PREFETCH_BLOCKS_ON_OPEN => 'false', 
	COMPRESSION => 'NONE',
	BLOCKCACHE => 'true', 
	BLOCKSIZE => '65536'
}                                                                                                                                                             
1 row(s)
Took 0.4369 seconds                                                                                                                                                                                               
hbase(main):037:0> 
```



##### 新增数据

```shell
hbase(main):037:0> put 'mytable', 'row1', 'mycf:name', 'billyWangpaul'
Took 0.2954 seconds  
```



##### 扫描数据(Scan)：

```shell
# scan 所有数据
hbase(main):010:0> scan 'mytable'
ROW								COLUMN+CELL                                                                                                                                       row1      				column=mycf:name, timestamp=1555313062579, value=billyWangpaul                                                                                              
row2              column=mycf:name, timestamp=1555313081637, value=sara                                                                                                       
row3              column=mycf:name, timestamp=1555313092355, value=chris                                                                                                      
row4              column=mycf:name, timestamp=1555313105704, value=helen                                                                                                      
row5              column=mycf:name, timestamp=1555313117451, value=andyWang                                                                                                   
row6              column=mycf:name, timestamp=1555313132684, value=kateWang                                                                                                   
6 row(s)
Took 0.0167 seconds                                                                                                                                                                                               
hbase(main):011:0> 

# scan 加 limit
# HBase 中用 STARTROW 和 ENDROW来进行限制(左闭右开)
hbase(main):016:0> scan 'mytable', {STARTROW=>'row2', ENDROW=>'row4'}
ROW               COLUMN+CELL                                                                                                                                                 
 row2             column=mycf:name, timestamp=1555313081637, value=sara                                                                                                       
 row3             column=mycf:name, timestamp=1555313092355, value=chris                                                                                                      
2 row(s)
Took 0.0517 seconds 
```



##### Get a single row of data:

```shell
hbase(main):011:0> get 'mytable', 'row1'
COLUMN             CELL                                                                                                                                                        
 mycf:name         timestamp=1555313062579, value=billyWangpaul                                                                                                                
1 row(s)
```



##### Disable  a Table:

在删除表或者更新表属性设置的时候，需要先Disable对应表

客户端连接该表较多，直接修改更新对性能影响大，不如直接暂停服务，快速更新

```shell
disable 'mytable'

enable 'mytable'

disable_all '正则表达式'

# 检测是否被停用
is_disabled '表名'
```



##### 删除表：

```shell
disable 'mytale'

drop 'mytable'
```



##### 文件句柄/线程限制：

```shell
ulimit -n
计算打开文件数
StoreFiles per ColumnFamily(HFile) * Regions per RegionServer

3个CF 每个平均3个StoreFile 100个Region  3*3*100个文件描述

ulimit -u 
```



##### 版本

```shell
# 只有修改了表的版本，才可以存储多版本数据，否则，默认VERSIONS=>1，无论你怎么添加多版本的Value，只会覆盖
alter 'mytable', {NAME=>'mycf', VERSIONS=>5}
put 'mytable', 'row1', 'mycf:name', 'verison5value'

hbase(main):022:0> get 'mytable', 'row1', {COLUMN=>'mycf:name', VERSIONS=>5}
COLUMN                   CELL                                                                                                                                                        
 mycf:name               timestamp=1555319448555, value=verison5value                                                                                                                
 mycf:name               timestamp=1555313062579, value=billyWangpaul                                                                                                                
1 row(s)
Took 0.0470 seconds                                                                                                                                                                                               
hbase(main):023:0> get 'mytable', 'row1', {COLUMN=>'mycf:name', VERSIONS=>4}
COLUMN                   CELL                                                                                                                                                        
 mycf:name               timestamp=1555319448555, value=verison5value                                                                                                                
 mycf:name               timestamp=1555313062579, value=billyWangpaul                                                                                                                
1 row(s)
Took 0.0176 seconds                                                                                                                                                                                               
hbase(main):024:0> get 'mytable', 'row1', {COLUMN=>'mycf:name', VERSIONS=>3}
COLUMN                   CELL                                                                                                                                                        
 mycf:name               timestamp=1555319448555, value=verison5value                                                                                                                
 mycf:name               timestamp=1555313062579, value=billyWangpaul                                                                                                                
1 row(s)
Took 0.0107 seconds                                                                                                                                                                                               
hbase(main):025:0> get 'mytable', 'row1', {COLUMN=>'mycf:name', VERSIONS=>2}
COLUMN                   CELL                                                                                                                                                        
 mycf:name               timestamp=1555319448555, value=verison5value                                                                                                                
 mycf:name               timestamp=1555313062579, value=billyWangpaul                                                                                                                
1 row(s)
Took 0.0110 seconds                                                                                                                                                                                               
hbase(main):026:0> get 'mytable', 'row1', {COLUMN=>'mycf:name', VERSIONS=>1}
COLUMN                   CELL                                                                                                                                                        
 mycf:name               timestamp=1555319448555, value=verison5value                                                                                                                
1 row(s)
Took 0.0227 seconds                                                                                                                                                                                               
hbase(main):027:0> 

# 插入时直接指定Timestamp
hbase(main):018:0> put 'mytable', 'row1', 'mycf:name', 'newversion', 2
hbase(main):021:0> get 'mytable', 'row1', {COLUMN=>'mycf', VERSIONS=>5}
COLUMN             CELL                                                                                                                                                        
 mycf:name         timestamp=1555313062579, value=billyWangpaul                                                                                                                
 mycf:name         timestamp=2, value=newversion 
```



##### 删除

```shell
# 精确到列，删除哪一列
delete 'mytable', 'row1', 'mycf:name', {VERSIONS=>5}
# deleteall删除整行
deleteall 'mytable', 'row1'
# 删除指定版本记录
delete 't1', 'r1', 'cf:cq', 'timestamp'
# 再次新增已删除的记录，将不会显示，因为墓碑记录(暂时未真正删除)
scan 'mytable', {RAW=>true,VERSIONS=>5}
ROW            COLUMN+CELL    
row1           column=mycf:name, timestamp=1555319448555, type=Delete                                                                                                      
row1           column=mycf:name, timestamp=1555319448555, value=verison5value                                                                                              
row1           column=mycf:name, timestamp=1555313062579, value=billyWangpaul                                                                                              
row1           column=mycf:name, timestamp=3, type=Delete                                                                                                                  
row1           column=mycf:name, timestamp=3, value=newversion33                                                                                                           
row1           column=mycf:name, timestamp=2, value=newversion                                                                                                             
row2           column=mycf:name, timestamp=1555313081637, value=sara                                                                                                       
row3           column=mycf:name, timestamp=1555313092355, value=chris   
```



##### 查看rowkey在哪个Region里面

```shell
hbase(main):061:0> locate_region 'mytable', 'row1'
HOST                 					REGION                                                                                                                                                      
apollo03.dev.zjz:16020        {ENCODED => 293bc8263d092bb39caefcef637ebe87, NAME => 'mytable,,1555311814232.293bc8263d092bb39caefcef637ebe87.', STARTKEY => '', ENDKEY => ''}

# Region名 mytable,,1555311814232.293bc8263d092bb39caefcef637ebe87.
# Region名的Hash值：293bc8263d092bb39caefcef637ebe87

# 获取某个Region的信息
get 'hbase:meta', 'mytable,,1555311814232.293bc8263d092bb39caefcef637ebe87.'
COLUMN                     CELL                                                                                                                                                        
 info:regioninfo         timestamp=1555319406583, value={ENCODED => 293bc8263d092bb39caefcef637ebe87, NAME => 'mytable,,1555311814232.293bc8263d092bb39caefcef637ebe87.', STARTKEY =>
                                                       '', ENDKEY => ''}                                                                                                                                          
 info:seqnumDuringOpen   timestamp=1555319406583, value=\x00\x00\x00\x00\x00\x00\x00\x14                                                                                             
 info:server             timestamp=1555319406583, value=apollo03.dev.zjz:16020                                                                                                       
 info:serverstartcode    timestamp=1555319406583, value=1554812095615                                                                                                                
 info:sn                 timestamp=1555319406307, value=apollo03.dev.zjz,16020,1554812095615                                                                                         
 info:state              timestamp=1555319406583, value=OPEN 

# 服务器标识码：apollo03.dev.zjz,16020,1554812095615 
```



### API
https://www.zifangsky.cn/1286.html

数字类型：使用HBase Java Api操作





### 命名空间

可实现分库：不同组进行不同的环境设定：如配额管理、安全管理



### Memstore

整理数据，按（rowkey）顺序存放，提供读性能

LSM树结构存储数据



### Region

> * 一个RegionServer包含多个Region，划分规则：
>   * 一个表的一段键值在一个RegionServer上会产生一个Region
>   * 如果一行的数据量太大(非常大，否则默认都是不切分的)，HBase也会将你的这个Region根据列簇切分到不同机器上
> * 一个Region中包含多个Store，划分规则：
>   * 一个列簇分为一个Store，如果一个表只有一个列簇，那么这个表在机器上的每个Region里面都只有一个Store
> * 一个Store里面只有一个Memstore，多个HFile





### 真正删除

合并(Compaction) : default 7 Days

合并的时候，发现数据被打上墓碑标记，则在新生产的HFile里面就会忽略这条记录




### High API 

#### 一、过滤器

在Get和Scan时，过滤结果用

> Filter:
>
> - ValueFilter：值过滤，匹配到值的所有列都会被scan到
> - SingleColumnValueFilter：单列过滤，匹配指定列：对比ValueFilter
>   - 需所有数据列都包含该单列：
>     - 解决：替换为过滤器列表：包含列过滤器、列簇过滤器、值过滤器
>     - 缺点：使用三个过滤器，执行速度慢
> - PageFilter：类似limit，需自实现skip(记录最后一条rowkey，在下一次scan设置startRowkey)
> - FilterList：过滤器列表：**顺序执行**
>   - 逻辑运算
>     - Operator.MUST_PASS_ALL   and 
>     - Operator.MUST_PASS_ONE  or
>   - 过滤器列表嵌套过滤器列表来实现嵌套查询
> - RowFilter：行键过滤器
> - MultiRowRangeFilter：多行范围过滤器
> - PrefixFilter：行键前缀过滤器
> - FuzzyRowFilter：模糊行键过滤器
> - InclusiveStopFilter：包含结尾过滤器：默认Scan的stopRow是不包含的
> - RandomRowFilter：随机行过滤器：随机抽取系统中的一部分数据：可用于**数据分析**场景
> - FamilyFilter：列簇过滤器
> - QualifierFilter：列过滤器
> - DependentColumnFilter：依赖列过滤器：防止并发写后，读导致的出现脏读现象
> - ColumnPrefixFilter：列前缀过滤器
> - MultipleColumnPrefixFilter：多列前缀过滤器
> - KeyOnlyFilter：列名过滤器：只过滤列名，不管值：在结果集中使用CellUtil的cloneQualifier(cell)方法获取列名
> - FirstKeyOnlyFilter：首个列明名过滤器：当读到一行的第一个列的时候，立刻跳到下一行，放弃其他列的遍历：可用于**计算行数**
> - ColumnRangerFilter：列名范围过滤器
> - TimestampFilter：时间戳过滤器，精确到毫秒



> Comparator:
>
> - CompareFilter.CompareOp
>   - EQUAL  =
>   - GREATER >
>   - LESS <
>   - LESS_OR_EQUAL <=
>   - NOT_EQUAL !=
>   - GREATER_OR_EQUAL >=
>   - NO_OP 无操作
>
> - SubstringComparator：%target%比较器，判断目标字符串是否包含指定字符串
> - BinaryComparator:  =target比较器；可用于比较数字
> - RegexStringComparator：搭配CompareFilter.CompareOp.EQUAL
> - NullComparator：空值比较器
> - LongComparator： new BinaryComparator(Bytes.toBytes(10)) -> new LongComparator(10L)
> - BitComparator：比特位比较器
> - BinaryPrefixComparator：字节数组前缀比较器：找出所有以目标字节数组大头的记录



> 自定义过滤器



```java
// 行键模糊过滤
// 2016_??_??_4444 行键
// {0, 0, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0}  行键掩码
FuzzyRowFilter filter = new FuzzyRowFilter(Arrays.asList(
		new Pair<byte[], byte[]> (
				Bytes.toBytesBinary("2016_??_??_4444"), 
				new byte[]{0, 0, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0}
	  )
));
```



### 协处理器(Coprocessor)

类似关系型数据库的 存储过程和触发器

> 终端程序：EndPoint  -> 实现存储过程
>
> 观察者：Observers    -> 实现触发器



### 事务

不支持表关联

部分支持ACID

Apache Omid



### 参考：

[HBase表设计](https://blog.csdn.net/zhouzhaoxiong1227/article/details/47260585)

[2.0Java操作API](https://www.zifangsky.cn/1286.html)

[高性能分布式数据库HBase](https://sq.163yun.com/blog/article/174620451741294592)

[HBase优化实战](https://sq.163yun.com/blog/article/170708590806372352)

[HBase高可用原理与实践](https://sq.163yun.com/blog/article/173155879264116736)

[淘宝双11大屏](https://promotion.aliyun.com/ntms/act/hbase.html)