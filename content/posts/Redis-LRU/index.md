+++
title = "Redis LRU"
date = "2020-12-25T14:26:24+08:00"
draft = false
description = ""
[taxonomies]
tags = ["Data Structure", "Redis", "Database"]
+++

## 基础知识

`LRU (Least recently used)` 是一个非常常用的缓存置换算法.

在缓存空间有限的情况下, 在新的数据写入时, 需要淘汰一些旧数据. 一般期望最不可能被继续访问的数据淘汰. 由于无法对未来的情况进行预测, 只能基于现有的信息推测.

`LRU` 基于这样一个假定, 在一个时间点上, 如果一个数据距离上次被访问的时间越长, 则这个数据在未来被访问的可能性越小. 所以, 应该淘汰掉距离上次被访问的时间最久的数据.

常见的 `LRU` 实现是使用一个双向链表, 当 Item 被访问时, 将其移动到链表的头部. 当缓存不足时, 从链表的尾部开始淘汰.

单纯的双向链表的实现实际上是不太符合现实的. 一般的, 我们希望缓存可以尽快被检索到, 而在双向链表中检索 Item 的效率是 O(n), 特别是检索的 Item 在链表中不存在时, 效率稳定的是 O(n). 这样, 在缓存系统中维护这个双向链表的成本是非常高的. 一般的, 会将 `哈希表` 或者 `二叉搜索树` 和双向链表组合起来, 在更高效率的数据结构中记录 `Key`, 并在 `Key` 中记录 Item 在双向链表中的指针.

另外的, 在并发量较大的时候, 双向链表中的操作需要加锁, 否则链表很容易出问题.

## 具体实现

### Redis 2.8

在较早版本的 `Redis` 上, 并没有实现 `LRU`. 在后续 2.8 添加的时候, 并没有使用常见的双向链表的方式来实现 `LRU`, 而使用了一个近似的实现.

从空间和时间上考虑, `Redis` 中的 `Key` 的数量可能非常的多, 双向链表可能会非常大, 占用内存非常多; 另一方面, `Redis` 中的操作可能也非常频繁, 每一次访问都需要操作一次双向链表, 在时间上也显得非常不划算.

`Antirez` 在 `Redis Object` 中挤出了 24 个位元 bits, 并用其存储按秒计算的 `unix timestamp` 的低 24 位. 这个被称为 `LRU clock`.

```C
/* https://github.com/redis/redis/blob/3.0/src/redis.h#L420 */

/* A redis object, that is a type able to hold a string / list / set */

/* The actual Redis Object */
#define REDIS_LRU_BITS 24
#define REDIS_LRU_CLOCK_MAX ((1<<REDIS_LRU_BITS)-1) /* Max value of obj->lru */
#define REDIS_LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;
```

24 个 bits 明显不够存储一个完整的时间戳, 当第二十四位向前进位的时候, 就会发生溢出. 此时, 最近被访问的 Item 的 `LRU clock` 反而较小, 更容易被淘汰. 考虑到这个溢出需要 194 天, 而 `Redis` 中的操作应该比较频繁, 所以 `antirez` 认为这个问题可以接受.

理论上, 可以精心构造一些数据, 让 `Redis` 的 `LRU` 失效. 比如, 总是在溢出前访问一个 Item, 这个 Item 的 `LRU clock` 总是很大, 虽然这个 Item 的访问周期总是 194 天才访问一次, 但是它总不会被淘汰.

```C
/* https://github.com/redis/redis/blob/3.0/src/db.c#L43 */

robj *lookupKey(redisDb *db, robj *key) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
            val->lru = LRU_CLOCK();
        return val;
    } else {
        return NULL;
    }
}
```

接下来的问题在于如何找到最久没有被访问的 Item. 如果一定要找到最久没有被访问到的 Item, 那么需要遍历所有的 `Key`, 而且在遍历的过程中, 要么禁止在这期间做任何的访问操作, 要么可能出现找到的 `Key` 恰好又刚刚被访问到的问题.

`Antirez` 在这里又使用了一个近似的实现, 随机选取 3 个 `Key`, 把他们之中最久没有被访问到的淘汰. 随后, 这个数值变成了可配置项 `maxmemory-samples` , 默认值是 5. 考虑到选出的结果不一定是最好的, 但是很大可能不是一个坏的结果, 即选出一个非常近被访问的 Item, 这个实现还算可以接受.

```C
/* https://github.com/redis/redis/blob/2.8/src/redis.c#L2982 */

/* volatile-lru and allkeys-lru policy */
else if (server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_LRU ||
    server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_LRU)
{
    for (k = 0; k < server.maxmemory_samples; k++) {
        sds thiskey;
        long thisval;
        robj *o;

        de = dictGetRandomKey(dict);
        thiskey = dictGetKey(de);
        /* When policy is volatile-lru we need an additional lookup
          * to locate the real key, as dict is set to db->expires. */
        if (server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_LRU)
            de = dictFind(db->dict, thiskey);
        o = dictGetVal(de);
        thisval = estimateObjectIdleTime(o);

        /* Higher idle time is better candidate for deletion */
        if (bestkey == NULL || thisval > bestval) {
            bestkey = thiskey;
            bestval = thisval;
        }
    }
}
```

