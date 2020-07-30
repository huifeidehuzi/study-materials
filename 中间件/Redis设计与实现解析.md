# FAILOVER_AUTH_ACKRedis

## 字符串 SDS 

### 字符串结构

```
struct __attribute__((__packed__))sdshdr8 {` 

`uint8_t len; /* 已使用长度，用1字节存储 */` 

`uint8_t alloc; /* 总长度，用1字节存储*/` 

`unsigned char flags; /* 低3位存储类型, 高5位预留 */` 

`char buf[];/*柔性数组，存放实际内容*/` 

`};`

`struct __attribute__((__packed__))sdshdr16 {` 

`uint16_t len; /*已使用长度，用2字节存储*/` 

`uint16_t alloc; /* 总长度，用2字节存储*/` 

`unsigned char flags; /* 低3位存储类型, 高5位预留 */` 

`char buf[];/*柔性数组，存放实际内容*/` 

`};`

`struct __attribute__((__packed__))sdshdr32 {` 

`uint32_t len; /*已使用长度，用4字节存储*/` 

`uint32_t alloc; /* 总长度，用4字节存储*/` 

`unsigned char flags;/* 低3位存储类型, 高5位预留 */` 

`char buf[];/*柔性数组，存放实际内容*/` 

`};`

`struct __attribute__((__packed__))sdshdr64 {` 

`uint64_t len; /*已使用长度，用8字节存储*/` 

`uint64_t alloc; /* 总长度，用8字节存储*/` 

`unsigned char flags; /* 低3位存储类型, 高5位预留 */` 

`char buf[];/*柔性数组，存放实际内容*/` 

`};
```



### 创建字符串

通过sdsnewlen函数创建，依据字符串长度选择不同类型。

```
sds sdsnewlen(const void *init, size_t initlen) {` 

`void *sh;` 

`sds s;` 

`char type = sdsReqType(initlen);//根据字符串长度选择不同的类型` 

`if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;//SDS_TYPE_5强制转化为SDS_TYPE_8` 

`int hdrlen = sdsHdrSize(type);//计算不同头部所需的长度` 

`unsigned char *fp; /* 指向flags的指针 */` 

`sh = s_malloc(hdrlen+initlen+1);//"+1"是为了结束符'\0'` 

`...` 

`s = (char*)sh+hdrlen;//s是指向buf的指针` 

`fp = ((unsigned char*)s)-1;//s是柔性数组buf的指针,-1即指向flags` 

`...` 

`s[initlen] = '\0';//添加末尾的结束符` 

`return s; //其实S就是buff的指针`  

