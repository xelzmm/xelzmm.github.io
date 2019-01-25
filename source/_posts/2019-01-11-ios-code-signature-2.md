---
layout: post
title: "细说iOS代码签名(二)"
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

## 0x03 开发者证书

在了解了签名和证书的基本结构之后，我们来研究一下iOS的开发者证书，它是开发过程中必不可少的东西，相信大家都有接触。众所周知，iOS设备并不能像Android那样任意地安装app，app必须被Apple签名之后才能安装到设备上。而开发者在开发App的时候需要频繁地修改代码并安装到设备上进行测试，不可能每次都先上传给Apple进行签名，因此需要一种不需要苹果签名就可以运行的机制。

<!-- more -->

这个机制的实现方式是：

- 开发者自己持有一套密钥和证书，可以自行对app进行签名
- 由Apple对开发者的身份进行“背书”，让设备间能够接信任开发者自行签名的app，这个“背书”的方式就是后面会提到的`Provisioning Profile`

那么先研究一下开发者证书是如何产生的：在Xcode 8及之后的版本，Xcode会自动帮我们管理证书，我们可能根本不会有机会去研究它，但是在早期的版本中，需要我们自己动手操作，获取开发者证书主要有两个步骤

#### 生成CSR文件(Certificate Signing Request)

在Keychain菜单栏选择"从证书颁发机构请求证书..."

![csr1](/assets/2019/csr1.png)

![csr2](/assets/2019/csr2.png)

这个操作会产生一个名为`CertificateSigningRequest.certSigningRequest` 的签名请求文件，在生成这个文件之前其实Keychain已经自动生成了一对公、私钥

![csr3](/assets/2019/csr3.png)

![csr4](/assets/2019/csr4.png)

可以在Keychain中选中这个条目，右键选择导出，将密钥文件导出为p12文件，使用openssl查看其内容

```bash
$ openssl pkcs12 -in JustForTesting.p12 -out private_key.pem  # 导出p12文件中的密钥
Enter Import Password:    # 输入p12文件的密码
MAC verified OK
Enter PEM pass phrase:    # 设定导出的密钥文件的密码
Verifying - Enter PEM pass phrase:    # 确认密码
$ openssl rsa -in private_key.pem -noout -text  # 查看密钥文件的内容
Enter pass phrase for private_key.pem:   # 输入密钥文件的密码
Private-Key: (2048 bit)
modulus:
    00:c2:98:f5:02:eb:dc:a6:fd:4b:12:4c:70:17:a6:
    xx:xx:xx:xx:xx:xx:xx:...
publicExponent: 65537 (0x10001)
privateExponent:
    00:a1:67:68:e1:51:6c:a4:fd:36:45:29:2d:58:10:
    xx:xx:xx:xx:xx:xx:xx:...
prime1:
    00:f3:91:5d:5b:dc:c1:de:d2:ab:7a:5f:b2:27:41:
    xx:xx:xx:xx:xx:xx:xx:...
prime2:
    00:cc:87:b5:c9:7e:81:39:94:13:c1:ff:3f:d7:7b:
    xx:xx:xx:xx:xx:xx:xx:...
exponent1:
    00:a5:a0:22:c0:f5:d3:eb:86:8c:4e:b1:c6:3e:85:
    xx:xx:xx:xx:xx:xx:xx:...
exponent2:
    00:8b:e1:00:85:a6:7c:10:79:e2:2d:5a:39:3a:51:
    xx:xx:xx:xx:xx:xx:xx:...
coefficient:
    7e:30:60:84:fc:47:6b:90:fe:e7:32:1a:2f:b0:c4:
    xx:xx:xx:xx:xx:xx:xx:...
```

这里出现了几个熟悉的面孔：

- prime1/prime2 就是生成密钥所使用的两个超大的素数`p, q`
- modulus 是这两个超大素数的乘积 `n = p * q`
- publicExponent 是公钥因子，也就是前文中的`e`, 这里固定为 0x10001 (65535)
- privateExponent 是私钥因子，即前文中的`d`

CSR文件的内容其实就是个人信息、公钥(Modulus + PublicExponent)，以及自签名(使用自己的私钥进行签名)， 可通过openssl命令查看其内容：

