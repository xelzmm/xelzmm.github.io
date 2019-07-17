---
layout: post
title: "细说iOS代码签名(三)"
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

## 0x05 CodeSign

万事具备，只欠东风，已经具备了签名所需的所有条件，接下来就可以开始研究签名的具体过程了。

<!-- more -->

在编译iOS App时，Xcode在编译的打包的流程中会自动进行代码签名， 可以在编译日志界面找到一个`Sign`的步骤，内部是调用了`codesign`这个命令对app进行签名

![codesign](/assets/2019/sign1.png)

codesign有几个关键参数

- `--sign sign_identity` 指定签名所用的证书，可以指定证书的名字，比如`"iPhone Developer: xxx (xxx)"`也可以直接写证书文件的sha1值，xcode中就是直接指定sha1值的。通过观察图中的sha1值可以看出xcode自动选择了刚申请的最新证书。
- `--entitlements entitlements_file` 指定签名所需要的entitlements文件，这里的entitlements文件跟前面看到的并不是同一个文件，而是基于原有entitlements文件，补充上缺省权限后生成的临时文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>application-identifier</key>
	<string>xxxxxxxxxx.test.CodeSign</string>
	<key>com.apple.developer.team-identifier</key>
	<string>xxxxxxxxxx</string>
	<key>get-task-allow</key>
	<true/>
	<key>inter-app-audio</key>
	<true/>
</dict>
</plist>
```

如果想对比签名前后的区别，可以在`Build Settings`中找到`Code Signing Identity`，选择`Other`并将内容清除(即设置为空)，即可跳过代码签名。分别编译一个不签名的版本和签名的版本，对比可以发现

![compare](/assets/2019/sign2.png)

- 签名过的app中多了一个`_CodeSignature`文件夹，里面只有一个文件`CodeResources`
- 还多了一个`embedded.mobileprovision` 文件
- 二进制文件的内容存在差异，并且签名后体积变大了

其中`embedded.mobileprovision`就是前文提到的Provisioning Profile文件，它直接被拷贝到了app的根目录并重命名，在此不再赘述，重点研究下另外两个不同点。

#### _CodeSignature/CodeResources

首先是`_CodeSingature/CodeResources`，这是一个plist文件，里面保存了app中每个文件（除了App的可执行文件）的`明文哈希值`

```xml
<plist version="1.0">
<dict>
	<key>files</key>
	<dict>
        <key>Base.lproj/Main.storyboardc/Info.plist</key>
        <data>
            MDrKFvFWroTb0+KEbQShBcoBvo4=
        </data>
		...
	</dict>
	<key>files2</key>
	<dict>
        <key>Base.lproj/Main.storyboardc/Info.plist</key>
        <dict>
            <key>hash</key>
            <data>
                MDrKFvFWroTb0+KEbQShBcoBvo4=
            </data>
            <key>hash2</key>
            <data>
                PpvapAjR62rl6Ym4E6hkTgpKmBICxTaQXeUqcpHmmqQ=
            </data>
        </dict>
		...
	</dict>
	<key>rules</key>
	...
	<key>rules2</key>
	...
</dict>
</plist>
```

`files`和`files2`分别是旧版本和新版本的文件列表，而`rules`与`rules2`分别是与之对应的规则说明，里面描述了计算hash时需要被排除的文件以及每个文件的权重。

`files`中保存的是每个文件的sha1值，而`files2`中同时保存了sha1和sha256，因为sha1在计算机硬件高度发达的今天，已经相对没有那么安全了，因此最新的签名算法中，引入了sha256。注意，这里的hash值都是base64编码的明文，有些文章说这些值是使用私钥加密的哈希，这是很不负责任的错误说法，通过几条简单的命令就可以进行验证：

```bash
$ cat Base.lproj/Main.storyboardc/Info.plist | shasum -a 1
303aca16f156ae84dbd3e2846d04a105ca01be8e  -
$ echo -n 'MDrKFvFWroTb0+KEbQShBcoBvo4=' | base64 -D | hexdump
0000000 30 3a ca 16 f1 56 ae 84 db d3 e2 84 6d 04 a1 05
0000010 ca 01 be 8e
$ # =========== 分割线 ===========
$ cat Base.lproj/Main.storyboardc/Info.plist | shasum -a 256
3e9bdaa408d1eb6ae5e989b813a8644e0a4a981202c536905de52a7291e69aa4  -
$ echo -n 'PpvapAjR62rl6Ym4E6hkTgpKmBICxTaQXeUqcpHmmqQ=' | base64 -D | hexdump
0000000 3e 9b da a4 08 d1 eb 6a e5 e9 89 b8 13 a8 64 4e
0000010 0a 4a 98 12 02 c5 36 90 5d e5 2a 72 91 e6 9a a4
```

`_CodeSignature/CodeResources`文件的主要作用是保存签名时每个文件的哈希值，而这些哈希值并不需要都进行加密，因为非对称加密的性能是比较差的，全部都加密只会拖慢签名和校验的速度。其实只需要确保这个文件没有被篡改，自然也就可以确保每个文件都是签名时的原始状态，这一点在后续的内容中可以得到验证。

#### LC_CODE_SIGNATURE

使用`otool -l`对比签名前后的二进制文件，可以发现签名后二进制文件多了一个名为`LC_CODE_SIGNATURE`的Load Command

```bash
$ otool -l TestCodeSign | tail -n 5
Load command 21
      cmd LC_CODE_SIGNATURE
  cmdsize 16
  dataoff 54016
 datasize 19888
