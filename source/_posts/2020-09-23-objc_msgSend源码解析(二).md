---
title:      "objc_msgSend源码解析(二)" 
date:       2020-09-23
tags:
    - iOS底层原理
categories:
    - iOS底层原理
---

在[objc_msgSend源码解析(一)](https://www.jianshu.com/p/55daf526120d)中最后进入`_lookUpImpOrForward`函数调用
#### 1. `_lookUpImpOrForward`源码分析
```
_objc_msgForward_impcache->_objc_msgForward->_objc_forward_handler  

__attribute__((noreturn, cold)) void
objc_defaultForwardHandler(id self, SEL sel)
{
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
                object_getClassName(self), sel_getName(sel), self);
}
void *_objc_forward_handler = (void*)objc_defaultForwardHandler;

IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    const IMP forward_imp = (IMP)_objc_msgForward_impcache;
    IMP imp = nil;
    Class curClass;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    if (fastpath(behavior & LOOKUP_CACHE)) {
        imp = cache_getImp(cls, sel);
        if (imp) goto done_nolock;
    } 

    runtimeLock.lock();
 
    // TODO: this check is quite costly during process startup.
    checkIsKnownClass(cls);

    if (slowpath(!cls->isRealized())) {
        cls = realizeClassMaybeSwiftAndLeaveLocked(cls, runtimeLock);
        // runtimeLock may have been dropped but is now locked again
    }

    if (slowpath((behavior & LOOKUP_INITIALIZE) && !cls->isInitialized())) {
        cls = initializeAndLeaveLocked(cls, inst, runtimeLock);
    }

    runtimeLock.assertLocked();
    curClass = cls; 

    for (unsigned attempts = unreasonableClassCount();;) {
        // curClass method list.
        Method meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            imp = meth->imp;
            goto done;
        }

        if (slowpath((curClass = curClass->superclass) == nil)) {
            // No implementation found, and method resolver didn't help.
            // Use forwarding.
            imp = forward_imp;
            break;
        }

        // Halt if there is a cycle in the superclass chain.
        if (slowpath(--attempts == 0)) {
            _objc_fatal("Memory corruption in class list.");
        }

        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (slowpath(imp == forward_imp)) {
            break;
        }
        if (fastpath(imp)) {
            // Found the method in a superclass. Cache it in this class.
            goto done;
        }
    }

    // No implementation found. Try method resolver once.
    if (slowpath(behavior & LOOKUP_RESOLVER)) {
        behavior ^= LOOKUP_RESOLVER;
        return resolveMethod_locked(inst, sel, cls, behavior);
    }

 done:
    log_and_fill_cache(cls, imp, sel, inst, curClass);
    runtimeLock.unlock();
 done_nolock:
    if (slowpath((behavior & LOOKUP_NIL) && imp == forward_imp)) {
        return nil;
    }
    return imp;
}
```
> 分析：
> 1. 首先还是会在当前类`curClass`的缓存中查找一遍，避免由于多线程问题，此时已经将该方法放进缓存
> 2. 然后进行类的一系列准备及确认工作
> 3. 此时的for循环只要没有跳出循环，则相当于死循环，在for循环中先调用`getMethodNoSuper_nolock`方法，从当前类`curClass`的`methodlist`中进行查找,如果找到则放到缓存中
> 4. 如果没有找到，则将当前类修改为当前类的父类，即`curClass=curClass->superclass`，如果父类为nil，则给imp赋值为`unrecognized selector`
> 5. 然后调用`cache_getImp`去父类的缓存中进行查找，`cache_getImp`在父类缓存中如果找到，则返回imp，并放入自己类的缓存中
> 6. 如果父类缓存中也没有找到，即在`cache_getImp`函数中没有找到imp，则继续循环，通过`getMethodNoSuper_nolock`函数查找父类的methodlist，如果没有找到则递归查找父类缓存，父类方法列表，直到找到对应的imp或者走到第4步
> 7. 在返回`unrecognized selector`之前，会走一次`resolveMethod_locked`动态方法解析的流程

#### 2. `getMethodNoSuper_nolock`源码分析
```
getMethodNoSuper_nolock -> search_method_list_inline -> findMethodInSortedMethodList

ALWAYS_INLINE static method_t *
findMethodInSortedMethodList(SEL key, const method_list_t *list)
{
    ASSERT(list);

    const method_t * const first = &list->first;
    const method_t *base = first;
    const method_t *probe;
    uintptr_t keyValue = (uintptr_t)key;
    uint32_t count;
    
    for (count = list->count; count != 0; count >>= 1) {
        probe = base + (count >> 1);
        
        uintptr_t probeValue = (uintptr_t)probe->name;
        
        if (keyValue == probeValue) { 
            while (probe > first && keyValue == (uintptr_t)probe[-1].name) {
                probe--;
            }
            return (method_t *)probe;
        }
        
        if (keyValue > probeValue) {
            base = probe + 1;
            count--;
        }
    }
    
    return nil;
}
```
> 分析：此处采用二分查找方式，由于`method`在`methodlist`中存放的特点，如果找到`sel`对应的`method`，则会检查该`method`的前一个`method`的`name`是否相同，直到找到第一个为止并返回，如果没有找到则返回nil