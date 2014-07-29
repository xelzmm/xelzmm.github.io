---
layout: post
title: "Write Up of a Very Interesting Wargame"
date: 2014-06-09 18:00:00
categories: ctf wargame
comments: true
---

Recently I'm playing a wargame named [shhhh... edited].

I've hidden the game name so that challengers could not find here by some searching work.

If you guys are about to cheat by this, get lost now.

You can find the game at [url]`c-a-n-y-o-u-h-a-c-k.i-t`(replace the dash with nothing)

Try to figure out by yourself, if you are really really really stucked, have a sight for some hints.

## Logic

#### Logic 1

password is just `password`

#### Logic 2

It's a kind of pun. If you cannot guess the riddle, just answer `no`.

Acturually the answer is Nitric Oxide, as known as `NO`

#### Logic 3

Inspect the source code, you will find the password in comment.

#### Logic 4

**Fibonacci Prime**

prime(n) = 2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101, 103, 107, 109, 113, 127, 131, 137, 139, 149, 151, 157, 163, 167, 173, 179, 181, 191, 193, 197, 199, 211, 223, 227, 229, 233, 239, 241, 251, 257, 263, 269, 271 ...

fibonacci(n) = 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144 ...


* prime(fibonacci(1)) = prime(1) = 2
* prime(fibonacci(2)) = prime(2) = 3
* prime(fibonacci(3)) = prime(3) = 5
* prime(fibonacci(4)) = prime(5) = 11
* ...
* prime(fibonacci(8)) = prime(55) = 139
* prime(fibonacci(9)) = prime(55) = 257

## Script

#### Script 1

``` javascript 
if($('#password').val() == "javascript")
```

password is `javascript`

#### Script 2

Run this code in javasript console, then check the value of variable `password`.

``` javascript 
    var a = "de9f8caa7ea6fe56830925a124d605d4";
    
    var password = "";
    
    for(var i = 0; i < 20; i++)
        password += a.substring((i%3),(i%5)+(i%3));
```

#### Script 3

Run this code in javasript console, then check the value of variable `password`.

``` javascript 
    keys = new Array("0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z","a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z");
    password = "";
    k = "17 4 59 0 53 28".split(" ");
    for (i in k) {
        password += keys[parseInt(k[i])];
    }
```    


## Cryptography

#### Cryptography 1

The **Salad Cipher**, aka **ROT13**

Decryption Key

    A|B|C|D|E|F|G|H|I|J|K|L|M
    -------------------------
    N|O|P|Q|R|S|T|U|V|W|X|Y|Z
    
letter above equals below, and vice versa

#### Cryptography 2

Try to combine some words using the numbers with T9 IME on a mobile phone.
    
<table style="text-align: center;">
<tr><td><p>1</p>'</td><td><p>2</p>ABC</td><td><p>3</p>DEF</td></tr>
<tr><td><p>4</p>GHI</td><td><p>5</p>JKL</td><td><p>6</p>MNO</td></tr>
<tr><td><p>7</p>PQRS</td><td><p>8</p>TUV</td><td><p>9</p>WXYZ</td></tr>
</table>

#### Cryptography 3

**Base64** decode it.

#### Cryptography 4

**Caesar's Square**

    TSDLN ILHSY OGSRE WOOFR OPOUK OAAAR RIRID
    
Count the number of letters, here we have 35
We can put 35 into 5 rows of 7

    TSDLNIL
    HSYOGSR
    EWOOFRO
    POUKOAA
    ARRIRID
    
Read it, downwards from the top left, then the next column.

#### Cryptography 6

**Morse Alphabet**

#### Cryptography 7

**ASCII**

``` python 
    #python
    text = ''.join([chr(int(i)) for i in '84 104 101 32 115 101 99 114 101 116 32 119 111 114 100 32 121 111 117 39 114 101 32 115 101 97 114 99 104 105 110 103 32 102 111 114 32 105 115 32 115 101 99 114 101 116'.split(' ')])
```    

#### Cryptography 8

**Atbash** (similar with the Salad Cipher)

    A|B|C|D|E|F|G|H|I|J|K|L|M
    -------------------------
    Z|Y|X|W|V|U|T|S|R|Q|P|O|N

