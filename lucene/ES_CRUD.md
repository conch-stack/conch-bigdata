### Kibana Shell

```
GET _search
{
  "query": {
    "match_all": {}
  }
}

GET /customer/customer/_mapping
DELETE /myindex
GET customer/customer/_search
GET /customer/_settings


# 添加索引
PUT /myindex/
{
  "settings": {
    "index":{
      "number_of_shards":3,
      "number_of_replicas":0
    }
  }
}

# 查看索引信息
GET /myindex/_settings
GET _all/_settings
GET /myindex/_mapping
GET /myindex/test/_mapping

# 添加数据
PUT /myindex/test/1
{
  "first_name": "Jane",
  "last_name":"smith",
  "age":32,
  "about":"I Like to collect rock albums",
  "interests":["music"]
}

# 不指定id 用POST 
POST /myindex/test/
{
  "first_name": "Douglas",
  "last_name":"Fir",
  "age":23,
  "about":"I Like to build cabinets",
  "interests":["forestry"]
}

GET /myindex/test/1
GET /myindex/test/_search
# 过滤source
GET /myindex/test/1?_source=age,about

GET /goodscenter_goods/repo/_search
{
  "query": {
    "term": {
      "_id":"_update"
    }
  }
}

# 更新
# 1. 覆盖
PUT /myindex/test/1
{
  "first_name": "Jane",
  "last_name":"smith",
  "age":33,
  "about":"I Like to collect rock albums",
  "interests":["music"]
}

# 2. POST
POST /myindex/test/1/_update
{
  "doc": {
    "age": 34
  }
}

# 删除 
DELETE /myindex/test/aOsYImsBJbHqWal4B2p2

# 批量 
# 批量获取
GET /_mget
{
  "docs":[
    {
      "_index":"myindex",
      "_type":"test",
      "_id":"1",
      "_source":"interests"
    },
    {
      "_index":"myindex",
      "_type":"test",
      "_id":"2"
    },
    {
      "_index":"myindex",
      "_type":"test",
      "_id":"3"
    }
  ]
}

# 简写1 
GET /myindex/test/_mget
{
  "docs":[
    {
      "_id":"1"
    },
    {
      "_type":"test",
      "_id":"2"
    }
  ]
}

# 简写2
GET /myindex/test/_mget
{
  "ids":["1", "2"]
}

# bulk 批量
# {action:{metadata}} \n
# {requestbody}\n
# action: 
#     create 创建文档
#     update 更新文档
#     index  创建新文档或替换已有文档
#     delete 删除一个文档
# metadata： _index、_type、_id
# create 与 index的区别
#     如果数据存在，使用create操作失败，index则可成功执行
# bulk 将数据载入内存，一般建议5-15M，默认最大100M，可在es的config中的elasticsearch.yml中配置


POST /myindex1/test/_bulk
{"index":{"_id":1}}
{"title":"Java","price":33}
{"index":{"_id":2}}
{"title":"Html","price":44}
{"index":{"_id":3}}
{"title":"Python","price":55}

# 删除
POST /myindex1/test/_bulk
{"delete":{"_index":"myindex1","_type":"test","_id":"3"}}
{"create":{"_index":"myindex2","_type":"test","_id":"1"}}
{"name":"lili"}
{"index":{"_index":"myindex2","_type":"test","_id":"1"}}
{"name":"bianle"}
{"update":{"_index":"myindex2","_type":"test","_id":"1"}}
{"doc":{"name":"youbianle"}}

GET /myindex1/test/_mget
{
  "ids":["1", "2","3"]
}

GET /myindex2/test/_search

# 相比brown OR dog，我们更想要的结果是brown AND dog
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": {      
                "query":    "BROWN DOG!",
                "operator": "and"
            }
        }
    }
}

# 控制精度(Controlling Precision)
# 在all和any中选择有种非黑即白的感觉。如果用户指定了5个查询词条，而一份文档只包含了其中的4个呢？将"operator"设置成"and"会将它排除在外。
# 有时候这正是你想要的，但是对于大多数全文搜索的使用场景，你会希望将相关度高的文档包含在结果中，将相关度低的排除在外。换言之，我们需要一种介于两者中间的方案。match查询支持minimum_should_match参数，它能够让你指定有多少词条必须被匹配才会让该文档被当做一个相关的文档。尽管你能够指定一个词条的绝对数量，但是通常指定一个百分比会更有意义，因为你无法控制用户会输入多少个词条：
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query":                "quick brown dog",
        "minimum_should_match": "75%"
      }
    }
  }
}

GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2 
    }
  }
}

# 合并查询(Combining Queries)
# 在合并过滤器中我们讨论了使用bool过滤器来合并多个过滤器以实现and，or和not逻辑。bool查询也做了类似的事，但有一个显著的不同。
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "must":     { "match": { "title": "quick" }},
      "must_not": { "match": { "title": "lazy"  }},
      "should": [
                  { "match": { "title": "brown" }},
                  { "match": { "title": "dog"   }}
      ]
    }
  }
}

# 全文搜索 should查询子句的匹配数量越多，那么文档的相关度就越高
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "match": {
                    "content": { 
                        "query":    "full text search",
                        "operator": "and"
                    }
                }
            },
            "should": [ 
                { "match": { "content": "Elasticsearch" }},
                { "match": { "content": "Lucene"        }}
            ]
        }
    }
}

# 通过指定一个boost值来控制每个查询子句的相对权重，该值默认为1。一个大于1的boost会增加该查询子句的相对权重
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "match": {  
                    "content": {
                        "query":    "full text search",
                        "operator": "and"
                    }
                }
            },
            "should": [
                { "match": {
                    "content": {
                        "query": "Elasticsearch",
                        "boost": 3 
                    }
                }},
                { "match": {
                    "content": {
                        "query": "Lucene",
                        "boost": 2 
                    }
                }}
            ]
        }
    }
}

# boost参数被用来增加一个子句的相对权重(当boost大于1时)，或者减小相对权重(当boost介于0到1时)，但是增加或者减小不是线性的。换言之，boost设为2并不会让最终的_score加倍。相反，新的_score会在适用了boost后被归一化(Normalized)。每种查询都有自己的归一化算法(Normalization Algorithm)，算法的细节超出了本书的讨论范围。但是能够说一个高的boost值会产生一个高的_score。如果你在实现你自己的不基于TF/IDF的相关度分值模型并且你需要对提升过程拥有更多的控制，你可以使用function_score查询，它不通过归一化步骤对文档的boost进行操作。
```

