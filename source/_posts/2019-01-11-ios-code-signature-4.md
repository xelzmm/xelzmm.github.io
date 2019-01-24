---
layout: post
title: "细说iOS代码签名(四)"
date: 2019-01-11 11:12:14 +0800
comments: true
categories: ios
typora-root-url: ../../source
keywords: "ios, codesign, code signature, resign, developer certificate, entitlements, provisioning profile, mobileprovision"
description: "All you need to know about iOS Code Signature. What is codesign？ Why do we need codesign？ How to resign an app？How does iDevice verify codesign? 什么是代码签名？为什么要进行代码签名？ iOS代码签名的结构是什么？ 代码签名是如何被校验的？如何重签名？"
---

#### 导航 

- 一口气读完，大约需要40-60分钟
  - [深度长文：细说iOS代码签名](/blog/2019/01/11/ios-code-signature/)
- 分步阅读
  - [细说iOS代码签名(一)](/blog/2019/01/11/ios-code-signature-1/)：签名的作用及原理
  - [细说iOS代码签名(二)](/blog/2019/01/11/ios-code-signature-2/)：开发者证书、Entitlements、Provisioning Profile
  - [细说iOS代码签名(三)](/blog/2019/01/11/ios-code-signature-3/)：签名的过程及代码签名的数据结构
  - [细说iOS代码签名(四)](/blog/2019/01/11/ios-code-signature-4/)：签名校验、越狱、重签名

## 0x06 签名的校验

签名的校验并非一次性完成，在安装、启动、和运行时有着不同的校验规则。

<!-- more -->

#### 安装

App安装时的校验由位于iOS设备上的/usr/lib/libmis.dylib (dyld_shared_cache)提供。

![](/assets/2019/libmis.png)

App的安装是由`/usr/libexec/installd`完成的，`installd`会通过`libmis.dylib`校验ProvisioningProfile、Entitlements及签名的合法性，并递归地校验签名时每一个步骤生成的哈希值：CDHash, Code Directory, _CodeSignature/CodeResources。

```bash
$ otool -L installd | grep mis
	/usr/lib/libmis.dylib (compatibility version 1.0.0, current version 1.0.0)   
$ nm installd | grep ValidateSignature
                 U _MISValidateSignatureAndCopyInfo
                 U _kMISValidationOptionValidateSignatureOnly    
```

#### 启动

