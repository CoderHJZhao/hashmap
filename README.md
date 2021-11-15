# 关于散列表的研究以及NSDictionary底层探索

> 众所周知，在iOS开发当中便利一个数组，去不断匹配value是一种很低效的方式，但是如果直接通过NSArry的index（索引）去取值，则会快很多，但是字典是无序的，也就是没有一个对外暴露的索引，我们该如何通过字典的key去快速定位Value呢？

#### 在了解这些之前，我们就必须要了解散列表，即哈希表（Hash table）。

> 散列表（Hash table，也叫哈希表），是根据键（Key）而直接访问在内存存储位置的数据结构。也就是说，它通过计算一个关于键值的函数，将所需查询的数据映射到表中一个位置来访问记录，这加快了查找速度。这个映射函数称做散列函数，存放记录的数组称做散列表。

#### 所以我们知道哈希表底层其实是一个数组，那么只要我们通过一定的算法将key值转化为数组中index和value的对应关系就可以极大的提升字典差值的效率了。

## 话不多说，开搞！

> 已知Key对象的值为0b0000 0001(计算机储存的所有对象都为二进制数据)，当我们设定这个key的时候，我们会通过一个hash函数去生成一个特定索引。这个函数可能是用“&”与运算符，也可能用“%”取余，不同平台的算法各不相同，但都是相通的。
> 
> 比如我们定义一个mask，值为：0b0000 1111（取余即为15）；我们用key1 & mask，如下：

```
//通过“&”运算符，我们得到一个明确的索引，即1。
  0b0000 0001
& 0b0000 1111
--------------
  0b0000 0001
```
哈希表对应分布如下：

索引 | value
---|---
0 | NULL
1 | buket(key1,value2)
2 | NULL
3 | NULL
4 | NULL
5 | NULL

> 如果我们继续往字典储存新值，如key2为：0b0000 0010

```
//通过“&”运算符，我们得到一个明确的索引，即2。
  0b0000 0010
& 0b0000 1111
--------------
  0b0000 0010
```
哈希表对应分布如下：

索引（地址） | value
---|---
0 | NULL
1 | buket(key1,value2)
2 | buket(key2,value2)
3 | NULL
4 | NULL
5 | NULL

> 以此类推，我们将会得到一个Array，所以了解了这些后，我们很容易就明白，当我们传入一个key时，字典并不是通过key去循环便利去找出我们想要的value，而是通过一个函数去生成value的一个索引，从而更精准的定位我们想要的value，如传入：（0b0000 0001），我们就得到了索引为1，再通过索引1，直接拿到Value。


## 那么新问题产生了！

> 在上面我们定义了一个mask为0b0000 1111的值，根据“&”运算的逻辑，值都为1才为1，只要其中有一个为0，则为0。所以我们得到的索引最大值都不会超过mask，即15。即数组内储存的元素最多不能超过15条。

1. 那么一旦超过呢？没错，字典底层为我们编写了“扩容”的操作。（源码如下）
2. 如果生成的索引index已经被占用了呢？这时候iOS底层源码编写了向前/向后查找，直到查找到存在空位时，储存（但是实际开发中，冲突很麻烦，甚至会影响散列表的查找速度，所以应该采用最大限度减少冲突的散列函数）。

