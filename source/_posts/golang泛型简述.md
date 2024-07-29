---
title: golang泛型简述
date: 2024-07-26 10:42:22
tags:
  - go
  - Genrics
  - EBNF
---

## 简述Generics

`Generics` 是一种基本所有现代高级强类型编程语言中都会提供的特性，各个语言的实现模式不一样，例如：

- java基于**类型擦除（Type Erasure）**实现，简单来说就是被**在运行时被实际转换为声明的类型（如果没有声明就是object）**。这意味着在运行时我们已经丢失实际的类型。
- `cpp` 基于 `Template` 来实现泛型，**模版在编译时进行实例化**。
- `golang` 基于 **类型参数** 和 **约束** 来实现泛型。

总的来说优缺点如下表所示：

| 特性/语言          | Java 泛型                                                    | C++ 模板                                                     | Go 泛型                                                      |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **实现方式**       | 类型擦除（Type Erasure）                                     | 编译时实例化                                                 | 类型参数和约束                                               |
| **类型安全**       | 有                                                           | 有                                                           | 有                                                           |
| **编译时类型检查** | 有                                                           | 有                                                           | 有                                                           |
| **灵活性**         | 差                                                           | 极高，支持模板化和模板元变成                                 | 一般                                                         |
| **复杂性**         | 简单                                                         | 极复杂                                                       | 简单                                                         |
| **运行时开销**     | 有                                                           | 无                                                           | 有                                                           |
| **缺点**           | - 运行时无法获取泛型类型信息<br/>- 不能创建泛型数组<br/>- 不能使用基本类型作为泛型参数<br/>- 不能在静态上下文中使用泛型类型参数<br/>- 不能直接实例化泛型类型参数 | - 编译速度慢：模板实例化可能导致编译时间增加<br/>- 代码膨胀：大量模板实例化可能导致生成的二进制代码体积增大<br/>- 错误信息复杂：模板错误信息可能难以理解和调试 | - 相对较新的特性，生态系统和社区支持可能还不够成熟<br/>- 可能引入一些性能开销，具体需要根据实际使用场景评估<br/> |

### Type Erasure 的运行时类型

下面的代码中，结果是 true，这是因为 String 和 Integer 在运行时都被擦除为 Object 类型。

```java
public class TypeErasureExample {
    public static void main(String[] args) {
        List<String>  stringList  = new ArrayList<String>();
        List<Integer> integerList = new ArrayList<Integer>();

        System.out.println(stringList.getClass() == integerList.getClass()); // 输出 true
    }
}
```

### Type Erasure 的运行时安全保证

在下面的代码中，只是会有一些 `WARNING` 信息，然而实际代码是可以执行的，只是在运行时会抛出异常 `java.lang.ClassCastException`，这是因为由于我们使用了 `<Integer>` 作为限制，类型会被擦除为 `<Integer>` 而不是上面例子中的 `Object`。然而，运行时会对类型进行检查，并转换 String 到 Integer。

```java
public class MyList<T> {

    private final List<T> array;

    public MyList() {
        this.array = new ArrayList<>();
    }

    public void add(T t) {
        array.add(t);
    }
}
```

```java
public class IntegerList extends MyList<Integer> {
    @Override
    public void add(Integer integer) {
        super.add(integer);
    }

    public static void main(String[] args) {
        IntegerList integerList = new IntegerList();
        // WARNING: Raw use of parameterized class 'MyList'
        MyList      myList      = integerList;
        // Exception in thread "main" java.lang.ClassCastException: java.lang.String cannot be cast to java.lang.Integer
        myList.add("hello");
    }
}
```

### CPP的模版

CPP的模板是在编译时，对模版代码根据用户调用进行实例化，也就是说，模版对应于java里的class，而最终会生成一个真正的函数，对应于java里的实例（instance）。灵活性非常之高， 但是也极度复杂，写多了掉头发。

PS：在我多年工作经验中，几乎没有碰见过**常用**cpp模版的人。

```cpp
// 定义一个模板函数，用于比较两个值并返回较大的那个
template <typename T>
T getMax(T a, T b) {
    return (a > b) ? a : b;
}

// 如果存在如下代码，那会在编译时生成一个新的函数
auto m = getMax(1, 2);
int getMax(int a, int b) {
    return (a > b) ? a : b;
}
```

## Golang Generics

### golang generics 的定义

> golang 的 generics 主要包含三个特性：
>
> 1. Type parameters for functions and types
> 2. Type sets defined by interfaces
> 3. Type inference
>
> 其中：
>
> 1. 声明了函数，struct等golang常用特性应该如何声明，使用 generics；
> 2. 声明了如何在 `interface` 中定义 generics；
> 3. 声明了编译器如何对generics的类型进行推导；
>
> 我们会在后面对这几个主要特性进行探讨。

### 几个简单的例子

#### type parameter for functions and types

下面的例子说明了如何在function中使用泛型。

```go
// Sum sums the values of map m.
// It supports both int64 and float64 as typed of map values
func Sum[K comparable, V int64 | float64](m map[K]V) V {
	var s V
	for _, value := range m {
		s += value
	}
	return s
}
```

