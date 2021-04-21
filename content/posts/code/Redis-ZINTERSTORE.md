---
title: Redis ZINTERSTORE
description:
toc: true
authors: []
tags: []
categories: []
series: []
date: 2021-04-21T20:30:43+08:00
lastmod: 2021-04-21T20:30:43+08:00
featuredVideo:
featuredImage:
draft: false
---

## 结构推测
今天与人争执的时候觉得应该探究一下 `Redis` 中 `ZINTERSTORE` 的实现. 先看文档中的描述

> ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]

> Available since 2.0.0.
> 
> Time complexity: O(N*K)+O(M*log(M)) worst case with N being the smallest input sorted set, K being the number of input sorted sets and M being the number of elements in the resulting sorted set.

大致意思就是最坏情况下时间复杂度是 `O(N*K)+O(M*log(M))`, 其中 `N` 是输入中元素最少的 `sorted set` 的元素数目, `K` 是输入的 `sorted set` 的数量, `M` 是结果中的 `sorted set` 中元素的数目.

其实光从时间复杂度就能推测出来大致的操作了. `O(M*log(M))` 这个复杂度很有可能是一次排序操作, 而且和结果中的元素数目相关, 那么很有可能是先取交集之后得出 `M` 个元素, 再在 `M` 个元素中进行排序的. 当然, 也有可能是相应的, 在有序的序列中进行插入的操作.

而 `N` 是最小的 `sorted set` 的元素个数. 这个一开始我有点想不明白, 难道不应该是最大的才对吗, 因为这里我还是想着去遍历其他的 `sorted set` 进行比较. 但是思考一下, `sorted set` 也是 `set` 嘛, 完全可以使用 `O(1)` 的效率在其中进行查找. 这样实际就变成了分别在 (K - 1) 个 `sorted set` 中寻找 `N` 个元素, 这样自然就是 `O(N*K)` 了.

## 源码分析

以下源码基于 `Redis 3.0` 分析, 实际的函数操作为

```C
/* https://github.com/redis/redis/blob/3.0/src/t_zset.c#L1905 */
void zunionInterGenericCommand(redisClient *c, robj *dstkey, int op)
```

这里我们可以看到, 在 `Redis` 的源码中将 `ZINTERSTORE` 和 `ZUNIONSTORE` 放在了一起处理, 使用 `op` 做区分.

随后是一段漫长的代码, 主要是处理输入参数的, 比如要操作的 `sorted set` 的列表, 存到了 `src` 中; 然后处理 `WEIGHTS` 和 `AGGREGATE`. 代码比较长而且逻辑也很简单.

```C
/* https://github.com/redis/redis/blob/3.0/src/t_zset.c#L1993 */

    /* sort sets from the smallest to largest, this will improve our
     * algorithm's performance */
    qsort(src,setnum,sizeof(zsetopsrc),zuiCompareByCardinality);
```

然后这里把所有 `sorted set` 按元素的个数从小到大排列, 以提高效率. 随后是具体的取交集的代码.

```C
/* https://github.com/redis/redis/blob/3.0/src/t_zset.c#L2001 */

    if (op == REDIS_OP_INTER) {
        /* Skip everything if the smallest input is empty. */
        if (zuiLength(&src[0]) > 0) {
            /* Precondition: as src[0] is non-empty and the inputs are ordered
             * by size, all src[i > 0] are non-empty too. */
            zuiInitIterator(&src[0]);
            while (zuiNext(&src[0],&zval)) {
                double score, value;

                score = src[0].weight * zval.score;
                if (isnan(score)) score = 0;

                for (j = 1; j < setnum; j++) {
                    /* It is not safe to access the zset we are
                     * iterating, so explicitly check for equal object. */
                    if (src[j].subject == src[0].subject) {
                        value = zval.score*src[j].weight;
                        zunionInterAggregate(&score,value,aggregate);
                    } else if (zuiFind(&src[j],&zval,&value)) {
                        value *= src[j].weight;
                        zunionInterAggregate(&score,value,aggregate);
                    } else {
                        break;
                    }
                }

                /* Only continue when present in every input. */
                if (j == setnum) {
                    tmp = zuiObjectFromValue(&zval);
                    znode = zslInsert(dstzset->zsl,score,tmp);
                    incrRefCount(tmp); /* added to skiplist */
                    dictAdd(dstzset->dict,tmp,&znode->score);
                    incrRefCount(tmp); /* added to dictionary */

                    if (sdsEncodedObject(tmp)) {
                        if (sdslen(tmp->ptr) > maxelelen)
                            maxelelen = sdslen(tmp->ptr);
                    }
                }
            }
            zuiClearIterator(&src[0]);
        }
    }

```

