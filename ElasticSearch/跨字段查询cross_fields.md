#  跨字段查询

在ES查询DSL中，我们了解了`multi_match`这种查询方式，知道它的查询方式有`best_fields`和`most_fields`两种以及基于这两种的扩展。`most_fields`致力于返回所有的相关文档，而`best_fields`则致力于返回尽可能精确的文档。他们会为每个字段都生成一个查询，而`best_fields`取子查询中算分最高的最为最终算分，`most_fields`则取所有子查询的算分的和作为最终算分，我们可以把这种方式，叫做以**字段为中心**的查询方式。具体区别参考`multi_match`中关于他俩的介绍以及`dis_max`的介绍。

今天这里我们在看这样一种场景，假设我们有一个索引，存储的是地址信息，其数据如下，我们发现城市、省份、街道等信息都散落的不同的字段，一个完整的地址需要用多个字段来唯一标识。

```json
"hits" : {
    "total" : {
      "value" : 5,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "provice" : "Gansu",
          "city" : "Lanzhou",
          "street" : "Anning"
        }
      },
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "provice" : "Gansu",
          "city" : "Lanzhou",
          "street" : "Chenguan"
        }
      },
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "provice" : "Gansu",
          "city" : "Lanzhou",
          "street" : "Qilihe"
        }
      },
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.0,
        "_source" : {
          "provice" : "Shanxi",
          "city" : "Xian",
          "street" : "Yanta"
        }
      },
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 1.0,
        "_source" : {
          "provice" : "Shanxi",
          "city" : "Anning",
          "street" : "Anning"
        }
      }
    ]
  }
```

那么对于这种场景，我们通过`most_fields`方式查询看看有什么结果？我们发现ID为1的这条数据本应该是最符合条件的，但是他的算分反而低，被排在了ID为5的数据的后面，而ID为5的数据因为`Anning`这个单词出现了两次，被排在了前面。还有一个问题是，这里我是想找`Gansu Lanzhou Anning`这个确定的地址，要么找到，要么找不到，但是他却给我们返回了一堆无用的文档，我们把这个叫做**长尾效应**。

```json
POST address/_search
{
  "query": {
    "multi_match": {
      "query": "Gansu Lanzhou Anning",
      "type": "most_fields",
      "fields": ["provice", "city", "street"]
    }
  }
}

"hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 2.261763,
    "hits" : [
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 2.261763,
        "_source" : {
          "provice" : "Shanxi",
          "city" : "Anning",
          "street" : "Anning"
        }
      },
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.9534616,
        "_source" : {
          "provice" : "Gansu",
          "city" : "Lanzhou",
          "street" : "Anning"
        }
      },
      {
          // ... 这里还有其他结果，为了空间不罗列了
      }
    ]
  }
```

因此，通过这个例子，我们就发现了这个需求用`most_fields`这种方法解决的弊端，那我们一起来找找原因。

1. ID为5的文档之所以排在前面，是因为它是以**字段为中心**的查询方式，他在查询的时候会为每个字段生成一个子查询，最终将算分加起来作为总的算分，ID为5的文档中，关键字`anning`出现了两次，因此算分最高，排列最靠前。
2. 存在长尾效应。对于这个问题，我们可能会想到通过设置`operate`为`and`或者限制`minimum_should_match`来保证尽可能的精确，但是`and`操作符要求所有的词都必须要出现在同一个字段，如果我们用了它，会导致什么也查不到。
3. 除了上述问题，还有另一个问题，**词频**的问题。在算分的时候，ES是有一个公式的，这个公式中，有两个很重的变量**TF**和**IDF**，**TF**叫做**词频**，一个文档在**某一个**文章中出现的频率越高，那么相关度就越高。但是像**的**、**呢**、**is**这种词本身出现的频率就很高，我们能说明他相关度高吗？答案是不能，那么这个时候有了另一个概念：**IDF**叫做**逆向文档频率**，如果一个词在**所有**文档中出现的频率越高，那么这个词的相关度就越低。举个例子，比如**北京**`这个地名可能经常出现，所以导致他有一个很高的IDF指，而**环县**也作为一个地名，却有着比较低的IDF值。这样最终可能导致在算分的时候出现一些意料之外的结果。

存在这些问题的根源就是因为我们将数据分散在多个字段，那么如果将他们组合成单个字段，问题自然而然就不存在了，比如下面这样的格式。但是这样会带来的问题是，数据的冗余，而且这样看起来并不优雅。好在ES为我们提供了一种解决方案：`cross_fileds 跨字段查询`。

```json
{
    "provice" : "Gansu",
	"city" : "Lanzhou",
	"street" : "Chenguan",
    "address": "Gansu Lanzhou Chenguan"
}
```

`cross_fileds`这种方式会分对查询的关键字进行分词，然后从所有字段中搜索每个词，解决了**以字段为中心**这种方式的问题，我们把它叫做**以词为中心**的方式。而且他通过混合不同字段逆向索引文档频率的方式解决了词频的问题，但是这个需要为每个字段使用相同的分析器。

```json
POST address/_search
{
  "query": {
    "multi_match": {
      "query": "Gansu Lanzhou Anning",
      "type": "cross_fields",
      "operator": "and", 
      "fields": ["provice", "city", "street"]
    }
  }
}

# 查询结果
"hits" : [
      {
        "_index" : "address",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.9534616,
        "_source" : {
          "provice" : "Gansu",
          "city" : "Lanzhou",
          "street" : "Anning"
        }
      }
    ]
  }
```

