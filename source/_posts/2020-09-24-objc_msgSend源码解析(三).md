---
title:      "objc_msgSend源码解析(三)" 
date:       2020-09-24
tags:
    - iOS底层原理
categories:
    - iOS底层原理
---

#### 1. 动态方法解析

`resolveMethod_locked`源码
```
static NEVER_INLINE IMP
resolveMethod_locked(id inst, SEL sel, Class cls, int behavior)
{
    runtimeLock.assertLocked();
    ASSERT(cls->isRealized());
    runtimeLock.unlock();
    if (! cls->isMetaClass()) { 
        resolveInstanceMethod(inst, sel, cls);
    } 
    else { 
        resolveClassMethod(inst, sel, cls);
        if (!lookUpImpOrNil(inst, sel, cls)) {
            resolveInstanceMethod(inst, sel, cls);
        }
    }
    return lookUpImpOrForward(inst, sel, cls, behavior | LOOKUP_CACHE);
}
```
> 分析：根据传入的`cls`是否是元类分别调用`resolveInstanceMethod`或者`resolveClassMethod`

对于类方法的动态解析示例
```
void myClassMethod() {
    NSLog(@"%s", __FUNCTION__);
}

+ (BOOL)resolveClassMethod:(SEL)sel {
    if ([NSStringFromSelector(sel) isEqualToString:@"classMethod"]) {
        const char *cls = [NSStringFromClass([self class]) UTF8String];
        Class metaClass = objc_getMetaClass(cls);
        
        return class_addMethod(metaClass, sel, (IMP)myClassMethod, "");
    }
    return [super resolveClassMethod:sel];
}
```
> 注意：
> 1. 对于类方法的动态解析需要给元类添加IMP
> 2. 由于NSObject的元类的父类为NSObject，所以也可以给NSObject的分类添加对应的实例方法

对于实例方法的动态解析示例
```
void myInstanceMethod() {
    NSLog(@"%s", __FUNCTION__);
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if ([NSStringFromSelector(sel) isEqualToString:@"instanceMethod"]) {        
        return class_addMethod(self, sel, (IMP)myInstanceMethod, "");
    }
    return [super resolveInstanceMethod:sel];
}
```

#### 2. 快速转发流程
在`lookUpImpOrForward`函数中如果找到sel对应的imp，则走到`log_and_fill_cache`这步，将找到的imp放入缓存。

`log_and_fill_cache`源码解析
```
static void
log_and_fill_cache(Class cls, IMP imp, SEL sel, id receiver, Class implementer)
{
#if SUPPORT_MESSAGE_LOGGING
    if (slowpath(objcMsgLogEnabled && implementer)) {
        bool cacheIt = logMessageSend(implementer->isMetaClass(), 
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(), 
                                      sel);
        if (!cacheIt) return;
    }
#endif
    cache_fill(cls, sel, imp, receiver);
}
```
可以跟进`logMessageSend`函数->`logMessageSend`->`instrumentObjcMessageSends`，在`instrumentObjcMessageSends`传入的flag即可以打印到文件。  
所以在外部定义函数如下
```
#import <Foundation/Foundation.h>
#import "LSPerson.h"

extern void instrumentObjcMessageSends(BOOL flag); 

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        LSPerson *person = [[LSPerson alloc] init];
        instrumentObjcMessageSends(true);
        [person performSelector:@selector(eat)];
        instrumentObjcMessageSends(false);
    }
    return 0;
}
```  
crash报错`unrecognized selector sent to instance`,然后由于是macos项目，则在`/tmp/msgSends-xxx`文件中可以看到如下调用顺序  
```
- LSPerson NSObject performSelector:
+ LSPerson NSObject resolveInstanceMethod:
+ LSPerson NSObject resolveInstanceMethod:
- LSPerson NSObject forwardingTargetForSelector:
- LSPerson NSObject forwardingTargetForSelector:
- LSPerson NSObject methodSignatureForSelector:
- LSPerson NSObject methodSignatureForSelector:
- LSPerson NSObject class
+ LSPerson NSObject resolveInstanceMethod:
+ LSPerson NSObject resolveInstanceMethod:
- LSPerson NSObject doesNotRecognizeSelector:
- LSPerson NSObject doesNotRecognizeSelector:
```
可以看到调用之后没有找到方法除了会走动态解析逻辑`resolveInstanceMethod`，还走了`forwardingTargetForSelector`和`methodSignatureForSelector`分别对应**快速转发流程**和**慢速转发流程**。

> 我理解的**快速转发流程**和**慢速转发流程**中的快慢是指在`objc_msgSend`中的先后顺序，并不是快速一定比慢速要快，或者效率更高。

> 通过反汇编查看`____forwarding___`流程猜测如果函数返回空或者`self`，仍然会报错
```
loc_64bdc:
    rdi = rbx;
    rax = [rdi forwardingTargetForSelector:var_140];
    if ((rax == 0x0) || (rax == rbx)) goto loc_64c47;
```

所以可以给自己类或者其他类添加sel对应的imp后返回
```
@implementation LSPerson

- (void)run{
    NSLog(@"%s", __FUNCTION__);
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"eat"]) {
        //获取LSPerson中的run方法
        Method method = class_getInstanceMethod([self class], @selector(run));
        const char *type = method_getTypeEncoding(method);
        IMP imp = method_getImplementation(method);
        //将aSelector指向run的imp
        class_addMethod([self class], aSelector, imp, type);
        return [LSPerson alloc];
    }
    return [super forwardingTargetForSelector:aSelector];
}
@end
```

#### 3. 慢速转发流程
```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(eat)) {
        NSMethodSignature *methodSignature = [NSMethodSignature signatureWithObjCTypes:"v@@:"];
        return methodSignature;
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL selector = anInvocation.selector;
    if ([[LSPerson alloc] respondsToSelector:selector]) {
        [anInvocation invokeWithTarget:[LSPerson alloc]];
    }else {
        [self doesNotRecognizeSelector:[anInvocation selector]];
    }
}

- (void) doesNotRecognizeSelector: (SEL)aSelector
{
    NSLog(@"%@ does not recognize %@",NSStringFromClass([self class]), NSStringFromSelector(aSelector));
}
```
> 分析：
> 1. 在`methodSignatureForSelector`这个函数中返回一个aSelector对应的`NSMethodSignature *`的函数签名
> 2. 在`forwardInvocation`函数中同样像上述**快速转发流程**一样，修改消息接收者，即修改`anInvocation.target`指向，也可以根据模块前缀做相应的UI操作以及上报处理等。