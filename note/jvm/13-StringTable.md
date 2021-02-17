# String的基本特性

## 基本概念

- String：字符串，使用一对” “引起来表示。
  - String s1="atguigu";//字面量的定义方式
  - String s2 = new String ("hello")
- String声明为final的，不可被继承
- String实现了 Serializable 接口：表示字符串是支持序列化的
- 实现了 Comparable接口：表示 String可以比较大小
- String在jdk8及以前内部定义了 final char[] value用于存储字符串数据。jdk9时改为byte[]

## String在jdk9中存储结构变更

> [JEP 254: Compact Strings](http://openjdk.java.net/jeps/254)
>
> ```html
> Motivation
> The current implementation of the String class stores characters in a char array, using two bytes (sixteen bits) for each character. Data gathered from many different applications indicates that Strings are a major component of heap usage and, moreover, that most String objects contain only Latin-1 characters. Such characters require only one byte of storage, hence half of the space in the internal char arrays of such String objects is going unused.
> 
> Description
> We propose to change the internal representation of the String class from a UTF-16 char array to a byte array plus an encoding-flag field. The new String class will store characters encoded either as ISO-8859-1/Latin-1 (one byte per character), or as UTF-16 (two bytes per character), based upon the contents of the String. The encoding flag will indicate which encoding is used.
> ```

结论：String再也不用char[]来存储了，改成byte[]加上编码标记，节约了一些空间。

```java
public final class String implements Serializable, Comparable<String>, CharSequence {
    @Stable
    private final byte[] value;
}
```

那StringBuffer和StringBuilder是否仍无动于衷那？

```html
String-related classes such as AbstractStringBuilder, StringBuilder, and StringBuffer will be updated to use the same representation, as will the HotSpot VM's intrinsic(固有的、内置的) String operations.
```

## String的基本特性一

- String：代表不可变的字符序列。简称：不可变性。
  - 当对字符串重新赋值时，需要重写指定内存区域赋值，不能使用原有的value进行赋值
  - 当对现有的字符串进行连接操作时，也需要重新指定内存区域赋值不能使用原有的value进行赋值。
  - 当调用 String的replace()方法修改指定字符或字符串时，也需要重新指定内存区域赋值，不能使用原有的value进行赋值。
- 通过字面量的方式（区别于new）给一个字符串赋值，此时的字符串值声明在字符串常量池中。

### 代码验证-字符定义在常量池中

```java
public class StringTest1 {
    @Test
    public void test1() {
        String s1 = "abc";//字面量定义的方式，"abc"存储在字符串常量池中
        String s2 = "abc";//使用字面量定义的字符串，都是使用的是字符串常量池中的。
        s1 = "hello";// 注释掉为true，反之为false

        System.out.println(s1 == s2);//判断地址：true  --> false

        System.out.println(s1);//
        System.out.println(s2);//abc

    }

    @Test
    public void test2() {
        String s1 = "abc";
        String s2 = "abc";
        s2 += "def";
        System.out.println(s2);//abcdef
        System.out.println(s1);//abc
    }

    @Test
    public void test3() {
        String s1 = "abc";
        String s2 = s1.replace('a', 'm');
        System.out.println(s1);//abc
        System.out.println(s2);//mbc
    }
}
```

### 代码验证-字符串值传递

```java
/**
 * @author shkstart  shkstart@126.com
 * @create 2020  23:44
 */
public class StringExer {
    String str = new String("good");
    char[] ch = {'t', 'e', 's', 't'};

    public void change(String str, char ch[]) {
        str = "test ok";
        ch[0] = 'b';
    }

    public static void main(String[] args) {
        StringExer ex = new StringExer();
        ex.change(ex.str, ex.ch);
        System.out.println(ex.str);//good
        System.out.println(ex.ch);//best
    }

}
```

## String的基本特性二

- **字符串常量池中是不会存储相同内容的字符串的。**
- String的 String Pool是一个固定大小的 Hashtable，默认值大小长度是1009。如果放进 String Pool的 String非常多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用String.intern时性能会大幅下降。
- 使用`-XX:StringTableSize`可设置 StringTable的长度
- 在jdk6中 StringTable是固定的，就是**1009**的长度，所以如果常量池中的字符串过多就会导致效率下降很快。 StringTableSize设置没有要求。
- 在jdk7中， StringTable的长度默认值是**60013**，1009是可设置的最小值。

### 代码验证-StringTableSize

> JDK8

代码

```java
    public static void main(String[] args) {
        //测试StringTableSize参数
        System.out.println("我来打个酱油");
        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

结果

```shell
C:\Users\Qiongchao>jps
2272 org.eclipse.equinox.launcher_1.6.0.v20200915-1508.jar
21140 RemoteMavenServer36
15000 XMLServerLauncher
8520 Launcher
21484 Jps
7980 StringTest2
9484

C:\Users\Qiongchao>jinfo -flag StringTableSize 7980
-XX:StringTableSize=60013
```

修改`-XX:StringTableSize=1008`后报错

```shell
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
StringTable size of 1008 is invalid; must be between 1009 and 2305843009213693951
```



### 代码验证-StringTableSize大小影响

生成字符代码

```java
import java.io.FileWriter;
import java.io.IOException;

/**
 * 产生10万个长度不超过10的字符串，包含a-z,A-Z
 * @author shkstart  shkstart@126.com
 * @create 2020  23:58
 */
public class GenerateString {
    public static void main(String[] args) throws IOException {
        FileWriter fw =  new FileWriter("words.txt");

        for (int i = 0; i < 100000; i++) {
            //1 - 10
           int length = (int)(Math.random() * (10 - 1 + 1) + 1);
            fw.write(getString(length) + "\n");
        }

        fw.close();
    }

    public static String getString(int length){
        String str = "";
        for (int i = 0; i < length; i++) {
            //65 - 90, 97-122
            int num = (int)(Math.random() * (90 - 65 + 1) + 65) + (int)(Math.random() * 2) * 32;
            str += (char)num;
        }
        return str;
    }
}
```

执行测试代码

> `StringTableSize=1009` 花费192ms
>
> `StringTableSize=100009` 花费59ms

```java
/**
 *  -XX:StringTableSize=1009 
 *  -XX:StringTableSize=100009
 * 
 * @author shkstart  shkstart@126.com
 * @create 2020  23:53
 */
public class StringTest2 {
    public static void main(String[] args) {

        BufferedReader br = null;
        try {
            br = new BufferedReader(new FileReader("words.txt"));
            long start = System.currentTimeMillis();
            String data;
            while((data = br.readLine()) != null){
                data.intern(); //如果字符串常量池中没有对应data的字符串的话，则在常量池中生成
            }

            long end = System.currentTimeMillis();

            System.out.println("花费的时间为：" + (end - start));//1009:192ms  100009:59ms
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if(br != null){
                try {
                    br.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }

            }
        }
    }
}
```

结果：

- StringTableSize的大小影响着常量池的效率。（类似于HashMap的大小的影响。）



# String的内存分配

- 在Java语言中有8种基本数据类型和一种比较特殊的类型 String。这些类型为了使它们在运行过程中速度更快、更节省内存，都提供了一种常量池的概念。
- 常量池就类似一个Java系统级别提供的缓存。8种基本数据类型的常量池都是系统协调的， String类型的常量池比较特殊。它的主要使用方法有两种。
  - 直接使用双引号声明出来的 String对象会直接存储在常量池中。
    - 比如：String info = "atguigu.com";
  - 如果不是用双引号声明的 String对象，可以使用 String提供的intern()方法。这个后面重点谈。

- Java6及以前，字符串常量池存放在永久代。
- Java7中 oracle的工程师对字符串池的逻辑做了很大的改变，即**将字符串常量池的位置调整到Java堆内**。
  - 所有的字符串都保存在堆（Heap）中，和其他普通对象一样，这样可以让你在进行调优应用时仅需要调整堆大小就可以了。
  - 字符串常量池概念原本使用得比较多，但是这个改动使得我们有足够的理由让我们重新考虑在Java7中使用String.intern()。
- Java8元空间，字符串常量在堆中。

![image-20210204172335410](https://gitee.com/clancy/images/raw/master/img/image-20210204172335410.png)

![image-20210204172402585](https://gitee.com/clancy/images/raw/master/img/image-20210204172402585.png)

![image-20210204172433359](https://gitee.com/clancy/images/raw/master/img/image-20210204172433359.png)

## StringTable为什么要调整？

### [官网说明](https://www.oracle.com/java/technologies/javase/jdk7-relnotes.html#jdkchanges)

```html
Area: HotSpot

Synopsis: Classfiles with version number 51 are exclusively verified using the type-checking verifier, and thus the methods must have StackMapTable attributes when appropriate. For classfiles with version 50, the HotSpot JVM would (and continues to) failover to the type-inferencing verifier if the stackmaps in the file were missing or incorrect. This failover behavior does not occur for classfiles with version 51 (the default version for JDK7).

Any tool that modifies bytecode in a version 51 classfile must be sure to update the stackmap information to be consistent with the bytecode in order to pass verification.

RFE: 6693236
```

### 总结：

1. permSize默认比较小
2. 永久代垃圾回收频率低。

## 永久代（JDK7+的堆）OOM

### 代码

```java
/**
 * jdk6中：
 * -XX:PermSize=6m -XX:MaxPermSize=6m -Xms6m -Xmx6m
 *
 * jdk8中：
 * -XX:MetaspaceSize=6m -XX:MaxMetaspaceSize=6m -Xms6m -Xmx6m
 * 
 * @author shkstart  shkstart@126.com
 * @create 2020  0:36
 */
public class StringTest3 {
    public static void main(String[] args) {
        //使用Set保持着常量池引用，避免full gc回收常量池行为
        Set<String> set = new HashSet<String>();
        //在short可以取值的范围内足以让6MB的PermSize或heap产生OOM了。
        short i = 0;
        while(true){
            set.add(String.valueOf(i++).intern());
        }
    }
}
```

### 结果

> 使用JDK8演示结果

```shell
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.lang.Integer.toString(Integer.java:401)
	at java.lang.String.valueOf(String.java:3099)
	at com.chaocode.jvm.atguigu.chapter13.java.StringTest3.main(StringTest3.java:23)
```



# String的基本操作

## 例子一

Java语言规范里要求完全相同的字符串字面量，应该包含同样的Unicode字符序列（包含同一份码点序列的常量），并且必须是指向同一个String类实例。

```java
public static void main(String[] args) {
        System.out.println();//2954
        System.out.println("1");//2955
        System.out.println("2");//2956
        System.out.println("3");
        System.out.println("4");
        System.out.println("5");
        System.out.println("6");
        System.out.println("7");
        System.out.println("8");
        System.out.println("9");
        System.out.println("10");//2964
        //如下的字符串"1" 到 "10"不会再次加载
        System.out.println("1");//2965
        System.out.println("2");//2965
        System.out.println("3");
        System.out.println("4");
        System.out.println("5");
        System.out.println("6");
        System.out.println("7");
        System.out.println("8");
        System.out.println("9");
        System.out.println("10");//2965
    }
```

> 通过IDEA的memory查看String的对象数，发现打印后面的1~10时，String的对象没有再增加过。因为使用的是前面字面量在常量池中生成的字符串。
>
> [官网字面量说明](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.5)



## 例子二

```java
class Memory {
    public static void main(String[] args) {//line 1
        int i = 1;//line 2
        Object obj = new Object();//line 3
        Memory mem = new Memory();//line 4
        mem.foo(obj);//line 5
    }//line 9

    private void foo(Object param) {//line 6
        String str = param.toString();//line 7
        System.out.println(str);
    }//line 8
}
```

操作图

![image-20210213215448378](https://gitee.com/clancy/images/raw/master/img/20210213215456.png)





# 字符串拼接操作

## 概念

1. 常量常量的拼接结果在常量池，原理是编译期优化。
2. 常量池中不会存在相同内容的常量。
3. 只要其中有一个是变量，结果就在堆中。变量拼接的原理是new了个StringBuilder进行append操作，最后调用toString()方法。
4. 如果拼接的结果调用intern()方法，则主动将常量池中还没有的字符串对象放入池中，并返回此对象地址。

## 代码测试-字符拼接

### 代码

```java
    @Test
    public void test1(){
        String s1 = "a" + "b" + "c";//编译期优化：等同于"abc"
        String s2 = "abc"; //"abc"一定是放在字符串常量池中，将此地址赋给s2
        /*
         * 最终.java编译成.class,再执行.class
         * String s1 = "abc";
         * String s2 = "abc"
         */
        System.out.println(s1 == s2); //true
        System.out.println(s1.equals(s2)); //true
    }


```

### 字节码

```shell
 0 ldc #2 <abc> // s1
 2 astore_1
 3 ldc #2 <abc> // s2
 5 astore_2
……
……
```



## 代码测试-字符变量拼接

### 代码1

```java
    @Test
    public void test2(){
        String s1 = "javaEE";
        String s2 = "hadoop";

        String s3 = "javaEEhadoop";
        String s4 = "javaEE" + "hadoop";//编译期优化为"javaEEhadoop"
        //如果拼接符号的前后出现了变量，则相当于在堆空间中new String()，具体的内容为拼接的结果：javaEEhadoop
        String s5 = s1 + "hadoop";
        String s6 = "javaEE" + s2;
        String s7 = s1 + s2;

        System.out.println(s3 == s4);//true
        System.out.println(s3 == s5);//false
        System.out.println(s3 == s6);//false
        System.out.println(s3 == s7);//false
        System.out.println(s5 == s6);//false
        System.out.println(s5 == s7);//false
        System.out.println(s6 == s7);//false
        //intern():判断字符串常量池中是否存在javaEEhadoop值，如果存在，则返回常量池中javaEEhadoop的地址；
        //如果字符串常量池中不存在javaEEhadoop，则在常量池中加载一份javaEEhadoop，并返回次对象的地址。
        String s8 = s6.intern();
        System.out.println(s3 == s8);//true
    }
```

### 字符变量拼接底层原理

```shell
……
 13 new #9 <java/lang/StringBuilder> // 拼接s5
 16 dup
 17 invokespecial #10 <java/lang/StringBuilder.<init>>
 20 aload_1 // 加载本地变量表中s1
 21 invokevirtual #11 <java/lang/StringBuilder.append>
 24 ldc #7 <hadoop> // 加载常量池的字符串 hadoop
 26 invokevirtual #11 <java/lang/StringBuilder.append>
 29 invokevirtual #12 <java/lang/StringBuilder.toString>
 32 astore 5
 ……
```

### 代码2

```java
    @Test
    public void test3(){
        String s1 = "a";
        String s2 = "b";
        String s3 = "ab";
        /*
        如下的s1 + s2 的执行细节：(变量s是我临时定义的）
        ① StringBuilder s = new StringBuilder();
        ② s.append("a")
        ③ s.append("b")
        ④ s.toString()  --> 约等于 new String("ab")

        补充：在jdk5.0之后使用的是StringBuilder,在jdk5.0之前使用的是StringBuffer
         */
        String s4 = s1 + s2;//
        System.out.println(s3 == s4);//false
    }
```



## 代码测试-字符常量拼接

### 代码1

```java
    /*
    1. 字符串拼接操作不一定使用的是StringBuilder!
       如果拼接符号左右两边都是字符串常量或常量引用，则仍然使用编译期优化，即非StringBuilder的方式。
    2. 针对于final修饰类、方法、基本数据类型、引用数据类型的量的结构时，能使用上final的时候建议使用上。
     */
    @Test
    public void test4(){
        final String s1 = "a";
        final String s2 = "b";
        String s3 = "ab";
        String s4 = s1 + s2;
        System.out.println(s3 == s4);//true
    }
```

### 字节码

```shell
 0 ldc #14 <a>
 2 astore_1
 3 ldc #15 <b>
 5 astore_2
 6 ldc #16 <ab>
 8 astore_3
 9 ldc #16 <ab>
11 astore 4
13 getstatic #3 <java/lang/System.out>
16 aload_3
17 aload 4
19 if_acmpne 26 (+7)
22 iconst_1
23 goto 27 (+4)
26 iconst_0
27 invokevirtual #4 <java/io/PrintStream.println>
30 return
```

### 代码2

```java
    //练习：
    @Test
    public void test5(){
        String s1 = "javaEEhadoop";
        String s2 = "javaEE";
        String s3 = s2 + "hadoop";
        System.out.println(s1 == s3);//false

        final String s4 = "javaEE";//s4:常量
        String s5 = s4 + "hadoop";
        System.out.println(s1 == s5);//true

    }
```

## 代码测试-字符串拼接优化

```java
    /*
    体会执行效率：通过StringBuilder的append()的方式添加字符串的效率要远高于使用String的字符串拼接方式！
    详情：① StringBuilder的append()的方式：自始至终中只创建过一个StringBuilder的对象
          使用String的字符串拼接方式：创建过多个StringBuilder和String的对象
         ② 使用String的字符串拼接方式：内存中由于创建了较多的StringBuilder和String的对象，内存占用更大；如果进行GC，需要花费额外的时间。

     改进的空间：在实际开发中，如果基本确定要前前后后添加的字符串长度不高于某个限定值highLevel的情况下,建议使用构造器实例化：
               StringBuilder s = new StringBuilder(highLevel);//new char[highLevel]
     */
    @Test
    public void test6(){

        long start = System.currentTimeMillis();

//        method1(100000);//4014
        method2(100000);//7

        long end = System.currentTimeMillis();

        System.out.println("花费的时间为：" + (end - start));
    }

    public void method1(int highLevel){
        String src = "";
        for(int i = 0;i < highLevel;i++){
            src = src + "a";//每次循环都会创建一个StringBuilder、String
        }
//        System.out.println(src);

    }

    public void method2(int highLevel){
        //只需要创建一个StringBuilder
        StringBuilder src = new StringBuilder();
        for (int i = 0; i < highLevel; i++) {
            src.append("a");
        }
//        System.out.println(src);
    }
```

### method1字节码

```shell
 0 ldc #23
 2 astore_2
 3 iconst_0
 4 istore_3
 5 iload_3
 6 iload_1
 7 if_icmpge 36 (+29)
10 new #9 <java/lang/StringBuilder>
13 dup
14 invokespecial #10 <java/lang/StringBuilder.<init>>
17 aload_2
18 invokevirtual #11 <java/lang/StringBuilder.append>
21 ldc #14 <a>
23 invokevirtual #11 <java/lang/StringBuilder.append>
26 invokevirtual #12 <java/lang/StringBuilder.toString>
29 astore_2
30 iinc 3 by 1
33 goto 5 (-28)
36 return
```

### methond2字节码

```java
 0 new #9 <java/lang/StringBuilder>
 3 dup
 4 invokespecial #10 <java/lang/StringBuilder.<init>>
 7 astore_2
 8 iconst_0
 9 istore_3
10 iload_3
11 iload_1
12 if_icmpge 28 (+16)
15 aload_2
16 ldc #14 <a>
18 invokevirtual #11 <java/lang/StringBuilder.append>
21 pop
22 iinc 3 by 1
25 goto 10 (-15)
28 return
```



# intern()的使用

## intern()方法

```java
    /**
     * Returns a canonical representation for the string object.
     * <p>
     * A pool of strings, initially empty, is maintained privately by the
     * class {@code String}.
     * <p>
     * When the intern method is invoked, if the pool already contains a
     * string equal to this {@code String} object as determined by
     * the {@link #equals(Object)} method, then the string from the pool is
     * returned. Otherwise, this {@code String} object is added to the
     * pool and a reference to this {@code String} object is returned.
     * <p>
     * It follows that for any two strings {@code s} and {@code t},
     * {@code s.intern() == t.intern()} is {@code true}
     * if and only if {@code s.equals(t)} is {@code true}.
     * <p>
     * All literal strings and string-valued constant expressions are
     * interned. String literals are defined in section 3.10.5 of the
     * <cite>The Java&trade; Language Specification</cite>.
     *
     * @return  a string that has the same contents as this string, but is
     *          guaranteed to be from a pool of unique strings.
     */
    public native String intern();
```

## intern()的使用

如果不是用双引号声明的 String对象，可以使用 String提供的 intern方法：intern方法会从字符串常量池中査询当前字符串是否存在，若不存在就会将当前字符串放入常量池中。
比如：String myInfo = new String("I love atguigu").intern();
也就是说，如果在任意字符串上调用 String.intern方法，那么其返回结果所指向的那个类容例，必须和直接以常量形式出现的字符串实例完全相同。因此，下列表达式的值必定是true
`("a" + "b" + "c")intern() == "abc"`
通俗点讲， Interned String就是确保字符串在内存里只有一份拷贝，这样可以节约内存空间，加快字符串操作任务的执行速度。注意，这个值会被存放在字符串内部池（ String Intern Pool）。

## 面试题

```java
/**
 * 题目：
 * new String("ab")会创建几个对象？看字节码，就知道是两个。
 *     一个对象是：new关键字在堆空间创建的
 *     另一个对象是：字符串常量池中的对象"ab"。 字节码指令：ldc
 *
 *
 * 思考：
 * new String("a") + new String("b")呢？
 *  对象1：new StringBuilder()
 *  对象2： new String("a")
 *  对象3： 常量池中的"a"
 *  对象4： new String("b")
 *  对象5： 常量池中的"b"
 *
 *  深入剖析： StringBuilder的toString():
 *      对象6 ：new String("ab")
 *       强调一下，toString()的调用，在字符串常量池中，没有生成"ab"
 *
 * @author shkstart  shkstart@126.com
 * @create 2020  20:38
 */
public class StringNewTest {
    public static void main(String[] args) {
//        String str = new String("ab");

        String str = new String("a") + new String("b");
    }
}
```

`new String("ab")`字节码

```shell
 0 new #2 <java/lang/String> // 堆中new了个String的对象
 3 dup
 4 ldc #3 <ab> // 常量池中一个
 6 invokespecial #4 <java/lang/String.<init>>
 9 astore_1
10 return
```

`new String("a") + new String("b")`字节码

```shell
 0 new #2 <java/lang/StringBuilder>
 3 dup
 4 invokespecial #3 <java/lang/StringBuilder.<init>>
 7 new #4 <java/lang/String>
10 dup
11 ldc #5 <a>
13 invokespecial #6 <java/lang/String.<init>>
16 invokevirtual #7 <java/lang/StringBuilder.append>
19 new #4 <java/lang/String>
22 dup
23 ldc #8 <b>
25 invokespecial #6 <java/lang/String.<init>>
28 invokevirtual #7 <java/lang/StringBuilder.append>
31 invokevirtual #9 <java/lang/StringBuilder.toString>
34 astore_1
35 return
```



## jdk6 vs jdk7/8

### 例子1

```java
/**
 * 如何保证变量s指向的是字符串常量池中的数据呢？
 * 有两种方式：
 * 方式一： String s = "shkstart";//字面量定义的方式
 * 方式二： 调用intern()
 *         String s = new String("shkstart").intern();
 *         String s = new StringBuilder("shkstart").toString().intern();
 *
 * @author shkstart  shkstart@126.com
 * @create 2020  18:49
 */
public class StringIntern {
    public static void main(String[] args) {

        String s = new String("1");
        s.intern();//调用此方法之前，new String("1")时，字符串常量池中会创建一个"1"
        String s2 = "1";
        System.out.println(s == s2);//jdk6：false   jdk7/8：false


        String s3 = new String("1") + new String("1");//s3变量记录的地址为：new String("11")
        //执行完上一行代码以后，字符串常量池中，是否存在"11"呢？答案：不存在！！
        s3.intern();//在字符串常量池中生成"11"。如何理解：jdk6:创建了一个新的对象"11",也就有新的地址。
                                            //         jdk7:此时常量中并没有创建"11",而是创建一个指向堆空间中new String("11")的地址
        String s4 = "11";//s4变量记录的地址：使用的是上一行代码代码执行时，在常量池中生成的"11"的地址
        System.out.println(s3 == s4);//jdk6：false  jdk7/8：true
    }
}
```

字节码

```shell
 0 new #2 <java/lang/String>
 3 dup
 4 ldc #3 <1>
 6 invokespecial #4 <java/lang/String.<init>>
 9 astore_1
10 aload_1
11 invokevirtual #5 <java/lang/String.intern>
14 pop
15 ldc #3 <1>
17 astore_2
18 getstatic #6 <java/lang/System.out>
21 aload_1
22 aload_2
23 if_acmpne 30 (+7)
26 iconst_1
27 goto 31 (+4)
30 iconst_0
31 invokevirtual #7 <java/io/PrintStream.println>
34 new #8 <java/lang/StringBuilder>
37 dup
38 invokespecial #9 <java/lang/StringBuilder.<init>>
41 new #2 <java/lang/String>
44 dup
45 ldc #3 <1>
47 invokespecial #4 <java/lang/String.<init>>
50 invokevirtual #10 <java/lang/StringBuilder.append>
53 new #2 <java/lang/String>
56 dup
57 ldc #3 <1>
59 invokespecial #4 <java/lang/String.<init>>
62 invokevirtual #10 <java/lang/StringBuilder.append>
65 invokevirtual #11 <java/lang/StringBuilder.toString>
68 astore_3
69 aload_3
70 invokevirtual #5 <java/lang/String.intern>
73 pop
74 ldc #12 <11>
76 astore 4
78 getstatic #6 <java/lang/System.out>
81 aload_3
82 aload 4
84 if_acmpne 91 (+7)
87 iconst_1
88 goto 92 (+4)
91 iconst_0
92 invokevirtual #7 <java/io/PrintStream.println>
95 return
```



![image-20210213225328059](https://gitee.com/clancy/images/raw/master/img/20210213225329.png)

jdk7调用`intern()`方法时，的使用指针指向了该对象，而不是在常量池中再创建一个相同的字符串常量。（节省内存空间）

![image-20210213225349243](https://gitee.com/clancy/images/raw/master/img/20210213225350.png)

### 例子2

```java
public class StringIntern1 {
    public static void main(String[] args) {
        //StringIntern.java中练习的拓展：
        String s3 = new String("1") + new String("1");//new String("11")
        //执行完上一行代码以后，字符串常量池中，是否存在"11"呢？答案：不存在！！
        String s4 = "11";//在字符串常量池中生成对象"11"
        String s5 = s3.intern();
        System.out.println(s3 == s4);//false
        System.out.println(s5 == s4);//true
    }
}
```

总结String的intern()的使用：

- jdk1.6中，将这个字符串对象尝试放入串池。
  - 如果串池中有，则并不会放入。返回已有的串池中的对象的地址
  - 如果没有，会把**此对象复制一份**，放入串池，并返回串池中的对象地址
- jdk1.7起，将这个字符串对象尝试放入串池。
  - 如果串池中有，则并不会放入。返回已有的串池中的对象的地址
  - 如果没有，则会**把对象的引用地址复制一份**，放入串池，并返回串池中的引用地址

### 例子3

```java
public class StringExer1 {
    public static void main(String[] args) {
//        String x = "ab";
        String s = new String("a") + new String("b");//new String("ab")
        //在上一行代码执行完以后，字符串常量池中并没有"ab"

        String s2 = s.intern();//jdk6中：在串池中创建一个字符串"ab"
                               //jdk8中：串池中没有创建字符串"ab",而是创建一个引用，指向new String("ab")，将此引用返回

        System.out.println(s2 == "ab");//jdk6:true  jdk8:true
        System.out.println(s == "ab");//jdk6:false  jdk8:true
    }
}
```

jdk6内存图

![image-20210213230242767](https://gitee.com/clancy/images/raw/master/img/20210213230244.png)

jdk7+内存图

![image-20210213230332002](https://gitee.com/clancy/images/raw/master/img/20210213230333.png)

先在常量池中定义字符串的结果

![image-20210213230458276](https://gitee.com/clancy/images/raw/master/img/20210213230500.png)

### 例子4

```java
public class StringExer2 {
    public static void main(String[] args) {
        String s1 = new String("ab");//执行完以后，会在字符串常量池中会生成"ab"
//        String s1 = new String("a") + new String("b");////执行完以后，不会在字符串常量池中会生成"ab"
        s1.intern();
        String s2 = "ab";
        System.out.println(s1 == s2);
    }
}
```

## 使用intern()测试执行效率（空间方面）

```java
public class StringIntern2 {
    static final int MAX_COUNT = 1000 * 10000;
    static final String[] arr = new String[MAX_COUNT];

    public static void main(String[] args) {
        Integer[] data = new Integer[]{1,2,3,4,5,6,7,8,9,10};

        long start = System.currentTimeMillis();
        for (int i = 0; i < MAX_COUNT; i++) {
            // 通过JvisualVM或jProfiler查看String的实例数相差也比较大。
//            arr[i] = new String(String.valueOf(data[i % data.length])); // 3558 ms
            arr[i] = new String(String.valueOf(data[i % data.length])).intern(); // 1390 ms

        }
        long end = System.currentTimeMillis();
        System.out.println("花费的时间为：" + (end - start));

        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.gc();
    }
}
```

### 总结：

大的网站平台，需要内存中存储大量的字符串。比如社交网站，很多人都存储：北京市、海淀区等信息。这时候如果字符串都调用intern()方法，就会明显降低内存的大小。



# StringTable的垃圾回收

代码

```java
/**
 * String的垃圾回收:
 * -Xms15m -Xmx15m -XX:+PrintStringTableStatistics -XX:+PrintGCDetails
 *
 * @author shkstart  shkstart@126.com
 * @create 2020  21:27
 */
public class StringGCTest {
    public static void main(String[] args) {
//        int time = 100;
        int time = 100000;
        for (int j = 0; j < time; j++) {
            String.valueOf(j).intern();
        }
    }
}
```

空方法体

```shell
Heap
 PSYoungGen      total 4608K, used 3198K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
  eden space 4096K, 78% used [0x00000000ffb00000,0x00000000ffe1fa78,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
  to   space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
 ParOldGen       total 11264K, used 0K [0x00000000ff000000, 0x00000000ffb00000, 0x00000000ffb00000)
  object space 11264K, 0% used [0x00000000ff000000,0x00000000ff000000,0x00000000ffb00000)
 Metaspace       used 3262K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 353K, capacity 388K, committed 512K, reserved 1048576K
SymbolTable statistics:
Number of buckets       :     20011 =    160088 bytes, avg   8.000
Number of entries       :     13332 =    319968 bytes, avg  24.000
Number of literals      :     13332 =    572088 bytes, avg  42.911
Total footprint         :           =   1052144 bytes
Average bucket size     :     0.666
Variance of bucket size :     0.667
Std. dev. of bucket size:     0.817
Maximum bucket size     :         6
StringTable statistics:
Number of buckets       :     60013 =    480104 bytes, avg   8.000
Number of entries       :      1743 =     41832 bytes, avg  24.000
Number of literals      :      1743 =    156392 bytes, avg  89.726
Total footprint         :           =    678328 bytes
Average bucket size     :     0.029
Variance of bucket size :     0.029
Std. dev. of bucket size:     0.171
Maximum bucket size     :         2
```

time = 100

比空方法体多大致100个

```shell
Heap
 PSYoungGen      total 4608K, used 3198K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
  eden space 4096K, 78% used [0x00000000ffb00000,0x00000000ffe1fa78,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
  to   space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
 ParOldGen       total 11264K, used 0K [0x00000000ff000000, 0x00000000ffb00000, 0x00000000ffb00000)
  object space 11264K, 0% used [0x00000000ff000000,0x00000000ff000000,0x00000000ffb00000)
 Metaspace       used 3259K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 353K, capacity 388K, committed 512K, reserved 1048576K
SymbolTable statistics:
Number of buckets       :     20011 =    160088 bytes, avg   8.000
Number of entries       :     13334 =    320016 bytes, avg  24.000
Number of literals      :     13334 =    572120 bytes, avg  42.907
Total footprint         :           =   1052224 bytes
Average bucket size     :     0.666
Variance of bucket size :     0.667
Std. dev. of bucket size:     0.817
Maximum bucket size     :         6
StringTable statistics:
Number of buckets       :     60013 =    480104 bytes, avg   8.000
Number of entries       :      1838 =     44112 bytes, avg  24.000 // 1838个字符常量条目数
Number of literals      :      1838 =    160920 bytes, avg  87.552 // 1838个字符常量字面量
Total footprint         :           =    685136 bytes
Average bucket size     :     0.031
Variance of bucket size :     0.031
Std. dev. of bucket size:     0.175
Maximum bucket size     :         2
```

time = 100000

比`time=100`多大概6000多个，没有10万，说过经过两次垃圾回收，对字符串常量池进行的垃圾回收。

```shell
[GC (Allocation Failure) [PSYoungGen: 4096K->512K(4608K)] 4096K->1153K(15872K), 0.0027053 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 4608K->512K(4608K)] 5249K->1177K(15872K), 0.0014481 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 4608K, used 874K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
  eden space 4096K, 8% used [0x00000000ffb00000,0x00000000ffb5aa88,0x00000000fff00000)
  from space 512K, 100% used [0x00000000fff80000,0x0000000100000000,0x0000000100000000)
  to   space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
 ParOldGen       total 11264K, used 665K [0x00000000ff000000, 0x00000000ffb00000, 0x00000000ffb00000)
  object space 11264K, 5% used [0x00000000ff000000,0x00000000ff0a6540,0x00000000ffb00000)
 Metaspace       used 3308K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 359K, capacity 388K, committed 512K, reserved 1048576K
SymbolTable statistics:
Number of buckets       :     20011 =    160088 bytes, avg   8.000
Number of entries       :     13385 =    321240 bytes, avg  24.000
Number of literals      :     13385 =    573880 bytes, avg  42.875
Total footprint         :           =   1055208 bytes
Average bucket size     :     0.669
Variance of bucket size :     0.670
Std. dev. of bucket size:     0.818
Maximum bucket size     :         6
StringTable statistics:
Number of buckets       :     60013 =    480104 bytes, avg   8.000
Number of entries       :      7188 =    172512 bytes, avg  24.000 // 7188个字符常量条目数
Number of literals      :      7188 =    461560 bytes, avg  64.213 // 7188个字符常量字面量
Total footprint         :           =   1114176 bytes
Average bucket size     :     0.120
Variance of bucket size :     0.123
Std. dev. of bucket size:     0.351
Maximum bucket size     :         4
```



# G1中的String去重操作



> # [JEP 192: String Deduplication in G1](http://openjdk.java.net/jeps/192)
>
> | Owner       | Per Liden                                                    |
> | ----------- | ------------------------------------------------------------ |
> | Type        | Feature                                                      |
> | Scope       | Implementation                                               |
> | Status      | Closed / Delivered                                           |
> | Release     | 8u20                                                         |
> | Component   | hotspot / gc                                                 |
> | Discussion  | hotspot dash gc dash dev at openjdk dot java dot net         |
> | Effort      | M                                                            |
> | Duration    | L                                                            |
> | Relates to  | [JEP 254: Compact Strings](http://openjdk.java.net/jeps/254) |
> | Reviewed by | Bengt Rutisson, John Coomes, Jon Masamitsu                   |
> | Endorsed by | Mikael Vidstedt                                              |
> | Created     | 2013/11/22 20:00                                             |
> | Updated     | 2017/06/07 22:25                                             |
> | Issue       | [8046182](https://bugs.openjdk.java.net/browse/JDK-8046182)  |
>
> ## Summary
>
> Reduce the Java heap live-data set by enhancing the G1 garbage collector so that duplicate instances of `String` are automatically and continuously deduplicated.
>
> ## Non-Goals
>
> It not a goal to implement this feature for garbage collectors other than G1.
>
> ## Motivation
>
> Many large-scale Java applications are currently bottlenecked on memory. Measurements have shown that roughly 25% of the Java heap live data set in these types of applications is consumed by `String` objects. Further, roughly half of those `String` objects are duplicates, where duplicates means `string1.equals(string2)` is true. Having duplicate `String` objects on the heap is, essentially, just a waste of memory. This project will implement automatic and continuous `String` deduplication in the G1 garbage collector to avoid wasting memory and reduce the memory footprint.



- 背景：对许多Java应用（有大的也有小的）做的测试得出以下结果：
  - 堆存活数据集合里面String对象占了25%
  - 堆存活数据集合里面重复的String对象有13.5%
  - String对象的平均长度是45
- 许多大规模的Java应用的瓶颈在于内存，测试表明，在这些类型的应用里面，**Java堆中存活的数据集合差不多25%是String对象**。更进一步，这里面差不多一半String对象是重复的，重复的意思是说：`string1.equals(string2)=true`。**堆上存在重复的String对象必然是一种内存的浪费**。这个项目将在G1垃圾收集器中实现自动持续对重复的String对象进行去重，这样就能避免浪费内存。

- 实现
  - 当垃圾收集器工作的时候，会访问堆上存活的对象。**对每一个访问的对象都会检查是否是候选的要去重的String对象**。
  - 如果是，把这个对象的一个引用插入到队列中等待后续的处理。一个去重的线程在后台运行，处理这个队列。处理队列的一个元素意味着从队列删除这个元素，然后尝试去重它引用的String对象。
  - 使用一个hashtable来记录所有的被String对象使用的不重复的char数组。当去重的时候，会查这个hashtable，来看堆上是否已经存在一个一模一样的char数组。
  - 如果存在， String对象会被调整引用那个数组，释放对原来的数组的引用，最终会被垃圾收集器回收掉。
  - 如果查找失败，char数组会被插入到hashtable，这样以后的时候就可以共享这个数组了。

- 命令行选项
  - UseStringDeduplication (bool)：开启String去重，**默认是不开启的，需要手动开启**。
  - PrintStringDeduplicationStatistics (bool)：打印详细的去重统计信息
  - StringDeduplicationAgeThreshold (uintx)：达到这个年龄的String对象被认为是去重的候选对象



