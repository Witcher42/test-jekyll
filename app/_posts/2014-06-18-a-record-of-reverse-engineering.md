---
layout: post
title: "记一次逆向工程"
description: ""
category: 
tags: [reverse engineering]
---
{% include JB/setup %}

> 其实我购买了这个产品，不过它只能在 windows 上使用，于是我将其移植到 mac 上。
> 
> 上学时了解过一些逆向工程方面的东西，这些年没有接触完全忘记了。

### 准备

故事的源头已不可考，总之现在我手里有一份程序，我要对其进行深入。

初步观察，程序的主体和入口是一个 exe，我们姑且称之为 crackme.exe，同时并列有一个 jedata.dll，五个 *.zxp 文件，所以文件结构是这样的：

    .
    ├── jedata.dll
    ├── a1.zxp.zxp
    ├── a2.zxp.zxp
    ├── a3.zxp.zxp
    ├── a4.zxp.zxp
    ├── a5.zxp.zxp
    └── crackme.exe
    
zxp 是 Photoshop 的扩展，我的目标就是安装这些 zxp 文件。运行下 *.zxp 都是坏掉的文件，不能安装，应该是被加密了。

运行 crackme.exe 弹出让注册的窗口，窗口中展示了机器码和可以输入注册码的框。看作者给出的注册过程，需要将机器码发送给作者，然后由作者给自己注册码，才能完成注册。

比较神奇的是作者给出的过程没有遮掉注册码和机器码，看到这些数字的时候，第一直觉是发现开头一串字符似乎有些规律，类似于 ASCII 码的映射关系，但是后边就看不出了。（其实这个时候已经可以有一个思路了，就是将注册码和机器码都修改为作者暴漏出来的字符串，但是第一次逆向不免摸不着头绪）

### 基础

有了上面的一些基础工作，怎么去逆向呢？作为一个零基础的人，自然是依赖搜索引擎，按照我原来的知识，是搜索「脱壳」，知道了软件「peid」可以查看是否加壳，但是没有看到什么有用的信息。然后搜索了和 jedata.dll 相关的信息，资料比较少。

搜索时无意间看到了「OD」字样，一查果然是一条门径，这个软件是「OllyDBG」，应该是使用相当普遍的一款软件，而这个领域的标准叫法应该是「逆向工程」。找了一下 OD 的教程，尝试搜索字符串的方法，一无所获。接着发现了针对不同的语言有不同的逆向方式，决定先判断这个程序是什么语言写的，反复查找反编译后的汇编代码，发现了「易语言」这个关键字。

搜索「易语言 反编译」、「易语言 破解」、「易语言 OD」、「易语言 逆向」找到了几种破解方式

* 按钮事件 FF 55 FC
* 找字符串
* Ecode段

期间，还找到了「[看雪学院](http://www.pediy.com/)」的[一篇帖子](http://bbs.pediy.com/showthread.php?t=76940)，是一个妹子发布的用易语言写的 crackme。在[回帖](http://bbs.pediy.com/showpost.php?p=658531&postcount=61)中有人总结了当时破解易语言的几种方式，也就是上面说的三种（事隔多年又看雪……）。另外找到了易语言的一个反编译工具「易语言转换工具」，和一个「E-Debug Events」。

### 脱壳

事实上，上面的调试方式和工具都不起作用，因为一是找不到调试方式所对应的特征，二是运行工具得不到希望得到的结果。

经过一番搜索发现有人说加壳的问题，再次使用 peid 发现了这样的字符串

    UPX 2.93 - 3.00 [LZMA] -> Markus Oberhumer, Laszlo Molnar & John Reiser *
    
搜索得知是 UPX 加的壳，使用「upx.exe」进行脱壳，得到了新的可执行文件，我们姑且称之为「NewCrackMe.exe」。

### 爆破

对 NewCrackMe.exe 执行上面的一系列操作，只有「E-Debug Events」发挥了作用，每次点击事件都会被其检测出地址，找到了特定的地址，是 MessageBoxA 的地址，针对特征码「FF 15 20 14」 进行查找，找到了几个 MessageBoxA，分别断点之后，的确是对应的弹窗操作。我判断 `ret` 之后是返回到上一级中断，于是在 ret 到上一层后再前一个 call 打上断点。最终定位到了获取输入框「机器码」和获取输入框「注册码」的中断地址，由此可以中断到判断是否正确注册的地址。接下来的思路就是将其改成相反的指令（或者 nop），再次执行就应该可以成功了。

不过此时我发现了一个新的字符串，是一个网址，指向了 hi.baidu.com 上第一篇文章，上面居然记录了几个完整的注册码，粗略一看就发现后面的字符都是一模一样的，于是我推敲了一下前面的字符和机器码的关系，发现了一个规律，是非常简单的古典加密方式，于是写了一个小程序生成了自己的注册码。

### 安装

接着点击安装，正常运行，安装时肯定是要将 zxp 都解密到临时目录中的，在根目录中搜索 zxp，找到了解密后的文件。

在 Mac 中正确安装，至此已经完成了破解。