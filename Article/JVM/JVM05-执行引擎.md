# 执行引擎

## 1、执行引擎概述

- 执行引擎是Java虚拟机的核心组成部分之一。

- 虚拟机是一个相对于“物理机”的概念，这两种机器都有代码执行能力，其区别是物理机的执行引擎是直接建立在处理器、缓存、指令集和操作系统层面上的，而==虚拟机的执行引擎则是由软件自行实现==的，因此可以不受物理条件制约地定制指令集与执行引擎的结构体系，能够执行那些不被硬件直接支持的指令集格式。

- JVM的主要任务是==负责装载字节码到其内部==，但字节码并不能够直接运行在操作系统之上，因为字节码指令并非等价于本地机器指令，它内部包含的仅仅只是一些能够被JVM所识别的字节码指令、符号表和其他辅助信息

- 那么，如果想让一个Java程序运行起来、执行引擎的任务就是==将字节码指令解释/编译为对应平台上的本地机器指令才可以==。简单来说，JVM中的执行引擎充当了将高级语言翻译为机器语言的译者.

- 工作过程

  - 从外观上来看，所有的Java虚拟机的执行引擎输入、输出都是一致的：输入的是字节码二进制流，处理过程是字节码解析执行的等效过程，输出的是执行结果。

  - 具体工作过程

    ![image-20211124222351883](D:\Blog\My-Blog\pic\jvm05_1)

    - 执行引擎在执行的过程中究竟需要执行什么样的字节码指令完全依赖于PC寄存器。
    - 每当执行完一项指令操作后，PC寄存器就会更新下一条需要被执行的指令地址。
    - 当然方法在执行的过程中，执行引擎有可能会通过存储在局部变量表中的对象引用准确定位到存储在Java堆区中的对象实例信息，以及通过对象头中的元数据指针定位到目标对象的类型信息。

  







# 2.	Java代码编译执行过程

### 大部分的程序代码转换成物理机的目标代码或虚拟机能执行的指令集之前，都需要经过下面图中的各个步骤

![image-20211124222501868](D:\Blog\My-Blog\pic\jvm05_2)

- 解释型语言走中间一行
- 编译型语言走下边一行



### 2.1	解释器

当Java虚拟机启动时会根据预定义的规范对字节码采用逐行解释的方式执行，将每条字节码文件中的内容“翻译”为对应平台的本地机器指令执行。

### 2.2	JIT编译器

就是虚拟机将源代码直接编译成和本地机器平台相关的机器语言。



### 2.3	为什么说Java是半编译半解释型语言？

JDK1.0时代，将Java语言定位为“解释执行”还是比较准确的。再后来，Java也发展出可以直接生成本地代码的编译器。 现在JVM在执行Java代码的时候，通常都会将解释执行与编译执行二者结合起来进行。

![image-20211124222602412](D:\Blog\My-Blog\pic\jvm05_3)

## 3.	解释器

JVM设计者们的初衷仅仅只是单纯地为了==满足Java程序实现跨平台特性==，因此避免采用静态编译的方式直接生成本地机器指令，从而诞生了实现解释器在运行时采用逐行解释字节码执行程序的想法。

![image-20211124222656248](D:\Blog\My-Blog\pic\jvm05_4)

- 解释器真正意义上所承担的角色就是一个运行时“翻译者”，将字节码文件中的内容“翻译”为对应平台的本地机器指令执行。
- 当一条字节码指令被解释执行完成后，接着再根据PC寄存器中记录的下一条需要被执行的字节码指令执行解释操作。

  在Java的发展历史里，一共有两套解释执行器，即古老的==字节码解释器==、现在普遍使用的==模板解释器==。

- 字节码解释器在执行时通过纯软件代码模拟字节码的执行，效率非常低下。· - 而模板解释器将每一 条字节码和一个模板函数相关联，模板函数中直接产生这条字节码执行时的机器码，从而很大程度上提高了解释器的性能。
  - 在HotSpot VM中，解释器主要由Interpreter模块和Code模块构成。
    - Interpreter模块：实现了解释器的核心功能
    - Code模块：用于管理HotSpot VM在运行时生成的本地机器指令

#### 现状

- 由于解释器在设计和实现上非常简单，因此除了Java语言之外，还有许多高级语言同样也是基于解释器执行的，比如Python、 Perl、Ruby等。但是在今天，基于解释器执行已经沦落为低效的代名词，并且时常被一些C/C+ +程序员所调侃。
- 为了解决这个问题，JVM平台支持一种叫作即时编译的技术。即时编译的目的是避免函数被解释执行，而是将整个函数体编译成为机器码，每次函数执行时，只执行编译后的机器码即可，这种方式可以使执行效率大幅度提升。
- 不过无论如何，基于解释器的执行模式仍然为中间语言的发展做出了不可磨灭的贡献。



## 4.	编译器

### 4.1	HotSpot VM 为何解释器与编译器共存

java代码的执行分类：

- 第一种是将源代码编译成字节码文件，然后再运行时通过解释器将字节码文件转为机器码执行
- 第二种是编译执行（直接编译成机器码）。现代虚拟机为了提高执行效率，会使用即时编译技术（JIT,Just In Time）将方法编译成机器码后再执行

  HotSpot VM是目前市面上高性能虛拟机的代表作之一。它采用==解释器与即时编译器并存的架构==。在Java虛拟机运行时，解释器和即时编译器能够相互协作，各自取长补短，尽力去选择最合适的方式来权衡编译本地代码的时间和直接解释执行代码的时间。  在今天，Java程序的运行性能早已脱胎换骨，已经达到了可以和C/C++程序一较高下的地步。

