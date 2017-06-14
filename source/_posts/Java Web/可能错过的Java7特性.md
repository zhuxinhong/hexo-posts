---
title: Java7 -- 可能错过的Java7特性
date: 2017-03-23
categories: Java Web
tags: 
- Java7
---

#	可能错过的Java7特性

*	异常改进
* 	文件IO
*  	equals、hashCode和CompareTo方法
*  	其他

##	异常改进

###	try-with-resources

Java7 提供了一个简单实用的代码格式

```Java
打开一个资源
try	{
	实用该资源
}
finally {
	关闭该资源
}
```
其中资源所属类必须实现了 AutoCloseable 接口。

void close() throws Exception

try-with-resources 语句最简形式如下

```Java
try (Resource res = ...) {
	使用 res
}
```
当 try 语句块退出时，会自动调用 res.close() 方法。

```Java
try (Scanner in = new Scanner(Paths.get(File.separator, "data", "word"))) {
    while (in.hasNext()) {                                                 
        System.out.println(in.next());                                     
    }                                                                      
}                                                                          
```
当代码块像往常一样退出或发生异常时，都会调用 in.close() 方法，就如同你之前使用 finally 一样。

###	忽略异常

无论何时使用输入或输出，在异常产生后如何关闭资源都是一个麻烦的问题。假设产生了一个 IOException ，接下来关闭资源事， close 方法又抛出了另一个异常。

实际在Java中，finally分支中抛出的异常会丢弃掉之前的异常。

try-with-resources 修正了这个行为。当 AutoCloseable 的 close 方法抛出异常时，原来的异常会被重新抛出，而调用 close 方法产生的异常会被捕获，并被标注为“被忽略”的异常。

###	捕获多个异常

假设多个异常的行为一致，允许将多个异常写在一个 catch 分支中：

```Java
try {
	
} catch (FileNotFoundException | UnknownHostException ex) {

} catch (IOException ex){

}
```

只有当捕获的异常均不是其他异常的子类时，才能用这种方法。

###	ReflectiveOperationException

在调用一个反射方法时，不得不捕获许多不相关的检查器异常。例如，创建一个类并调用它的 main 方法：

Class.forName(className).getMethod("main").invoke(null, new String[]{});

该语句可能会导致一个 ClassNotFoundException、NoSuchMethodException、IllegalAccessException 或者 InvocationTargetException。

Java7 修复了这一设计上的缺陷，引入了一个新的父类 ReflectiveOperationException, 这样就可以通过这一个异常来捕获其他所有子异常。

##	使用文件

文件处理上，Java7 的 Path 接口取代了 File 类。

### Path Paths

###	Files

Files 类可以快速实现一些常用的文件 IO 操作，比之前的读写文件编码简洁太多。例如：

####	读写文件

读取一个文件全部内容

```Java
byte[] bytes = Files.readAllBytes(path);
```

将文件内容读取为一个字符串

```Java
String content = new String(bytes, StandardCharsets.UTF_8);
```

按行读取文件

```Java
List<String> lines = Files.readAllLines(path);
```

将字符串写入文件

```Java
Files.write(path, "content".getBytes(StandardCharsets.UTF_8));
```

按行来写入文件

```Java
List<String> lines = Arrays.asList("abc", "xyz");
Files.write(path, lines);
```

将内容追加到文件

```Java
List<String> lines = Arrays.asList("abc", "xyz");
Files.write(path, lines, StandardOpenOption.APPEND);
```

默认情况下，Files 类的所有方法都使用 UTF-8 来读取或写入字符。

处理一般大小文本文件时，通常最简单方法是将文件内容作为一个字符串或者字符串列表进行处理。如果文件很大，或者是二进制文件，仍然可以使用熟悉的 stream 类或者 reader/writer 类。

```Java
InputStream in = Files.newInputStream(path);
OutputStream out = Files.newOutPutStream(path);
Reader reader = Files.newBufferedReader(path);
Writer writer = Files.newBufferedWriter(path);
```

这些方便的方法可以帮你省去处理 FileInputStream、FileOutputStream、BufferedReader 或者 BufferedWriter 时的工作。

如果希望将一个 InputStream 中的内容保存到一个文件中，可以使用

```Java
Files.copy(in, path);
```

相反，可以将某个文件内容复制到一个 OutputStream 中。

```Java
Files.copy(path, out);
```

####	创建文件和目录

创建目录				

```Java
Files.createDirectories(path);
```

创建空文件

```Java
Files.createFile(path);
```

文件已经存在会抛异常，对已有文件的检查和创建都属于原子操作，所以如果文件不存在，那么它一定会在其他线程之前创建该文件。

####	复制、移动和删除文件

复制

```Java
Files.copy(fromPath, toPath);
```

移动

```Java
Files.move(fromPath, toPath);
```

