---
title: Python辅助iOS App瘦身实践
date: 2019-08-28 18:00:00
---

源码地址：https://github.com/linzhiman/ATKit

App安装包是由资源和可执行文件两部分组成的。因此从两方面分别来处理。

### 资源

这里重点关注图片资源的自动化分析，其他诸如音视频等资源因为数量少，人工一一检查排除即可。

可以打开IPA文件，或者通过工程Build Phases中的Copy Bundle Resources来查验所有文件。

针对图片资源，需要关注无用图片，重复图片，图片体积异常等情况。这里实现了2个脚本。

#### findUnusedImages.py - 查找无用图片，原理如下：

- 查找目录中.png图片(去除@2x及图片后缀)及imageset名字(去除后缀)，记为列表A；

- 查找所有字符串，对字符串做一些过滤，如不包含空格等，以减少列表长度。分为两个列表，一个不包含%号的列表B，一个包含%号的列表C；
  
- 遍历列表A，判断是否在列表B中，是则认为是有用图片，否则放入待定列表D；

- 将列表B的项，去除第一个%后之后的内容，记为前缀列表E；
  
- 遍历待定列表D，判断是否存在以前缀列表E中的项为前缀，是则认为是可能有用的图片，否则认为是可能无用的图片。对应输出两个列表，即待查验的图片列表。

#### trackImageSize.py - 监控图片增减及体积增量，输出html文件，原理比较简单：

- 使用json文件记录每个版本的图片路径及大小；
  
- 再比较两个记录的差异，得出新增图片列表、移除图片列表、变更图片列表；
  
- 计算对应体积增量，总体增量，按体积大小排序，输出Top10；
  
- 通过定期执行监控脚本，找到Top10图片列表，做进一步的处理，如压缩、动态下载等。

### 可执行文件

#### trackLinkmapObjSize.py - 监控linkmap各个.o模块大小变化

- linkmap文件是记录链接相关信息的文本文件，包含以下部分。通过Object files和Symbols块可以计算出每个目标文件.o所有段的总和；
  
- 以下示例linkmap文件内容：

```
# Path                    对应安装包路径
# Arch: x86_64            对应架构
# Object files:           所有目标文件.o及系统动态库描述文件.tbd等，方括号里面是序号
# Sections:               各分段分区的内存分地址、大小，主要包含只读段__TEXT、可读写__DATA
# Symbols:                类名、变量名、方法名等符号的内存地址、大小、所属.o文件序号
# Dead Stripped Symbols:  没用的符号，可以看到此处没有内存地址

# Path: ……/ATKitDemo
# Arch: x86_64
# Object files:
[  0] linker synthesized
…
[  7] ……/x86_64/AppDelegate.o  
…
# Sections:
# Address  Size      Segment  Section
0x1000019D0  0x0000DC50  __TEXT  __text
0x10000F620  0x000000CC  __TEXT  __stubs
…
0x10001AD88  0x00000420  __DATA  __data
…
# Symbols:
# Address  Size      File  Name
0x1000019D0  0x00000020  [  2] -[ATKitTransformViewController navigationBarHidden]
…
0x100005DE0  0x00000080  [  7] -[AppDelegate application:didFinishLaunchingWithOptions:]
…
# Dead Stripped Symbols:
#          Size      File  Name
<<dead>>   0x00000018  [  2] CIE
…
```

#### findUnusedCode.py - 查找无法类、方法，原理如下：

- otool(object file displaying tool)，可以对指定Mach-O以特定方法解析显示；

- otool -s <segname> <sectname> 可以列出某个section的内容；

- otool -s __DATA __objc_classlist 可以获取到所有类列表A；

- otool -s __DATA __objc_classrefs 可以获取到所有引用到的类列表B；

- A - B 即所有无用类的列表；

- 此处查看命令输出，可以看到输出的是内存地址，如68 a5 01 00 01 00 00 00 e0 a5 01 00 01 00 00，即为0x10001a568和0x10001a5e0

- otool -o -v (print the Objective-C segment)，可以打印详细的类的内存布局，包括baseMethods/baseProtocols/ivars/...等等

- 从输出第4行可以看到0x10001a568对应的类就是ATKitTransformViewController，据此可以将所有类名枚举出来；

