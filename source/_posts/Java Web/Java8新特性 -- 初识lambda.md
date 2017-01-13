---
title: Java8 -- 初识lambda
date: 2016-12-13
categories: Java Web
tags: 
- Java8
---

#	Java8新特性 -- lambda表达式

###	新引入概念

*	默认接口方法

JDK8之前，接口不能定义任何实现，现在可以添加默认方法允许为接口方法定义一个或多个默认实现。

```Java
public interface A {

    default void hello() {
        System.out.println("This is A .");
    }
    
}
```

*	函数式接口

函数式接口是指仅指定了一个抽象方法的接口，例如 Runnable 或者 Comparator。

接着上面默认接口方法，当一个接口仅定义了唯一一个方法，就称之为函数式接口，其功能由其唯一方法定义。 

可以在任意函数式接口上标注 @FunctionalInterface ，这样做有两个好处：	
1.	编译器会检查标注改注解的实体，检查它是否是只包含一个抽象方法的接口。
2.	javadoc 页面上也会包含一条说明，说明这个接口是一个函数式接口。

ps:	该注解并不要求强制使用，使用注解会让代码看上去更清楚。
 
 
##	lambda表达式是什么

lambda表达式本质是一个匿名方法，但仅限于实现由函数式接口定义的方法。因此，lambda表达式会产生一个匿名类，常被称为闭包。

lambda表达式是一段可以传递的代码，因此它可以被执行一次或多次。

函数式接口可以指定Object定义的任何公有方法，例如equals()，而不影响其作为“函数式接口”的状态。

##	lambda语法特征

1.	运算符 ->
2. 	没有方法名称
3. 	形参类型推导
4. 	返回类型推导


##	示例
```Java
public class Sample0 {

    class S1 implements Runnable {
        @Override
        public void run() {
            System.out.println("This is S1");
        }
    }


    public static void main(String[] args) {
        test1();

        test2();
    }


    static void test1() {
        Sample0 sample0 = new Sample0();
        S1 s1 = sample0.new S1();
        s1.run();

        Runnable abc = () -> System.out.println("This is lambda");
        abc.run();
    }

    static void test2() {
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(i);
        }

        list.forEach(System.out::println);
    }
}
```


```Java
public interface MyValue {

    double myValue();

}
```

```Java
public interface MyValue2 {

    double myValue(int i);

}
```

```Java
public class Sample1 {

    private static MyValue myValue;

    public static void main(String[] args) {
        myValue = () -> Math.random();

        MyValue _myValue = () -> 1.1;


        MyValue2 myValue2 = n -> n * 100;

        MyValue2 _myValue2 = (n) -> n * 100;
    }
}
```

##	变量捕获

lambda表达式和块lambda表达式作用域访问和普通作用域基本一致，但是，当lambda表达式使用其外层作用域内定义的局部变量时，会产生一种特殊的情况，成为变量捕获。这种情况下，lambda表达式只能使用实际上为final的局部变量，修改局部变量会移除其实质上的final状态(可以不显示声明final关键字)，从而使捕获该变量变得不合法。

lambda表达式可以使用和修改其调用类的实例变量，只是不能使用其外层作用域中的局部变量，除非该变量实质上是final。

```Java
public interface MyFunc {

    int func(int n);
}
```

```Java
public class Sample2 {

    public static void main(String[] args) {
        int num = 10;

        MyFunc myFunc = n -> {
            int res = n + num;
            return res;
        };

		//输出18
        System.out.println(myFunc.func(8));
		//非法 编译器检查错误
       // num = 9;
    }
}
```

##	方法引用

方法引用提供了一种引用方法而不执行方法的方式。该特性与lambda表达式相关，因为它也需要由兼容的函数式接口构成的目标类型上下文。

###	静态方法的方法引用

####	语法
ClassName::methodName

类名与方法名之间使用双冒号分隔开。::是JDK8新增的分隔符，专门用于此目的。

```Java
public interface IntPredicate {

    boolean test(int i);
}
```

```Java
public class MyIntPredicates {

    static boolean isPrime(int n) {
        if (n < 2)
            return false;

        for (int i = 2; i < n / i; i++) {
            if (n % i == 0)
                return false;
        }

        return true;
    }

    static boolean isEven(int n) {
        return n % 2 == 0;
    }

    static boolean isPositive(int n) {
        return n > 0;
    }
}
```

```Java
public class Sample1 {

    static boolean numTest(IntPredicate p, int v) {
        return p.test(v);
    }

    public static void main(String[] args) {
        boolean result;

        result = numTest(MyIntPredicates::isPrime, 17);

        result = numTest(MyIntPredicates::isEven, 12);

        result = numTest(MyIntPredicates::isPositive, 11);
    }
}

```

注意
```Java
result = numTest(MyIntPredicates::isPrime, 17);
```
这里将对静态方法 isPrime() 引用传递给了 numTest() 方法的第一个参数，这是可行的，因为 isPrime 与 IntPredicate 函数式接口兼容( 其中 isPrime 提供了 IntPredicate 的 test() 方法的实现)。

###	实例方法的方法引用

####	语法

objRef::methodName

语法与静态方法语法类似，只不过这里是对象引用，而不是类名。

```Java
public interface IntPredicate {

    boolean test(int i);
}
```

```Java
public class MyIntNum {

    private int v;

    MyIntNum(int x) {
        this.v = x;
    }

    int getNum() {
        return v;
    }

    boolean isFactor(int n) {
        return (v % n) == 0;
    }
}
```

```Java
public class Sample2 {

    public static void main(String[] args) {
        boolean result;

        MyIntNum myIntNum = new MyIntNum(12);

        MyIntNum myIntNum2 = new MyIntNum(16);

        IntPredicate ip = myIntNum::isFactor;
        result = ip.test(3);


        ip = myIntNum2::isFactor;
        result = ip.test(3);

    }
}
```

###	构造函数引用

####	语法
classname::new

```Java
public interface MyFunc {

    MyClass func(String s);
}
```

```Java
public class MyClass {

    MyClass(String str) {
        this.str = str;
    }

    MyClass() {
        this.str = "";
    }

    private String str;

    public String getStr() {
        return str;
    }

    public void setStr(String str) {
        this.str = str;
    }
}
```

```Java
public class Sample3 {

    public static void main(String[] args) {
        MyFunc myFunc = MyClass::new;

        MyClass mc = myFunc.func("test");

        System.out.println(mc.getStr());
    }
}
```

表达式 MyClass:new 创建了对 MyClass 构造函数的引用，并不针对哪个构造函数。后面的 func() 方法接受一个 String 类型的形参，所以被引用的构造函数是 MyClass(String str)。

###	预定义函数式接口

JDK8 中包含了新包 java.util.function，其中提供了一些预定义的函数式接口。现在 Java8 本身API使用了这些接口，任何一个 lambda 表达式都可以等价转换成现在所使用API中对应的函数式接口。