```

MachOView中查看如下

![](/assets/2019/codesign1.png)

代码签名是一段纯二进制的数据，可以在[https://opensource.apple.com/source/Security/Security-55471/sec/Security/Tool/codesign.c.auto.html](https://opensource.apple.com/source/Security/Security-55471/sec/Security/Tool/codesign.c.auto.html) 看到一些结构定义，结合数据定义来分析

![](/assets/2019/codesign2.png)

```c
// 红色部分①  Offset: 0xD300 = 54016 LC_CODE_SIGNATURE->dataoff
struct __SuperBlob {
    uint32_t magic;	  /* 0xFADE0CC0 = CSMAGIC_EMBEDDED_SIGNATURE */
    uint32_t length;  /* 0x1A1E -> 6686 */
    uint32_t count;	  /* 5 */
    CS_BlobIndex index[];  /* 蓝色部分 */
}
// 蓝色部分②  5个BlobIndex
struct __BlobIndex {
    uint32_t type;    /* 0x0 -> Code Directory */
    uint32_t offset;  /* 0x34 -> 0xD300 + 0x34 = 0xD334 指向绿色③*/
}
struct __BlobIndex {
    uint32_t type;    /* 0x2 -> Requirements */
    uint32_t offset;  /* 0x221 -> 0xD300 + 0x221 = 0xD521 */
}
struct __BlobIndex {
    uint32_t type;    /* 0x5 -> Entitlements */
    uint32_t offset;  /* 0x2CD -> 0xD300 + 0x2CD = 0xD5CD */
}
struct __BlobIndex {
    uint32_t type;    /* 0x1000 -> Code Directory */
    uint32_t offset;  /* 0x475 -> 0xD300 + 0x475 = 0xD775 */
}
struct __BlobIndex {
    uint32_t type;    /* 0x10000 -> CMS Signature */
    uint32_t offset;  /* 0x746 -> 0xD300 + 0x746 = 0xDA46 */
}
```
这部分是典型的数据头结构，声明了5个Blob，以及每个Blob的类型和相对签名头部的偏移量。接下来把每个部分分别提取出来进行分析。

#### CodeDirectory

CodeDirectory是签名数据中最终要的部分，直译过来就是代码目录，其实里面是整个MachO文件的哈希值，这里的哈希并不是一次性对整个文件进行哈希，而是将MachO文件按照pageSize(一般是4k也就是4096字节)进行分页，每一页单独计算哈希，并按照顺序保存下来，就像目录一样。

细心的同学会发现上面的数据中出现了两个CodeDirectory，type分别是`0x0`和`0x1000`，这也是历史遗留问题，`0x0`对应的是旧版本的代码签名，使用sha1算法进行哈希值的计算，而`0x1000`是后来引入的，采用sha256作为哈希算法，除了算法和哈希的长度不同之外，其他内容基本是一样的。取第一个进行分析：

```c
// 绿色部分③ Offset: 0xD334
struct __CodeDirectory {
    uint32_t magic;         /* 0xFADE0C02 -> CSMAGIC_CODEDIRECTORY */
    uint32_t length;        /* 0x1ED -> 493 */
    uint32_t version;       /* 0x00020400 -> v2.4.0 */
    uint32_t flags;         /* 0 */
    uint32_t hashOffset;    /* 0xD5 -> 0xD334 + 0xD5 = 0xD409 指向⑤*/
    uint32_t identOffset;   /* 0x58 -> 0xD334 + 0x58 = 0xD38B 指向④*/
    uint32_t nSpecialSlots; /* 5 */
    uint32_t nCodeSlots;    /* 0xE -> 14 */
    uint32_t codeLimit;     /* 0xD300 */
    uint8_t hashSize;       /* 0x14 -> 20bytes -> 160bits (sha1) */
    uint8_t hashType;       /* 0x01 (sha1) */
    uint8_t spare1;         /* unused (must be zero) */
    uint8_t pageSize;       /* 0x0C -> 2 ^ 0x0C = 0x1000 = 4096 */
    uint32_t spare2;        /* unused (must be zero) */
    /* followed by dynamic content as located by offset fields above */
}
```

hashOffset就是"目录"第一页的偏移，从这个位置(0xD409)可以提取到一串20字节的sha1值(图中黄色⑤):

```
9D452342F9ED06189E4F099BCA7CB68D6432F775
```

这个值代表的就是该文件第一页的哈希值，通过以下命令计算文件前4096字节的sha1可进行验证

```bash
$ dd bs=1 skip=0 count=0x1000 if=TestCodeSign 2>/dev/null | shasum -a 1
9d452342f9ed06189e4f099bca7cb68d6432f775  -
```

而紧接着的20个字节就是第二页的哈希值，以此类推，直到原始文件的最后一页。

由于文件不一定是pageSize的整数倍，最后一页往往不足"一整页"的大小，因此需要额外的字段`codeLimit`记录文件的实际大小，也就是需要签名的数据的实际大小，通过这个值计算出最后一页的实际大小，并提取相应数据计算最后一页的签名。例子中`codeLimit=0xD300`，很容易得出最后一页大小为`0x300`

```bash
$ dd bs=1 skip=0xD000 count=0x300 if=TestCodeSign 2>/dev/null | shasum -a 1
9dc960fc86f803c1fa100f2a1145cf7cbe58e803  -
```

计算出最后一页的sha1值与CodeDirectory中(图中黄色⑥)一致。

nCodeSlots记录了文件的总页数14，可通过`0xD300 / 0x1000 = 13.1875`得出确实是14页。

细心的朋友已经发现了，④ identifier和 ⑤ hashSlots 之间有一段多出的数据⑦，并且CodeDirectory中还有一个奇怪的值`nSpecialSlots=5`，整个文件的哈希值都已经包含在⑤和⑥之间了，这多出来的数据是怎么回事呢？

原来，在第一页的前面，还有5个特殊的负数页，用来保存这些额外信息的哈希值。

| 序号 | 对应内容                                              |
| ---- | ----------------------------------------------------- |
| -1   | App根目录的Info.plist文件                            
| -2   | Requirements(代码签名的第二部分)                      
| -3   | Resource Directory (_CodeSignature/CodeResources文件) 
| -4   | 暂未使用                                              
| -5   | Entitlements (代码签名的第三部分)                     

同样地，出于性能考虑，这些哈希值并未经过任何加密，只需要确保这些哈希值未经篡改，就可以说明代码本身没有被篡改。

#### Requirements

用于指定签名校验时的一些额外的约束，签名时codesign命令会自动生成这部分数据，但目前并没有看到什么地方使用了它，就不深入分析了，官方文档有对这部分内容的详细描述

- [Code Signing Tasks](https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/Introduction/Introduction.html)
- [Code Signing Requirement Language](https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/RequirementLang/RequirementLang.html#//apple_ref/doc/uid/TP40005929-CH5-SW1)

#### Entitlements

![](/assets/2019/codesign3.png)

通过头部的偏移定位到数据的位置，显然，这是一个Blob结构

```c
struct __Blob {       /* Address: 0xD5CD */
    uint32_t magic;   /* 0xFADE7171 -> CSMAGIC_ENTITLEMENT */
    uint32_t length;  /* 0x1A8 -> 424 */
}
```

之前由Xcode生成的Entitlements文件被整个嵌入到签名数据中。

#### CMS Signature

CMS是`Cryptographic Message Syntax`的缩写，是一种标准的签名格式，由[RFC3852](https://www.ietf.org/rfc/rfc3852.txt)定义。还记得Provisioning Profile的签名吗？它们是相同的格式。CMS格式的签名中，除了包含前面我们推导出的加密哈希和证书之外，还承载了一些其他的信息。由于是二进制格式，不方便分析，可以将其内容从MachO文件中剥离出来，再找合适的工具进行解析。根据偏移量定位到CMS Signature的位置`0xDA46` 

![](/assets/2019/codesign4.png)

```c
struct __Blob {       /* Address: 0xDA46 */
    uint32_t magic;   /* 0xFADE0B01 -> CSMAGIC_BLOBWRAPPER */
    uint32_t length;  /* 0x12D8 -> 4824 */
}
```

除去头部的8个字节，把对应的内容提取出来

```bash
$ dd bs=1 skip=0xDA4E count=0x12D0 if=TestCodeSign of=cms_signature
```

可以将导出的cms_signature文件上传到[在线ASN.1解析工具](http://lapo.it/asn1js/)(支持CMS格式解析)进行分析

![](/assets/2019/codesign5.png)

文件被解析为树状结构，看起来还是不够直观，因为这个工具只是按照数据格式把内容进行了格式化，但是并没有标注所有字段的确切含义。其实我们还可以使用openssl进行查看，但是因为Mac上自带的openssl以及通过HomeBrew安装的openssl都是没有开启cms支持的，所以可以将文件拷贝到linux机器上或者自行编译openssl进行查看，具体方法在此不表。

```bash
$ openssl cms -cmsout -print -inform DER -in cms_signature
CMS_ContentInfo:
  contentType: pkcs7-signedData (1.2.840.113549.1.7.2)
  d.signedData:
    version: 1
    digestAlgorithms:
        algorithm: sha256 (2.16.840.1.101.3.4.2.1)
        parameter: NULL
    encapContentInfo:
      eContentType: pkcs7-data (1.2.840.113549.1.7.1)
      eContent: <ABSENT>
    certificates:
      ... [stripped] Apple Worldwide Developer Relations Certification Authority
      ... [stripped] Apple Root CA
      ... [stripped] iPhone Developer: xxxxxxx
    signerInfos:
        version: 1
        d.issuerAndSerialNumber:
          issuer: C=US, O=Apple Inc., OU=Apple Worldwide Developer Relations, CN=Apple Worldwide Developer Relations Certification Authority
          serialNumber: 1008862887770590428
        digestAlgorithm:
          algorithm: sha256 (2.16.840.1.101.3.4.2.1)
          parameter: NULL
        signedAttrs:
            ... [stripped]
              SEQUENCE:
    0:d=0  hl=2 l=  29 cons: SEQUENCE
    2:d=1  hl=2 l=   5 prim:  OBJECT            :sha1
    9:d=1  hl=2 l=  20 prim:  OCTET STRING      [HEX DUMP]:669421362B2F2B5303BCEBB47D793A75A6BBD32F

            ... [stripped]
        signatureAlgorithm:
          algorithm: rsaEncryption (1.2.840.113549.1.1.1)
          parameter: NULL
        signature:
          0000 - 77 00 50 9c 5c 6d 50 1e-cb 4b ca b7 91 d3 5b   w.P.\mP..K....[
          000f - 2e 28 fe f3 5d 20 73 ef-0a 59 ac 2e ed bd 2a   .(..] s..Y....*
          ... [stripped]
        unsignedAttrs:
          <EMPTY>
```

由于输出内容太多，将部分内容做了删减，可以观察到签名中主要包含了这些内容

- **contentType**， 表明消息的类型，有6种取值，这里使用的是表示签名的signedData类型
  - Data
  - SignedData
  - EnvelopedData
  - DigestedData
  - EncryptedData
  - AuthenticatedData
- **content**，SignedData类型的数据
  - version等：略
  - certificates： 证书链，包含用于签名的开发者证书及所有上游CA的证书
  - signerInfos：真正的签名信息！
    - version：版本号
    - issuerAndSerialNumber：签名者信息，根据签名者的名称找到证书链中对应的证书，使用证书中的公钥即可验证签名是否有效
    - digestAlgorithm：哈希算法
    - signedAttrs：需要签名的属性, 是可选项，为空表示被签名的数据是原始文件的内容，如果不为空则至少要包含原始文件的类型以及其哈希值，此时被签名的数据就是signedAttrs的内容
    - signatureAlgorithm：签名算法，这里指对哈希值进行加密所使用的算法
    - signature：加密后的哈希值

由于在Code Directory中已经保存了所有资源及代码的哈希值，那么我们只需要确保CodeDirectory不被篡改，即可确保整个app的完整性， 因此CMS Signature中只需要对CodeDirectory进行签名即可。而signedAttrs中支持这样一种特性：可以先计算被签名数据的哈希，然后再对哈希值进行签名。听起来有点绕，不过仔细体会一下应该不难理解。

我们把CodeDirectory的内容抠出来，计算其哈希值，以第一个CodeDirectory为例，计算其sha1：

```bash
$ dd bs=1 skip=0xD334 count=0x1ED if=TestCodeSign 2>/dev/null | shasum -a 1
669421362b2f2b5303bcebb47d793a75a6bbd32f  -
```

这个值叫做CDHash(Code Directory's Hash)，对比前面从cms_signature中解析出的 signedAttrs，会发现这两个值是一样的，也就是说CodeDirectory的哈希值被放在了signerInfos->signedAttrs中，作为最终真正被`签名`(计算哈希并加密)的内容。

根据[RFC5652 - Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652#section-5.4)中的规定，整个signedAttrs的内容会作为最终被签名的对象，我们可以按照RFC的规则来手动验证签名的计算过程。结合在线ASN.1解析工具的解析结果，定位到signedAttrs的偏移量为4016，先将这部分内容通过dd或者openssl命令提取出来，由于dd命令需要知道偏移和长度，而openssl可以直接将指定起始位置的整个节点dump出来，使用openssl会更为方便一些

```bash
$ openssl asn1parse -in cms_signature -inform DER -strparse 4016 -noout -out signedAttrs
$ hexdump signedAttrs | head -n 1
0000000 a0 82 02 25 30 18 06 09 2a 86 48 86 f7 0d 01 09
```

这是一段ASN.1编码的数据，使用BER(BasicEncoding Rules)规则编码，在编码时，表示`SET OF`的tag(编码为0x31)会被替换为`IMPLICIT [0]`(编码为0xA0)，因此，在计算时需要将数据还原，即将首字节`a0`替换回`31`。

```bash
$ dd bs=1 skip=1 count=1000 if=signedAttrs of=signedAttrs_1
$ (echo -en '\x31'; cat signedAttrs_1) > signedAttrs_2
$ hexdump signedAttrs_2 | head -n 1
0000000 31 82 02 25 30 18 06 09 2a 86 48 86 f7 0d 01 09
```

计算其哈希值，由于singerInfos->digestAlgorithm指明了使用sha256，所以我们计算这个文件的sha256值

```bash
$ shasum -a 256 signedAttrs_2
8ea2964f63f4066b31092c08dae2dfcdb42b10e7b4658c69679eae015d7f0366  signedAttrs_2
```

这个hash值最终会使用开发者证书对应的私钥进行加密，得到签名数据，并保存在signerInfos->signature中。如果要验证签名，则需要使用公钥对签名数据进行解密， 再将解密后的数据与上述hash值进行对比。

首先先从文件中分别提取签名的开发者证书和最终的签名数据，然后再从开发者证书中提取公钥对其进行解密

```bash
# 提取证书链，cert0即为签名证书，和前文申请到的开发者证书是完全一样的
$ codesign -d --extract-certificates=cert TestCodeSign
$ shasum cert0 ios_development.cer
11447116f2c5521b057b9b67290f0fdadeadfa0a  cert0
11447116f2c5521b057b9b67290f0fdadeadfa0a  ios_development.cer
# 从cms_signature文件中偏移4584处提取最终的签名数据，保存为signature
# 这部分内容是使用开发者的私钥对signedAttrs的hash值进行加密而来的
$ openssl asn1parse -in cms_signature -inform DER -strparse 4584 -noout -out signature
# 提取签名证书中的公钥，保存为pub_key.pem
$ openssl x509 -inform DER -in cert0 -pubkey -noout > pub_key.pem
# 使用公钥对签名数据进行解密，并对解密出的数据按照asn.1格式进行解析
$ openssl rsautl -in signature -verify -asn1parse -inkey pub_key.pem -pubin
    0:d=0  hl=2 l=  49 cons: SEQUENCE
    2:d=1  hl=2 l=  13 cons:  SEQUENCE
    4:d=2  hl=2 l=   9 prim:   OBJECT            :sha256
   15:d=2  hl=2 l=   0 prim:   NULL
   17:d=1  hl=2 l=  32 prim:  OCTET STRING
      0000 - 8e a2 96 4f 63 f4 06 6b-31 09 2c 08 da e2 df cd   ...Oc..k1.,.....
      0010 - b4 2b 10 e7 b4 65 8c 69-67 9e ae 01 5d 7f 03 66   .+...e.ig...]..f
