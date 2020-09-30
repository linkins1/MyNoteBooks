## 6.类文件结构

### 6.1概述

#### 6.1.1 无关性

这里所指的无关性包括**平台无关性和语言无关性**，两者都依赖于字节码的存储格式和虚拟机。

#### 6.1.2 编译过程

.java文件需要经过javac编译成class文件，之后再通过jvm去进行加载、初始化、链接。

.c文件的编译过程包含了预处理-编译-汇编-链接，最终形成机器码直接在机器上运行。

#### 6.1.3 与类的对应关系

一般的，一个class文件对应于一个class类，但是class文件中使用到的类并不一定都在此class文件中定义。

如Class A{  Class.forName(B.class); }对应A.class，但是其中通过forname加载了B类，B类并不定义在A.class中

### 6.2Class类文件结构

> 几乎所有的**常量信息**都陈列在常量池中，class文件的其他部分通过给定**想要的常量在常量池中的索引**，从而在常量池中获取对应的常量

#### 6.2.1 定义

Class文件是**一组以8个字节为基础单位的二进制流**，各个数据项目严格按照顺序紧凑地排列在文件之中，中间**没有添加任何分隔符**，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。

#### 6.2.2 格式

Class文件格式采用一种类似于C语言**结构体的伪结构**来存储数据，这种伪结构中只有**两种**数据类型：“无符号数”和“表”。

- 无符号数

  属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值。

- 表

  表是由多个无符号数或者其他表作为数据项构成的复合数据类型，为了便于区分，所有表的命名都习惯性地以“_info”结尾。表用于描述有层次关系的复合结构的数据。

**整个Class文件本质上也可以视作是一张表**，这张表由表6-1所示的数据项**按严格顺序排列构成**。

<img src="/resources/imgs/jvm/image-20200921143430640.png" alt="image-20200921143430640" style="zoom: 67%;" />

下面按照顺序，对类文件结构分析

##### 1）魔数

每个Class文件的**头4个字节**被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。Java中采用**0XCAFEBABE**来作为魔数。

##### 2）版本号

紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号（Minor Version），第7和第8个字节是主版本号（Major Version）

##### 3）常量池

###### （1）定义

常量池可以比喻为Class文件里的**资源仓库**，它是Class文件结构中与其他项目关联最多的数据，通常也是占用Class文件**空间最大的数据项目之一**，另外，它还是在Class文件中**第一个出现的表类型**数据项目。由于常量池中的常量数**不固定**，因而在常量池开始处为一个**u2类型的常量池容量计数值**（constant_pool_count）。这个容量计数索引会保留"0"索引，因而，当此计数值赋值为22时，代表有21个常量，索引从1~21。其中索引"0"用于表示不引用任何一个常量池项目。

###### （2）包含内容

**字面量（Literal）和符号引用（Symbolic References）**

- 字面量

  一般指**文本字符串**和**final修饰的常量**，以及类元信息和方法元信息，譬如Integer类中的内部类IntegerCache中的变量static final Integer[] cache，就是用于存储-128~127的整形变量，当定义了Integer i = 1；就会在常量池中获取这个对象。

- 符号引用

  - 被模块导出或者开放的包（Package）
  - 类和接口的全限定名（Fully Qualified Name）
  - 字段的名称和描述符（Descriptor）
  - 方法的名称和描述符
  - 方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）
  - 动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）

###### （3）结构

常量池中的每一项都是一个**表**，这些表的结构各不相同，但是其相同处是开头都是一个u1类型的标志位，用于指示当前表中存储的常量类型

###### （4）示例

```java
package org.fenixsoft.clazz;
public class TestClass {
    private int m;
    public int inc() {
    	return m + 1;
    }
}
```

```reStructuredText
C:\>javap -verbose TestClass
Compiled from "TestClass.java"
public class org.fenixsoft.clazz.TestClass extends java.lang.Object
    SourceFile: "TestClass.java"
    minor version: 0
    major version: 50
    Constant pool:
const #1 = class #2; // org/fenixsoft/clazz/TestClass
const #2 = Asciz org/fenixsoft/clazz/TestClass;
const #3 = class #4; // java/lang/Object
const #4 = Asciz java/lang/Object;
const #5 = Asciz m;
const #6 = Asciz I;
const #7 = Asciz <init>;
const #8 = Asciz ()V;
const #9 = Asciz Code;
const #10 = Method #3.#11; // java/lang/Object."<init>":()V
const #11 = NameAndType #7:#8;// "<init>":()V
const #12 = Asciz LineNumberTable;
const #13 = Asciz LocalVariableTable;
const #14 = Asciz this;
const #15 = Asciz Lorg/fenixsoft/clazz/TestClass;;
const #16 = Asciz inc;
const #17 = Asciz ()I;
const #18 = Field #1.#19; // org/fenixsoft/clazz/TestClass.m:I
const #19 = NameAndType #5:#6; // m:I
const #20 = Asciz SourceFile;
const #21 = Asciz TestClass.java;
```

