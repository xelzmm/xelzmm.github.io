---
layout: post
title: "关于bitcode, 知道这些就够了"
date: 2018-11-24 17:17:28 +0800
comments: true
categories: ios
typora-root-url: ../../source
---



## 0x00 前言

苹果在WWDC 2015大会上引入了bitcode，随后在Xcode7中添加了在二进制中嵌入bitcode(Enable Bitcode)的功能，并且默认设置为开启状态。很多开发者在集成第三方SDK的时候都被bitcode坑过一把，然后google百度一番发现只要关闭bitcode就可以了，但是大部分开发者都不清楚bitcode到底是什么东西。这篇文档将给大家详细地介绍与bitcode有关的内容。

## 0x01 什么是bitcode

研究bitcode之前需要先了解一下LLVM，因为bitcode是由LLVM引入的一种中间代码(Intermediate Representation，简称IR)，它是源代码被编译为二进制机器码过程中的中间表示形态，它既不是源代码，也不是机器码。从代码组织结构上看它比较接近机器码，但是在函数和指令层面使用了很多高级语言的特性。

LLVM是一套优秀的编译器框架，目前NDK/Xcode均采用LLVM作为默认的编译器。LLVM的编译过程可以简单分为3个部分:

![](/assets/2018/RetargetableCompiler.png)

<center>图来自 http://www.aosabook.org/en/llvm.html</center>

1. 前端(Frontend)，负责把各种类型的源代码编译为中间表示，也就是bitcode，在LLVM体系内，不同的语言有不同的编译器前端，最常见的如clang负责c/c++/oc的编译，flang负责fortran的编译，swiftc负责swift的编译等等
2. 优化(Optimizer)，负责对bitcode进行各种类型的优化，将bitcode代码进行一些逻辑等价的转换，使得代码的执行效率更高，体积更小，比如DeadStrip/SimplifyCFG
3. 后端(Backend)，也叫CodeGenerator，负责把优化后的bitcode编译为指定目标架构的机器码，比如X86Backend负责把bitcode编译为x86指令集的机器码

在这个体系中，不同语言的源代码将会被转化为统一的bitcode格式，三个模块可以充分复用，防止重复造轮子。如果要开发一门新的`x语言`，只需要造一个x语言的前端，将x语言的源代码编译为bitcode，优化和后端的事情完全不用管。同理，如果新的芯片架构问世，则只需要基于LLVM重新写一套目标平台的后端，非常方便。

## 0x02 bitcode初探

既然bitcode是代码的一种表示形式，因此它也会有自己的一套独立的语法，可以通过一个简单的例子来一探究竟，这里以clang为例，swift的操作和结果可能稍有不同。

本文所涉及的内容可以自行操作，也可以直接下载[我写这篇文章时保存的副本](/assets/2018/bitcode-demo.zip)

先编写一段helloworld代码(test.c)：

```c
#include <stdio.h>
int main(void) {
    printf("hello, world.\n");
    return 0;
}
```

通过以下命令可以将源代码编译为object文件:

```bash
$ clang -c test.c -o test.o
$ file test.o
test.o: Mach-O 64-bit object x86_64
```

其实，这个命令同时完成了前端、优化、后端三个部分，可以通过 `-emit-llvm -c` 将前端这一步单独拆出来，这样就可以看到bitcode了:

```bash
$ clang -emit-llvm -c test.c -o test.bc # 将源代码编译为bitcode
$ file test.bc
test.bc: LLVM bitcode, wrapper x86_64
$ clang -c test.bc -o test.bc.o # 将bitcode编译为object
$ file test.bc.o
test.bc.o: Mach-O 64-bit object x86_64
$ md5 test.bc.o test.o
MD5 (test.bc.o) = 70ea3a520c26df84d1f7ca552e8e6620
MD5 (test.o) = 70ea3a520c26df84d1f7ca552e8e6620
```

