## HBase-Rowkey设计

##### rowkey = [userid]:[yyyymmdd]

- userid放前面，可以避免数据因为time的连续性导致一段时间内，数据进入同一Region分区，导致热点数据问题
- userid放前面，可以在同一用户数据连续查询时，提供查询速度



##### 小时维度的统计

- cq = 小时为维度的long值

- 表名：hour_table  ，设计成宽表
- rowkey = [userid]:[yyyymmdd]
- cf = a
- cq = 00 - 23 的long时间戳值
- value = 值



##### 天维度统计

- cq = 天为维度的int值
- 表名：day_table_yyyyMM   ，以月为维度建表
- rowkey = [userid]
- cf = a
- cq = 01 - 31  天编码
- value = 值



##### rowkey的设计取决于业务

- 业务查询维度多，则rowkey可以继续往后扩展
- 但也不需要全部交给rowkey来做
  - 可以设计成高表：每个月生成一张表，表名带日期信息
  - 设计成宽表：每个cq列的取值范围代表的一个维度