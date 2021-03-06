## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## LRU机制相关的数据结构属性
redis 内存使用量超过某个阈值时,会触发淘汰机制,将原本有效的还没过期的空间强行释放
### redisObject 的 lru属性
```c
// src/server.h
#define LRU_BITS 24

// redis的值无论什么类型,都是保存在一个redisObject结构里

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    // 这个lru 即可用来保存时间,也可用来保存LFU策略的使用频率
    // 注:LFU策略既需要保存频数,也需要保存一个上次更新时间,
    // 当需要给LFU频数+1前,先判断上次更新时间是否够久远,如果太久远了,需要减少相应的频数,避免以前很频繁现在没人用的object一直占坑
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

### lru(lfu)的更新
```c
// src/db.c
// 几乎所有操作都是要先找到key所代表的redisObject
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        if (!hasActiveChildProcess() && !(flags & LOOKUP_NOTOUCH)){
            // 根据策略的不同更新LFU值或lru值
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                updateLFU(val);
            } else {
                val->lru = LRU_CLOCK();
            }
        }
        return val;
    } else {
        return NULL;
    }
}

// LFU的更新
void updateLFU(robj *val) {
    // 根据上次更新的时间的久远情况,减去相应的频数
    unsigned long counter = LFUDecrAndReturn(val);
    // 增加频数并记录时间
    counter = LFULogIncr(counter);
    val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}

// 获取LRU需要的当前时间值(由调用方再去设置lru的值)
unsigned int LRU_CLOCK(void) {
    unsigned int lruclock;
    if (1000/server.hz <= LRU_CLOCK_RESOLUTION) {
        lruclock = server.lruclock;
    } else {
        lruclock = getLRUClock();
    }
    return lruclock;
}
```

### 触发淘汰机制
```c
// src/server.c
// redis所有命令的执行都需要经过processCommand,所以在这里判断是否需要执行淘汰机制
int processCommand(client *c) {
    // ...
    if (server.maxmemory && !server.lua_timedout) {
        int out_of_memory = freeMemoryIfNeededAndSafe() == C_ERR;
    }
    // ...
}

// src/evict.c
int freeMemoryIfNeededAndSafe(void) {
    if (server.lua_timedout || server.loading) return C_OK;
    return freeMemoryIfNeeded();
}

