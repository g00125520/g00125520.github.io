---
title: distribute locks
description: ways to implment distribute locks 
categories:
  - micro-services
tags:
  - distribute-locks
---

## 基于redisson实现

redisson的实现思路，client加锁时，如果是redis集群，会首先hash到一台机器，然后发送一段lua脚本（保证操作原子性）：

```java
if (redis.call('exists',keys[1]) == 0) then
    redis.call('hset',keys[1],argv[2],1);
    redis.call('pexpire',keys[1],argv[1]);
    return nil;
end;
if(redis.call('hexists',keys[1],argv[2] ==1) then
    redis.call('hincrby',keys[1],argv[2],1);
    redis.call('pexpire',keys[1],argv[1]);
    return nil;
end;
return redis.call('pttl',keys[1]);
```

其中，keys[1]为要加的锁，如'SyncTaskLock'，argv[1]为锁的默认生存时间，默认30秒，argv[2]为加锁的客户端id。

上面的逻辑为，如果锁不存在，则加锁，通过hset lock完成，设置一个hash的数据结构，并设置超时时间。如果另外一个client尝试加锁，发现lock已经存在，然后判断lock的hash中是否存在client2的id不存在，则返回client2一个数字，pttl lock代表lock剩余生存时间。此后，client2进入一个循环，不停尝试加锁。而client1一但加锁成功，则会启动一个watch dog每隔10s检查一下，如果client1还持有锁，则不断延长锁的生存时间。如果client1已经持有锁但又想重复加锁，则进入第二个if，通过hincrby增client1加锁的次数。当释放锁的时候，则每次对加锁的次数减一，如果发现加锁次数为0，则删除lock，然后client2就可以尝试加锁了。

如果对某个redis master写入了锁，此时会异步复制给对应的master slave，但该过程中一旦发生master宕机，主备切换，就会导致client2在新的master上加锁成功client1也加锁成功。

## 基于zk实现

curator基于zk实现，锁是zk上的一个节点，zk的节点分为四种，持久节点，持久顺序节点，临时节点和临时顺序节点。分布式锁用到了临时顺序节点。当a和b对zk发起加锁请求时，两者都在锁节点下创建一个临时顺序节点。假如a先发起，创建完临时节点后，a会检查下lock节点下的所有子节点，如果a是第一个创建节点的，那么a加锁成功。当b来尝试加锁时，也会先创建一个临时顺序节点，b也会检查lock节点下的所有子节点，发现之前a已经创建过了，则加锁失败，之后b会对a创建的节点添加一个监听器，监听这个节点是否被删除。a释放锁的时候，删除自己创建的临时节点，zk通知监听这个节点的监听器，也就是b，b会重新尝试去获取锁。采用临时节点，如果某个客户端创建临时顺序节点之后，不小心宕机，zk会感知到那个客户端宕机，自动删除对应的临时顺序节点。

## ref

- [redisson](https://github.com/redisson/redisson/wiki/8.-Distributed-locks-and-synchronizers)
- [curator](http://curator.apache.org/getting-started.html)
