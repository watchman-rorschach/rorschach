9200用于外部通讯，基于http协议，程序与es的通信使用9200端口。
9300jar之间就是通过tcp协议通信，遵循tcp协议，es集群中的节点之间也通过9300端口进行通信。
# 概述
## 概述
ELK=ES +logstash+kibana
在客户端部署轻量级的filebeat用户收集日志到kafka。kafka将日志输出到logstash，再输出到ES，filebeat再客户端负责收集日志，kafaka可以防止数据丢失。
#### 字段（Fields）
字段是ES中最小的独立单元数据，每一个字段有自己的数据类型（可以自己定义覆盖ES自动设置的数据类型），我们还可以对单个字段设置是否分析、分词器等等。
核心的数据类型有string、Numeric、DateDate、Boolean、Binary、Range等等，复杂类型有Object、Nested，详细的可以[参考官方的介绍](https://link.juejin.cn?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2Fcurrent%2Fmapping-types.html)
#### 文档（Documents）
在ES中文档的概念相当于RDBMS中的一行数据，不同的是在ES中文档的存储是直接使用json格式存储的（也就是可以嵌套），而不是像RDBMS中把数据"压平"了存储，这一点也是Nosql和关系型数据库比较大的区别。
#### 映射（Mapping）
Mapping是定义文档和字段如何存储和索引，使用Mapping可以定义下面这些信息

- 哪些字段应该作为全文索引
- 哪些字段包括numbers, dates, geolocations.
- 时间类型的格式
- 定义规则控制动态增加字段的mapping
#### 索引（Index）
在ES中是最大的数据存储概念，**它是由许多具有相同特征的文档组成的一个集合**。 由于在ES7.0之后逐渐废除Type类型，所以Index从”**数据库**“的概念变成了_实际上_的”**表**“概念，我们可以把它近似地当成RDBMS中的表，但是要注意Index只是一个逻辑上的概念，真实的数据是分开存储在各个分片中的。
#### 分片（Shards）
首先，每个分片都是一个Lucene索引实例，我们可以将其视作一个独立的搜索引擎，它能够对Elasticsearch集群中的数据子集进行索引并处理相关查询。
分片分为两种：**主分片（Primary Shard）、副本分片（Replica Shard）**

- 主分片：由于所有的数据都会在主分片中存储，所以**主分片决定了文档存储数量的上限**，但是一个索引的主分片数在创建时一旦指定，那么主分片的数量就不能改变，这是因为当索引一个文档时，它是通过其主键（默认）Hash到对应的主分片上（类似于RDBMS分库分表路由策略），所以我们一旦修改主分片数量，那么将无法定位到具体的主分片上。在mapping时我们可以设置number_of_shards值，最大值默认为1024。
- 副本分片：我们可以为一个主分片根据实际的硬件资源指定任意数量的副本分片，上边已经说过，每个分片都可以处理查询，所以我们可以**增加副本分片的资源（相应硬件资源提升）来提升系统的处理能力**。同样，在mapping时，可以通过number_of_replicas参数来设置每个主分片的副本分片数量

但是要注意，为了容错（节点主机宕机导致数据丢失），**主分片和副本分片不能在同一个节点上，防止节点宕机导致部分数据丢失**。

## ES的优缺点
### 优点

- 高可用
- 天然分布式
- 支持插件
### 缺点

- 不支持事务
- 没有细致的用户权限框架
- 最快也要1秒后才能查询到插入后的数据。【或者手动refresh数据后可以立即查询到】
- 无法对已存在的索引修改字段

# 安装
## 安装【docker方式】
```bash
docker run -p 9200:9200 -p 9300:9300 --name es7.10.1  -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms512m -Xmx512m" -d elasticsearch:7.10.1
```
## 安装【docker-compose方式】

```yaml
services:
elasticsearch:
image: elasticsearch:7.13.0
container_name: elasticsearch
ports:
- 9200:9200
- 9300:9300
restart: always
environment:
- "discovery.type=single-node"
- "ES_JAVA_OPTS=-Xms4g -Xmx4g" #设置使用jvm内存大小
- "xpack.security.enabled=true"
- "xpack.security.transport.ssl.enabled=true"
- "cluster.max_shards_per_node=2000"
volumes:
- /data/es/data:/usr/share/elasticsearch/data
- /data/es/logs:/usr/share/elasticsearch/logs

kibana:
image: kibana:7.13.0
container_name: kibana
ports:
- 5601:5601
restart: always
links:
#可以用es这个域名访问elasticsearch服务
- elasticsearch:es 
depends_on:
- elasticsearch 
environment:
#设置访问elasticsearch的地址
      - elasticsearch.hosts=http://elasticsearch:9200
```
# 分词器
## 安装分词器
```bash
首先进入容器
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.10.1/elasticsearch-analysis-ik-7.10.1.zip

```
## yum安装分词器
```bash
/usr/share/elasticsearch/bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.0/elasticsearch-analysis-ik-7.17.0.zip
```
# IK分词器
Elasticsearch中文分词我们采用Ik分词，ik有两种分词模式，ik_max_word,和ik_smart模式;

- **ik_max_word**: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合；
- **ik_smart**: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”。
## springboot集成ES

# 八股文
## 节点角色
### master
其实这个是master准确的来说是具有成为master节点资格的节点，即master-eligible node，具体哪个节点会成为master node则由master选举算法选举出来的，其中6版本的es使用的是Bully算法进行选举；7版本的es使用的定制版本的Raft算法进行选举，
主节点负责集群范围内的元数据（即Cluster State）相关的操作，例如创建或删除索引，跟踪哪些节点是集群的一部分以及确定将哪些 shard 分配给哪些节点。 拥有稳定的主节点对于群集健康非常重要。 对于大规模的elasticsearch集群配置专用的master节点的必要且有效的，能够提升系统的稳定性。
### data
数据节点包含包含已建立索引的文档的分片。 
数据节点处理与数据相关的操作，例如 CRUD，搜索和聚合。
在通用部署的场景下，data节点仅有一个角色data，所有的数据存储和操作都在这个统一的角色下进行。另外为了保证系统的稳定性以及高可用性，配置单独的data节点是必须的，尤其对于大型的elasticsearch集群而言，专用的data节点可以有效的降低节点压力，提升数据处理的稳定性和实时性
在高版本的elasticsearch中，elasticsearch的data节点可以采用多层部署。在这种部署模式下，data节点被细分成data_content，data_hot，data_warm 或 data_cold四类角色，用来存储和处理不同的数据，
### ingest
摄取节点可以执行由一个或多个ingest processor组成的预处理pipeline。用于对写入或者查询的数据进行预处理，即可以将部分client需要预处理的工作放到了server端，如常见的格式转换，空值处理以及时间处理。
### Remote-eligible node
默认情况下，集群中的任何节点都可以充当跨集群客户端并连接到其他集群。 连接后，你可以使用跨集群搜索来搜索远程集群。 你还可以使用跨集群复制在集群之间同步数据。这些操作实现的基础就是远程连接节点。
### Coordinating only node（协调节点）
如果一个节点不担任 master 节点的职责，不保存数据，也不预处理文档，那么这个节点将拥有一个仅可路由请求，处理搜索缩减阶段并分配批量索引的协调节点。 本质上，仅协调节点可充当智能负载平衡器。默认elasticsearch的所有节点都可以作为协调节点去路由和分发请求。
### Machine learning node（X-pack专用角色）
机器学习节点提供了机器学习功能，该节点运行作业并处理机器学习 API 请求。
### Transform node（X-pack专用角色）
转换节点运行转换并处理转换 API 请求。
## master选举过程
### 选举出发机制
当一个节点发现包括自己在内的多数派的master-eligible「候选主节点」认为集群没有master时，就可以发起master选举。
前置前提：
（1）只有候选主节点（master：true）的节点才能成为主节点。
（2）最小主节点数（min_master_nodes）的目的是防止脑裂。
核对了一下代码，核心入口为 findMaster。
### 选举流程：
第一步：确认候选主节点数达标，elasticsearch.yml 设置的值  discovery.zen.minimum_master_nodes；
第二步：比较：先判定是否具备 master 资格【是否是master-eligible节点】，具备候选主节点资格的优先返回；
若两节点都为候选主节点，选择clusterStateVersion最大的节点作为master，若clusterStateVersion相同则 id 小的值会主节点。id是节点第一次启动的时候生成，注意这里的 id 为 string 类型。
## 新节点加入集群的流程。
1、设置一个新的es实例；
2、在elasticsearch.yml中指定相同的cluster.name属性，；
3、启动es，会自动发现节点并加入集群；

## 写入数据流程

1. 客户端选择任意一个节点发送请求，则该节点成为协调节点（coordinating node）。
2. 协调节点对docmount进行路由，将请求转发给到对应的primary shard。
3. primary shard 处理请求，将数据同步到所有的replica shard。
4. 此时协调节点，发现primary shard 和所有的replica shard都处理完之后，就反馈给客户端。
### 写数据的底层原理

1. 在到达primary shard的时候 ，数据先写入内存buffer ， 此时，在buffer里的数据是不会被搜索到的，然后生成一个translog日志文件 ， 将数据写入translog里。
2. 而且es里每隔1s就会将buffer里的数据写入到一个新的segment file中，如果内存buffer空间快满了，就会将数据refresh到一个新的segment file文件中，这个segment file就存储最最近1s中buffer写入的数据，如果buffer里面没有数据，就不会执行refresh操作，当建立segment file文件的时候，就同时建立好了倒排索引库。
3. 在buffer refresh到segment之前 ，会先进入到一个叫os cache中，只要被执行了refresh操作，就代表这个数据可以被搜索到了。数据被输入os cache中，buffer就会被清空了。默认是每隔1秒refresh一次的，所以es是准实时的，因为写入的数据1秒之后才能被看到。还可以通过es的restful api或者java api，手动执行一次refresh操作，就是手动将buffer中的数据刷入os cache中，让数据立马就可以被搜索到。
4. 就这样新的数据不断进入buffer和translog，translog会变得越来越大。当translog达到一定长度的时候，就会触发commit操作。translog也是先进入os cache中，然后每隔5s持久化到translog到磁盘中。
5. es也有可能会数据丢失 ，有5s的数据停留在buffer、translog os cache, segment file os cache中，有5s的数据不在磁盘上，如果此时宕机，这5s的数据就会丢失，如果项目要求比较高，不能丢失数据，就可以设置参数，每次写入一条数据写入buffer，同时写入translog磁盘文件中，但这样做会使es的性能降低。
6. 如果是删除操作，commit操作的时候就会生成一个.del文件，将这个document标识为deleted状态，在搜索的搜索的时候就不会被搜索到了。
7. 如果是更新操作，就是将原来的document标识为deleted状态，然后新写入一条数据
## 读取数据的过程

1. 客户端发送get请求到任意一个node节点，然后这个节点就称为协调节点，
2. 协调节点对document进行路由，将请求转发到对应的node，此时会使用随机轮询算法，在primary shard 和replica shard中随机选择一个，让读取请求负载均衡，
3. 接收请求的node返回document给协调节点，
4. 协调节点，返回document给到客户端
## 搜索过程

1. 客户端发送请求到协调节点，
2. 协调节点将请求发送到所有的shard对应的primary shard或replica shard ；
3. 每个shard将自己搜索到的结果返回给协调节点，返回的结果是dou.id或者自己自定义id，然后协调节点对数据进行合并排序操作，最终得到结果。
4. 最后协调节点根据id到个shard上拉取实际 的document数据，最后返回给客户端。
# 基本操作
## 节点操作
### 查看服务状态与信息

```powershell
# 查看所有节点
GET _cat/nodes
# 查看健康状态
GET _cat/health
# 查看主节点
GET _cat/master
```
## 索引操作
### 索引的增删改查
```powershell
# 查看所有索引，?v 的作用是让返回的数据格式更方便查看
GET _cat/indices?v
# 查看名为 product 的索引
GET _cat/indices/product?v
或
GET product
# 删除名为 product 的索引
DELETE product

# 创建一个名为 product 的索引
PUT product
# 创建索引，并指定字段类型，type=text才能进行全文检索
PUT https://es-9hc62cdw.public.tencentelasticsearch.com:9200/zgt1
Content-Type：application/json
{
    "mappings": {
        "properties": {
            "id": {
                "type": "keyword"
            },
            "path": {
                "type": "keyword"
            },
            "protocol": {
                "type": "keyword"
            },
            "title": {
                "type": "text"
            },
            "to": {
                "type": "keyword"
            }
        }
    }
}


```
## 文档操作
### 增删改文档
```powershell
# 批量导入数据
POST localhost:9200/how2java/product/_bulk?refresh
Content-Type：application/json
--data-binary "@products.json"
product.json文件中的格式如下。
「
{"index":{"_index":"how2java","_type":"product","_id":10001}}
{"code":"540785126782","price":398,"name":"房屋卫士自流平美缝剂瓷砖地砖专用双组份真瓷胶防水填缝剂镏金色","place":"上海","category":"品质建材"}
{"index":{"_index":"how2java","_type":"product","_id":10002}}
{"code":"24727352473","price":21.799999237060547,"name":"艾瑞泽手工大号小号调温热熔胶枪玻璃胶枪硅胶条热溶胶棒20W-100W","place":"山东青岛","category":"品质建材"}
{"index":{"_index":"how2java","_type":"product","_id":10003}}
{"code":"40972207020","price":2198,"name":"HIGOLD/悍高 水槽双槽 厨房洗菜盆304不锈钢加厚拉丝手工水槽套餐","place":"广东佛山","category":"品质建材"}
」


# 不指定 id 的创建方式，自动生成一个随机 id
POST product/_doc
{
  "title": "小米手机",
  "category": "小米",
  "images": "http://www.gulixueyuan.com/xm.jpg",
  "price": 3999
}
# 指定 id，如果 id 存在则会修改这个数据，版本号加 1
POST product/_doc/1
{
  "title": "小米手机",
  "category": "小米",
  "images": "http://www.gulixueyuan.com/xm.jpg",
  "price": 3999
}
# 只修改指定的字段
POST product/_update/1
{
  "doc": {
    "price": 2000  
  }
}
# PUT 请求也可以新增或修改数据，但 PUT 必须指定 id，所以一般使用 PUT 修改数据
PUT product/_doc/1
{
  "title": "小米手机 new",
  "category": "小米",
  "images": "http://www.gulixueyuan.com/xm.jpg",
  "price": 3999
}
# 根据 id 删除文档
DELETE product/_doc/1
# 根据条件删除文档（删除 price 为 2000 的文档）
POST product/_delete_by_query
{
  "query": {
    "match": {
      "price": 2000
    }
  }
}
```

### 根据 id 查看文档数据

```powershell
# 根据主键获取文档数据
GET product/_doc/1
```
## Query DSL 语法

### 查看分词效果
```bash
GET _analyze
{
  "text": "开发者可调用接口完成模板选用和消息下发"
}


```
### query查询基本语法
# 检索数据
# query 定义如何查询
# match_all 查询类型（代表查询所有的所有），es 中可以在 query 中组合非常多的查询类型完成复杂查询
# 除了 query 参数之外，我们也可以传递其它的参数以改变查询结果。如 sort、size，可以用 from + size 实现分页

```powershell
# sort 排序，多字段排序，会在前序字段相等时后续字段内部排序，否则以前序为准
GET bank/_search
{
  "query": {
    "match_all": {}
  }
}
```
### 返回部分字段

```powershell
# _source 定义返回部分字段
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "_source": [
    "balance",
    "firstname"
  ]
}
```

### match 匹配查询

```powershell
## match 匹配查询
GET bank/_search
{
  "query": {
    "match": {
      "account_number": 20
    }
  },
  "_source": [
    "balance",
    "firstname",
    "account_number"
  ]
}
# match 字符串，全文检索，查询出 address 包含 mill 的记录
GET bank/_search
{
  "query": {
    "match": {
      "address": "mill"
    }
  },
  "_source": [
    "address",
    "firstname",
    "account_number"
  ]
}
# 字符串，多个单词(分词+全文检索)，查询出 address 包含 mill 和 road 的记录，并给出相关性得分
GET bank/_search
{
  "query": {
    "match": {
      "address": "mill road"
    }
  },
  "_source": [
    "address",
    "firstname",
    "account_number"
  ]
}
```

### match_phrase 短语匹配，不分词

```powershell
# 短语匹配，不分词
GET bank/_search
{
  "query": {
    "match_phrase": {
      "address": "mill road"
    }
  },
  "_source": [
    "address",
    "firstname",
    "account_number"
  ]
}
```

### multi_match 多字段匹配

```powershell
# multi_match 多字段匹配，查询 state 或者 address 包含 mill 的记录
GET bank/_search
{
  "query": {
    "multi_match": {
      "query": "mill",
      "fields": [
        "state",
        "address"
      ]
    }
  },
  "_source": [
    "address",
    "firstname",
    "state"
  ]
}
```

### bool 复合查询
```powershell
# bool 复合查询
# 复合语句可以合并任何其它查询语句，包括复合语句，了解这一点是很重要的
# 这就意味着复合语句之间可以互相嵌套，可以表达非常复杂的逻辑
# must：必须达到must列举的所有条件
# should：应该达到should列举的条件，如果达到会增加相关文档的评分，并不会改变 查询的结果。
# 如果 query 中只有 should 且只有一种匹配规则，那么 should 的条件就会被作为默认匹配条而改变查询结果
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        },
        {
          "match": {
            "gender": "M"
          }
        }
      ],
      "should": [
        {
          "match": {
            "address": "lane"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "email": "baluba.com"
          }
        }
      ]
    }
  }
}
```

### filter 结果过滤

///
```powershell
# 并不是所有的查询都需要产生分数，特别是那些仅用于 “filtering”(过滤)的文档
# 为了不计算分数 Elasticsearch 会自动检查场景并且优化查询的执行
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        }
      ],
      "filter": {
        "range": {
          "balance": {
            "gte": 10000,
            "lte": 20000
          }
        }
      }
    }
  },
  "_source": [
    "balance",
    "address"
  ]
}
```
### term

//////////
```powershell
### term 和 match 一样。匹配某个属性的值。全文检索字段用 match，其他非 text 字段匹配用 term，这是一个规范
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "age": {
              "value": "28"
            }
          }
        }
      ]
    }
  }
}
```
### aggregations 聚合

//////
```powershell
### 搜索 address 中包含 mill 的所有人的年龄分布以及平均年龄，但不显示这些人的详情
GET bank/_search
{
  "query": {
    "match": {
      "address": "mill"
    }
  },
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "age"
      }
    },
    "avg_age": {
      "avg": {
        "field": "age"
      }
    }
  },
  "size": 0
}
# 按照年龄聚合，并且请求这些年龄段的这些人的平均薪资
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 1000
      },
      "aggs": {
        "balanceAvg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
# 查出所有年龄分布，并且这些年龄段中 M 的平均薪资和 F 的平均薪资以及这个年龄段的总体平均薪资
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age"
      },
      "aggs": {
        "genderAgg": {
          "terms": {
            "field": "gender.keyword",
            "size": 5
          },
          "aggs": {
            "balanveAvg": {
              "avg": {
                "field": "balance"
              }
            }
          }
        },
        "ageBalanceAvg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```



## Mapping 基本操作
#### 查看索引的 mapping 信息

```powershell
# 查看 product 索引的 mapping 信息
GET product/_mapping
```
#### 创建 mapping

/
```powershell
PUT product
{
  "mappings": {
    "properties": {
      "age": {
        "type": "integer"
      },
      "email": {
        "type": "keyword"
      },
      "name": {
        "type": "text"
      }
    }
  }
}
```
#### 添加新的字段 mapping

/
```powershell
# 添加新的字段 mapping
PUT product2/_mapping
{
  "properties": {
    "price": {
      "type": "integer"
    }
  }
}
```
#### 更新 mapping
对于已经存在的 mapping，不能对它进行更新，只能创建新的索引再进行数据迁移。

# 最佳实践
## 数据迁移

- 创建新索引，命名为 product2
```powershell
# 创建新索引，修改字段类型
PUT product2
{
  "mappings": {
    "properties": {
      "category": {
        "type": "text"
      },
      "images": {
        "type": "text"
      },
      "price": {
        "type": "half_float"
      },
      "title": {
        "type": "text"
      }
    }
  }
}
```

- 进行数据迁移，将 product 的数据迁移到 product2
```powershell
POST _reindex
{
  "source": {
    "index":"product"
  },
  "dest": {
    "index": "product2"
  }
}
```
[ES 数据迁移策略（ES 系列二）](https://www.yuque.com/shiwangzhou/simpread/1629970656344?view=doc_embed)
## 批量数据导入
[https://how2j.cn/k/search-engine/search-engine-curl-batch/1704.html](https://how2j.cn/k/search-engine/search-engine-curl-batch/1704.html)
### 使用curl读取数据导入
```bash
curl -H "Content-Type: application/json" -XPOST "localhost:9200/how2java/product/_bulk?refresh" --data-binary "@products.json"


product.json数据如下：
{"index":{"_index":"how2java","_type":"product","_id":10001}}
{"code":"540785126782","price":398,"name":"房屋卫士自流平美缝剂瓷砖地砖专用双组份真瓷胶防水填缝剂镏金色","place":"上海","category":"品质建材"}
{"index":{"_index":"how2java","_type":"product","_id":10002}}
{"code":"24727352473","price":21.799999237060547,"name":"艾瑞泽手工大号小号调温热熔胶枪玻璃胶枪硅胶条热溶胶棒20W-100W","place":"山东青岛","category":"品质建材"}
 
```

## 查询优化
背景知识：

1. 往 es 里写的数据，实际上都写到磁盘文件里去了，查询的时候，操作系统会将磁盘文件里的数据自动缓存到 filesystem cache 【内存】里面去。



- 让 es 性能要好，最佳的情况下，就是你的机器的内存，至少可以容纳你的总数据量的一半。
- 数据预热
- 冷热分离



