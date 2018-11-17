# 虚拟机篇 之「运行时数据区域及虚拟机对象」

## 运行时数据区域

Java 虚拟机在执行 Java 程序的过程中会把它所管理的内存划分为若干个不同的数据区域，这些区域都有各自的用途以及创建和销毁的时间。有的区域随着虚拟机进程的启动而存在，有些区域则依赖用户线程的启动和结束而建立和销毁，这些区域被称之为运行时数据区域，其划分大致如下图所示：

![runtime-data-area](https://github.com/guobinhit/java-skills/blob/master/images/virtual-machine/vmo-runtime/runtime-data-area.png)

 - **程序计数器**：它是一块较小的内存空间，可以看做是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里，字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。如果线程正在执行的是一个 Java 方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的 Native 方法，这个计数器的值则为空。
- **虚拟机栈**：与程序计数器一样，虚拟机栈也是线程私有的，它的声明周期与线程相同。虚拟机栈描述的是 Java 方法的内存模型，每个方法在执行的同时都会创建一个栈帧，用于存储局部变量表、动态链接、方法出口等信息。每一个方法从调用到执行完成，都对应着一个栈帧在虚拟机栈中入栈到出栈的过程。局部变量表存放了编译期可知的各种基本数据类型、对象引用和`returnAddress`，其中 64 位长度的`long`和`double`类型的数据会占用 2 个局部变量空间（`Slot`），其余的数据类型只占用 1 个。
- **本地方法栈**：本地方法栈与虚拟机栈的用途一样，只不过虚拟机栈为虚拟机执行 Java 方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。此区域并不是虚拟机规范中强制必须实现的，因此如 HotSpot 就像虚拟机栈和本地方法栈合二为一。
- **堆**：对于大多数应用来说， Java 堆是 Java 虚拟机所管理的内存中最大的一块。Java 堆是被所有线程共享的一块内存区域，在虚拟机启动时创建，此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。Java 堆是垃圾收集器管理的主要区域，因此很多时候也被称作“GC堆”。根据 Java 虚拟机规范的规定，Java 堆可以处于物理上不连续的内存空间，只要逻辑上是连续的即可，就像我们的磁盘空间一样。目前主流的虚拟机都是可以支持堆扩展的，具体可以通过`-Xms`和`-Xmx`来控制，其中`-Xms`为设置最小堆内存，`-Xmx`为设置最大堆内存。
- **方法区**：方法区与 Java 堆一样，都是线程共享的，它用于存储已被虚拟机加载的类信息、常量、静态常量、即时编译器编译后的代码等数据。在 HotSpot 中，方法区也被称之为“永久代”，并且可以通过`-XX:PermSize`和`-XX:MaxPermSize`来控制方法区的大小，其中`-XX:PermSize`为设置方法区最小可用内存，`-XX:MaxPermSize`为设置方法区最大可用内存。

实际上，除了上述的运行时数据区域之外，还有一个内存区域值得我们了解，即

- **直接内存**：它并不是虚拟机运行时数据区的一个部分，也不是 Java 虚拟机规范中定义的内存区域，但在实际应用中，这部分内存也被频繁地使用，而且也可能导致`OutOfMemoryError`异常的出现。在 JDK 1.4 中新加入了`NIO`类，引入了一种基于通道与缓冲区的 I/O 方式，它可以使用 Native 函数库直接分配对外内存，然后通过一个存储在 Java 堆中的`DirectByteBuffer`对象作为这块内存的引用进行操作。这样能在一些场景汇总显著提高性能，因为避免了在 Java 堆和 Native 堆中来回复制数据。显然，本机直接内存的分配不会受到 Java 堆大小的限制，但是既然是内存，肯定还是会收到本机总内存大小以及处理器寻址空间的限制。因此，如果我们在分配内存的时候，忽略了直接内存，很可能使得各个内存区域总和大于物理内存限制，从而导致动态扩展的时候出现`OOM`异常。

## HotSpot 虚拟机对象

### 对象的创建

在虚拟机中创建对象，大致经过以下这些步骤，分别为：

- **加载类**：当虚拟机遇到一个`new`指令时，首先将去检查这个指令的参数是否能在常量池汇总定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。
- **分配内存**：在类加载检查通过后，接下来虚拟机将为新生对象分配内存。分配内存的方式主要有“指针碰撞”和“空闲列表”两种，而采用哪种内存分配方式则取决于堆内存是否规整，而堆内存是否规整则又取决于垃圾收集器是否带有压缩整理功能决定。此外，由于对象的创建操作非常频繁，因此导致对象的内存分配操作也非常频繁，为了保证线程安全，常见的解决方案主要有“CAS加失败重试”和“本地线程分配缓冲”两种。
- **设置必要信息**：在对象的内存分配完成后，虚拟机将对对象进行必要的设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息，这些信息存放在对象的对象头之中。

在上面的工作都完成后，从虚拟机的角度来看，一个新的对象已经产生了，但是从 Java 的角度来看，对象的创建才刚刚开始，`<init>`方法还没有执行，所有的字段都还未零。所以，一般来说（由字节码中是否跟随着`invokespecial`指令所决定），执行`new`指令之后会接着执行`<init>`方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全创建出来。

### 对象的内存布局

在 HotSpot 虚拟机中，对象的内存中存储的布局可以分为 3 块区域：对象头、实例数据和对其填充。

- **对象头**：它大致包含两部分信息，第一部分用于存储对象自身的运行时数据，如哈希码、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等；第二部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
- **实例数据**：它存储了对象真正有效的信息，也是在程序代码中所定义的各种类型的字段内容。无论是从父类继续下来的，还是在子类中定义的，都需要记录起来。这部分的存储顺序会受到虚拟机分配策略参数和字段在 Java 源码中定义顺序的影响。
- **对齐填充**：它不是必然存在的，也没有特别的含义，仅仅起着占位符的作用。由于 HotSpot VM 的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，换句话说，就是对象的大小必须是 8 字节的整数倍。而对象头部分正好是 8 字节的倍数，因此当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

### 对象的访问定位

创建对象是为了使用对象，我们的 Java 程序需要通过栈上的`reference`数据来操作堆上的具体对象。由于虚拟机规范仅规定了`reference`是一个引用对象，具体如何实现、如何定位、如何访问并没有做限制，因此对象访问方式也是取决于虚拟机实现而定的。目前主流的访问方式有使用“句柄”和“直接指针”两种。

- **句柄**：如果使用句柄访问的话，那么 Java 堆中将会划分出一块内存作为句柄池，`reference`中存储的就是对象的句柄地址，而句柄中包括了对象实例数据与数据类型各自的具体地址信息。使用句柄来访问的最大好处就是`reference`中存储的是稳定的句柄地址，在对象被移动（垃圾收集时移动对象是非常普通的行为）时只会改变句柄中的实例数据指针，而`reference`本身不需要修改。
- **直接指针**：如果使用直接指针访问的话，那么 Java 堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而`reference`中存储的直接就是对象地址。使用直接指针访问的最大好处就是速度更快，它节省了一次指针定位的时间开销，由于对象的访问在 Java 中非常频繁，因此这类开销极少成多以后也是一项非常可观的执行成本。HopSpot VM 就是使用直接指针的方式进行对象访问的。



----------

———— ☆☆☆ —— 返回 -> [The Skills of Java](http://blog.csdn.net/qq_35246620/article/details/78695893) <- 目录 —— ☆☆☆ ————