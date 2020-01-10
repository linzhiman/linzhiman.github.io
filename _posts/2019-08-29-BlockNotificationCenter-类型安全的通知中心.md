---
layout: post
title: BlockNotificationCenter-类型安全的通知中心
---
# {{ page.title }}

[-->完整源码传送门<--](https://github.com/linzhiman/ATKit/tree/master/ATKit/Utility/Notification)

众所周知，使用系统的NSNotificationCenter，我们需要定义一个方法来订阅通知，而且参数需要使用字典来传递。

这导致了很多参数类型的强制转换，而且订阅和通知处理方法不在一处，不便于阅读代码。

为了解决这些问题，笔者尝试了一些方法，下面分别介绍。

#### 1、尝试解决通知命名、方法命名的问题

```
#define AT_DECLARE_NOTIFICATION(__notification__)  NSString* const __notification__ = @#__notification__;
#define AT_EXTERN_NOTIFICATION(__notification__)   extern NSString *const __notification__;

#define AT_ADD_NOTIFY_DEFAULT_SELECTOR_NAME(__notification__) [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(on##__notification__:) name:__notification__ object:nil];
#define AT_HANDLER_NOTIFY_DEFAULT_SELECTOR_NAME(__notification__) - (void)on##__notification__:(NSNotification *)notification
```

例如订阅UIApplicationDidBecomeActiveNotification，则这样使用：

```
AT_ADD_NOTIFY_DEFAULT_SELECTOR_NAME(UIApplicationDidBecomeActiveNotification);

AT_HANDLER_NOTIFY_DEFAULT_SELECTOR_NAME(UIApplicationDidBecomeActiveNotification)
{
  // do something
}
```

#### 2、尝试解决订阅和通知处理方法不在一处的问题

由于订阅的方法类型是一致的，故定义一个block类型如下:

```
typedef void (^ATBNNativeBlock)(NSDictionary * _Nullable userInfo);
```

实现一个单例（ATBlockNotificationCenter）来包装对NSNotificationCenter的调用，存储通知名与订阅对象及其block的对应，并以ATBlockNotificationCenter的名义订阅通知，在统一的处理函数中，根据通知名回调对应的block；

ATBlockNotificationCenter主要有以下几个接口：

```
- (void)addNativeObserver:(id)observer name:(NSString *)name block:(ATBNNativeBlock)block;
- (void)removeNativeObserver:(id)observer name:(NSString *)name;
- (void)removeNativeObserver:(id)observer;
```

为了便于调用，实现一个NSObject的分类，其中post方法确保在主线程调用postNotificationName；

```
- (void)atbn_addNativeName:(NSString *)name block:(ATBNNativeBlock)block;
- (void)atbn_removeNativeName:(NSString *)name;
- (void)atbn_removeNativeAll;
- (void)atbn_postNativeName:(NSString *)name;
- (void)atbn_postNativeName:(NSString *)name userInfo:(NSDictionary *)userInfo;
```

#### 3、尝试解决类型安全的问题

针对系统的NSNotificationCenter，走到第2步，基本没想到优化方案了。为了解决类型安全的问题，只能自己造轮子了。

还是用宏来生成代码：

```
// 字符串定义
#define AT_STRING_EXTERN(atName) extern NSString * const atName;
#define AT_BN_DEFINE_NAME(atName) NSString * const atName = @"ATBN_"#atName;

// 头文件添加申明，声明通知名，对应block类型及便捷订阅函数
// AT_BN_DECLARE(kName, int a, NSString *b)
#define AT_BN_DECLARE(atName, ...) \
    AT_STRING_EXTERN(atName); \
    typedef void(^AT_BN_TYPE(atName))(__VA_ARGS__); \
    @interface NSObject (ATBN##atName) \
    - (void)atbn_on##atName:(AT_BN_TYPE(atName))block; \
    @end

// 实现文件添加定义
// AT_BN_DEFINE(kName, int a, NSString *b)
#define AT_BN_DEFINE(atName, ...) \
    AT_BN_DEFINE_NAME(atName); \
    @implementation NSObject (ATBN##atName) \
    - (void)atbn_on##atName:(AT_BN_TYPE(atName))block \
    { \
        [AT_BN_CENTER addObserver:self name:atName block:block]; \
    } \
    @end

// 发送通知
#define AT_BN_POST(atName, …) \
  NSArray *blocksNamed = [AT_BN_CENTER blocksNamed:atName]; \
  for (id block in blocksNamed) { \
      ((AT_BN_TYPE(atName))block)(__VA_ARGS__); \
  } \
```

ATBlockNotificationCenter定义如下，基本上是实现block的存储和访问，可以忽略具体实现；

```
@interface ATBlockNotificationCenter : NSObject

- (void)addObserver:(id)observer name:(NSString *)name block:(id)block;
- (void)removeObserver:(id)observer name:(NSString *)name;
- (void)removeObserver:(id)observer;
- (NSArray *)blocksNamed:(NSString *)name;

@end
```

至此，轮子实现完毕。下面通过一个例子来看看使用过程。

假设我们有个通知kName，需要带2个参数int a和NSString *b，接下来看看宏展开后的代码：

AT_BN_DECLARE(kName, int a, NSString *b) 展开：

```
extern NSString * const kName;
typedef void(^ATBN_kName)(int a, NSString *b);
@interface NSObject (ATBNkName)
- (void)atbn_onkName:(ATBN_kName)block;
@end
```

AT_BN_DEFINE(kName, int a, NSString *b) 展开：

```
NSString * const kName = @"ATBN_kName”;
@implementation NSObject (ATBNkName)
- (void)atbn_onkName:(ATBN_kName)block
{
    [AT_BN_CENTER addObserver:self name:atName block:block];
}
@end
```

AT_BN_POST(kName, 123, @"abc") 展开：

```
NSArray *blocksNamed = [AT_BN_CENTER blocksNamed:kName];
for (id block in blocksNamed) {
    ((ATBN_kName)block)(123, @"abc");
}
```

订阅时，直接调用atbn_onkName方法，XCode会有代码提示；

```
[self atbn_onkName:^(int a, NSString *b) {}];
```

发送时，通过AT_BN_POST宏调用，由于是可变参数宏，XCode没有代码提示；

```
AT_BN_POST(kName, 123, @"abc")
```

#### 4、尝试解决发送时代码提示问题

这就需要在生成代码时，生成带对应参数的post方法。综合起来，需要宏定义能展开成如下的代码：

```
typedef void(^ATBN_kName)(int a, NSString *b);
- (void)atbn_onkName:(ATBN_kName)block
{
  …
}
- (void)atbn_postkName_a:(int)a b:(NSString *)b
{
  …
  block(a, b);
  …
}
```

核心就是生成这3个代码串：

```
[int a, NSString *b]、[a:(int)a b:(NSString *)b]、[a, b]
```

假如，还是按照下面的方式来生成代码，由于int a或者NSString *b是一个参数，没法拆分，也就没法用形参(a, b)来回调block。

```
AT_BN_DECLARE(kName, int a, NSString *b)
```

因此需要把参数类型和形参拆分成2个参数，即：

```
AT_BN_DECLARE(kName, int, a, NSString *, b)
```

假如限定post只有2个参数，我们可以这样定义：

```
#define AT_PAIR_CONCAT_ARGS(A, B, C, D) A B, C D #int a, NSString *b
#define AT_POST_ARGS(A, B, C, D) B:(A)B D:(C)D   #a:(int)a b:(NSString *)b
#define AT_EVEN_ARGS(A, B, C, D) B, D            #a, b
```

那如果支持可变参数要怎么处理呢？

ReactiveCocoa(metamacros.h)实现了计算可变参数个数的方法。具体实现原理就不深入介绍了。

metamacro_argcount(a, b, c, d) 展开得到4

根据ReactiveCocoa的思路，我们也可以定义出需要的宏了。

终于，BlockNotificationCenter成型了。

附AT_EVEN_ARGS宏定义，AT_EVEN_ARGS(int, a, NSString *, b) 展开得到 a, b

```
#define metamacro_concat_(A, B) A##B
#define metamacro_concat(A, B) metamacro_concat_(A, B)

#define metamacro_head_(FIRST, ...) FIRST
#define metamacro_head(...) metamacro_head_(__VA_ARGS__, 0)

#define metamacro_at20(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, ...) metamacro_head(__VA_ARGS__)

#define metamacro_at(N, ...) \
    metamacro_concat(metamacro_at, N)(__VA_ARGS__)

#define metamacro_argcount(...) \
    metamacro_at(20, __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)

#define AT_MAKE_ARG_0()
#define AT_MAKE_ARG_2(first, second) second
#define AT_MAKE_ARG_4(first, second, ...) AT_MAKE_ARG_2(first, second), AT_MAKE_ARG_2(__VA_ARGS__)
#define AT_MAKE_ARG_6(first, second, ...) AT_MAKE_ARG_2(first, second), AT_MAKE_ARG_4(__VA_ARGS__)
#define AT_MAKE_ARG_8(first, second, ...) AT_MAKE_ARG_2(first, second), AT_MAKE_ARG_6(__VA_ARGS__)
#define AT_MAKE_ARG_10(first, second, ...) AT_MAKE_ARG_2(first, second), AT_MAKE_ARG_8(__VA_ARGS__)
#define AT_MAKE_ARG_12(first, second, ...) AT_MAKE_ARG_2(first, second), AT_MAKE_ARG_10(__VA_ARGS__)
#define AT_MAKE_ARG_14(first, second, ...) AT_MAKE_ARG_2(first, second), AT_MAKE_ARG_12(__VA_ARGS__)
#define AT_MAKE_ARG_16(first, second, ...) AT_MAKE_ARG_2(first, second), AT_MAKE_ARG_14(__VA_ARGS__)

#define AT_EVEN_ARGS_(...) metamacro_concat(AT_MAKE_ARG_, metamacro_argcount(__VA_ARGS__))(__VA_ARGS__)
#define AT_EVEN_ARGS(...) AT_EVEN_ARGS_(__VA_ARGS__)
```

#### 5、参数变动时，所有订阅的地方都需要调整代码，怎么避免？

自动生成数据模型，将所有参数作为模型的属性，Block类型修改为^(ATBNXXXObj * _Nonnull obj) {}

具体实现这里就不赘述了，感兴趣的同学请查看代码。

[-->完整源码传送门<--](https://github.com/linzhiman/ATKit/tree/master/ATKit/Utility/Notification)