bitcode文件使用后缀名`.bc`表示，可以看到，将bitcode文件作为clang的输入，编出的object文件跟直接编源代码是相同的。然后在来看一下bitcode文件:

```bash
$ hexdump -C test.bc  | head
00000000  de c0 17 0b 00 00 00 00  14 00 00 00 08 0b 00 00  |................|
00000010  07 00 00 01 42 43 c0 de  35 14 00 00 07 00 00 00  |....BC..5.......|
00000020  62 0c 30 24 96 96 a6 a5  f7 d7 7f 4d d3 b4 5f d7  |b.0$.......M.._.|
00000030  3e 9e fb f9 4f 0b 51 80  4c 01 00 00 21 0c 00 00  |>...O.Q.L...!...|
00000040  74 02 00 00 0b 02 21 00  02 00 00 00 13 00 00 00  |t.....!.........|
00000050  07 81 23 91 41 c8 04 49  06 10 32 39 92 01 84 0c  |..#.A..I..29....|
00000060  25 05 08 19 1e 04 8b 62  80 10 45 02 42 92 0b 42  |%......b..E.B..B|
00000070  84 10 32 14 38 08 18 4b  0a 32 42 88 48 90 14 20  |..2.8..K.2B.H.. |
00000080  43 46 88 a5 00 19 32 42  04 49 0e 90 11 22 c4 50  |CF....2B.I...".P|
00000090  41 51 81 8c e1 83 e5 8a  04 21 46 06 51 18 00 00  |AQ.......!F.Q...|
```

通过hexdump可以看出这个文件并非文本文件，全是乱码，这样的文件是很难分析的。其实LLVM提供了`llvm-dis`/ `llvm-as` 两个工具，用于将bitcode在二进制格式和可读的文本格式之间进行相互的转化，但遗憾的是Xcode的编译器工具链中并没有附带这个命令，因此只能另寻他法。

<!-- more -->

我们知道通过编译器的`-S`参数可以将源代码编译为文本的assembly代码，不进行最后一步assembly到机器码的翻译工作，而assembly和机器码是等价的两种表示形式，bitcode同样也是有文本和二进制(bitcode)两种等价表示形式，clang也为bitcode保留了这一特性，可以通过`-emit-llvm -S` 将源代码编译为文本格式的bitcode， 也叫做LLVM Assembly Language，一般后缀名使用`.ll`:

```bash
$ clang -emti-llvm -S test.c -o test.ll # 将源代码编译为LLVM Assembly
```

test.ll的全部内容如下

```llvm
; ModuleID = 'test.c'
source_filename = "test.c"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.14.0"

@.str = private unnamed_addr constant [15 x i8] c"hello, world.\0A\00", align 1

; Function Attrs: noinline nounwind optnone ssp uwtable
define i32 @main() #0 {
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([15 x i8], [15 x i8]* @.str, i32 0, i32 0))
  ret i32 0
}

declare i32 @printf(i8*, ...) #1

attributes #0 = { noinline nounwind optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.module.flags = !{!0, !1}
!llvm.ident = !{!2}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 7, !"PIC Level", i32 2}
!2 = !{!"Apple LLVM version 10.0.0 (clang-1000.11.45.5)"}
```

这样看上去就很清晰明了了，我们重点关注下函数定义这部分，我加了一些注释方便理解

```llvm
; 定义全局常量 @.str, 内容初始化为 'hello, world.\n\0'
@.str = private unnamed_addr constant [15 x i8] c"hello, world.\0A\00", align 1

; Function Attrs: noinline nounwind optnone ssp uwtable
define i32 @main() #0 { ; 定义函数 @main，返回值为i32类型
  %1 = alloca i32, align 4 ; 声明变量 %1 = 分配i32的内存空间
  store i32 0, i32* %1, align 4 ; 将 0 存入 %1 的内存空间
  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([15 x i8], [15 x i8]* @.str, i32 0, i32 0)) ; 调用 @printf 函数，并将 @.str 的地址作为参数
  ret i32 0 ; 返回 0
}

declare i32 @printf(i8*, ...) #1 ; 声明一个外部函数 @printf
```

