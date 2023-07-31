---
title: Redis信号量
date: 2019-05-30 15:16:10
tags:
---
信号量可以只使用一个数字来实现，但是这样的信号量没有timeout，如果获得信号量的client出错退出了，那么这个信号量就得不到释放，我们可以利用zset来实现一个支持timeout的信号量。

思路是每次获取信号量的时候，生成一个id，然后把id添加到zset中，value设置为当前的时戳。然后检查id在zset中的rank，如果超过信号量数量limit，那么获取信号量失败，否则获取信号量成功，返回id。

client.js
```
async function acquireSemaphore(name, limit = 5, timeout = 10000) {
  const id = uuidv4(); // 生成一个id
  const now = new Date().getTime(); // 当前的时戳

  const pipeline = redis.pipeline();
  await pipeline.zremrangebyscore(name, '-inf', now - timeout); // 删除已经超时的信号量
  await pipeline.zadd(name, now, id); 
  await pipeline.zrank(name, id);
  const results = await pipeline.exec();
  // 如果新添加的数据的rank没有超过limit，那么获取信号量成功
  if (results[results.length - 1][1] < limit) {
    return id;
  }
  redis.zrem(name, id); // 如果获取信号量失败，那么把id从zset中删除
  return null;
}
```

上面的代码依赖于多个client的时间是同步的，假设有两个client A和B，B的时间落后A 10ms。A和B都尝试获取最后一个信号量，A先获取成功。A在zset中添加了最后一项。
zset
```
1804e2d0-0272-4bf8-97a5-2e30c195b5ff  1559220791205
1804e2d0-0272-4bf8-97a5-2e30c195b5f1  1559220791206
1804e2d0-0272-4bf8-97a5-2e30c195b5f2  1559220791207
1804e2d0-0272-4bf8-97a5-2e30c195b5f3  1559220791208
1804e2d0-0272-4bf8-97a5-2e30c195b5f4  1559220791249
```

接着B尝试获取信号量，因为B的时间比A落后，B写入的值是1559220791239，落后A 10ms。
```
1804e2d0-0272-4bf8-97a5-2e30c195b5f4  1559220791239
```

zset会根据value排序，结果是
```
1804e2d0-0272-4bf8-97a5-2e30c195b5ff  1559220791205
1804e2d0-0272-4bf8-97a5-2e30c195b5f1  1559220791206
1804e2d0-0272-4bf8-97a5-2e30c195b5f2  1559220791207
1804e2d0-0272-4bf8-97a5-2e30c195b5f3  1559220791208
1804e2d0-0272-4bf8-97a5-2e30c195b5f4  1559220791239
1804e2d0-0272-4bf8-97a5-2e30c195b5f4  1559220791249
```

这样相当于B插入在了A的前面。这样B在zset中的rank是4，没有超过信号量的限制。

```
await pipeline.zrank(name, id);
```

这样A和B同时获得了信号量，超过了信号量的限制，这不是我们想要的结果。

如果在zset中保存时戳，会有不同机器的时间不同步的问题。如果在zset中存储一个递增的id，这样不同的client在zset中的排序就不会受机器时戳的影响了。

```
async function acquireSemaphore(name, limit = 5, timeout = 10000) {
  const id = uuidv4();
  const now = new Date().getTime();
  const timeKey = name + ':time';
  const countKey = name + ':count'; // 这里增加了一个递增的id

  let pipeline = redis.pipeline();
  await pipeline.zremrangebyscore(timeKey, '-inf', now - timeout);
  await pipeline.zinterstore(
    countKey, 
    2, 
    countKey,
    timeKey, 
    'weights',
    1,
    0
  ); // 求timeKey和countKey的交集，过滤掉timeout的信号量

  await pipeline.incr('semaphore:count');
  let results = await pipeline.exec();
  const count = results[results.length-1][1];

  pipeline = redis.pipeline();
  await pipeline.zadd(timeKey, now, id);
  await pipeline.zadd(countKey, count, id);
  await pipeline.zrank(countKey, id); // 后获取信号量的client的rank更高

  let results2 = await pipeline.exec();
  if (results2[results2.length - 1][1] < limit) {
    return id;
  }

  redis.zrem(timeKey, id);
  redis.zrem(countKey, id);
  return null;
}
```