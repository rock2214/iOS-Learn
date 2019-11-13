KVOController，FaceBook出品。对kvo功能的封装。

源码解析，举两个例子

[《FBKVOController 源码解析》](https://satanwoo.github.io/2016/02/27/FBKVOController/)

[《如何优雅地使用 KVO》](https://github.com/Draveness/Analyze/blob/master/contents/KVOController/KVOController.md)


阅读上面两篇文章后基本可以了解它的原理。

我们还可以从它的源码中学习到什么？

 所以以下内容不是研究KVOController的使用方法，和源码的原理。而是看看FaceBook出品的代码。是怎样让你觉得爽的。

 打开FBKVOControllr.h文件看到代码。

代码整齐规范，注释清晰,可以点开代码结构看一下

从上到下几个部分

```
#define FBKVOKeyPath(KEYPATH) \
@(((void)(NO && ((void)KEYPATH, NO)), \
({ const char *fbkvokeypath = strchr(#KEYPATH, '.'); NSCAssert(fbkvokeypath, @"Provided key path is invalid."); fbkvokeypath + 1; })))

#define FBKVOClassKeyPath(CLASS, KEYPATH) \
@(((void)(NO && ((void)((CLASS *)(nil)).KEYPATH, NO)), #KEYPATH))

```

宏定义 - 宏定义在NS_ASSUME_NONNULL_BEGIN 外部


 静态变量声明

```
extern NSString * const FBKVONotificationKeyPathKey;

```

类型定义

```
typedef void (^FBKVONotificationBlock)(id _Nullable observer, id object, NSDictionary<NSString *, id> *change);

```


接口声明

接口声明中使用 #pragma mark 分割了三个部分

```
#pragma mark - Initialize
//初始化方法
#pragma mark - Observe
//Observe方法
#pragma mark - Initialize
//Unobserve方法
```

初始化方法中

自定义类方法放在第一个位置

NS_DESIGNATED_INITIALIZER 方法放在第二个位置

```
+ (instancetype)controllerWithObserver:(nullable id)observer;
- (instancetype)initWithObserver:(nullable id)observer retainObserved:(BOOL)retainObserved NS_DESIGNATED_INITIALIZER;
- (instancetype)initWithObserver:(nullable id)observer;
- (instancetype)init NS_UNAVAILABLE;
+ (instancetype)new NS_UNAVAILABLE;

```

init方法 和new方法禁用，使用NS_UNAVAILABLE修饰

打开FBKVOControllr.m文件

由于KVO依赖于ARC特性，所以第一行就判断是否支持ARC，不支持ARC直接编译器报错

```
#if !__has_feature(objc_arc)

#error This file must be compiled with ARC. Convert your project to ARC or specify the -fobjc-arc flag.

#endif
```


接着定义静态方法

```
static NSString *describe_option(NSKeyValueObservingOptions option)
{
  switch (option) {
    case NSKeyValueObservingOptionNew:
      return @"NSKeyValueObservingOptionNew";
      break;
    case NSKeyValueObservingOptionOld:
      return @"NSKeyValueObservingOptionOld";
      break;
    case NSKeyValueObservingOptionInitial:
      return @"NSKeyValueObservingOptionInitial";
      break;
    case NSKeyValueObservingOptionPrior:
      return @"NSKeyValueObservingOptionPrior";
      break;
    default:
      NSCAssert(NO, @"unexpected option %tu", option);
      break;
  }
  return nil;
}
```


这个方法将NSKeyValueObservingOptions 转为字符串

default方法中返回了一个断言 ，增加代码可靠性。

```
static void append_option_description(NSMutableString *s, NSUInteger option)
{
  if (0 == s.length) {
    [s appendString:describe_option(option)];
  } else {
    [s appendString:@"|"];
    [s appendString:describe_option(option)];
  }
}
```



这个方法拼接describe_option上面方法返回的字符串。

这两个方法的命名方式都是使用单词加下划线分割，不同于平时在Cocoa框架中方法的命名。

enumerate_flags

describe_options

这两个方法是啥没看明白

这些方法，终究都是用来打印调试信息时候使用的。无关业务。


然后声明了内部类 `_FBKVOInfo` 内部类用下划线开头

头部声明了一堆成员变量

```
{
@public
  __weak FBKVOController *_controller;
  NSString *_keyPath;
  NSKeyValueObservingOptions _options;
  SEL _action;
  void *_context;
  FBKVONotificationBlock _block;
  _FBKVOInfoState _state;
}
```

`__weak`是ARC环境下内存管理的修饰符，所以这里联想到很多内存管理的知识，以及为什么要用`__weak`? 这里是一个疑问。首先使用`__weak`可以大概猜测是防止循环引用，那么如果不用这个`__weak` 在什么地方会产生循环引用呢，带着这个问题继续看源码。


这个类只是把各种需要的数组整合到一起，没有对数据进行加工。

```
- (NSUInteger)hash
{
  return [_keyPath hash];
}

- (BOOL)isEqual:(id)object
{
  if (nil == object) {
    return NO;
  }
  if (self == object) {
    return YES;
  }
  if (![object isKindOfClass:[self class]]) {
    return NO;
  }
  return [_keyPath isEqualToString:((_FBKVOInfo *)object)->_keyPath];
}
```


从这两个方法看出最终是通过`_keyPath`来判断是否相等



然后是`_FBKVOSharedController`类

从名称也可以看出来，这个类也是不对外暴露的，单例。

声明了四个方法

```
/** A shared instance that never deallocates. */
+ (instancetype)sharedController;

/** observe an object, info pair */
- (void)observe:(id)object info:(nullable _FBKVOInfo *)info;

/** unobserve an object, info pair */
- (void)unobserve:(id)object info:(nullable _FBKVOInfo *)info;

/** unobserve an object with a set of infos */
- (void)unobserve:(id)object infos:(nullable NSSet *)infos;
```



主要提供是增加监听和解除监听。



实现部分最开始

```
{
  NSHashTable<_FBKVOInfo *> *_infos;
  pthread_mutex_t _mutex;
}
```

声明了两个成员，NSHashTable这个结构平时用的较少，类似一个字典结构，只是key可以使用对象类型。这个表的作用是，保存当前对象的`<_FBKVOInfo *> *_infos;`这里使用了范型。


实现部分的具体功能别的文章都已经讲的很详细了，这里就不多说

有一个地方 对NSHashTable 的操作进行了加锁


```
 pthread_mutex_lock(&_mutex);
 [_infos addObject:info];
 pthread_mutex_unlock(&_mutex);

```
而早期的KVOController 版本的锁是
```
  OSSpinLockLock(&_lock);
  [_infos addObject:info];
  OSSpinLockUnlock(&_lock);
```
这里可以看到作者更换了一个锁。



最后是`FBKVOController`类

实现了两个成员变量

```
{
  NSMapTable<id, NSMutableSet<_FBKVOInfo *> *> *_objectInfosMap;
  pthread_mutex_t _lock;
}

```



这里又使用到了 NSMapTable 这个结构

存储的数据结构更加复杂

Key 是 id 类型

Value 是 存储了`_FBKVOInfo *`类型 的 `NSMutableSet`的指针。



功能不多介绍

注释方面

```
#pragma mark Properties -

#pragma mark Utilities -

#pragma mark API -
```

对区域进行分隔



之前_FBKVOInfo中弱引用了FBKVOController
所以找一下FBKVOController是否引用了_FBKVOInfo
找来找去 FBKVOController 引用objectInfosMap 存储了_FBKVOInfo对象


方法开始时候都使用了类似的非空判断

```
NSAssert(0 != keyPath.length && **NULL** != block, @"missing required parameters observe:%@ keyPath:%@ block:%p", object, keyPath, block);
```
## 总结如下

- 代码规范
  + 内部类使用下划线开头
  + 对初始化方法修饰。不使用的初始化方法使用NS_UNAVAILABLE修饰。明确NS_DESIGNATED_INITIALIZER方法。
  + 注释的方式 #pragma mark API -

- 使用数据结构的方式NSHashTable
- 使用锁的方式
- 使用范型的方式
- 不过分使用属性，能使用成员变量的情况下使用成员变量
- 数据结构的使用方式 `_FBKVOInfo` 类似于瘦模型，没有过多的逻辑操作，在定义变量时候要注意内存管理，适当的使用weak
