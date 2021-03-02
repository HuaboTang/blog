---
title: ElasticSearch中文分词及查询条件
---

# Install IK中文分词器
```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.3.0/elasticsearch-analysis-ik-6.3.0.zip

restart
```

# 查询条件
### match

最简单的一个match例子：

查询和"我的宝马多少马力"这个查询语句匹配的文档。

```javascript
{
  "query": {
    "match": {
        "content" : {
            "query" : "我的宝马多少马力"
        }
    }
  }
}
```

上面的查询匹配就会进行分词，比如"宝马多少马力"会被分词为"宝马 多少 马力", 所有有关"宝马 多少 马力", 那么所有包含这三个词中的一个或多个的文档就会被搜索出来。

并且根据lucene的评分机制(TF/IDF)来进行评分。

### match_phrase

比如上面一个例子，一个文档"我的保时捷马力不错"也会被搜索出来，那么想要精确匹配所有同时包含"宝马 多少 马力"的文档怎么做？就要使用 match_phrase 了

```javascript
{
  "query": {
    "match_phrase": {
        "content" : {
            "query" : "我的宝马多少马力"
        }
    }
  }
}
```

完全匹配可能比较严，我们会希望有个可调节因子，少匹配一个也满足，那就需要使用到slop。

```javascript
{
  "query": {
    "match_phrase": {
        "content" : {
            "query" : "我的宝马多少马力",
            "slop" : 1
        }
    }
  }
}
```
### multi_match

如果我们希望两个字段进行匹配，其中一个字段有这个文档就满足的话，使用multi_match

```javascript
{
  "query": {
    "multi_match": {
        "query" : "我的宝马多少马力",
        "fields" : ["title", "content"]
    }
  }
}
```
但是multi_match就涉及到匹配评分的问题了。

我们希望完全匹配的文档占的评分比较高，则需要使用best_fields

```javascript
{
  "query": {
    "multi_match": {
      "query": "我的宝马发动机多少",
      "type": "best_fields",
      "fields": [
        "tag",
        "content"
      ],
      "tie_breaker": 0.3
    }
  }
}
```

意思就是完全匹配"宝马 发动机"的文档评分会比较靠前，如果只匹配宝马的文档评分乘以0.3的系数

我们希望越多字段匹配的文档评分越高，就要使用most_fields

```javascript
{
  "query": {
    "multi_match": {
      "query": "我的宝马发动机多少",
      "type": "most_fields",
      "fields": [
        "tag",
        "content"
      ]
    }
  }
}
```

我们会希望这个词条的分词词汇是分配到不同字段中的，那么就使用cross_fields

```javascript
{
  "query": {
    "multi_match": {
      "query": "我的宝马发动机多少",
      "type": "cross_fields",
      "fields": [
        "tag",
        "content"
      ]
    }
  }
}
```

### term

term是代表完全匹配，即不进行分词器分析，文档中必须包含整个搜索的词汇

```javascript
{
  "query": {
    "term": {
      "content": "汽车保养"
    }
  }
}
```

查出的所有文档都包含"汽车保养"这个词组的词汇。

使用term要确定的是这个字段是否“被分析”(analyzed)，默认的字符串是被分析的。

拿官网上的例子举例：

mapping是这样的：

```javascript
// PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "full_text": {
          "type":  "string"
        },
        "exact_value": {
          "type":  "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```

```javascript
// PUT my_index/my_type/1
{
  "full_text":   "Quick Foxes!",
  "exact_value": "Quick Foxes!"  
}
```

其中的full_text是被分析过的，所以full_text的索引中存的就是[quick, foxes]，而extra_value中存的是[Quick Foxes!]。

那下面的几个请求：

```javascript
GET my_index/my_type/_search
{
  "query": {
    "term": {
      "exact_value": "Quick Foxes!"
    }
  }
}
```

请求的出数据，因为完全匹配

```javascript
GET my_index/my_type/_search
{
  "query": {
    "term": {
      "full_text": "Quick Foxes!"
    }
  }
}
```

请求不出数据的，因为full_text分词后的结果中没有[Quick Foxes!]这个分词。

bool联合查询: must,should,must_not
如果我们想要请求"content中带宝马，但是tag中不带宝马"这样类似的需求，就需要用到bool联合查询。

联合查询就会使用到must,should,must_not三种关键词。

这三个可以这么理解

- must: 文档必须完全匹配条件
- should: should下面会带一个以上的条件，至少满足一个条件，这个文档就符合should
- must_not: 文档必须不匹配条件

比如上面那个需求：

```Javascript
{
  "query": {
    "bool": {
      "must": {
        "term": {
          "content": "宝马"
        }
      },
      "must_not": {
        "term": {
          "tags": "宝马"
        }
      }
    }
  }
}
```