`}` 
```

大致流程：首先计算好不同类型的头部长度和初始长度

1.创建字符串时，SDS_TYPE_5会被强制转换成SDS_TYPE_8

2.长度计算时+1，是为了算上结束字符'\0'

3.返回值是指向SDS结构buff字段的指针，好处是通过指针偏移可以得到结构中的各个变量



### 释放字符串

通过sdsfree函数释放，通过s指针的偏移量，可以定位到sds的头部，然后调用s_free释放内存。

```
void sdsfree(sds s) {` 

`if (s == NULL) return;` 

`s_free((char*)s-sdsHdrSize(s[-1]));//此处直接释放内存` 

`}
```

为了优化性能，节省申请内存的开销，SDS通过重置统计值的方法（sdsclear） 达到请款的目的，

该方法仅将len字段值归零，buff没有被真正的清除，新数据不用重新申请内存，直接覆盖即可。

```
void sdsclear(sds s) {` 

`sdssetlen(s, 0); //统计值len归零` 

`s[0] = '\0';//清空buf` 

`}
```



### 拼接字符串

通过sdscatsds函数拼接，其内部调用了sdscatlen函数拼接字符串

```
sds sdscatsds(sds s, const sds t) {` 

`return sdscatlen(s, t, sdslen(t));` 

`}`



`sds sdscatlen(sds s, const void *t, size_t len) {` 

`size_t curlen = sdslen(s);` 

`s = sdsMakeRoomFor(s,len); //扩容检查` 

`if (s == NULL) return NULL;` 

`memcpy(s+curlen, t, len);//直接拼接，保证了二进制安全` 

`sdssetlen(s, curlen+len);` 

`s[curlen+len] = '\0';//加上结束符` 

`return s;` 

`}` 
```

扩容策略：

1.若sds剩余空闲长度avail大于新增内容长度addlen，直接在buff尾部追加即可

2.若sds剩余空闲长度avail小于新增内容长度addlen

   2.1. 新增后总长度len+addlen<1MB，按新长度2倍扩容

```
`sds sdsMakeRoomFor(sds s, size_t addlen)` 

`{` 

`void *sh, *newsh;` 

`size_t avail = sdsavail(s);size_t len, newlen;` 

`char type, oldtype = s[-1] & SDS_TYPE_MASK;//s[-1]即flags` 

`int hdrlen;` 

`if (avail >= addlen) return s;//无须扩容，直接返回` 

`...` 

`}` 
```

   2.2. 新增后总长度len+addlen>1MB，按新长度加上1MB扩容

```
sds sdsMakeRoomFor(sds s, size_t addlen)` 

`{` 

`...` 

`newlen = (len+addlen);` 

`if (newlen < SDS_MAX_PREALLOC)// SDS_MAX_PREALLOC这个宏的值是1MB` 

`newlen *= 2;` 

`else`

`newlen += SDS_MAX_PREALLOC;` 

`...` 

`}` 
```

3.根据新长度重新选择存储类型，分配空间，若无需更改类型，直接通过realloc扩容即可，否则需要重新

申请内存，将原字符串的buff移动到新位置，具体流程图如下

![image-20200627153949302](.\\redis-image\image-20200627153949302.png)



### 其余常用API

![image-20200627154042288](.\\redis-image\image-20200627154042288.png)



### 总结

1.SDS暴露出来的是buff的指针

2.读操作的复杂度是O(1)，直接读取成员变量（通过指针），设计修改的写操作，可能会触发扩容，

条件是剩余可用长度小于新值

3.SDS通过len限制读取长度，而非'\0'，保证二进制安全

4.SDS设计修改字符串会调用sdsMarkroomFor函数进行检查，根据不同情况扩容。



## 跳跃表

![image-20200627180224480](.\\redis-image\image-20200627180224480.png)

特质：

1.跳跃表由很多层构成

2.有header节点，结构为64层（如图source=0的节点高度为64层），每层结构包含指向下个节点的指针，

指向本层下个节点跨越的节点数为本层的跨度（span）

3.除头节点外，层数最多的节点的层高为跳跃表的高度

4.每层都是有序链表，数据递增

5.除头节点外，一个元素在上层链表中出现就肯定也会在下层链表中出现

6.每层最后一个节点指向NULL，表示结束

7.跳跃表的tail节点，指向跳跃表的最后一个节点

8.最底层的链表包含所有节点且节点个数为跳跃表的长度（不含header节点）

9.每个节点包含后退指针（backward），头结点和第一个节点指向NULL，其他节点指向最底层的前一个节点



### Redis实现跳跃表

理想的跳跃表是：上一层的元素个数是下一层的1/2，便于二分查找，通过抛硬币（1/2概率）的方式插入元素

#### 跳跃表节点

```
typedef struct zskiplistNode {` 

`sds ele; //存储字符串类型的数据` 

`double score; //存储排序的分值,分值相同时，按member的字典顺序排序` 

`struct zskiplistNode *backward; //后退指针` 

`struct zskiplistLevel {` 

`struct zskiplistNode *forward; //指向本层下一个节点的指针，最后一个节点指向NULL` 

`unsigned int span; //当前节点和下一个节点之间的跨越的节点数`  

`} level[]; //存储值的柔性数组` 

`} zskiplistNode;` 
```



#### 跳跃表结构

用来管理跳跃表节点

```
typedef struct zskiplist {` 

`struct zskiplistNode *header, *tail;` 

`unsigned long length;` 

`int level;` 

`} zskiplist;` 
```

header：结构为64层（如图source=0的节点高度为64层），每层结构包含指向下个节点的指针，

指向本层下个节点跨越的节点数为本层的跨度（span）

tail：尾节点

length：跳跃表长度

level：高度



#### 创建跳跃表

##### 节点层高

最小值为1，最大值为ZSKIPLIST_MAXLEVEL，redis5中节点层高值为64

```
int zslRandomLevel(void) {` 

`int level = 1;` 

`while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))` 

`level += 1;` 

`return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;` 

`}` 
```

通过zslRandomLevel函数随机生成1-64的随机值，作为新建节点的高度，值越大出现的概念越小，层高确定后不会再修改。



##### 创建跳跃表节点

在创建节点时，层高，分值，member都已确定。

```
zskiplistNode *zn =` 

`zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));` 
```

通过以上代码申请内存



##### 头节点

头节点是一个特殊的节点，不存储有序集合的member信息。头 

节点是跳跃表中第一个插入的节点，其level数组的每项forward都为 

NULL，span值都为0。

```
`for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {` 

`zsl->header->level[j].forward = NULL;` 

`zsl->header->level[j].span = 0;` 

`}` 
```



##### 创建跳跃表的步骤

1.创建跳跃表结构体对象zsl

2.将zsl的头节点指针指向头节点

3.跳跃表层高初始值为1，长度初始值为0，尾节点指向NULL

```
zskiplist *zsl;` 

`zsl = zmalloc(sizeof(*zsl));` 

`zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL); // 创建头节点并指向头节点`  

`zsl->header->backward = NULL; // 回归指针指向NULL` 

`zsl->level = 1; // 层高为1`  

`zsl->length = 0; // 链表长度为0` 

`zsl->tail = NULL // 尾节点指向NULL;`


```



##### 跳跃表的应用

主要应用于有序集合（另外一种实现是压缩列表，下文会讲到）

redis配置文件中关于有序底层实现有2个配置

1.zset-max-ziplist-enries 128：zset采用压缩列表时，元素最大个数为128

2.zset-max-ziplist-value 64：zset采用压缩列表时候，每个元素的字符串长度最大值为64

zset添加元素的主要逻辑是zaddGenericCommand函数，插入第一个元素时，会判断以下条件

1.zset-max-ziplist-enries的值是否等于0

2.zset-max-ziplist-value小于要插入元素的字符串长度

满足任意条件就会采用跳跃表作为底层实现，否则就采用压缩列表

```
if (server.zset_max_ziplist_entries == 0 ||` 

`server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))` 

`{` 

`zobj = createZsetObject();//创建跳跃表结构` 

`} else {` 

`zobj = createZsetZiplistObject();//创建压缩列表结构` 

`}` 
```

一般情况下zset-max-ziplist-enries不会设置为0，元素字符串长度也不会太长，所以默认实现为压缩列表

zset在插入新元素时，会判断元素个数是否大于zset-max-ziplist-enries或者元素的字符串长度大于zset-max-ziplist-value，满足任意条件，底层实现就会将压缩列表换成跳跃表

zset在转为跳跃表之后，即使元素被逐渐删除， 也不会重新转为压缩列表



## 压缩列表

压缩列表是一个字节数组，每个元素可以是一个字节数组或者整数

Redis的有序集合、散列和列表都直接或者间接使用了压缩列表，当有序集合元素较少且元素字符串长度较短就选择压缩列表为实现。列表使用快速链表结构存储，而快速链表就是双向链表+压缩列表的组合

![image-20200629210156427](.\\redis-image\image-20200629210156427.png)

zlbytes：字节长度，占4个字节，因此压缩列表最多有有2的32次方 -1个字节

zltail：尾元素相对于起始地址的偏移量，占4个字节

zllen：元素个数，占2个字节，必须遍历整个列表才能获取到zllen

entryX：存储的元素，可以是字节数组或整数，长度不限

zlend：列表的结尾，占1个字节，恒为0xFF

![image-20200629210602074](.\\redis-image\image-20200629210602074.png)

previous_entry_length：表示前一个元素的长度，占1个或者5个字节，当前一个元素长度小于254字节时，用1个字节表示，反之用5个字节表示

encoding：表示当前元素的编码，也就是说content字段存储的数组类型

content：当前元素的存储

![image-20200629210929067](.\\redis-image\image-20200629210929067.png)



### 结构体

```
typedef struct zlentry {` 

`unsigned int prevrawlensize;// 前一个元素的长度` 

`unsigned int prevrawlen; // 前一个存储的内容` 

`unsigned int lensize;//encoding字段的长度 ` 

`unsigned int len;// 元素内容的长度`

`unsigned char encoding;//数据类型` 

`unsigned int headersize;// 当前元素的首部长度，prevrawlensize+encoding字段长度之和`

`unsigned char *p;//当前元素首地址（指针） ` 

`} zlentry;` 
```



### 基本操作

#### 创建压缩列表

```
unsigned char *ziplistNew(void) {` 

`//ZIPLIST_HEADER_SIZE = zlbytes + zltail + zllen;` 

`unsigned int bytes = ZIPLIST_HEADER_SIZE+1;` 

`unsigned char *zl = zmalloc(bytes);` 

`ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);` 

`ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);` 

`ZIPLIST_LENGTH(zl) = 0;` 

`//结尾标识0XFF` 

`zl[bytes-1] = ZIP_END;` 

`return zl;` 

`}
```



#### 插入元素

##### 编码

编码即计算previous_entry_length字段、encoding字段和content字段的内容

![image-20200629212017852](.\\redis-image\image-20200629212017852.png)

1.当列表为空时，插入位置为p0，不存在前一个元素，则前一个元素长度为0

2.当列表不为空时，假设插入位置为p1，需要获取entryX元素的长度，这个时候只需要获取entryX+1元素的previous_entry_length即可，因为entryX+1元素的previous_entry_length就是entryX元素的长度

3.当列表不为空时，假设插入位置为P2，需要获取entryN元素的长度，entryN是尾元素，需要计算，计算方式为：前一个元素的长度（prevrawlensize）+ encoding字段的长度（lensize）+元素内容的长度（len）



##### 重新分配空间

##### 数据复制



## 字典

![image-20200630205604540](.\\redis-image\image-20200630205604540.png)

散列表，存储K-V数据结构，Redis整个数据库是用字典来存储的，时间复杂度为O(1)

特征：

1.可以存储海量数据

2.K-V中K的类型可以是字符串，整型，浮点型等，且K是唯一的

3.K-V中V的类型可为String，Hash，List，Set，SortedSet

### Hash

#### Hash表

```
typedef struct dictht { 

dictEntry **table; /*指针数组，用于存储键值对*/ 

unsigned long size; /*table数组的大小 */ 

unsigned long sizemask; /*掩码 = size - 1 */ 

unsigned long used; /*table数组已存元素个数，包含next单链表的数据*/ 

} dictht;


```



#### Hash表节点结构

```
typedef struct dictEntry {` 

`void *key; /*存储键*/` 

`union {` 

`void *val; /*db.dict中的val*/` 

`uint64_t u64;` 

`int64_t s64; /*db.expires中存储过期时间*/` 

`double d;` 

`} v; /*值，是个联合体*/struct dictEntry *next; /*当Hash冲突时，指向冲突的元素，形成单链表*/` 

`} dictEntry
```



#### Hash函数

Hash函数可以把不同键转换成唯一的整型数据

1.同样的值hash后一定相同

2.不同的值hash后可能相同（hash冲突）也可能不相同

redis默认hash算法是“time 33“

以下是redis客户端hash函数

```
static unsigned int dictGenHashFunction (const unsigned char *buf, int len) {` 

`unsigned int hash = 5381;` 

`while (len--)` 

`hash = ((hash << 5) + hash) + (*buf++); /* hash * 33 + c */` 

`return hash;` 

`}
```



#### Hash冲突

不同的值hash后是可能会相同的，所以会造成hash冲突

![image-20200630204801350](.\\redis-image\image-20200630204801350.png)

#### Hash扩容

扩容一倍



### 字典结构

```
typedef struct dict {` 

`dictType *type; /*该字典对应的特定操作函数*/` 

`void *privdata; /*该字典依赖的数据*/` 

`dictht ht[2]; /*Hash表，键值对存储在此*/` 

`long rehashidx; /*rehash标识。默认值为-1，代表没进行rehash操作；不为-1时，代表正进行rehash操作，存储的值表示Hash表ht[0]的rehash操作进行到了哪个索引值*/` 

`unsigned long iterators; /* 当前运行的迭代器数*/` 

`} dict;
```



**type字段指向dictType结构，里面包含了对该字典的函数指针**

```
typedef struct dictType {` 

`uint64_t (*hashFunction)(const void *key); /*该字典对应的Hash函数*/` 

`void *(*keyDup)(void *privdata, const void *key); /*键对应的复制函数*/` 

`void *(*valDup)(void *privdata, const void *obj); /*值对应的复制函数*/` 

`int (*keyCompare)(void *privdata, const void *key1, const void *key2); /*键的比对函数*/` 

`void (*keyDestructor)(void *privdata, void *key); /*键的销毁函数*/` 

`void (*valDestructor)(void *privdata, void *obj); /*值的销毁函数*/` 

`} dictType;` 
```

**privdata：私有数据，配合type字段使用**

**ht：固定大小为2，元素类型为dictht，一般情况只会使用ht[0]，当字典扩容，缩容需要进行rehash时才会使用ht[1]，下面会介绍**

**rehashidx：标记当前字典是否正在rehash，没rehash的时候是-1，反之该值表示ht[0] rehash到了哪个元素，并记录该元素的当前下标**

**iterators：安全迭代器，下文会介绍**



### 基本操作

#### 字典初始化

redis-server启动时，会先初始化一个空字典存储整个数据库的键值对，调用的是dictCreate函数



#### 添加元素

如 set name 张三 

主要逻辑：

1.调用dictFind函数查询key是否存在，存在调用dbOverwrite函数修改键值对，否则调用dbAdd添加元素

2.dbAdd最终调用dictAdd添加元素（计算hash值，得到下标，将元素插入下标处）

3.used+1

#### 字典扩容

容量不足时，需要对字典进行扩容，调用dictExpand函数



#### 查找元素

如 get name ，调用dictFind函数

步骤：

1.计算key的hash值

2.遍历ht[0]和ht[1]，计算下标值，获取对应下标的entry

3.如果有hash冲突的值，则找到key相等的entry返回



#### 修改元素

如set name 李四，调用dbOverwrite函数

步骤：

1.调用dictFind查找元素是否存在，不存在直接return

2.将dictFind查到的元素的value设置为新值

3.释放旧值内存



#### 删除元素

如 del name，调用dictDelete函数

步骤：

1.查找元素是否存在

2.存在则把元素从数组剔除

3.释放内存

4.对应hash表的used-1

5.如果使用量<总空间10%，会进行缩容操作（公式：used/size<10%）



### 字典的遍历

1.全遍历（keys）：一次命令执行就遍历完整个数据库

2.·间断遍历（hscan）：每次命令执行只取部分数据，分多次遍历





## 整数集合

一个有序的，存储整型的数据结构，当元素都是整数且元素都处在64位有符号整数范围内就使用整数集合

注：

1.当元素个数超过默认值512个后，底层结构会转换成hash表，配置项：set-max-intset-entries 512 

2.当新增非整型元素时（如新增元素“abcd”）底层结构也会转换成hash表

可通过object encoding key 查看底层编码结构（数据结构）



### 数据存储

整数类型可以存储int16_t、int32_t、int64_t类型的整数，并保证集合中不会出现重复数据，整数类型用intset结构存储

```
typedef struct intset {` 

`uint32_t encoding;//编码类型` 

`uint32_t length;//元素个数` 

`int8_t contents[];//柔性数组,根据encoding字段决定几个字节表示一个元素` 

`} intset
```



![image-20200701201325597](.\\redis-image\image-20200701201325597.png)

**encoding：编码类型，决定每个元素占用的字节大小，有3种：**

**1.INTSET_ENC_INT16：占用2个字节**

**2.INTSET_ENC_INT32：占用4个字节**

**3.INTSET_ENC_INT64：占用8个字节**

**判断使用哪种编码，只需要判断值的范围即可**



![image-20200701205623630](.\\redis-image\image-20200701205623630.png)



**length：元素个数**

**contents：存储元素的数组**

![image-20200701205719107](.\\redis-image\image-20200701205719107.png)



### 基本操作

#### 查询元素

查询元素的入口函数是intsetFind

1.判断编码方式，如果超出3种编码方式的范围，直接返回0

2.intset是有序的数组，所以会通过二分查找元素

#### 添加元素

调用intsetAdd函数

1.判断插入值的编码是否需要升级

2.判断插入值是否已存在

3.如果插入值在数组中间位置，则需要挪动空间

4.插入元素

5.length字段+1

#### 删除元素

调用intsetRemove函数

1.获取待删除元素的编码，编码<=当前数组编码且元素存在才会删除

2.如果元素位于数组中间位置，则直接覆盖此元素

3.如果元素数组末尾，直接缩容即可

4.length字段-1

#### 常用API

![image-20200701210442484](.\\redis-image\image-20200701210442484.png)

#### 操作复杂度

![image-20200701210520621](.\\redis-image\image-20200701210520621.png)

## 快速列表（quicklist）

在3.2版本之前，redis采用的是压缩列表和双向链表作为List的实现，当元素长度较短，元素个数较少时，采用压缩列表存储，反之采用双向链表存储，因为当元素个数过多，元素长度较长时，压缩列表需要的空间是连续的，在修改元素时，需要对空间进行重新分配，比较耗时。

它是Redis对外提供的6种基本数据结构中List的底层实现



### 简介

**quicklist由list和ziplist组合**

**quicklist是一个双向链表，每个节点是一个ziplist**

**1.当ziplist中节点个数过多时，quicklist会将底层结构转换为双向链表（list）**

**2.即使每个ziplist中只包含一个元素，当ziplist节点个数较少时，quicklist会将底层结构转换为ziplist**



### 结构

```
typedef struct quicklist {` 

`quicklistNode *head;` 

`quicklistNode *tail;` 

`unsigned long count; /* total count of all entries in all ziplists */` 

`unsigned long len; /* number of quicklistNodes */` 

`int fill : 16; /* fill factor for individual nodes */` 

`unsigned int compress : 16; /* depth of end nodes not to compress;0=off */` 

`} quicklist;
```



![image-20200701211615322](.\\redis-image\image-20200701211615322.png)



**quicklist：整个快速列表集合，其中head是头节点，tail是尾节点，count是所有节点中ziplist内的元素数量总和，len为节点的数量**

**quicklistnode：集合中的单个节点，pre是指向前一个节点的指针，next是指向下一个节点的指针，zl是存储元素的数组**

**ziplist：上文有介绍，这里不做阐述**



### 数组压缩

#### 基本操作

## Stream

消息队列，由消费，生产者，消费者，消费组组成

![image-20200702211605788](.\\redis-image\image-20200702211605788.png)

通过指令：xadd mystream1 * name hb age 20  插入一条消息

mystream1代表stream名称，*代表自动生成消息id，name，age为消息的字段，hb，20对应字段的值，每个消息由2部分组成。

1. 每个消息有唯一id且递增
2. 消息内容由多个k-v组成

生产者向队列发送消息，消费者不归属任何消费组时，可以消费队列中任何消息，反之只能消费归属的消费组内的消息

消费组：

1.组名唯一，每个消费组独立且能消费该消息队列的全部消息

2.可以有多个消费者，消费者通过唯一名称标识，一个消息只能由消费组内的一个消费者消息

3.消费者消费消息后需要确认，每个消费组都有一个待确认的消息队列（pending entry hst，pel）用以维护该消费组已消费但未确认的消息

4.消费组的每个消费者也有一个待确认消息队列，维护该消费者已消费未确认的消息



stream底层使用listpack及rax树。



### listpack

一个字符串列表序列化后存储，可以储存字符串或者整型

![image-20200703231518711](.\\redis-image\image-20200703231518711.png)

Total Bytes：整个listpack空间大小，占4个字节

Num Elem：listpack中元素数量，占用2个字节

End：结束标识

Entry：存储的元素

1.Encode：元素编码

2.content：元素内容

3.backlen：元素长度，不包含backlen本身的长度，主要用于从后向前遍历



### Rax

前缀树，字符串查找时常用的数据结构，能快速在字符串集合中找到某个字符串。

rax不仅可以存储字符串，还可以为这个字符串设置一个值，也就是存储key-value



### stream结构

xxxxxxx恶心的一B 我先跳过了



## 命令处理生命周期

### 对象结构体

redis是一个k-v型数据库，key只能是字符串，value可以是字符串，列表，集合，有序集合和散列表，这五种数据类型可以用robj表示（redis对象），robj的type字段表示对象类型，五种类型定义如下：

1. OBJ_STRING
2. OBJ_LIST
3. OBJ_SET
4. OBJ_ZSET
5. OBJ_HASH

针对某一种类型的对象，不同情况下可能使用不同的数据结构，robj的encoding字段表示当前对象底层存储的数据结构（对象的编码）

​																			对象编码类型表

| encoding常量           | 数据结构     | 可存储对象类型       |
| ---------------------- | ------------ | -------------------- |
| OBJ_ENCODING_RAW       | SDS          | 字符串               |
| OBJ_ENCODING_INT       | 整数         | 字符串               |
| OBJ_ENCODING_HT        | 字典（dict） | 集合、列表、有序集合 |
| OBJ_ENCODING_ZIPLIST   | 压缩列表     | 散列表、有序集合     |
| OBJ_ENCODING_INTSET    | 整数集合     | 集合                 |
| OBJ_ENCODING_SKIPLIST  | 跳跃表       | 有序集合             |
| OBJ_ENCODING_EMBSTR    | SDS          | 字符串               |
| OBJ_ENCODING_QuickList | 快速链表     | 列表                 |
| OBJ_ENCODING_STREAM    | stream       | stream               |



## 字符串相关命令

字符串是以k-v形式存储在dict中，key经过hash后作为dict的key，key只能是String，dict的值是value，用robj（type为OBJ_STRING）表示，当字符串值是String时，encoding为OBJ_ENCODING_RAW或OBJ_ENCODING_EMBSTR，当字符串是long类型时，encoding为OBJ_ENCODING_INT

字符串超时时间格式为毫秒和秒，当设置为秒时会转换成毫秒，当前时间毫秒数+过期时间毫秒数=过期时间，过期时间存储在redisDb的expire字典中，key是字符串的key，value是过期时间毫秒数



### set命令

`SET key value [NX] [XX] [EX <seconds>] [PX <milliseconds>]` 

如果key已存在，则set值会覆盖旧值，如果在设置时不指定EX或PX参数，set命令会清除原有超时时间



NX：key不存在时可set值，反之不可set值

XX：key存在时可set值，反之不可set值

EX：key的超时秒数（set值时会自动转成毫秒）

PX：key的超时毫秒数，与EX互斥（只能设置一个）

`setnx key value`  当key不存在时才可set值

`setex key seconds value` set值并指定超时秒数



### mset命令

批量设置值

```
mset key value [key value ...]  key存在则覆盖旧值

msetnx key value [key value ...] 所有的key都不存在值才能set值
```



### 修改字符串

#### append命令

`append key value`  如果key存在，将value追加到旧值的后面，反之插入值

**注意：**

1. **当追加的值类型不为String时会报错**
2. **字符串长度必须小于512MB，否则也会报错**

为什么字符串长度在追加的时候才判断大小？set的时候不判断？因为服务端在接收set请求的时候是会对字符串长度做判断的（不能大于512MB）



#### setrange命令

`setrange key offset value` 

从offset偏移量开始替换值，如果key不存在则会插入值

**注意：**

1. **offset类型必须为long，否则会设置失败**
2. **只有当offset+sdslen(value)(字符串长度) >原值长度空间才会扩充空间，否则直接返回原值**





#### 计数器命令 

```
incr key 将key值+1

decr key 将key值-1

incrby key increment 将key值+指定数

decrby key decrement 将key值-指定数
```

调用incrDecrCommand函数

判断是否存在key，不存在设置默认值0，反之判断value值是否为long类型，不是就报错



#### get命令

`get key`  将key经过hash后根据hash值从字典中获取对应数据

`getset key value`   设置新值并返回旧值



#### getrange命令 

`getrange key start end` 获取从start到end截止的字符串内容



#### strlen命令

`strlen key`  获取字符串长度



#### mget命令

`mget key [key …]`  获取多个key的内容



## 事务

redis提供了multi和exec支持事务，开启事务后，所有的命令不会先执行，只有exec提交才会执行

如：

```
127.0.0.1:6379> multi //开启事务 

OK

127.0.0.1:6379> incr counter1 //counter1加1 

QUEUED //命令入队 

127.0.0.1:6379> incr counter2 //counter2加1 

QUEUED //命令入队 

127.0.0.1:6379> exec or discard //执行事务 or 放弃事务

1) (integer) 1 

2) (integer) 1 
```

另外，redis提供了watch命令监听多个key，这是一种乐观锁机制，只有被监听的key没有被修改时才会执行exec，等同于mysql的version版本号要小于等于被修改的版本号时才会修改数据

```
127.0.0.1:6379> watch counter1 counter2 //监控counter1和counter2 

OK

127.0.0.1:6379> incr counter1 //修改counter1，注意此时counter1的值为2 

(integer) 2 

127.0.0.1:6379> multi //开启事务 

OK

127.0.0.1:6379> incr counter1 

QUEUED 

127.0.0.1:6379> incr counter2 

QUEUED 

127.0.0.1:6379> exec //执行事务 

(nil) //返回nil127.0.0.1:6379> get counter1 //counter1的值仍然为2 

"2" 

127.0.0.1:6379> get counter2 //counter2的值仍然为1 

"1"
```

通过上面的示例可以看出，当counter1被再次incr时，事务并未提交，当一个事务exec或discard后，所有watch的key会自动unwatch



## 发布订阅

生产者可以向指定的channel发送消息，消费者可以指定channel消费

`publish channel message` 指定channel发送消息，返回收到消息的客户端数量

`subscribe channel [channel ...]`  订阅指定channel，返回值为数组，第一个元素固定为subscribe，第二个元素为订阅的channel，第三个元素为订阅channel的数量（包含模式订阅）

`unsubscribe [channel [channel ...]]`   取消订阅，如果不指定channel，则取消所有订阅

`psubscribe pattern [pattern ...]`  模式订阅（匹配订阅）pattern：？表示匹配1个字符，*匹配多个字符，[]匹配任意一个字符

`punsubscribe [pattern [pattern ...]]`  取消模式订阅





## LUA

lua是所有客户端共用的，这也是redis原子性操作的保证

在lua中调用redis脚本





## 持久化

### RDB

1.`save 60 1000`  通过参数配置，如60s内有1000个key发生变化则触发一次RDB快照执行，以二进制保存在dump.rdb文件中

2.在客户端通过bgsave命令触发

3.fork子进程进行快照

4.机器shutdown会触发快照



### AOF

aof保存的每次新增，修改，删除的命令，保存到aof文件，缺点是每次执行都会保存命令，导致文件过大

相对于rdb，aof恢复数据的性能明显比rdb慢，但是持久化数据相对较全，试想下，如果访问量很大，导致rdb的次数过多，那么redis性能就会越慢，发生故障丢失的数据也会过多

所以redis一般会同时开启这2种持久化方式，就是aof和rdb的混合持久化



#### 命令同步

比如保存 set key aof

写入到缓存区的内容为：

```
(gdb) p dst 
// sds 字符串格式
$1 = (sds) 0x7f2050834463 "*3\r\n$3\r\nset\r\n$3\r\nkey\r\n$3\r\naof\r\n" 
```

\r\n 为分隔符

![image-20200707220905084](.\\redis-image\image-20200707220905084.png)

如上图

*3 表示有3个参数

第一个$3 表示第一个参数长度为3，读取为set，到此第一个参数解析完毕

第二个$3表示第二个长度的长度为3，读取为key

第三个$3表示第三个参数长度为3，读取为aof

到此，命令解析完毕



#### 文件写入

aof最终是将缓冲区的内容写入到aof文件，真正是调用fsync函数，这是个阻塞并且执行缓慢的操作。所以redis提供了appendfsync配置控制fsync的执行频率

1. no：不执行fsync，由系统负责数据的写入aof文件，安全性最低但性能最高
2. always：每执行一条命令执行一次fsync，安全性最高但性能最低
3. everysec：每秒执行一次fsync，折中方案，安全性和性能平衡（默认配置）



#### 文件重写

对于这种每次操作都将命令写入文件的方式显然是不太合理的，在大量的请求场景下aof文件会越来越大，恢复数据的性能会越来越慢，所以redis提供了aof文件重写的方案来解决这个问题

aof重写通过fork出一个子进程来执行，重写不会操作原有的aof文件，它会对redis中所有的key都生成一条相应的执行命令，最后将重写开始后父进程继续执行的命令进行回放，生成一个新的aof文件

如：

127.0.0.1:6379> rpush list 1 2 3 //list中增加1,2,3三个元素 

```
(integer) 3 

127.0.0.1:6379> rpush list 4 //list中增加4 

(integer) 4 

127.0.0.1:6379> rpush list 5 //list中增加5 

(integer) 5 

127.0.0.1:6379> lpop list //弹出第一个元素 

"1"
```

如上示例，aof文件保存了对list操作的4条命令

aof重写的方式就是直接将命令改为：rpush list 2345 ，变成一条命令，这样aof文件将大大减少，恢复数据的性能也提升很多



#### 重写触发方式

1.自动触发

```
auto-aof-rewrite-percentage 100 
auto-aof-rewrite-min-size 64mb 
```

文件内容大于64MB，且aof当前文件大小比基准大小增长了100%就会触发一次aof重写

基准大小：

起始的基准大小为redis重启并加载完aof文件后，aof_buf的大小（缓存区），当执行完一次重写后，基准大小更新为重写后aof文件的大小



2.手动触发

执行bgrewriteaof命令触发



#### 混合持久化

aof-use-rdb-preamble yes // 开启混合划配置

aof重写时子进程将当前时间点的快照保存为RDB文件格式，再将父进程累计的命令保存为aof格式，最终生成以下格式：

![image-20200707222801263](.\\redis-image\image-20200707222801263.png)



加载时，会识别aof文件是否以redis字符串开头，如果是就按RDB格式加载，加载完后继续按aof格式加载剩余数据



#### 配置项

![image-20200707222942065](.\\redis-image\image-20200707222942065.png)



1. stop-writes-on-bgsave-error：开启后，如果开启了RDB（配置了save），且最后一次快照执行失败，redis会停止接收写相关的请求
2. rdb-save-incremental-fsync：开启后，生成RDB文件时每产生32MB数据就执行一次fsync
3. no-appendfsync-on-rewrite：开启后，如果子进程正在执行RDB或者AOF重写，则主进程不再执行fsync操作，即便appendfsync配置为always或者everysec
4. aof-load-truncated：开启后，aof文件以追加日志的方式生成，当服务器故障时可能有命令不完整的情况下，会丢弃（截断）不完整的命令然后继续加载，并且在日志中进行提示，如果不开启此参数，则加载aof文件时会打印错误日志并退出



## 主从复制

1.执行salveof命令

2.设置salveof配置

比如：有两台服务 器——127.0.0.1:6379和127.0.0.1:7000

```
127.0.0.1:6379> slaveof 127.0.0.1 7000 
```

6379向7000发送salveof，6379会成为7000的salver，7000会成为6379的master，6379的数据和7000的数据保持一致



### 主从复制功能实现

1.读写分离：单台机器QPS是有限的，通过主从复制，比如mater机器处理写请求，salver处理读请求

2.数据容灾：主从复制后，单台机器宕机都是无所谓的，只要切换机器就行



slaveof命令流程：

1.slaver向mater发送sync命令，请求同步数据

2.master收到sync命令，开始执行bgsave命令生成rdb快照，在持久化期间会将所有的写命令都保存到缓存区（2.8之前版本）

3.master将生成的rdb文件发送给slaver，slaver接收rdb文件并将数据加载到内存

4.master将生成rdb文件期间的写入缓存区的命令发送给slaver。salver接收并执行

5.之后，每当master有写命令都会发送给slaver，slaver接收并执行

**注意：**

2.8版本用**完整重同步**实现，slaver发送psync命令给master请求同步数据

每台redis机器都会有一个RUN ID（运行ID），slaver每次发送psync时会携带需要同步的master运行ID，master接收请求时会判断此ID是否与自己的ID一致，一致才会执行部分重同步操作，如果是第一次则执行完整同步操作。

第一次同步，slaver是没有master的RUN ID，此时会默认填充？，OFFSET为-1

![image-20200711223715769](.\\redis-image\image-20200711223715769.png)



而在实际环境中，经常会遇到2个问题

1.slaver重启（复制信息丢失）

2.master故障，其他slaver会重新选举master，导致RUN ID变了，导致无法执行部分重同步

解决方案：

1.持久化主从复制信息：机器宕机时，将主从复制信息（RUN ID + OFFSET）持久化到RDB文件中，机器重启时加载RDB文件就可以拿到RUN ID + OFFSET，此时就可以重新进行同步了

2.存储上一个机器的复制信息：机器宕机时，被重新选举的master会使用replid2字段记录宕机机器的RUN ID，使用second_replid_offset字段记录宕机机器的OFFSET

例如：m为master（RUN ID = M_ID），A、B为slaver，此时m宕机，A被选举为master（记录replid2=M_ID，second_replid_offset=M_OFFSET），当B发送psync请求同步时依然是没问题的
### 心跳检测
在命令传播节点，slave默认每秒向master发送REPLCONF ACK <OFFSET> 命令，发送此命令有3个作用
1. 检查主从的网络链接状态
2. 辅助实现min-slaves选项
3. 检测命令丢失

#### 检查主从的网络链接状态
如果master超过1秒没有收到来自slave的REPLCONF ACK命令，那么它就知道主从之间链接肯定出问题了。
我们可以在master上通过INFO replication命令查看所有slave发送REPLCONF ACK的最后一次时间
 ![image-20200722204842787](.\\redis-image\image-20200722204842787.png)

lag单位为：秒

一般情况下，lag的值是在0-1(秒)直接，如果超过1就说明主从主键的链接有问题

#### 辅助实现min-slaves配置选项
通过配置master的2个参数，master将拒绝写命令
min-slaves-to-write：3
min-slaves-max-log：5
如果slave数量少于3，或3个slave的lag都>=5秒，master将拒绝写命令

#### 检测命令丢失
如果因网络故障导致master给slave的写命令半路丢失，那么slave再次发送REPLCONF ACK命令时，master会比较命令中的偏移量，如果发现slave的偏移量小于master的，则会从缓冲区中找到丢失的数据重新发送给slave
此操作与部分重同步类似，区别在于部分重同步是在主从重连后执行的，而补发丢失数据是在master正常情况下执行的
注意：REPLCONF ACK是2.8版本新增的，如果是2.8之前的版本，丢失数据是不会补发的，所以为了保证主从复制一致性，最好是使用2.8或者以上的版本

## 数据库

```
struct redisServer{
    redisDb *db; // db数组，默认0-15
    int dbnum; // 数据库数量 默认16
}

// 数据库结构
struct redisDb{
    dict *dict; //数据库键空间(字典)，保存所有键值对
    dict *expiress; //过期字典，保存key的过期时间
}
```

设置过期时间，就是在expiress中新增一个kv，k是过期key，v是过期时间戳
移除过期时间，就是从expiress中将对应字典移除即可

### 过期key删除策略
1. 定时删除：在设置key过期时间时，创建一个定时器，在key过期时间快到时，立即删除key
2. 惰性删除：获取key的时检查key是否过期，如果过期就删除，反之就返回key
3. 定期删除：每隔一段时间，定时删除过期key，具体删除多少可以，检查多少库由算法决定

#### 定时删除
1. 对内存非常友好，因为可以尽快的删除过期key，释放内存
2. 对cpu不友好，试想下，在过期key比较多且请求比较大的时候，cpu应关注的是处理请求而不是来删除过期key，这会对性能和吞吐量有影响

#### 惰性删除
1. 对CPU非常友好，因为只有在需要获取key的时候才会执行删除操作
2. 对内存非常不友好，因为过期key如果不被获取，那么会一直存储在库里，浪费内存（内存泄漏？），比如日志
#### 定期删除
上述2种方式的结合体，保证了cpu的性能和内存的释放，但是要根据实际情况合理的设置删除key的执行时长和频率



## 集群

命令：

添加节点到集群中：CLUSTER MEET <ip> <port>

查看节点的集群：CLUSTER NODES

向一个节点发送 cluster meet 命令，可以让发送命令的该节点(A)与ip和port指向的节点(B)进行握手（handshake），握手成功后A节点会将B节点添加到A节点所在的集群中

例如：7000、7001（省略IP）2个节点，首先连接7000节点，使用cluster nodes命令可以看到目前集群中就7000一个节点

![image-20200722215806082](.\\redis-image\image-20200722215806082.png)

然后发送cluster meet 127.0.0.1:7001命令将7001节点添加到7000节点的集群中

![image-20200722215911284](.\\redis-image\image-20200722215911284.png)

此时可以看到7000节点集群中有2个节点（7000和7100）

![image-20200722215935261](.\\redis-image\image-20200722215935261.png)



### 启动节点

Redis服务启动时会根据cluster-enabled配置是否为true来决定是否开启集群



### 集群数据结构

每个节点使用clusterNode保存，并且也会为集群中其他节点创建对应的clusterNode



```
struct clusterNode(`
	// 节点创建时间
	`mstime_t  ctime; `

	// 节点的名称，40个16进制的字符组成`
	`// 例如：4645f4sd45fds4f65ds4f6sd4f6d4`
	`char name[REDIS_CLUSTER_NAMELEN];  
	
	// 节点标识，1.节点的角色（主或从）2.状态（在线或下线）`
	`int flags; 
	
	// 节点当前的配置纪元，用于实现故障转移`
	`uint64_t configEpoch; 
	
	// 节点的ip`
	`char ip[REDIS_IP_STR_LEN] ;
	
	// 节点的端口号`
	`int port; 
	
	// 保存连接节点所需的信息`
	`clusterLink * link; 
	// 其余字段可能会在下面文章中补充
`）
```



其中clusterLink结构如下

```
typeedf struct clusterLink(
	// 连接的创建时间
	mstime_t ctime;
	// TCP套接字描述符
	int fd;
	// 输出缓冲区，保存等待发送给其他节点的message
	sds sndbuf;
	// 输入缓冲区，保存从其他节点收到的message
	sds rcvbuf;
	// 与这个节点相关联的节点，如果没有的就为null
	struct clusterNode *node;
)
```

此外，每个节点都保存着一个clusterState结构，用来保存在当前节点视觉下，集群的状态，节点数量，配置纪元等等。

```
typedef struct clusterState(
	// 指向当前节点的指针
	clusterNode *myself;
	// 集群当前的配置纪元，用于实现故障转移
	uint64_t currentEpoch;
	// 集群当前的状态：在线或下线
	int state;
	// 集群中至少处理着一个槽的节点数量
	int size;
	// 集群的节点名单，包含myself节点
	// key为节点名称，value为clusterNode
	dict *nodes;
	// 其余字段可能会在下面文章中补充
)
```



### cluster meet握手步骤

1. 节点A会为节点B创建一个clusterNode，并将node添加到节点A的nodes字段里
2. 节点A发送cluster meet ip port 给节点B，B会收到meet命令，B会为A创建一个clusterNode，并将node添加到节点B的nodes字段里
3. 节点B向A返回一条Pong消息
4. A收到B的pong消息就知道B已成功接收到自己的meet消息
5. A给B返回一条ping消息
6. B收到A的ping消息就知道A已经接收到B的pong消息
7. 握手完成
8. 握手成功后，连个节点之间会定期发送ping/pong消息，交换数据信息
9. A将B通过Gossip协议传播给集群中其他节点，其他节点也会与B进行握手，从而被其他节点认识

![image-20200725185206841](.\\redis-image\image-20200725185206841.png)

redis集群内节点，每秒都在发ping消息。规律如下

- (1)每秒会随机选取5个节点，找出最久没有通信的节点发送ping消息
- (2)每100毫秒(1秒10次)都会扫描本地节点列表，如果发现节点最近一次接受pong消息的时间大于cluster-node-timeout/2 则立刻发送ping消息



### 槽指派

槽：redis有16384个槽，每个节点对应一部分，比如有3个master节点A、B、C，那么每个节点对应的槽可以为A0-5000，B5001-10000，C10001-16383, redis将Key进行CRC16(key)&16383，得到对应的槽位，从而找到对应的节点

集群环境下，redis通过分片来保存数据，每个master节点存储一部分数据，且每个master节点至少有1个slave，集群的整个数据库被分散到16384个槽中

当所以槽点都有节点在处理的时候，集群状态=上线，反之如果有任何一个槽点没有节点处理则集群状态=下线



### 记录节点的槽指派信息

```
struct clustNode{
	// ...
	unsigned char slots[16384/8];
	int numslots;
	// ...
}
```

slots 是一个二进制位数组（bit array），长度为16384/8=2048个字节，包含16384个二进制位

1. 如果slots数组在索引上的值为1，表示处理此槽位
2. 如果slots数组在索引上的值为0，表示不处理此槽位

![image-20200727221346724](.\\redis-image\image-20200727221346724.png)

numslots记录节点处理的槽数量，如上图，numslots为8

同时，集群中每个节点会将自己slots发送给其他节点，告知它们自己要处理哪些槽位，所以每个节点都知道其他节点处理的槽位



### cluster addslots 命令

```
cluster addslots <slot> [slot ..]
```

将指定的槽位给发送命令的节点处理

![image-20200727222453043](.\\redis-image\image-20200727222453043.png)

比如上图，clusterState.slots数组的所有指针都为NULL，并且clusterNode的slots数组中值为0，说明当前节点没有被指派槽，且所有槽位都没有被指派

当客户端执行：cluster addslots 1 2，将会将槽1和2指派给该节点

1. 该节点的slots中的索引0和1的值会变更成1
2. clusterState.slots中的0和1槽位都会指向该节点（clusterNode）
3. 该节点会通知集群中其他节点自己处理的槽位

![image-20200727223002961](.\\redis-image\image-20200727223002961.png)



### 在集群中执行命令

16384个槽位被指派之后，集群状态=上线，此时客户端就可以像集群中的节点发送命令了

当客户端执行命令时候（数据有关），接收命令的节点会做如下操作

1. CRC16(key) &16383 得到槽位索引，如果槽位正好对应当前节点则直接执行命令
2. 如果没有对应当前节点，则返回MOVE错误，客户端会重试，转向正确的节点

![image-20200727223504586](.\\redis-image\image-20200727223504586.png)



```
// 查看key对应的槽位
cluster keyslot key
// move错误
move ip port
```

1. 计算出key对应槽位索引后，节点会判断clusterState.slots[i] = clusterState.myself，如果相等则执行命令
2. 如果不相等，则记录clusterState.slots[i]对应的clusterNode的ip和port，返回MOVE错误，客户端根据ip+port转到对应节点执行命令

另外，clusterState.slots_to_keys（跳跃表）字段会记录槽位索引和key的关系

```
typedef struct clusterState{
	zkiplist *slots_to_keys; //保存槽位索引和key的对应关系
}
```

slots_to_keys每个节点的score对应槽位索引，member对应key

1. 添加一个新的kv时，节点会将key和对应槽位索引保存到slots_to_keys
2. 删除kv时，slots_to_keys删除key和对应槽位索引的记录

通过slots_to_keys，节点可以很方便的对1个或多个槽的key进行批量操作，例如cluster getkeysinslot <slot> <count> 可以返回某个槽位最多count个key，这个命令就是通过遍历slots_to_keys实现的



### 重新分片

集群的重新分配操作可以将已经指派给某个节点（sourceNode）的槽重新指派给另一个节点（targetNode），且被重新指派的槽对应的kv也会从sourceNode被移动到targetNode

重新分片可以在线操作，集群不需要下线，并且sourceNo和targetNode可以继续处理请求

重新分片是由redis-trib负责执行，以下是redis-trib对单个槽slot重新分片的步骤

1. redis-trib对targetNode发送cluster setslot <slot> importing <source_id>命令，让targetNode准备好从sourceNo导入slot的kv（键值对）
2. redis-trib对sourceNode发送cluster setslot <slot> migrating <taget_id>命令，让sourceNo准备好将slot对应的kv迁移至targetNode
3. redis-trib对sourceNode发送cluster getkeysinslot <slot> count命令，获取slot下count个key
4. 获取到步骤3的key后，redis-trib向sourceNode发送一个migrate <target_ip>  <target_port> <key> 0 <time_out>命令，将key原子操作从sourceNode迁移至targetNode
5. 重复执行步骤3和4，直到所有key被迁移完成
6. redis-trib向集群中任意节点发送cluster setslot <slot> node <target_id> 命令，将该槽位指派给targetNode，这一指派会通知给整个集群，所有节点都知道该槽位已经被指派给了targetNode
7. 如果重新分配涉及到多个槽，redis-trib会对每个槽进行步骤1-6的操作
8. 注：上述的source_id和target_id指集群中节点的id，而非运行id

![image-20200728222738756](.\\redis-image\image-20200728222738756.png)

![image-20200728222832256](.\\redis-image\image-20200728222832256.png)



### ASK错误

在重新分片期间，可能会出现：被迁移槽的部分k-v存储在sourceNode中，另一部分k-v存储在targetNode中，当客户端像sourceNode发送一条操作数据的命令并且操作的key恰好落在被迁移槽位中时

1. sourceNode会先在自己的db中查找key，如果存在则直接返回
2. 反之，这个key可能已经被迁移至targetNode中，此时sourceNode会返回ASK错误，指引客户端转向到targetNode，重新执行命令

```
// ask返回错误
ASK slot taget_ip:target_port
```



### cluster seslot importing命令实现

```
typedet struct clusterState{
	// ...
	// 记录当前节点正在从其他节点导入的槽
	clusterNode *importing_slots_from[16384];
	// ...
}
```

如果importing_slots_from[i]不为空，而是指向一个clusterNode，那么表示当前节点正在从clusterNode对应的节点导入槽[i]

发送cluster setslot <i> importing <source_id>，会将targetNode的clusterState.importing_slots_from[i]设置为source_id对应的clusterNode



![image-20200728224456790](.\\redis-image\image-20200728224456790.png)



### cluster setslot migrating命令实现

```
typedet struct clusterState{
	// ...
	// 记录当前节点正在迁移至其他节点的槽位
	clusterNode *migrating_slots_to[16384];
	// ..
}
```

如果migrating_slots_to[i]不为空，而是指向一个clusterNode，表示当前节点正在将槽[i]迁移至clusterNode节点

发送cluster setslot <i> mirgating <target_id> 会将sourceNode的clusterState.mirgating_slots_to[i]设置为target_id对应的clusterNode

![image-20200728224928996](.\\redis-image\image-20200728224928996.png)





### ASK错误实现

迁移过程中，如果sourceNode收到请求，那么节点会检查clusterState.migrating_slots_to[i]是否正在迁移，如果在迁移则返回ASK slot target_ip:target_port，指引客户端去targetNode查找key



### 复制与故障转移

集群中的master负责处理槽，slave负责负责master，并在被复制的master下线时代替master（成为master）

#### 设置从节点

```
// 设置某个node成为发送命令节点的slave，且同时对发送命令的节点进行辅助
cluster replcate <node_id>
```

1. 接收到命令的节点会在clusterState.nodes中找到node_id对应的clusterNode，并将clusterNode.myself.slaveof指针指向发送消息节点对应的clusterNode

```
struct clustNode{
	// ...
	//如果当前clusterNode是slave，那么该字段指向master
	struct clustNode *slaveof;
	// ...
}
```

2. 然后该slave会修改自己在clusterState.myself.flags=reids_node_salve
3. 最后，根据clusterState.myself.slavof指向的主节点clusterNode获取到master的ip和port，对master进行复制（发送slave of master_ip master_port）

![image-20200729213720969](.\\redis-image\image-20200729213720969.png)



#### 故障检测

```
// 集群节点PING默认超时时间，在redis.conf配置
cluster-node-timeout 15000
```

集群中每个节点会定期向其他节点发送PING来检测对方是否在线，如果接收PING的节点在规定时间内没有返回PONG，那么发送PING的节点会将对应节点标记为疑似下线（probable fail，PFAIL）

![image-20200729214033047](.\\redis-image\image-20200729214033047.png)

之后，集群中各个节点会互相发送消息来交换集群中各个节点的状态，如：某个节点在线、下线、疑似下线

如果半数以上负责处理槽的master都报告某个master（X）为疑似下线，那么X会被标记为已下线状态，将X标记为下线状态的节点会广播一条X下线的FAIL消息，收到消息的所有节点都会将X标记为下线状态



#### 故障转移

当某个节点发现正在复制的master下线时，该节点将开始对下线的master进行故障转移，步骤如下：

1. 复制master的所有slave会有一个被选中
2. 被选中的slave会执行slaveof no one，成为新的master
3. 新的master会撤销所有对已下线maste的槽指派，并将这些槽位指派给自己
4. 新的master会广播一条PONG消息给集群内其他节点，其他节点就知道这个节点是新的master
5. 新的master开始处理自己负责槽位的命令，故障转移完成



#### 选举新的master

1. slave发现master下线（FAIL）
2. 将clusterState.currentEpoch加1，广播FAILOVER_AUTH_REQUEST（拉票）信息
3. 其他master判断合法性，发送FAILOVER_AUTH_ACK（投票），且对每一个Epoch只发送一次ack
4. slave收集FAILOVER_AUTH_ACK，超过半数master投票给自己则成为新master（master数量/2+1）
5. 广播Pong给集群其他节点告诉自己是master
6. 之前下线的master的所有slave的master指针指向新的master
7. 如果之前下线的master恢复，则自动成为新master的slave



### 消息

发送消息的节点称为：sender，接收消息的节点称为：receiver

消息有以下5种：

1. meet：receiver接收到cluster meet消息后，加入发送消息的节点集群中
2. ping：集群内每个节点每隔一秒会挑选5个节点，在这5个节点中找出最久没有发送ping的节点，并向该节点发送ping，以此来检测节点是否在线
3. pong：当receiver收到sender的ping或meet，则应该返回pong消息告诉sender已经接收到消息，此外一个节点也可以广播pong给集群其他节点刷新对这个节点的认知，比如故障转移后pong告知其他节点自己成为master了
4. fail：当一个节点A发现另一个节点B已经下线，则A会广播一条fail消息告知其他节点B已下线
5. publsh：当一个receiver收到publsh命令时，该节点会执行此命令，并广播一条publsh消息给其他节点，其他节点也会执行此命令

一条消息由消息头（header）和消息正文（data）组成