这段代码不难阅读， 其含义和逻辑与我们所写的源代码完全一致，只是用了另外一种语法表示出来。bitcode的具体格式及语法在此不做展开，虽然这个例子看起来非常简单易懂，但真实场景中，bitcode的格式远比这个复杂，有兴趣的同学可以直接阅读[LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html)。

## 0x03 Enable Bitcode

在对bitcode有了一个直观的认识之后，再来看一下Apple围绕bitcode做了什么。Xcode中对Enable Bitcode这个配置的解释是:

> 以下摘自Xcode Help
>
>  https://help.apple.com/xcode/mac/10.1/index.html?localePath=en.lproj#/itcaec37c2a6
>
> #### Enable Bitcode (ENABLE_BITCODE)
>
> Activating this setting indicates that the target or project should generate bitcode during compilation for platforms and architectures that support it. For Archive builds, bitcode will be generated in the linked binary for submission to the App Store. For other builds, the compiler and linker will check whether the code complies with the requirements for bitcode generation, but will not generate actual bitcode.

具体展开一下：

- 开启此设置将会在支持的平台和架构中开启bitcode
  - 当前支持的平台主要是iPhoneOS(armv7/arm64)，watchOS等
  - 注意不包括iPhoneSimulator(i386/x86_64)和macos，也就是说模拟器架构下不会编出bitcode
- 进行Archive时，bitcode会被嵌入到链接后的二进制文件中，用于提交给App Store
  - Enable Bitcode 设置为 YES 时，从编译日志中可以看出，Archive时多了一个编译参数 `-fembed-bitcode`
- 进行其他类型的Build(非Archive)时，编译器只会检查是否满足开启bitcode的条件，但并不会真正生成bitcode
  - 非Archive编译时，Enable Bitcode 将会增加编译参数 `-fembed-bitcode-marker`， 只是在object文件中做了标记，表明`我可以有bitcode，但是现在暂时没有带上它`。因为本地编译调试时并不需要bitcode，只有AppStore需要这玩意儿，去掉这个不必要的步骤，会加快编译速度。
  - 这就是为什么有的同学在开发SDK时，明明开启了Enable Bitcode，交给客户后客户却说：你的sdk里没有bitcode，因为你没有使用Archive方式打包。
  - 当然，你可以将 Enable Bitcode 设置为NO， 然后在Other Compiler Flags 和 Other Linker Flags 中手动为真机架构添加`-fembed-bitcode` 参数，这样任何类型的Build都会带上bitcode

接下来看一下 Enable Bitcode 之后，编译出的文件发生了什么变化， 直接在clang的参数中添加 `-fembed-bitcode` 即可

```bash
$ clang -fembed-bitcode -c test.c -o test_bitcode.o
$ otool -l test_bitcode.o
# 以下为otool输出节选
Section
  sectname __bitcode
   segname __LLVM
      addr 0x0000000000000040
      size 0x0000000000000b10
    offset 776
     align 2^4 (16)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
Section
  sectname __cmdline
   segname __LLVM
      addr 0x0000000000000b50
      size 0x0000000000000042
    offset 3608
     align 2^4 (16)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
```

或者使用MachOView

![](/assets/2018/machoview.png)

可以发现生成的 object 文件中多了两个 Section，分别是 `__LLVM,__bitcode` 和 `__LLVM,__cmdline`，并且otool的输出中给出了这两个section在object文件中的偏移和大小，通过 `dd` 命令可以很方便地将这两个Section提取出来

```bash
$ dd bs=1 skip=776 count=0x0000000000000b10 if=test_bitcode.o of=test_bitcode.o.bc
2832+0 records in
2832+0 records out
2832 bytes transferred in 0.017339 secs (163331 bytes/sec)
$ dd bs=1 skip=3608 count=0x0000000000000042 if=test_bitcode.o of=test_bitcode.o.cmdline
66+0 records in
66+0 records out
66 bytes transferred in 0.001312 secs (50304 bytes/sec)
```