int freeMemoryIfNeeded(void) {
    int keys_freed = 0;
    // slave机器不需要
    if (server.masterhost && server.repl_slave_ignore_maxmemory) return C_OK;

    // ...
    // 获取当前的内存使用情况,确定是否需要触发淘汰机制;
    // 如果需要触发淘汰机制,将此次打算释放的内存数量写入mem_tofree
    if (getMaxmemoryState(&mem_reported,NULL,&mem_tofree,NULL) == C_OK)
        return C_OK;

    // 到这里就代表已经触发淘汰策略,初始化释放的空间数量
    mem_freed = 0;

    // 记录下因为内存淘汰而多花的时间
    latencyStartMonitor(latency);
    if (server.maxmemory_policy == MAXMEMORY_NO_EVICTION) 
        goto cant_free; /* We need to free memory, but policy forbids. */


    while (mem_freed < mem_tofree) { // 当已经释放的空间小于预计想要释放的空间时,继续
        int j, k, i;
        static unsigned int next_db = 0;
        sds bestkey = NULL;
        int bestdbid;
        redisDb *db;
        dict *dict;
        dictEntry *de;

        if (server.maxmemory_policy & (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU) ||
            server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL)
        {
            struct evictionPoolEntry *pool = EvictionPoolLRU;

            while(bestkey == NULL) {
                unsigned long total_keys = 0, keys;

                // server的每个db都要轮一遍
                for (i = 0; i < server.dbnum; i++) {
                    db = server.db+i;
                    dict = (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) ?
                            db->dict : db->expires;
                    if ((keys = dictSize(dict)) != 0) {
                        // 往淘汰池子里加入候选robj
                        evictionPoolPopulate(i, dict, db->dict, pool);
                        total_keys += keys;
                    }
                }
                if (!total_keys) break; /* No keys to evict. */

                // 遍历淘汰候选池,找到第一个有效的key,选为bestKey,然后break循环
                for (k = EVPOOL_SIZE-1; k >= 0; k--) {
                    if (pool[k].key == NULL) continue;
                    bestdbid = pool[k].dbid;

                    if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
                        de = dictFind(server.db[pool[k].dbid].dict,
                            pool[k].key);
                    } else {
                        de = dictFind(server.db[pool[k].dbid].expires,
                            pool[k].key);
                    }

                    // 移除池子里的key信息
                    if (pool[k].key != pool[k].cached)
                        sdsfree(pool[k].key);
                    pool[k].key = NULL;
                    pool[k].idle = 0;

                    
                    if (de) {
                        // 当key对应的内存还有效时,选为bestkey,跳出循环
                        bestkey = dictGetKey(de);
                        break;
                    } else {
                        /* Ghost... Iterate again. */
                    }
                }
            }
        }

        // 淘汰策略是随机时的逻辑
        else if (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM ||
                 server.maxmemory_policy == MAXMEMORY_VOLATILE_RANDOM)
        {
            // 直接遍历所有db,随机取一个有效的key做bestkey,触发后面的evic逻辑
            for (i = 0; i < server.dbnum; i++) {
                j = (++next_db) % server.dbnum;
                db = server.db+j;
                dict = (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM) ?
                        db->dict : db->expires;
                if (dictSize(dict) != 0) {
                    de = dictGetRandomKey(dict);
                    bestkey = dictGetKey(de);
                    bestdbid = j;
                    break;
                }
            }
        }

        /* Finally remove the selected key. */
        if (bestkey) {
            db = server.db+bestdbid;
            robj *keyobj = createStringObject(bestkey,sdslen(bestkey));

            // 主从模式时需要把淘汰的这个行为传递给下线
            propagateExpire(db,keyobj,server.lazyfree_lazy_eviction);

            // ...
            // 根据配置决定 同步执行淘汰机制还是异步执行淘汰机制
            if (server.lazyfree_lazy_eviction)
                // 异步删除
                dbAsyncDelete(db,keyobj);
            else
                // 同步删除
                dbSyncDelete(db,keyobj);
            
            // 手动触发修改key时的hook方法们
            signalModifiedKey(NULL,db,keyobj);
            
            // 统计释放的空间
            delta -= (long long) zmalloc_used_memory();
            mem_freed += delta;
            server.stat_evictedkeys++;
            notifyKeyspaceEvent(NOTIFY_EVICTED, "evicted",
                keyobj, db->id);
            decrRefCount(keyobj);
            keys_freed++;

            //...
            // 当淘汰触发是异步执行时,需要把预计能省出的空间加到mem_freed上
            if (server.lazyfree_lazy_eviction && !(keys_freed % 16)) {
                if (getMaxmemoryState(NULL,NULL,NULL,NULL) == C_OK) {
                    mem_freed = mem_tofree;
                }
            }
        } else {
            goto cant_free; /* nothing to free... */
        }
    }
    result = C_OK;

