---
author : "jihongwei"
tags : ["分布式"]
date : 2022-05-13T11:24:00Z
title : "几种常见的分布式锁"
---

业界流行的分布式锁实现，一般有这3种方式：

* 基于数据库实现的分布式锁
* 基于Redis实现的分布式锁
* 基于Zookeeper实现的分布式锁


### 基于数据库实现的分布式锁

#### 悲观锁

借助`select for update`(for update仅适用于InnoDB，且必须在事务块(BEGIN/COMMIT)中才能生效。在进行事务操作时，通过“for update”语句，MySQL会对查询结果集中每行数据都添加排他锁，其他线程对该记录的更新与删除操作都会阻塞。排他锁包含行锁、表锁。)

```
begin;
select * from goods where id = 1 for update;

// do some tings

update goods set stock = stock - 1 where id = 1;
commit;
```

####  乐观锁
> 搞个version字段，每次更新修改，都会自增加一，然后去更新余额时，把查出来的那个版本号，带上条件去更新，如果是上次那个版本号，就更新，如果不是，表示别人并发修改过了，就继续重试。

###  基于redis实现分布式锁

* setnx + expire
>加锁操作和设置超时时间是分开的。假设在执行完setnx加锁后，正要执行expire设置过期时间时，进程crash掉或者要重启维护了，那这个锁就长生不老了，别的线程永远获取不到锁啦，所以分布式锁不能这么实现！

* setnx + value值是过期时间
```
long expires = System.currentTimeMillis() + expireTime; //系统时间+设置的过期时间
String expiresStr = String.valueOf(expires);

// 如果当前锁不存在，返回加锁成功
if (jedis.setnx(key, expiresStr) == 1) {
        return true;
} 
// 如果锁已经存在，获取锁的过期时间
String currentValueStr = jedis.get(key);

// 如果获取到的过期时间，小于系统当前时间，表示已经过期
if (currentValueStr != null && Long.parseLong(currentValueStr) < System.currentTimeMillis()) {

     // 锁已过期，获取上一个锁的过期时间，并设置现在锁的过期时间（不了解redis的getSet命令的小伙伴，可以去官网看下哈）
    String oldValueStr = jedis.getSet(key, expiresStr);
    
    if (oldValueStr != null && oldValueStr.equals(currentValueStr)) {
         // 考虑多线程并发的情况，只有一个线程的设置值和当前值相同，它才可以加锁
         return true;
    }
}
        
//其他情况，均返回加锁失败
return false;
}
```
> 1) 过期时间是客户端自己生成的，分布式环境下，每个客户端的时间必须同步。
> 2) 没有保存持有者的唯一标识，可能被别的客户端释放/解锁。
> 3) 锁过期的时候，并发多个客户端同时请求过来，都执行了jedis.getSet()，最终只能有一个客户端加锁成功，但是该客户端锁的过期时间，可能被别的客户端覆盖

* set的扩展命令（set ex px nx）
```
SET key value [EX seconds] [PX milliseconds] [NX|XX]
```
> 1) 锁过期释放了，业务还没执行完。
> 2) 锁被别的线程误删。
* set ex px nx + 校验唯一随机值,再删除
```
if（jedis.set(key, uni_request_id, "NX", "EX", 100s) == 1）{ //加锁
    try {
        do something  //业务处理
    }catch(){
  }
  finally {
       //判断是不是当前线程加的锁,是才释放
       if (uni_request_id.equals(jedis.get(key))) {
          jedis.del(key); //释放锁
        }
    }
}
```
> 锁过期释放了，业务还没执行完的问题。

* Redisson

> 只要线程一加锁成功，就会启动一个watch dog看门狗，它是一个后台线程，会每隔10秒检查一下，如果线程1还持有锁，那么就会不断的延长锁key的生存时间。因此，Redisson就是使用watch dog解决了锁过期释放，业务没执行完问题(总的来说就是如果锁被使用的话就对锁续期)

* Redisson + RedLock

>  针对redis的一种集群做出的优化。
>按顺序向5个master节点请求加锁，
根据设置的超时时间来判断，是不是要跳过该master节点。
如果大于等于3个节点加锁成功，并且使用的时间小于锁的有效期，即可认定加锁成功啦。
如果获取锁失败，解锁！

###  Zookeeper分布式锁


>https://mp.weixin.qq.com/s/ZeEMVbzwincbhECb1jjNCQ