---
title:      "iOS cache_t结构分析" 
date:       2020-09-18
tags:
    - iOS底层原理
categories:
    - iOS底层原理
---

#### 1. `cache_t`源码结构  

精简后的`cache_t`源码如下：
```OBJC
struct cache_t {
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED
    // MAC
    struct bucket_t * _buckets;
    uint32_t _mask;
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
    //真机
    uintptr_t _maskAndBuckets;
    mask_t _mask_unused;
    
    static constexpr uintptr_t maskShift = 48;
    static constexpr uintptr_t maskZeroBits = 4;
    static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - maskShift)) - 1;
    static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << (maskShift - maskZeroBits)) - 1;
#else
#error Unknown cache mask storage type.
#endif
    
    uint16_t _flags;
    uint16_t _occupied;//占用的个数

public:
    static bucket_t *emptyBuckets();
    struct bucket_t *buckets();
    uint32_t mask();
    uint32_t occupied();
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, uint32_t newMask);
    void initializeToEmpty();

    unsigned capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();
};
```
```
void cache_t::incrementOccupied() 
{
    _occupied++;
}
```

可以看到有个重要的函数`void incrementOccupied();`，字面意思增加占用的个数，该函数只对内部成员变量加1操作。在源码中全局搜索调用，即可看到在`cache_t::insert`函数中调用。

```
ALWAYS_INLINE
void cache_t::insert(Class cls, SEL sel, IMP imp, id receiver)
{ 
    // Use the cache as-is if it is less than 3/4 full
    //occupied() 获取_occupied变量
    mask_t newOccupied = occupied() + 1;//因为要新增一个缓存，所以+1
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    if (slowpath(isConstantEmptyCache())) {
        // Cache is read-only. Replace it.
        //如果之前没有缓存，开辟大小为4的内存
        if (!capacity) capacity = INIT_CACHE_SIZE;
        reallocate(oldCapacity, capacity, /* freeOld */false);
    }
    else if (fastpath(newOccupied + CACHE_END_MARKER <= capacity / 4 * 3)) {
        // Cache is less than 3/4 full. Use it as-is.
        //如果新的占用个数加1  小于等于  capacity的四分之三 则继续
    }
    else {
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;
        //否则进行扩容到 capacity的二倍
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE;
        }
        reallocate(oldCapacity, capacity, true);
    }

    bucket_t *b = buckets();
    mask_t m = capacity - 1;
    mask_t begin = cache_hash(sel, m);//通过hash计算出要存放的index
    mask_t i = begin;

    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot because the
    // minimum size is 4 and we resized at 3/4 full.
    //确保计算出存放的index没有被占用，否则继续计算index
    do {
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(sel, imp, cls);
            return;
        }
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;
        }
    } while (fastpath((i = cache_next(i, m)) != begin));

    cache_t::bad_cache(receiver, (SEL)sel, cls);
}
```

#### 2. 模仿cache_t结构，进行探索
```
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits

struct bucket_t {
    SEL _sel;
    IMP _imp;
};

struct ls_cache_t {
    struct bucket_t * _buckets;
    mask_t _mask;
    uint16_t _flags;
    uint16_t _occupied;
};

//不影响cache_t获取，所以写个空结构体类型
struct ls_class_data_bits_t {

};

struct ls_objc_class {
    Class ISA;
    Class superclass;
    struct ls_cache_t cache;             // formerly cache pointer and vtable
    struct ls_class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};

void printCachet(Class pClass) {
    struct ls_objc_class *ls_class = (__bridge struct ls_objc_class *)(pClass);
    
    uint16_t occupied = ls_class->cache._occupied;
    mask_t mask = ls_class->cache._mask;
    
    printf("cache_t._occupied = %d cache_t._mask = %d \n", occupied, mask);
    for (int i = 0; i < mask; i++) {
        struct bucket_t bucket = ls_class->cache._buckets[i];
        printf("sel=%s  imp=%p\n", [NSStringFromSelector(bucket._sel) UTF8String], bucket._imp);
    }
    printf("----------------------------------\n");
}


int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        LSPerson *person = [LSPerson alloc];
        
        Class pClass = [LSPerson class];
        
        printCachet(pClass);
        [person eat];
        [person run];
        printCachet(pClass);
        [person sleep];
        [person smile];
        printCachet(pClass);
        

    }
    return 0;
}
```

```
//output
//不调用方法时，_occupied = 0， _mask = 0
cache_t._occupied = 0 cache_t._mask = 0 
----------------------------------
2020-09-17 17:43:11.080167+0800 Cache_t_Analyze[64281:11027140] -[LSPerson eat]
2020-09-17 17:43:11.080673+0800 Cache_t_Analyze[64281:11027140] -[LSPerson run]
//调用2个方法后 _occupied = 2，_mask = 4 - 1
cache_t._occupied = 2 cache_t._mask = 3 
sel=run  imp=0x2c90
sel=(null)  imp=0x0
sel=eat  imp=0x2ce0
----------------------------------
2020-09-17 17:43:11.080923+0800 Cache_t_Analyze[64281:11027140] -[LSPerson sleep]
2020-09-17 17:43:11.080973+0800 Cache_t_Analyze[64281:11027140] -[LSPerson smile]
//调用4个方法，由于在调用第三个方法时触发了扩容，清空之前的缓存，所以只缓存了调用的第三个和第四个方法，但容量扩容到了8
//所以 _occupied = 2  _mask = 8 - 1
cache_t._occupied = 2 cache_t._mask = 7 
sel=smile  imp=0x2c70
sel=(null)  imp=0x0
sel=(null)  imp=0x0
sel=(null)  imp=0x0
sel=(null)  imp=0x0
sel=(null)  imp=0x0
sel=sleep  imp=0x2c40
----------------------------------
Program ended with exit code: 0
```