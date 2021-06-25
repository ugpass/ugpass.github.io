---
title:      "Objective-C Dispatch Direct" 
date:       2021-06-25
tags:
    - iOS
categories:
    - iOS
---

## Objective-C Dispatch Direct

### Method 通过给Clang 传递 objc_direct

```
@interface Tool : NSObject
- (void)printMySelf;
@end

    0x100a60825 <+85>:  movq   %rax, -0x28(%rbp)
->  0x100a60829 <+89>:  movq   -0x28(%rbp), %rax
    0x100a6082d <+93>:  movq   0x7fec(%rip), %rsi        ; "printMySelf"
    0x100a60834 <+100>: movq   %rax, %rdi
    0x100a60837 <+103>: callq  *0x27d3(%rip)             ; (void *)0x00007fff20175280: objc_msgSend
```

上述代码 调用时，根据汇编显示callq 调用objc_msgSend发送“printMySelf”的消息



```
@interface Tool : NSObject
- (void)printMySelf __attribute__((objc_direct));
@end

    0x10270c835 <+85>:  movq   %rax, -0x28(%rbp)
->  0x10270c839 <+89>:  movq   -0x28(%rbp), %rax
    0x10270c83d <+93>:  movq   %rax, %rdi
    0x10270c840 <+96>:  callq  0x10270c2f0               ; -[Tool printMySelf] at Tool.m:33
    0x10270c845 <+101>: xorl   %r8d, %r8d
    0x10270c848 <+104>: movl   %r8d, %esi
    
    
 /**
 打印类的实例方法
 */
void printAllInstanceMethod(Class cls) {
    unsigned int count;
    Method *methods = class_copyMethodList(cls, &count);
    NSMutableArray *mutableA = [NSMutableArray arrayWithCapacity:10];
    for (int i = 0; i < count; i++) {
        NSString *methodName = NSStringFromSelector(method_getName(methods[i]));
        [mutableA addObject:methodName];
    }
    for (NSString *methodName in mutableA) {
        NSLog(@"%@ has Method - %@", cls, methodName);
    }
}
```

1. 上述代码 给函数通过给Clang传递**objc_direct**属性调用时，汇编显示callq 直接调用函数
2. 此时通过 `Method originMethod = class_getInstanceMethod([self class], @selector(printMySelf));`  获取该方法的Method时，会报如下错误 @selector expression formed with direct selector 'printMySelf' ，证明此时该方法不能被常见的runtime进行方法交换
3. 通过NSSelectorFromString 反射获取的SEL 为 0x0
4. 所以通过performSelector 调用方式会报 找不到方法的错误
5. 通过runtime获取类的方法，并获取不到被direct修饰的方法，应该在编译器确定了调用地址，直接调用
6. 一个direct method，不能被子类重写，会报如下错误 Cannot override a method that is declared direct by a superclass，同时在direct method中调用super也会报错
7. 协议中不能声明属性和方法为 direct，会报以下错误 'objc_direct' attribute cannot be applied to properties declared in an Objective-C protocol

### Property 通过 **direct**进行修饰

```
@interface Tool : NSObject
@property (nonatomic, strong) NSString *name;
@end
```

我们都知道平时声明属性时，编译器会帮助我们生成_name成员变量、setter、getter方法



```
@interface Tool : NSObject
@property (nonatomic, strong, direct) NSString *name;
@end

/**
 打印实例对象的成员变量
 */
void printAllIvars(NSObject *obj) {
    unsigned int count;
    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
    Ivar *list = class_copyIvarList([obj class], &count);
    for (int i = 0; i < count; i++) {
        NSString *ivarName = [NSString stringWithUTF8String:ivar_getName(list[i])];
        id ivarValue = [obj valueForKey:ivarName];
        if (ivarValue) {
            dict[ivarName] = ivarValue;
        }else{
            dict[ivarName] = @"";
        }
    }
    for (NSString *ivarName in dict.allKeys) {
        NSLog(@"ivarName:%@,ivarValue:%@",ivarName,dict[ivarName]);
    }
}
```

通过查看汇编，对应的setter 和 getter方法都是直接调用，不再走objc_msgSend流程；通过runtime获取成员变量列表，仍然有对应的_name成员变量

```
t.name= @"zhangsan";
[t addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
t.name = @"lisi";
```

由于没有setter方法，故kvo时，并没有任何回调

现在属性修饰词有以下16个

```
 setter getter
 readwrite readonly
 strong weak copy assign unsafe_unretained
 nullable nonnull null_resettable
 atomic nonatomic
 class direct
```



### `objc_direct_members` 修饰

```
__attribute__((objc_direct_members))
@interface Tool()

@end
```

在接口处添加修饰，没有任何作用



```
__attribute__((objc_direct_members))
@implementation Tool
```

在实现处添加修饰，则之前未声明的成员为直接成员，以及之前未声明的方法也会被直接调用



>  直接派发的效率和C语言调用一样，但是Objective-C的objc_msgSend在汇编和缓存层面做了很大的优化，调用效率相差并不会很大。
>
> 禁止通过runtime动态调用，作为私有实现。
>
> 最大的影响应该是在包体积方面的减少，主要是在SDK这方面。主要原因是因为方法不会放在method list中
>
> 如果OC的方法直接派发，仍然想要被外部可见，可以包装在C函数中

学习自：

[Objective-C Direct Methods](https://nshipster.com/direct/)

https://clang.llvm.org/docs/AttributeReference.html#objc-direct

