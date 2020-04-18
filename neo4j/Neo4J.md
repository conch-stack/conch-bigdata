##### 下载安装

下载：https://neo4j.com/download-center/

文档：https://neo4j.com/docs/operations-manual/current/docker/introduction/



Docker:

https://hub.docker.com/_/neo4j/

docker pull neo4j



You can start a Neo4j container like this:

```console
mkdir -p neo4j/data

docker run -d --name neo4j\
    --publish=7474:7474 --publish=7687:7687 \
    --volume=$HOME/neo4j/data:/data \
    neo4j
```

which allows you to access neo4j through your browser at [http://localhost:7474](http://localhost:7474/).

This binds two ports (`7474` and `7687`) for HTTP and Bolt access to the Neo4j API. A volume is bound to `/data` to allow the database to be persisted outside the container.

By default, this requires you to login with `neo4j/neo4j` and change the password. You can, for development purposes, disable authentication by passing `--env=NEO4J_AUTH=none` to docker run.

新密码：123456



##### [概念](https://cloud.tencent.com/developer/article/1336299)

- Node 节点
- Label 标签
- Property 属性



##### 简单使用：

```sql
-- 创建没属性的节点
create(emp:Employee)
-- 创建有属性的节点
create(emp:Employee{name:"zjz",age:18})
-- 创建多标签
create (m:Movie:Cinema:Film:Picture)
-- 查询
match(m:Employee) return m
match(e:Employee) return e.name,e.age

-- 方向关系
-- 双向
create (e:Employee) <- [r:DemoRepation] -> (e:Employee)
-- 单向
CREATE (e:Employee)-[r:DemoRelation]->(c:Employee)

-- 使用已知节点创建带属性的关系
MATCH (m:Employee),(e:Employee) 
CREATE (m)-[r:DO_SHOPPING_WITH{shopdate:"12/12/2014",price:55000}]->(e) 
RETURN r
-- 查询关系信息
MATCH (m)-[r:DO_SHOPPING_WITH]->(e) 
RETURN m,e

-- 查询带Where条件
match (m:Employee)
Where m.name="zjz"
return m

-- 删除：需要与match一起使用
-- 删除节点
MATCH (e: Picture) DELETE e
-- 报错，提示需先删除关
-- 删除节点关系
MATCH (m: Employee)-[rel]-(e:Employee) 
DELETE rel
-- 删除一个空节点（空节点名称，空标签，空属性，空关系）的节点
-- 需要知道他的id
match (n) where id(n) = 20 delete n

-- 删除标签或属性remove
-- remove属性
match (m:Employee)
remove m.age
return m
-- remove标签
-- CREATE (m:Movie:Pic)
-- MATCH (n:Movie) RETURN n
MATCH (m:Movie) 
REMOVE m:Pic

-- set属性
MATCH (m:Movie)
SET m.view = 3456
RETURN m

-- limit skip
MATCH (emp:Employee) 
RETURN emp
SKIP 2
LIMIT 2
```



##### 插件

```
# 1. apoc
# 作用：动态操作Neo4j
# 地址：https://github.com/neo4j-contrib/neo4j-apoc-procedures
# 参考：http://weikeqin.cn/2018/04/17/neo4j-apoc-use/#more

# 2. 对于neo4j自增id，没有好的办法
# 只能用neo4j提供的自增id
# 可以设置一个属性用uuid来标识。(记得给这个属性建索引)
# neo4j-uuid可以直接用neo4j其中一个作者写的插件https://github.com/graphaware/neo4j-uuid
```





##### Spring-data-neo4j

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-neo4j</artifactId>
</dependency>
<dependency>
	<groupId>org.neo4j</groupId>
	<artifactId>neo4j-ogm-http-driver</artifactId>
</dependency>

// 1
@NodeEntity(label = "Bot")
public class BotNode {
    @Id
    @GeneratedValue
    private Long id; //id
    @Property(name = "name")
    private String name;//名
    @Property(name = "kind")
    private String kind;//类
    @Property(name = "weight")
    private long weight;//权重

    public BotNode() {
    }

    public BotNode(Long id, String name, String kind, long weight) {
        this.id = id;
        this.name = name;
        this.kind = kind;
        this.weight = weight;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getKind() {
        return kind;
    }

    public void setKind(String kind) {
        this.kind = kind;
    }

    public long getWeight() {
        return weight;
    }

    public void setWeight(long weight) {
        this.weight = weight;
    }

    @Override
    public String toString() {
        return "BotNode{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", kind='" + kind + '\'' +
                ", weight=" + weight +
                '}';
    }

}

// 2 有必要说明一下， @StartNode 和@EndNode注释的类可以不是同一个类。
@RelationshipEntity(type = "BotRelation")
public class BotRelation {
    @Id
    @GeneratedValue
    private Long id;
    @StartNode
    private BotNode startNode;
    @EndNode
    private BotNode endNode;
    @Property
    private String relation;

    public BotRelation() {
    }

    public BotRelation(Long id, BotNode startNode, BotNode endNode, String relation) {
        this.id = id;
        this.startNode = startNode;
        this.endNode = endNode;
        this.relation = relation;
    }

    public String getRelation() {
        return relation;
    }

    public void setRelation(String relation) {
        this.relation = relation;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public BotNode getStartNode() {
        return startNode;
    }

    public void setStartNode(BotNode startNode) {
        this.startNode = startNode;
    }

    public BotNode getEndNode() {
        return endNode;
    }

    public void setEndNode(BotNode endNode) {
        this.endNode = endNode;
    }

    @Override
    public String toString() {
        return "BotRelation{" +
                "id=" + id +
                ", startNode=" + startNode +
                ", endNode=" + endNode +
                ", relation='" + relation + '\'' +
                '}';
    }
}

// 3
@Repository
public interface BotRepository extends Neo4jRepository<BotNode,Long> {
    BotNode findAllByName(String name);
}

// 4
@Repository
public interface BotRelationRepository extends Neo4jRepository<BotRelation,Long> {
    //返回节点n以及n指向的所有节点与关系
    @Query("MATCH p=(n:Bot)-[r:BotRelation]->(m:Bot) WHERE id(n)={0} RETURN p")
    List<BotRelation> findAllByBotNode(BotNode botNode);

    //返回节点n以及n指向或指向n的所有节点与关系
    @Query("MATCH p=(n:Bot)<-[r:BotRelation]->(m:Bot) WHERE m.name={name} RETURN p")
    List<BotRelation> findAllBySymptom(@Param("name") String name);

    //返回节点n以及n指向或指向n的所有节点以及这些节点间的所有关系
    @Query("MATCH p=(n:Bot)<-[r:BotRelation]->(m:Bot)<-[:BotRelation]->(:Bot)<-[:BotRelation]->(n:Bot) WHERE n.name={name} RETURN p")
    List<BotRelation> findAllByStartNode(@Param("name") String name);
}

// 5
@Repository
public interface MovieRepository extends Neo4jRepository<Movie, Long> {
    @Query("MATCH (n:Movie) where id(n)={id}  RETURN n")
    Movie findAllById(@Param("id") Long id);
}

// 6
@RunWith(SpringRunner.class)
@SpringBootTest
public class Neo4jApplicationTests {
    @Autowired
    MovieRepository movieRepository;

    @Test
    public void contextLoads() {
        movieRepository.save(new Movie("《奥特曼》"));
        System.out.println(movieRepository.findAll());
        
        // 保存关系
        MedicalNode node = new MedicalNode(-1l,"节点","测试");
        medicalNodeRepository.save(node);
        MedicalNode node1 = new MedicalNode(-1l,"节点","测试");
        medicalNodeRepository.save(node1);
        medicalRelationRepository.save(new MedicalRelation(-1l,node,node1,"关系"));
    }
    
    @Test
    public void updata(){
        Movie movie = movieRepository.findAllById(8183l);
        movie.setName("《迪迦》");
        movieRepository.save(movie);
        System.out.println(movieRepository.findAll());
    }

}
```



##### neo4j.conf

```
For more details and a complete list of settings, please see https://neo4j.com/docs/operations-manual/current/reference/configuration-settings/

# 如果想自定义neo4j数据库数据的存储路径，要同时修改dbms.active_database 和 dbms.directories.data 两项配置，
# 修改配置后，数据会存放在${dbms.directories.data}/databases/${dbms.active_database} 目录下
# 安装的数据库的名称，默认使用${NEO4J_HOME}/data/databases/graph.db目录
# The name of the database to mount  
#dbms.active_database=graph.db

#安装Neo4j数据库的各个配置路径，默认使用$NEO4J_HOME下的路径
#Paths of directories in the installation. 
# 数据路径
#dbms.directories.data=data  
# 插件路径
#dbms.directories.plugins=plugins  
#dbms.directories.certificates=certificates  证书路径
#dbms.directories.logs=logs 日志路径
#dbms.directories.lib=lib jar包路径
#dbms.directories.run=run 运行路径

#默认情况下想load csv文件，只能把csv文件放到${NEO4J_HOME}/import目录下，把下面的#删除后，可以在load csv时使用绝对路径，这样可能不安全
#This setting constrains all `LOAD CSV` import files to be under the `import` directory. Remove or comment it out to allow files to be loaded from anywhere in the filesystem; this introduces possible security problems. See the `LOAD CSV` section of the manual for details.  
#此设置将所有“LOAD CSV”导入文件限制在`import`目录下。删除注释允许从文件系统的任何地方加载文件;这引入了可能的安全问题。
dbms.directories.import=import

#把下面这行的#删掉后，连接neo4j数据库时就不用输密码了
#Whether requests to Neo4j are authenticated. 是否对Neo4j的请求进行了身份验证。
#To disable authentication, uncomment this line 要禁用身份验证，请取消注释此行。
#dbms.security.auth_enabled=false

#Enable this to be able to upgrade a store from an older version. 是否兼容以前版本的数据
dbms.allow_format_migration=true

#Java Heap Size: by default the Java heap size is dynamically calculated based on available system resources. Java堆大小：默认情况下，Java堆大小是动态地根据可用的系统资源计算。
#Uncomment these lines to set specific initial and maximum heap size. 取消注释这些行以设置特定的初始值和最大值
#dbms.memory.heap.initial_size=512m
#dbms.memory.heap.max_size=512m

#The amount of memory to use for mapping the store files, in bytes (or kilobytes with the 'k' suffix, megabytes with 'm' and gigabytes with 'g'). 用于映射存储文件的内存量（以字节为单位）千字节带有'k'后缀，兆字节带有'm'，千兆字节带有'g'）。
#If Neo4j is running on a dedicated server, then it is generally recommended to leave about 2-4 gigabytes for the operating system, give the JVM enough heap to hold all your transaction state and query context, and then leave the rest for the page cache. 如果Neo4j在专用服务器上运行，那么通常建议为操作系统保留大约2-4千兆字节，为JVM提供足够的堆来保存所有的事务状态和查询上下文，然后保留其余的页面缓存 。
#The default page cache memory assumes the machine is dedicated to running Neo4j, and is heuristically set to 50% of RAM minus the max Java heap size.  默认页面缓存存储器假定机器专用于运行Neo4j，并且试探性地设置为RAM的50％减去最大Java堆大小。
#dbms.memory.pagecache.size=10g


### Network connector configuration

#With default configuration Neo4j only accepts local connections. Neo4j默认只接受本地连接(localhost)
#To accept non-local connections, uncomment this line:  要接受非本地连接，请取消注释此行
dbms.connectors.default_listen_address=0.0.0.0 (这是删除#后的配置，可以通过ip访问)

#You can also choose a specific network interface, and configure a non-default port for each connector, by setting their individual listen_address. 还可以选择特定的网络接口，并配置非默认值端口，设置它们各自的listen_address

#The address at which this server can be reached by its clients. This may be the server's IP address or DNS name, or it may be the address of a reverse proxy which sits in front of the server. This setting may be overridden for individual connectors below. 客户端可以访问此服务器的地址。这可以是服务器的IP地址或DNS名称，或者可以是位于服务器前面的反向代理的地址。此设置可能会覆盖以下各个连接器。
#dbms.connectors.default_advertised_address=localhost

#You can also choose a specific advertised hostname or IP address, and configure an advertised port for each connector, by setting their individual advertised_address. 您还可以选择特定广播主机名或IP地址，
为每个连接器配置通告的端口，通过设置它们独特的advertised_address。

#Bolt connector 使用Bolt协议
dbms.connector.bolt.enabled=true
dbms.connector.bolt.tls_level=OPTIONAL
dbms.connector.bolt.listen_address=:7687

#HTTP Connector. There must be exactly one HTTP connector. 使用http协议
dbms.connector.http.enabled=true
dbms.connector.http.listen_address=:7474

#HTTPS Connector. There can be zero or one HTTPS connectors. 使用https协议
dbms.connector.https.enabled=true
dbms.connector.https.listen_address=:7473

#Number of Neo4j worker threads. Neo4j线程数
#dbms.threads.worker_count=


#Logging configuration  日志配置

#To enable HTTP logging, uncomment this line  要启用HTTP日志记录，请取消注释此行
dbms.logs.http.enabled=true

#Number of HTTP logs to keep. 要保留的HTTP日志数
#dbms.logs.http.rotation.keep_number=5

#Size of each HTTP log that is kept. 每个HTTP日志文件的大小
dbms.logs.http.rotation.size=20m

#To enable GC Logging, uncomment this line 要启用GC日志记录，请取消注释此行
#dbms.logs.gc.enabled=true

#GC Logging Options see http://docs.oracle.com/cd/E19957-01/819-0084-10/pt_tuningjava.html#wp57013 for more information.  GC日志记录选项 有关详细信息，请参见http://docs.oracle.com/cd/E19957-01/819-0084-10/pt_tuningjava.html#wp57013
#dbms.logs.gc.options=-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintPromotionFailure -XX:+PrintTenuringDistribution

#Number of GC logs to keep. 要保留的GC日志数
#dbms.logs.gc.rotation.keep_number=5

#Size of each GC log that is kept. 保留的每个GC日志文件的大小
#dbms.logs.gc.rotation.size=20m

#Size threshold for rotation of the debug log. If set to zero then no rotation will occur. Accepts a binary suffix "k", "m" or "g".  调试日志旋转的大小阈值。如果设置为零，则不会发生滚动(达到指定大小后切割日志文件)。接受二进制后缀“k”，“m”或“g”。
#dbms.logs.debug.rotation.size=20m

#Maximum number of history files for the internal log. 最多保存几个日志文件
#dbms.logs.debug.rotation.keep_number=7


### Miscellaneous configuration  其他配置


#Enable this to specify a parser other than the default one. 启用此选项可指定除默认解析器之外的解析器
#cypher.default_language_version=3.0

#Determines if Cypher will allow using file URLs when loading data using `LOAD CSV`. Setting this value to `false` will cause Neo4j to fail `LOAD CSV` clauses that load data from the file system.    确定当使用加载数据时，Cypher是否允许使用文件URL `LOAD CSV`。将此值设置为`false`将导致Neo4j不能通过互联网上的URL导入数据，`LOAD CSV` 会从文件系统加载数据。
dbms.security.allow_csv_import_from_file_urls=true

#Retention policy for transaction logs needed to perform recovery and backups.  执行恢复和备份所需的事务日志的保留策略
#dbms.tx_log.rotation.retention_policy=7 days

#Enable a remote shell server which Neo4j Shell clients can log in to.  启用Neo4j Shell客户端可以登录的远程shell服务器
dbms.shell.enabled=true
#The network interface IP the shell will listen on (use 0.0.0.0 for all interfaces).
dbms.shell.host=127.0.0.1
#The port the shell will listen on, default is 1337.
dbms.shell.port=1337

#Only allow read operations from this Neo4j instance. This mode still requires write access to the directory for lock purposes.  只允许从Neo4j实例读取操作。此模式仍然需要对目录的写访问以用于锁定目的。
#dbms.read_only=false

#Comma separated list of JAX-RS packages containing JAX-RS resources, one package name for each mountpoint. The listed package names will be loaded under the mountpoints specified. Uncomment this line to mount the org.neo4j.examples.server.unmanaged.HelloWorldResource.java from neo4j-server-examples under /examples/unmanaged, resulting in a final URL of http://localhost:7474/examples/unmanaged/helloworld/{nodeId}      包含JAX-RS资源的JAX-RS软件包的逗号分隔列表，每个安装点一个软件包名称。所列出的软件包名称将在指定的安装点下加载。取消注释此行以装载org.neo4j.examples.server.unmanaged.HelloWorldResource.java neo4j-server-examples下/ examples / unmanaged，最终的URL为http//localhost7474/examples/unmanaged/helloworld/{nodeId}
#dbms.unmanaged_extension_classes=org.neo4j.examples.server.unmanaged=/examples/unmanaged


#JVM Parameters  JVM参数

#G1GC generally strikes a good balance between throughput and tail latency, without too much tuning. G1GC通常在吞吐量和尾部延迟之间达到很好的平衡，而没有太多的调整。
dbms.jvm.additional=-XX:+UseG1GC

#Have common exceptions keep producing stack traces, so they can be debugged regardless of how often logs are rotated. 有共同的异常保持生成堆栈跟踪，所以他们可以被调试，无论日志被旋转的频率
dbms.jvm.additional=-XX:-OmitStackTraceInFastThrow

#Make sure that `initmemory` is not only allocated, but committed to the process, before starting the database. This reduces memory fragmentation, increasing the effectiveness of transparent huge pages. It also reduces the possibility of seeing performance drop due to heap-growing GC events, where a decrease in available page cache leads to an increase in mean IO response time. Try reducing the heap memory, if this flag degrades performance.    确保在启动数据库之前，“initmemory”不仅被分配，而且被提交到进程。这减少了内存碎片，增加了透明大页面的有效性。它还减少了由于堆增长的GC事件而导致性能下降的可能性，其中可用页面缓存的减少导致平均IO响应时间的增加。如果此标志降低性能，请减少堆内存。    
dbms.jvm.additional=-XX:+AlwaysPreTouch

#Trust that non-static final fields are really final. This allows more optimizations and improves overall performance. NOTE: Disable this if you use embedded mode, or have extensions or dependencies that may use reflection or serialization to change the value of final fields!    信任非静态final字段真的是final。这允许更多的优化和提高整体性能。注意：如果使用嵌入模式，或者有可能使用反射或序列化更改最终字段的值的扩展或依赖关系，请禁用此选项！
dbms.jvm.additional=-XX:+UnlockExperimentalVMOptions
dbms.jvm.additional=-XX:+TrustFinalNonStaticFields

#Disable explicit garbage collection, which is occasionally invoked by the JDK itself.  禁用显式垃圾回收，这是偶尔由JDK本身调用。
dbms.jvm.additional=-XX:+DisableExplicitGC

#Remote JMX monitoring, uncomment and adjust the following lines as needed. Absolute paths to jmx.access and jmx.password files are required.  远程JMX监视，取消注释并根据需要调整以下行。需要jmx.access和jmx.password文件的绝对路径。
#Also make sure to update the jmx.access and jmx.password files with appropriate permission roles and passwords, the shipped configuration contains only a read only role called 'monitor' with password 'Neo4j'. 还要确保使用适当的权限角色和密码更新jmx.access和jmx.password文件，所配置的配置只包含名为“monitor”的只读角色，密码为“Neo4j”。
#For more details, see: http://download.oracle.com/javase/8/docs/technotes/guides/management/agent.html On Unix based systems the jmx.password file needs to be owned by the user that will run the server, and have permissions set to 0600. Unix系统，有关详情，请参阅：http：//download.oracle.com/javase/8/docs/technotes/guides/management/agent.html，jmx.password文件需要由运行服务器的用户拥有，并且权限设置为0600。
#For details on setting these file permissions on Windows see: http://docs.oracle.com/javase/8/docs/technotes/guides/management/security-windows.html   Windows系统  有关在设置这些文件权限的详细信息，请参阅：http://docs.oracle.com/javase/8/docs/technotes/guides/management/security-windows.html
#dbms.jvm.additional=-Dcom.sun.management.jmxremote.port=3637
#dbms.jvm.additional=-Dcom.sun.management.jmxremote.authenticate=true
#dbms.jvm.additional=-Dcom.sun.management.jmxremote.ssl=false
#dbms.jvm.additional=-Dcom.sun.management.jmxremote.password.file=/absolute/path/to/conf/jmx.password
#dbms.jvm.additional=-Dcom.sun.management.jmxremote.access.file=/absolute/path/to/conf/jmx.access

#Some systems cannot discover host name automatically, and need this line configured:  某些系统无法自动发现主机名，需要配置以下行：
#dbms.jvm.additional=-Djava.rmi.server.hostname=$THE_NEO4J_SERVER_HOSTNAME

#Expand Diffie Hellman (DH) key size from default 1024 to 2048 for DH-RSA cipher suites used in server TLS handshakes. 对于服务器TLS握手中使用的DH-RSA密码套件，将Diffie Hellman（DH）密钥大小从默认1024展开到2048。
#This is to protect the server from any potential passive eavesdropping. 这是为了保护服务器免受任何潜在的被动窃听。
dbms.jvm.additional=-Djdk.tls.ephemeralDHKeySize=2048


### Wrapper Windows NT/2000/XP Service Properties  包装器Windows NT / 2000 / XP服务属性包装器Windows NT / 2000 / XP服务属性

#WARNING - Do not modify any of these properties when an application using this configuration file has been installed as a service.  WARNING - 当使用此配置文件的应用程序已作为服务安装时，不要修改任何这些属性。
#Please uninstall the service before modifying this section.  The service can then be reinstalled. 请在修改此部分之前卸载服务。 然后可以重新安装该服务。

#Name of the service 服务的名称
dbms.windows_service_name=neo4j


### Other Neo4j system properties  其他Neo4j系统属性
dbms.jvm.additional=-Dunsupported.dbms.udc.source=zip
```



##### 参考:

[史上最全Neo4j使用指南](https://cloud.tencent.com/developer/article/1336299)

对于中国的Neo4j学习者的一些推荐资料
[Neo4j官网](https://neo4j.com/docs/)
[Neo4j中文社区](http://neo4j.com.cn/)
[neo4j亚太区技术专家 俞方桦 博士 中文社区文章](http://neo4j.com.cn/user/graphway)
[neo4j亚太区技术专家 俞方桦 博士 csdn文章](https://blog.csdn.net/graphway/)
[大白 知乎 回答](https://www.zhihu.com/people/liu-da-bai-50/answers)

[developer-manual](https://neo4j.com/docs/developer-manual/current/)
[ogm-manual](https://neo4j.com/docs/ogm-manual/current/)
[java](https://neo4j.com/docs/java-reference/current/)
[rest-docs](https://neo4j.com/docs/rest-docs/current/)

[neo4j-rest-documentation-3.4.pdf ](https://neo4j.com/docs/pdf/neo4j-rest-documentation-3.4.pdf)
[neo4j-java-reference-3.4.pdf](https://neo4j.com/docs/pdf/neo4j-java-reference-3.4.pdf)
[neo4j-ogm-manual-3.1.pdf](https://neo4j.com/docs/pdf/neo4j-ogm-manual-3.1.pdf)
[neo4j-graph-algorithms-3.4.pdf](https://neo4j.com/docs/pdf/neo4j-graph-algorithms-3.4.pdf)
[neo4j-developer-manual-3.4-java.pdf](https://neo4j.com/docs/pdf/neo4j-developer-manual-3.4-java.pdf)

[docs](http://neo4j.com/developer)
[Free Neo4j e-books: Graph Databases](http://graphdatabases.com/)
[Online training classes](http://weikeqin.cn/graphacademy/online-training/)

[operations-manual](https://neo4j.com/docs/operations-manual/current/)
[operations-manual.pdf](https://neo4j.com/docs/pdf/neo4j-operations-manual-current.pdf)

[groups.google.neo4j](https://groups.google.com/forum/#!forum/neo4j)

neo4j导入csv文件
https://neo4j.com/developer/kb/how-do-i-define-a-load-csv-fieldterminator-in-hexidecimal-notation/

使用batch-import工具向neo4j中导入海量数据
https://my.oschina.net/u/2538940/blog/883829　　

前端展示
https://bl.ocks.org/mbostock/4062045　　Force-Directed Graph
[http://visjs.org/](http://visjs.org/)