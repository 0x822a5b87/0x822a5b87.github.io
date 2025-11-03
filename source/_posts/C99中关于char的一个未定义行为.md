---
title: C99中关于char的一个未定义行为
date: 2025-11-03 16:25:47
tags:
---

# 前言

最近在学习 RISCV 的过程中，碰见了一个比较奇特的BUG，在下面的代码中，我们出现了一个神秘的未定义行为（Undefined Behavior）：

```c
#include <stdio.h>

void main()
{
    char a;
    unsigned int b;
    unsigned long c;

    a = 0x88;
    b = ~a;
    c = ~a;

    printf("a=0x%x, ~a=0x%x, b=0x%x, c=0x%lx\n", a, ~a, b, c);
}
```

本来是一段很简单的用于测试C语言下的隐式转换规则的代码，结果很神奇的是在两个不同的平台下出现了截然不同的输出！

## x86/x86_64

在 `x86/x86_64` 平台下，我使用的编译器版本如下：

```
gcc (Debian 10.3.0-9) 10.3.0
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

## RISC-V

在 `riscv` 平台下，我使用的编译器版本如下：

```
gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

## 实际输出

在 `x86/x86_64` 平台下，我们的输出是这样的：

```
a=0xffffff88, ~a=0x77, b=0x77, c=0x77
```

在 `riscv` 平台下，我们的输出是这样的：

```
a=0x88, ~a=0xffffff77, b=0xffffff77, c=0xffffffffffffff77
```

其实观察输出就可以知道一个很明显的结论：在两个平台下，对char类型是有符号还是无符号的实现存在差别！

在x86平台下，char 是一个有符号的整数：

1. a  被**符号扩展**到 0xffffff88；
2. ~a 先**符号扩展**到int，再取反得到 0x77；
3. b  先**符号扩展**到int，再取反得到 0x77；
4. c  先**符号扩展**到int，再取反得到 0x77；

在 riscv 下，char 是一个无符号的整数：

1. a  被**零扩展**到 0x88；
2. ~a 先**零扩展**到int，再取反得到 0xffffff77；
3. b  先**零扩展**到int，再取反得到 0xffffff77；
4. c  先**零扩展**到int，再取反得到 0xffffffffffffff77；

# 详细解释

要想知道为什么结果是这样，我们必须先了解清楚几个重要的问题：

1. `char` 的定义是什么？
2. 类型转换的符号规则是什么？

## 什么是char

翻阅C99 SPECIFICATION，我们可以找到一下几个关键结论：

>An object declared as type char is large enough to store any member of the **basic execution character set**. If a member of the basic execution character set is stored in a char object, its value is guaranteed to be `positive`. **If any other character is stored in a char object, the resulting value is implementation-defined but shall be within the range of values that can be represented in that type**.

在 `6.2.5 Types - 3` 这一小节中，我们可以得出关于char的几个重要的结论：

1. 大小必须足够存储基础执行字符集（basic execution character set）；
2. 基础执行字符集存储在char中必须为非负数；
3. 如果有其他的字符被存储在char中，结果值是**基于实现**的，但是需要可以被char类型的范围所覆盖。

这是一个比较令我震惊的结论，因为我一直以为char的类型是无符号类型。

## 类型转换中的符号

在 `6.3.1.3 Signed and unsigned integers` 这一小节中，我们可以找到如下内容：

>1. When a value with integer type is converted to another integer type other than `_Bool`, if the value can be represented by the new type, it is unchanged.
>2. Otherwise, if the new type is unsigned, the value is converted by repeatedly adding or subtracting one more than the maximum value that can be represented in the new type until the value is in the range of the new type.
>3.  Otherwise, the new type is signed and the value cannot be represented in it; either the result is implementation-defined or an implementation-defined signal is raised.

简答来说，就是：

1. 当一个integer类型被转换为除了 `_Bool` 类型之外的其他 integer类型，如果值可以被新的类型表示，那么他不变。例如 integer 被提升为long。
2. 当一个值无法被目标无符号类型直接表示时，通过反复加减目标类型的 “模”（最大值 + 1），将其映射到目标类型的取值范围内。
	- 例如，将一个int类型的值 `300` 转换为 unsigned char（范围 0~255，模 256）时，我们通过 `300` - `256` 得到 `44`。
3. 如果新值是有符号的并且旧值无法被新值表示，新值是基于实现的或者一个基于实现的信号被发出。

## 总结

基于以上内容，我们可以得到几个结论：

1. char的默认符号是基于实现来确定的，在不同的平台和编译器下会有不同的结果；
2. 在x86中，char 是有符号的；在riscv中，char是无符号的；
3. 在对char进行符号扩展的时候：
	3.1 将char视为有符号，变量提升时会进行**符号扩展**；
	3.2 将char视为无符号，变量提升时会进行**零扩展**。

以上几点，共同决定了对于 `0x88` 这个数字的处理，对于将char看做有符号数的平台，他是一个负数。他在扩展的时候得到的也是一个负数；对于将char看做无符号数的平台，他是一个整数，他在扩展的时候按照零扩展，得到了一个同样的正数。