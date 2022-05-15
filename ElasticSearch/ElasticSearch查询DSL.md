# Query DSL

除了通过URI这种方式进行查询外，ES还支持通过**DSL(Domain Specific Language 领域专用语言)**的方式进行查询，这种查询方式比URI更加灵活，功能也更加丰富。

## Term级别查询

### ids（[Ids Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-ids-query.html)）

ids是相对来说比较简单的一种dsl，类似于mysql的`where id in ()`的语义，他支持value属性，可以传入一个数组，里面填上你想要查询的ID的文档即可

```json
GET /_search
{
  "query": {
    "ids" : {
      "values" : ["1", "4", "100"]  # 返回属性_id为1、4、100的文档，如果存在的话
    }
  }
}
```

### exists（[Exists Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-exists-query.html)）

exists是用来匹配文档的mapping中是否包含某一个字段，如果是就返回。比如我有下面两个文档：ID为4的文档有两个字段：title和body，ID为5的文档还有一个字段content。那么如果我们想要查询包含content字段的文档，就可以用exists去做。

```json
POST baike/_doc/4
{
  "title": "Quick brown rabbits",
  "body": "Brown rabbits are commonly seen."
}

POST baike/_doc/5
{
  "title": "Keeping pets healthy",
  "body": "My quick brown fox eats rabbits on a regular basis.",
  "content": "test exists"
}
```

```json
# 这条dsl只会返回ID为5的文档
POST /baike/_search
{
  "query": {
    "exists": {"field": "content"}
  }
}
```

还有一天需要注意的是，如果content的value值为`null`或者`[]`，那么是不会被匹配到的。

### prefix （[Prefix Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html)）

prefix查询很好理解，就是将查询关键字作为一个前缀进行匹配，比如下面的例子，如果title是以**Ela**作为前缀的都会返回，这里是区分大小写，如果要设置不区分大小写，可以将`case_insensitive`设置为true。

```json
POST /baike/_search
{
  "query": {
    "prefix": {
      "title.keyword": {
        "value": "Ela"  # title是以Ela作为前缀的都可以匹配到，注意ela并不会被匹配到
      }
    }
  }
}

"hits" : {
    "total" : {
        "value" : 1,
        "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 1.0,
            "_source" : {
                "title" : "ElasticSearch",
                "body" : "the learn note about es"
            }
        }
    ]
}
```

上面的查询可以简写为下面这样：

```json
POST /baike/_search
{
  "query": {
    "prefix": { "title.keyword":"Ela"}
  }
}
```

### range（[Range Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html)）

range查询，更好理解了，就是范围查询嘛，他的语法可以参考下面的代码。这里呢出现了`gte`、`lte`是什么意思呢，其实就是ES中对于范围的描述，这里给大家总结一下：

- `gt`：大于
- `gte`：大于等于
- `lt`：小于
- `lte`：小于等于

```json
# 查询在10到20之间的
GET /_search
{
  "query": {
    "range": {
      "age": {
        "gte": 10,
        "lte": 20,
        "boost": 2.0
      }
    }
  }
}
```

除了上面说到的表示范围的参数之外，range查询还支持下面的一些参数：

- `format`：string类型，支持对查询的时间进行格式化，如果我们不提供会用默认的格式：`"yyyy-MM-dd"`
- `time_zone`：用来指定时区，可以是UTC时区的偏移量，比如`+01:00`，或者IANA时区，比如`America/Los_Angeles`
- `relation`：用来表示我们指定的范围和文档中数据的关系。一共有三种枚举值：
  - `INTERSECTS (Default)`：匹配的文档中的范围字段的值和我们的指定的查询范围是一个交集的关系
  - `CONTAINS`：匹配的文档中的范围字段的值完全包含了我们指定的查询范围
  - `WITHIN`：匹配的文档中的范围字段的值在我们的查询范围之中

range查询也可以用在时间上，并且还可以做时间的计算，在es中，用y表示年，M表示月，d表示天，h表示小时。。。因此可以用+1d、-2h这种方法来进行时间的计算，可以通过`/d`表示四舍五入到最近的一天。也可以支持now取当前时间。更多关于时间计算的可以参考：[Date Math](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math)

```json
GET /_search
{
  "query": {
    "range": {
      "timestamp": {
        "time_zone": "+01:00", 
        "gte": "2020-01-01T00:00:00", 
        "lte": "now"
      }
    }
  }
}
```

### wildcard（[Wildcard Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html)）

**waildcard**中文是通配符的意思，通配符是匹配一个或者多个字符的占位符，我们平时也总会遇到。es也为我们提供了这种查询方式。例如下面的例子，只要文档中的user.id以**ki**开始，以**y**结尾，那么都能匹配到。这里的*****表示可以匹配多个字符。wildcard查询支持下面几个参数：

- `value`：查询条件，就是我们指定的通配符，目前支持两种：`*`表示可以匹配0到多个字符，`?`表示匹配任何单个字符。注意尽量不要用`*`或者`?`开头，因为这样可能匹配到大量的数据，导致性能下降严重。

- `boost`：用来减少或增加查询相关性算分的参数。
- `case_insensitive`：默认值false，如果设置为true，表示不区分大小写。
- `rewrite`：可以重写查询方法，目前还没实践到，关于更多说明可以参考：[rewrite parameter](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-term-rewrite.html)

```json
GET /_search
{
  "query": {
    "wildcard": {
      "user.id": {
        "value": "ki*y",
        "boost": 1.0,
        "rewrite": "constant_score"
      }
    }
  }
}
```

