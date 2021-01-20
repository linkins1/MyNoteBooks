## 7.类加载机制

### 7.1概述

#### 7.1.1类加载流程

一个类型从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期将会经历**加载**（Loading）、**验证**（Verification）、**准备**（Preparation）、**解析**（Resolution）、**初始化**（Initialization）、**使用**（Using）和**卸载**（Unloading）**七个**阶段，**其中验证、准备、解析三个部分统称为连接**（Linking）

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200904210055167.png" alt="image-20200904210055167" style="zoom: 80%;" />

> **注意**：
>
> - 解析阶段可能会在初始化之后进行，如发生动态绑定时
> - 上面几个阶段是顺序开始，不是顺序进行或者完成，往往是交叉进行

#### 7.1.2类加载机制

Java虚拟机把描述类的数据从Class文件**加载到内存**，并对数据进行**校验**、转换**解析**和**初始化**，最终形成可以**被虚拟机直接使用的Java类型**，这个过程被称作虚拟机的**类加载机制**

> 这里所指的**.class**文件是存在于某个具体磁盘上的文件，以一串二进制流存在

#### 7.1.3与C语言的区别

C语言需要在**编译时**需要进行连接，而在Java语言里面，类型的加载、连接和初始化过程都是在**程序运行期间完成**的，这种策略让Java语言进行提前编译会面临额外的困难，也会**让类加载时稍微增加一些性能开销**，但是却为Java应用**提供了极高的扩展性和灵活性**，Java天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的

例如，编写一个面向接口的应用程序，可以等到**运行时**再指定其实际的实现类，用户可以通过Java预置的或自定义**类加载器**，让某个本地的应用程序在运行时从网络或其他地方上加载一个二进制流作为其程序代码的一部分

### 7.2主动引用&被动引用

#### 7.2.1区别

主动引用和被动引用**主要区分**在类加载过程中是否**对类立即初始化**(在加载、验证、准备之后进行)，主动引用存在以下几种情况，除了下面的几种情况，全部是被动引用

:one:遇到**new、getstatic、putstatic或invokestatic**这四条字节码指令时，如果类型没有进行过初始
化，则需要先触发其初始化阶段。能够生成这四条指令的典型Java代码场景有：

- 使用**new关键字实例化对象**的时候。
- 读取或设置一个**类型的静态字段**（被final修饰、已在编译期把结果放入常量池的静态字段除外）
  的时候。
- 调用一个类型的**静态方法**的时候。

:two:通过反射加载类时，如Class.forName，如果类型没有进行过初始化，则需要先触发其初始化。

:three: ​当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化

:four:当虚拟机启动时，用户需要指定一个启动类（**包含main()方法的类**），虚拟机会先初始化这个主类。

:five:当使用JDK 7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化

:six:当一个接口中定义了JDK 8新加入的默认方法（被default关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化

#### 7.2.2示例

