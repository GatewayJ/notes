---
author : "jihongwei"
tags : ["README"]
date : 2020-11-16T11:24:00Z
title : "elasticsearch-helloword"
---


一些基本的请求

*http://10.52.3.41:9200/_aliases?pretty=true

*http://wsl.local:9200/_cat/indices?v

*http://wsl.local:9200/_stats

query查询

  &nbsp;&nbsp;在过滤结果集的同时，会计算结果文档和查询条件的相关度，并将返回结果集按照相关度的高低排序。从下图中也可以看出，在返回的数据中每个文档都有_score属性，并且属性值都不为0，返回结果集也是按照_score属性值的高低排序的。由于每次新增或者删除文档都有可能导致计算出来的_score值发生变化，需要重新计算，所以ES不会缓存Query的结果。

Filter查询
&nbsp;&nbsp;只是按照搜索条件过滤出需要的数据而已，不计算任何相关度分数，对相关度没有任何影响。从下图中也可以看出，虽然返回的每个文档都有_score属性，但属性值都为0。由于Filter相当于只是回答是或者不是，所以ES会使用Page Cache将Filter的结果缓存起来，当新增和删除文档时， 也会更新FIlter的结果。



term、terms查询
term query会去倒排索引中寻找确切的term，它并不知道分词器的存在，这种查询适合keyword、numeric、date等明确值的（即没有被分词的）
term：查询某个字段里含有某个关键词的文档
```
GET /customer/doc/_search/
{
  "query": {
    "term": {
      "title":   "blog"
    }
  }
}

terms：查询某个字段里含有多个关键词的文档
GET /customer/doc/_search/
{
  "query": {
    "terms": {
      "title":  [ "blog","first"]
    }
  }
}
```


match查询
match query 知道分词器的存在，会对field进行分词操作，然后再查询
match会分词拆分，并根据match.field_name.operator 控制查询规则
```
GET /customer/doc/_search/
{
  "query": {
    "match": {
      "title":  "my ss"   #它和term区别可以理解为term是精确查询，这边match模糊查询；match会对my ss分词为两个单词，然后term对认为这是一个单词
    }
  }
}

multi_match：可以指定多个字段
GET /customer/doc/_search/
{
  "query": {
    "multi_match": {
      "query" : "blog",
      "fields":  ["name","title"]   #只要里面一个字段包含值 blog 既可以
    }
  }
}
```


bool查询   联合查询

shoule在与must或者filter同级时，默认是不需要满足should中的任何条件的，此时我们可以加上minimum_should_match 参数



查询上下文和过滤器上下文


Query查询上下文：


&nbsp;&nbsp;在查询上下文中，查询会回答这个问题——“这个文档匹不匹配这个查询，它的相关度高么？”


如何验证匹配很好理解，如何计算相关度呢？之前说过，ES中索引的数据都会存储一个_score分值，分值越高就代表越匹配。另外关于某个搜索的分值计算还是很复杂的，因此也需要一定的时间。


查询上下文 是在 使用query进行查询时的执行环境，比如使用search的时候。


Filter过滤器上下文：


在过滤器上下文中，查询会回答这个问题——“这个文档匹不匹配？”


答案很简单，是或者不是。它不会去计算任何分值，也不会关心返回的排序问题，因此效率会高一点。


过滤上下文 是在使用filter参数时候的执行环境，比如在bool查询中使用Must_not或者filter。

 
另外，经常使用过滤器，ES会自动的缓存过滤器的内容，这对于查询来说，会提高很多性能。
总结



1 查询上下文中，查询操作不仅仅会进行查询，还会计算分值，用于确定相关度；在过滤器上下文中，查询操作仅判断是否满足查询条件


2 过滤器上下文中，查询的结果可以被缓存。



分数

#todo