当然, 如果按元素数从小到大排序的第一个 `sorted set` 元素数为 0, 那就可以直接返回了, 随后在此 `sorted set` 上建立一个 `Iterator`, 主要就是帮助在 `sorted set` 上做遍历的, 因为在 `sorted set` 中, 遍历不是一个常用操作, 排序才是.

然后就是通过 `while (zuiNext(&src[0],&zval))` 取出第一个 `sorted set` 中的每一个元素, 再用 `for (j = 1; j < setnum; j++)` 将其在每一个其他的 `sorted set` 中查找一遍.

当两个 `sorted set` 指向同一个对象是, 那么毫无疑问一定会有同样的元素. 否则就在 `src[j]` 中查找当前元素. `zuiFind(&src[j],&zval,&value)` 实现了这个操作. 具体的代码在后面分析. 如果找不到的话, 则没有必要继续找下去了, 跳出即可.

查找的过程中也使用了 `zunionInterAggregate(&score,value,aggregate)` 来更新当前元素的 `score`, 具体的 `score` 更新规则是根据 `AGGREGATE` 参数来定的.

当元素在所有的 `sorted set` 中时, 就可以把这个元素添加进结果的 `sorted set` 中了. 这里就是根据 `zval` 和 `score` 往 `dstzset` 里面插入元素. 因为 `sorted set` 里面既有 `dict` 也有 `skiplist`, 所以两个都要添加.

这里还有一个操作就是更新最大元素的长度, 这个和 `sorted set` 的内部优化有关.

```C
/* https://github.com/redis/redis/blob/3.0/src/t_zset.c#L2119 */

    if (dbDelete(c->db,dstkey)) {
        signalModifiedKey(c->db,dstkey);
        touched = 1;
        server.dirty++;
    }
    if (dstzset->zsl->length) {
        /* Convert to ziplist when in limits. */
        if (dstzset->zsl->length <= server.zset_max_ziplist_entries &&
            maxelelen <= server.zset_max_ziplist_value)
                zsetConvert(dstobj,REDIS_ENCODING_ZIPLIST);

        dbAdd(c->db,dstkey,dstobj);
        addReplyLongLong(c,zsetLength(dstobj));
        if (!touched) signalModifiedKey(c->db,dstkey);
        notifyKeyspaceEvent(REDIS_NOTIFY_ZSET,
            (op == REDIS_OP_UNION) ? "zunionstore" : "zinterstore",
            dstkey,c->db->id);
        server.dirty++;
    } else {
        decrRefCount(dstobj);
        addReply(c,shared.czero);
        if (touched)
            notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,"del",dstkey,c->db->id);
    }
    zfree(src);
```

随后就是收尾工作. 如果目标的坑上已经有值了, 就毫不犹豫的干掉它. 然后再看作为结果的 `sorted set`. 如果它满足转化为 `ziplist` 的条件, 就可以把它转化为 `ziplist`. 后面是一些 `hook` 的通知, 可以暂时忽略. 如果结果为空, `Integer reply` 会返回 0, 否则会返回结果中元素的个数.

至此大致流程已经结束了. 如同推测的那样, 算法复杂度完美的反映了操作的内部过程和数据结构. 不过, 我们还有几个问题没有解决. `zuiFind` 中的具体操作是什么, `ziplist` 又是什么.

## 搁这搁这

这个标题充分提现了递归的思想. 学习新东西, 然后发现另一些新东西, 然后再学习这些新东西, 然后......

首先来看 `zuiFind`

