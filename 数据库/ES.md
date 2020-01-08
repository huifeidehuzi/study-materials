# ES（v6.x）
##API文档
`restful api`：https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html 

`java api`：https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
##简介
* `index` 索引：必须小写，不能有下划线和逗号，等同于mysql的库
* `type` 类型：不能有下划线和逗号，等同于mysql的表，注：6.0版本后一个index下只能有一个type
* `doc` 文档：等同于mysql行
* `mapping`：等同于mysql表结构，且mapping创建后不能修改  
* `number_of_shards`：分片数量
* `number_of_replicas`：每个分片对应的副本数量
* `ik`：中文分词器 sudo elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.8.2/elasticsearch-analysis-ik-6.8.2.zip


##mapping
###1.手动创建mapping
```
PUT /{index}
 {
  "mappings": {
    "{type}":{
      "properties":{
        "{字段}" : {
            "type" : "{类型,如：text，keyword，long，double，date等}",
            "analyzer":"english" //分词器，如english，ik等等 
          },
          "{字段}" : {
            "type" : "{类型,如：text，keyword，long，double，date等}"
          }
      },
      "dynamic_templates":{ //动态映射模板
        "{自定义模板名称}":{ 
          "match_mapping_type": "string", //匹配映射类型
          "match":   "long_*", //匹配以long_开头的字段
          "unmatch": "*_text", //不匹配以_text结尾的字段
          "mapping": { //映射
            "type": "long" /类型
          }
        }
      }
    }
  }
}
```
###2.查看mapping
`GET /{index}/_mapping/{type}`
##文档
###1.操作文档
* 新增文档：不传id，es会基于GUID算法自动生成id，并且在并发场景下也能保证唯一

```
POST或PUT /{index}/{type}/{id}
{
  "整形": 123,
  "字符串": "i like java",
  "对象": {
    "address": "北京市东城区",
    "lat": 113.44,
    "lng": 128.33
  },
  "数组":[
    1,
    2,
    3
  ],
  "数组对象":[
    {"name":"张三"},
    {"name":"李四"}
  ]
}         
```
* 全量修改文档：修改文档内所有字段，先标记原有的doc为delete，然后再创建一个新的doc，es会在合适的时间真正删除掉被标记delete的doc，比如磁盘不够的情况下

```
PUT /{index}/{type}/{id}
{
  "整形": 234,
  "字符串": "i like java too",
  "对象": {
    "address": "北京市西城区",
    "lat": 113.44,
    "lng": 128.33
  },
  "数组":[
    1,
    2,
    3
  ],
  "数组对象":[
    {"name":"王柳"},
    {"name":"李商"}
  ]
}   
```
* 部分修改文档：修改文档内部分字段，先查询出doc，封装新doc，标记原doc为delete，再创建封装好的新doc，出现并发时，默认是用乐观锁（version字段）来控制，es会在合适的时间真正删除掉被标记delete的doc，比如磁盘不够的情况下

```
PUT /{index}/{type}/{id}?_update
{
  "修改的字段": "修改的值"
}   
```
* `bulk` 批量操作文档


```
//bulk api对json的语法，有严格的要求，每个json串不能换行，只能放一行，同时一个json串和一个json串之间，必须有一个换行
POST /_bulk
//删除文档
{"delete":{"_index":"employee","_type":"user","_id":"5"}}
//创建文档，_id如果存在会报错
{"create":{"_index":"employee","_type":"user","_id":"5"}} 
{"name":"小王八","age":12}
//创建/修改文档，id不存在即新增，反之全量修改文档
{"index":{"_index":"employee","_type":"user","_id":"6"}}
{"name":"大鸡儿","age":100}
//修改文档，部分修改
{"update":{"_index":"employee","_type":"user","_id":"5"}}
{"doc":{"name":"小王八","age":22}}
```
* `新增文档原理`

```
 1.请求某一节点
 2.通过路由算法找到对应分片。
 3.如果分片不在该请求的节点上，该节点会转发请求至对应节点
 4.存储文档且同步文档至对应所有的副本分片上
```
* `路由算法` shard = hash(routingkey)%shards

