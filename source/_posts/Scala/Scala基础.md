---
title: Scala基础
date: 2016-06-07
categories: Scala
tags: Scala
---

##	变量	
*	val，定义immutable variable，不可变		
* 	var，定义mutable variable，可变
*  	lazy val，惰性常量	


##	数据类型

超级父类 Any

Byte, Short, Int, Long, Char, String, Float, Double, Boolean

null表示对象引用类型为空，nothing表示程序异常终止。

```scala
scala> def foo()=throw new Exception("error")
foo: ()Nothing
```

String构建与Java的String之上，新增字符串插值特性。

```scala
val name = "Hello"	
s"new name is ${name}"
```

可以不显示指定变量类型，Scala会自动进行类型推导。

##	代码块	
```scala
{exp1 ;exp2}

{
exp1
exp2
}
```
表达式可以写在一行，分号分隔。也可以写多行。

##	函数	

```scala
def functionName( param: ParamType): ReturnType = {
	//function body: expressions
}
```
```scala
scala> def add(x:Int, y:Int)= {x+y}
add: (x: Int, y: Int)Int

scala> add(1,1)
res4: Int = 2

scala> def add(x:Int, y:Int):Int= {x+y}
add: (x: Int, y: Int)Int

scala> add(1,1)
res5: Int = 2
```
多个参数用逗号分隔。

## if表达式
```scala
if (logical_exp) valA else valB
```
和Java类似。

##	for表达式
```scala
scala> val l = List("a", "b", "c")
l: List[String] = List(a, b, c)

scala> for (
s <- l
) println(s)
a
b
c
```

##	try表达式
```scala
try{ }
catch{ }
finally{ }
```
与Java不同，try是表达式，不是关键字。

```scala
try{
	Integer.parseInt("abc")
} catch {
	case _ => 0
} finally {
	println("always be printed")
}
```

##	match表达式
```scala
exp match {
	case p1 => val1
	case p2 => val2
	...
	case _ => valn
}
```
```scala
val code = 1
val result_match = code match {
	case 1 => "ONE"
	case 2 => "TWO"
	case _ => "OTHERS"
}
```

_ 表示通配，相当于Java中switch语句的default关键字。

##	求值策略
*	Call By Value 对函数实参求值，且仅求值一次
* 	Call By Name	函数实参每次在函数体内被用到时都会求值

Scala默认使用Call By Value	
如果函数形参类型以 => 开头，那么会使用Call By Name		
	
def foo(x: Int) = x //call by value		
def foo(x: => Int) = x //call by name	

call by name在调用时不会解析入参中的表达式，call by value则相反。

##	函数
Scala函数特性：	
1.	把函数作为实参传递给另一个函数。	
2. 	把函数作为返回值。	
3. 	把函数赋值给变量。	
4. 	把函数存储在数据结构里。	
5. 	函数和普通变量一样，具有函数的类型。

##	函数类型
函数类型格式为 A => B ，表示一个接受类型A的参数，并返回类型B的函数。

例子： Int => String 是把整型映射为字符串的函数类型。

##	高阶函数
用函数作为形参或返回值的函数，称为高阶函数。

```scala
def operate(f: (Int, Int) => Int) = {
	f(4, 4)
}

def greeting() = (name: String) => {"hello" + " " + name}	//匿名函数
```

##	柯里化
柯里化函数，把具有多个参数的函数转换为一条函数链，每个节点上是单一参数。

例子：以下2个add函数定义是等价的

```scala
def add(x: Int, y: Int) = x + y	
def add(x: Int)(y: Int) = x + y //Scala柯里化语法
```

##	尾递归函数
尾递归函数中所有递归形式的调用都出现在函数的末尾。

当编译器检测到一个函数调用是尾递归的时候，它就覆盖当前的活动记录而不是在栈中去创建一个新的。（避免堆栈内存溢出）

例子：

```scala
@annotation.tailrec
def factorial(n: Int, m: Int): Int =
	if (n <= 0) m
	else factoria(n-1, m*n)
factorial(5, 1)
```
```scala
def sum(f: Int => Int)(a: Int)(b: Int): Int = {

    @annotation.tailrec
    def loop(n: Int, acc: Int): Int = {
      if (n > b) {
        println(s"n=${n}, acc=${acc}")
        acc
      } else {
        println(s"n=${n}, acc=${acc}")
        loop(n + 1, acc + f(n))
      }
    }

    loop(a, 0)
}
  
  sum(x => x)(1)(5)

//输出
n=1, acc=0
n=2, acc=1
n=3, acc=3
n=4, acc=6
n=5, acc=10
n=6, acc=15
```