还有一种更便捷的方式，Xcode 提供的 `segedit` 命令可以直接将指定的Section导出，只需要给定Section的名字，和上面的命令效果是一样的，并且更为方便

```bash
$ segedit -extract __LLVM __bitcode test_bitcode.o.bc \
          -extract __LLVM __cmdline test_bitcode.o.cmdline \
          test_bitcode.o
```

观察一下导出的文件

```bash
$ file test_bitcode.o.bc
test_bitcode.o.bc: LLVM bitcode, wrapper x86_64
$ cat test_bitcode.o.cmdline | tr '\0' ' '
-triple x86_64-apple-macosx10.14.0 -emit-obj -disable-llvm-passes
$ md5 test.bc test_bitcode.o.bc
MD5 (test.bc) = 1592ed7db86742184a559e86cb9d1355
MD5 (test_bitcode.o.bc) = 9901ac8db63be30dafc19c2f06b0cae8
```

不难得出结论：

- object文件中嵌入的`__LLVM,__bitcode` 正是完整的，未经任何加密或者压缩的bitcode文件
- `__LLVM,__cmdline` 是编译这个文件所用到的参数，如果要通过导出的bitcode重新编译这个object文件，必须带上这些参数
  - 导出的参数是`cc1` 也就是clang中真正"前端"部分的参数(clang命令其实是整合了各个环节，所以clang一个命令可以从源代码编出可执行文件)，所以编译时要带上`-cc1`
- 导出的bitcode文件似乎和直接编译的bitcode不一样，先留个疑问，后面再研究

首先， 来测试一下导出的bitcode文件结合cmdline能否编译出正常的object:

```bash
$ clang -cc1 -triple x86_64-apple-macosx10.14.0 -emit-obj -disable-llvm-passes test_bitcode.o.bc -o test_rebuild.o
$ file test_rebuild.o
test_rebuild.o: Mach-O 64-bit object x86_64
$ md5 test.o test_rebuild.o
MD5 (test.o) = 70ea3a520c26df84d1f7ca552e8e6620
MD5 (test_rebuild.o) = 70ea3a520c26df84d1f7ca552e8e6620
```

没有任何问题，并且通过内嵌的bitcode编译出的object文件与直接从源代码编译出来的object完全一样！鹅妹子嘤~！

回到遗留的问题：为什么导出的bitcode文件和直接编译的bitcode会不一样？明明编出的object都是一模一样的！这是因为二进制的bitcode文件中还保存了一些与实际代码无关的meta信息。如果能将bitcode转换为文本格式，将能更直观地进行对比。前面已经提到，xcode中并没有附带转换工具，但是我们依然可以通过clang来完成这一操作：

```bash
$ clang -emit-llvm -S test_bitcode.o.bc -o test_bitcode.o.ll
```

神奇吧？其实clang内部是先将输入的文件转换成Module对象，然后再执行对应的处理：

- 如果输入是源代码，会先进行前端编译，得到一个Module
- 如果输入是bitcode或者LLVM Assembly，那么直接进行parse操作，即可得到Module对象
- 如果输出类型是LLVM Assembly，将Module对象序列化为文本格式
- 如果输出类型是bitcode，则将Module对象序列化为二进制格式

所以完全可以通过clang进行bitcode和LLVM Assembly的相互转换。

现在，可以对比一下前后两次生成的`.ll`文件:

```bash
$ diff test_bitcode.o.ll test.ll
1c1
< ; ModuleID = 'test_bitcode.o.bc'
---
> ; ModuleID = 'test.c'
```

除了ModuleID，也就是来源的文件名以外，其余部分完全相同，这也就解决了前面的疑虑。

再来回顾一下，前文提到非Archive类型的build，比如直接`⌘ + B`，即使开启了bitcode，也不会编出bitcode，那么会产生什么样的文件呢？通过观察编译日志可以看出xcode在此时使用了`-fembed-bitcode-marker` 这样一个参数，我们来试一下：