如果目标文件或目录存在，copy 或 move 方法会失败。如果希望覆盖目标文件，使用 REPLACE_EXISTING 选项。如果希望复制所有文件属性，使用 COPY_ATTRIBUTES 选项，也可同时提供这两个选项：

```Java
Files.copy(fromPath, toPath, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.COPY_ATTRIBUTES);
```

可指定使用原子方式来执行移动，这样要么移动操作成功完成，要么源文件依然存在：

```Java
Files.move(fromPath, toPath, StandardCopyOption.ATOMIC_MOVE);
```

删除

```Java
Files.delete(path);
```

如果文件不存在会抛异常，可能希望使用另一个方法：

```Java
boolean deleted = Files.deleteIfExists(path);
```

```
没有直接删除或复制非空目录的方法。要实现请参考 FileVisitor 接口的API文档。
```

##	equals、hashcode、和 compareTo 方法

###	安全的 Null 值相等测试

假设需要为下面的类实现 equals 方法：

```Java
public class Person {
	private String first;
	private String last;
	...
}
```

首先，你不得不将参数转换为一个 Person 对象：

```Java
public boolean equals(Object otherObject) {
	if (this == otherObject) return true;
	if (otherObject == null) return false;
	if (this.getClass() != otherObject.getClass()) return false;
	Person other = (Person) otherObject;
	...
}
```

Java7出现后，再不用担心 first 或者 last 可能为空，只需调用

```Java
return Obejcts.equals(a, b);
```
如果 a 和 b 都是 null，则返回 true；如果只有 a 为 null，则返回 false；其他情况则返回 a.equals(b)。

###	计算哈希码

```Java
return Objects.hash(first, last);
```

Objects.hash 只是会调用从 Java5 开始就存在的 Arrays.hash 方法。但他不是一个支持可变参数的方法。

###	toString()

String.valueOf(obj)，可以安全处理 null 对象。如果 obj 为 null，会返回字符串 “null”。如果不喜欢这种方式，可以调用 Objects.toString 方法，并提供一个用于 null 对象的值。例如，Objects.toString(obj, "");

```Java
    /**
     * Returns the result of calling {@code toString} on the first
     * argument if the first argument is not {@code null} and returns
     * the second argument otherwise.
     *
     * @param o an object
     * @param nullDefault string to return if the first argument is
     *        {@code null}
     * @return the result of calling {@code toString} on the first
     * argument if it is not {@code null} and the second argument
     * otherwise.
     * @see Objects#toString(Object)
     */
    public static String toString(Object o, String nullDefault) {
        return (o != null) ? o.toString() : nullDefault;
    }
```


###	比较数值类型对象

使用静态方法 Integer.compare 

```Java
int diff = Integer.compare(x, other.x);
if (diff != 0) return diff;
return Integer.compare(y, other.y); 
```

过去某些开发人员会使用 new Integer(x).compareTo(other.x)的方式，但这会创建两个会自动装箱/拆箱的整形对象。相比之下，静态方法 compare 使用的是 int 参数。

此外，Long、Short、Byte、和 Boolean 也都提供了各自的静态方法 compare。

```
Double 和 Float 中的静态方法 compare 从 Java1.2 开始就存在了。
```
##		其他

###	字符串转数字

```Java
double x = Double.parseDouble("+1.0");
int n = Integer.parseInt("+1");
```

在 Java7 之前，+1 一直不是一个有效整数。

现在这个问题已经在所有通过字符串来构造 int、long、short、byte 和 BigInteger 的方法中被修复了，还包括包装类、处理十六进制和八进制数输入的 decode 方法，一级生成包装类对象的 valueOf 方法。BigInteger(String) 也被修复了。

###	全局 Logger

```Java
    public static final Logger getGlobal() {
        // In order to break a cyclic dependence between the LogManager
        // and Logger static initializers causing deadlocks, the global
        // logger is created with a special constructor that does not
        // initialize its log manager.
        //
        // If an application calls Logger.getGlobal() before any logger
        // has been initialized, it is therefore possible that the
        // LogManager class has not been initialized yet, and therefore
        // Logger.global.manager will be null.
        //
        // In order to finish the initialization of the global logger, we
        // will therefore call LogManager.getLogManager() here.
        //
        // To prevent race conditions we also need to call
        // LogManager.getLogManager() unconditionally here.
        // Indeed we cannot rely on the observed value of global.manager,
        // because global.manager will become not null somewhere during
        // the initialization of LogManager.
        // If two threads are calling getGlobal() concurrently, one thread
        // will see global.manager null and call LogManager.getLogManager(),
        // but the other thread could come in at a time when global.manager
        // is already set although ensureLogManagerInitialized is not finished
        // yet...
        // Calling LogManager.getLogManager() unconditionally will fix that.

        LogManager.getLogManager();

        // Now the global LogManager should be initialized,
        // and the global logger should have been added to
        // it, unless we were called within the constructor of a LogManager
        // subclass installed as LogManager, in which case global.manager
        // would still be null, and global will be lazily initialized later on.

        return global;
    }
```

