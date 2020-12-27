 ## ElasticSearch

ES 是一个机遇 Lucene 构建的开源的,分布式的,RESTful 接口的全文搜索引擎,ES 还是一个分布式文档数据库,其中每个字段均可以被引用,而且每个字段的数据均可被搜索,ES 能够横向扩展至数以百计的服务器存储以及处理 PB 级别的数据,可以在极短的时间内存储,搜索和分析大量的数据,通常作为具有复制搜索场景下的核心发动机

## 基本概念

### Near Realtime(NRT) 几乎实时

ElasticSearch 是一个几乎实时的搜索平台,意思是,从索引一个文档到这个文档可被搜索只需要一点点延迟,这个时间一般为毫秒级

### Cluster 集群
集群是一个过多个节点的集合,这些节点共同保存整个数据,并且在所有的节点上提供连个索引和搜索的功能,集群由一个唯一的集群 ID 确定,并指定一个集群名称,这个集群的名称非常重要,因为节点是哪个可以通过这个集群名加入到集群,一个节点只能是集群的一部分

### Node 节点

节点是单个服务器的实例,它是集群的一部分,可以存储数据,并参与集群的索引和搜索功能,节点的名称默认为一个随机的通用标识符 (UUID),也可以自定义

### index 索引
索引是具有相似特性的文档的集合,例如,可以为客户数据提供索引服务,为产品建立目录至另一个索引,以及为订单数据建立另一个索引,索引由名称标识(必须全部为小写),该名称用户在对其中的文档执行搜索,索引,更新和删除的时候引用

### Type 类型

在索引中,可以定义一个或者多个类型,类型是索引的逻辑类别/分区,一般来说,类型的定义具有公共字段集的文档

### Document 文档
文档是可以被索引引用的信息的基本单位,例如,你可以为单个客户提供一个文档,单个产品提供另一个文档,以及单个订单提供另一个文档,文档的表现形式为 JSON 格式

在索引/类型中,可以尽可能多的存储文档,文档实际上必须索引或者分配到索引中的类型

### Shards & Replicas 分片与副本

索引可以存储大量的数据,这些数据可能超过单个节点的硬性限制,为了解决这一个问题,ES提供细分指标称为多个块的能力,称为分片,当你创建一个索引的时候,可以简单的定义想要的分片数量,每个分辨本身是一个全功能,独立的指数,可以托管在任何一个节点

分片的主要体现在一下两个特征:

- 分片允许水平拆分或者缩放内容的大小
- 分片允许你分配和并行操作的碎片,从而提供性能/吞吐量,这个机制的碎片是分布式的以及其文件汇总到搜索请求是完全由 ElasticSearch 管理,对于用户来说是透明的

副本的重要特性如下:

- 副本为分片或节点失败提供了高可用,一个副本的分片不会分配在同一个节点作为原始的或者主分片,副本是从主分片那里复制过来的
- 副本允许用户扩展你的搜索量或者吞吐量,因为搜索可以在所有副本上并行执行

#### ES基本概念与关系型数据库的比较

| ES概念                                         | 关系型数据库       |
| ---------------------------------------------- | ------------------ |
| Index（索引）支持全文检索                      | Database（数据库） |
| Type（类型）                                   | Table（表）        |
| Document（文档），不同文档可以有不同的字段集合 | Row（数据行）      |
| Field（字段）                                  | Column（数据列）   |
| Mapping（映射）                                | Schema（模式）     |

## ES API

以下示例使用`curl`演示。

### 查看健康状态

```bash
curl -X GET 127.0.0.1:9200/_cat/health?v
```

输出：

```bash
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1564726309 06:11:49  elasticsearch yellow          1         1      3   3    0    0        1             0                  -                 75.0%
```

### 查询当前es集群中所有的indices

```bash
curl -X GET 127.0.0.1:9200/_cat/indices?v
```

输出：

```bash
health status index                uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_task_manager LUo-IxjDQdWeAbR-SYuYvQ   1   0          2            0     45.5kb         45.5kb
green  open   .kibana_1            PLvyZV1bRDWex05xkOrNNg   1   0          4            1     23.9kb         23.9kb
yellow open   user                 o42mIpDeSgSWZ6eARWUfKw   1   1          0            0       283b           283b
```

### 创建索引

```bash
curl -X PUT 127.0.0.1:9200/www
```

输出：

```bash
{"acknowledged":true,"shards_acknowledged":true,"index":"www"}
```

### 删除索引

```bash
curl -X DELETE 127.0.0.1:9200/www
```

输出：

```bash
{"acknowledged":true}
```

### 插入记录

```bash
curl -H "ContentType:application/json" -X POST 127.0.0.1:9200/user/person -d '
{
	"name": "dsb",
	"age": 9000,
	"married": true
}'
```

输出：

```bash
{
    "_index": "user",
    "_type": "person",
    "_id": "MLcwUWwBvEa8j5UrLZj4",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 3,
    "_primary_term": 1
}
```

也可以使用PUT方法，但是需要传入id

```bash
curl -H "ContentType:application/json" -X PUT 127.0.0.1:9200/user/person/4 -d '
{
	"name": "sb",
	"age": 9,
	"married": false
}'
```

### 检索

Elasticsearch的检索语法比较特别，使用GET方法携带JSON格式的查询条件。

全检索：

```bash
curl -X GET 127.0.0.1:9200/user/person/_search
```

按条件检索：

```bash
curl -H "ContentType:application/json" -X PUT 127.0.0.1:9200/user/person/4 -d '
{
	"query":{
		"match": {"name": "sb"}
	}	
}'
```

ElasticSearch默认一次最多返回10条结果，可以像下面的示例通过size字段来设置返回结果的数目。

```bash
curl -H "ContentType:application/json" -X PUT 127.0.0.1:9200/user/person/4 -d '
{
	"query":{
		"match": {"name": "sb"},
		"size": 2
	}	
}'
```

## Go操作Elasticsearch

### elastic client

我们使用第三方库[https://github.com/olivere/elastic](https://github.com/olivere/elastic)来连接ES并进行操作。

注意下载与你的ES相同版本的client，例如我们这里使用的ES是7.2.1的版本，那么我们下载的client也要与之对应为`github.com/olivere/elastic/v7`。

使用`go.mod`来管理依赖：

```go
require (
    github.com/olivere/elastic/v7 v7.0.4
)
```

简单示例：

```go
package main

import (
	"context"
	"fmt"

	"github.com/olivere/elastic/v7"
)

// Elasticsearch demo

type Person struct {
	Name    string `json:"name"`
	Age     int    `json:"age"`
	Married bool   `json:"married"`
}

func main() {
    // 初始化一个客户端
	client, err := elastic.NewClient(elastic.SetURL("http://192.168.1.7:9200"))
	if err != nil {
		// Handle error
		panic(err)
	}

	fmt.Println("connect to es success")
	p1 := Person{Name: "rion", Age: 22, Married: false}
  // 链式操作,index 相当于数据库,type 相当于表,BodyJson相当于一行数据
	put1, err := client.Index().
		Index("user").
        Type("go").
		BodyJson(p1).
		Do(context.Background())
	if err != nil {
		// Handle error
		panic(err)
	}
	fmt.Printf("Indexed user %s to index %s, type %s\n", put1.Id, put1.Index, put1.Type)
}
```

更多使用详见文档：[https://godoc.org/github.com/olivere/elastic](https://godoc.org/github.com/olivere/elastic)