```bash
$ clang -fembed-bitcode-marker -c test.c -o test_bitcode_marker.o
$ otool -l test_bitcode_marker.o
# 以下为otool输出节选
Section
  sectname __bitcode
   segname __LLVM
      addr 0x0000000000000039
      size 0x0000000000000001    # 只有一个字节
    offset 769
     align 2^0 (1)
    reloff 0
    nreloc 0
     flags 0x00000000
$ objdump -s -section=__bitcode test_bitcode_marker.o
Contents of section __bitcode:
 0039 00                                   . # 只有一个字节 0x00
```

这样的方式编译出的文件结构与`-fembed-bitcode` 的结果是一样的，唯一的区别就是 `__LLVM,__bitcode` 的内容并没有将bitcode文件嵌入进来，取而代之的是一个空的文件，只有一个占位符`0x00`

## 0x04 Bitcode Bundle

已经搞清楚了bitcode是如何嵌入在object文件里的，但是object只是编译过程的中间产物，真正运行的代码是多个object文件经过链接之后的可执行文件，接下来要分析下object中嵌入的bitcode是如何被链接的：

```bash
$ clang test.o -o test # 链接原始object
$ ./test
hello, world.
$ clang -fembed-bitcode test_bitcode.o -o test_bitcode # 链接带bitcode的object
$ ./test_bitcode
hello, world.
$ otool -l test_bitcode
# 以下为otool输出节选
Section
  sectname __bundle
   segname __LLVM
      addr 0x0000000100002000
      size 0x0000000000001261
    offset 8192
     align 2^0 (1)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
```

object中的 `__LLVM,__bitcode` 和 `__LLVM,__cmdline` 不见了，取而代之的是一个 `__LLVM,__bundle` 的Section， 通过名字可以基本推断出object中的bitcode被打包在了一起，把它从可执行文件中dump出来一探究竟：

```bash
$ segedit -extract __LLVM __bundle bundle test_bitcode
$ file bundle
bundle: xar archive version 1, SHA-1 checksum
```

这个bundle文件是一个`xar`格式的压缩包，xar格式包含了一个`xml`格式的文件头(TOC)，里面用于存放各种文件的基本属性以及一些附加附加信息，可以通过xar命令查看并解压

```bash
$ xar -d toc.xml -f bundle # 导出文件头
$ mkdir bundle.extract
$ xar -x -C bundle.extract -f bundle # 解压文件
$ ls bundle.extract
1
$ file bundle.extract/1
bundle.extract/1: LLVM bitcode, wrapper x86_64
$ md5 bundle.extract/1 test_bitcode.o.bc
MD5 (bundle.extract/1) = 9901ac8db63be30dafc19c2f06b0cae8
MD5 (test_bitcode.o.bc) = 9901ac8db63be30dafc19c2f06b0cae8
```

查看导出的toc.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xar>
 <subdoc subdoc_name="Ld">
  <version>1.0</version>
  <architecture>x86_64</architecture>
  <platform>macOS</platform>
  <sdkversion>10.14.0</sdkversion>
  <dylibs>
   <lib>{SDKPATH}/usr/lib/libSystem.B.dylib</lib>
  </dylibs>
  <link-options>
   <option>-execute</option>
   <option>-macosx_version_min</option>
   <option>10.14.0</option>
   <option>-e</option>
   <option>_main</option>
   <option>-executable_path</option>
   <option>test</option>
  </link-options>
 </subdoc>
 <toc>
  <checksum style="sha1">
   <size>20</size>
   <offset>0</offset>
  </checksum>
  <creation-time>2018-12-19T12:07:24</creation-time>
  <file id="1">
   <name>1</name>
   <type>file</type>
   <data>
    <archived-checksum style="sha1">56346f644ab01200e0ad56eaefb9346a863cb473</archived-checksum>
    <extracted-checksum style="sha1">56346f644ab01200e0ad56eaefb9346a863cb473</extracted-checksum>
    <size>2832</size>
    <offset>20</offset>
    <encoding style="application/octet-stream"/>
    <length>2832</length>
   </data>
   <file-type>Bitcode</file-type>
   <clang>
    <cmd>-triple</cmd>
    <cmd>x86_64-apple-macosx10.14.0</cmd>
    <cmd>-emit-obj</cmd>
    <cmd>-disable-llvm-passes</cmd>
   </clang>
  </file>
 </toc>
