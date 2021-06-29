---
author : "jihongwei"
tags : ["clickhouse"]
date : 2021-05-24T11:24:00Z
title : "clickhouse 集群基础使用 分布式表"
---

0. 创建网络保证重启服务ip不会变 
```bash
docker network create --subnet=171.18.0.1/16 clickhouse
```
1. 搭建zookeeper
```bash
docker run --restart=always \
--name zookeeper -p 2182:2181 \
--net clickhouse --ip 171.18.0.5 \
-v /root/meiqia/clickhouse/zk/zookeeper/data/:/data \
-v /root/meiqia/clickhouse/zk/zookeeper/datalog/:/datalog -v /home/allspark/zookeeper/logs/:/logs -d zookeeper


docker run --name zkui -p 9090:9090 \
 --link zookeeper:zookeeper \
 -e ZK_SERVER="zookeeper:2181" \
 -d registry.cn-hangzhou.aliyuncs.com/wkaca7114/zkui
# zkui默认用户名和密码admin:manager
 ```

2. 搭建节点

(1) 先搭建一个单节点  拷贝出配置文件 用来当节点配置的模板，并将此配置复制两份用来给两个节点使用,分别放在`/root/meiqia/clickhouse/clickhouse-server,/root/meiqia/clickhouse/clickhouse-server2`
```
docker run -d \
--name clickhouse-server \
-p 9000:9000 \
-p 8123:8123 \
-p 9009:9009 \
--ulimit nofile=262144:262144 \
yandex/clickhouse-server

docker cp clickhouse-server:/etc/clickhouse-server/ /root/meiqia/clickhouse/
```

(2) 修改配置

在配置文件目录内创建 metrika.xml文件
```xml
<yandex>
    <clickhouse_remote_servers><!-- confix.xml内定义标签 -->
        <clickhouse_cluster_name><!-- 集群节点名称 可自定义 -->
        <shard><!-- 分片配置 -->
            <internal_replication>false</internal_replication>
            <replica>
                <host>172.17.0.4</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard><!-- 分片配置 -->
            <internal_replication>false</internal_replication>
            <replica>
                <host>172.17.0.7</host>
                <port>9000</port>
            </replica>
        </shard>
        </clickhouse_cluster_name>
    </clickhouse_remote_servers>


<zookeeper-servers> <!--zookeeper配置 -->
  <node index="1">
    <host>172.17.0.5</host>
    <port>2181</port>
  </node>
</zookeeper-servers>

<macros>
    <shard>01</shard> <!--当前节点配置的分片号-->
    <replica>01</replica>
</macros>

<networks>
    <ip>::/0</ip>
</networks>

<clickhouse_compression>
        <case>
            <min_part_size>10000000000</min_part_size>
            <min_part_size_ratio>0.01</min_part_size_ratio>
            <method>lz4</method>
        </case>
    </clickhouse_compression>
</yandex>
```
在config.xml文件内添加
```bash
<timezone>Asia/Shanghai</timezone> <!-- 修改时区  -->
include_from>/etc/clickhouse-server/metrika.xml</include_from><!-- 引用配置文件  -->
```

将confix.xml内文件的`<remote_servers > `修改成` <remote_servers incl="clickhouse_remote_servers"> `以便使用 metrika.xml文件内的标签

> If element has 'incl' attribute, then for it's value will be used corresponding substitution from another file.
> By default, path to file with substitutions is /etc/metrika.xml. It could be changed in config in 'include_from' element.
> Values for substitutions are specified in /yandex/name_of_substitution elements in that file


(3)
```bash
# 节点一
docker run --restart always \
-d \
--name clickhouse-server \
--ulimit nofile=262144:262144 \
--volume=/root/meiqia/clickhouse/clickhouse/:/var/lib/clickhouse/ \
--volume=/root/meiqia/clickhouse/clickhouse-server/:/etc/clickhouse-server/ \
--volume=/root/meiqia/clickhouse/log/clickhouse-server/:/var/log/clickhouse-server/  \
-p 9000:9000 \
-p 8123:8123 \
-p 9009:9009 \
--net clickhouse --ip 171.18.0.4 \
yandex/clickhouse-server

# 节点二
docker run --restart always \
-d \
--name clickhouse-server2 \
--ulimit nofile=262144:262144 \
--net clickhouse --ip 171.18.0.7 \
--volume=/root/meiqia/clickhouse/clickhouse2/:/var/lib/clickhouse/ \
--volume=/root/meiqia/clickhouse/clickhouse-server2/:/etc/clickhouse-server/ \
--volume=/root/meiqia/clickhouse/log/clickhouse-server2/:/var/log/clickhouse-server/  \
-p 9001:9000 \
-p 8124:8123 \
-p 9010:9009 \
yandex/clickhouse-server
```



3.验证
```bash
clickhouse-client --host localhost  -u default  --password 
```
分别链接两个节点
```sql
SELECT * FROM system.clusters
```



然后再01中执行创建分布式表 是否可以再02中查询到
```sql
CREATE TABLE IF NOT EXISTS all_hits \
ON CLUSTER clickhouse_cluster_name(p Date, i Int32) \
ENGINE = Distributed(clickhouse_cluster_name, default, hits)
```

>https://blog.csdn.net/appleyuchi/article/details/106854052
>https://blog.csdn.net/qq_42016966/article/details/107687181