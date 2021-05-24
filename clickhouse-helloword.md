---
author : "jihongwei"
tags : ["clickhouse"]
date : 2021-05-19T11:24:00Z
title : "clickhouse 单节点基础使用 通过kafka"
---

![clickhouse](https://demoio.cn:90/blog-image/clickhouse-helloword.png)
1. 创建引擎表
```sql
drop table if exists topic_ch_kafka;
CREATE TABLE queue (
    q_date Date,\
    level String,\
    message String\
) ENGINE = Kafka('localhost:9093',  'my_topic',  'kafka_group_test',  'JSONEachRow');
```
2. 创建物化视图
```sql
CREATE MATERIALIZED VIEW consumer TO daily
    AS SELECT q_date AS day, level, message
    FROM queue;

CREATE MATERIALIZED VIEW default.aggview
ENGINE = AggregatingMergeTree()  ORDER BY (q_date, level)
AS SELECT
    day,
    level ,
    count()
FROM default.daily
GROUP BY q_date, level;
```
3. 创建结果表
```sql
CREATE TABLE daily (
    day Date,
    level String,
    message String
  ) ENGINE = MergeTree(day, (day, level), 8192);



```
4. kafka生产数据
```
生产者  ./kafka-console-producer.sh  --topic my_topic --broker-list 127.0.0.1:9093
        {"q_date":"2020-06-28","level":"level3","message":"message"}        

消费者  ./kafka-console-consumer.sh  --topic my_topic --bootstrap-server 127.0.0.1:9093  --from-beginning

移动游标 ./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9093 --topic my_topic --group kafka_group_test --execute  --reset-offsets  --to-latest 
```