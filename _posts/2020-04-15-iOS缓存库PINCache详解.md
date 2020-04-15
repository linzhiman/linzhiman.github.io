---
layout: post
title: iOS缓存库PINCache详解
---
# {{ page.title }}

### 简介

PINCache是一个高效、线程安全的键值缓存系统。

包含内存缓存PINMemoryCache和磁盘缓存PINDiskCache。

通过PINCache使用，将是一个二级缓存的结构，即数据会同时缓存在内存和磁盘。

由于PINMemoryCache和PINDiskCache都是公开属性，必要时也可以分别使用。

以下基于GitHub上最新master分支的源码分析。[-->PINCache传送门<--](https://github.com/pinterest/PINCache)


### 功能

PINCache内部组合PINMemoryCache及PINDiskCache，提供基本接口，支持同步和异步两种调用方式。

由于PINMemoryCache和PINDiskCache都是公开属性，可以调用具体类的更多接口。

（PINCache核心接口如下）：
```
Asynchronous Methods
* 		– containsObjectForKeyAsync:completion:
* 		– objectForKeyAsync:completion:
* 		– setObjectAsync:forKey:completion:
* 		– setObjectAsync:forKey:withAgeLimit:completion:
* 		– setObjectAsync:forKey:withCost:completion:
* 		– setObjectAsync:forKey:withCost:ageLimit:completion:
* 		– removeObjectForKeyAsync:completion:
* 		– trimToDateAsync:completion:
* 		– removeExpiredObjectsAsync:
* 		– removeAllObjectsAsync:
Synchronous Methods
* 		– containsObjectForKey:
* 		– objectForKey:
* 		– setObject:forKey:
* 		– setObject:forKey:withAgeLimit
* 		– setObject:forKey:withCost
* 		– setObject:forKey:withCost:ageLimit
* 		– removeObjectForKey:
* 		– trimToDate:
* 		– removeExpiredObjects
* 		– removeAllObjects
```

#### PINMemoryCache

PINMemoryCache类似于NSCache。

在iOS系统，可以设置当App收到内存警告/进入后台时自动清理缓存。

支持同步/异步两种调用方式。每个异步方法通过内建的并发队列执行，通过信号量保护缓存的读写。

支持LRU，会记录每次缓存访问时间，当缓存超标时移除最老的缓存。

支持设置ageLimit，将启动定时器定期检查并移除过期缓存。可以手动移除某个时间点前的缓存。

支持设置costLimit，添加缓存时指定对应的cost（可以是字节大小或其他自定义数值），当缓存超标时移除最老的缓存。可以手动减少缓存到某个cost（最贵优先或最老优先）。

(PINMemoryCache新增以下接口)：
```
Asynchronous Methods
* 		– trimToCostAsync:completion:
* 		– trimToCostByDateAsync:completion:
* 		– enumerateObjectsWithBlockAsync:completionBlock:
Synchronous Methods
* 		– trimToCost:
* 		– trimToCostByDate:
* 		– enumerateObjectsWithBlock:
```

#### PINDiskCache

PINDiskCache基于文件系统。

接受任何符合NSCoding协议的对象。同样支持同步/异步两种调用方式。

默认使用NSKeyedArchiver执行序列化/反序列化。

由于磁盘I/O比较慢，一般建议使用PINCache而不是直接使用PINDiskCache。

同样的，PINDiskCache支持ageLimit和byteLimit（cost具体化为byte，无需外部传值）。

(PINDiskCache新增以下接口)：
```
Asynchronous Methods
* 		– lockFileAccessWhileExecutingBlockAsync:
* 		– fileURLForKeyAsync:completion:
* 		– trimToSizeAsync:block:
* 		– trimToSizeByDateAsync:block:
* 		– enumerateObjectsWithBlockAsync:completionBlock:
Synchronous Methods
* 		– synchronouslyLockFileAccessWhileExecutingBlock:
* 		– fileURLForKey:
* 		– trimToSize:
* 		– trimToSizeByDate:
* 		– enumerateObjectsWithBlock:
```

### 实现


#### PINCache

由于PINCache内部使用PINMemoryCache和PINDiskCache实现具体功能，这里仅简单描述。

##### init

    创建PINOperationQueue、PINDiskCache、PINMemoryCache

##### containsObjectForKey

    PINMemoryCache中存在或PINDiskCache中存在

##### objectForKey

    先从PINMemoryCache获取，如果存在更新PINDiskCache中访问时间

    再从PINDiskCache获取，如果存在将缓存存入PINMemoryCache

##### setObject/removeObject/trimToDate

    分别操作PINMemoryCache、PINDiskCache，都完成后再返回/回调
    

#### PINMemoryCache

使用NSMutableDictionary分别存储缓存数据及对应的createdDate、accessDate、cost、ageLimit。

使用pthread_mutex_t保证线程安全。

使用PINOperationQueue实现异步调用。

会记录和维护totalCost，避免从字典中计算。

会订阅内存警告/进入后台通知。收到内存警告时，会根据设置移除所有缓存或者过期缓存。进入后台时，会根据设置移除所有缓存或者无操作。

具体实现上，包括添加、移除缓存，基本上都是在维护各个字典数据及totalCost。

这里需要注意的是锁的粒度，不能锁住回调，很容易造成死锁。


#### PINDiskCache

PINDiskCache将缓存写入文件，一个key对应一个文件。

使用NSMutableDictionary保存元数据，即createdDate、lastModifiedDate、size、ageLimit。

使用pthread_mutex_t保证线程安全。

使用PINOperationQueue实现异步调用。

会记录和维护byteCount，避免从字典中计算。

PINDiskCache初始化必须传入有效的name，用于创建缓存目录，多个PINDiskCache必须使用不同name。

默认使用NSKeyedArchiver执行序列化/反序列化。可以自定义序列化/反序列化回调。

由于I/O操作比较慢，创建目录是在子线程执行，PINDiskCache使用condition来保证创建完目录才能写入文件。

key将作为文件名，由于OS的文件名字符限制，需要对key做encode/decode。PINDiskCache提供默认的encoder/decoder，用户可以自定义。默认encoder将key中的.:/%字符进行Percent-encoding（URL编码）。

移除文件时，会将文件移动到一个Trash目录，然后移除Trash目录。为了保证线程安全，PINDiskCache创建了一个串行队列，将所有移除Trash目录的操作放到队列中执行。为了减少线程数量，这个串行队列通过dispatch_set_target_queue委托给系统创建的globalQueue。

文件的元数据createdDate、lastModifiedDate、size是通过文件属性获取的，分别对应NSURLCreationDateKey、NSURLContentModificationDateKey、NSURLTotalFileAllocatedSizeKey。而ageLimit，是通过文件的扩展属性实现，具体方法是setxattr/removexattr/getxattr。


#### PINOperationQueue

接下来，我们看看PINOperationQueue的设计和实现。

（PINOperationQueue核心接口）
```
* 		– scheduleOperation:
* 		– scheduleOperation:withPriority:
* 		– scheduleOperation:withPriority:identifier:coalescingData:dataCoalescingBlock:completion:
* 		– cancelOperation:
* 		– cancelAllOperations
* 		– waitUntilAllOperationsAreFinished
* 		– setOperationPriority:withReference:
```

PINOperationQueue支持设置最大并发数，当值为1时退化为FIFO的串行队列。默认值为CPU核心数量。

为了支持取消Operation，内部维护一个自增的id，用于标识Operation。

使用PTHREAD_MUTEX_RECURSIVE的pthread_mutex_t实现线程安全。

使用dispatch_semaphore_t控制并发。因为内部会维护一个串行队列和一个并发队列，故semaphore初始值为最大并发数减一。

由于支持修改最大并发数，内部创建了一个串行队列用于控制semaphore的操作。

为了支持优先级，内部有4个NSMutableOrderedSet，其中3个用于存储3个优先级（Low/Default/High）的Operation，另1个用于存储所有Operation，即queuedOperations。

执行时，当串行队列空闲时，会获取queuedOperations的第一个Operation执行，同时当最大并发数大于1时，会根据优先级顺序，获取一个Operation执行。（获取到Operation后会从所有队列中移除）

为了支持waitUntilAllOperationsAreFinished，即等待当前所有Operation完成，内部维护了一个dispatch_group_t。通过dispatch_group_wait实现当前线程等待。添加Operation时调用dispatch_group_enter，取消或完成Operation时调用dispatch_group_leave。


#### PINOperationGroup

（PINOperationGroup核心接口）
```
* 		– addOperation:
* 		– addOperation:withPriority:
* 		– start
* 		– cancel
* 		– setCompletion:
* 		– waitUntilComplete
```

PINOperationGroup是为了支持执行一组Operation，并感知全部执行完成。

具体实现是为每一组Operation维护一个dispatch_group_t，为每一个Operation创建一个新的Operation。

新的Operation包装了dispatch_group_enter/dispatch_group_leave的调用，最后交给PINOperationQueue去执行。