#### 解释器依然存在的必要性

有些开发人员会感觉到诧异，既然HotSpotVM中已经内置JIT编译器了，那么为什么还需要再使用解释器来“拖累”程序的执行性能呢？比如JRockit VM内部就不包含解释器，字节码全部都依靠即时编译器编译后执行。

**首先明确**： 当程序启动后，解释器可以马上发挥作用，省去编译的时间，立即执行。 编译器要想发挥作用，把代码编译成本地代码，需要一定的执行时间。但编译为本地代码后，执行效率高。

**所以**： 尽管JRockitVM中程序的执行性能会非常高效，但程序在启动时必然需要花费更长的时间来进行编译。对于服务端应用来说，启动时间并非是关注重点，但对于那些看中启动时间的应用场景而言，或许就需要采用解释器与即时编译器并存的架构来换取一一个平衡点。在此模式下，==当Java虚拟器启动时，解释器可以首先发挥作用，而不必等待即时编译器全部编译完成后再执行，这样可以省去许多不必要的编译时间。随着时间的推移，编译器发挥作用，把越来越多的代码编译成本地代码，获得更高的执行效率。==

同时，解释执行在编译器进行激进优化不成立的时候，作为编译器的“逃生门”。

#### HostSpot JVM的执行方式

当虛拟机启动的时候，解释器可以首先发挥作用，而不必等待即时编译器全部编译完成再执行，这样可以省去许多不必要的编译时间。并且随着程序运行时间的推移，即时编译器逐渐发挥作用，==根据热点探测功能，将有价值的字节码编译为本地机器指令，以换取更高的程序执行效率。==

### 4.2.	JIT编译器

#### 4.2.1	概念解释

- Java 语言的“编译器” 其实是一段“不确定”的操作过程，因为它可能是指一个==前端编译器==（其实叫“编译器的前端” 更准确一些）把.java文件转变成.class文件的过程；
- 也可能是指虚拟机的==后端运行期编译器==（JIT 编译器，Just In Time Compiler）把字节码转变成机器码的过程。
- 还可能是指使用==静态提前编译器==（AOT 编译器，Ahead Of Time Compiler）直接把. java文件编译成本地机器代码的过程。

前端编译器： Sun的Javac、 Eclipse JDT中的增量式编译器（ECJ）
JIT编译器： HotSpot VM的C1、C2编译器。
AOT编译器： GNU Compiler for the Java （GCJ） 、Excelsior JET。

#### 4.2.2	热点代码及探测方式

当然是否需要启动JIT编译器将字节码直接编译为对应平台的本地机器指令，则需要根据代码被调用执行的频率而定。关于那些需要被编译为本地代码的字节码，也被称之为“热点代码” ，JIT编译器在运行时会针对那些频繁被调用的“热点代码”做出深度优化，将其直接编译为对应平台的本地机器指令，以此提升Java程序的执行性能。

- 一个被多次调用的方法，或者是一个方法体内部循环次数较多的循环体都可以被称之为“热点代码”，因此都可以通过JIT编译器编译为本地机器指令。由于这种编译方式发生在方法的执行过程中，因此也被称之为栈上替换，或简称为OSR （On StackReplacement）编译。
- 一个方法究竟要被调用多少次，或者一个循环体究竟需要执行多少次循环才可以达到这个标准？必然需要一个明确的阈值，JIT编译器才会将这些“热点代码”编译为本地机器指令执行。这里主要依靠==热点探测功能==。
- ==目前HotSpot VM所采用的热点探测方式是基于计数器的热点探测==。
- 采用基于计数器的热点探测，HotSpot VM将会为每一个 方法都建立2个不同类型的计数器，分别为方法调用计数器（Invocation Counter） 和回边计数器（BackEdge Counter） 。
  - 方法调用计数器用于统计方法的调用次数
  - 回边计数器则用于统计循环体执行的循环次数

##### 方法调用计数器

- 这个计数器就用于统计方法被调用的次数，它的默认阈值在Client 模式 下是1500 次，在Server 模式下是10000 次。超过这个阈值，就会触发JIT编译。
- 这个阈值可以通过虚拟机参数一XX ：CompileThreshold来人为设定。
- 当一个方法被调用时， 会先检查该方法是否存在被JIT编译过的版本，如 果存在，则优先使用编译后的本地代码来执行。如果不存在已被编译过的版本，则将此方法的调用计数器值加1，然后判断方法调用计数器与回边计数器值之和是否超过方法调用计数器的阈值。如果已超过阈值，那么将会向即时编译器提交一个该方法的代码编译请求。

![image-20211124223022648](D:\Blog\My-Blog\pic\jvm05_5)

##### 回边计数器

它的作用是统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令称为“回边” （Back Edge）。显然，建立回边计数器统计的目的就是为了触发OSR编译。

![image-20211124223054520](D:\Blog\My-Blog\pic\jvm05_6)

### 4.3	HotSpot VM 可以设置程序执行方式

缺省情况下HotSpot VM是采用解释器与即时编译器并存的架构，当然开发人员可以根据具体的应用场景，通过命令显式地为Java虚拟机指定在运行时到底是完全采用解释器执行，还是完全采用即时编译器执行。如下所示：

- -Xint： 完全采用解释器模式执行程序；
- -Xcomp： 完全采用即时编译器模式执行程序。如果即时编译出现问题，解释器会介入执行。
- -Xmixed：采用解释器+即时编译器的混合模式共同执行程序。