### regexp（[Regexp Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html)）

上面提到的通配符查询，功能还是太有限了，如果能支持正则表达式岂不是堪称完美？说曹操曹操到，正则表达式查询，ES也为我们提供了。例如下面的例子，会匹配到user.id是以**k**开头，**y**结尾的单词。中间的`.*`表示匹配任意长度的任意字符像ky、kay、kimchy等都可以被匹配到。

在regexp中，有这么几个参数，需要关注一下，除过value，其他都是可选参数：

- `value`：查询条件，指定我们输入的正则表达式，关于更多的正则表达式语句可以参考：[Regular expression syntax](https://www.elastic.co/guide/en/elasticsearch/reference/current/regexp-syntax.html)
- `flags`：可以通过这个参数为正则表达式提供一些更丰富的选项
  - `ALL`：默认值，允许所有的操作符
  - `COMPLEMENT`：允许操作符`~`，用来实现一个求反的效果，例如：`a~bc`可以匹配到`adc`，`aec`，但是不会匹配到`abc`。
  - `INTERVAL`：允许操作符`<>`，用来匹配数字范围，例如：`foo<01-100>`可以匹配到`foo01`一直到`foo100`
  - `INTERSECTION`：允许操作符`&`，用来and的意思，就是说`&`符号两遍的表达式都满足才能被匹配到，例如`aaa.+&.+bbb`可以匹配到`aaabbb`
  - `ANYSTRING`：允许操作符`@`，用来匹配任何一个完整的字符串。例如`@&~(abc.+)`匹配任何字符串，除过以abc开头的
- `case_insensitive`：默认值false，如果设置为true，表示正则表达式不区分大小写。
- `max_determinized_states`：据官网介绍，es底层对正则表达式的解析是通过**lucene**来实现的，那么在lucene解析正则表达式的过程中，会将每个正则表达式转换为包含多个确定状态的状态机，默认值是**10000**，我们也可以通过这个参数去修改。而且据官网说这个生成状态机的过程是一个很耗时的过程，为了防止资源耗尽，我们应该控制这个参数。如果一个正则表达式特别复杂的话，也可以适当的去把这个参数往大调整。这块目前没有进行测试和深入研究，后续有机会补充，这里有一篇es中文社区的博客，对这个参数做了一个较为详细的说明：[ElasticSearch集群故障案例分析: 警惕通配符查询](https://elasticsearch.cn/article/171)
- `rewrite`：参考上文

```json
GET /_search
{
  "query": {
    "regexp": {
      "user.id": {
        "value": "k.*y",
        "flags": "ALL",
        "case_insensitive": true,
        "max_determinized_states": 10000,
        "rewrite": "constant_score"
      }
    }
  }
}
```

### term（[Term Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html)）

**term**汉语翻译是术语、条款等意思，大概我觉得可以理解为一个最小单位的概念，他在ES中用来做精确查询。但是有一点需要注意的是，在ES中保存的文档，都会对单词进行分词处理，然后建立倒排索引，比如我们存了一个`HelloWorld`，但是很有可能会被分词成`hello`和`world`两个单词，这个时候如果你用Term查询，通过`HelloWorld`是查不到的，因为Term是不做分词处理的。这个时候可以通过match或者keyword来代替。

```json
# 通过term这种查询方式在blogs这个索引中查找title为Quick disjunction的文档，是找不到的，因为Quick disjunction已经做了分词处理
POST blogs/_search
{
  "query": {
    "term": {
      "title": {  # 查找的字段
        "value": "Quick disjunction"  # 查找的数据
      }
    }
  }
}

# 将上面的语句稍作改动即可
POST blogs/_search
{
  "query": {
    "term": {
      "title.keyword": { # 每个字段都有一个keyword这个属性，是没有分词之前元数据
        "value": "Quick disjunction"
      }
    }
  }
}
```

通过上面的方式去做查询，返回的结果会给每一个文档做一个**相关性算分**，用`_score`来表示，如果一个查询匹配到多条数据，那么`_score`最高的会排在最前面，表示匹配度最高。这个算分的过程其实是比较消耗性能的，如果我们不关注这个属性的话，可以通过**Filter**的方式绕过算分这个环节，避免一些开销，并且Filter还可以利用缓存，提升查询效率。

```json
POST blogs/_search
{
  "query": {
    "constant_score": {  # constant：常数，表示跳过算分，返回的_score是一个常数
      "filter": {
        "term": {
          "title.keyword": {
            "value": "Quick disjunction"
          }
        }
      }
    }
  }
}
```

但是可能大多数情况下，我们还是比较关注score这个分数的，甚至有可能希望为某一次特殊的查询去调整这个分数，比如百度竞价排名的广告。。那么这个时候可以通过另一个参数`boost (提高，使增长)`这个参数去实现。他是一个浮点类型的数字，默认是1.0，如果我们想去降低分数，只需要把他控制在0.0-1.0之间，越小分数越低，如果我们想提高分数，只需要把他从1.0开始往大调整，越大算出来的分数越高。

```json
POST blogs/_search
{
  "query": {
    "term": {
      "title.keyword": {
        "value": "Quick disjunction",
        "boost": 2
      }
    }
  }
}
```

### terms（[Terms Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html)）

term查询是单字符串单字段的查询方式，terms就是多字符串单字段的查询，下面是一个例子：

```json
POST /baike/_search
{
  "query": {
    "terms": {
      "title.keyword": [   # 这里是一个数组，可以写多个关键字作为查询条件，只要title符合其中一个就可以匹配到
        "ElasticSearch",
        "Keeping pets healthy"
      ]
    }
  }
}
```

terms除了上述特性外，还有一个特别有用的功能，那就是关联索引查询，官网叫做**Terms lookup**，可以实现类似数据库的join查询。这个可以用在什么地方呢？比如我是一个B站的用户，那么每次我进入B站后，他都会根据我的喜好就行定制化推荐，比如我经常浏览鬼畜、二次元、计算机方面的视频，就会给我推荐这方面的视频和up主。但是如果是另一个同学，可能喜欢的是游戏，旅游，音乐，那么B站也应该推荐这些方面的内容。这个需求呢就可以通过**terms lookup**来做。我们一起来看下。

```json
# 假设这是B站的用户信息，id是用户名，categories里记录了B大数据分析得出的他平时喜欢的视频分类
PUT /user-profiles/_doc/hxy
{
  "categories" : ["technology","java","ghost livestock"]
}

PUT /user-profiles/_doc/yj
{
  "categories" : ["tourism"]
}

# 这里是B站的一些视频，其中每一个视频有一个分类，用category来表示
PUT /bilibili/_bulk
{"index":{"_id":"elasticsearch-definitive-guide"}}
{"name":"Elasticsearch - The definitive guide","category":"technology"}
{"index":{"_id":"seven-databases"}}
{"name":"Seven Databases in Seven Weeks","category":"technology"}
{"index":{"_id":"SpringBoot"}}
{"name":"Springboot learn note","category":"java"}
{"index":{"_id":"The potala palace"}}
{"name":"Day trip to Potala Palace","category":"tourism"}
```

接下来我们通过Terms lookup的方式进行查询，这里将索引bilibili和user-profiles进行了关联，首先在user-profiles这个索引里通过用户ID查询到用户的categories，然后将他作为动态的参数，最后在bilibili这个索引中，将前面的动态参数作为检索条件，从bilibili中查询，如果他里面的某个文档的category字段的数据和我们的动态参数中的能匹配上，就会被命中。

这里有四个参数，对上面说的过程进行了抽象

- `index`：用来表示你从中获取字段值的索引名称，这个是能确定的，比如上文提到的user-profiles

- `id`：你的文档ID，这个也是能确定的，用户一登录你就可以获取到

- `path`：用来表示你从中获取字段值的字段名称

- `routing`：这是一个非必选参数，在有自定义路由的时候使用，后续有机会在探讨

```json
POST /bilibili/_search
{
  "query": {
    "terms": {
      "category": {
        "index":"user-profiles",
        "id": "hxy",
        "path": "categories"
      }
    }
  }
}

"hits" : {
    "total" : {
        "value" : 3,
        "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
        {
            "_index" : "bilibili",
            "_type" : "_doc",
            "_id" : "elasticsearch-definitive-guide",
            "_score" : 1.0,
            "_source" : {
                "name" : "Elasticsearch - The definitive guide",
                "category" : "technology"
            }
        },
        {
            "_index" : "bilibili",
            "_type" : "_doc",
            "_id" : "seven-databases",
            "_score" : 1.0,
            "_source" : {
                "name" : "Seven Databases in Seven Weeks",
                "category" : "technology"
            }
        },
        {
            "_index" : "bilibili",
            "_type" : "_doc",
            "_id" : "SpringBoot",
            "_score" : 1.0,
            "_source" : {
                "name" : "Springboot learn note",
                "category" : "java"
            }
        }
    ]
}
```

### fuzzy（[Fuzzy Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html)）

上面提到的term查询是一种精确查询，必须要求你输入的查询条件和文档中的数据完全匹配才可以。但是有时候可能用户忘了一个单词怎么写，只记得前3个字母，或者拼写错了，那么如果用term是查不到的。这个时候我们就需要根据用户的输入做一个猜测，进行一个模糊匹配。那么**fuzzy**正好是为了解决这个问题而出现的。

为了理解fuzzy的用法，这里我们需要先了解一个概念：**编辑距离**，他是一个单词要变成另一个单词，需要经过几次转换的这个次数。比如java这个单词要变成jbva，只需要将b替换成a，因此这个编辑距离就是1。那么这个**转换**一共有四种手段：

1. 改变其中一个字母，比如**b**ox -> **f**ox
2. 删除其中一个字符，比如**b**lack -> lack
3. 插入一个新的字母，比如sic -> sic**k**
4. 调整两个相邻字母的位置，比如**ca**t -> **ac**t

fuzzy实现模糊查询就是基于这个编辑距离去做的，他会将用户输入的搜索条件，结合一个编辑距离，然后基于我们上面提到的四种手段去做一个扩展和变形，得到一个集合，这个过程我们可以叫做**扩展模糊选项**，然后将扩展后的模糊选项作为查询条件展开精确匹配，最终将所有的结果进行返回。那么这个编辑距离就需要我们去指定，在es中是通过`fuzziness`去指定的，他的含义就是最大编辑距离。

```json
POST /baike/_search
{
  "query": {
    "fuzzy": {
      "title.keyword": {
        "value": "ElasticSearh",   # es中存储的是ElasticSearch，编辑距离是2，那么只要你的输入经过两次转换都变成ElasticSearch，就会匹配到
        "fuzziness": 2
      }
    }
  }
}
```

fuzzy除了可以指定最大编辑距离之外，还有其他几个参数，这里在做一个总结：

- `fuzziness`：最大编辑距离，值可以是数字类型，也可以是`AUTO:[low],[high]`这种格式，表示如果查询的单词长度在**[0,low)**这个范围内，编辑距离为0，如果在**[low,high)**这个范围之内，编辑距离为1，如果**大于high**，则编辑距离为2；除了这两种格式外，还支持`AUTO`这种写法，等同于`AUTO:3,6`

- `max_expansions`：最大扩展数量，前面我们提到了扩展模糊选项，假如一个查询扩展了3到5个扩展选项，那么是是很有意义的，如果扩展了1000个模糊选项，其实也就意义不大了，会让我们又迷失在海量的数据中。因此有了`max_expansions`这个参数，限制最大扩展数量，默认值是50。切记这个值不可以太大，否则会导致性能问题

- `prefix_length`：指定开始多少个字符不可以被模糊

- `transpositions`：boolean值，默认true，表示扩展模糊选项的时候，是否包含两个相邻字符的位置互换这种手段。实践过程设置了false，理论上我存储ElasticS**ea**rch，检索ElasticS**ae**rch应该搜不到，但是却搜索到了。有点不太理解。

- `rewrite`：参考上文

## 组合查询

### bool （[Boolean Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)）

其实大多数时候我们是需要通过好几个条件去查询的，比如说豆瓣搜索一部电影，我们可能会限定根据电影的类型、年份、豆瓣评分等多个条件去查询，那么这种场景其实就是多个检索条件和多个字段匹配的一种场景。在ES中有一种查询，叫做bool查询，他可以组合多个查询字句，然后将结果输出，并且他是支持嵌套子句的。他支持的查询字句可以分为四种类型：

- must：必须匹配，会影响算分结果
- should：选择性匹配，也会影响算分结果
- must_not：必须不能匹配，不会影响算分
- filter：必须匹配，不会影响算分

下面是官网提供的一个关于bool查询的例子，其中must和should字句会进行相关性算分，并且累计到最终的分数中。bool查询可以通过`minimum_should_match`来指定should查询中的term子查询必须匹配几个才可以算是真正的匹配到这条数据。假设现在就是查询一部电影，我们搜索限定评分要大于9分，类型是文艺片，上映时间是2021年，演员有张国荣。那么如果不指定`minimum_should_match`，可能这四个条件中有一个满足就能查到，但是如果指定了`minimum_should_match=3`，那么这四个条件中必须满足三个才会返回。

```json
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user.id" : "kimchy" }
      },
      "filter": {
        "term" : { "tags" : "production" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [    # 一个数组，包括了两个term查询，如果没有指定must条件，那么should查询中的term必须至少满足一条查询
        { "term" : { "tags" : "env1" } },
        { "term" : { "tags" : "deployed" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```

### boosting（[Boosting query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-boosting-query.html)）

假设现在我们有下面这样一个索引，包括三个文档，其中前两条是Apple公司的电子产品介绍，后面一条是水果Apple的百科介绍，那么如果我们通过下面的查询条件去匹配，会既查询到苹果手机，也会查询到水果里的苹果。

```json
POST /baike/_search
{
  "query": {
    "bool": {
      "must": {
        "match":{"title":"Apple"}
      }
    }
  }
}
```

```json
"hits" : {
    "total" : {
        "value" : 3,
        "relation" : "eq"
    },
    "max_score" : 0.1546153,
    "hits" : [
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 0.1546153,
            "_source" : {
                "title" : "Apple Pad"
            }
        },
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : 0.1546153,
            "_source" : {
                "title" : "Apple Mac"
            }
        },
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "3",
            "_score" : 0.1546153,
            "_source" : {
                "title" : "Apple Pie and Apple Fruit"
            }
        }
    ]
}
```

但是也许我们的用户关注的并不是水果的苹果，而是电子产品，那么我们应该如何进行更精确的匹配呢？当然我们可以对前面的查询做一些修改，通过mast_not来排除title中包括pie或者fruit的文档，只返回Apple Pad和Apple Mac。但是这么做又似乎有一点绝对，虽然很多人确实是想找苹果手机，但是也总有人是要看看什么苹果好吃，那么有没有什么折中的办法呢？

```json
POST /baike/_search
{
  "query": {
    "bool": {
      "must": {
        "match":{"title":"Apple"}
      },
      "must_not": {
        "match":{"title":"Pie"}
      }
    }
  }
}
```

在ES中，为我们提供了**Boosting query**这种查询方式（boosting：boost的现在分词形式，有提高，助推的意思，这里我理解是提高_score这个分值），他可以为我们匹配到用户最关心的苹果手机，也可以匹配到吃的苹果。并且可以指定让最受关注的苹果手机展示在搜索结果的最前面。写法大概如下：

这里对几个属性做一个简单的分析：

- `positive`：翻译过来有积极地意思，用来指定我们最关心的，希望靠前展示，算分高的文档
- `negative`：翻译过来有消极地意思，用来指定我们不是很关心，但是还是希望他能被匹配到的文档
- `negative_boost`：这个是为`negative`里面的条件指定一个boost值，用来降低他们的算分，在0.0-1.0之间的一个float数字

```json
POST /baike/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "title": "Apple"
        }
      },
      "negative": {
        "match": {
          "title": "fruit"
        }
      },
      "negative_boost": 0.5   # 通过这个字段结合上面的negative里的条件，在查询的时候就会将包含fruit的那条数据的算分打的很低，让他排在最后展示
    }
  }
}
```

### costant_score （[Constant score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html)）

我们知道filter查询是不会进行算分的，而且es会自动缓存一些filter查询，以此来提高一个效率。有时候可能确实需要返回一个期望的算分，那么`constant_score`可以用来做这件事，他可以对filter查询进行一次包装，然后通过`boost`这个参数来指定返回一个常量的算分。**constant**（常量）

```json
POST /baike/_search
{
  "query": {
    "constant_score": {
      "filter": {"term": {"title.keyword": "Quick brown rabbits"}},
      "boost": 1.2
    }
  }
}

"hits" : {
    "total" : {
        "value" : 1,
        "relation" : "eq"
    },
    "max_score" : 1.2,
    "hits" : [
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "4",
            "_score" : 1.2,   # 通过上面查询，这里返回的算分和我们指定的boost分值相等
            "_source" : {
                "title" : "Quick brown rabbits",
                "body" : "Brown rabbits are commonly seen."
            }
        }
    ]
}
```

### dis_max（[Disjunction max query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-dis-max-query.html)）

上面说到了bool查询，我们这里在回顾一下，首先这里我从es中文网站找了两条测试数据，

```json
POST baike/_doc/4
{
  "title": "Quick brown rabbits",
  "body": "Brown rabbits are commonly seen."
}

POST baike/_doc/5
{
  "title": "Keeping pets healthy",
  "body": "My quick brown fox eats rabbits on a regular basis."
}
```

假设我们现在要在title或者body里查询`brown fox`相关的内容，那么我们通过观察发现ID为5的这条数据应该是相关性更高的，因为他的body里出现了完整的`brown fox`这个搜索条件，那么我们当然希望他能获得更高的算分，稍微靠前一点展示，接下来我们用bool查询试试看会不会和我们想的一样，下面是结果：

```json
POST /baike/_search
{
  "query": {
    "bool": {
      "should": [
        {"match": {"title": "Brown fox"}},
        {"match": {"body": "Brown fox"}}
      ]
    }
  }
}

# 查询结果
"hits" : {
    "total" : {
        "value" : 2,
        "relation" : "eq"
    },
    "max_score" : 1.5974034,
    "hits" : [
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "4",
            "_score" : 1.5974034,
            "_source" : {
                "title" : "Quick brown rabbits",
                "body" : "Brown rabbits are commonly seen."
            }
        },
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "5",
            "_score" : 0.77041256,
            "_source" : {
                "title" : "Keeping pets healthy",
                "body" : "My quick brown fox eats rabbits on a regular basis."
            }
        }
    ]
}
```

实际操作过程中我们发现ID为5的这条数据并没有得到更高的算分，这是为什么呢？为了回答这个问题，我们要知道在es中也可以类似mysql查询sql的执行计划一样，通过`explain`这个关键字来展示dsl的执行计划，包括算分方式。接下来让我们一起拭目以待吧：

```json
POST /baike/_search
{
  "query": {
    "bool": {
      "should": [
        {"match": {"title": "Brown fox"}},
        {"match": {"body": "Brown fox"}}
      ]
    }
  },
  "explain": true
}

# 查询结果
"hits" : {
  "total" : {
    "value" : 2,
    "relation" : "eq"
  },
  "max_score" : 1.5974034,
  "hits" : [
    {
      "_shard" : "[baike][0]",
      "_node" : "aPt8G7vHTzOJU_L2FdLBpA",
      "_index" : "baike",
      "_type" : "_doc",
      "_id" : "4",
      "_score" : 1.5974034,
      "_source" : {
        "title" : "Quick brown rabbits",
        "body" : "Brown rabbits are commonly seen."
      },
      "_explanation" : {
        "value" : 1.5974034,  # 这个值约等于38行的value + 49行的value
        "description" : "sum of:",   # !!! 求和
        "details" : [
          {
            "value" : 1.3862942, 
            "description" : "weight(title:brown in 0) [PerFieldSimilarity], result of:",  # title中有关键字brown，算一次
            "details" : [
              {
                "value" : 1.3862942,
                "description" : "score(freq=1.0), computed as boost * idf * tf from:",
                "details" : [] # 算分细节，因为太长省略
              }
            ]
          },
          {
            "value" : 0.21110919,
            "description" : "weight(body:brown in 0) [PerFieldSimilarity], result of:", # body中有关键字brown，算一次分
            "details" : [
              {
                "value" : 0.21110919,
                "description" : "score(freq=1.0), computed as boost * idf * tf from:",
                "details" : []
              }
            ]
          }
        ]
      }
    },
    {
      "_shard" : "[baike][0]",
      "_node" : "aPt8G7vHTzOJU_L2FdLBpA",
      "_index" : "baike",
      "_type" : "_doc",
      "_id" : "5",
      "_score" : 0.77041256,
      "_source" : {
        "title" : "Keeping pets healthy",
        "body" : "My quick brown fox eats rabbits on a regular basis."
      },
      "_explanation" : {
        "value" : 0.77041256,
        "description" : "sum of:",
        "details" : [
          {
            "value" : 0.160443,
            "description" : "weight(body:brown in 0) [PerFieldSimilarity], result of:", # body中有关键字brown，算一次分
            "details" : [
              {
                "value" : 0.160443,
                "description" : "score(freq=1.0), computed as boost * idf * tf from:",
                "details" : []
              }
            ]
          },
          {
            "value" : 0.60996956,
            "description" : "weight(body:fox in 0) [PerFieldSimilarity], result of:", # body中有关键字fox，算一次分
            "details" : [
              {
                "value" : 0.60996956,
                "description" : "score(freq=1.0), computed as boost * idf * tf from:",
                "details" : []
              }
            ]
          }
        ]
      }
    }
  ]
}
```

通过对执行计划的分析，我们发现在bool查询会将should里面两个子查询分别进行算分，然后做加法，得到一个总的分数，在ID为4的文档中，title和body中分别包含了brown这个关键字，而ID为5的文档呢，因为title中没有包含查询条件中任何一个字符，因此它的算分下来就偏低，最终排在了后面。

显而易见，这种局面并不是我们想要看到的，那么有没有什么办法呢？es中就提供了一种解决方案，叫做`dis_max`。接下来我们用他再做一次查询，看看有什么结果，很明显ID为5的这条数据这一次获得了一个较高的算分。

```json
POST /baike/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match": {"title": "Brown fox"}},
        {"match": {"body": "Brown fox"}}
      ]
    }
  }
}

"hits" : {
    "total" : {
        "value" : 2,
        "relation" : "eq"
    },
    "max_score" : 0.77041256,
    "hits" : [
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "5",
            "_score" : 0.77041256,
            "_source" : {
                "title" : "Keeping pets healthy",
                "body" : "My quick brown fox eats rabbits on a regular basis."
            }
        },
        {
            "_index" : "baike",
            "_type" : "_doc",
            "_id" : "4",
            "_score" : 0.6931471,
            "_source" : {
                "title" : "Quick brown rabbits",
                "body" : "Brown rabbits are commonly seen."
            }
        }
    ]
}
}
```

我们在用explain看看他的执行计划，发现他这次不是单纯的将两个子查询的算分加起来，而是选了两个子查询中算分的最大值做为他的最终得分。

```json
"hits" : {
  "total" : {
    "value" : 2,
    "relation" : "eq"
  },
  "max_score" : 0.77041256,
  "hits" : [
    {
      "_shard" : "[baike][0]",
      "_node" : "tc1MvVwdRcO-2A5L6j_l0Q",
      "_index" : "baike",
      "_type" : "_doc",
      "_id" : "5",
      "_score" : 0.77041256,
      "_source" : {
        "title" : "Keeping pets healthy",
        "body" : "My quick brown fox eats rabbits on a regular basis."
      },
      "_explanation" : {
        "value" : 0.77041256,
        "description" : "max of:",  # !!! 求最大值
        "details" : [
          {
            "value" : 0.77041256,
            "description" : "sum of:",
            "details" : [
              {
                "value" : 0.160443,
                "description" : "weight(body:brown in 1) [PerFieldSimilarity], result of:",
                "details" : []
              },
              {
                "value" : 0.60996956,
                "description" : "weight(body:fox in 1) [PerFieldSimilarity], result of:",
                "details" : []
              }
            ]
          }
        ]
      }
    },
    {
      "_shard" : "[baike][0]",
      "_node" : "tc1MvVwdRcO-2A5L6j_l0Q",
      "_index" : "baike",
      "_type" : "_doc",
      "_id" : "4",
      "_score" : 0.6931471,
      "_source" : {
        "title" : "Quick brown rabbits",
        "body" : "Brown rabbits are commonly seen."
      },
      "_explanation" : {
        "value" : 0.6931471,
        "description" : "max of:",
        "details" : [
          {
            "value" : 0.6931471,
            "description" : "sum of:",
            "details" : [
              {
                "value" : 0.6931471,
                "description" : "weight(title:brown in 0) [PerFieldSimilarity], result of:",
                "details" : []
              }
            ]
          },
          {
            "value" : 0.21110919,
            "description" : "sum of:",
            "details" : [
              {
                "value" : 0.21110919,
                "description" : "weight(body:brown in 0) [PerFieldSimilarity], result of:",
                "details" : []
              }
            ]
          }
        ]
      }
    }
  ]
}
```

但是有时候完全取最高的，直接忽略掉其他查询字句的分值，也不是很合理。毕竟优秀的人总是凤毛麟角，普通人的力量也不容小觑，因此我们也要考虑到。ES也为我们提供了一个参数：`tie_breaker`。他的有效值在0.0-1.0之间的一个浮点数，默认是0.0，如果我们设置了这个字段，那么在算分的时候，首先他会取最高分，然后和所有子查询的得分乘以`tie_breaker`的值相加，求取一个最终的算分。那么在这个过程中，他给了最高算分和其他子查询算分一个权重，既考虑了极个别优先卓越人物的贡献，也考虑到了人民群众的力量。那么分析了这么多，我们在理解下为什么叫**dis_max**，dis也就是Disjunction的缩写，有分离，提取的意思，max是最大的意思，因此他就是将组合查询分离成多个子查询，去算分最高的作为最终得分。他是一个帮助我们选取最佳匹配的一种有效手段。

```json
POST /baike/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match": {"title": "Brown fox"}},
        {"match": {"body": "Brown fox"}}
      ],
      "tie_breaker": 0.7
    }
  }
}
```

## 全文检索

### match_all

**match_all**是没有任何条件，检索全部数据

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_all": {}
  }
}
```

### match（[Match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)）

**match**用来做基本的模糊匹配，在es中会对文本进行分词，在match查询的时候也会对查询条件进行分词，然后通过倒排索引找到匹配的数据。在match中支持以下参数：

- `query`：查询条件
- `operator`：匹配条件（**AND**、**OR (Default)**）
- `minimum_should_match`：最小匹配的数量，用来指定文档中至少包含几个关键字才算匹配到
- `fuzziness`：最大编辑距离，详细参考上文
- `prefix_length`：通过最大编辑距离进行模糊查询额时候，开始的多少个字符不允许被模糊，默认值为0。下面有例子
- `fuzzy_transpositions`：boolean值，默认true，表示扩展模糊选项的时候，是否包含两个相邻字符的位置互换这种手段
- `fuzzy_rewrite`：参考前文提到的**rewrite**
- `analyzer`：可以指定分词器，如果不指定，用默认的
- `max_expansions`：参考上文
- `zero_terms_query`：在实际的文档中，可能有很多这样的词，比如中文中的**的，了，呢**，或者英文中的**or、and、is、do**等。那么这样的词对我们的搜索可能是没有任何帮助的，我们把这些的词叫做**停用词**，ES中有一个停用词分析器：[Stop token filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-stop-tokenfilter.html)，如果我们的查询请求的关键字中包括这些词，并且用到了这个分析器，那么他会帮我们把这些停用词移除掉。如果我们的查询请求中所有的关键字都被移除掉了，就不会匹配到任何文档，那么这个时候，是否要给用户返回一个空呢？ES提供了两种策略，也就是通过这个字段去表示的：
  - `none (default)`：不返回任何文档
  - `all`：返回所有文档，相当于执行了`match_all`
- `lenient`：**lenient**有仁慈的，宽容的意思。这里表示是否忽略一些输入错误，比如为一个数字类型的字段输入了一个字符串去匹配，如果设置为true，会忽略，默认值是false
- `auto_generate_synonyms_phrase_query`：在有些场景，可能一个意思有两种写法，比如ElasticSearch有些人可能会写成ES，虽然写法不一样，但是描述的是一个东西，那么如果我们限定查询条件为ElasticSearch，其实也是希望能搜索到ES相关内容的。我们可以把这种词叫做同义词查询。因此ES为我们提供了这个参数，表示是否开启同义词查询，默认是true，也就是开启的。但是有一个问题就是ES他怎么知道那些词是同义词呢？Lucene中有一个概念叫**Synonym Graph Token Filter**，那么ES中也是有的，我们可以通过对这个进行配置来实现同义词查询，配置方式参考：[ Synonym graph token filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-synonym-graph-tokenfilter.html)

```json
# match查询中，可以指定查询条件
# 下面的语句会将Eddie Underwood分词，然后只要customer_full_name有这Eddie和Underwood中一个就会命中
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name": "Eddie Underwood"
    }
  }
}

