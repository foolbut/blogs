*本文中分析的源码来源于github的[redis项目](https://github.com/antirez/redis)。其中代码部分有出入可能是因为本文书写后，redis项目不断更新造成的。*
Redis作为一个内存级KV存储(我一直不认为它是一个数据库)，采用单线程的服务模式，其内存结构直接决定了本身的服务性能。本文则在于解析Redis的具体内存结构。
##内存结构-数据类型##
###redis object###
广泛而言，redis的所有数据存储对象(不仅仅是用户所能用到的数据)，都叫做redis object。具体的定义就是[server.h](https://github.com/antirez/redis/blob/unstable/src/server.h)中的```robj```。我们看下具体的结构代码:
```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits decreas time). */
    int refcount;
    void *ptr;
} robj;
```
首先看```type```字段。robj的type指的是实际的redis对象结构，也就是redis官方文档支持的数据类型
```c
#define OBJ_STRING 0
#define OBJ_LIST 1
#define OBJ_SET 2
#define OBJ_ZSET 3
#define OBJ_HASH 4
```
其次看```encoding```。这里的encoding指的是：内存的encoding。也就是指针```ptr```指向的内存区域应该如何被解释成哪种对象。具体的encoding值有以下几种:
```c
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
```
具体含义参考代码中的注释。其中```OBJ_ENCODING_EMBSTR```就是redis的KV结构中K的类型，它有一个别名:sds。
可以看到,encoding和type存在一定的含义重复。
而字段```refcount```则不用解释。之前说过，robj被使用的场景，不仅仅是用户用到的数据对象，redis运行中的内存对象，也是robj类型。当然，看到refcount，redis的内存回收算法也呼之欲出了。而经典的计数回收算法带来的问题(引用循环导致内存泄漏，内存堆碎片化等),留给读者自己发现。
###string###
string在redis的数据结构中最为普遍，相对于JAVA的String类型，redis采用了c语言中最基本的结构:char[]。由于这个原因，string是没有charset或者encoding这个概念的。这也直接导致了redis整个存储服务并没有字符编码的概念，只有字节串，至于这些字节串到底该如何编码，由连接redis的客户端来解释。
string在redis整个工作机制中，最大的作用是命令。为了方便命令的解析，redis作者对string进行了一定的函数封装，具体代码可以参见[t_string.c](https://github.com/antirez/redis/blob/unstable/src/t_string.c).
值得注意的是，所以命令的解析，第一个参数是```client```类型的变量。也就是说。命令解析的时候，已经被tokenized。换句话说t_string负责的是command parse。client对象生成的过程则是command lexical analyze，具体参见[server.h](https://github.com/antirez/redis/blob/unstable/src/server.h)。具体实现细节在此不讨论。
###list###
我们先看下list是怎么创建的
```c
quicklist *quicklistCreate(void) {
    struct quicklist *quicklist;

    quicklist = zmalloc(sizeof(*quicklist));
    quicklist->head = quicklist->tail = NULL;
    quicklist->len = 0;
    quicklist->count = 0;
    quicklist->compress = 0;
    quicklist->fill = -2;
    return quicklist;
}
robj *createQuicklistObject(void) {
    quicklist *l = quicklistCreate();
    robj *o = createObject(OBJ_LIST,l);
    o->encoding = OBJ_ENCODING_QUICKLIST;
    return o;
}
```
函数实现没有什么问题。redis号称list是非固定长度的，那么显然，底层实现应该是链表，看下源码
```c
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned int len;           /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```
看到这里，已经显然不用看```quicklistNode```的结构了，因为这是很典型的一个链表结构。
重点在于```compress```参数。准确说,compress应该叫做needCompress。为什么呢，因为这个参数是指亟待压缩的节点个数（当为1的时候表明所有节点都没有压缩并且不需要压缩）。那么为什么会有这个参数呢？关键在于redis本身对于性能的追求。
**对性能的压榨，贯穿redis的整个设计思路**。
在这个大前提下，压缩一个长度不确定的数组，一口气完成是不现实的。那么***将长任务分割成几步来完成***就变成了redis的一个很重要的选项。
OK，那么问题来了，如果15个元素，压缩了13个，中间又插/改了一个，该怎么办呢？下次压缩节点时，怎么知道哪个节点在中间没有压缩呢？
解决方案很简单：list只允许pop,push,shift,unshift，不能随意插入。所以说，其实list只是一个queue和stack的混合体。
###hash###
hash结构作为经典的数据结构，其实绝大多数的实现是双表：单维固长数组（俗称key bucket）以及二维数组(俗称 value bucket)。其中，第二维是不限长度的链表，链表内每个节点由hash值,值,next指针 组成。那么redis的hash是不是这么做的呢？看[源码](https://github.com/antirez/redis/blob/unstable/src/dict.h)
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
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```
其中```dictht```就是真正的hash结构。
代码中可以看到,·```dictht```并没有key bucket，而是直接的一个二维数组```table```。那么redis为什么要省去key bucket呢？关键在于sizemask。
在寻找某一个key的value时候，redis会首先计算这个key的hash值，然后用这个hash值与sizemask做按位乘，得出的结果，就是这个值在```table```中所在的index。
这里有一个问题：sizemask理论上是由size生成的。最好的方式就是在用到sizemask的时候用size动态生成。这样是最容易保持两个数据一致性的。那么为什么作者不用担心这个问题呢？因为redis是单线程对外服务的。也就是说，不存在两个线程同时操作这两个值。那么，只要redis开发者注意代码中不会存在两者修改的时候不一致就行了。
而在```dicEntry```结构中的```v```。当它对应的对象是存储数据类型hash表的时候，它是```void*```结构。
回到```dictht```,```size```这个字段一般是2的幂次方，所以一般sizemask的生成就比较容易了
```c
sizemask = size<<1 -1
```
当然上述代码并不是真实的这样，因为每次size都是翻倍的，所以真实的代码是
```c
sizemask = sizemask<<1 +1
```
第二行代码的效率是明显比第一行代码效率高的，具体原因读者自己体会。
那么剩下的问题是:hash值是如何计算的呢？
在redis早期版本，所用的版本的代码如下
```c
/* Generic hash function (a popular one from Bernstein).
 * I tested a few and this was the best. */
static unsigned int dictGenHashFunction(const unsigned char *buf, int len) {
    unsigned int hash = 5381;
    while (len--)
        hash = ((hash << 5) + hash) + (*buf++); /* hash * 33 + c */
    return hash;
}
```
这是一个典型的Times33算法。当然，不同的Times33算法，其细节会有差别，性能差异并不会太大(大约1%以内)。这里扯个淡，代码中```hash <<5 +hash```来实现33倍乘的做法是Blizzard大神的的得意之作，虽然其效率是否真的如宣传所言那么高，我表示怀疑(加法永远比乘法慢很多，尤其是高位加法)。而到了最新的redis中，此方法已经被取代为[SipHash](https://131002.net/siphash/siphash.pdf)算法取代。在redis中，具体的实现逻辑参考[siphash.c](https://github.com/antirez/redis/blob/unstable/src/siphash.c)。具体细节不在此文中讨论。
我们说过，redis中所有的对象都是robj，那么hash对应的robj的值是多少呢
```c
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+1;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```
可以看到，robj的type是```OBJ_HASH```,encoding则是```OBJ_ENCODING_ZIPLIST```。
在ziplistNew函数中，你会发现它做的其实就是初始化一个char*类型的header，这个类似于tcp的报文头：有一些原信息以及真实数据的偏移量。具体可以参见代码实现。
###set###
redis的set就简单了，相对于hash只是少了一个key bucket而已。
```c
robj *createSetObject(void) {
    dict *d = dictCreate(&setDictType,NULL);
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}
```
设置值的代码则是在```setTypeAdd```中:
```c
int setTypeAdd(robj *subject, sds value) {
    long long llval;
    if (subject->encoding == OBJ_ENCODING_HT) {
        dict *ht = subject->ptr;
        dictEntry *de = dictAddRaw(ht,value,NULL);
        if (de) {
            dictSetKey(ht,de,sdsdup(value));
            dictSetVal(ht,de,NULL);
            return 1;
        }
    } else if (subject->encoding == OBJ_ENCODING_INTSET) {
        if (isSdsRepresentableAsLongLong(value,&llval) == C_OK) {
            uint8_t success = 0;
            subject->ptr = intsetAdd(subject->ptr,llval,&success);
            if (success) {
                /* Convert to regular set when the intset contains
                 * too many entries. */
                if (intsetLen(subject->ptr) > server.set_max_intset_entries)
                    setTypeConvert(subject,OBJ_ENCODING_HT);
                return 1;
            }
        } else {
            /* Failed to get integer from object, convert to regular set. */
            setTypeConvert(subject,OBJ_ENCODING_HT);

            /* The set *was* an intset and this value is not integer
             * encodable, so dictAdd should always work. */
            serverAssert(dictAdd(subject->ptr,sdsdup(value),NULL) == DICT_OK);
            return 1;
        }
    } else {
        serverPanic("Unknown set encoding");
    }
    return 0;
}
```
可以看到，set的结构和hash几乎一样，只不过在value bucket中，对值有唯一性的判断，也就是```dictSetKey(ht,de,sdsdup(value))```这一行代码。从这一点看得出来，set里面的元素只能是string或者数字。另外，还可以得出另外一个结论:**redis的数据类型不支持嵌套。**
###zset###
相对于上面的set,zset则是一个特殊的set：它的key是数值，而且内存中的顺序是和key正相关的。但是现实中是不是这样实现的呢？
```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```
代码说明，好像并不是这样，而是一个简单的链表，每个链表元素里面存储score，而这个score是唯一的。那么问题来了，它是如何定位到某个score的元素呢,我们看下```zslGetRank```
```c
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        if (x->ele && sdscmp(x->ele,ele) == 0) {
            return rank;
        }
    }
    return 0;
}
```
很失望，没有跳表概念，只有一个双向链表来方便操作。那这么做有没有问题？当然有，有没有解决方案？也当然有。限制最大长度就是了。
###zip###
redis中的zip并不是传统中的数据压缩，而仅仅是内存空间压缩，比如ziplist(相对于list而言)：
```
<zlbytes><zltail><zllen><entry><entry><zlend>
```
可以看出，相对于正常的list，ziplist只是采用内存空间偏移量来压缩空间，当然，所有的压缩存储都有访问快更新慢的问题。
和ziplist相似的，就是zipmap结构，结构也是采用偏移量
```
<zmlen><len>"k1"<len><free>"v1"<len>"k2"<len><free>"v2"
```
那么这种压缩结构什么时候用到呢？不幸的是，面向用户的数据结构是肯定不适合这种存储方式的。取而代之的是，在redis的运行环境中，这种方式却是大量使用的，尤其是很多变化较小的数据。
##内存结构-运行数据##
redis中整体的数据是kv结构，所以，不出意外，内存整体是一个大hashtable, 那我们看下它的定义
```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
不出意外，是使用了前文提到的的```dictht```。但是奇怪的是，这是一个长度为2的一维数组。为什么呢？
因为redis是需要rehash的。也就是说，如果redis的内存结构达到了一定的饱满度，那么就会rehash。所以正常情况下，redis的kv是分布在两个hashtable中的。至于其中的怎么分布以及如何不影响正常功能,参考另一篇文章《redis的rehash机制》。