</xar>
```

header的结构非常清晰，内容基本包含这些：

- ld 的基本参数，我们链接时使用的是clang，实际上clang内部调用了ld，这里记录的是ld的参数
  -  version: bitcode bundle 的版本号
  - architecture: 目标架构
  - platform: 目标平台
  - sdkversion: sdk版本
  - dylibs: 链接的动态库
  - link-options: 其他链接参数
- 文件目录
  - checksum类型
  - 创建时间
  - 每个文件的信息
    - 文件名，这里并非原始文件名，而是按照链接时输入的顺序被重命名为数字序号
    - 基本属性，包括checksum、偏移、大小等
    - 文件类型，一般是Bitcode，还有两种特殊类型，Object以及Bundle，这里卖个关子，大家有兴趣可已自行研究(想想如果一个源代码文件是.s格式，要如何支持bitcode)
    - 编译器类型(clang/swift)及编译参数，这部分就是object文件中 `__LLVM,__cmdline` 的内容
  - 下一个文件的信息(如有)
  - 重复

从bundle中解压出来的文件，就是object中嵌入的bitcode，通过MD5对比可以看出链接时对bitcode文件自身没有做任何处理。可以注意到，用于编译各个bitcode文件的参数(cmdline)被放进了TOC中文件描述的区域，而TOC中多出了一个部分用于存放链接时所需要的信息和必要的参数，有了这些信息， 我们不难通过bitcode重新编译，并链接出一个新的可执行文件：

```bash
$ clang -cc1 -triple x86_64-apple-macosx10.14.0 -emit-obj -disable-llvm-passes bundle.extract/1 -o bundle.extract/1.o -x ir 
# 由于解压出的文件没有后缀名，clang无法判断输入文件的格式，因此使用 -x ir 强制指定输入文件为ir格式
# 也可以将其重命名为1.bc，这样就不用指定-x ir
$ ld \
    -arch x86_64 `# architecture` \
    -syslibroot `xcrun --show-sdk-path --sdk macosx` `# platform` \
    -sdk_version 10.14.0 `# sdkversion` \
    -lSystem `# dylibs` \
    -execute `# link-options` \
    -macosx_version_min 10.14.0 `# link-options` \
    -e _main `# link-options` \
    -executable_path test `# link-options` \
    -o test_rebuild `# 输出文件` \
    bundle.extract/1.o `# 输入文件`