```bash
$ openssl req -in ~/Desktop/CertificateSigningRequest.certSigningRequest -text -noout
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: emailAddress=me@xelz.info, CN=JustForTesting, C=CN
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c2:98:f5:02:eb:dc:a6:fd:4b:12:4c:70:17:a6:
                    xx:xx:xx:xx:xx:xx:xx:...
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
    Signature Algorithm: sha256WithRSAEncryption
         b7:11:aa:48:2f:b3:10:e9:71:c7:93:c3:ec:44:8d:0f:a0:5a:
         xx:xx:xx:xx:xx:xx:xx:...
```

#### 提交给Apple进行签名

在苹果开发者网站，将CSR提交给Apple进行签名，Apple会返回一个签好名的`证书文件`，后缀名为`cer`。

先查看一下他的`sha1`值，后面会用到

```bash
$ shasum ios_development.cer
11447116f2c5521b057b9b67290f0fdadeadfa0a  ios_development.cer
```

双击即可将其导入到Keychain中，Keychain会自动把它之前创建CSR时自动生成的密钥归为一组。无论是在证书列表中查看还是在密钥列表中查看，都能看到与之匹配的`另一半`。

![](/assets/2019/csr5.png)

查看证书的内容

![](/assets/2019/cert1.png)

可以从证书中得到几个关键信息：

1. 证书的所有者，这部分信息并非由我们自行指定，而是签发者Apple根据我们的账号信息自动生成
2. 证书的签发者，即前文所述的`CA`
3. 证书的公钥信息，与之前生成的密钥文件及CSR完全一致

现在应该可以理解证书和密钥的关系了，密钥中保存了私钥和公钥，私钥用于签名，而证书里面有且只有公钥，并且是被第三方`CA` "认证" 过，用于解密和校验。

一般我们说使用`证书`签名，实际上是使用与证书所匹配的私钥进行`签名`，`证书`只是作为签名数据的一部分被嵌入到签名结构中。如果Keychain中只有证书，没有对应的密钥文件，是无法进行签名的，会得到`Missing private key`之类的报错提示。

图中可以看到这个证书的签发者是`Apple Worldwide Developer Relations Certification Authority`，在Keychain中搜索这个名字， 可以看到它的证书详情。我们会发现，它的类型是`中级证书颁发机构(中级CA)`，它也包含签名，并且是由另外一个叫做`Apple Root CA`的`根证书颁发机构(根CA)`进行签发的，这样就形成了一条证书链。而继续查看`Apple Root CA`的证书，会发现它是自签名的，因为它会被内置在设备中，设备无条件信任它，也就不需要其他的机构为其背书了。

![](/assets/2019/cert2.png)

这样的证书链机制可以简化根证书颁发机构的工作，同时提升证书管理的安全性。将颁发底层证书的工作分散给多个中级证书颁发机构进行处理，根证书颁发机构只需要对下一级机构的证书进行管理和签发，降低根证书颁发机构私钥的使用频率，也就降低了私钥泄露的风险。中级证书颁发机构各司其职，即使出现私钥泄露这样的重大安全事故，也不至于波及整个证书网络。

#### 开发证书与发布证书

开发者证书按用途可分为Development证书和Distribution证书：

- Development证书是用于开发及测试阶段使用的证书，它用于在设备安装上开发阶段的App后对App的完整性进行校验，一般证书名称为 iPhone Developer: xxxxxxx。如果是多人协作的开发者账号，任意成员都可以申请自己的Development证书。
- Distribution证书是用于提交AppStore的证书，一般命名为 iPhone Distribution: xxxxxxxxx，用于让AppStore校验提交上来的App的完整性，只有管理员以上身份的开发者账号才可以申请，因此可以控制提交权限的范围。同时，Distribution证书不能用于开发及调试。

#### 企业级开发者证书

除了普通开发者证书(个人开发者账号和公司开发者账号使用的证书)外，还有一种特殊的`企业级开发者证书`，这种证书签名的App可以被直接安装在任意的iOS设备上，只要用户主动信任该证书即可。它的作用是方便企业给内部员工分发生产力工具，比如往往存在这样一些场景：企业内部无法访问互联网，自然也就无法通过AppStore安装应用，或是使用私有API，完成一些AppStore不允许的功能。前面所说的不需要苹果签名即可安装运行的机制同样适用于企业级开发者证书，并且是企业级开发者证书的基础。