![image-20200921145525393](/resources/imgs/jvm/image-20200921145525393.png)

常量池入口从offset=8开始，头两个字节代表常量池中的常量数目，观察为0x0016，也即22，由于索引0保留，索引有21个常量。

接着就是常量池中的每个表，首先为tag字段，此处为0x07，查[表](######常量池项目类型)得此**tag为CONSTANT_Class_info型常量**，此常量(表)的第二个字段是u2类型的，存储的是常量池中的索引值(也叫做对应常量的**符号引用**)，本例子中为0x0002，也即指向常量池中的第二个常量，第二个常量的tag为0x01，查表为CONSTANT_Utf8_info类型，常量(表)的第二个字段是u2类型的，代表表中存储的字符串长度是多少字节(length个)，之后紧跟的是length个u1类型的字节，用于存储真正的常量值。本例中长度为0x001D，也即29个字节，那么证明从offset第二行开始的29个字节为实际的常量值，换算可知为org/fenixsoft/clazz/TestClass

##### 4）访问标志

常量池之后的**两个字节**代表访问标志(access flag)，包含如下几种类型

<img src="/resources/imgs/jvm/image-20200921152833830.png" alt="image-20200921152833830" style="zoom: 67%;" />

实际使用时需要根据修饰的对象不同(类/接口)，根据给定的修饰符将**涉及到的标志**进行**按位或**得到最终的标志位的值。对于如一个普通的public类，那么其标志位的值即为0x0001和0x0020按位或(注意16进制要转为2进制)得到0x0021。

##### 5）类索引、父类索引与接口索引集合

###### （1）类索引（this_class）

u2类型，指向该类的全限定类名

###### （2）父类索引（super_class）

u2类型，指向父类的全限定类名

###### （3）接口索引集合（interfaces）

**u2类型的数据的集合**，用来描述这个类实现了哪些接口，这些被实现的接口将按implements关键字（如果这个Class文件表示的是一个接口，则应当是extends关键字）后的接口顺序从左到右排列在接口索引集合中。

其中第一项为u2类型的接口计数器，表示实现的接口的总数。

###### （4）示例

![image-20200921154043830](/resources/imgs/jvm/image-20200921154043830.png)

通过上面可以看出，0x0001代表类索引，0x0003代表父类索引，0x0000代表接口集合的数量为0，也即没有实现任何接口。结合常量池中的示例，可知类索引指向第一个常量，该常量又指向第二个常量，得到最终的类索引；父类索引同理。

```text
const #1 = class #2; // org/fenixsoft/clazz/TestClass
const #2 = Asciz org/fenixsoft/clazz/TestClass;
const #3 = class #4; // java/lang/Object
const #4 = Asciz java/lang/Object;
```

##### 6）字段表集合

###### （1）定义

字段表（field_info）用于**描述接口或者类中声明的变量**，由于字段的数目不确定因而在入口处仍为一个**u2类型的值**用于记录字段表中的**字段数目**。Java语言中的“字段”（Field）包括**类级变量以及实例级变量**，但**不包括**在方法内部声明的局部变量。字段表的结构如下：

![image-20200921154950686](/resources/imgs/jvm/image-20200921154950686.png)

- **access_flags存储字段修饰符**(public final static transient volatile enum)，类似于访问标志，都需要把对应变量的所有修饰符都取出并进行**按位或**，得到最终的值作为此字段的值。

- **之后的两个索引值分别是name_index和descriptor_index**，都是对常量池的引用，分别代表该字段的**简单名称**和该字段的**描述符**(int,java/lang/String,[,L)。举例如下

```java
private String sArr[][];
private int m;
protected int iArr[];
```

那么对于sArr，其访问类型为private，所以访问标志为0x0002，由于String是对象类型，因而需要用**L加String的全限定类名**来标识，又因是二维数组，因而需要标记为**[[**，所以字段描述符为**[[Ljava/lang/String**

同理，第三个字段会被标识为**[I**，第二个字段会被标识为**I**

- 之后是类型为**u2的attributes_count**，用于记录该字段的属性表集合中的属性个数
- 之后是**属性表**，用于存储该字段的各个不同的属性

举例来说，如果有“final static int m=123；”，那就可能会存在一项名称为**ConstantValue的属性**，其值指向常量123

###### （2）示例

![image-20200921161226530](/resources/imgs/jvm/image-20200921161226530.png)

上图中，前两个字节代表计数值，为0x0001，代表只有一个字段，之后的字段是0x0002，代表是private，之后是0x0005，指向常量池中第五个常量为m，之后是0x0006，只想常量池中的第六个常量为I，因而可以反推出这个字段为private int m

```text
const #5 = Asciz m;
const #6 = Asciz I;
```

###### （3）注意

字段表集合中**不会列出从父类或者父接口中继承而来的字段**，但有**可能出现原本Java代码之中不存在的字段**，譬如在内部类中为了保持对外部类的访问性，编译器就会自动添加指向外部类实例的字段。

##### 7）方法表集合

###### （1）定义

方法表的结构如同字段表一样，**入口处为u2类型的计数值**，用于表示有多少个方法表项，之后依次包括**访问标志**（access_flags）、**名称索引**（name_index）、**描述符索引**（descriptor_index）、**属性表集合**（attributes），如下所示

<img src="/resources/imgs/jvm/image-20200921170404261.png" alt="image-20200921170404261" style="zoom:67%;" />

注意到方法的修饰符等信息都存储在前面的字段中，方法体中的代码会被Javac编译成字节码指令后放入属性表中的**Code属性**中

用描述符来描述方法时，按照**先参数列表、后返回值的顺序**描述，参数列表按照参数的严格顺序放在一组小括号“()”之内。

| 方法名                                                       | 描述符              |
| :----------------------------------------------------------- | ------------------- |
| void inc()                                                   | ()V                 |
| indexOf(char[]source，int sOffset，int sCount，char[]target，int tOffset，int tCount，int fIndex) | ([CII[CIII)I        |
| java.lang.String toString()                                  | ()Ljava/lang/String |

###### （2）示例

![image-20200921170714622](/resources/imgs/jvm/image-20200921170714622.png)

上图中，头两个字节代表方法表中的个数，0x0002，代表有两个方法，之后为0x0001，代表访问标志为public，之后0x0007指向常量池中的第七个常量，值为`<init>`，是编译器添加的实例构造器，描述符索引值为0x0008，对应常量为“()V”，属性表计数器attributes_count的值为0x0001，表示此方法的属性表集合有1项属性，属性名称的索引值为0x0009，对应常量为“Code”，说明此属性是方法的字节码描述

###### （3）注意

与字段表集合相对应地，如果父类/接口的方法在子类中**没有被重写**（Override），方法表集合中就不会出现来自父类的方法信息。但同样地，有可能会出现由编译器自动添加的方法，最常见的便是类构造器`“<clinit>()”`方法和实例构造器`“<init>()”`方法。
在Java语言中，要**重载一个方法**，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名。**特征签名**是指一个方法中各个参数在常量池中的字段符号引用的集合，也**正是因为返回值不会包含在特征签名之中，所以Java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的**。但是在Class文件格式之中，特征签名的范围明显要更大一些，只要描述符不是完全一致的两个方法就可以共存。也就是说，如果两个方法有相同的名称和特征签名，但返回值不同，那么也是可以合法共存于同一个Class文件的。

##### 8）属性表集合

Class文件、字段表、方法表都可以携带自己的属性表集合，以描述某些场景专有的信息，属性表中的每一个属性，它的名称都要从常量池中引用一个**CONSTANT_Utf8_info类型的常量**来表示，而**属性值的结构则是完全自定义**的，只需要通过一个**u4的长度属性**去说明属性值所占用的位数即可。因而，每个属性的结构如下

![image-20200921173826776](/resources/imgs/jvm/image-20200921173826776.png)

下面介绍重要属性

###### （1）Code属性

Java程序方法体里面的代码经过Javac编译器处理之后，最终变为**字节码指令**存储在Code属性内。Code属性出现在**方法表的属性集合**之中，但**并非所有**的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性

- Code属性表结构

  <img src="/resources/imgs/jvm/image-20200922132904069.png" alt="image-20200922132904069" style="zoom:67%;" />

  - attribute_name_index是一项指向CONSTANT_Utf8_info型常量的索引，此常量值固定为“Code”
- attribute_length指示了**属性值的长度**，由于属性名称索引与属性长度一共为6个字节，所以属性值的长度固定为**整个属性表长度减去6个字节**。
  - max_stack代表了操作数栈（Operand Stack）深度的最大值
    - 操作数栈中存放栈帧，方法执行时，操作数栈的深度不可以超过max_stack
  - max_locals代表了局部变量表所需的存储空间
    - 局部变量表中的表项是变量槽slot，不超过32位的基本数据类型占一个slot，64位的占两个slots
    - 为了减少局部变量表的大小，可以**重用**局部变量槽，以**同时存活的最大变量数量来决定槽的数量**
    - 局部变量表存储下列数据
      - 方法参数（包括实例方法中的隐藏参数“this”），也正因如此，对于实例方法，其局部变量表至少存在一个this变量。而静态方法不存在此变量
      - 显式异常处理程序的参数（Exception Handler Parameter，就是try-catch语句中catch块中所定义的异常）
      - 方法体中定义的局部变量
  - code_length和code用来存储Java源程序编译后生成的字节码指令
    - code_length代表字节码长度
    - code是用于存储字节码指令的一系列字节流，每个**字节可通过查表得到对应的字节码指令**
  - exception_table_length和exception_table（异常表）属性由Java代码中的try-catch-finally转化而来
    - [附](###附)部分给出了一个try-catch-finally的例子
  - Code属性表中的StackMapTable属性值得关注，这是为了代替以前比较消耗性能的基于数据流分析的
    类型推导验证器而增设的，可以减缓类加载过程中验证环节的压力。

###### （2）Exceptions属性

  用于列举出方法中可能抛出的受查异常（Checked Excepitons）

###### （3）LineNumberTable属性

  用于描述Java源码行号与字节码行号（字节码的偏移量）之间的对应关系。**非必要**，但是会影响异常抛出时显示行号以及按行号打断点调试

###### （4）LocalVariableTable及LocalVariableTypeTable属性

  LocalVariableTable属性用于描述栈帧中局部变量表的变量与Java源码中定义的变量之间的关系，**非必要**，但是会影响在调试期间根据参数名称从上下文中获得参数值。

  LocalVariableTypeTable属性用于引入特征签名，以便于在泛型情况下可以唯一确定一个变量

###### （5）SourceFile及SourceDebugExtension属性

  SourceFile属性用于记录生成这个Class文件的源码文件名称。SourceDebugExtension属性用于存储额外的代码调试信息

###### （6）ConstantValue属性

  用于**通知虚拟机自动为静态变量赋值**。只有被static关键字修饰的变量（类变量）才可以使用这项属性

  对非static类型的变量（也就是实例变量）的赋值是在**实例构造器**`<init>()`方法中进行的；而对于类变量，则有两种方式可以选择：在类构造器`<clinit>()`方法中或者使用ConstantValue属性。使用ConstantValue属性需要用final修饰。此时这个属性存在于字段表中字段的属性表中。	

###### （7）Signature属性

  任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variable）或参数化类型（Parameterized Type），则Signature属性会为它记录**泛型签名信息**。

  >之所以要专门使用这样一个属性去记录泛型类型，是因为Java语言的泛型采用的是**擦除法**实现的伪泛型，字节码（Code属性）中所有的泛型信息编译（类型变量、参数化类型）在编译之后都通通被擦除掉
  >
  >为了能够**在运行期间通过反射获取泛型的值**，增设了Signature属性

  ### 6.3 字节码指令

#### 6.3.1基本概念

##### 1）定义

Java虚拟机的指令由**一个字节长度**的、代表着某种特定操作含义的数字（称为操作码，Opcode）以及跟随其后的零至多个代表此操作所需的参数（称为操作数，Operand）构成。由于Java虚拟机采用**面向操作数栈而不是面向寄存器的架构**，所以**大多数指令都不包含操作数，只有一个操作码**，指令参数都**存放在操作数栈**中。

##### 2）特点

###### （1）劣势

- 指令集的操作码总数**不能够超过256条**
- 由于Class文件格式放弃了编译后代码的操作数长度对齐，这就意味着虚拟机在处理那些超过一个字节的数据时，不得不在运行时从字节中重建出具体数据的结构，譬如要将一个16位长度的无符号整数使用两个无符号字节存储起来（假设将它们命名为byte1和byte2）

###### （2）优势

- 放弃了操作数长度对齐，可以省略掉大量的填充和间隔符号
- 操作码简单高效

#### 6.3.2字节码与数据类型

##### 1）加载和存储指令

###### （1）定义

加载和存储指令用于将数据在**栈帧中的局部变量表和操作数栈之间**来回传输(栈帧位于Java虚拟机栈中)

###### （2）示例

- 将一个局部变量加载到操作栈-load
- 将一个数值从操作数栈存储到局部变量表-store
- 将一个常量加载到操作数栈-push

##### 2）运算指令

###### （1）定义

算术指令用于对**两个操作数栈上的值**进行某种特定运算，并把结果重新**存入到操作栈顶**

不存在直接支持byte、short、char和boolean类型的算术指令，对于上述几种数据的运算，应使用操作int类型的指令代替。浮点类型为另一类

###### （2）示例

add/sub/mul/div/....

>**只有除法指令（idiv和ldiv）以及求余指令**（irem和lrem）中当出现**除数为零时**会导致虚拟机抛出**ArithmeticException**异常，其余任何整型数运算场景都不应该抛出运行时异常。

##### 3）类型转换指令

###### （1）定义

类型转换指令可以将两种不同的数值类型相互转换，这些转换操作一般用于实现**用户代码中的显式类型转换操作**，或者用来处理字节码指令集中数据类型的转换(byte转int等)

###### （2）示例

i2b/i2c/....

> 如果浮点值是NaN，那转换结果就是int或long类型的0

##### 4）对象创建与访问指令

- 创建类实例的指令：new
- 创建数组的指令：newarray、anewarray、multianewarray
- 访问类字段（static字段，或者称为类变量）和实例字段（非static字段，或者称为实例变量）的指令：getfield、putfield、getstatic、putstatic
- ...

##### 5）操作数栈管理指令

- 将操作数栈的栈顶一个或两个元素出栈：pop、pop2
- 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1...
- 将栈最顶端的两个数值互换：swap

##### 6）控制转移指令

让Java虚拟机有条件或无条件地从指定位置指令（而不是控制转移指令）的下一条指令继续执行程序，从概念模型上理解，可以认为控制指令就是**在有条件或无条件地修改PC寄存器的值**。

##### 7）方法调用和返回指令

##### 8）异常处理指令

##### 9）同步指令

### 附

#### try-catch-finally例子

##### 1）代码

```java
// Java源码
public int inc() {
    int x;
    try {
        x = 1;
        //发生异常
        return x;
    } catch (Exception e) {
        x = 2;
        return x;
    } finally {
        x = 3;
    }
}
```

##### 2）字节码

<img src="/resources/imgs/jvm/image-20200922142136819.png" alt="image-20200922142136819" style="zoom:80%;" />

字节码中第0～4行所做的操作就是将整数1赋值给变量x，并且**将此时x的值复制一份副本到最后一个本地变量表的变量槽中**（这个变量槽里面的值在ireturn指令执行前将会被重新读到操作栈顶，作为方法返回值使用。为了讲解方便，笔者给这个变量槽起个名字：returnValue）。

如果**没有出现异常**，则会**继续走到第5～9行**，将变量x赋值为3，然后将之前保存在returnValue中的整数1读入到操作栈顶，最后ireturn指令会以int形式返回操作栈顶中的值，方法结束。

如果**出现了异常**，PC寄存器指针转到第10行，第**10～20行所做的事情是将2赋值给变量x，然后将变量x此时的值赋给returnValue，最后再将变量x的值改为3**。方法返回前同样将returnValue中保留的整数2读到了操作栈顶。从**第21行**开始的代码，作用是将变量x的值赋为3，并将栈顶的异常抛出，方法结束。

从上面的分析可以看出，尽管finally中的代码会将x的值置为3，**但是**，由于只有try和catch中的x赋值操作后有return，也即**只有这两个赋值操作后的值会被复制到本地变量表的变量槽中**，并把该变量槽的值放到栈顶后返回。

如果把finally中的代码修改为下面

```java
finally {
        x = 3;
    	return x;
    }
```

那么此时，会将x=3复制到变量槽中，并作为最终的返回值。此时finally中的赋值操作就会覆盖返回值，直观体现为finally中的return覆盖了try和catch中的return，在执行顺序上也是如此，会变为下面顺序

**无异常**：try中的x=1->finally中的x=3->finally中的return x结束；

**有异常**：try中的x=1->catch中的x=2->finally中的x=3->finally中的return x结束；

**有异常但不是Exception及其子类异常**：try中的x=1->finally中的x=3->finally中的return x结束。

---

### 参考表

###### 常量池项目类型

![image-20200921150606737](/resources/imgs/jvm/image-20200921150606737.png)