- 从输出19-21行可以看到0x10000f9cf对应方法名是navigationBarHidden，因为不同类可能存在同名方法，这里分析imp这一行可以得到完整方法名，存储为字典{ address1:[method1,method2]}，记为方法列表C；

- otool -s __DATA __objc_selrefs 可以获取到所有引用到的方法的内存地址，如第7行包含两个地址0x100011611和0x10000f9cf，枚举所有内存地址记为列表D；

- C - D 即所有无用的方法列表；

- 注意：__objc_selrefs得到的是方法名，没有对应类的信息，故两个类存在同名方法时，没法分辨调用的是哪个类的方法；

- 由于OC的动态性，需要查找NSClassFromString，performSelector等方法进行过滤掉反射调用；

- 系统API的Protocol会找不到调用方而被标记为未使用，可以通过白名单方式过滤掉。

- 以下为各个命令输出示例：



```
# otool -s __DATA __objc_classlist ATKitDemo 
ATKitDemo:
Contents of (__DATA,__objc_classlist) section
0000000100014c28  68 a5 01 00 01 00 00 00 e0 a5 01 00 01 00 00 00 
0000000100014c38  08 a6 01 00 01 00 00 00 58 a6 01 00 01 00 00 00

# otool -s __DATA __objc_classrefs ATKitDemo 
ATKitDemo:
Contents of (__DATA,__objc_classrefs) section
000000010001a2d8  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
... 
000000010001a318  68 a5 01 00 01 00 00 00 b8 aa 01 00 01 00 00 00 
000000010001a328  c0 a7 01 00 01 00 00 00 90 aa 01 00 01 00 00 00

# otool -o -v ATKitDemo
ATKitDemo:
Contents of (__DATA,__objc_classlist) section
0000000100014c28 0x10001a568 _OBJC_CLASS_$_ATKitTransformViewController
           isa 0x10001a590 _OBJC_METACLASS_$_ATKitTransformViewController
    superclass 0x10001ab08 _OBJC_CLASS_$_ATBaseViewController
         cache 0x0 __objc_empty_cache
        vtable 0x0
          data 0x100014e00 (struct class_ro_t *)
                    flags 0x80
            instanceStart 8
             instanceSize 8
                 reserved 0x0
               ivarLayout 0x0
                     name 0x100012605 ATKitTransformViewController
              baseMethods 0x100014db0 (struct method_list_t *)
       entsize 24
         count 3
          name 0x10000f9cf navigationBarHidden
         types 0x100012934 q16@0:8
           imp 0x1000019d0 -[ATKitTransformViewController navigationBarHidden]
          name 0x10000f8d7 viewDidLoad
         types 0x10001293c v16@0:8
           imp 0x1000019f0 -[ATKitTransformViewController viewDidLoad]
          name 0x10000f975 onBack
         types 0x10001293c v16@0:8
           imp 0x1000024f0 -[ATKitTransformViewController onBack]
            baseProtocols 0x0
                    ivars 0x0
           weakIvarLayout 0x0
           baseProperties 0x0
Meta Class
           isa 0x0 _OBJC_METACLASS_$_NSObject
    superclass 0x10001ab30 _OBJC_METACLASS_$_ATBaseViewController
         cache 0x0 __objc_empty_cache
        vtable 0x0
          data 0x100014d68 (struct class_ro_t *)
                    flags 0x81 RO_META
            instanceStart 40
             instanceSize 40
                 reserved 0x0
               ivarLayout 0x0
                     name 0x100012605 ATKitTransformViewController
              baseMethods 0x0 (struct method_list_t *)
            baseProtocols 0x0
                    ivars 0x0
           weakIvarLayout 0x0
           baseProperties 0x0
0000000100014c30 0x10001a5e0 _OBJC_CLASS_$_ATKitDemoConfig

# otool -s __DATA __objc_selrefs ATKitDemo 
ATKitDemo:
Contents of (__DATA,__objc_selrefs) section
0000000100019ce0  c0 f8 00 00 01 00 00 00 d7 f8 00 00 01 00 00 00 
0000000100019cf0  e3 f8 00 00 01 00 00 00 e7 f8 00 00 01 00 00 00
...
000000010001a020  11 16 01 00 01 00 00 00 cf f9 00 00 01 00 00 00  
```

### 其他

主要是通过调整编译选项来实现。通过读取pbxproj工程文件，进而对有优化意义的选项做出提示，具体代码这里就不实现了。

