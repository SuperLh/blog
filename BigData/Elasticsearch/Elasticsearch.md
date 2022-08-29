# Elasticsearch

## 简介

- Elasticsearch是一个基于Lucence的开源的分布式全文检索引擎，它可以近乎实时的存储、检索数据



## 核心概念

- Near RealTime
  - ES号称对外提供的是近实时的搜索服务，数据从写入到可以被搜索仅仅需要一秒钟，基于ES执行的搜索和分析可以达到秒级

- Cluster
  - 集群，由一个或多个Node组成，一起保存存放进去的数据，用户可以在所有的Node质检进行检索
- Node
  - 节点，每个server就是一个Node，不是每个机器，一个机器可以开启多个服务
- Index
  - 索引，一类拥有相似属性的document的集合，索引名称必须是小写，类似于关系型数据库概念中的“库”
- Type
  - 类型，逻辑类型，用于区分Index中的document，新版本ES已经取消，类似于关系型数据库概念中的“表”
- Document
  - 文档，存储的一条数据，类似于关系型数据库概念中的“行”
- Field
  - 字段，存储数据中的某一个属性，类似于关系型数据库概念中的“列”
- Shard
  - 如果让一个index存储过量的数据，响应速度就会下降，为了解决这个问题，ES将数据做了index分片，每个分片叫做Shard，实现了将整体庞大的数据分布在不同的服务器上进行存储
  - Shard分为primary shard和replica shard，一个是主shard，一个事备份Shard，负责容错以及承担部分读请求
  - Shard可以理解为ES中最小的工作单元，所有Shard中的数据之和，才是整个ES中存储的数据
  - 每个Shard可以理解成是一个luncence的实现，拥有完整的创建索引，处理请求的能力
  - 主Shard和备份Shard不会同时存在于同一个Node上
  - 主Shard数量在创建索引后，不能再进行修改，备份Shard的数据量可以修改，进行横向扩展



## 入门指令

- 查看集群的健康状态

  ```json
  GET /_cat/health?v
  
  // 集群的三种状态：
  //   green：当前集群中所有的节点全部可用
  //   yellow：当前es中的所有数据都可以访问，但并不是所有备份Shard都能使用
  //   red：集群宕机，数据不可访问
  ```

  

- 查看集群的索引信息

  ```json
  GET /_cat/indices?v
  ```



- 创建索引

  ```json
  PUT /product?pretty
  ```

  

- 添加

  ```json
  // 不指定ID时，均为新增
  POST /product/_doc?pretty
  {
  	"name" : "BMW"
  }
  ```

  

- 添加/修改

  ```json
  // 如果ES中没有id为1的数据，则新增，如果有则覆盖
  PUT /product/_doc/1?pretty
  {
  	"name" : "Audi"
  }
  ```
  



- 强制创建，如果库中已存在，则报错

  ```json
  PUT /product/_doc/1?op_type=create
  PUT /product/_doc/1/_create
  {
  	"name" : "Audi"
  }
  ```

  

- 更新，对原有所有字段进行覆盖

  ```json
  POST /product/_doc/1?pretty
  {
    "name" : "Audi111"
  }
  ```

  

- 局部更新

  ```json
  POST /product/_doc/1/_update
  {
    "doc" : {
        "name" : "Audi111"
    }
  }
  ```



- 检索

  ```json
  GET /product/_doc/1?pretty
  ```



- 删除

  ```json
  // 删除操作，大部分情况下，不会被立即删除，而是被标记为deleted，被标记成deleted的数据不会被检索，当数据越来越多时，才会真正删除
  DELETE /product/_doc/1
  ```
  
  

- 搜索

  ```json
  GET /product/_search
  ```



- 批量查询

  ```json
  GET /ecommerce/_mget
  {
    "docs":[
      {
        "_id": "9ssT6oIBQoCeMYCUEHcl"
      },
      {
        "_id": "98sT6oIBQoCeMYCUEHcl"
      }
    ]
  }
  ```



## 分页查询

### Deep Paging问题 

- 假设某个索引有四个分片，当我们需要进行排序查询时，查询数据量时没有问题，当查询数据量过大时，就会出现Deep Paging问题

- 比如说，我们要按某个字段对这个索引进行排序查询前1w条数据时，我们需要从该索引的四个分片中，分别各抽取前1w条数据，然后在进行聚合，对这4w条数据再次进行排序，然后取出前1w条数据

- 这也是在分布式系统中，对结果排序的成本都随分页的深度成指数上升

  

### Scroll遍历查询

- ES官方不再推荐使用Scroll API进行深度分页，如果需要在分页超过10000个点击时，保留索引状态，请使用带有时间点（PIT）的search_after

- Scroll原理是对某次查询生成一个游标scroll_id，后续的查询只需要根据这个游标去取数据，直到结果集中返回的hits字段为空，则结束

- Scroll可以理解为建立了一个临时的历史快照，在此之后的增删改查等操作不会影响这个快照的结果

