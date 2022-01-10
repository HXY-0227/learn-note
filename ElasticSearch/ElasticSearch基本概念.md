# ElasticSearch基本概念
## Document(文档)
ES是面向文档的，文档是所有可搜索数据的最小单位，一部电影、一首歌、一篇文章都是一个文档，类比到关系型数据库，相当于一行数据。

文档会被序列化成JSON格式，每个JSON格式由字段组成，每个字段有对应的字段类型。

每个文档都有一个UniqueID。可以自己指定，也可以由ES自己生成

## 文档的元数据
文档的元数据 标注文档的相关信息。
```
{
    "_index" : "movies",
    "_type" : "_doc",
    "_id" : "1178",
    "_score" : 1.0,
    "_source" : {
        "id" : "1178",
        "genre" : [
        "Drama",
        "War"
        ],
        "title" : "Paths of Glory",
        "@version" : "1",
        "year" : 1957
    }
}
```
- _index：文档所属的索引
- _type：文档所属的类型
- _id 文档的唯一ID
- _score 相关性打分
- _source 原始数据

## 索引
如果文档看成关系型数据库里的一行数据，那么索引就相当于关系型数据库的表，是一类文档的集合。以下是一个索引结构，他涉及到两个概念：mappings和settings。mappings对索引库中索引的字段名称及其数据类型进行定义，相当于关系型数据库里面的schema。settings对分片和副本数进行定义。
```
# 查看索引相关信息
GET movies
```
```
{
  "movies" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "@version" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "genre" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "id" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "year" : {
          "type" : "long"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1572545279868",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "NY-SRj1TSDCO0CJ5J0I-IQ",
        "version" : {
          "created" : "7040099"
        },
        "provided_name" : "movies"
      }
    }
  }
}
```
