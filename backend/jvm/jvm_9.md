## 9.前端编译与后端编译

### 9.1前端编译(Javac)

#### 9.1.1定义

前端编译是前端编译器（叫“编译器的前端”更准确一些）**把*.java文件转变成*.class文件的过程**；

具体执行的步骤有如下部分

![image-20200929164743445](/resources/imgs/jvm/image-20200929164743445.png)

> 此处省略了每一步的具体细节

#### 9.1.2Java语法糖

##### 1）泛型（Type Erasure Generics）

###### （1）类型擦除

通过下面这个例子来分析

```java
ArrayList<Integer> ilist = new ArrayList<Integer>();
ArrayList<String> slist = new ArrayList<String>();
ArrayList list; // 裸类型
list = ilist;
list = slist;
```

上面提到的**裸类型**(其中包含的是Object)是作为`ArrayList<String>`和`ArrayList<Integer>`共同的父类型，但是他们之间**不存在继承关系**，只是进行了类型擦除后都以父类型的形式存在。下面是一个类型擦除的例子

- 擦除前

```java
public static void main(String[] args) {
    Map<String, String> map = new HashMap<String, String>();
    map.put("hello", "你好");
    map.put("how are you?", "吃了没？");
    System.out.println(map.get("hello"));
    System.out.println(map.get("how are you?"));
}
```

- 擦除后

```java
public static void main(String[] args) {
    Map map = new HashMap();
    map.put("hello", "你好");
    map.put("how are you?", "吃了没？");
    System.out.println((String) map.get("hello"));
    System.out.println((String) map.get("how are you?"));
}
```

从上面可以看出，**只有泛型是一个类时才可以进行类型擦除转换为Object，由于基本数据类型不存在父类，所以需要提供包装类来辅助类型擦除**

###### （2）类型擦除与重载（在JDK6下讨论，不同的前端编译器结果不同！！！）

先阅读下面例子

```java
public class GenericTypes {
    public static void method(List<String> list) {
        System.out.println("invoke method(List<String> list)");
    }
    public static void method(List<Integer> list) {
        System.out.println("invoke method(List<Integer> list)");
    }
}
```

这段代码不能通过编译，因为类型擦除后，两个方法的特征签名一致，也就无法实现重载，但是再阅读下面的例子

```java
public class GenericTypes {
    public static String method(List<String> list) {
        System.out.println("invoke method(List<String> list)");
        return "";
    }
    public static int method(List<Integer> list) {
        System.out.println("invoke method(List<Integer> list)");
        return 1;
    }
    public static void main(String[] args) {
        method(new ArrayList<String>());
        method(new ArrayList<Integer>());
    }
}
```

这段代码可以通过编译且成功实现了重载，这是因为返回值不同，尽管重载不是由返回值来确定的，但是由于返回值不同，所以可以同时存在于class文件中，从而可以通过编译。

##### 2）自动装、拆箱与遍历循环

首先阅读一个例子

```java
public static void main(String[] args) {
    List<Integer> list = Arrays.asList(1, 2, 3, 4);
    int sum = 0;
    for (int i : list) {
        sum += i;
    }
    System.out.println(sum);
}
```

上面的代码被编译后再反编译如下

```java
public static void main(String[] args) {
    List list = Arrays.asList( new Integer[] {
        Integer.valueOf(1),
        Integer.valueOf(2),
        Integer.valueOf(3),
        Integer.valueOf(4) });
    int sum = 0;
    for (Iterator localIterator = list.iterator(); localIterator.hasNext(); ) {
        int i = ((Integer)localIterator.next()).intValue();
        sum += i;
    }
    System.out.println(sum);
}
```

可以看出在调用Integer对象进行运算时调用`intValue()`转换为int值在进行计算，且在遍历时，需要将获得到的Iterator强制转换为Integer类型。

下面是一个容易判断错误的例子

```java
public static void main(String[] args) {
    Integer a = 1;
    Integer b = 2;
    Integer c = 3;
    Integer d = 3;
    Integer e = 321;
    Integer f = 321;
    Long g = 3L;
    System.out.println(c == d);
    System.out.println(e == f);
    System.out.println(c == (a + b));
    System.out.println(c.equals(a + b));
    System.out.println(g == (a + b));
    System.out.println(g.equals(a + b));
}
```

运行结果如下

```text
true
false
true
true
true
false
```

下面逐一解释

:one: 是因为值为[-128,127]的Integer会被存入常量池中，因而c和d的引用(地址)一致

:two: 由于不属于常量池范围，所以两者是不同的引用

:three: 由于运算符触发了自动拆箱

:four: 观察下面Integer的equals方法，当a+b会触发一次自动拆箱，之后进入equals进行自动装箱，最后比较两个int值

:five: 由于运算符触发了自动拆箱，且==会对基本类型进行自动类型转换(基本数据类型之间的)

:six:  Long的equals方法同理，由于自动拆装箱后为Integer类型，不是Long类型，所以为false

```java
public boolean equals(Object obj) {
    if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

##### 3）条件编译

观察下面的例子

```java
public static void main(String[] args) {
    if (true) {
    System.out.println("block 1");
    } else {
    System.out.println("block 2");
    }
}
```

经过编译在反编译后结果如下

```java
public static void main(String[] args) {
    System.out.println("block 1");
}
```

可以看出只包含一条sout，这点类似于C语言中的#ifdef

> **只有使用if语句才可以触发条件编译**

### 9.2 后端编译（JIT+AOT）

#### 9.2.1定义

后端编译是指Java虚拟机的即时编译器（常称JIT编译器，Just In Time Compiler）运行期**把字节码转变成本地机器码的过程**；和使用静态的提前编译器（常称AOT编译器，Ahead Of Time Compiler）**直接把程序编译成与目标机器指令集相关的二进制代码的过程**

#### 9.2.2解释器与编译器

解释器与编译器两者各有优势：
当程序需要迅速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间，立即运行。当程序启动后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译成本地代码，这样可以减少解释器的中间损耗，获得更高的执行效率

当程序运行环境中内存资源限制较大，可以使用解释执行节约内存（如部分嵌入式系统中只有解释器的存在），反之可以使用编译执行来提升效率

<img src="/resources/imgs/jvm/image-20200929165341076.png" alt="image-20200929165341076" style="zoom:67%;" />