进程启动时，loader会先将可执行文件加载到虚拟内存，在加载的过程中mach_loader会自动解析MachO文件中的LC_CODE_SIGNATURE并进行校验，可以参考mach_loader的代码 [bsd/kern/mach_loader.c](https://opensource.apple.com/source/xnu/xnu-4570.71.2/bsd/kern/mach_loader.c.auto.html)

![](/assets/2019/verify1.png)

`load_code_signature`在解析完签名的数据后会调用`mac_vnode_check_singature`函数进行验证，而这个函数会被名为`AFMI`(AppleMobileFileIntegrity)的内核扩展(kext)通过Hook的方式接管，而AFMI只是一层壳，最终也是调用了libmis.dylib来实现签名的校验，这一校验过程基本与安装时一致，防止安装后的篡改。

需要注意的是，加载过程中为了提升加载效率，签名校验并不会去检查Code Directory与实际的代码是否匹配，仅仅只检查了CMS Signature及CDHash的合法性。

#### 运行时

当一页代码被加载到虚拟内存后，会立即触发`page fault`，此时内核中的`vm_fault`函数会被调用，紧接着调用`vm_fault_enter`，在`vm_fault_enter`的实现中会判断代码页是否需要签名校验，并执行校验的操作，参考代码[osfmk/vm/vm_fault.c](https://opensource.apple.com/source/xnu/xnu-4570.71.2/osfmk/vm/vm_fault.c.auto.html)

```c
kern_return_t vm_fault_enter(...) {
// ...
    /* Validate code signature if necessary. */
	if (VM_FAULT_NEED_CS_VALIDATION(pmap, m, object)) {
		vm_object_lock_assert_exclusive(object);

		if (m->cs_validated) {
			vm_cs_revalidates++;
		}

		/* VM map is locked, so 1 ref will remain on VM object -
		 * so no harm if vm_page_validate_cs drops the object lock */
		vm_page_validate_cs(m);
	}
// ...
}
```

对于宏`VM_FAULT_NEED_CS_VALIDATION`的解释是

```c
/*
* CODE SIGNING:
* When soft faulting a page, we have to validate the page if:
* 1. the page is being mapped in user space
* 2. the page hasn't already been found to be "tainted"
* 3. the page belongs to a code-signed object
* 4. the page has not been validated yet or has been mapped
for write. */
#define VM_FAULT_NEED_CS_VALIDATION(pmap, page)
((pmap) != kernel_pmap /*1*/ && !(page)->cs_tainted /*2*/ && (page)->object->code_signed /*3*/ && (!(page)->cs_validated || (page)->wpmapped /*4*/))
```

`vm_page_validate_cs`会计算当前代码页的哈希值，并与签名中CodeDirectory记录的值进行比对，完成代码签名的验证。如果不符，且不满足系统预设的例外条件，则会向内核发出CS_KILL指令，将进程结束。

至此签名的校验流程就全部完成了。

## 0x07 越狱与重签名

#### 越狱

越狱之后，签名校验机制会被破坏掉，否则用于实现越狱的代码自身就无法运行。比如在iOS6/7时代，典型的方式是替换 `libmis.dylib`中的`_MISValidateSignature`函数，使其永远返回验证成功，简单粗暴但很有效，因此越狱的设备可以不受签名限制运行任意程序。但是单纯解决掉这个函数只是解决了MachO文件的Load问题，运行时仍然会有沙盒和Code Directory的校验，想要对系统完全的控制权必须同时解决掉这两个问题。

由于沙盒机制的实现分散在系统的各个角落，没有简单的方式可以将沙盒一刀切地屏蔽掉，因此一般越狱并不会破坏掉沙盒。但因为越狱设备签名校验机制被绕过，不再会根据embedded.mobileprovision文件检查Entitlements的合法性，因此我们可以在沙盒范围内，声明任意的权限。Code Directory的校验在内核层，破解难度相对较大，并且完全没有必要进行破解，因为Code Directory只是单纯地校验未加密的哈希值而已，只需要按照代码签名的格式做好Code Directory即可。

越狱之父Saurik为此创造了[ldid](http://iphonedevwiki.net/index.php/Ldid)这个工具，用于给越狱设备上的程序制造"假"的签名。使用ldid进行签名只需要指定一个可选的`Entitlements`文件，签名之后，产生的LC_CODE_SIGNATURE中只会两个有效的Blob，分别是 Code Directory和 Entitlements，并没有最重要的CMS Signature部分，因为`_MISCalidateSignature`永远都会告诉系统签名是正确的。

```bash
$ cp TestCodeSign TestCodeSign.ldid
$ ldid -Sxxx.entitlements TestCodeSign.ldid
$ jtool --sig TestCodeSign.ldid -arch arm64
Blob at offset: 54016 (928 bytes) is an embedded signature
Code Directory (442 bytes)
    ...
 Empty requirement set (12 bytes)
Entitlements (424 bytes) (use --ent to view)
```

#### 重签名

有的时候出于各种原因，我们需要对一个App进行重签名，然后在自己的设备上进行测试。回顾一下签名的必备条件：

- 开发者证书，以及对应的密钥
- Entitlements文件
- embedded.mobileprovision

开发者证书和密钥我们已经有了，对于Entitlements和embedded.mobileprovision文件，为了确保重签后的App能够正常运行，必须使用和原App相同或者至少包含原App所需权限的Entitlements文件。这个并不难操作，只需要新建一个工程，开启相应的功能，让Xcode自动为我们生成即可。但是Entitlements文件中还有一些跟Team ID和App ID相关的配置，这两个是没有办法伪造的，因为我们不能使用已经被其他开发者注册过的ID。使用自己的ID一般也不会有什么问题，但在某些情况下可能导致最终的程序逻辑出现异常，这根具体的代码实现细节有关。

现在，只要确保有正确的Entitlements文件，Provisioning Profile与Entitlements文件匹配，且包含重签时使用的证书及目标设备的UUID，就可以进行重签名了，如果重签名后无法安装，请检查Provisioning Profile文件是否满足上述条件。

Entitlements文件中还标识了`application-identifier`，也就是Bundle ID，正常签名的App中，这个值和Info.plist中的`CFBundleIdentifier`的值是相同的，但实际在签名校验过程中，系统并不会检查二者是否一致。因此即使Entitlements中与Info.plist文件使用了不同的Bundle ID，理论上也不会影响重签名之后的运行。

需要注意，App中除了可执行程序文件外，还会可能会有Frameworks及Plugins，里面都会包含二进制的代码文件，他们的哈希值也会被存储在 _CodeSignature/CodeResources中。所有的二进制代码都必须进行签名，而签名后二进制文件的哈希值就会产生变化，因此需要先对这两个文件夹下的二进制文件进行签名，再对App进行签名。

重签名的基本流程，使用-f参数可以强制覆盖掉已有的签名

```bash
$ # 对Frameworks及Plugins中的每一个文件进行签名，此时不需要指定entitlements
$ codesign -f -s "证书名称或者SHA1值" Target.app/Frameworks/xxxxx.framework
$ codesign -f -s "证书名称或者SHA1值" Target.app/Frameworks/libxxxx.dylib
$ ...
$ # 将准备好的Provisioning Profile拷贝到App根目录
$ cp ~/Library/MobileDevice/Provisioning\ Profiles/xxxxx.mobileprovision Target.app/embedded.mobileprovision
$ # 对App进行签名
$ codesign -f -s "证书名称或者SHA1值" --entitlements resign.entitlements Target.app
```

## 0x08 References

|   reference    |    link    |
|----------------|------------|
| Code Signing Guide | [https://developer.apple.com/...](https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/Introduction/Introduction.html) 
| ASN.1 JavaScript decoder | [http://lapo.it/asn1js/](http://lapo.it/asn1js/) 
| Cryptographic Message Syntax (CMS) | [https://www.ietf.org/rfc/rfc3852.txt](https://www.ietf.org/rfc/rfc3852.txt) 
| iSign in python | [https://github.com/saucelabs/isign](https://github.com/saucelabs/isign) 
| CodeSigning (RSACon 2015) | [http://newosxbook.com/articles/CodeSigning.pdf](http://newosxbook.com/articles/CodeSigning.pdf) 
| jtool | [http://www.newosxbook.com/tools/jtool.html](http://www.newosxbook.com/tools/jtool.html) 
| mistool | [http://newosxbook.com/tools/mistool.html](http://newosxbook.com/tools/mistool.html) 
| evasi0n7 jailbreak writeup |[https://geohot.com/e7writeup.html](https://geohot.com/e7writeup.html) 
| iOS hacker's handbook | [https://books.google.com.hk/books?id=1kDcjKcz9GwC](https://books.google.com.hk/books?id=1kDcjKcz9GwC) 

