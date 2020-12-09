---
title:      "objc_msgSend源码解析(一)" 
date:       2020-09-20
tags:
    - iOS底层原理
categories:
    - iOS底层原理
---

[objc_msgSend源码解析(一)](https://www.jianshu.com/p/55daf526120d)
[objc_msgSend源码解析(二)](https://www.jianshu.com/p/7128dac1f0e8)

#### 1. `objc_msgSend`源码解析
在 objc4源码中全局搜索`objc_msgSend`可知该方法由汇编实现，在iOS平台下实现是在`objc-msg-arm64.s`文件中。

```
ENTRY _objc_msgSend 
//进入objc_msgSend流程

cmp	p0, #0			// nil check and tagged pointer check
//p0为第一个参数消息接收者，判断消息接收者是否为nil或者为taggedPointer类型

#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
    //如果为taggedPointer 则跳转到LNilOrTagged
#else
	b.eq	LReturnZero
    //如果为nil 则跳转到LReturnZero
#endif

ldr	p13, [x0]		// p13 = isa
//取出消息接收者的isa

	GetClassFromIsa_p16 p13		// p16 = class
//获取isa指向的class

LGetIsaDone:
	// calls imp or objc_msgSend_uncached
	CacheLookup NORMAL, _objc_msgSend
//调用CacheLookup的NORMAL模式

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
	b.eq	LReturnZero		// nil check
    //taggedPointer作为消息接收者第一步同样也是跳转到LReturnZero

	// tagged 为taggedPointer的特殊处理
	adrp	x10, _objc_debug_taggedpointer_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_classes@PAGEOFF
	ubfx	x11, x0, #60, #4
	ldr	x16, [x10, x11, LSL #3]
	adrp	x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGE
	add	x10, x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGEOFF
	cmp	x10, x16
	b.ne	LGetIsaDone

	// ext tagged
	adrp	x10, _objc_debug_taggedpointer_ext_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_ext_classes@PAGEOFF
	ubfx	x11, x0, #52, #8
	ldr	x16, [x10, x11, LSL #3]
	b	LGetIsaDone
// SUPPORT_TAGGED_POINTERS
#endif

LReturnZero:
    //消息接收者为nil或者taggedPointer类型都是直接return
	// x0 is already zero
	mov	x1, #0
	movi	d0, #0
	movi	d1, #0
	movi	d2, #0
	movi	d3, #0
	ret

	END_ENTRY _objc_msgSend
```
> 分析：
> 1. p0为第一个参数消息接收者，判断消息接收者是否为nil或者为taggedPointer类型
> 2. 如果为nil或者taggedPointer 则跳转到LReturnZero，直接return
> 3. 如果不为nil或taggedPointer，则将消息接收者第一个地址的isa取出，赋值给p13
> 4. 将p13即消息接收者的isa作为参数，调用`GetClassFromIsa_p16`，获取isa指向的类或者元类
> 5. 然后调用`CacheLookup`函数的NORMAL模式

![objc_msgSend.png](https://upload-images.jianshu.io/upload_images/1395687-d6d9b57eee1f2ee3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 2. `GetClassFromIsa_p16`源码解析
```
.macro GetClassFromIsa_p16 /* src */

#if SUPPORT_INDEXED_ISA
	// Indexed isa
	mov	p16, $0			// optimistically set dst = src
	tbz	p16, #ISA_INDEX_IS_NPI_BIT, 1f	// done if not non-pointer isa
	// isa in p16 is indexed
	adrp	x10, _objc_indexed_classes@PAGE
	add	x10, x10, _objc_indexed_classes@PAGEOFF
	ubfx	p16, p16, #ISA_INDEX_SHIFT, #ISA_INDEX_BITS  // extract index
	ldr	p16, [x10, p16, UXTP #PTRSHIFT]	// load class from array
1:

//在64位下根据传入的第一个参数 即 isa & ISA_MASK 取出class
#elif __LP64__
	// 64-bit packed isa
	and	p16, $0, #ISA_MASK

#else
	// 32-bit raw isa
	mov	p16, $0

#endif

.endmacro
```
> 分析： 在64位下根据传入的第一个参数 即 isa & ISA_MASK 取出class

#### 3. `CacheLookup`源码解析
```
#define PTRSHIFT 3  // 1<<PTRSHIFT == PTRSIZE

.macro CacheLookup

LLookupStart$1:

	// p1 = SEL, p16 = isa
	ldr	p11, [x16, #CACHE]				// p11 = mask|buckets
//获取isa指向class中的cache_t cache结构中的mask|buckets

//在iOS下走以下分支
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
//将p11中存的mask|buckets & 0x0000ffffffffffff，获取低48位的为buckets
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets

//将p11中存的mask|buckets 逻辑右移48位，即将buckets顶出去，留下16位的mask；然后将获取到的mask & p1(_cmd)，获取到buckets数组中要查找的index，即index = _cmd & mask
	and	p12, p1, p11, LSR #48		// x12 = _cmd & mask
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	and	p10, p11, #~0xf			// p10 = buckets
	and	p11, p11, #0xf			// p11 = maskShift
	mov	p12, #0xffff
	lsr	p11, p12, p11				// p11 = mask = 0xffff >> p11
	and	p12, p1, p11				// x12 = _cmd & mask
#else
#error Unsupported cache mask storage for ARM64.
#endif


	add	p12, p10, p12, LSL #(1+PTRSHIFT)
		             // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))
//(1+PTRSHIFT) = (1+3) = 4，由于一个bucket结构包含 IMP _imp，SEL _sel，即16位，所以((_cmd & mask) << (1+PTRSHIFT))含义为index * 16，buckets + ((_cmd & mask) << (1+PTRSHIFT))含义为获取到index位置bucket的首地址赋值到p12

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
//在iOS下bucket结构为{imp, sel}，mac下为{sel, imp}

1:	cmp	p9, p1			// if (bucket->sel != _cmd)
//比较bucket的sel和所需查找的_cmd是否相等

	b.ne	2f			//     scan more
//如果不相等 跳转到2

	CacheHit $0			// call or return imp
//如果缓存命中，则直接返回imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
//首先检查bucket的_sel是否等于0，如果等于0，跳转到CheckMiss

	cmp	p12, p10		// wrap if bucket == buckets
//判断获取的bucket是否是buckets中的第一个

	b.eq	3f
//如果遍历到数组的首个元素，则跳转到3，之后不会返回来继续，否则形成死循环

	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
//如果不是数组首个元素，则取得前一个bucket的imp和sel，继续进行比较

	b	1b			// loop

3:	// wrap: p12 = first bucket, w11 = mask
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	add	p12, p12, p11, LSR #(48 - (1+PTRSHIFT))
					// p12 = buckets + (mask << 1+PTRSHIFT)
//mask为数组个数capacity减一，(mask << 1+PTRSHIFT)为mask * 16，即获取数组最后一个bucket赋值给p12

#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	add	p12, p12, p11, LSL #(1+PTRSHIFT)
					// p12 = buckets + (mask << 1+PTRSHIFT)
#else
#error Unsupported cache mask storage for ARM64.
#endif

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

LLookupEnd$1:
LLookupRecover$1:
3:	// double wrap
	JumpMiss $0

.endmacro
```
> 分析：
> 1. 找到isa指向的类，偏移16字节后取到cache_t cache元素的首地址，首地址在iOS下为`_maskAndBuckets`赋值到p11
> 2. p11 & 0x0000ffffffffffff，得到低48位的buckets
> 3. p11逻辑右移48位，得到mask，mask & _cmd得到数组buckets起始查找位置的index
> 4. index << 4，即index * 16字节，获取到index所指向bucket的首地址p12
> 5. p17, p9 分别为imp，sel
>  ///第一步
> 6. 比较p9和p1(即传入的第二个参数，所要查找的SEL _cmp)
> 7. 如果不想等，则跳转到2f，如果相等则缓存命中，返回sel所对应的函数指针地址imp
> ///第二步
> 8. 首先检查bucket的sel是否等于0，如果等于0则跳转到`CheckMiss`
> 9. 如果bucket的sel不等于0，判断bucket是否为数组第一个元素，如果是，则跳转到第三步，之后不会走下一步，而是直接走第三步之后的代码，否则形成死循环
> 10. 如果bucket不是数组第一个元素，则获取前一个bucket，将新的imp和sel赋值到p17和p9，并跳转到第一步进行循环

> ///第三步
> 11. mask为数组buckets元素个数(capacity-1)，mask << 4，即获取数组最后一个元素bucket的首地址，赋值给p12

> 12.之后的第一步和上面第一步一样，第二步如果走到数组buckets的第一个元素仍然没有找到，则直接跳转到第三步即`JumpMiss`

![cachelookup.png](https://upload-images.jianshu.io/upload_images/1395687-45650706f1f09463.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 4. `CheckMiss`源码分析
```
.macro CheckMiss
	// miss if bucket->sel == 0
.if $0 == GETIMP
	cbz	p9, LGetImpMiss
.elseif $0 == NORMAL
	cbz	p9, __objc_msgSend_uncached
.elseif $0 == LOOKUP
	cbz	p9, __objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro
```
> 分析：此时为NORMAL模式，则进入`__objc_msgSend_uncached`流程

#### 5. `__objc_msgSend_uncached`源码分析
```
STATIC_ENTRY __objc_msgSend_uncached
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band p16 is the class to search
	
	MethodTableLookup
	TailCallFunctionPointer x17

	END_ENTRY __objc_msgSend_uncached
```
> 分析：此时快速缓存查找没有找到对应imp，调用`MethodTableLookup`函数中调用`_lookUpImpOrForward`，进入对应类或元类的methodlist查找流程