被动引用示例参见[[1]](#####[1]深入理解Java虚拟机(第三版)-周志明)中对应章节

### 7.3类加载流程

#### 7.3.1加载(Loading)

> 用户应用程序和JVM共同完成

##### 1）定义

“加载”（Loading）阶段是整个“类加载”（Class Loading）过程中的一个阶段，分下面三个步骤进行

:one: 通过一个类的**全限定名**来**获取**定义此类的**二进制字节流**

:two: 将这个**字节流**所代表的静态存储结构**转化为方法区的运行时数据结构**

:three: 在内存中生成一个代表这个类的**java.lang.Class对象**，作为**访问方法区**中这个类的各种数据**的入口**

对于上面提到的第一步，可以采用多种方式来获取class文件的字节流

- 从ZIP压缩包中读取，JAR\WAR包等
- 从网络中读取
- 运行时计算生成，这种场景使用得最多的就是动态代理技术，在java.lang.reflect.Proxy中，就是用了ProxyGenerator.generateProxyClass()来为特定接口生成形式为“*$Proxy”的代理类的二进制字节流
- 由其他文件生成(JSP)
- 可以从加密文件中获取
- ......

##### 2）加载方式

###### （1）非数组类型

根据加载的类文件的不同，可以通过引导类加载器或用户自定义加载器来实现加载

###### （2）数组类型

**数组类本身不通过类加载器创建**，它是由**Java虚拟机直接在内存中动态构造**出来的，**数组中的元素需要通过类加载器加载**，对于不同的元素类型，采用的方式也不同，如下

- 如果元素是引用类型，则采用该引用类型对应的类的情况，调用对应的类加载器完成每个元素的加载，数组中元素的可访问性和数组的可访问性保持一致
- 如果元素是基本数据类型，则使用引导类加载器完成加载，数组中元素的可访问性默认为publc

##### 3）加载结束

加载结束后，Java虚拟机外部的二进制字节流会按照虚拟机所设定的格式存储在**方法区**之中。并在**堆**中生成一个对应的**java.lang.Class类的对象**

>加载阶段与连接阶段的部分动作（如一部分字节码文件格式验证动作）是交叉进行的，这两个阶段的开始时间仍然保持着固定的先后顺序

#### 7.3.2连接(Linking)

> JVM独自完成

##### 1）验证

 ###### （1）定义

验证是连接阶段的**第一步**，这一阶段的目的是确保Class文件的**字节流中包含的信息符合《Java虚拟机规范》的全部约束要求**，保证这些信息被当作代码运行后**不会危害虚拟机自身的安全**

验证阶段是**非常重要**的，这个阶段是否严谨，直接决定了Java虚拟机**是否能承受恶意代码的攻击**，从代码量和耗费的执行性能的角度上讲，验证阶段的工作量在虚拟机的类加载过程中占了**相当大的比重**

###### （2）验证阶段

:one: 文件格式验证

要验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理。包含下面步骤

- 是否以魔数0xCAFEBABE开头
- 主、次版本号是否在当前Java虚拟机接受范围之内
- 常量池的常量中是否有不被支持的常量类型（检查常量tag标志）
- ...

字节流只有过了验证阶段，才**允许存储到方法区中**，后续的三个验证阶段都**直接对方法区中的数据进行验证**

:two: 元数据验证

对字节码描述的信息进行**语义分析**，以保证其描述的信息符合《Java语言规范》的要求，验证点有如下几个

- 这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）
- 这个类的父类是否继承了不允许被继承的类（被final修饰的类）
- 如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法
- 类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，例如方法参数都一致，但返回值类型却不同等等

:three: 字节码验证

通过数据流分析和控制流分析，确定**程序语义是合法的、符合逻辑的**。需要对类的方法体（Class文件中的Code属性）进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的行为，例如

- 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似于“在操作栈放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中”这样的情况
- 保证任何跳转指令都不会跳转到方法体以外的字节码指令上
- ...

> JDK6之后将许多辅助校验措施都借用Code属性中的StackMapTable属性帮助辅助校验，减少了校验阶段的压力

:four: 符号引用验证

符号引用验证发生在解析阶段中将符号引用转化为直接引用时，主要验证下面几个方面

- 符号引用中通过字符串描述的全限定名是否能找到对应的类
- 在指定类中是否存在符合方法的字段描述符及简单名称所描述的方法和字段。
- 符号引用中的类、字段、方法的可访问性（private、protected、public、<package>）是否可被当前类访问

> 对于已经被反复使用和验证过，在生产环境的实施阶段就可以考虑使用-Xverify：none参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

##### 2）准备

###### （1）定义

准备阶段是正式为类中定义的变量（即静态变量，被static修饰的变量，不包括实例变量）**分配内存并设置类变量初始值**(不是初始化)的阶段。

###### （3）注意

:one: 实例变量随着对象实例化时一同分配到堆内存中

:two:对于变量public static int value = 123;那变量value**在准备阶段过后的初始值为0而不是123**，因为这时尚未开始执行任何Java方法，而把value赋值为123的putstatic指令是程序被编译后，存放于类构造器`<clinit>()`方法之中，所以把value赋值为123的动作要到类的初始化阶段才会被执行

:three:对于变量public static final int value = 123;其被final修饰，会存储在class文件常量池(位于元空间中)，且会在编译期间为该字段的属性表中添加ConstantValue属性来存储123，这时在准备阶段就会将此变量直接赋值为123

##### 3）解析

###### （1）定义

解析阶段是Java虚拟机**将常量池内的符号引用替换为直接引用的过程**。符号引用和直接引用的概念如下

:one:符号引用（Symbolic References）

符号引用以一组符号来描述所引用的目标，符号可以是**任何形式的字面量**(必须是字面量)，与虚拟机实现的内存布局无关

:two:直接引用（Direct References）

直接引用是可以**直接指向目标的指针、相对偏移量**或者是一个能间接定位到目标的**句柄**。与虚拟机实现的内存布局相关，且引用目标必须在虚拟机内存中存在

###### （2）解析分类

:one: 类或接口的解析

假设当前代码所处的**类为D**，如果要把一个**从未解析过的符号引用N解析为一个类或接口C的直接引用**，需要如下几步

- 如果C**不是数组类型**，那虚拟机将会把代表**N的全限定名传递给D的类加载器去加载这个类C**
- 如果C是一个数组类型，并且数组的元素类型为对象，也就是N的描述符会是类似“[Ljava/lang/Integer”的形式，那将会按照第一点的规则加载数组元素类型
- 如果上面两步没有出现任何异常，那么C在虚拟机中实际上已经成为一个有效的类或接口了，开始进行符号引用验证，确认D是否具备对C的访问权限。如果发现不具备访问权限，将抛出java.lang.IllegalAccessError异常。

:two: 字段解析

首先需要对字段所属的类或接口的符号引用进行解析(触发:one:的解析过程 )，如果成功，用C表示所属的类或接口，开始如下步骤

- 如果C本身有简单名称和描述符都匹配的字段则返回直接引用，结束解析
- 如果在C中实现了接口，将会按照继承关系从下往上递归搜索各个接口和它的父接口，如果接口中有简单名称和描述符都匹配的字段，返回直接引用，结束解析
- 如果C不是java.lang.Object的话，将会按照继承关系从下往上递归搜索其父类，如果在父中有简单名称和描述符都匹配的字段，返回直接引用，结束解析
- 否则，查找失败，抛出java.lang.NoSuchFieldError异常
- 如果解析成功，开始进行权限验证，如果不满足权限，则抛出java.lang.IllegalAccessError

> JavaC编译器会对存在继承关系的接口中的相同类变量报错，抛出The field X is ambiguous，并结束编译

:three: 类方法解析

 首先需要对字段所属的类或接口的符号引用进行解析(触发:one:的解析过程 )，如果成功，用C表示所属的类，如果此步解析出C为接口，则直接抛出java.lang.IncompatibleClassChangeError，否则，开始如下步骤

- 如果类C中有简单名称和描述符都与目标相匹配的方法，则返回直接引用，查找结束
- 如果类C的父类中递归查找发现有简单名称和描述符都与目标相匹配的方法，则返回直接引用，查找结束
- 在类C实现的接口列表及它们的父接口之中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果存在匹配的方法，说明类C是一个抽象类(由于在查找本类和父类时都没有找到，那么证明该方法没有被实现，那么C必须为抽象类)，这时候查找结束，抛出java.lang.AbstractMethodError异常
- 否则，抛出java.lang.NoSuchMethodError
- 如果解析成功，开始进行权限验证，如果不满足权限，则抛出java.lang.IllegalAccessError

:four: 接口方法解析

首先需要对字段所属的类或接口的符号引用进行解析(触发:one:的解析过程 )，如果成功，用C表示所属的接口，如果此步解析出C为类，则直接抛出java.lang.IncompatibleClassChangeError，否则，开始如下步骤

- 如果接口C中有简单名称和描述符都与目标相匹配的方法，则返回直接引用，查找结束
- 如果接口C的父接口中递归查找发现有简单名称和描述符都与目标相匹配的方法，则返回直接引用，查找结束。由于存在多重继承，如果找到多个匹配，则为失败
- 否则，抛出java.lang.NoSuchMethodError
- 如果解析成功，开始进行权限验证(验证是否为private static和模块访问权限)，如果不满足权限，则抛出java.lang.IllegalAccessError

#### 7.3.3初始化(Initialization)

> 用户应用程序独自完成

##### 1）`clinit()`方法

###### （1）定义

`<clinit>()`方法是**由编译器自动**收集类中的**所有类变量的赋值动作和静态语句块**（static{}块）中的语句**合并产生**的，编译器**收集的顺序**是由语句在**源文件中出现的顺序**决定的。

###### （2）注意

:one: 非法前向引用

静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问，如果访问，这就是一次非法的前向引用，示例如下

```java
public class Test {
    static {
        i = 0; // 给变量复制可以正常编译通过
        System.out.print(i); // 这句编译器会提示“非法向前引用”
    }
    static int i = 1;
}
```

:two: 类的`<clinit>()`执行顺序

`<clinit>()`方法与类的构造函数（即在虚拟机视角中的实例构造器`<init>()`方法）不同，它不需要显式地调用父类构造器，Java虚拟机会保证在子类的`<clinit>()`方法执行前，父类的`<clinit>()`方法已经执行完毕。因此在Java虚拟机中第一个被执行的`<clinit>()`方法的类型肯定是java.lang.Object。
由于父类的`<clinit>()`方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作

:three: 存在是否必要

`<clinit>()`方法对于类或接口来说并不是必需的，如果一个类中没有静态语句块，也没有对变量的
赋值操作，那么编译器可以不为这个类生成`<clinit>()`方法。

:four: 接口的`<clinit>()`执行顺序

由于接口中存在变量初始化的赋值操作，所以也会存在`<clinit>()`方法。执行接口的`<clinit>()`方法不需要先执行父接口的`<clinit>()`方法，因为只有当父接口中定义的变量被使用时，父接口才会被初始化。此外，接口的实现类在初始化时也一样不会执行接口的`<clinit>()`方法。

:five: 并发访问顺序

如果多个线程都要执行某个`<clinit>()`方法，那么在一个线程执行时，其他线程会被阻塞，且只会被执行一次

##### 2）定义

进行准备阶段时，变量**已经赋过一次系统要求的初始零值**，而在初始化阶段，则会根据程序员通过实际代码中的值去初始化类变量和其他资源，也即**初始化阶段就是执行类构造器`<clinit>()`方法的过程**。

### 7.4类加载器

>类加载阶段中的“通过一个类的全限定名来获取描述该类的二进制字节流”这个动作**在Java虚拟机外部去实现**，以便让应用程序**自己决定如何去获取所需的类**。实现这个动作的代码被称为“类加载器”（Class Loader）

#### 7.4.1 类加载器分类

##### 1）引导类(启动类)加载器（Bootstrap ClassLoader）

:one: 定义

引导类(启动类)加载器是由C++编写的，属于JVM一部分的加载器（也就意味着，在代码中获取由**此类加载器所加载的类**的加载器时为null）

:two: 涉及目录

<JAVA_HOME>\lib目录或者被-Xbootclasspath参数所指定的路径中存放的，而且是Java虚拟机能够识别的（按照文件名识别，如rt.jar、tools.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库

此外，如果用户程序想要将自定义的类交给引导类加载器加载时，可以将类加载器的引用置为null即可。负责返回类加载器的代码如下

```java
/**
Returns the class loader for the class. Some implementations may use null to represent the bootstrap class loader. */
public ClassLoader getClassLoader() {
    ClassLoader cl = getClassLoader0();
    if (cl == null)
        return null;
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        ClassLoader ccl = ClassLoader.getCallerClassLoader();
        if (ccl != null && ccl != cl && !cl.isAncestor(ccl)) {
            sm.checkPermission(SecurityConstants.GET_CLASSLOADER_PERMISSION);
        }
    }
    return cl;
}
```

从上面代码中也可以看出，如果获取到cl为null，则直接返回null，代表要使用引导类加载器

##### 2）自定义类加载器

###### （1）扩展类加载器(JDK9之前)（Extension Class Loader）

:one: 定义

扩展类加载器是由Java编写的类加载器，存在于sun.misc.Launcher$ExtClassLoader类中，该类继承于ClassLoader类

:two: 涉及目录

<JAVA_HOME>\lib\ext目录，或者被java.ext.dirs系统变量所指定的路径中所有的类库

###### （2）应用程序类(系统类)加载器（Application Class Loader）

:one: 定义

应用程序类(系统类)加载器是由Java编写的类加载器，存在于sun.misc.Launcher$AppClassLoader类中，该类继承于ClassLoader类，可以通过 `ClassLoader.getSystemClassLoader()` 来获取它

:two: 涉及目录

classpath下的类

###### （3）自定义类加载器

通过继承应用程序类加载器，并重写findClass方法来实现自定义类加载器

#### 7.4.2 类的唯一性

##### 1）定义

**类和加载这个类的类加载器共同确定了一个唯一的类**，也就意味着，在比较两个类是否相同时，必须要在同一个类加载器的环境下去讨论

##### 2）示例

```java
/**
* 类加载器与instanceof关键字演示
*
* @author zzm
*/
public class ClassLoaderTest {
    public static void main(String[] args) throws Exception {
        ClassLoader myLoader = new ClassLoader() {
        @Override
        public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1)+".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                        throw new ClassNotFoundException(name);
                    }
            }
        };
        Object obj = myLoader.loadClass("org.fenixsoft.classloading.ClassLoaderTest").newInstance();
        System.out.println(obj.getClass());
        System.out.println(obj instanceof org.fenixsoft.classloading.ClassLoaderTest);
    }
}
```

运行结果如下

```text
class org.fenixsoft.classloading.ClassLoaderTest
false
```

上面例子中，首先通过应用程序类加载器将ClassLoaderTest加载，但是在创建obj时，使用的是自定义的类加载器，所以即使obj的全限定类名也是ClassLoaderTest，但是由于类加载器不同，因而instanceof报错

#### 7.4.3 双亲委派模型（Parents Delegation Model）

##### 1）定义

如果一个类加载器收到了类加载的请求，按下面步骤进行

- 把这个请求委派给父类加载器去完成，直到传送到最顶层的启动类加载器中
- 如果父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。

##### 2）层级关系

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200924160330978.png" alt="image-20200924160330978" style="zoom: 67%;" />

上图中表明的层级关系并不是Java中的继承关系，只是代码反应(以组合形式复用父加载器的代码)出来的逻辑上的层级关系，这很直观，引导类加载器没有父类，扩展类和应用程序类都有父类ClassLoader，但是在收到加载器请求时，会按照上图中的层级关系进行请求递交。

##### 3）特点

###### （1）优势

Java中的类随着它的类加载器一起具备了一种带有**优先级的层次关系**。

例如类java.lang.Object，它存放在rt.jar之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都能够保证是同一个类

###### （2）劣势

一定程度上限制了自定义类加载器的灵活性

##### 4）代码实现

```java
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
{
    // 首先，检查请求的类是否已经被加载过了
    Class c = findLoadedClass(name);
    if (c == null) {
        try {
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // 如果父类加载器抛出ClassNotFoundException
            // 说明父类加载器无法完成加载请求
        }
        if (c == null) {
            // 在父类加载器无法加载时
            // 再调用本身的findClass方法来进行类加载
            c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

上面代码中可以看出，如果c为null，则查看父类加载器是不是 null，如果是，则使用引导类加载器，如果不是则查找对应的父类加载器，如果不是null，则使用父类加载器加载；如果是null或者父类加载器抛出ClassNotFoundException时，则使用用户重写的findClass进行加载，如果还无法加载，则宣告失败。

如果c不是null，则证明已经被加载过，直接进行解析即可

### 7.5Java模块化系统

#### 7.5.1 基本概念

##### 1）定义

Java module在JDK9引入，是一种用于进行代码隔离管理的手段。通过将不同的package放入不同的module中，形成多个.jmod文件，并定义好module-info.class文件来指定不同module之间的访问关系（可以访问哪些模块及本模块可以被哪些模块访问）。但是区分一个文件是通过类路径还是模块路径来访问，是通过该文件是存在于类路径还是模块路径来区分的。

##### 2）与Maven的模块的比较

java的module是将不同分类的代码按照模块进行隔离，只是将JDK进行功能性的划分，便于在使用时不必引入整个JDK而只使用指定模块即可。

而maven的module是用于指定不同工程使用到的依赖关系，来划分出多个不同的工程。

#### 7.5.2 模块化下的类加载器

引入模块化机制后，主要变化如下

##### 1）平台类加载器（Platform Class Loader）

扩展类加载器（Extension Class Loader）被平台类加载器（Platform Class Loader）取代。

由于采用模块化机制，因而不再需要使用<JAVA_HOME>\lib\ext来专门用于放置扩展类

##### 2）继承关系变化

变化前的继承关系

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200924163302693.png" alt="image-20200924163302693" style="zoom: 80%;" />

变化后的继承关系

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200924163345084q.png" alt="image-20200924163345084" style="zoom: 80%;" />

##### 3）双亲委派模型改动

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-202009241632385751.png" alt="image-20200924163238575" style="zoom: 67%;" />

##### 4）可访问关系验证

由于引入模块，原本public的类如果没有进行做模块关系引用，那么仍然是不可见的，那么在进行可访问性验证时，会更加苛刻
