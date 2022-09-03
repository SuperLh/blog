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

  

- Most Field策略：优先返回有更多的field匹配到你关键字的doc

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "multi_match": {
        "query": "Underwood",
        "fields": ["customer_last_name", "customer_full_name"],
        "type": "most_fields"
      }
    }
  }



- Cross Field策略：穿越查询，比如一类实体存储时，不能存储在单一的field里面，就需要用到穿越查询来确定搜索数据

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "multi_match": {
        "query": "Cairo",
        "fields": ["geoip.continent_name", "geoip.city_name"],
        "type": "cross_fields"
      }
    }
  }
  ```



- 查询空

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "match_none": {}
    }
  }
  ```



- 精确匹配

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "constant_score": {
        "filter": {
          "term": {
            "customer_full_name.keyword": "Mary Bailey"
          }
        }
      }
    }
  }
  ```



- 短语搜索：要求doc的这个字段和给定的值完全相同，顺序也不能变

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "match_phrase": {
        "products.product_name": "Jersey dress"
      }
    }
  }
  ```



- 短语搜索优化：使用slop，让指定短语中的词最多经过slop次移动后如果能匹配某个doc，那么该doc也作为返回结果

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "match_phrase": {
        "customer_full_name": "Bailey Mary",
        "slop": 2
      }
    }
  }
  ```



- 短语搜索优化：使用match和match_phrase平衡精准度和召回率

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "bool": {
        "must": {
          "match": {
            "currency" : "EUR"
          }
        },
        "should": {
          "match_phrase": {
            "customer_full_name": "Mary Bailey",
            "slop": 10
          }
        }
      }
    }
  }
  ```



- 重新打分机制：使用rescore_query重新打分，提高精准度和召回率

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "match": {
        "category": {
          "query": "Women's Clothing",
          "minimum_should_match": "50%"
        }
      }
    },
    "rescore": {
      "query": {
        "rescore_query": {
          "match_phrase": {
            "customer_full_name": "Mary Bailey"
          }
        }
      },
      "window_size": 50
    }
  }
  ```

  

- 前缀搜索

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "prefix": {
        "customer_full_name.keyword": {
          "value": "Mary",
          "boost": 5
        }
      }
    }
  }
  ```



- 通配符搜索

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "wildcard": {
        "customer_full_name.keyword": {
          "value": "Mary*",
          "boost": 5
        }
      }
    }
  }
  ```



- 正则搜索

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "regexp": {
        "products.sku": "ZO04[a-z0-9]+"
      }
    }
  }
  ```

  

- 模糊查询

  ```json
  GET /test/_search
  {
    "query": {
      "fuzzy": {
        "name.姓名.keyword": {
          "value": "吴彦祖",
          "fuzziness": 2,
          "prefix_length": 0
        }
      }
    }
  }
  ```