从证书的申请方式和内容来看，企业级开发者证书和普通开发者证书并无不同，只是开发者账号的申请方式和费用有区别。此外，Apple对这两种证书所能提供的Provisioning Profile有细微的差异，下一节马上就会分析。

## 0x04 Entitlements & Provisioning Profile

除了开发者证书，在进行iOS代码签名的时候还需要有这两个文件，他们是被签名内容的一部分

#### Entitlements

沙盒(Sandbox)技术是iOS安全体系中非常重要的一项技术，他的目的是通过各种技术手段限制App的行为，比如可读写的路径，允许访问的硬件，允许使用的服务等等，即使应用出现任意代码执行的漏洞，也无法影响到沙盒外的系统。（图来自[Apple开发者网站](https://developer.apple.com/library/archive/documentation/Security/Conceptual/AppSandboxDesignGuide/AboutAppSandbox/AboutAppSandbox.html)）

![](/assets/2019/sandboxing.png)

通常所说的Entitlements(授权文件)，也就是指iOS沙盒的配置文件，这个文件中声明了app所需的权限，如果app中使用到了某项沙盒限制的功能，但没有声明对应的权限，可能运行到相关的代码时会直接Crash。

全新的iOS工程中是没有这个文件的，如果在`Capabilities`中开启了一些需要权限的功能之后，Xcode会自动(Xcode 8及之后的版本)生成Entilements文件，并将对应的权限声明添加到Entitlements文件中。

![](/assets/2019/ent1.png)

![](/assets/2019/ent2.png)

这个文件其实是xml格式的`plist`文件，内容如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>inter-app-audio</key>
	<true/>
</dict>
</plist>
```

实际上，这个文件的内容并非是全部的授权内容，因为缺省状态下，App默认会包含以下与Team ID及App ID相关的权限声明：

```xml
<dict>
    <key>keychain-access-groups</key>
    <array>
        <string>xxxxxxxxxx.*</string>
    </array>
    <key>get-task-allow</key>
    <true/>
    <key>application-identifier</key>
    <string>xxxxxxxxxx.test.CodeSign</string>
    <key>com.apple.developer.team-identifier</key>
    <string>xxxxxxxxxx</string>
</dict>
```

其中`get-task-allow`代表是否允许被调试，它在开发阶段是必需的一项权限，而在进行Archive打包用于上架时会被去除。

进行代码签名时，会将这个Entitlements文件(如有)与上述缺省内容进行合并，得到最终的授权文件，并嵌入二进制代码中，作为被签名内容的一部分，由代码签名保证其不可篡改性。

#### Provisioning Profile

Xcode对Provisioning Profile的解释是

>  A provisioning profile is a collection of digital entities that uniquely ties developers and devices to an authorized iPhone Development Team and enables a device to be used for testing.

Provisioning Profile在这里就起到了一个对设备和开发者授权的作用，他将开发者账号、证书、entitlements文件以及设备进行了绑定。

同样地，在开发过程中，Xcode 8及后续版本默认情况下会自动帮我们管理Provisioining Profile，自动下载的Provisioning Profile都被存放在`~/Library/MobileDevice/Provisioning\ Profiles/`路径下，以`UUID`格式命名。直接拖拽下图中的齿轮图标到Finder中也可以将其复制出来。

![](/assets/2019/provision1.png)

由于这个文件是被苹果签过名的，所以我们没有办法伪造或者修改这个文件，它使用的是标准的CMS(Cryptographic Message Syntax)格式，可以通过security命令查看它的签名信息，并将文件的内容提取出来：

```bash
$ security cms -D -i xxxxxxxxxxx.mobileprovision -h 1 -n  # 查看签名信息
SMIME: 	level=1.2; type=signedData; nsigners=1;
		signer0.id="Apple iPhone OS Provisioning Profile Signing"; signer0.status=GoodSignature;
	level=1.1; type=data;
$ security cms -D -i ea8585cd-c2da-4b08-81c2-e32b28c34871.mobileprovision -o provision.plist  # 将内容导出
```

Provisioning Profile统一都是由`Apple iPhone OS Provisioning Profile Signing`进行签名的，机构名称言简意赅。导出的provision.plist内容如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>AppIDName</key>
	<string>TestCodeSign</string>
    ...
	<key>DeveloperCertificates</key>
	<array>
		<data>xxxxx</data>
        <data>xxxxx</data>
        <data>xxxxx</data>
	</array>
	<key>Entitlements</key>
	<dict>
		<key>keychain-access-groups</key>
		<array>
			<string>xxxxx.*</string>
		</array>
		<key>inter-app-audio</key>
		<true/>
		<key>get-task-allow</key>
		<true/>
		<key>application-identifier</key>
		<string>xxxxx.test.CodeSign</string>
		<key>com.apple.developer.team-identifier</key>
		<string>xxxxx</string>
		<key>com.apple.developer.siri</key>
		<true/>
	</dict>
	<key>ExpirationDate</key>
	<date>2020-01-22T05:14:57Z</date>
	<key>Name</key>
	<string>iOS Team Provisioning Profile: test.CodeSign</string>
	<key>ProvisionedDevices</key>
	<array>
		<string>xxxxx</string>
		<string>xxxxx</string>
		<string>xxxxx</string>
	</array>
	...
</dict>
</plist>
```

很明显可以看出这是一个xml格式的plist文件，里面的内容不难理解，最关键的是这几项

- **DeveloperCertificates**：允许使用的开发者证书，这是一个列表，一般包含生成这个Provisioning Profile文件时，当前开发者账号下所有有效的Development证书，以base64格式保存，使用base64解码之后就可以得到DER格式的开发者证书。通过计算每个证书的sha1值，可以看出，前文中新申请的证书，就在这个列表中

```bash
$ for i in `seq 3`; do /usr/libexec/PlistBuddy -x -c 'Print:DeveloperCertificates:'$i provision.plist | sed -n '/<data>/,/<\/data>/p' | sed -e '1d;$d' | base64 -D | shasum ; done
  11447116f2c5521b057b9b67290f0fdadeadfa0a  -    # <--- 新申请的证书
  df446e4fad5aa292c7323da4cf7b8869fa5c89e7  -
  9d31f7e8c27760ffa061598ba90ea614948224bf  -
```

- **Entitlements**：允许使用的权限列表，实际在App中使用的权限必须是这个列表的子集，否则安装时会无法通过校验而失败。如果曾经开启过某个功能，Xcode自动更新了Provisioning Profile，后来又关闭它，Xcode并不会将其从Provisioning Profile中删去，如示例中的`com.apple.developer.siri`。
- **ProvisionedDevices**：允许安装的设备列表，如果目标设备的UUID不在这个列表中，会安装失败。对于这一项，普通开发者证书和企业级开发者证书的待遇是不同的。普通开发者证书使用Provisioning Profile的方式安装App到设备，只是出于测试和调试的需要，因此Apple只允许最多注册100台用于测试的设备，否则开发者就可以以测试的名义任意任意分发自己的App了。而对于企业级开发者来说，本身就有任意安装的需求，因此在分发时，这一项会被`ProvisionsAllDevices`取代，代表授权任意设备。

这些信息中有任何变动的时候，比如开发者证书有新增或者失效，在Capabilities中启用了当前App从未使用过的新功能，或是将新的iPhone连接到Xcode用于测试，Xcode都会自动重新申请Provisioning Profile。

Provisioning Profile会被内置在App中，置于App根目录下的`embedded.mobileprovision`。安装App时如果签名校验通过，这个文件会自动被拷贝到iOS设备的`/Library/MobileDevice/Provisioning\ Profiles/`路径下。由于该文件已被Apple官方签名，系统可以无条件信任它，并用它来校验App的签名、权限，以及本机的UUID等是否满足来自官方的授权。通过这种方式，间接信任了使用开发者证书签名的App，让iOS设备可以运行非苹果官方签名的App。

假如你有一台越狱的设备，查看任意一个从AppStore上下载下来的App，里面都不会有embedded.mobileprovision这个文件，因为经过Apple重新签名以后，设备就不再需要它了。