```C
/* https://github.com/redis/redis/blob/3.0/src/t_zset.c#L1826 */

int zuiFind(zsetopsrc *op, zsetopval *val, double *score) {
    if (op->subject == NULL)
        return 0;

    if (op->type == REDIS_SET) {
        if (op->encoding == REDIS_ENCODING_INTSET) {
            if (zuiLongLongFromValue(val) &&
                intsetFind(op->subject->ptr,val->ell))
            {
                *score = 1.0;
                return 1;
            } else {
                return 0;
            }
        } else if (op->encoding == REDIS_ENCODING_HT) {
            dict *ht = op->subject->ptr;
            zuiObjectFromValue(val);
            if (dictFind(ht,val->ele) != NULL) {
                *score = 1.0;
                return 1;
            } else {
                return 0;
            }
        } else {
            redisPanic("Unknown set encoding");
        }
    } else if (op->type == REDIS_ZSET) {
        zuiObjectFromValue(val);

        if (op->encoding == REDIS_ENCODING_ZIPLIST) {
            if (zzlFind(op->subject->ptr,val->ele,score) != NULL) {
                /* Score is already set by zzlFind. */
                return 1;
            } else {
                return 0;
            }
        } else if (op->encoding == REDIS_ENCODING_SKIPLIST) {
            zset *zs = op->subject->ptr;
            dictEntry *de;
            if ((de = dictFind(zs->dict,val->ele)) != NULL) {
                *score = *(double*)dictGetVal(de);
                return 1;
            } else {
                return 0;
            }
        } else {
            redisPanic("Unknown sorted set encoding");
        }
    } else {
        redisPanic("Unsupported type");
    }
}
```

先忽略掉 `set` 的操作, 来看 `sorted set`. 我们又见到了收尾工作时出现过的 `ziplist`.

> The ziplist is a specially encoded dually linked list that is designed to be very memory efficient. It stores both strings and integer values, where integers are encoded as actual integers instead of a series of characters. It allows push and pop operations on either side of the list in O(1) time. However, because every operation requires a reallocation of the memory used by the ziplist, the actual complexity is related to the amount of memory used by the ziplist.

查了一下资料, 这是 `Redis` 在面对小元素时可以做的一个内存优化, 本体是一个经过特殊编码的双向链表. 经过特殊编码后的数据会变得更加紧凑, 连续的内存使用也对于缓存更加友好. 有两个配置决定了 `sorted set` 中使用 `ziplist` 的阈值.

```C
#define REDIS_ZSET_MAX_ZIPLIST_ENTRIES 128
#define REDIS_ZSET_MAX_ZIPLIST_VALUE 64
```

![有序集合](/images/Redis-ZINTERSTORE/zset.svg)
图. 有序集合

然后我们尴尬的发现, 在 `ziplist` 查找一个元素实际上是一个遍历, 时间复杂度为 `O(N)`, 如下

```C
/* https://github.com/redis/redis/blob/3.0/src/t_zset.c#L915 */

unsigned char *zzlFind(unsigned char *zl, robj *ele, double *score) {
    unsigned char *eptr = ziplistIndex(zl,0), *sptr;

    ele = getDecodedObject(ele);
    while (eptr != NULL) {
        sptr = ziplistNext(zl,eptr);
        redisAssertWithInfo(NULL,ele,sptr != NULL);

        if (ziplistCompare(eptr,ele->ptr,sdslen(ele->ptr))) {
            /* Matching element, pull out score. */
            if (score != NULL) *score = zzlGetScore(sptr);
            decrRefCount(ele);
            return eptr;
        }

        /* Move to next element. */
        eptr = ziplistNext(zl,sptr);
    }

    decrRefCount(ele);
    return NULL;
}
```

不过当 `zset` 使用 `REDIS_ENCODING_SKIPLIST` 作为 encoding 的时候, 使用 `Hash table` 做查询的时间复杂度是 `O(1)` 这是肯定的.

当然, 这里面其实还有很多细节没有说到, 比如 `ziplist` 的内部表示, 和 `zset` 混在一起的 `set`, `zset` 中的 `dict` 和 `skiplist` 的详细分析等等. 下次有机会的吧.

## 参考资料

1. [The ziplist representation](https://redislabs.com/ebook/part-2-core-concepts/01chapter-9-reducing-memory-use/9-1-short-structures/9-1-1-the-ziplist-representation/)
2. [压缩列表 — Redis 设计与实现](https://redisbook.readthedocs.io/en/latest/compress-datastruct/ziplist.html)