#### Type sets defined by interfaces

下面的例子中说明了如何在interface中使用泛型

```go
// Number declare Number interface type to use as a type constraint
// Essentially, you’re moving the union from the function declaration into a new type constraint.
// That way, when you want to constrain a type parameter to either int64 or float64,
// you can use this Number type constraint instead of writing out int64 | float64.
type Number interface {
	int64 | float64
}

// SumNumber a function with generics with constraint that indicate map's value has type of Number
func SumNumber[K comparable, V Number](data map[K]V) V {
	var s V
	for _, value := range data {
		s += value
	}
	return s
}
```

#### type infer

下面的例子中说明了如何手动的指定泛型类型以及如何使用编译器的类型推导（type infer）来实现函数调用。

```go
func sumInt64() {
	ints := map[string]int64{
		"one": 1,
		"two": 2,
	}

	// we should specify type arguments before we call the generic function
	ret := generics.Sum[string, int64](ints)
	fmt.Println(ret)

	// call the generic function without specify type argument, leave it to type infer
	ret = generics.Sum(ints)
	fmt.Println(ret)
}
```

#### struct with generics

下面的例子定义了一个泛型接口，但是泛型接口的实例是非泛型的

```go
type Id[T interface{}] interface {
	GetId() T
}

type PageView struct {
	IntId int
}

func (p PageView) GetId() int {
	return p.IntId
}
```

如果我们想要泛型实例本身也是泛型的，那么他们本身也必须是泛型的，对于上面的例子中的 `PageView`，他并不是一个泛型的结构体，所以他无法实现泛型的接口。如果我们需要实现一个泛型的接口，我们可以这样处理：

```go
type IntNumber interface {
	int | int64
}

type GenericPageView[T IntNumber] struct {
	IntNumberId T
}

func (g GenericPageView[T]) GetId() T {
	return g.IntNumberId
}
```

#### 使用接口作为泛型的返回值/类型断言

注意，这里必须要是使用**类型断言**将接口类型转换为T并返回。

```go
type Gender interface {
	GetGender() string
}

type Male struct {
}

func (m Male) GetGender() string {
	return "male"
}

type Female struct {
}

func (m Female) GetGender() string {
	return "male"
}

func NewGenderValue[T Gender](gender string) T {
	var g Gender
	if gender == "male" {
		g = Male{}
	} else {
		g = Female{}
	}
	return g.(T)
}
```

#### 对底层类型基于 `~` 进行约束

`~` 是一种类型约束符号，用于表示一组类型的集合。具体来说，`~int` 可以用于表示所有底层类型为int的类型。下面是一个简单的例子：

```go
type MyInt int

// IntArray 声明一个元素底层类型为int的数组
type IntArray[T ~int] []T

func FillIntArray() {
	var intArr IntArray[int]
	for i := 0; i < 10; i++ {
		intArr = append(intArr, i)
	}
}

func FillMyIntArray() {
	var intArr IntArray[MyInt]
	var i MyInt
	for i = 0; i < 10; i++ {
		intArr = append(intArr, i)
	}
}
```

#### 泛型可能涉及到的断言

