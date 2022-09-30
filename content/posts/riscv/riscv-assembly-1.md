---
author: "Parker Ma"
title: "RISC-V 汇编总结 1"
date: 2022-09-30T00:21:05+08:00
description: "学习 cs61c 的一些总结"
tags: ["RISC-V", "CS61c"]
ShowToc: true
---

## 从寄存器开始

C 语言是对变量进行各种操作，可以用 int double 这样的基本类型定义需要的变量，也可以用 struct 通过对基本类型的组合，定义所需的复合类型；而在汇编语言上， 不能像 C 语言那样直接定义所需的变量，能够操作的是寄存器，通过指令、寄存器和内存，完成 C 语言之间的各种逻辑。

RISC-V 有 32 个寄存器，分别为 x0-x31, 为了更好的操作寄存器，根据功能的不同，每个寄存器都有一个单独的 symbolic name 。 sp(x2) 寄存器始终保存当前栈的内存地址，a0-a7 是用于参数传递的寄存器。另外， RISC-V 还有个单独的 pc 寄存器， 始终保存着程序下一个指令的地址。

| Register | ABI Name | Description                       | Saver  |
| -------- | -------- | --------------------------------- | ------ |
| x0       | zero     | Hard-wired zero                   | -      |
| x1       | ra       | Return address                    | Caller |
| x2       | sp       | Stack pointer                     | Callee |
| x3       | gp       | Global pointer                    | -      |
| x4       | tp       | Thread pointer                    | -      |
| x5       | t0       | Temporary/alternate link register | Caller |
| x6-x7    | t1-t2    | Temporaries                       | Caller |
| x8       | s0/fp    | Saved regsiter/frame pointer      | Callee |
| x9       | s1       | Saved regsiter                    | Callee |
| x10-11   | a0-1     | Function arguments/return values  | Caller |
| x12-17   | a2-a7    | Function arguments                | Caller |
| x18-27   | s2-11    | Saved regsiter                    | Callee |
| x28-31   | t3-t6    | Temporaries                       | Caller |

### Caller VS Callee 保存

Caller 是调用其他函数的函数， Callee 是被调用的函数

所以，Caller 保存是指该寄存器需要在调用函数之前保存到栈，调用函数结束后，再从栈中取回，被调用函数在使用时不会考虑该寄存器是否存在值，Callee 保存指该寄存器的指需要被调用函数在函数开始时保存到栈上，在函数返回前从栈上恢复。

~~这些关于寄存器的操作都是约定俗成的，a0-a7, s0-s7,t0-t7，也可以完全按照自己的的想法来使用和保存，但是，这样你写的程序就不能正常的和其他程序一起工作，所以还是按照这些约定俗成的规则来使用寄存器。~~



汇编函数调用流程如下：

1. 将函数参数保存到a0-a7寄存器

2. 调用被调用函数 jump

3. 增加栈，将要保存的寄存器保存到栈上

4. 完成该函数需要的功能

5. 将返回值保存到 a0

6. 恢复保存的寄存器，减小栈

7. 返回到调用函数，jr ra



## 汇编指令

RISC-V 的指令规则比较简单， 基本上可以总结成一个规则：

```
OP dec source1 source2
```

RISC-V 的指令是定长的， 每个指令都是32位。

具体的指令可以看 [RISC-V Reference Card](https://github.com/parkma99/parkma99.github.io/raw/main/resources/pdf/reference-card.pdf)