```
1.新增和修改文档会有个routingkey ，默认是文档的id，也可以手动指定
2.计算此routingkey的hash值
3.hash值%分片个数,取余数(分片个数一旦确定就不能更改了，原因就是会影响油路算法)
```
* `consistency` 策略验证，保证主、副本分片数据一致性

```
//三个参数值如下
1.one：有一个活跃的主分片就可以执行成功
2.all：所有的主分片和副本分片都存活才能执行成功
3.quorum：半数及以上分片(主+对应副本分片数量)存活才能执行成功，默认此值，计算规则：(主+对应副本分片数量)/2+1，注：这里副本分片数量并不是一共有多少副本，而是主分片对应的副本分片数量，如果活跃个数没有达到要求的个数，则默认等待1分钟，1分钟内活跃个数没有达到要求个数，则timeout
设置超时时间:带上timeout参数。如：put /index/type/id?timeout=10s
```
###2.搜索文档
* 按id搜索 

```
GET /{index}/{type}/{id}
```
* `query string` 

```
//按指定字段搜索
GET /{index}/{type}/_search?q=搜索字段:关键词&sort=排序字段:desc/asc
//按关键词搜索全部字段，性能低
GET /{index}/{type}/_search?q=关键词

```
* `term` 对关键词搜索，不会对搜索字段分词 ，source（只展示此字段)、highlight(高亮搜索匹配到某个字段的内容)、sort、from、size以上为可选参数，以下所有查询都一样

```
GET /{index}/{type}/_search
{
  "highlight":{
    "fields": {"要高亮展示的字段":{}//空着即可}
  },
  "from": 从第N条开始取值,
  "size": 取N条文档
  "source":["要展示的字段","要展示的字段"]
  "query": {
    "term": {
      "字段":值
    }
  },
  "sort": [
    {
      "排序字段": {
        "order": "asc"
      }
    }
  ]
}
``` 
* `terms` 对关键词集合搜索，不会对搜索的关键词分词

```
GET /{index}/{type}/_search
{
  "query": {
    "terms": {
      "字段":[值，值]
    }
  }
}
``` 
* `match` 对关键词搜索，会对搜索的关键词分词，如：阿里巴巴  可能会分为阿里、巴巴、阿里巴巴三个词语进行搜索

```
GET /{index}/{type}/_search
{
  "query": {
    "match": {
      "字段":值
    }
  }
}
``` 
* `match_all` 搜索全部文档

```
GET /{index}/{type}/_search
{
  "query": {
    "match_all": {}
  }
}
``` 
* `multi_match` 多字段匹配搜索

```
GET /{index}/{type}/_search
{
  "query": {
    "multi_match": {
      "query": 搜索内容,
      "fields": ["字段","字段"]
    }
  }
}
``` 
* `match_phrase` 短语匹配搜索，匹配其中某一段的值，会对搜索的关键词分词

```
GET /{index}/{type}/_search
{
  "query": {
    "match_phrase": {
      "字段":"值"
    }
  }
}
``` 
* `match_phrase_prefix` 搜索以某个字段某个值开头的文档

```
GET /{index}/{type}/_search
{
  "query": {
    "match_phrase_prefix": {
      "字段":"值"
    }
  }
}
``` 
* `wildcard` 通配符匹配搜索，可用`*`和`?`查询，`*`代表0个或多个字符，如："name":"zhang*"表示查询name是zhang开头或zhang的,`?`代表任意一个字符，如："name:”zhang?an"表示查询名字zhang任意an的数据，避免使用这类搜索，因为非常消耗资源

```
GET /{index}/{type}/_search
{
  "query": {
    "wildcard": {
      "字段":"值* 或者 值?"
    }
  }
}
``` 
* `fuzzy` 模糊搜索，比如数据是zhangsan，关键词为：zsan也能查出来

```
GET /{index}/{type}/_search
{
  "query": {
    "fuzzy": {
      "字段":{
        "value": "值"
      }
    }
  }
}
``` 
* `multiGet` 批量搜索文档

```
//不同索引不同类型
GET /_mget
{
  "docs":[
      {
        "_index":"索引",
        "_type":"类型",
        “_id”:id
      }
    ]
} 
//同个索引不同类型
GET /{index}/_mget
{
  "docs":[
      {
        "_type":"类型",
        “_id”:id
      }
    ]
} 
//同个索引同个类型
GET /{index}/{type}/_mget
{
  "ids":[1,2,3]
}

``` 
* `Filter` 过滤搜索，类似于 SQL 里面的where 语句，和上面的基础查询比起来，也能实现搜索的功能，同时 filter 可以将查询缓存到内存当中,这样可以大大加大下一次的查询速度