参考 [Cannot use type assertion on type parameter value](https://stackoverflow.com/questions/71587996/cannot-use-type-assertion-on-type-parameter-value)，我们在使用泛型的过程中，可能会遇到一些类型无法使用**类型断言**的情况，可以使用如下方法绕过限制。

> `x`'s type `T` is a type parameter, not an interface. It is only constrained by an interface. The Go (revised `1.18`) language spec explicitly states [type parameters](https://go.dev/ref/spec#Type_parameter_declarations) are not allowed in a [type assertion](https://go.dev/ref/spec#Type_assertions):
>
> For an expression `x` of ***interface type, but not a type parameter***, and a type `T` ... the notation `x.(T)` is called a type assertion.

```go
func GenericIsInt[T interface{}](x T) bool {
	_, ok := any(x).(int)
	return ok
}
```

### 一些常见的错误

#### golang目前不支持方法泛型

```go
type Person struct{}

// Method cannot have type parameters
func (p *Person) Say[T int | string](s T) {
    fmt.Println(s)
}
```

#### Cannot use a type parameter as RHS in type declaration

Go 语言不允许在类型声明中直接使用类型参数作为右侧的类型，因为在部分情况下可能会产生歧义。

```go
// Cannot use a type parameter as RHS in type declaration
type FloatNumber[T float64 | float32] T
```

#### Invalid recursive type

Go不允许直接嵌套自身类型的实例，这是为了防止无限递归导致的编译器和内存问题，我们可以通过指针来避免这个问题。

```go
// ERROR : Invalid recursive type 'Tree'
type Tree[T interface{}] struct {
	Left  Tree[T]
	Right Tree[T]
	Val   T
}

// the code below is legal
type Tree[T interface{}] struct {
	Left  *Tree[T]
	Right *Tree[T]
	Val   T
}
```

### generics EBNF

现在我们差不多了解了golang generics的一些常用用法了，我们可以再回到go spec的一些EBNF来查看泛型的使用。

#### Prerequisite

这里，必须先解释一些重要的概念：

- `type arguments` 类型参数是用于定义泛型类型或泛型函数的占位符，通常在定义时指定。
- `type parameter` 类型实参是用于替换类型参数的具体类型，当使用泛型类或者泛型函数时，需要提供具体的类型作为类型实参。

例如下面的这个例子：

```go
// 定义一个泛型函数，T 是 type arguments
func Print[T any](value T) {
    fmt.Println(value)
}

// 使用具体类型 int 作为 type parameter
Print[int](42)
```

#### [Types](https://go.dev/ref/spec#Types)

> A type may be denoted by a *type name*, if it has one, which must be followed by [type arguments](https://go.dev/ref/spec#Instantiations) if the type is generic.

```elixir
Type      = TypeName [ TypeArgs ] | TypeLit | "(" Type ")" .
TypeName  = identifier | QualifiedIdent .
TypeArgs  = "[" TypeList [ "," ] "]" .
TypeList  = Type { "," Type } .
TypeLit   = ArrayType | StructType | PointerType | FunctionType | InterfaceType |
            SliceType | MapType | ChannelType .
```

#### [Type parameter declarations](https://go.dev/ref/spec#Type_parameter_declarations)

> A type parameter list declares the *type parameters* of a generic function or type declaration. The type parameter list looks like an ordinary [function parameter list](https://go.dev/ref/spec#Function_types) except that the type parameter names must all be present and the list is enclosed in square brackets rather than parentheses [[Go 1.18](https://go.dev/ref/spec#Go_1.18)].

```elixir
TypeParameters  = "[" TypeParamList [ "," ] "]" .
TypeParamList   = TypeParamDecl { "," TypeParamDecl } .
TypeParamDecl   = IdentifierList TypeConstraint .
```

### [Instantiations](https://go.dev/ref/spec#Instantiations)

A generic function or type is *instantiated* by substituting `type arguments` for the `type parameters` [[Go 1.18](https://go.dev/ref/spec#Go_1.18)]. Instantiation proceeds in two steps:

1. Each type argument is substituted for its corresponding type parameter in the generic declaration. This substitution happens across the entire function or type declaration, including the type parameter list itself and any types in that list.
2. After substitution, each type argument must [satisfy](https://go.dev/ref/spec#Satisfying_a_type_constraint) the [constraint](https://go.dev/ref/spec#Type_parameter_declarations) (instantiated, if necessary) of the corresponding type parameter. Otherwise instantiation fails.

> **Instantiating a type results in a new non-generic [named type](https://go.dev/ref/spec#Types); instantiating a function produces a new non-generic function.**

When using a generic function, type arguments may be provided explicitly, or they may be partially or completely [inferred](https://go.dev/ref/spec#Type_inference) from the context in which the function is used. Provided that they can be inferred, type argument lists may be omitted entirely if the function is:

- [called](https://go.dev/ref/spec#Calls) with ordinary arguments,
- [assigned](https://go.dev/ref/spec#Assignment_statements) to a variable with a known type
- [passed as an argument](https://go.dev/ref/spec#Calls) to another function, or
- [returned as a result](https://go.dev/ref/spec#Return_statements).

here is a example below:

```go
// sum returns the sum (concatenation, for strings) of its arguments.
func sum[T ~int | ~float64 | ~string](x ...T) T {
	var t T
	for _, v := range x {
		t += v
	}
	return t
}

x := sum                       // illegal: the type of x is unknown
intSum := sum[int]             // intSum has type func(x... int) int
a := intSum(2, 3)              // a has value 5 of type int
b := sum[float64](2.0, 3)      // b has value 5.0 of type float64
c := sum(b, -1)                // c has value 4.0 of type float64

type sumFunc func(x... string) string
var f sumFunc = sum            // same as var f sumFunc = sum[string]
f = sum                        // same as f = sum[string]
```

A partial type argument list cannot be empty; at least the first argument must be present. The list is a prefix of the full list of type arguments, leaving the remaining arguments to be inferred. Loosely speaking, type arguments may be omitted from "right to left".

```go
func apply[S ~[]E, E any](s S, f func(E) E) S { … }

f0 := apply[]                  // illegal: type argument list cannot be empty
f1 := apply[[]int]             // type argument for S explicitly provided, type argument for E inferred
f2 := apply[[]string, string]  // both type arguments explicitly provided

var bytes []byte
r := apply(bytes, func(byte) byte { … })  // both type arguments inferred from the function arguments
```

## 引用

- [泛型](https://www.liwenzhou.com/posts/Go/generics/#c-0-2-0)
- [跟着 Go 作者学泛型](https://polarisxu.studygolang.com/posts/go/generics/gophercon2021-generics/)
- [Type Erasure in Java Explained](https://www.baeldung.com/java-type-erasure)
- [Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics)
- [Cannot use type assertion on type parameter value](https://stackoverflow.com/questions/71587996/cannot-use-type-assertion-on-type-parameter-value)
- [The Go Programming Language Specification[Language version go1.22 (Feb 6, 2024)]](https://go.dev/ref/spec)