cant_free:
    // 当遇到不能成功释放空间时需要记录下便于debug
    if (result != C_OK) {
        latencyStartMonitor(lazyfree_latency);
        while(bioPendingJobsOfType(BIO_LAZY_FREE)) {
            if (getMaxmemoryState(NULL,NULL,NULL,NULL) == C_OK) {
                result = C_OK;
                break;
            }
            usleep(1000);
        }
        latencyEndMonitor(lazyfree_latency);
        latencyAddSampleIfNeeded("eviction-lazyfree",lazyfree_latency);
    }
    latencyEndMonitor(latency);
    latencyAddSampleIfNeeded("eviction-cycle",latency);
    return result;
}
```
### 淘汰池
#### 初始化淘汰池
redis 在initServer时初始化一个实现key的LRU机制的pool
```c
// src/server.c
void initServer(void) {
    // ...
    evictionPoolAlloc(); /* Initialize the LRU keys pool. */
    // ...

}
```
```c
// src/evict.c
void evictionPoolAlloc(void) {
    struct evictionPoolEntry *ep;
    int j;

    ep = zmalloc(sizeof(*ep)*EVPOOL_SIZE);
    for (j = 0; j < EVPOOL_SIZE; j++) {
        ep[j].idle = 0;
        ep[j].key = NULL;
        ep[j].cached = sdsnewlen(NULL,EVPOOL_CACHED_SDS_SIZE);
        ep[j].dbid = 0;
    }
    EvictionPoolLRU = ep;
}
```
#### 淘汰池新加入值
```c
// src/evict.c
void evictionPoolPopulate(int dbid, dict *sampledict, dict *keydict, struct evictionPoolEntry *pool) {
    int j, k, count;
    dictEntry *samples[server.maxmemory_samples];

    // TODO 随机获取一些样例空间(直接获取的就是dictEntry所在位置)
    count = dictGetSomeKeys(sampledict,samples,server.maxmemory_samples);
    // 遍历这些样例数据
    for (j = 0; j < count; j++) {
        unsigned long long idle;
        sds key;
        robj *o;
        dictEntry *de;

        de = samples[j];
        key = dictGetKey(de);

        if (server.maxmemory_policy != MAXMEMORY_VOLATILE_TTL) {
            // 找到数据存放的实际位置
            if (sampledict != keydict) de = dictFind(keydict, key);
            o = dictGetVal(de);
        }

        // 区分不同的淘汰策略,根绝策略的不同计算当前 robj的idle(分数),分数越高表明越有淘汰潜力
        if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
            idle = estimateObjectIdleTime(o);
        } else if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
            idle = 255-LFUDecrAndReturn(o);
        } else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
            idle = ULLONG_MAX - (long)dictGetVal(de);
        } else {
            serverPanic("Unknown eviction policy in evictionPoolPopulate()");
        }

        // 在淘汰池中安idle分数高低找到合适自己的位置,按idle分数排,或者找到了第一个空位
        k = 0;
        while (k < EVPOOL_SIZE &&
               pool[k].key &&
               pool[k].idle < idle) k++;
        if (k == 0 && pool[EVPOOL_SIZE-1].key != NULL) {
            // 当前idle分数比池子中第一个robj小,且池子最后一个元素也已经有值(这应该近似代表当前的idle分数太低,不配淘汰),则直接跳过本次循环
            continue;
        } else if (k < EVPOOL_SIZE && pool[k].key == NULL) {
            // 找到空位了,后续直接插入就行
        } else {
            // 为当前idle分数寻找合适位置时,停在了一个本身有值的位置上
            if (pool[EVPOOL_SIZE-1].key == NULL) {
                // 如果池子末尾还有空间,直接将当前位置及其后面的值往池子后面移一个身位,空出当前位置
                sds cached = pool[EVPOOL_SIZE-1].cached;
                memmove(pool+k+1,pool+k,
                    sizeof(pool[0])*(EVPOOL_SIZE-k-1));
                pool[k].cached = cached;
            } else {
                // 池子末尾没有空间了,移除池子最左边的robj,然后把当前位置左边的数据全部往左移一个身位,空出当前位置
                k--;
                
                sds cached = pool[0].cached; /* Save SDS before overwriting. */
                if (pool[0].key != pool[0].cached) sdsfree(pool[0].key);
                memmove(pool,pool+1,sizeof(pool[0])*k);
                pool[k].cached = cached;
            }
        }

        // 将当前robj的数据写入池子的当前位置
        int klen = sdslen(key);
        if (klen > EVPOOL_CACHED_SDS_SIZE) {
            pool[k].key = sdsdup(key);
        } else {
            memcpy(pool[k].cached,key,klen+1);
            sdssetlen(pool[k].cached,klen);
            pool[k].key = pool[k].cached;
        }
        pool[k].idle = idle;
        pool[k].dbid = dbid;
    }
}
```

### 释放淘汰池里的空间
#### 异步 dbAsyncDelete
```c
// src/lazyfree.c
#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {
    // 先删除expires db中的信息
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

    // 释放dict里dictEntry中的信息,但具体存放存放数据的robj 和 dictEntry本身并没有释放,后面还需要用到
    // 注:这个dictUnlink 和 后面的 dictFreeUnlinkedEntry需要配合着用
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        // 算出要释放的空间的工作量
        size_t free_effort = lazyfreeGetFreeEffort(val);

        // 当工作量大于 LAZYFREE_THRESHOLD 阈值时,触发lazyFree,不然就直接同步释放就行
        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1);
            // 将工作量加入到异步处理队列中,由其他线程处理
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            // 将当前entry的值置为NULL,后面就不会再调起同步free了
            dictSetVal(db->dict,de,NULL);
        }
    }

    // 如果没有达到异步free的阈值,这里就会调用同步free,删除键,值,释放空间
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key->ptr);
        return 1;
    } else {
        return 0;
    }
}
```
#### 同步 dbSyncDelete
```c
// src/db.c
int dbSyncDelete(redisDb *db, robj *key) {
    // 同步DELETE直接删除键和值,释放空间
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    if (dictDelete(db->dict,key->ptr) == DICT_OK) {
        if (server.cluster_enabled) slotToKeyDel(key->ptr);
        return 1;
    } else {
        return 0;
    }
}
```