### Redis 3.0

`Antirez` 在 3.0 版本中进一步提升了近似算法的准确性. 一个显而易见的方法是, 通过过去累计的信息来提升准确性.

`Redis` 中维护了一个默认大小为 16 的 `pool`, 里面存储了备选的 `Key`. 当需要淘汰时, 从随机选择的 N 个 `Key` 中与 `pool` 中的 `Key` 做对比, 在 `pool` 中维护其中最久没有被访问到的 16 个 `Key`, 然后在 `pool` 中淘汰其中最久没有被访问到的 Item. 这个 `pool` 非常类似于小顶堆.

```C
/* https://github.com/redis/redis/blob/3.0/src/redis.c#L3275 */

/* volatile-lru and allkeys-lru policy */
else if (server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_LRU ||
    server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_LRU)
{
    struct evictionPoolEntry *pool = db->eviction_pool;

    while(bestkey == NULL) {
        evictionPoolPopulate(dict, db->dict, db->eviction_pool);
        /* Go backward from best to worst element to evict. */
        for (k = REDIS_EVICTION_POOL_SIZE-1; k >= 0; k--) {
            if (pool[k].key == NULL) continue;
            de = dictFind(dict,pool[k].key);

            /* Remove the entry from the pool. */
            sdsfree(pool[k].key);
            /* Shift all elements on its right to left. */
            memmove(pool+k,pool+k+1,
                sizeof(pool[0])*(REDIS_EVICTION_POOL_SIZE-k-1));
            /* Clear the element on the right which is empty
              * since we shifted one position to the left.  */
            pool[REDIS_EVICTION_POOL_SIZE-1].key = NULL;
            pool[REDIS_EVICTION_POOL_SIZE-1].idle = 0;

            /* If the key exists, is our pick. Otherwise it is
              * a ghost and we need to try the next element. */
            if (de) {
                bestkey = dictGetKey(de);
                break;
            } else {
                /* Ghost... */
                continue;
            }
        }
    }
}
```

`pool` 中排序操作的核心代码在 `evictionPoolPopulate` 函数中

```C

/* https://github.com/redis/redis/blob/3.0/src/redis.c#L3145 */

#define EVICTION_SAMPLES_ARRAY_SIZE 16
void evictionPoolPopulate(dict *sampledict, dict *keydict, struct evictionPoolEntry *pool) {
    int j, k, count;
    dictEntry *_samples[EVICTION_SAMPLES_ARRAY_SIZE];
    dictEntry **samples;

    /* Try to use a static buffer: this function is a big hit...
     * Note: it was actually measured that this helps. */
    if (server.maxmemory_samples <= EVICTION_SAMPLES_ARRAY_SIZE) {
        samples = _samples;
    } else {
        samples = zmalloc(sizeof(samples[0])*server.maxmemory_samples);
    }

    count = dictGetSomeKeys(sampledict,samples,server.maxmemory_samples);
    for (j = 0; j < count; j++) {
        unsigned long long idle;
        sds key;
        robj *o;
        dictEntry *de;

        de = samples[j];
        key = dictGetKey(de);
        /* If the dictionary we are sampling from is not the main
         * dictionary (but the expires one) we need to lookup the key
         * again in the key dictionary to obtain the value object. */
        if (sampledict != keydict) de = dictFind(keydict, key);
        o = dictGetVal(de);
        idle = estimateObjectIdleTime(o);

        /* Insert the element inside the pool.
         * First, find the first empty bucket or the first populated
         * bucket that has an idle time smaller than our idle time. */
        k = 0;
        while (k < REDIS_EVICTION_POOL_SIZE &&
               pool[k].key &&
               pool[k].idle < idle) k++;
        if (k == 0 && pool[REDIS_EVICTION_POOL_SIZE-1].key != NULL) {
            /* Can't insert if the element is < the worst element we have
             * and there are no empty buckets. */
            continue;
        } else if (k < REDIS_EVICTION_POOL_SIZE && pool[k].key == NULL) {
            /* Inserting into empty position. No setup needed before insert. */
        } else {
            /* Inserting in the middle. Now k points to the first element
             * greater than the element to insert.  */
            if (pool[REDIS_EVICTION_POOL_SIZE-1].key == NULL) {
                /* Free space on the right? Insert at k shifting
                 * all the elements from k to end to the right. */
                memmove(pool+k+1,pool+k,
                    sizeof(pool[0])*(REDIS_EVICTION_POOL_SIZE-k-1));
            } else {
                /* No free space on right? Insert at k-1 */
                k--;
                /* Shift all elements on the left of k (included) to the
                 * left, so we discard the element with smaller idle time. */
                sdsfree(pool[0].key);
                memmove(pool,pool+1,sizeof(pool[0])*k);
            }
        }
        pool[k].key = sdsdup(key);
        pool[k].idle = idle;
    }
    if (samples != _samples) zfree(samples);
}
```

## 参考资料

1. [Random notes on improving the Redis LRU algorithm](http://antirez.com/news/109)
2. [Using Redis as an LRU cache](https://redis.io/topics/lru-cache)
