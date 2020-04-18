```
###------------------------------------ Node ------------------------------------
#集群名字，同一网段内相同名字加入同一集群   
   cluster.name: test-es
#节点名字
   node.name: node-1
#节点是否为主节点
   node.master: true
#节点是否存储数据
   node.data: false
#关闭多播，默认为开启(true)
   directory.zen.ping.multicast.enabled: false
   
###-------------------------------------------单播配置--------------------------------------------
   discovery.zen.ping.unicast.hosts: ["192.168.20.156:9300","192.168.20.156:9301","192.168.20.156:9302"]
#单播最大并发连接数默认为10
   discovery.zen.ping.unicast.concurrent_connects: 10
#最小N主节点才能组成集群(防止脑裂)
   discovery.zen.minimum_master_nodes: 1

###-------------------------------------------多播配置--------------------------------------------
#通信接口可赋值为地址或者接口名,默认为所有可用接口
   discovery.zen.ping.multicast.address:
#通信端口默认54328
   discovery.zen.ping.multicast.port: 54328
#发送多播的消息地址默认224.2.2.4
   discovery.zen.ping.multicast.group: 224.2.2.4
#缓冲区大小默认2048K
   discovery.zen.ping.multicast.buffer_size: 2048
#定义多播生存期,默认为3
   discovery.zen.ping.multicast.ttl: 3
   
###----------------------------------------zen错误检测--------------------------------------------
#节点向目标节点ping频率默认为1s
   discovery.zen.fd.ping_interval: 1s
#节点向目标节点ping时常默认为30s
   discovery.zen.fd.ping_timeout: 30s
#目标节点不可用重试次数默认为3
   discovery.zen.fd.ping_retries: 3

###---------------------------------------开启script使用-------------------------------------------
   script.inline: on
   script.indexed: on

###---------------------------------------索引段合并策略-------------------------------------------
#tiered:(默认)合并大小相似的索引段，每层允许的索引段的最大个数
#log_byte_size:策略不断地以字节数的对数为计算单位，选择多个索引来合并创建新索引
#log_doc:与log_byte_size类似，不同的是前者基于索引的字节数计算，后者基于索引段文档数计算

#  index.merge.policy.type: tiered

###---------------------------------- Gateway -----------------------------------
#启动集群需要节点数5表示要有5个节点才能启动集群
#   gateway.recover_after_nodes: 5
#启动集群需要的数据节点数
#   gateway.recover_after_data_nodes: 3
#启动集群候选主节点数目
#   gateway.recover_after_master_nodes: 3
#上述条件满足后，启动恢复前等待时间
#   gateway.recover_after_time: 10m
#立即启动恢复过程需要集群中存在的节点数
#   gateway.expected_nodes: 3
#立即启动恢复过程需要集群中存在的数据节点数
#   gateway.expected_data_nodes: 3
#立即启动恢复过程需要集群中存在的候选主节点数
#   gateway.expected_master_nodes: 3

###---------------------------------集群级的恢复配置---------------------------------
#从分片源恢复分片时允许打开的并发流的数量默认为3
#   indices.recovery.concurrent_streams: 3
#分片恢复过程中每秒传输数据的最大值默认为20M
#   indices.recovery.max_bytes_per_sec: 20M
#分片恢复过程中是否压缩传输的数据默认为true
#   indices.recovery.compress: true
#从分片源向目标分片拷贝时数据块的大小默认512K
#   indices.recovery.file_chunk_size: 512K
#恢复过程中分片间传输数据时，单个请求可以传输多少行日志事务默认为1000
#   indices.recovery.translog_ops: 1000
#从源分片拷贝事务日志使用的数据块的大小默认为512K
#   indices.recovery.translog_size: 512K

###---------------------------------索引级缓存过滤器配置---------------------------------
#缓存类型默认值为node 可选参数[{"resident":"不会被jvm清理"},{"soft":"内存不足时会被jvm清理在weak之后"},{"weak":"内存不足时会被jvm优先清理"},{"node":"缓存会在节点级控制"}]
#index.cache.filter.type: node
#存储到缓存中的最大记录数默认为-1无限制
#index.cache.filter.max_size: -1
#存储到缓存中的数据失效时间默认为-1永不失效
#index.cache.filter.expire: -1
#节点级的缓存大小默认为20%也可以使用1024mb
#index.cache.filter.size: 20%

###---------------------------------索引级字段缓存配置---------------------------------
#字段数据缓存最大值可以为百分比20%或者具体值10G
#index.fielddata.cache.size: 20%     
#字段数据缓存失效时间默认为-1永不失效
#index.fielddata.cache.expire: -1
```