- 所有文档获取完毕之后，需要手动清理掉scroll_id，虽然es有自动清理机制，但是scroll_id的存在会消耗大量的资源来保存当前查询结果

  ```json
  // 首次查询，并获取scroll_id
  // scroll=1m，滚动查询有效时间1min
  GET /ecommerce/_search?scroll=1m
  {
    "query": {
      "match_all": {}
    },
    "sort": [
      {
        "products.base_price": {
          "order": "desc"
        }
      }
    ],
    "size": 10
  }
  
  // 根据scroll_id遍历数据
  POST /_search/scroll
  {
    "scroll": "1m",
    "scroll_id" : "FGluY2x1ZGVfY29udGV4dF91dWlkDXF1ZXJ5QW5kRmV0Y2gBFFhNdGI2b0lCUW9DZU1ZQ1VsWkpLAAAAAAAACLAWQmxoWEVxSmZUQzIyd1BZU1N3UmthUQ=="
  }
  
  // 删除游标
  DELETE /_search/scroll
  {
    "scroll_id" : "FGluY2x1ZGVfY29udGV4dF91dWlkDXF1ZXJ5QW5kRmV0Y2gBFFhNdGI2b0lCUW9DZU1ZQ1VsWkpLAAAAAAAACLAWQmxoWEVxSmZUQzIyd1BZU1N3UmthUQ=="
  }
  ```

- 优缺点

  - scroll查询的相应数据是非实时的，如果遍历的过程中，插入了新的数据，是查询不到的
  - 需要较大的内存空间

- 适用场景

  - 全量或者数据量很大时，遍历结果，而非分页查询



## 查询语法

### TimeOut机制

- 假设用户查询结果有1w数据，但是需要10s查询完毕，但是用户设置了1s的timeout，那么不管当前一共查询到了多少数据，都会在1s后停止查询

  ```json
  GET /ecommerce/_search?from=0&size=200&sort=order_id:asc&timeout=1ms

### query_string

- 将所有请求参数全部写到url中，较少使用

  ```json
  GET /ecommerce/_search?q=currency:EUR
  GET /ecommerce/_search?from=0&size=2&sort=order_id:asc
  ```



### Query DSL

- 查询指定index下的所有doc

  ```json
  GET /ecommerce/_search
  ```

  

- 针对某个字段进行全文检索（match）

  ```json
  // ES会将用户输入的字符串通过分词器拆开，然后扫描
  GET /ecommerce/_search
  {
    "query": {
      "match": {
        "category": "Clothing"
      }
    }
  }
  ```

  

- 全文检索：手动控制全文检索的精度

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "match": {
        "category": {
          "query": "Clothing Shoes",
          "operator": "and"
        }
      }
    }
  }
  ```

  

- 全文检索：控制匹配数量

  ```json
  // 三个条件满足两个即可
  GET /ecommerce/_search
  {
    "query": {
      "match": {
        "category": {
          "query": "Clothing Shoes Men's",
          "operator": "or",
          "minimum_should_match":2
        }
      }
    }
  }
  ```



- 全文检索：通过boost控制权重

  ```json
  // must必须条件，权重统一
  // should条件中，customer权重为2，day_of_week权重为9
  GET /ecommerce/_search
  {
    "query": {
      "bool": {
        "must": {
          "match": {
            "currency" : {
              "query":"EUR"
            }
          }
        },
        "should": [
          {
            "match": {
              "customer_id": {
                "query": 38,
                "boost": 2
              }
            }
          },
          {
            "match": {
              "day_of_week": {
                "query": "Monday",
                "boost": 9
              }
            }
          }
        ]
      }
    }
  }
  ```



- 复杂查询：bool查询

  ```json
  // must:day_of_week必须是Monday
  // should:category尽可能是Clothing
  // must_not:category必须不是Men's
  GET /ecommerce/_search
  {
    "query": {
      "bool": {
        "must": [
          {"match": {
            "day_of_week": "Monday"
          }}
        ],
        "should": [
          {"match": {
            "category": "Clothing"
          }}
        ],
        "must_not": [
          {"match": {
            "category": "Men's"
          }}
        ]
      }
    }
  }
  ```

  

- Best Fields策略：选取多个query中得分最高的分作为doc的最终得分

  ```json
  // 选取下面规则中最高分
  // tie_breaker:也考虑其他field的得分影响，但是他的影响会被弱化成原来的0.7
  GET /ecommerce/_search
  {
    "query": {
      "tie_breaker": 0.7, 
      "dis_max": {
        "queries": [
          {"match": {
            "category": "Clothing"
          }},
          {"match": {
            "day_of_week": "Monday"
          }}
        ]
      }
    }
  }
  ```

  

- 同时指定多个字段进行检索

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "multi_match": {
        "query": "Underwood",
        "fields": ["customer_last_name", "customer_full_name"]
      }
    }
  }
  ```

  

  





https://developer.aliyun.com/article/919943?spm=a2c6h.24874632.expert-profile.113.260e3580zeARSr