```
//可结合term、match等使用
GET /{index}/{type}/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "字段": 值
        }
      }
    }
  }
}
``` 

* `exists` 搜索某个字段不为空的文档

```
GET /{index}/{type}/_search
{
  "query": {
    "exists": {"field": "字段"}
  }
}
``` 

* `Bool` 过滤查询 可嵌套使用

```
//{“bool”:{“must”:[],”should”:[],”must_not”:[]}}
//must:必须满足的条件  =mysql的and
//should: =mysql的or
//must_not: 不需满足的条件，=mysql的 not

GET /{index}/{type}/_search
{
  "query": {
    "bool": {
      "should": [
        {"term": {
          "字段":"值"
        }},
        {"term": {
          "字段": "值"
        }}
      ],
      "must": [
        {"term": {
          "字段":值
        }}
      ],
      "must_not": [
        {"term": {
          "字段":值
        }}
      ]

    }
  }
}
``` 

* `rang` 范围查询

```
//可用from、to ,gt、lt 范围查询的符号
GET /{index}/{type}/_search
{
  {"query": {
    "range": {
      "age": {
        "gte": 10,
        "lte": 20
      }
    }
  }}
}
``` 


* `聚合查询` 

```
//语法 {“size”:0 //获取文档list数量,”aggs”:{“自定义聚合名称”:{“聚合函数”:{“field”:”字段"}}}}
//sum  求和
//avg 平均值
//min 最小值
//max 最大值
//cardinaltiy 求基数(不同值的数量)，
//terms//分组

//查询xxx条件的文档，且求age的平均值，然后对age平均值分组后倒序
GET /{index}/{type}/_search
{
  "query": {
    "match": {
      "name": "zhangsan"
    }
  }, 
  "aggs": {
    "age_avg": {
      "avg": {
        "field": "age",
        "order": {
          "age_group": "desc"
        }
      },"aggs": {
        "age_group": {
          "terms": {
            "field": "age"
          }
        }
      }
    }
  }
}

``` 
* `多index，多type查询` type可以不写，因为6.0之后一个index只能有一个type，也可以把index替换为_all或者不写index 查询所有索引下文档，

```
GET /{index1},{index2}.../{type1},{type2}/_search
//这里与基本查询一致
{
  "query": {
    "条件": {"字段": "值"}
  }
}
``` 
* `分页查询` 通过from和size 实现分页查询，from指从0开始取数，size值取数的数量，尽量不使用deep查询，因为会涉及到多节点间的网络开销，协调节点的内存（存储汇总数据）和cpu（对汇总数据排序取值）开销

```
GET /{index1},{index2}.../{type1},{type2}/_search
//这里与基本查询一致
{
  "query": {
    "条件": {"z": "字段"}
  }
}
``` 
###其他
* `解决字符串排序问题(Doc Values)` 创建索引时，给需要排序的字段新增另外一个类型keyword并创建正排索引，使其即可分词也可排序

```
Doc values
//1.存储在磁盘上
//2.对排序，分组和一些聚合操作能提升很大的性能
//3.默认对分词字段不开启，开启：把fielddata设置为ture
PUT /employee
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "user": {
      "properties": {
        "name": {
          "type": "text",
          "fields": { //新增此属性
            "row": {
              "type": "keyword" //设置keyword类型
            }
          },
          "fielddata": true //设置为true，fileddata默认是不开启的。Fielddata可能会消耗大量的堆空间，尤其是在加载高基数文本字段时。一旦fielddata已加载到堆中，它将在该段的生命周期内保留。此外，加载fielddata是一个昂贵的过程，可能会导致用户遇到延迟命中。这就是默认情况下禁用fielddata的原因
        },
        "age": {
          "type": "integer"
        }
      }
    }
  }
}

```

