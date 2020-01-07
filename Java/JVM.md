### JVM组成部分说明:
* ClassLoader
  ClassLoader是负责加载class文件,class文件在文件开头有特定的文件标示,并且ClassLoader只负责class文件的加载,至于它是否可以运行,则由Execution Engine决定

* Native Interface
  Native Interface是负责调用本地接口的.它的作用是调用不同语言的接口给JAVA用,他会在Native Method Stack中记录对应的本地方法,然后调用该方法时就通过Execution Engine加载对应的本地lib

* Execution Engine
  Execution Engine是执行引擎,也叫Interpreter.Class文件被加载后,会把指令和数据信息放入内存中,Execution Engine则负责把这些命令解释给操作系统

* Runtime Data Area
  Runtime Data Area则是存放数据的,分为五部分:Stack,Heap,Method Area,PC Register,Native Method Stack.几乎所有的关于java内存方面的问题,都是集中在这块
  * Stack
    Stack是java栈内存,它等价于C语言中的栈,栈的内存地址是不连续的,每个线程都拥有自己的栈.
    栈里面存储着的是StackFrame,在《JVM Specification》中文版中被译作java虚拟机框架,也叫做栈帧.
    StackFrame包含三类信息: 局部变量,执行环境,操作数栈
    * 局部变量用来存储一个类的方法中所用到的局部变量
    * 执行环境用于保存解析器对于java字节码进行解释过程中需要的信息,包括:上次调用的方法,局部变量指针和操作数栈的栈顶和栈底指针
    * 操作数栈用于存储运算所需要的操作数和结果
    StackFrame在方法被调用时创建,在某个线程中,某个时间点上,只有一个框架是活跃的,该框架被称为Current Frame,而框架中的方法被称为Current Method,其中定义的类为Current Class.局部变量和操作数栈上的操作总是引用当前框架.当Stack Frame中方法被执行完之后,或者调用别的StackFrame中的方法时,则当前栈变为另外一个StackFrame.Stack的大小是由两种类型:固定和动态的,动态类型的栈可以按照线程的需要分配
  * Heap
    Heap是用来存放对象信息的,和Stack不同,Stack代表着一种运行时的状态.换句话说,栈是运行时单位,解决程序该如何执行的问题,而堆是存储的单位,解决数据存储的问题.
    Heap是伴随着JVM的启动而创建,负责存储所有对象实例和数组的.堆的存储空间和栈一样是不需要连续的,它分为Young Generation和Old Generation(也叫Tenured Generation)两大部分.Young Generation分为Eden和Survivor,Survivor又分为From Space和 ToSpace.

    和Heap经常一起提及的概念是PermanentSpace,它是用来加载类对象的专门的内存区,是非堆内存,和Heap一起组成JAVA内存,它包含MethodArea区(在没有CodeCache的HotSpotJVM实现里,则MethodArea就相当于GenerationSpace)

    在JVM初始化的时候,我们可以通过参数来分别指定,PermanentSpace的大小,堆的大小,以及Young Generation和Old Generation的比值,Eden区和From Space的比值,从而来细粒度的适应不同JAVA应用的内存需求
  * PC Register
    PC Register是程序计数寄存器,每个JAVA线程都有一个单独的PC Register,他是一个指针,由Execution Engine读取下一条指令.如果该线程正在执行java方法,则PC Register存储的是正在被执行的指令的地址;如果是本地方法,PC Register的值没有定义.PC寄存器非常小,只占用一个字宽,可以持有一个returnAdress或者特定平台的一个指针
  * Method Area
    Method Area在HotSpot JVM的实现中属于非堆区,非堆区包括两部分:Permanet Generation和Code Cache,而Method Area属于Permanert Generation的一部分.
    Permanent Generation用来存储类信息,比如说:class definitions,structures,methods,field,method(data and code)和constants.
    Code Cache用来存储Compiled Code,即编译好的本地代码,在HotSpot JVM中通过JIT(Just In Time)Compiler生成,JIT是即时编译器,他是为了提高指令的执行效率,把字节码文件编译成本地机器代码
  * Native Method Stack
    Native Method Stack是供本地方法(非java)使用的栈.每个线程持有一个Native Method Stack


### JVM运行原理
    
    Java程序被javac工具编译为.class字节码文件之后,我们执行java命令,该class文件便被JVM的Class Loader加载,可以看出JVM的启动是通过JAVA Path下的java.exe或者java进行的.

    JVM的初始化,运行到结束大概包括这么几步:
    调用操作系统API判断系统的CPU架构,根据对应CPU类型寻找位于JRE目录下的/lib/jvm.cfg文件,然后通过该配置文件找到对应的jvm.dll文件(如果我们参数中有-server或者-client,则加载对应参数所指定的jvm.dll,启动指定类型的JVM),初始化jvm.dll并且挂接到JNIENV结构的实例上,之后就可以通过JNIENV实例装载并且处理class文件了.class文件是字节码文件,它按照JVM的规范,定义了变量,方法等的详细信息,JVM管理并且分配对应的内存来执行程序,同时管理垃圾回收.直到程序结束,一种情况是JVM的所有非守护线程停止,一种情况是程序调用System.exit(),JVM的生命周期也结束