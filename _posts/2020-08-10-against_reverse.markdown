---
layout: post
title:  "对抗反汇编之花指令"
description: 书摘<恶意代码>
date:   2020-08-10 14:12:09 +0530
categories: 逆向
---
### 静态反汇编对抗

反汇编器有两种反汇编方式:**线性反汇编**和**面向代码流反汇编**.

#### 线性反汇编
线性反汇编最大的问题是其会把二进制文件中所有的数据都进行反汇编,但应用程序中有大量数据不是指令,这会造成错误的反汇编.
作为恶意软件制作者,可以使用`垃圾跳转指令`配合`混淆数据`进行编译,从而阻止此类反汇编器进行解析.

#### 面向代码留反汇编
这种方式不会盲目的反汇编.而是检查每一条指令,建立一个反汇编地址表.
![面向代码流反汇编,注意数据部分被单独提了出来](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/%E9%9D%A2%E5%90%91%E4%BB%A3%E7%A0%81%E6%B5%81.png)
![但线性反汇编就不行](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/%E7%BA%BF%E6%80%A7%E6%B5%81.png)

##### 面向代码流的反汇编对抗

这一部分的核心是迷惑反编译器将不该编译的部分进行编译.因为指令是不等长的,只要精巧的填充这些不该被编译的部分,可以使编译器把余下的部分错误编译.

* 控制流欺骗概述  
条件跳转语句在静态反编译时面临着不确定性--例如没有运行时反编译器不知道指令会跳转至何方,因此会默认选择 true 或 false 处的
汇编流进行反编译.攻击者可固定条件跳转的位置,随后对另一部分的控制流语句进行混淆.
没有ret的call指令也会有同样的效果:反汇编器会把call指令后方指令全部反汇编,但实际上这部分指令并没有被运行.

* 相同目标的跳转指令  
例如jz与jnz这样的指令连在一起并跳转至相同的位置时,相当于一个jmp指令.但静态反汇编器无法辨别,而导致其错误的将跳转指令后方的字节识别为指令.
![注意第5个字节处不是指令,却被错误的识别](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/15-2.png)

**tip**:在ida中你可以使用`c`键将数据转换为代码,用`D`键将代码转换为数据.

* 固定条件跳转  
看似为条件跳转,实际上跳转条件被锁死了.由此迷惑静态编译器,将垃圾数据部分进行编译.  
![xor指令使得jz指令总是跳转,而显然反编译器没有意识到这点](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/15-3.png)

* 被拆解的语句  
如果程序中没有用于混淆的垃圾指令,而是将一条指令拆开复用,这也会迷惑反编译器(个人觉得手工分析这种也挺恶心的).
x86系统中的指令是是由1到多个字节组成的,如果控制流跳转到一条指令中间的位置,则会形成新的语句.简单的示例由下图所示.  
![jmp语句跳转到别的语句内部](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/15-4.png)  
 当然这种方法可以复合着使用,使得逆向难度大大增加.
![两个代码内部跳转](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/15-5.png)  
这段代码原本被反编译为:  
![错误的反编译](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/%E6%97%A0%E6%95%88%E6%8C%87%E4%BB%A4-%E9%94%99%E8%AF%AF%E5%8F%8D%E7%BC%96%E8%AF%91.png)  
但可以将其中无效的指令给标记为指令,只留下`XOR eax,eax`(这条改变了eax中的内容).这样便不会影响后方语句的反编译.
![正确的反编译](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/%E6%97%A0%E6%95%88%E6%8C%87%E4%BB%A4-%E6%AD%A3%E7%A1%AE%E5%8F%8D%E7%BC%96%E8%AF%91.png)  
最好将这些无用指令替换为`nop`.

##### 面向控制流的反汇编对抗  
这一部分内容主要通过对函数部分进行混淆,使得反编译器无法正常的识别函数.
* 滥用函数指针  
在汇编语言中刻意使用函数指针或构造不标准的函数指针格式,会导致没有动态分析时很难进行逆向.  
![函数指针滥用](https://raw.githubusercontent.com/niedabao1/niedabao1.github.io/master/pic/%E5%87%BD%E6%95%B0%E6%8C%87%E9%92%88%E6%BB%A5%E7%94%A8.png)  
由于调用不标准,图中 2 和 3 处没有显示交叉引用的符号,此时传递的参数信息也会丢失.  
tip:使用idc脚本可以进行交叉引用修复:  
```python
AddCodeXref(0x004011DF, 0x004011C0, fl_CF)#第三个参数是指明指令类型,call为fl_CF,跳转指令为fl_JF
AddCodeXref(0x004011EA, 0x004011C0, fl_CF)
```