- 指定要查询的字段

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "match_all": {}
    },
    "_source": ["products._id","products.category"]
  }
  ```

  

- filter过滤：过滤器，不进行相关分数的计算，并且可以缓存

  ```json
  GET /ecommerce/_search
  {
    "query": {
      "bool": {
        "must": [
          {"match": {
            "day_of_week": "Monday"
          }}
        ],
        "filter": [
          {"range": {
            "order_id": {
              "gte": 600000
            }
          }}
        ]
      }
    }
  }
  ```

  

- 高亮显示

  ```json
  GET /test/_search
  {
    "query": {
      "match": {
        "name.姓名": "吴彦祖"
      }
    },
    "highlight": {
      "fields": {"name.姓名": {}}
    }
  }
  ```

  

### 聚合查询

- 聚合查询核心概念
  - bucket：聚合操作得到的结果集
  - metric：对bucket进行分析，比如取最大值，最小值，平均值
  - 下钻：基于现有分好组的bucket继续分组



- 聚合统计个数

  ```json
  // size：
  GET /ecommerce/_search
  {
    "size": 0,
    "aggs": {
      "group_by_name": {
        "terms": {
          "field": "day_of_week"
        }
      }
    }
  }
  ```



- 先进行搜索，然后再对搜索结果聚合

  ```json
  GET /ecommerce/_search
  {
    "size": 0,
    "query": {
      "term": {
        "geoip.city_name": "Cairo"
      }
    }, 
    "aggs": {
      "group_by_name": {
        "terms": {
          "field": "day_of_week"
        }
      }
    }
  }
  ```



- 按照分组，求取平均值

  ```json
  GET /ecommerce/_search
  {
    "size": 0,
    "aggs": {
      "group_by_name": {
        "terms": {
          "field": "day_of_week"
        },
        "aggs": {
          "average_price": {
            "avg": {
              "field": "products.base_price"
            }
          }
        }
      }
    }
  }
  ```



- 按照字段进行范围分组，然后在统计各分组下的结果

  ```json
  GET /ecommerce/_search
  {
    "size": 0,
    "aggs": {
      "group_by_price": {
        "range": {
          "field": "products.base_price",
          "ranges": [
            {
              "from": 10,
              "to": 20
            },{
              "from": 20,
              "to": 30
            },{
              "from": 30,
              "to": 40
            },{
              "from": 40,
              "to": 50
            }
          ]
        },
        "aggs": {
          "group_by_gender": {
            "terms": {
              "field": "day_of_week"
            },
            "aggs": {
              "avg_price": {
                "avg": {
                  "field": "products.base_price"
                }
              }
            }
          }
        }
      }
    }
  }
  ```

  

- 根据日期进行聚合

  ```json
  GET /ecommerce/_search
  {
    "size": 0,
    "aggs": {
      "agg_by_time": {
        "date_histogram": {
          "field": "order_date",
          "interval": "month",
          "format": "yyyy-MM-dd",
          "min_doc_count": 0,
          "extended_bounds": {
            "min": "2022-01-01",
            "max": "2022-12-01"
          }
        }
      }
    }
  }
  ```



## ES支持的核心数据类型

- 数字类型
  - long、integer、short、byte、doubl、float、half_float、scaled_float
- 日期类型
  - date
- boolean类型
  - boolean
- 二进制类型
  - binary
- 范围
  - integer_range、float_range、long_range、double_range、date_range
- 复杂数据类型
  - 对象类型、嵌套对象类型
- geo-type
  - geo_point、geo_line、geo_polygon、geo_shape



## 正排索引&倒排索引

 ![](E:\Private_LiuHui\Private_git\blog\BigData\Elasticsearch\index.jpeg)

- 正排索引
  - 以doc为维度，记录doc中出现了哪些词汇
- 倒排索引
  - 把doc打碎成一个个词条，以词语为维度，记录在哪些doc中出现过



## 分词器

- 创建索引时，指定分词器

  ```json
  // analyzer : 存储时使用的分词器
  // search_analyzer : 搜索时使用的分词器
  {
      "mappings":{
          "properties": {
              "name": {
                  "type": "text",
                  "analyzer": "jieba_index",
                  "search_analyzer": "jieba_search"
              }
          }
      }
  }



## Java API