下面时iOS中字典扩容的源码，有兴趣的可以看一下。
[iOS CFDictionary源码](https://opensource.apple.com/source/CF/CF-368/Collections.subproj/CFDictionary.c.auto.html)


```
//字典结构体
struct __CFDictionary {
    CFRuntimeBase _base;
    CFIndex _count;		/* number of values */
    CFIndex _capacity;		/* maximum number of values */
    CFIndex _bucketsNum;	/* number of slots */
    uintptr_t _marker;
    void *_context;		/* private */
    CFIndex _deletes;
    CFOptionFlags _xflags;      /* bits for GC */
    //保留的所有的key
    const void **_keys;		/* can be NULL if not allocated yet */
    //保存了所有value
    const void **_values;	/* can be NULL if not allocated yet */
};
```


```
static void __CFDictionaryGrow(CFMutableDictionaryRef dict, CFIndex numNewValues) {
    //获得原有的keys和values
    const void **oldkeys = dict->_keys;
    const void **oldvalues = dict->_values;
    //原来插槽数
    CFIndex idx, oldnbuckets = dict->_bucketsNum;
    //原来字典中的values
    CFIndex oldCount = dict->_count;
    CFAllocatorRef allocator = __CFGetAllocator(dict), keysAllocator, valuesAllocator;
    void *keysBase, *valuesBase;
    dict->_capacity = __CFDictionaryRoundUpCapacity(oldCount + numNewValues);
    dict->_bucketsNum = __CFDictionaryNumBucketsForCapacity(dict->_capacity);
    dict->_deletes = 0;
    if (_CFDictionaryIsSplit(dict)) {   // iff GC, use split memory sometimes unscanned memory
    unsigned weakOrStrong = (dict->_xflags & __kCFDictionaryWeakKeys) ? AUTO_MEMORY_UNSCANNED : AUTO_MEMORY_SCANNED;
    void *mem = _CFAllocatorAllocateGC(allocator, dict->_bucketsNum * sizeof(const void *), weakOrStrong);
        CF_WRITE_BARRIER_BASE_ASSIGN(allocator, dict, dict->_keys, mem);
        keysAllocator = (dict->_xflags & __kCFDictionaryWeakKeys) ? kCFAllocatorNull : allocator;  // GC: avoids write-barrier in weak case.
        keysBase = mem;
    
        weakOrStrong = (dict->_xflags & __kCFDictionaryWeakValues) ? AUTO_MEMORY_UNSCANNED : AUTO_MEMORY_SCANNED;
    mem = _CFAllocatorAllocateGC(allocator, dict->_bucketsNum * sizeof(const void *), weakOrStrong);
        CF_WRITE_BARRIER_BASE_ASSIGN(allocator, dict, dict->_values, mem);
        valuesAllocator = (dict->_xflags & __kCFDictionaryWeakValues) ? kCFAllocatorNull : allocator; // GC: avoids write-barrier in weak case.
        valuesBase = mem;
    } else {
        //扩容成原先大小的两倍
        CF_WRITE_BARRIER_BASE_ASSIGN(allocator, dict, dict->_keys, _CFAllocatorAllocateGC(allocator, 2 * dict->_bucketsNum * sizeof(const void *), AUTO_MEMORY_SCANNED));
        dict->_values = (const void **)(dict->_keys + dict->_bucketsNum);
        keysAllocator = valuesAllocator = allocator;
        keysBase = valuesBase = dict->_keys;
    }
    if (NULL == dict->_keys || NULL == dict->_values) HALT;
    if (__CFOASafe) __CFSetLastAllocationEventName(dict->_keys, "CFDictionary (store)");
    // 重新计算keys数据的hash值，存放到新的数组中
    for (idx = dict->_bucketsNum; idx--;) {
        dict->_keys[idx] = (const void *)dict->_marker;
        dict->_values[idx] = 0;
    }
    if (NULL == oldkeys) return;
    for (idx = 0; idx < oldnbuckets; idx++) {
        if (dict->_marker != (uintptr_t)oldkeys[idx] && ~dict->_marker != (uintptr_t)oldkeys[idx]) {
            CFIndex match, nomatch;
            __CFDictionaryFindBuckets2(dict, oldkeys[idx], &match, &nomatch);
            CFAssert3(kCFNotFound == match, __kCFLogAssertion, "%s(): two values (%p, %p) now hash to the same slot; mutable value changed while in table or hash value is not immutable", __PRETTY_FUNCTION__, oldkeys[idx], dict->_keys[match]);
            if (kCFNotFound != nomatch) {
                CF_WRITE_BARRIER_BASE_ASSIGN(keysAllocator, keysBase, dict->_keys[nomatch], oldkeys[idx]);
                CF_WRITE_BARRIER_BASE_ASSIGN(valuesAllocator, valuesBase, dict->_values[nomatch], oldvalues[idx]);
            }
        }
    }
    CFAssert1(dict->_count == oldCount, __kCFLogAssertion, "%s(): dict count differs after rehashing; error", __PRETTY_FUNCTION__);
    _CFAllocatorDeallocateGC(allocator, oldkeys);
    if (_CFDictionaryIsSplit(dict)) _CFAllocatorDeallocateGC(allocator, oldvalues);
}

```

> 但是扩容有一个很明显的缺点，就是会让字典的容量越来越大，甚至产生很多空间的浪费。所以哈希表是一种用空间换效率的技术。（会空闲一部分内存）

> 总结一下，所以散列表的核心是散列函数，大致可以用如下函数表示，不管是用“%”取余，还是用与运算符“&”，我们所定义的mask都决定了散列表的容量大小，所以应尽可能优化散列函数，避免mask的频繁变动和散列表数据的冲突，保证散列表查询的效率。以上就是我对于散列表的理解和研究，如果错漏，还请大家踊跃提出，共同交流。

```
f(key) = index
```

相关参考文档补充

[iOS字典底层原理：https://www.jianshu.com/p/0d7cd6341f65](https://note.youdao.com/)

[iOS 用OC简单还原hash值生成：https://juejin.cn/post/6844903608954126344](https://juejin.cn/post/6844903608954126344)

-----zhaohanjun