* `正排索引` 以文档的id为key作为索引，记录每个关键词出现的次数，查找时扫描每个文档中字的信息，直到找到所有包含查询关键字的文档。优点是：易维护（新增文档，只要在表末尾新增一个key，然后把词、词出现的频率和位置信息分析完成后就可以使用了）；缺点是搜索的耗时太长（扫全表）

```
//粗略的一个例子
//如果内容如下，那么进行正排索引建立的表结构大概是下面这样的
内容：tom is a boy tom is a student too
分词：tom is a boy tom is a student too

分词  doc_hit(出现次数)       doc_offset(出现位置)
tom      2                     1,5
is       2                     2,6
a        2                     3,7
boy      1                     4
student  1                     8
too      1                     9
```

*  `倒排索引` 由不同的”词“组成的索引表，称为：词典，其中包含不同的”词“和对应的统计信息（如出现次数等等），会忽略语气词，如：is、and、啊、嗯.....等等，html标签....等，具体的可自行查阅资料

```
//粗略的一个例子
doc1：it is a apple
doc2：it a banana
doc3：it is a orange
//倒排
分词   包含分词的文档
it      doc1,doc2,doc3
is      doc1,doc2
a       doc1,doc2,doc3
apple   doc1
banana  doc2
orange  doc3

//带位置的倒排
it      (doc1,1)(doc2,1)(doc3,1)
is      (doc1,2)(doc2,2)
a       (doc1,3)(doc2,2)(doc3,3)
apple   (doc1,4)
banana  (doc2,3)
orange  (doc3,4)
```

* `scroll` 滚动搜索，用于大数据量的场景，不建议使用在实时场景，解决es深度分页搜索问题（es默认分页搜索数量最多不能查过10000，因为每次分页查询需要把from-size的每个分页上的数据取出来排序（分片数*(from+size)），非常占内存，即使不会oom也会非常占用cpu资源）


```
//原理
1.初始化，将搜索条件保存快照
//指定时间窗口（这里是1m内完成就行）搜索，每次1条，如对结果顺序无要求可以按_doc排序，提高性能（es会默认按相关度分数排序）
GET /employee/user/_search?scroll=1m
{
  "query": {
    "match_all": {}
  },
  "size": 1,
  "sort": ["_doc"]
}
//返回
{
  "_scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAABVUWaFpRMWg3OHlSeGExeGF0bHpOd0pqZw==",
  "took" : 22,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : null,
    "hits" : [
      {
        "_index" : "employee",
        "_type" : "user",
        "_id" : "2",
        "_score" : null,
        "_source" : {
          "name" : "王永天",
          "age" : 30
        },
        "sort" : [
          0
        ]
      }
    ]
  }
}
2. 按_scroll_id继续搜索
GET /_search/scroll
{
  "scroll":"1m", //时间窗口
  "scroll_id":"DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAABVYWaFpRMWg3OHlSeGExeGF0bHpOd0pqZw==" //scrollid
}
```

* `dynamic mapping`

```
dynamic：在创建mapping时设置此属性（与properties同级）,默认为true
    1.ture：遇到mapping中未定义的字段则自动映射
    2.false：遇到mapping中未定义的字段则忽略
    3.strict：遇到mapping中未定义的字段则报错
//例子:
PUT /user
 {
  "mappings": {
    "employee":{
     "dynamic":true, //写在这，表示在employee中出现未定义的字段怎么处理
     "date_detection":false, //如果不设置此属性，遇到'2019-01-01'这种值，会自动映射成日期类型，视场景使用
      "properties":{
        "info" : {
            "type":"object",
            "dynamic":false // //写在这，表示在info中出现未定义的字段怎么处理
          }
      },
      "name":{
        "type":"text"
      }
    }
  }
}

```

* `重建索引且不用重启应用程序`


```
1.创建新索引，设置别名
2.旧索引数据迁移到新索引中，数据量大可用scroll
3.删除旧索引关联别名，新增新索引关联别名
```
###集群相关

* `容错机制` 注：主分片不能与其副本分片在同一节点，主分片对应的多个副本分片也不能在同一节点

```
1.前提是宕机后 分片数据还是完整的，不论主副分片
2.重新选举master节点
3.把丢失的主分片的其中一个副分片提升为主分片
4.服务器恢复后，master会把每个主分片的备份一份到此服务器上，也就是生成主分各自的副本分片
```