letter above equals below, and vice versa

in another way

    Plain:  ABCDEFGHIJKLMNOPQRSTUVWXYZ
    Cipher: ZYXWVUTSRQPONMLKJIHGFEDCBA
    
#### Cryptography 9

**Polybius Square**

<table>
<thead>
    <tr><td>\</td><th>1</th><th>2</th><th>3</th><th>4</th><th>5</th></tr>
</thead>
<tbody>
    <tr><th>1</th><td>A</td><td>B</td><td>C</td><td>D</td><td>E</td></tr>
    <tr><th>2</th><td>F</td><td>G</td><td>H</td><td>I</td><td>K</td></tr>
    <tr><th>3</th><td>L</td><td>M</td><td>N</td><td>O</td><td>P</td></tr>
    <tr><th>4</th><td>Q</td><td>R</td><td>S</td><td>T</td><td>U</td></tr>
    <tr><th>5</th><td>V</td><td>W</td><td>X</td><td>Y</td><td>Z</td></tr>
</tbody>
</table>

Each letter is then represented by its coordinates in the grid. For example, `BAT` becomes `12 11 44`. Because 26 characters do not quite fit in a square, it is rounded down to the next lowest square number by combining two letters (usually I and J).

#### Cryptography 11

The index of a letter in the `alphabt`. 0 indicates a blank.

#### Cryptography 13

A programming language named **BrainFuck**.

#### Cryptography 16

Read it in a human readable way. Starting with the top left `T`, then `H` under it, and then `E` on the right side. Hints in the title `clues`.

#### Cryptography 17

**MD5** ，brute force it with the hint `a-z*6`, or try cmd5.org.

#### Cryptography 22

Static crypto table with a reverse. the crypto table can be easily dumped.

    e7 a4 90 71 36 49 aa e6 5b 3a ef 64 a0 be eb 09 f2 8c 57 ec 8f 74 1f 01 51 98 
    Z  Y  X  W  V  U  T  S  R  Q  P  O  N  M  L  K  J  I  H  G  F  E  D  C  B  A
    
    91 72 61 3f 69 fe 4b fa 85 fd 14 68 73 26 0f ac cc a1 4d db ab 43 46 11 08 b7
    z  y  x  w  v  u  t  s  r  q  p  o  n  m  l  k  j  i  h  g  f  e  d  c  b  a
    
    d8 b0 31 07 cf 8e 45 24 0b 5a
    0  9  8  7  6  5  4  3  2  1
    
    92 35 00 c6 3d 55 96 54 7d f6 e9
    )  (  *  &  ^  %  $  #  @  !   
    
    cb d9 21 3e af 38 8b 4e 9e ea 0a 4c 04 58 6d b6 67 29 13 c5
    ?  >  <  "  :  |  }  {  +  _  /  .  ,  '  ;  \  ]  [  =  -
    
#### Cryptography 25

**Braille Alphabet**

##WEB Based

#### Web 1

    Page=Admin

#### Web 2

``` javascript 
    // javascript
    document.cookie='isAdmin=1';
```

#### Web 3

    /robots.txt

#### Web 4

``` bash 
    curl -H 'Referer: www.google.com' 'http://theurl/Content/Challenges/Web/Web4.php'
```

#### Web 5

Do not waste time on the form because nothing happend when you click the button.

    SESSION=abf3e2d32ec32' or '1'='1' --

#### Web 6

look around http://theurl/Content/Challenges/Web/Files6/

#### Web 7

    curl -d 'Type=admin' 'http://theurl/Content/Challenges/Web/Web7.php'
    
#### Web 8

    Page[]=Home 
    
will trigger a php `fatal error`, which will display the error stack including the full path of the file in the page.

#### Web 9

    File=Files9/passconfigs.php%00

#### Web 10

``` bash 
    curl 'http://theurl/Content/Challenges/Web/Web10.php'
```   

## Microhard 

#### CCTV

Try to find something in the terminal

``` bash 
    help
    echo learninglog.txt
    apt-get install ***
    ifconfig
    *** 192.***
```

Then an open port of a alive host which may be the remote camera.
Open it in Firefox and then successfully we can get the CCTV admin page.

Try to login with someone's name as the password.