# 在上面的查询中，Eddieh或者Underwood出现一个就可以，有时候如果查询目标内容庞大，其中包括了我们检索的一两个字，也会被搜索到
# 但是其实可能相关性并不大，因此我们还可以通过minimum_should_match指定最少匹配到多少个单词才算匹配到
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name": {
        "query": "Eddie Underwood",
        "minimum_should_match": 2
      }
    }
  },
  "_source":["customer_full_name","currency"]
}

# 当然match中还支持通过operator来指定匹配条件
# 下面的语句会的意思是只有customer_full_name这个字段中同时出现Eddie和Underwood才能命中
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name": {
        "query": "Eddie Underwood",
        "operator": "and"
      }
    }
  }
}

# 很多时候可能一开始就犯错的机会不大，走着走着路才走偏了，所以如果这个参数设置合理，可以减少模糊的次数，提升性能
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name": {
        "query": "Eadie Underwood",
        "fuzziness": 2,  # 限定最大编辑距离为2
        "prefix_length": 2, # 这里默认是0，那么我们输入Eadie是可以匹配到Eddie的，但是如果指定为2，就匹配不到了
        "operator": "and"   # title中如果有Eadie和Underwood才会被匹配到，在文档中有Eddie Underwood的文档
      }
    }
  }, 
  "_source": "customer_full_name"
}
```

### match_phrase（[Match phrase query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html)）

**match_phrase**（phrase： 短语） 会对输入做分词，但是需要结果中也包含所有的分词，并且顺序要求一致。这个条件其实有一点苛刻了，有时候可能我输入错了，或者一个短语，只记得其中两个单词，第三个单词死活记不起来怎么办呢？ES也提供了`slop`这个参数帮我们解决这个问题：

- `slop (default 0)`：（**slop**：溢出）来指定额外加几个单词也可以命中。

```json
# match_phrase这种方式，是把query中的条件作为一整个单词进行查询，只有精确匹配到Eddie Underwood才会命中
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_phrase": {
      "customer_full_name": {
        "query": "Eddie Underwood", 
        "slop": 1  # 如果是Eddie test Underwood也可以被命中
      }
    }
  }
}
```

### match_phrase_prefix（[Match phrase prefix query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html)）

**match_phrase_prefix**可以看做**match_phrase**的一个扩展，他会把query的查询条件进行分词，然后把最后一个单词看做是一个前缀，匹配索引中所有以这个单词为前缀的单词，然后进行返回

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_phrase_prefix": {
      "customer_full_name":{
        "query": "Eddie"  # 可以匹配到Eddie Underwood、Eddie Utest
      }
    }
  }
}
```

