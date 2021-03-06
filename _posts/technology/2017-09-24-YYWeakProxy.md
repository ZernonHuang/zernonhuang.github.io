---
layout: post
date: 2017-09-24 10:25:00
title: YYKit(一)--YYWeakProxy 
category: 技术
keywords: iOS
description: 最近工作使用 CAAnimation ,它的代理为 strong ，使用 YYWeakProxy 解决。故而阅读其代码学习。
---

[YYWeakProxy v1.0.9](https://github.com/ibireme/YYKit/blob/4e1bd1cfcdb3331244b219cbd37cc9b1ccb62b7a/YYKit/Utility/YYWeakProxy.h)
==============

## YYWeakProxy 作用

关于 YYWeakProxy 的作用，在它的头文件就可以清楚的看到:

```
/**
 A proxy used to hold a weak object.
 It can be used to avoid retain cycles, such as the target in NSTimer or CADisplayLink.
 
 sample code:
 
     @implementation MyView {
        NSTimer *_timer;
     }
     
     - (void)initTimer {
        YYWeakProxy *proxy = [YYWeakProxy proxyWithTarget:self];
        _timer = [NSTimer timerWithTimeInterval:0.1 target:proxy selector:@selector(tick:) userInfo:nil repeats:YES];
     }
     
     - (void)tick:(NSTimer *)timer {...}
     @end
 */
```
YYWeakProxy 是用来持有一个 weak 对象的代理，避免循环引用。

这里引用链是：

> self -> timer -> proxy -> (消息转发)target -> self

由于 target 为弱引用，当 self 引用计数为 0 时, target 将为 nil, 于是打破了引用链。

## NSProxy

YYWeakProxy 继承于 NSProxy.

NSProxy 是什么？

NSProxy 是除了NSObject之外的另一个基类。

同时它也是一个抽象类，你可以通过继承它，并重写消息转发的方法，以实现消息转发到另一个实例的目的。

文档是这么说的:
>`NSProxy` implements the basic methods required of a root class, including those defined in the [`NSObjectProtocol`](https://developer.apple.com/documentation/objectivec/nsobjectprotocol) protocol. However, as an abstract class it doesn’t provide an initialization method, and it raises an exception upon receiving any message it doesn’t respond to. A concrete subclass must therefore provide an initialization or creation method and override the [`forwardInvocation(_:)`](https://developer.apple.com/documentation/foundation/nsproxy/1416417-forwardinvocation) and [`methodSignatureForSelector:`](https://developer.apple.com/documentation/foundation/nsproxy/1589828-methodsignatureforselector) methods to handle messages that it doesn’t implement itself

`NSProxy`可以说除了重载消息转发机制外没有别的用法，这也是它被设计的初衷，自己什么都不干，转给代理对象就好。往 `proxy` 发消息是注定会走消息转发的。

文档规定，NSProxy 子类必须重写这两个方法进行消息转发：

```
- (void)forwardInvocation:(NSInvocation *)anInvocation;
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel;
```

### NSProxy 和 NSObject 差异

两者同样都可以作为消息转发创建的代理类，但是存在一定的差异。

#### NSProxy 自动转发

通过继承自 `NSObject` 的代理类是不会自动转发 `respondsToSelector:和isKindOfClass:` 这两个方法的, 而继承自 `NSProxy` 的代理类却是可以的.

#### NSObject 的所有 Category 方法不能完成转发

例如 `valueForKey:` 是定义在 `NSKeyValueCoding` 这个 NSObject 的 `Category` 中的方法.

通过继承自 NSObject 的代理类，无法完成 NSObject 的 Category 里方法的转发。

原因是 NSObject 具备了这样的接口, 而消息转发是只有当接收者无法处理时，才会走 `forwardInvocation:` 方法。

### NSProxy 模拟多继承

NSProxy 可以通过消息的转发，模拟多继承的效果。让 proxy 接受处理多个不同 class 里的消息。

## YYWeakProxy 实现

YYWeakProxy 作为 NSProxy 的子类， `必须` 实现     `forwardInvocation:` 和 `methodSignatureForSelector:` 方法进行对象转发，这是在苹果官方文档中说明的。

以开头的场景为例子,当发送消息时,proxy 的方法列表里找不到 `tick:` ,就会开始走消息转发。

### 消息转发机制

当对象接受到无法响应的消息时，就会进入消息转发(message forwarding)的流程。


#### 转发流程

##### Dynamic method resolution

第一阶段，先征询消息接收者所属的类，看其是否能动态添加方法，以处理当前这个无法响应的 selector，这叫做 动态方法解析（dynamic method resolution）。

如果运行期系统（runtime system） 第一阶段执行结束，接收者就无法再以动态新增方法的手段来响应消息，进入第二阶段。

调用以下两个方法，对其进行解析，动态增加方法。

```
+ (BOOL)resolveInstanceMethod:(SEL)sel

+ (BOOL)resolveClassMethod:(SEL)sel
```

首先，系统会调用resolveInstanceMethod(当然，如果这个方法是一个类方法，就会调用resolveClassMethod)让你自己为这个方法增加实现。


##### Fast forwarding 

第二阶段，看看有没有其他对象能处理此消息。

如果有，运行期系统会把消息转发给那个对象，转发过程结束；
如果没有，则启动完整的消息转发机制。

```
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

##### Normal forwarding

第三阶段，完整的消息转发机制。运行期系统会把与消息有关的全部细节，都封装到 NSInvocation 对象中，再给接收者最后一次机会，令其设法解决当前还未处理的消息。

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector;

- (void)forwardInvocation:(NSInvocation *)invocation;
```

`methodSignatureForSelector`用来生成方法签名，这个签名就是给`forwardInvocation`中的参数NSInvocation调用的。

如果 `methodSignatureForSelector:` 返回nil，Runtime则会发出`doesNotRecognizeSelector:`消息，程序这时也就挂掉了。

### YYWeakProxy 的消息转发实现

#### forwardingTargetForSelector

```
- (id)forwardingTargetForSelector:(SEL)selector {
    return _target;
}
```

**`forwardingTargetForSelector`** 会返回你需要转发消息的对象，假如返回的是 nil，那么就走到 `forwardInvocation:` 做转发处理。

这里返回的是 _target 对象，那么实际就是调用 _target 对应的 selector 。


#### methodSignatureForSelector

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    return [NSObject instanceMethodSignatureForSelector:@selector(init)];
}
```

这里返回的是 NSObject 的 init 方法的签名。


#### forwardInvocation

这里只进行对 invocation 设置了一个返回值的类型，却并没有做转发。

```
- (void)forwardInvocation:(NSInvocation *)invocation {
    void *null = NULL;
    [invocation setReturnValue:&null];
}
```


##### setReturnValue

```
- (void)setReturnValue:(void *)retLoc;
```

[苹果文档](https://developer.apple.com/documentation/foundation/nsinvocation/1437848-setreturnvalue?language=objc) 对于 setReturnValue 的解释是： 

> Sets the receiver’s return value.

设置消息接受者的返回值。 

这里应该是没有返回值。

#### YYWeakProxy 的实际指向

由于在 `forwardingTargetForSelector:` 方法里，返回的实际上是 weak 修饰的 target 。 

```
YYWeakProxy *proxy = [YYWeakProxy proxyWithTarget:self];
```

所以这段代码里，target 就是 weak 修饰的 self。

这就意味着当 self 引用计数为 0 的时候， target 将被置为 nil.从而打破循环引用.

### 疑问

根据消息转发的机制，我们知道，如果 `forwardingTargetForSelector:` 方法里，返回了不为 nil 的对象。那么就不会进入后面的转发方法。

YYWeakProxy 里重写的 `methodSignatureForSelector` 和 `forwardInvocation` 方法内容，又是有什么具体的作用呢？


