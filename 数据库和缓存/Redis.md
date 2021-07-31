# Redis

## 应用场景
 - 数据缓存
 - 分布式锁 redission/lua脚本
 - 消息队列 
 - 数据持久 `AOF/RDB`
 - 事务 multi()开启,exec()提交事务.

## redis为什么快?
- 单线程
- 内存操作
  - 内存为什么比硬盘快
    - 内存被CPU直接管理
    - 内存是电存储,硬盘是磁存储,电是瞬间到达比磁头转动快
- 非阻塞IO多路复用
## 缓存处理流程
前台请求,先从缓存中获取数据,取到直接返回结果.取不到从数据库中取,数据库取到则更新缓存,取不到直接空结果.

## 穿透/击穿/雪崩
| 类型 | 含义 | 解决方法|
| ---- | ---- | ---- |
| 击穿 | 访问缓存不存在但数据库存在的数据 | 设置永不过期
| 穿透 | 访问缓存数据库都不存在的数据 | 访问数据库存在但缓存不存在的数据 | 设置校验或者缓存写入key为null,设置一个短时间例如30秒,防止用户暴力攻击/布隆过滤器
| 雪崩 | 某一时刻key大面积时间过期 | 设置随机过期时间

## Redis数据结构类型
| 类型 | 对应Java | 含义 |
| ---- | ---- | ---- | 
| String  | String | 字符串| 
| Hash  | HashMap | 键值均为String的
| List | List< String > | 有序的,压缩列表(ZipList)或者链表(LinkedList)|
| Set  | Set< String > | 不能重复且无序,类似Hashset |
| zset  | Sort Set< String > |压缩列表(zipList)或者跳表(SkipList) |

## redis 淘汰策略
| 类型 | 含义 | 
| ---- | ---- |
| volatile-lru | 从已设置过期时间的数据集中挑选最近最少使用的数据淘汰 |
| volatile-ttl | 从已设置过期时间的数据集中挑选将要过期的数据淘汰 |
| volatile-random | 从已设置过期时间的数据集中任意选择数据淘汰 | 
| allkeys-lru | 从数据集中挑选最近最少使用的数据淘汰 | 
| allkeys-random | 从数据集中任意选择数据淘汰 | 
| no-enviction | 不淘汰,返回错误 | 

## redis 持久化
- RBD: fork一个子进程隔一段时间生产快照,最大化Redis的性能.RDB write和RDB load.
- AOF: 每个通过写命令都通过write()函数写入文件.

## 简述redis源码自己看过部分:
 - redis源码使用C编写,Redis中的每个键值对的键和值都是一个redisObject。
```C

typedef struct redisObject {
 
    unsigned type:4; // 对象的类型，包括 
 
    unsigned encoding:4; // 对象具体数据结构
 
    unsigned lru:REDIS_LRU_BITS; // 最后一次访问时间
 
    int refcount; // 引用计数
 
    void *ptr; // 实际指针
 
} robj;

type:

#define OBJ_STRING 0
 
#define OBJ_LIST 1
 
#define OBJ_SET 2
 
#define OBJ_ZSET 3
 
#define OBJ_HASH 4

encoding:

#define OBJ_ENCODING_RAW /* Raw representation */ 简单动态字符串
 
#define OBJ_ENCODING_INT /* Encoded as integer */ 整数
 
#define OBJ_ENCODING_HT /* Encoded as hash table */ 字典
 
#define OBJ_ENCODING_ZIPLIST /* Encoded as ziplist */ 压缩列表
 
#define OBJ_ENCODING_INTSET /* Encoded as intset */ 整数集合
 
#define OBJ_ENCODING_SKIPLIST /* Encoded as skiplist */ 跳跃表
 
#define OBJ_ENCODING_EMBSTR /* Embedded sds string encoding */ embstr编码的简单动态字符串
```
- 每种类型的对象至少都有两种或以上的编码方式


## Redisson - 单机 - RedssionLock
 - `lock()`
   - 获取锁,获取不到则阻塞在那里,直到其他线程释放锁.
   - 如果设置了时间,则在加锁指定时间后释放锁,其他线程可以尝试抢锁
   - 上锁时间一定要大于业务执行时间,否则其他线程会提前进入业务代码
 - `tryLock()`
   - 尝试获取锁,获取到返回true,获取不到返回false
   - 如果设置了时间,在指定时间内其他线程释放锁并且自己抢到返回true,在指定时间内其他线程没有释放锁返回false.
 - `lock()与锁续期watchDog`
   - lock()方法不显示指定上锁时间,则在加锁成功后开启守护线程.
    1. 每10秒给锁续期为30秒
    2. 如果持有锁的客户端没有宕机,就能保证一直持有锁,知道业务完成自动解锁
    3. 宕机在锁有效时间结束后自动解锁 lockWatchDogTimeOut     
  
## Redisson - 主从 - RedssionRedLock
 - `RLock在集群模式下的问题`
```text
1. 客户端A在Redis的master节点上拿到了锁；
2. master宕机，存储锁的key还没有来得及同步到Slave上；
3. master宕机引发故障转移，Slave节点升级为master节点；
4. 客户端B执行加锁操作，它可以从新的Master获取到锁；
这种情况下，客户端A和客户端B同时持有了同一个资源的锁。锁的安全性被打破了；所以就需要新的方式保证安全。
```
- RedissonRedLock 红锁
  - 获取少于N/2 + 1的锁则获取失败

## Redis更新数据失败了怎么办?
- 失败保存队列