### multi match（[Multi-match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)）

**multi_match**是在match的基础进行了一些加强，他支持在对个字段进行查询。同时在字段的描述上还支持通配符。

```json
# 在customer_full_name、customer_first_name中有一个字段包括查询条件就可以匹配到
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": "Eddie",
      "fields": ["customer_full_name","customer_first_name"]  # 可以写成["*_name"]
    }
  },
  "_source":["customer_full_name","currency","customer_first_name"]
}
```

在multi match查询的时候，有时候候可能存在这样的场景，比如说你在CSDN搜索一篇包括mutli和match关键字的博客，这两个关键字可能在文章的标题中有，在内容中也有，但是明显在标题中同时出现这两个关键字的文章可能是我们更关心的，也就是相关性更大的，那么如何让这篇文章脱颖而出，排列在搜索结果的最前面呢？

其实ES会为搜索到的每一个文档做一个算分，通过**_score**这个字段表示，最终会根据这个字段进行一个排序，分数越大的结果越靠前展示。那么在上面这个场景中，我们就可以指定让标题这个字段在算分的时候占有更大的权重，写法类似下面这样：

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": "Eddie",
      "fields": ["customer_full_name^3","customer_first_name"]  # 这里通过^3来指定他的算分比重，这个数字越大，他占的比重就越大，其他字段占得比重就越小
    }
  },
  "_source":["customer_full_name","currency","customer_first_name"]
}
```

在**multi_match**这种查询方式中，还有一个特别重要的字段`type`，他的每一个查询其实都是依赖这个字段的，虽然我们上面没有指定，那是因为他有一个默认值`best_fields`。接下来我们一起看看，除了`best_fields`之外还有那些类型以及每一个类型有什么用处。

- `best_fields`：以前面讨论`dis_max`为例，比如我们要在一个文档中的title和body两个字段中查找`brown fox`，如果我们要查找的两个单词按照顺序同时出现，那么肯定是比分别出现在不同的字段中是有意义的，我们就把两个单词同时按照顺序出现在一个字段中这种情况叫做`best_fields`。这样的话，`multi_match`就会被包装成`dis_max`这种方式,去查询。那我们知道`dis_max`还支持一个参数`tie_breaker`，因此这里也是支持的

  ```json
  GET baike/_search
  {
    "query": {
      "multi_match": {
        "query": "Brown fox",
        "fields": ["title","body"],
        "type": "best_fields",
        "tie_breaker": 0.1
      }
    }
  }
  
  # 上面的请求最终会按照这样的方式去执行
  POST /baike/_search
  {
    "query": {
      "dis_max": {
        "queries": [
          {"match": {"title": "Brown fox"}},
          {"match": {"body": "Brown fox"}}
        ],
        "tie_breaker": 0.1
      }
    }
  }
  ```

- `most_fields`：`best_fields`是取子查询中算分最高的作为最终算分，而`most_fields`则是取子查询算分的和作为最终算分，我们也可以理解按照关键字出现的次数来算，出现的越多，算分越高，这里还是可以参考前文关于对`dis_max`的讨论来理解。

  ```json
  GET baike/_search
  {
    "query": {
      "multi_match": {
        "query": "Brown fox",
        "fields": ["title","body"],
        "type": "most_fields",
        "tie_breaker": 0.1
      }
    }
  }
  
  # 上面的请求最终会按照这样的方式去执行
  POST baike/_search
  {
    "query": {
      "bool": {
        "should": [
          {"match": {"title":"Brown fox"}},
          {"match": {"body":"Brown fox"}}
        ]
      }
    }
  }
  ```

- `phrase` and `phrase_prefix`：这两种类型和`best_fields`的算分策略是一致的，我们讨论`best_fields`的时候在示例代码中说明他最终会被翻译为`dis_max`加上`match`去执行，那么这两种类型就是把`match`用`match_phrase`和`match_phrase_prefix`替换掉。

  ```json
  GET baike/_search
  {
    "query": {
      "multi_match": {
        "query": "Brow",
        "fields": ["title","body"],
        "type": "phrase_prefix",
        "tie_breaker": 0.1
      }
    }
  }
  
  # 上面的请求最终会按照这样的方式去执行
  GET baike/_search
  {
    "query": {
      "dis_max": {
        "tie_breaker": 0.1,
        "queries": [
          {"match_phrase_prefix": {"title": "Brow"}},
          {"match_phrase_prefix": {"body": "Brow"}}
        ]
      }
    }
  }
  ```

- `bool_prefix`：这种类型和`most_fields`的算分策略是一致的，不过他会用`match_bool_prefix`替换掉`match`。

  ```json
  GET baike/_search
  {
    "query": {
      "multi_match": {
        "query": "Brow",
        "fields": ["title","body"],
        "type": "bool_prefix"
      }
    }
  }
  
  # 上面的请求最终会按照这样的方式去执行
  GET baike/_search
  {
    "query": {
      "bool": {
        "should": [
          {"match_bool_prefix": {"title": "Brow"}},
          {"match_bool_prefix": {"body": "Brow"}}
        ]
      }
    }
  }
  ```

- `cross_fields`：参考**跨字段查询cross_fields**中的介绍。

## 脚本字段

-  脚本字段查询

  ```json
  # 在查询的时候支持通过脚本字段的方式进行一些逻辑处理
  POST kibana_sample_data_logs/_search
  {
    "from": 0, 
    "size": 20,
    "script_fields": {
      "new_field": {   # 通过painless这种es提供的脚本，来取出doc中的timestamp字段，拼接一个China字符作为new_field输出
        "script": {
          "lang": "painless",
          "source": "doc['timestamp'].value+' China'"
        }
      }
    }, 
    "query": {
      "match_all": {}
    },
    "_source": ["clientip", "ip", "message", "timestamp", "request","response"]
  }
  ```