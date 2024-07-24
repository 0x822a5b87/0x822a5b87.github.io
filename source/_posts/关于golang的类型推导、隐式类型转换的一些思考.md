---
title: 关于golang的类型推导、隐式类型转换的一些思考
date: 2024-04-19 17:02:53
tags:
---

## 参考

[Comparison operators in Go](https://medium.com/golangspec/comparison-operators-in-go-910d9d788ec0)

[golang类型推断与隐式类型转换](https://juejin.cn/post/7095489639024164894)

[golang 编译器的强制类型转换](https://github.com/golang/go/blob/master/src/cmd/compile/internal/typecheck/const.go#L339)

[golang SSA(Static Single Assignment)](https://github.com/golang/go/tree/master/src/cmd/compile/internal/ssa)

[Conversions](https://go.dev/ref/spec#Conversions)

[Conversions [complete list]](https://yourbasic.org/golang/conversions/)

## 前言

今天，有个朋友突然问了我一个问题，为什么在以下代码中会发生以下这种情况呢？

1. `secretId == ""` 可以正常的进行比较；
2. `printString(secretId)` 会提示类型不匹配；

```go
import "fmt"

type Password string

func printString(str string) {
	fmt.Println(str)
}

func testPrint() {
	var secretId Password = "0"

	// 这里不会报错
	if secretId == "" {
		fmt.Println(secretId)
	}

	// Cannot use 'secretId' (type Password) as the type string
	printString(secretId)
}
```

看到这个问题我的第一反应是由于 **类型推导（Type inference）**，`""` 被直接推导为 `Password` 类型，所以可以直接和 secretId 对象进行比较。可惜的是，在 golang 反编译的代码中已经丢失了类型信息，所以只能慢慢探索了。



## 探索

### 第一次遇到的问题



**如果"" 被隐式的推导为 Password 类型**，那么使用 `==` 对他们的比较是合法的，于是我就开始验证自己的想法，在验证自己想法过程中，却发现了一个更神奇的问题：**go语言中对于const变量和非const变量的类型推导难道是不一样的？**可以看到在下面的代码中：

1. `const` 标记的 `constPassString` 可以和 `Password` 类型比较；
2. 非 `const` 标记的 `passString` 却不能和 `Password` 比较；

```go
package main

import (
	"fmt"
	"reflect"
)

type Password string

func printString(str string) {
	fmt.Println(str)
}

func testPrint() {
	var secretId Password = "0"

	// 这里不会报错
	if secretId == "" {
		fmt.Println(secretId)
	}

	const constPassString = ""
	// 这里不会报错
	if constPassString == secretId {
		fmt.Println(constPassString)
	}
	// 输出 string
	fmt.Println(reflect.TypeOf(constPassString))

	// 编译失败
	// Invalid operation: passString == secretId (mismatched types string and Password)
	var passString = ""
	if passString == secretId {
		fmt.Println(passString)
	}
}

func main() {
	testPrint()
}
```

### 隐式类型转换的规则

查阅了golang spec之后，我们得到几个重要结论：

> 1. The only implicit conversion in Go is when an **untyped constant** is used in a situation where a type is required.
> 2. Except for shift operations, if one operand is an untyped [constant](https://go.dev/ref/spec#Constants) and the other operand is not, the constant is implicitly [converted](https://go.dev/ref/spec#Conversions) to the type of the other operand.

这里的 constant 很好理解，就是我们的 `const` 关键词指定的变量，而 `untyped` 我们可能会错误的理解为 **无类型**，实际确实指的是 **未明确声明的类型或者由 type inference 推导出来的类型**。

```go
// typedValue 表示的是 int 类型，这个类型是由 int 关键词声明的
var typedValue int = 10
// untypedValue 虽然也是 int 类型，但是它是由 type inference 推导出来的
// 根据 golang spec，每个 untyped 类型都会有他的默认类型
// 例如 10 的 默认类型是 int，也就是他默认会被推到为 int 类型
var untypedValue = 10

// const int 类型
const constTypedValue int = 10
// const untyped int
const constUntypedValue = 10
```

所以，在我们上面的代码就出现了这样的问题：

> 1. `""` 是一个典型的 const untyped 变量，默认类型是 string，也就是 const untyped string，由上面的原则，它被隐式的转换为 `Password` 类型，所以<1> 编译器不会报错
> 2. <2> `constPassString`也是一个 const untyped string，它也被隐式的转换为 `Password` 所以不会报错
> 3. <3> `passString` 是一个 non-const untyped string，它无法被隐式的转换为 Password 类型，所以编译器报错。

```go
package main

import (
	"fmt"
	"reflect"
)

type Password string

func printString(str string) {
	fmt.Println(str)
}

func testPrint() {
	var secretId Password = "0"

	// <1>
	if secretId == "" {
		fmt.Println(secretId)
	}

  // <2>
	const constPassString = ""
	if constPassString == secretId {
		fmt.Println(constPassString)
	}
	// 输出 string
	fmt.Println(reflect.TypeOf(constPassString))

  // <3>
	// 编译失败
	// Invalid operation: passString == secretId (mismatched types string and Password)
	var passString = ""
	if passString == secretId {
		fmt.Println(passString)
	}
}

func main() {
	testPrint()
}
```

### 隐式转换的验证

在我们下面的代码中：

1. <1>，<2>，<4> 都会报错，他们的类型分别是 untyped string, non-const typed string, const typed string，这些都不允许隐式类型转换；
2. <3> 是 const untyped string，可以被隐式的转换为 Password。

```go
package main

import "fmt"

type Password string

func printPassword(pwd Password) {
	fmt.Println(pwd)
}

func main() {
	var untypedStr = ""
	var nonConstTypedStr string = ""
	const constUntypedStr = ""
	const constTypedStr string = ""

	// <1>
	// Cannot use 'untypedStr' (type string) as the type Password
	printPassword(untypedStr)
	// <2>
	// Cannot use 'nonConstTypedStr' (type string) as the type Password
	printPassword(nonConstTypedStr)
	// <3>
	// 合法
	printPassword(constUntypedStr)
	// <4>
	// Cannot use 'constTypedStr' (type string) as the type Password
	printPassword(constTypedStr)
}
```

### 隐式类型源码

查阅资料后，我们找到了 golang 的隐式转换函数代码，位于 [const.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/typecheck/const.go)

> 令我们十分疑惑。按照我们的直观想法，我们的 secretId 不是 string，而 "" 则是 string。那么最后应该是没有办法执行默认的类型转换的。

```go
func DefaultLit(n ir.Node, t *types.Type) ir.Node {
	return convlit1(n, t, false, nil)
}

// DefaultLit on both nodes simultaneously;
// if they're both ideal going in they better
// get the same type going out.
// force means must assign concrete (non-ideal) type.
// The results of defaultlit2 MUST be assigned back to l and r, e.g.
//
//	n.Left, n.Right = defaultlit2(n.Left, n.Right, force)
func defaultlit2(l ir.Node, r ir.Node, force bool) (ir.Node, ir.Node) {
	// 如果l或者r为nil，则无法进行类型转换
  if l.Type() == nil || r.Type() == nil {
		return l, r
	}

  // 如果 l 和 r 都是非 interface 类型
	if !l.Type().IsInterface() && !r.Type().IsInterface() {
    // 无法转换bool值和非bool值
    // 也无法转换string和非string值
		if l.Type().IsBoolean() != r.Type().IsBoolean() {
			return l, r
		}
   
		if l.Type().IsString() != r.Type().IsString() {
			return l, r
		}
	}

  // 如果左边的变量有类型，则将右边的变量转换为左边的类型
  // var b int
  // b = 10
  // 此时尝试将 10 转换为 b 对应的类型
	if !l.Type().IsUntyped() {
		r = convlit(r, l.Type())
		return l, r
	}

  // 如果左边的变量为无类型，并且右边的变量有类型，则将左边的类型转换为右边的类型
  // var r int = 10
  // var l = r
  // 此时尝试将 l 转换为 r 对应的类型
	if !r.Type().IsUntyped() {
		l = convlit(l, r.Type())
		return l, r
	}

	if !force {
		return l, r
	}

	// Can't mix nil with anything untyped.
	if ir.IsNil(l) || ir.IsNil(r) {
		return l, r
	}
	t := defaultType(mixUntyped(l.Type(), r.Type()))
	l = convlit(l, t)
	r = convlit(r, t)
	return l, r
}
```

