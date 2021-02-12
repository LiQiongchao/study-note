# String的基本特性

## 基本概念

- String：字符串,使用一对” “引起来表示。
  - String s1="atguigu";//字面量的定义方式
  - String s2 = new String ("hello")
- String声明为final的,不可被继承
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
  - 当对字符串重新赋值时,需要重写指定内存区域赋值,不能使用原有的value进行赋值
  - 当对现有的字符串进行连接操作时,也需要重新指定内存区域赋值不能使用原有的value进行赋值。
  - 当调用 String的replace()方法修改指定字符或字符串时,也需要重新指定内存区域赋值,不能使用原有的value进行赋值。
- 通过字面量的方式（区别于new）给一个字符串赋值,此时的字符串值声明在字符串常量池中。

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
- String的 String Pool是一个固定大小的 Hashtable,默认值大小长度是1009。如果放进 String Pool的 String非常多,就会造成Hash冲突严重,从而导致链表会很长,而链表长了后直接会造成的影响就是当调用String.intern时性能会大幅下降。
- 使用`-XX:StringTableSize`可设置 StringTable的长度
- 在jdk6中 StringTable是固定的,就是**1009**的长度,所以如果常量池中的字符串过多就会导致效率下降很快。 StringTableSize设置没有要求。
- 在jdk7中, StringTable的长度默认值是**60013**,1009是可设置的最小值。

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









# String的基本操作











# 字符串拼接操作









# intern()的使用











# StringTable的垃圾回收













# G1中的String去重操作