$ ./test_rebuild
hello, world.
$ md5 test_rebuild test
MD5 (test_rebuild) = f4786288582decf2b8a1accb1aaa4a3c
MD5 (test) = f4786288582decf2b8a1accb1aaa4a3c
```

看！我们成功利用bitcode重新编了一份一模一样的可执行文件出来。

现在可以理解，为什么苹果要强推bitcode了吧？开发者把bitcode提交到App Store Connect之后，如果苹果发布了使用新芯片的iPhone，支持更高效的指令，开发者不需要做任何操作，App Store Connect自己就可以编译出针对新产品优化过的app并通过App Store分发给用户，不需要开发者自己重新打包上架，这样一来苹果的Store生态就不需要依赖开发者的积极性了。

## 0x05 使用Bitcode导出ipa

前面已经提到，如果要以bitcode方式上传app，必须在开启bitcode的状态下，进行Archive打包，才会得到带有bitcode的app。大部分app都会依赖一堆第三方sdk，如果此时项目里依赖的某一个或者几个sdk没有开启bitcode，那么很遗憾，Xcode会拒绝编译并给出类似这样的提示：

> ld: 'name_of_the_library_or_framework' does not contain bitcode. You must rebuild it with bitcode enabled (Xcode setting ENABLE_BITCODE), obtain an updated library from the vendor, or disable bitcode for this target.
>
> ld: bitcode bundle could not be generated because 'name_of_the_library_or_framework' was built without full bitcode.

第一种提示表示这个第三方库完全没有开启bitcode，而第二种提示表示它只有bitcode-marker，也就是说它的开发者虽然在工程配置中设置了 Enable Bitcode 为 YES，但并没有以Archive方式编译，可能只是⌘ + B，然后顺手把Products拷贝出来交付了。

遇到这种问题，也需要分两种情况来看：

- 如果这个库是在本地编译的， 比如自己项目里或者子项目里的target，或者通过Pods引入了源代码，找到这个target的Build Settings把Enable Bitcode置为YES即可
- 但如果是第三方提供的二进制库文件，则需要联系sdk的提供方确认是否能提供带bitcode的版本，否则只能关闭自己项目中的bitcode。这也是bitcode时至今日都没有得到大面积应用的最大障阻碍。

当使用Archive方式打包出带有bitcode的包时，你会发现这个包里的二进制文件比没有开启bitcode时大出了许多，多出来的其实就是bitcode的体积，并且bitcode的体积，一般要二进制文件本身还要大

```bash
$ ls -al test.o test_bitcode.o test.bc
-rw-r--r--  1 xelz  staff  2848 12 19 18:42 test.bc
-rw-r--r--@ 1 xelz  staff   784 12 19 18:24 test.o
-rw-r--r--@ 1 xelz  staff  3920 12 19 18:59 test_bitcode.o
```

当然，这部分内容并不会导致用户下载到的APP变大，因为用户下载到的代码中只会有机器码，不会包含bitcode。有的项目开启bitcode之后会发现二进制的体积增大到超出了苹果对[二进制体积的限制](https://help.apple.com/app-store-connect/#/dev611e0a21f)，但是完全不用担心，苹果的限制只是针对`__TEXT` 段，而嵌入的bitcode是存储在单独的`__LLVM` 段，不在苹果的限制范围内。

打包出带有bitcode的xcarchive之后，可以导出Development IPA进行上线前的最终测试，或者上传到App Store Connect进行提审上架。进行此类操作时会发现Xcode Organizer中多出了bitcode相关的选项：

- 导出Development版本时，可以勾选`Rebuild from Bitcode`，这时导出会变的很慢，因为Xcode在后台通过bitcode重新编译代码，这样导出的ipa最接近最终用户从AppStore下载的版本。而如果不勾选此选项，则会直接使用Archive时编译出的二进制代码，并把bitcode从二进制中去除以减小体积。

  ![rebuild from bitcode](/assets/2018/organizer-export.png)

- 导出Store版本或者直接进行上传时，默认会勾选`Include bitcode for iOS content`，如果不勾选，则跟前面类似，将会去除内嵌的bitcode，直接使用本地编译的二进制代码

  ![include](/assets/2018/organizer-upload.png)

  勾选后生成的ipa中将会`只包含bitcode`，这个ipa是无法重签后安装到设备上进行测试的，因为里面没有任何可执行代码：

  ![](/assets/2018/machoview1.png)

  `__TEXT` 和 `__DATA` 等跟已编译好的二进制相关的内容会被全部去除，但是会保留`__LINKEDIT`中的部分信息，其中最重要的就是 `LC_UUID`，用于在重编之后能跟原始的符号文件对应起来，如果用户下载经过AppStore重编之后的app发生了Crash，得到的backtrace地址是跟本地编译的版本对应不起来的，需要结合UUID和从App Store Connect下载的dSYM文件才能得到符号化的crash信息。

  ![](/assets/2018/machoview2.png)

## 0x06 拓展阅读

#### bitcode不是bytecode

bitcode不能翻译为字节码(bytecode)，显然从字面上看这两个词代表的含义并不等同：字节码是按照字节存取的，一般其控制代码的最小宽度是一个字节(也即8个bits)，而bitcode是按位(bit)存取，最大化利用空间。比如用bitcode中使用`6-bit characters`来编码只包含字母/数字的字符串

```
'a' .. 'z' ---  0 .. 25 ---> 00 0000 .. 01 1001
'A' .. 'Z' --- 26 .. 51 ---> 01 1010 .. 11 0011
'0' .. '9' --- 52 .. 61 ---> 11 0100 .. 11 1101
       '.' --- 62       ---> 11 1110
       '_' --- 63       ---> 11 1111
