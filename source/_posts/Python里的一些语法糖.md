---
title: Python里的一些语法糖
date: 2025-05-08 20:02:44
tags:
---

## 引用

- [The `yield` statement](https://docs.python.org/3/reference/simple_stmts.html#grammar-token-python-grammar-yield_stmt)
- [Operators and Expressions in Python](https://realpython.com/python-operators-expressions/)
- [Syntactic Sugar: Why Python Is Sweet and Pythonic](https://realpython.com/syntactic-sugar-python/#unpacking-in-python)
- [How to Use Generators and yield in Python](https://realpython.com/introduction-to-python-generators/)
- [When to Use a List Comprehension in Python](https://realpython.com/list-comprehension-python/)
- [Python Dictionary Comprehensions: How and When to Use Them](https://realpython.com/python-dictionary-comprehension/)
- [List Comprehensions](https://docs.python.org/2/tutorial/datastructures.html#list-comprehensions)
- [Dictionaries](https://docs.python.org/3/tutorial/datastructures.html#dictionaries)

## 关键词

- Syntactic  Sugar
- Operators
- Assignment Expressions
- for loops
- comprehensions
- constructor

## Operators

| Operator |                            Method                            |
| :------: | :----------------------------------------------------------: |
|   `+`    | [`__add__`](https://docs.python.org/3/reference/datamodel.html#object.__add__) |
|   `-`    | [`__sub__`](https://docs.python.org/3/reference/datamodel.html#object.__sub__) |
|   `*`    | [`__mul__`](https://docs.python.org/3/reference/datamodel.html#object.__mul__) |
|   `/`    | [`__truediv__`](https://docs.python.org/3/reference/datamodel.html#object.__truediv__) |
|   `//`   | [`__floordiv__`](https://docs.python.org/3/reference/datamodel.html#object.__floordiv__) |
|   `**`   | [`__pow__`](https://docs.python.org/3/reference/datamodel.html#object.__pow__) |
|   `in`   | [`__contains__`](https://docs.python.org/3/reference/datamodel.html#object.__contains__) |

every time we use an operator, Python calls the corresponding special method for us. For example: `1 + 2` equals to `(1).__add__(2)` but way more complex and unreadable.

```python
class NumberOperator:
    def __init__(self, number: int):
        self.number = number

    def __add__(self, other):
        v = self.number + other.number
        return NumberOperator(v)

def add():
    one = NumberOperator(1)
    two = NumberOperator(2)
    three = one + two
    print(f"one = {one.number}")
    print(f"two = {two.number}")
    print(f"three = {three.number}")
```

### membership tests

```python
def membership_test():
    print(1 in [1, 2])
    print(1 not in [1, 2])
    print(3 in [1, 2])
    print(3 not in [1, 2])
```

## Chained Conditions

```python
def chained_conditions(number: int) -> bool:
    if 1 <= number <= 10:
        return True
    else:
        return False
```

## [Conditional expressions](https://docs.python.org/3/reference/expressions.html#conditional-expressions)

```yacas
or_test  ::= and_test | or_test "or" and_test
and_test ::= not_test | and_test "and" not_test
not_test ::= comparison | "not" not_test

conditional_expression ::= or_test ["if" or_test "else" expression]
expression             ::= conditional_expression | lambda_expr
```

## Assignment Expressions

> Traditional assignments built with the `=` operator don’t return a value, so they’re [statements](https://docs.python.org/3/glossary.html#term-statement) but not [expressions](https://docs.python.org/3/glossary.html#term-expression). In contrast, **assignment expressions** are assignments that return a value. You can build them with the walrus operator (`:=`).

```python
# this is an assignment expression, so it's syntactically correct.
def assignment_expression():
    while (line := input("Type some text: ")) != "stop":
        print(line)
```

```python
# this is a normal assignment, it doesn't return a value, so it's syntactically incorrect
def assignment():
    # Unresolved reference 'line'
    # while (line = input("Type some text: ")) != "stop":
    #     print(line)
    pass
```

## Unpacking

> Unpacking an iterable means assigning its values to a series of variables one by one.

### Iterable Unpacking

```python
# Once Python runs this assignment, the values are unpacked into the corresponding variable by position.
one, two three, four = [1, 2, 3, 4]

a = 200
b = 100
# using the unpacking syntactic sugar, we can swap values between variables without assigning temporary variable
a, b = b, a
```

### `*args` and `**kwargs`

```python
# 1
# (2, 3, 4)
# {'hello': 'hello', 'world': 'world'}
# args will be bound to positional arguments, and kwargs will be bound to keyword arguments
def args_func(number:int, *args, **kwargs):
    print(number)
    print(args)
    print(kwargs)


if __name__ == '__main__':
    args_func(1, 2, 3, 4, hello="hello", world="world")
```

```python
# I suppose that there are two different types of arguments in Python's arguments,
# The first type is named argument and the other type is anonymous argument.
# In this situation, "name" is a named argument, and "hello" and "world" is anonymous arguments,
# so "**kwargs" will be bound to anonymous arguments rather than being bound to all arguments
def show_args(number: int, *args, name: str, **kwargs):
    args_func(number, *args, **kwargs)
    print(name)


if __name__ == '__main__':
    # TypeError: show_args() missing 1 required keyword-only argument: 'name'
    # show_args(1, 2, 3, "name", 4, hello="hello", world="world")

    # TypeError: show_args() missing 1 required keyword-only argument: 'name'
    # show_args(1, 2, 3, 4, "name", hello="hello", world="world")
    
    show_args(1, 2, 3, 4, name="name", hello="hello", world="world")
    # 1
		# (2, 3, 4)
		# {'hello': 'hello', 'world': 'world'}
		# name
```

> What will be happening if we pass a list as a `*args` and pass a dict as a `**kwargs`

```python
def show_args2(*args, **kwargs):
    print(args)
    print(kwargs)


def call_show_args2():
    list1 = [1, 2, 3, 4]
    list2 = [3, 4, 5, 6]

    dict1 = {"k1": "v1", "k2": "v2"}
    show_args2(list1, list2, dict1=dict1)
    # both of list1 and list2 are bound to *args, it's worth noticing that each list is an entirety which being passed to *args
    # ([1, 2, 3, 4], [3, 4, 5, 6])
    # {'dict1': {'k1': 'v1', 'k2': 'v2'}}

    show_args2(list1, dict1=dict1)
    # ([1, 2, 3, 4],)
    # {'dict1': {'k1': 'v1', 'k2': 'v2'}}

    show_args2(*list1, dict1=dict1)
    # using *list1 will decompose list into a series of variables bound to *args
    # (1, 2, 3, 4)
    # {'dict1': {'k1': 'v1', 'k2': 'v2'}}

```

## Loops and Comprehensions

### Exploring `for` Loops

```python
def simple_loops():
    numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    for number in numbers:
        print(number, end=', ')

def iter_loops():
    numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    it = iter(numbers)
    while True:
        try:
            number = next(it)
            print(number, end=' ')
        except StopIteration:
            break

if __name__ == '__main__':
    iter_loops()
```

### Using Comprehensions

> say that we have a list of numbers as strings, and we want to convert them into numeric values and build a new list of squared values.

```python
def square_string_list():
    strings = ["1", "2", "3", "4", "5", "6", "7", "8", "9"]
    numbers = [int(num)**2 for num in strings]
    print(numbers)
    # [1, 4, 9, 16, 25, 36, 49, 64, 81]

if __name__ == '__main__':
    square_string_list()
```

## Decorators

Decorators are functions that take another function as an argument and extend their behavior dynamically without explicitly modifying it. **In Python, you have a dedicated syntax that allows you to apply a decorator to a given function**: In this piece of syntax, the `@decorator` part tells Python to call `decorator()` with the `func` **object as an argument**.

```python
@decorator
def func():
    <body>
```

> The following code produces a TypeError called ** 'NoneType' object is not callable** since decorator function prev returns nothing to be executed:
>
> 1. before `f()` executing, the decorator `prev()` was executed;
> 2. the execution of `prev()` returns nothing which means the `()` in `f()` has nothing to execute;

```python
def prev(*args, **kwargs):
    print(f'args = {args}')
    print(f'kwargs = {kwargs}')

@prev
def f():
    print('running')

if __name__ == '__main__':
    f()
```

> The following code is correct.

```python
def prev(*args, **kwargs):
    print(f'args = {args}')
    print(f'kwargs = {kwargs}')
    return args[0]

@prev
def f():
    print('running')

if __name__ == '__main__':
    f()

# args = (<function f at 0x10b944ca0>,)
# kwargs = {}
# running
```

> Here is a bizarre statement that we never call `f()` but its decorator `prev()` was invoked automatically by compiler. It's because **decorator is loaded during loading modules**

```python
def prev(*args, **kwargs):
    print(f'args = {args}')
    print(f'kwargs = {kwargs}')
    return args[0]

@prev
def f():
    print('running')


if __name__ == '__main__':
    pass
```

> decorator with `lambda`

```python
def decorator_with_arguments(*args, **kwargs):
    def running(name: str, age: int):
        print(f'args = {args}')
        print(f'kwargs = {kwargs}')
        print(f'name = {name}')
        print(f'age = {age}')
        outer = args[0]
        return outer(name=name, age=age)

    return running


@decorator_with_arguments
def func_with_arguments(name: str, age: int):
    print(f'name = {name}, age = {age}')


if __name__ == '__main__':
    func_with_arguments(name='tony', age=20)

# args = (<function func_with_arguments at 0x10b553ca0>,)
# kwargs = {}
# name = tony
# age = 20
# name = tony, age = 20
```

> decorator with `lambda` two:

```python
def decorator_with_arguments2(func):
    def wrapper(*args, **kwargs):
        print(f'args = {args}')
        print(f'kwargs = {kwargs}')
        return func(*args, **kwargs)
    return wrapper

@decorator_with_arguments2
def func_with_arguments2(name: str, age: int):
    print(f'func_with_arguments2 : name = {name}, age = {age}')

if __name__ == '__main__':
    func_with_arguments2(name='tony', age=20)

# args = ()
# kwargs = {'name': 'tony', 'age': 20}
# func_with_arguments2 : name = tony, age = 20
```

## F-String Literals

```python
debit = 300
credit = 450

print(f"Debit: ${debit:.2f}, Credit: ${credit:.2f}, Balance: ${credit - debit:.2f}")
```

## The `yield` from Construct

Introduced with [PEP 255](https://www.python.org/dev/peps/pep-0255), **generator functions** are a special kind of function that return a [lazy iterator](https://en.wikipedia.org/wiki/Lazy_evaluation). These are objects that you can loop over like a [list](https://realpython.com/python-lists-tuples/). However, unlike lists, lazy iterators do not store their contents in memory.

```python
def gen_infinite_sequence():
    num = 0
    while True:
        yield num
        num += 1

if __name__ == '__main__':
    gen = gen_infinite_sequence()
    print(gen)
    for i in range(5):
        print(next(gen), end=' ')

# <generator object gen_infinite_sequence at 0x10be6cf90>
# 0 1 2 3 4
```

## Understanding Generators

> Python yield statement and the code that follows it. `yield` indicates where a value is sent back to the caller, but unlike `return`, you don’t exit the function afterward.
>
> Instead, the **state** of the function is remembered. That way, when `next()` is called on a generator object (either explicitly or implicitly within a `for` loop), the previously yielded variable `num` is incremented, and then yielded again.

### Profiling Generator Performance

```python
def profiling_yield():
    import cProfile
    cProfile.run('sum([i * 2 for i in range(10000)])')

if __name__ == '__main__':
    profiling_yield()
```

```
   Ordered by: standard name

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.001    0.001    0.001    0.001 <string>:1(<listcomp>)
        1    0.000    0.000    0.001    0.001 <string>:1(<module>)
        1    0.000    0.000    0.001    0.001 {built-in method builtins.exec}
        1    0.000    0.000    0.000    0.000 {built-in method builtins.sum}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}
```

### Understanding the python yield statement

```python
def multi_yield():
    yield_string = "first string"
    yield yield_string
    yield_string = "second string"
    yield yield_string

if __name__ == '__main__':
    gen = multi_yield()
    print(next(gen))
    print(next(gen))
# first string
# second string
```

### Using Advanced Generator Methods

|       method        | description                                                  |
| :-----------------: | ------------------------------------------------------------ |
| `generator.send()`  | Resumes the execution and “sends” a value into the generator function. The `value` argument becomes the result of the current `yield` expression. The `send()` method returns the next value yielded by the generator, or raises `StopIteration` if the generator exits without yielding another value. |
| `generator.throw()` | `.throw()` allows you to throw exceptions with the generator. I |
| `generator.close()` | As its name implies, `.close()` allows you to stop a generator. |

## The `with` Statement

Python’s `with` statement is a handy tool that allows you to manipulate [context manager](https://docs.python.org/3/glossary.html#term-context-manager) objects. These objects automatically handle the *setup* and *teardown* phases whenever you’re dealing with external resources such as files.

```python
with open("hello.txt", mode="w", encoding="utf-8") as file:
    file.write("Hello, World!")

# equals to
    
file = open("hello.txt", mode="w", encoding="utf-8")

try:
    file.write("Hello, World!")
finally:
    file.close()
```

## List Comprehension

> List comprehensions provide a concise way to create lists. Common applications are to make new lists where each element is the result of some operations applied to each member of another sequence or iterable, or to create a subsequence of those elements that satisfy a certain condition.

```python
squares = []

for x in range(10):
  	squares.append(x**2)
```

we can obtain the same result with

```python
squares = [x ** 2 for x in range(10)]
```

This is also simliar to the following code, except that the following code return **<map object at 0x100d01760>** which is a stream of data that is lazily initiated when the it's called.

```python
squares = map(lambda x: x**2, range(10))
print(squares)
# <map object at 0x100d01760>
for s in squares:
  print(s, end=' ')
  # 0 1 4 9 16 25 36 49 64 81 
```

A list comprehension consists of brackets containing an expression followed by a [`for`](https://docs.python.org/2/reference/compound_stmts.html#for) clause, then zero or more [`for`](https://docs.python.org/2/reference/compound_stmts.html#for) or [`if`](https://docs.python.org/2/reference/compound_stmts.html#if) clauses. The result will be a new list resulting from evaluating the expression in the context of the [`for`](https://docs.python.org/2/reference/compound_stmts.html#for) and [`if`](https://docs.python.org/2/reference/compound_stmts.html#if) clauses which follow it. For example, this listcomp combines the elements of two lists if they are not equal:

```python
def lc2():
    arr = [(x, y)
           for x in [1, 2, 3]
           for y in [3, 1, 4]
           if x != y]
    for item in arr:
        print(item, end=' ')
        # (1, 3) (1, 4) (2, 3) (2, 1) (2, 4) (3, 1) (3, 4)
```

The forementioned code is equivalent to the following code but way more readable:

```python
    for_arr = []
    for x in [1,2,3]:
        for y in [3,1,4]:
            if x != y:
                for_arr.append((x, y))
    for item in for_arr:
        print(item, end=' ')
    # (1, 3) (1, 4) (2, 3) (2, 1) (2, 4) (3, 1) (3, 4) 
```

If the expression is a tuple (e.g. the `(x, y)` in the previous example), it must be parenthesized.

```python
[x, x**2 for x in range(6)]
#   File "<stdin>", line 1, in <module>
#    [x, x**2 for x in range(6)]
```

#### [Nested List Comprehensions](https://docs.python.org/2/tutorial/datastructures.html#nested-list-comprehensions)

The initial expression in a list comprehension can be any arbitrary expression, including another list comprehension.

```python
def nested_list_comprehensions():
    matrix = [
        [1, 2, 3, 4],
        [5, 6, 7, 8],
        [9, 10, 11, 12],
    ]
    print([[row[i] for row in matrix] for i in range(len(matrix))])
    # [[1, 5, 9], [2, 6, 10], [3, 7, 11]]
    print([[row[i] for row in matrix] for i in range(4)])
    # [[1, 5, 9], [2, 6, 10], [3, 7, 11], [4, 8, 12]]
```

## Dictionary And dict comprehensions

In addition, dict comprehensions can be used to create dictionaries from arbitrary key and value expressions:

```python
def dictionaries():
    v = {x: x ** 2 for x in (2, 4, 6)}
    print(v)
    # {2: 4, 4: 16, 6: 36}
```