- elasticsearch-rest-high-level-client

  - 依赖

    ```xml
    <dependencies>
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>7.9.3</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>7.9.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.8.2</version>
        </dependency>
    </dependencies>
    ```

  - 客户端

    ```java
    package com.config;
    
    import org.apache.http.HttpHost;
    import org.elasticsearch.client.RestClient;
    import org.elasticsearch.client.RestHighLevelClient;
    
    public class ESConfig {
        public static RestHighLevelClient esRestClient() {
    
            RestHighLevelClient client = new RestHighLevelClient(
                    RestClient.builder(new HttpHost("localhost", 9200, "http")));
            return client;
        }
    }
    ```

  - index-api

    ```java
    package com.index;
    
    import com.config.ESConfig;
    import org.elasticsearch.action.admin.indices.delete.DeleteIndexRequest;
    import org.elasticsearch.action.support.master.AcknowledgedResponse;
    import org.elasticsearch.client.RequestOptions;
    import org.elasticsearch.client.RestHighLevelClient;
    import org.elasticsearch.client.indices.CreateIndexRequest;
    import org.elasticsearch.client.indices.CreateIndexResponse;
    import org.elasticsearch.client.indices.GetIndexRequest;
    import org.elasticsearch.client.indices.GetIndexResponse;
    
    public class IndexAPI {
    
        private static RestHighLevelClient client = ESConfig.esRestClient();
    
        public static void createIndex() throws Exception {
    
            CreateIndexRequest request = new CreateIndexRequest("create_index_test");
    
            CreateIndexResponse response = client.indices().create(request, RequestOptions.DEFAULT);
    
            boolean acknowledged = response.isAcknowledged();
    
            System.out.println("操作状态 = " + acknowledged);
        }
    
        public static void getIndex() throws Exception {
    
            GetIndexRequest request = new GetIndexRequest("admin");
    
            GetIndexResponse response = client.indices().get(request, RequestOptions.DEFAULT);
    
            System.out.println(response.getAliases());
            System.out.println(response.getMappings());
            System.out.println(response.getSettings());
        }
    
        public static void deleteIndex() throws Exception {
    
            DeleteIndexRequest request = new DeleteIndexRequest("create_index_test");
    
            AcknowledgedResponse response = client.indices().delete(request, RequestOptions.DEFAULT);
    
            System.out.println("操作状态 = " + response.isAcknowledged());
        }
    }
    
    ```

  - doc-api

    ```java
    package com.index;
    
    import com.config.ESConfig;
    import org.elasticsearch.action.bulk.BulkRequest;
    import org.elasticsearch.action.bulk.BulkResponse;
    import org.elasticsearch.action.delete.DeleteRequest;
    import org.elasticsearch.action.delete.DeleteResponse;
    import org.elasticsearch.action.get.GetRequest;
    import org.elasticsearch.action.get.GetResponse;
    import org.elasticsearch.action.index.IndexRequest;
    import org.elasticsearch.action.index.IndexResponse;
    import org.elasticsearch.action.update.UpdateRequest;
    import org.elasticsearch.action.update.UpdateResponse;
    import org.elasticsearch.client.RequestOptions;
    import org.elasticsearch.client.RestHighLevelClient;
    import org.elasticsearch.common.xcontent.XContentType;
    
    public class DocAPI {
    
        private static RestHighLevelClient client = ESConfig.esRestClient();
    
        public static void insertDoc() throws Exception {
    
            IndexRequest request = new IndexRequest();
    
            request.index("create_index_test").id("1001");
    
            String dataJSON = "{\"id\":1001,\"name\":\"name1001\"}";
    
            request.source(dataJSON, XContentType.JSON);
    
            IndexResponse response = client.index(request, RequestOptions.DEFAULT);
    
            System.out.println("_index:" + response.getIndex());
            System.out.println("_id:" + response.getId());
            System.out.println("_result:" + response.getResult());
        }
    
        public static void updateDoc() throws Exception {
    
            UpdateRequest request = new UpdateRequest();
    
            request.index("create_index_test").id("1001");
    
            request.doc(XContentType.JSON, "name", "name1002");
    
            UpdateResponse response = client.update(request, RequestOptions.DEFAULT);
    
            System.out.println("_index:" + response.getIndex());
            System.out.println("_id:" + response.getId());
            System.out.println("_result:" + response.getResult());
        }
    
        public static void getDoc() throws Exception {
    
            GetRequest request = new GetRequest().index("create_index_test").id("1001");
    
            GetResponse response = client.get(request, RequestOptions.DEFAULT);
    
            System.out.println("_index:" + response.getIndex());
            System.out.println("_type:" + response.getType());
            System.out.println("_id:" + response.getId());
            System.out.println("source:" + response.getSourceAsString());
        }
    
        public static void deleteDoc() throws Exception {
    
            DeleteRequest request = new DeleteRequest().index("create_index_test").id("1001");
    
            DeleteResponse response = client.delete(request, RequestOptions.DEFAULT);
    
            System.out.println(request.toString());
        }
    
        public static void batchInsert() throws Exception {
    
            BulkRequest request = new BulkRequest();
            request.add(
                    new IndexRequest().index("create_index_test").id("1001").source(XContentType.JSON, "name", "zhangsan"));
            request.add(
                    new IndexRequest().index("create_index_test").id("1002").source(XContentType.JSON, "name", "lisi"));
            request.add(
                    new IndexRequest().index("create_index_test").id("1003").source(XContentType.JSON, "name", "wangwu"));
    
            BulkResponse response = client.bulk(request, RequestOptions.DEFAULT);
    
            System.out.println("took:" + response.getTook());
            System.out.println("items:" + response.getItems());
        }
    
        public static void batchDelete() throws Exception {
    
            BulkRequest request = new BulkRequest();
    
            request.add(new DeleteRequest().index("create_index_test").id("1001"));
            request.add(new DeleteRequest().index("create_index_test").id("1002"));
            request.add(new DeleteRequest().index("create_index_test").id("1003"));
    
            BulkResponse response = client.bulk(request, RequestOptions.DEFAULT);
    
            System.out.println("took:" + response.getTook());
            System.out.println("items:" + response.getItems());
        }
    }
    ```

  - query-api

    ```java
    package com.index;
    
    import com.config.ESConfig;
    import org.elasticsearch.action.search.SearchRequest;
    import org.elasticsearch.action.search.SearchResponse;
    import org.elasticsearch.client.RequestOptions;
    import org.elasticsearch.client.RestHighLevelClient;
    import org.elasticsearch.index.query.BoolQueryBuilder;
    import org.elasticsearch.index.query.QueryBuilders;
    import org.elasticsearch.index.query.RangeQueryBuilder;
    import org.elasticsearch.search.SearchHit;
    import org.elasticsearch.search.SearchHits;
    import org.elasticsearch.search.aggregations.AggregationBuilders;
    import org.elasticsearch.search.builder.SearchSourceBuilder;
    import org.elasticsearch.search.sort.SortOrder;
    
    public class QueryAPI {
    
        private static RestHighLevelClient client = ESConfig.esRestClient();
    
        public static void page() throws Exception {
    
            SearchRequest request = new SearchRequest();
    
            request.indices("ecommerce");
    
            SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
            sourceBuilder.query(QueryBuilders.matchAllQuery());
    
            sourceBuilder.from(0);
            sourceBuilder.size(2);
    
            request.source(sourceBuilder);
    
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    
            SearchHits hits = response.getHits();
            System.out.println("took:" + response.getTook());
            System.out.println("timeout:" + response.isTimedOut());
            System.out.println("total:" + hits.getTotalHits());
            System.out.println("MaxScore:" + hits.getMaxScore());
            System.out.println("hits========>>");
    
            for (SearchHit hit : hits) {
                System.out.println(hit.getSourceAsString());
            }
        }
    
        public static void sort() throws Exception {
    
            SearchRequest request = new SearchRequest();
    
            request.indices("ecommerce");
    
            SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
            sourceBuilder.query(QueryBuilders.matchAllQuery());
    
            sourceBuilder.sort("products.base_price", SortOrder.ASC);
    
            request.source(sourceBuilder);
    
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    
            SearchHits hits = response.getHits();
            System.out.println("took:" + response.getTook());
            System.out.println("timeout:" + response.isTimedOut());
            System.out.println("total:" + hits.getTotalHits());
            System.out.println("MaxScore:" + hits.getMaxScore());
            System.out.println("hits========>>");
    
            for (SearchHit hit : hits) {
                System.out.println(hit.getSourceAsString());
            }
        }
    
        public static void bool() throws Exception {
    
            SearchRequest request = new SearchRequest();
    
            request.indices("ecommerce");
    
            SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
            sourceBuilder.query(QueryBuilders.matchAllQuery());
    
            BoolQueryBuilder boolQueryBuilder = new BoolQueryBuilder();
    
            boolQueryBuilder.must(QueryBuilders.matchQuery("currency", "EUR"));
    
            request.source(sourceBuilder);
    
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    
            SearchHits hits = response.getHits();
            System.out.println("took:" + response.getTook());
            System.out.println("timeout:" + response.isTimedOut());
            System.out.println("total:" + hits.getTotalHits());
            System.out.println("MaxScore:" + hits.getMaxScore());
            System.out.println("hits========>>");
    
            for (SearchHit hit : hits) {
                System.out.println(hit.getSourceAsString());
            }
        }
    
        public static void range() throws Exception {
    
            SearchRequest request = new SearchRequest();
    
            request.indices("ecommerce");
    
            SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
    
            RangeQueryBuilder rangeQueryBuilder = QueryBuilders.rangeQuery("products.base_price");
    
            rangeQueryBuilder.gte(10);
            rangeQueryBuilder.lte(20);
    
            sourceBuilder.query(rangeQueryBuilder);
            request.source(sourceBuilder);
    
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    
            SearchHits hits = response.getHits();
            System.out.println("took:" + response.getTook());
            System.out.println("timeout:" + response.isTimedOut());
            System.out.println("total:" + hits.getTotalHits());
            System.out.println("MaxScore:" + hits.getMaxScore());
            System.out.println("hits========>>");
    
            for (SearchHit hit : hits) {
                System.out.println(hit.getSourceAsString());
            }
        }
    
        public static void agg() throws Exception {
    
            SearchRequest request = new SearchRequest();
    
            request.indices("ecommerce");
    
            SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
            sourceBuilder.aggregation(AggregationBuilders.max("maxPrice").field("products.base_price"));
    
            request.source(sourceBuilder);
    
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    
            System.out.println(response);
            SearchHits hits = response.getHits();
            System.out.println("took:" + response.getTook());
            System.out.println("timeout:" + response.isTimedOut());
            System.out.println("total:" + hits.getTotalHits());
            System.out.println("MaxScore:" + hits.getMaxScore());
            System.out.println("aggregations :" + response.getAggregations());
            System.out.println("hits========>>");
    
            for (SearchHit hit : hits) {
                System.out.println(hit.getSourceAsString());
            }
        }
    }
    
    ```

    













