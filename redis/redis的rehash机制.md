在《Redis的内存结构》的内存结构中，我们设计到了redis的运行内存结构，为了方便本文的介绍，我再次把涉及到的数据结构都列出来
```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dictType {
    unsigned int (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;

/* If safe is set to 1 this is a safe iterator, that means, you can call
 * dictAdd, dictFind, and other functions against the dictionary even while
 * iterating. Otherwise it is a non safe iterator, and only dictNext()
 * should be called while iterating. */
typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator;
```
其中我们关注的是```dict```这个类型对象。
在```dict```中， 主要有两块数据存储区域```ht[0]```以及  ```ht[1]```。其中字段```used```表示此表的饱满度，也就是已存储的KV总数。那么问题来了，程序子啊找一个key的时候，是怎么知道先去哪一个dict中找呢？如果找不到，又是如何在另外一个dict中找的呢？
##rehash算法##
我们不妨从最简单的寻找key的函数中发现逻辑
```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    unsigned int h, idx, table;

    if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```
也就是说，程序永远先找```ht[0]```，如果找不到，再去找```ht[1]```（就是语句``` for (table = 0; table <= 1; table++)```）。
除了上述外，我们发现，在具体的某一个ht中，寻找key所在table中的index时，都是用的同一个逻辑
```c
idx = h & d->ht[table].sizemask;
```
很有意思，非常有意思。
也就是说，idx在两个ht中的位置逻辑都是差不多——和sizemask位乘后的结果。换言之，***redis在rehash迁移kv的时候，新的位置和sizemask以及key本身有关***。
那我们看一下sizemask是怎么来的
```c
/* Expand or create the hash table */
int dictExpand(dict *d, unsigned long size)
{
    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);

    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```
也就是说,sizemask就是realsize-1得来的，那么realsize是怎么来的？我们继续看代码
```c
/* Expand the hash table if needed */
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```
其中我们看到两句关键的话
```c
 dictExpand(d, DICT_HT_INITIAL_SIZE);
 dictExpand(d, d->ht[0].used*2);
 ```
也就是说，ht的大小初始值为```DICT_HT_INITIAL_SIZE=4```，然后扩展的时候以2的倍数增长bucket size。换句话：ht的大小永远是2的幂次方，而sizemask则是bit值全为1的正整数。我们于是得出结论:***redis的两个ht大小成2倍关系***。
在上述两个结论下，我们不难设想redis的rehash机制了。
举个例子先
比如两个ht的大小是4和8，那么对于ht[0]中table[1]中的值，如果rehash的话，那么此中的kv要么被迁移到ht[1]中的table[1],或者table[5]。
为什么呢？
因为在ht[0]中table[1]的kv，它的hash值要么是4(2n-1)+1,要么是4*2n +1。而前者在ht[1]中肯定是在table[5]中，后者落入table[1]中。例子清晰后，我们再理解源码就好多了
```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            unsigned int h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
其中我们前文所举例子反应的代买模块是从注解```/* Move all the keys in this bucket from the old to the new hash HT */```中开始。
不过，如果redis的rehash仅仅是这么简单，就没啥意思了，redis的rehash最精彩的,其实是***rehash中继***
##rehash中继##
还记得在《redis的内存结构》文中我强调的那句话吗？
**对性能的压榨，贯穿redis的整个设计思路**
那么问题来了，redis的应用场景是GB级别的缓存服务，那么，rehash一个几百MB大小的ht是一件很正常的事情，而redis又是单线程程序，如果每次rehash都要占用很长时间，那么外部的访问便会被block，从而导致服务长时间不可用。这绝对是不能接受的。于是，rehash必须被切割成一个个很小的任务，每次有时间就做一些，下次能接着做。
任务划分比较容易，但是任务中继却带来了一系列问题:
1. 如何保留中继现场
2. 如何查找？key被迁移会不会造成找不到？
3. 迁移过的bucket，如果又插入数据，该如何处理
4. 任务划分如果按照bucket中的slot为最小粒度，那么不同slot的key容量不一样，key类型不一样，会不会造成时间不一致，会不会造成时间粒度跨度太大？
我们在前文看到了，ht在迁移的时候，是按照slot的index从小到大开始迁移的。而这个值，是存在```rehashidx```中的，那么第一个问题的答案就是: 中继现场就是rehashindex这个值。凡是index小于等于 rehashindex的slot，肯定在ht[1]中了。
那么第二个问题也有答案了：如果发现正在rehash,key在sizemask运算后的结果小于等于rehashindex，则直接去ht[1]中找(当然 slot的index需要用ht[1]的sizemask重新运算得到)，反之则去ht[0]中找。
其实第三个问题答案也出来了：根据slot的index和rahashindex的比较结果，决定插入哪个ht中。
第四个问题比较烦。
首先要明确，rehash的时候迁移的是bucket，也就是说:**rehash不迁移value**。也就是说，value在被分配的一刹那，其开始地址就不会变了，除非你对它有写操作(曾改删)。也就是说——```dictEntry```这个类型对象的内存地址是不会被迁移的，那么就简单了，kv迁移的过程，其实都不涉及到内存分配，只是一个个赋值过程。那么对于slot中的链表长度，只要控制好不要过长就行(1和1000个赋值语句所消耗的时间其实没那么大)，而这其实是hash的饱满度控制所考虑的。
带着这些结论，我们看下源码
```c
int dictRehashMilliseconds(dict *d, int ms) {
    long long start = timeInMilliseconds();
    int rehashes = 0;

    while(dictRehash(d,100)) {
        rehashes += 100;
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
}
```
也就是说，redis在实际rehash的时候更激进，是按照100个slot为基本粒度来划分任务的。redis的作者还发誓，无论如何，100个slot的迁移耗时，永远在1ms内。
我只能说，请收下我的膝盖。
从整个过程来看，其实redis无论是rehash算法以及rehash中继，其核心就是——redis的ht大小是2的幂次方。这个设计直接让rehash方便了太多。
说句题外话，现实中mysql的resharding,其实是可以考虑这种思想的，尤其是如果mysql被装入了容器中，主键数被集中放置，这时候动态扩容，是可以借鉴redis的思想的。