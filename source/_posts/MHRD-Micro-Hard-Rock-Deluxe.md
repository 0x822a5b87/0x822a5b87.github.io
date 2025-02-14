---
title: MHRD(Micro Hard Rock Deluxe)
date: 2025-02-12 11:23:29
tags:
  - assmebly
  - CPU
  - hardware
---

 最近因为开了两个 [操作系统](https://github.com/0x822a5b87/Ardi) 和 [linkers](https://github.com/0x822a5b87/tiny-linkers) 的新坑，一直在学习硬件和汇编相关的知识，所以这里找了一个简单的小游戏 [MHRD(Micro Hard Rock Deluxe)](https://store.steampowered.com/app/576030/MHRD/) 用来复习一些基础的CPU相关的知识，以下是官方的描述：

> MHRD is a hardware design game, in which you design various hardware circuits in a hardware description language. The hardware circuits you design get more complex as you go until you create a fully functional CPU design.

## 语法

A design consists of the `Inputs`, `Outputs`, `Parts` and `Wires` section:

```mermaid
---
title: GRAMMER
---
flowchart LR

design("design"):::pink

subgraph grammer
	direction LR
	input_section("input_section"):::green
	output_section("output_section"):::green
	part_section("part_section"):::green
	wire_section("wire_section"):::green
end

design --> grammer

classDef pink 1,fill:#FFCCCC,stroke:#333;
classDef green fill: #696,color: #fff,font-weight: bold;
classDef purple fill:#969,stroke:#333;
classDef error fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
```



Each section starts with a corresponding section identifier, followed by a `colon`, one or more of its corresponding elements and ends with a `semicolon`:

```vhdl
[input_section] = "Inputs:" [input] (, [input])* ";"
  
[output_section] = "Outputs:" [output] (, [output])* ";"
  
[part_section] = "Parts:" [part] (, [part])* ";"
  
[wire_section] = "Wires:" [wire] (, [wire])* ";"
```

Inputs and outputs can be either single pins or a busses

```vhdl
// single pin
[input] = [input_pin_id]
  
// busses
[input] = [input_bus_id][bus_size]
```

Identifiers have to start with a letter optionally followed by an arbitrary amount of letters or numbers:

```vhdl
[id] = [letter] ([letter | digit])*
  
[letter] = [a-zA-Z]
[digit] = [0-9]
```

A bus size is given as a number in square brackets. A number has at least one digit.

```vhdl
[bus_size] = "[" [number] "]"
[number] = [digit]+
```

Parts can be written as pairs of part identifiers and part types. **Part types are always upper-case**:

```vhdl
[part] = [part_id] [part_type]

[part_id] = [id]
  
[part_type] = [uppercase_letter] ([uppercase_letter | digit])*
  
[uppercase_letter] = [A-Z]
```

Wires have a start and end. Start and end can be regular pins, busses of the designed elements or of its parts:

```vhdl
[wire] = [start] "->" [end]

[start] = [pin_value] | [input_pin] | [input_bus] | [part_id]"."[output_pin] | [part_id]"."[output_bus]
  
[end] = [output_pin] | [output_bus] | [part_id]"."[input_pin] | [part_id]"."[input_bus]
```

A pin can be either a reference to a reference to a pin id or a specific pin inside a bus :

```vhdl
[pin_value] = "0" | "1"

[input_pin] = [input_pin_id] | [input_bus_id][pin_selection]
  
[output_pin] = [output_pin_id] | [output_bus_id][pin_selection]
  
[pin_selection] = "[" [number] "]"
```

## 基础知识

> 在正式开始之前，我们需要了解一些必要的知识以方便于我们进行简单的数学计算

|         name         |                     formulation                     |
| :------------------: | :-------------------------------------------------: |
|    德摩根第一定律    |       $\neg(A \land B) = \neg A \lor \neg B$        |
| 德摩根第一定律逆定律 |      $A \lor B = \lnot(\lnot A \land \lnot B)$      |
|    德摩根第二定律    |     $\neg(A \lor B) \equiv \neg A \land \neg B$     |
| 德摩根第二定律逆定律 |      $A \land B = \lnot(\lnot A \lor \lnot B)$      |
|     逻辑与结合律     |     $(A \land B) \land C = A \land (B \land C)$     |
|     逻辑或结合律     |       $(A \lor B) \lor C = A \lor (B \lor C)$       |
|     逻辑与交换律     |               $A \land B = B \land A$               |
|     逻辑或交换律     |                $A \lor B = B \lor A$                |
|     逻辑与分配率     | $A \land (B \lor C) = (A \land B) \lor (A \land C)$ |
|     逻辑或分配率     | $A \lor (B \land C) = (A \lor B) \land (A \lor C)$  |

## tasks

> 在游戏刚开始的时候，我们只有一个默认的 `NAND（Not AND）` 模块，模块声明如下

### NAND

```vhdl
// Interface Specification:
// -----------------------
Inputs: in1, in2;
Outputs: out;
```

### NOT

```vhdl
Inputs: in;
Outputs: out;

// use NAND
Parts: myNand NAND;

Wires:
	in -> myNand.in1,
  in -> myNand.in2,
  myNand.out -> out
;
```

### AND

```vhdl
Inputs: in1, in2;
Outputs: out;

Parts: myNand NAND, myNot NOT;

Wires:
in1 -> myNand.in1,
in2 -> myNand.in2,
myNand.out -> myNot.in,
myNot.out -> out
;
```

### OR

我们简单的画一下 `OR` 的真值表：

| in1  | in2  | out  |
| ---- | ---- | ---- |
| 0    | 0    | 0    |
| 0    | 1    | 1    |
| 1    | 0    | 1    |
| 1    | 1    | 1    |

我们观察到，`OR` 的真值表可以通过转换 `NAND` 的真值表来得到

| in1  | in2  | out  |
| ---- | ---- | ---- |
| 0    | 0    | 1    |
| 0    | 1    | 1    |
| 1    | 0    | 1    |
| 1    | 1    | 0    |

```vhdl
Inputs: ini, in2;
Outputs: out;

Parts: nand NAND, notl NOT, not2 NOT;

Wires:
  inl -> notl."in,
  in2 -> not2."in,
  notl.out -> nand.ini,
  not2.out -> nand.in2,
  nand.out -> out
```

### XOR(`Exclusive-Or`)

> If _exactly_ one input is 1, the output is 1. Otherwise the output is 0.

两个输入中一个为 `1`，另一个为 `0`，可以转换为：

1. `in1` AND `in2` = `0`
2. `in1` OR `in2` = `1`

所以，我们要判断 XOR 可以表示为 $XOR(in1, in2) \equiv (\lnot (in1 \land in2)) \land (in1 \lor in2)$

#### XOR 的优化

前面的公式虽然可以实现 `XOR` 的功能，但是相对来说性能较差，我们可以通过以下方式来进行优化：
$$
\begin{aligned}
XOR(in1, in2) & = (in1 \land \lnot in2) \lor (in2 \land \lnot in1) \\
  & = \lnot(\lnot(in1 \land \lnot in2) \land \lnot(in2 \land \lnot in1))
\end{aligned}
$$
假设
$$
A = \lnot(in1 \land \lnot in2)\\
B = \lnot(in2 \land \lnot in1)
$$


那么此时我们知道
$$
XOR(in1, in2) = NAND(A, B)
$$
现在我们需要用尽可能少的门来表示B和C：
$$
\begin{aligned}
A & = \lnot(in1 \land \lnot in2)\\
& = \lnot((in1 \land \lnot in2) \lor (in1 \land \lnot in1))\\
& = \lnot(in1 \land (\lnot in2 \lor \lnot in1)) \\
& = \lnot (in1 \land (\lnot(in2 \land in1)))
\end{aligned}
$$

$$
\begin{aligned}
B &= \lnot(in2 \land \lnot in1) \\
&= \lnot((in2 \land \lnot in1) \lor (in2 \land \lnot in2))\\
&= \lnot(in2 \land (\lnot in1 \lor \lnot in2)) \\
&= \lnot(in2 \land (\lnot(in1 \land in2)))
\end{aligned}
$$

此时，我们只需要假设：
$$
C = (\lnot(in1 \land in2))
$$
我们即可得到如下，**总计使用四个 `NAND`**：

```c
C = NAND(in1, in2);
A = NAND(in1, C);
B = NAND(in2, C);
XOR(in1, in2) = NAND(A, B)
```

我们的设计代码也可以转换为如下：

```vhdl
Inputs: in1, in2;
Outputs: out;

Parts: a NAND, b NAND, c NAND, d NAND;

Wires:
	in1 -> a.in1,
  in2 -> a.in2,
	in1 -> b.in1,
	a.out -> b.in2,
	in2 -> c.in1,
	a.out -> c.in2,
	b.out -> d.in1,
  c.out -> d.in2,
	d.out -> out;
```



