---
layout: post
title: "Codegate 2017 2D Life writeup"
date: 2017-02-20 20:00:00
categroies: ctf
comments: true
---

###Description

#### 2D Life

470 points

```
http://110.10.212.135:24135
http://110.10.212.135:24136
http://110.10.212.147:24135
http://110.10.212.147:24136
```

I didn't have enough time to solve this challenge since I'm busy at work. It's a pity that my team didn't, neither. But I have to say it's a very challenging one. Combination of crypto and SQL injection. 

###First Sight

It seemed to be a web challenge because the entrance was a website. So let's start with HTTP requests and responses. In the source code of the page, a path to secret login page was commented.

```html
<div id="navbar" class="navbar-collapse collapse">
  <ul class="nav navbar-nav">
  <li><a href="/">Home</a></li>
  <li><a href="?p=pic">Pictures</a></li>
  <li><a href="?p=music">Music</a></li>
  <li><a href="?p=contact">Contact</a></li>
  <!--<li><a href="?p=secret_login">Login</a><li>-->
  </ul>
</div>
```

The login page set a cookie like this(using [httpie](https://httpie.org/))

```
$ http http://110.10.212.135:24135/\?p\=secret_login
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 579
Content-Type: text/html; charset=UTF-8
Date: Fri, 10 Feb 2017 05:57:50 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: identify=t93ZpEcFoz4%3D%7C6uDGkD5VtEk0H9kAOzOrQECDzRdVuuDYn4h8ISoWSUuetH5Cb%2BBgSfxSd9WfX9RxHGC7cnAZdnmxqneZrLkQ%2Bw%3D%3D
Vary: Accept-Encoding
```

It's easy to say that the cookie is two parts of base64 encoded string concatenated by a `|`.

```
t93ZpEcFoz4=|6uDGkD5VtEk0H9kAOzOrQECDzRdVuuDYn4h8ISoWSUuetH5Cb+BgSfxSd9WfX9RxHGC7cnAZdnmxqneZrLkQ+w==
```

Different cookies was returned when repeating the same request. Modify the tail of the cookie will got a message `Error has occur from decrypt..`, but the head won't.

<!-- more -->

###Cryptography

Look at the two parts of cookie:

```
part 1: t93ZpEcFoz4=
decode: b7 dd d9 a4 47 05 a3 3e
length: 8

part 2: 6uDGkD5VtEk0H9kAOzOrQECDzRdVuuDYn4h8ISoWSUuetH5Cb+BgSfxSd9WfX9RxHGC7cnAZdnmxqneZrLkQ+w==
decode: ea e0 c6 90 3e 55 b4 49 34 1f d9 00 3b 33 ab 40 40 83 cd 17 55 ba e0 d8 9f 88 7c 21 2a 16 49 4b 9e b4 7e 42 6f e0 60 49 fc 52 77 d5 9f 5f d4 71 1c 60 bb 72 70 19 76 79 b1 aa 77 99 ac b9 10 fb
length: 64
```

Now I believe it's a `Padding Oracle` Problem. I've read about it in *Web Security by White Hats* (刺总的《白帽子讲Web安全》). `Part 1` is the 8 bytes `iv` of encryption, and `Part 2`, obviously is 8 blocks of encrypted data, with 8 bytes in each block.

####CBC Mode

Every Block cipher can only deal with a message with fixed length (usually the same length as the key), so plain message is divided into several blocks and each block will be encrypted separately. To avoid data pattern sniffing, a vector is added befor encryption in CBC mode.

```
                 XOR
plain block ---> |+| ---> intermediate value ---> encrypted block
                  ^                           ^
                Vector                    encryption
```

Vector of each plain data block is the encrypted data of previous block. The Initial Vector for the first data block is provided additionally.

####PKCS#5 Padding

Length of every block must be exactly the same with the key. In this case, the length is 8 bytes. If there is less than 8 bytes(or just equal to 8 bytes) in the last block, a padding is introduced.

```
xx xx xx xx xx xx xx    -> xx xx xx xx xx xx xx 01
xx xx xx xx xx xx       -> xx xx xx xx xx xx 02 02
xx xx xx xx xx          -> xx xx xx xx xx 03 03 03
xx xx xx xx             -> xx xx xx xx 04 04 04 04
xx xx xx                -> xx xx xx 05 05 05 05 05
xx xx                   -> xx xx 06 06 06 06 06 06
xx                      -> xx 07 07 07 07 07 07 07
xx xx xx xx xx xx xx xx -> xx xx xx xx xx xx xx xx 
                           08 08 08 08 08 08 08 08
```

While decrypting, cipher will check the value of the last byte in the decrypted message. Assume that value is 0x04, then check the value of the last 4 bytes. It will be fine if they all equal to 0x04 and the 4 bytes will be directly removed to recover the original length of plain message. Otherwise a decryption exception occured as I tried above. 

####Padding Oracle Attack

We know a bad padding format of the last block will cause exception, so if we craft a fake data which can make the padding match the right format, the data will be accepted by the server without throwing a decryption exception(This does not means it will be completely accepted by server without any other excpetions because the data is totally a mess). At this moment we know the last few bytes in the decrypted message,  is one of the padding format.

We've got last bytes of plain block and the vector(we craft it), so we can get the last bytes of intermediate value of the corresponding encrypted block by

```
intermediate value  =  plain block with padding  (xor)  craft vector
```

and then, the real plain block

```
plain block = intermediate value  (xor)  actual vector
```

To make it clear, we can brute force every byte in a block, from the last byte to the first one. 

```
|            iv: b7 dd d9 a4 47 05 a3 3e
|encrypted data: ea e0 c6 90 3e 55 b4 49
|                34 1f d9 00 3b 33 ab 40
|                40 83 cd 17 55 ba e0 d8
|                9f 88 7c 21 2a 16 49 4b
|                9e b4 7e 42 6f e0 60 49
|                fc 52 77 d5 9f 5f d4 71
|                1c 60 bb 72 70 19 76 79
|                b1 aa 77 99 ac b9 10 fb
```

Start with the first block `ea e0 c6 90 3e 55 b4 49`, enumerate the last byte of iv, from 0x00 to 0xFF.

```
|               iv : ff ff ff ff ff ff ff 00
|       iv encoded : /////////wA=
|encrypted message : ea e0 c6 90 3e 55 b4 49
|  message encoded : 6uDGkD5VtEk=
|           cookie : /////////wA=|6uDGkD5VtEk=
```

visit secure login page with the fake cookie:

```
$ http '110.10.212.135:24135/?p=secret_login' cookie:'identity=/////////wA=|6uDGkD5VtEk='
```

got the message `Error has occur from decrypt..`

continue trying with different iv(this can be done with a piece of script)

```
ff ff ff ff ff ff ff 01
ff ff ff ff ff ff ff 02
...
ff ff ff ff ff ff ff 1f
...
```

a different message showed up when trying `0x1f` as the last byte in iv.

```
Is that all? HACKER?
```

BINGO!  It means the padding is 0x01 now(not quite), more clearly, the last byte of the plain message is 0x01.

PS: If the second to last byte in the plain message just happen to be 0x02, then the last byte may be 0x02, too. Both 0x01 and 0x02 are valid at this situation. Just change the last 0xff in iv to any other value and try again, which will break the combination of  `0x02 0x02` padding (into `0x?? 0x02`). If nothing different with 0xff(no decrypt error occuring), 0x01 is the right answer.

the last byte of intermediate value can be calculated by

```
intermediate value = iv   (xor)   plain message
        1e         = 1f    (+)         01
```

and then calculate the last byte of original plain message by the original iv

```
plain message = iv   (xor)   intermediate value
      20      = 3e    (+)           1e
```

The last byte of the first plain block is `0x20`!

Next byte, we need to make the plain message have a value of 0x02 in the last byte, to test the `0x02 0x02` padding. So last byte of iv must be `0x02 (+) 0x1e = 0x1c`

Trying like this

```
ff ff ff ff ff ff 00 1c
ff ff ff ff ff ff 01 1c
ff ff ff ff ff ff 02 1c
...
ff ff ff ff ff ff ff 1c
```

`ff ff ff ff ff ff e4 1c` will make the sense.  `0xe4 (+) 0x02 (+) 0xa3 = 0x45`

Finally we can get the first block:

```
|                iv : b7 dd d9 a4 47 05 a3 3e
|intermediate value : fa 98 8a f7 06 42 e6 1e
|     plain message : 4d 45 53 53 41 47 45 20
|        plain text : M  E  S  S  A  G  E  
```

Continue with the next block:

```
34 1f d9 00 3b 33 ab 40
```

Notice that the original vector of this block is the previous enctyped block `ea e0 c6 90 3e 55 b4 49`, not the iv.

After all the entire message came out:

```
MESSAGE FROM SPY<!--TABLE:agents NUMBER OF COLUMNS:5-->;SPY;66
```

### SQL Injection

We didn't got the flag but a hint

```
Table: agents
columns: 5
```

It should be a SQL injection attack.

I dinn't solve this until the server was shut down. TAT

[Writeup by cnc](http://crypto.rop.sh/post/71CBLOYIN034)