```

在这种编码模式下，4字节的字符串`abcd`只用3个字节就可以表示

```
  char:     a   |    b   |    c   |    d
binary: 00 00 00|00|00 01|00 00|10|00 00 11
   hex:     00     |     10    |    83
```

完整的编码格式可以参考官方文档[LLVM Bitcode File Format](http://llvm.org/docs/BitCodeFormat.html)

#### bitcode的兼容性

bitcode的格式目前是一直在变化的，并且无法向前兼容，举例来说Xcode8的编译器无法读取并解析xcode9产生的bitcode。

另外苹果的bitcode格式与社区版LLVM的bitcode有一定差异，但苹果并不会及时开源Xcode最新版编译器的代码，所以如果你使用第三方基于社区版LLVM制作的编译器进行开发，不要尝试开启并提交bitcode到App Store Connect，否则会因为App Store Connect解析不了你的bitcode而被拒。

#### bitcode不是架构无关代码

如果一个app同时要支持armv7和arm64两种架构，那么同一个源代码文件将会被编译出两份bitcode，也就是说，在一开始介绍LLVM的那张图中，并不是代表同一份bitcode代码可以直接被编译为不同目标机器的机器码。

LLVM只是统一了中间语言的结构和语法格式，但不能像Java那样，Compile Once & Run Everywhere.

#### 如何判断是否开启bitcode

可以通过otool检查二进制文件，网上有很多类似这样的方法：

```bash
otool -arch armv7 -l xxxx.a | grep __LLVM | wc -l
```

通过判断是否包含 `__LLVM` 或者关键字来判断是否支持bitcode，其实这种方式是完全错误的，通过前面的测试可以知道，这种方式区分不了bitcode和bitcode-marker，确定是否包含bitcode，还需要检查otool输出中`__LLVM` Segment 的长度，如

```bash
$ otool -l test_bitcode.o | grep -A 2  __LLVM | grep size
      size 0x0000000000000b10
      size 0x0000000000000042
$ otool -l test_bitcode_marker.o | grep -A 2  __LLVM | grep size
      size 0x0000000000000001
      size 0x0000000000000001
```

#### bitcode是否能反编译出源代码

从科学严谨的角度来说，无法给出确定的答案，但是这个问题跟“二进制文件是否能反编译出源代码”是一样的道理。编译是一个将源代码一层一层不断低级化的过程，每一层都可能会丢失一些特性，产生不可逆的转换，把源代码编译为bitcode或是二进制机器码是五十步之于百步的关系。在通常情况下，反编译bitcode要比反编译二进制文件更容易，但通过bitcode反编译出和源代码语义完全相同的代码，也是几乎不可能的。

另外，从安全的角度考虑，Xcode 引入了 `Symbol Hiding` 和 `Debug info Striping` 机制，在链接时，bitcode中所有非导出符号均被隐藏，取而代之的是 `__hidden#0_` 或者 `__ir_hidden#1_` 这样的形式，debug信息也只保留了line-table，所有跟文件路径、标识符、导出符号等相关的信息全部都从bitcode中移除，相当于做了一层混淆，防止源代码级别的信息泄露，可谓是煞费苦心。