###	Null 检查

```Java
    /**
     * Checks that the specified object reference is not {@code null}. This
     * method is designed primarily for doing parameter validation in methods
     * and constructors, as demonstrated below:
     * <blockquote><pre>
     * public Foo(Bar bar) {
     *     this.bar = Objects.requireNonNull(bar);
     * }
     * </pre></blockquote>
     *
     * @param obj the object reference to check for nullity
     * @param <T> the type of the reference
     * @return {@code obj} if not {@code null}
     * @throws NullPointerException if {@code obj} is {@code null}
     */
    public static <T> T requireNonNull(T obj) {
        if (obj == null)
            throw new NullPointerException();
        return obj;
    }
```

```Java
    /**
     * Checks that the specified object reference is not {@code null} and
     * throws a customized {@link NullPointerException} if it is. This method
     * is designed primarily for doing parameter validation in methods and
     * constructors with multiple parameters, as demonstrated below:
     * <blockquote><pre>
     * public Foo(Bar bar, Baz baz) {
     *     this.bar = Objects.requireNonNull(bar, "bar must not be null");
     *     this.baz = Objects.requireNonNull(baz, "baz must not be null");
     * }
     * </pre></blockquote>
     *
     * @param obj     the object reference to check for nullity
     * @param message detail message to be used in the event that a {@code
     *                NullPointerException} is thrown
     * @param <T> the type of the reference
     * @return {@code obj} if not {@code null}
     * @throws NullPointerException if {@code obj} is {@code null}
     */
    public static <T> T requireNonNull(T obj, String message) {
        if (obj == null)
            throw new NullPointerException(message);
        return obj;
    }
```

###	ProcessBuilder

Java5 之前，Runtime.exec 方法是在 Java 中执行一个外部命令的唯一方式。Java5 添加了 ProcessBuilder 类，加强了对 OS 进程的控制，尤其是通过 ProcessBuilder，你可以改变工作目录。

Java7 添加了一些新方法，使得开发人员能方便地将进程的标准输入、输出及错误重定向到文件中。

```Java
   public Redirect redirectInput() {
        return (redirects == null) ? Redirect.PIPE : redirects[0];
   }
   
   public Redirect redirectOutput() {
        return (redirects == null) ? Redirect.PIPE : redirects[1];
   }
   
   public ProcessBuilder redirectInput(File file) {
        return redirectInput(Redirect.from(file));
   }
   
   public ProcessBuilder redirectOutput(File file) {
        return redirectOutput(Redirect.to(file));
   }
   
   public Redirect redirectError() {
        return (redirects == null) ? Redirect.PIPE : redirects[2];
   }
   
   public ProcessBuilder redirectError(File file) {
        return redirectError(Redirect.to(file));
   }

	...
```

###	URLClassLoader

Java7 只是添加了一个用来关闭该类加载器的 close 方法，URLClassLoader 类现在实现了 AutoCloseable 接口，因此可使用 try-with-resources 语句来避免可能出现的资源泄露。

```Java
public class URLClassLoader extends SecureClassLoader implements Closeable {
	...
}
```

###	BitSet

BitSet 是一个表示 bit 序列的整数集合。该集合可以根据整数 i 来设置第 i 个 bit 位的值。这使得集合操作非常搞笑。求并集、交集、补集只不过是简单的按位或、与、非运算。

Java7 添加了一些构造 BitSet 的方法：

```Java
    public static BitSet valueOf(byte[] bytes) {
        return BitSet.valueOf(ByteBuffer.wrap(bytes));
    }
    
    public static BitSet valueOf(ByteBuffer bb) {
        bb = bb.slice().order(ByteOrder.LITTLE_ENDIAN);
        int n;
        for (n = bb.remaining(); n > 0 && bb.get(n - 1) == 0; n--)
            ;
        long[] words = new long[(n + 7) / 8];
        bb.limit(n);
        int i = 0;
        while (bb.remaining() >= 8)
            words[i++] = bb.getLong();
        for (int remaining = bb.remaining(), j = 0; j < remaining; j++)
            words[i] |= (bb.get() & 0xffL) << (8 * j);
        return new BitSet(words);
    }
    
    public static BitSet valueOf(long[] longs) {
        int n;
        for (n = longs.length; n > 0 && longs[n - 1] == 0; n--)
            ;
        return new BitSet(Arrays.copyOf(longs, n));
    }
    
    public static BitSet valueOf(LongBuffer lb) {
        lb = lb.slice();
        int n;
        for (n = lb.remaining(); n > 0 && lb.get(n - 1) == 0; n--)
            ;
        long[] words = new long[n];
        lb.get(words);
        return new BitSet(words);
    }
```