```

解密后的数据， 可以看出跟我们自己计算的signedAttrs的hash值是相同的，如此一来也就完成了整个代码签名的校验。

至此，我们已经从头到尾剖析了iOS代码签名的生成方式及数据结构，在这个过程中，至少存在4次计算哈希的行为，并且是环环相扣的

1. _CodeSignature/CodeResources中对每个资源文件计算哈希
2. Code Directory 中对MachO文件本身的每个分页，以及Info.plist、CodeResources、Entitlements等文件计算哈希
3. CMS Signature的signedAttrs中对Code Directory计算哈希
4. 对signedAttrs计算哈希并使用开发者的私钥加密

只有最后一步的哈希值是被加密的， 前面几步的哈希值是否加密都不影响签名的效果，只要任意内容有变化，均会因某个环节的哈希不匹配而导致签名校验的失败。

#### jtool

相信上面的二进制分析已经让你眼花缭乱了，不过已经有大神做出了[jtool](http://www.newosxbook.com/tools/jtool.tar)这个工具，它是一款强大的MachO二进制分析工具，用来替代otool、nm、segedit等命令，也包括codesign的部分功能。通过以下命令可以将代码签名解析为可读的文本格式

```bash
$ jtool --sig -vv TestCodeSign
Blob at offset: 54016 (19888 bytes) is an embedded signature of 6686 bytes, and 5 blobs
	Blob 0: Type: 0 @52: Code Directory (493 bytes)
		Version:     20400
		Flags:       none (0x0)
		CodeLimit:   0xd300
		Identifier:  test.CodeSign (0x58)
		Team ID:     xxxxxxxxxx (0x66)
		Executable Segment: Base 0x00000000 Limit: 0x00000000 Flags: 0x00000000
		CDHash:	     669421362b2f2b5303bcebb47d793a75a6bbd32f (computed)
		# of Hashes: 14 code + 5 special
		Hashes @213 size: 20 Type: SHA-1
			Entitlements blob:	19a92ca549e53593b384681245de14897df2a9dd (OK)
			Application Specific:	Not Bound
			Resource Directory:	fb7df05e17f3b347d6b64868f468def49feecf25 (OK)
			Requirements blob:	9d58965211c9cd83b208fffd575d741881ff81e4 (OK)
			Bound Info.plist:	89e1951413c3eb05fab8f6a5f06c13b48926eabe (OK)
			Slot   0 (File page @0x0000):	9d452342f9ed06189e4f099bca7cb68d6432f775 (OK)
			... [stripped]
	... [stripped]
	Blob 4: Type: 10000 @1862: Blob Wrapper (4824 bytes) (0x10000 is CMS (RFC3852) signature)
CA: Apple Certification Authority CN: Apple Root CA
... [stripped]
Time: 190122095805Z
```

#### Distribute App

在Xcode Organizer中导出或者提交App时，Xcode会将Entitlements文件及embedded.mobileprovision文件替换为对应的版本，并使用对应的证书重新签名，主要区别如下

| 类型        | Entitlements             | Provisioning Profile       | 证书           |
| ----------- | ------------------------ | -------------------------- | -------------- |
| AppStore    | 不可调试，推送为生产环境 | 无ProvisionedDevices       | 发布证书       
| Ad Hoc      | 不可调试，推送为生产环境 | 允许安装到已注册的测试设备 | 发布证书       
| Development | 可调试，推送为测试环境   | 允许安装到已注册的测试设备 | 开发证书       
| Enterprise  | 不可调试，推送为生产环境 | ProvisionAllDevices        | 企业级发布证书 

本篇完。

------

下一篇：[细说iOS代码签名(四)](/blog/2019/01/11/ios-code-signature-4/)：签名校验、越狱、重签名