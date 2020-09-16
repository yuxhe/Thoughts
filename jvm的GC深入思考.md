调用栈里的引用类型数据是GC的根集合（root set）的重要组成部分；找出栈上的引用是GC的根枚举（root enumeration）中不可或缺的一环。

JVM选择用什么方式会影响到GC的实现：

如果JVM选择不记录任何这种类型的数据，那么它就无法区分内存里某个位置上的数据到底应该解读为引用类型还是整型还是别的什么。这种条件下，实现出来的GC就会是“保守式GC（conservative GC）”。在进行GC的时候，JVM开始从一些已知位置（例如说JVM栈）开始扫描内存，扫描的时候每看到一个数字就看看它“像不像是一个指向GC堆中的指针”。这里会涉及上下边界检查（GC堆的上下界是已知的）、对齐检查（通常分配空间的时候会有对齐要求，假如说是4字节对齐，那么不能被4整除的数字就肯定不是指针），之类的。然后递归的这么扫描出去。

保守式GC的好处是相对来说实现简单些，而且可以方便的用在对GC没有特别支持的编程语言里提供自动内存管理功能。Boehm-Demers-Weiser GC是保守式GC中的典型代表，可以嵌入到C或C++等语言写的程序中。

小历史故事： 
微软的JScript和早期版VBScript也是用保守式GC的；微软的JVM也是。VBScript后来改回用引用计数了。而微软JVM的后代，也就是.NET里的CLR，则改用了完全准确式GC。 
为了赶上在一个会议上发布消息，微软最初的JVM原型只有一个月左右的时间从开工到达到符合Java标准。所以只好先用简单的办法来实现，也就自然选用了保守式GC。 
信息来源：Patrick Dussud在Channel 9的访谈，23分钟左右

保守式GC的缺点有： 
1、会有部分对象本来应该已经死了，但有疑似指针指向它们，使它们逃过GC的收集。这对程序语义来说是安全的，因为所有应该活着的对象都会是活的；但对内存占用量来说就不是件好事，总会有一些已经不需要的数据还占用着GC堆空间。具体实现可以通过一些调节来让这种无用对象的比例少一些，可以缓解（但不能根治）内存占用量大的问题。

2、由于不知道疑似指针是否真的是指针，所以它们的值都不能改写；移动对象就意味着要修正指针。换言之，对象就不可移动了。有一种办法可以在使用保守式GC的同时支持对象的移动，那就是增加一个间接层，不直接通过指针来实现引用，而是添加一层“句柄”（handle）在中间，所有引用先指到一个句柄表里，再从句柄表找到实际对象。这样，要移动对象的话，只要修改句柄表里的内容即可。但是这样的话引用的访问速度就降低了。Sun JDK的Classic VM用过这种全handle的设计，但效果实在算不上好。

由于JVM要支持丰富的反射功能，本来就需要让对象能了解自身的结构，而这种信息GC也可以利用上，所以很少有JVM会用完全保守式的GC。除非真的是特别懒…

JVM可以选择在栈上不记录类型信息，而在对象上记录类型信息。这样的话，扫描栈的时候仍然会跟上面说的过程一样，但扫描到GC堆内的对象时因为对象带有足够类型信息了，JVM就能够判断出在该对象内什么位置的数据是引用类型了。这种是“半保守式GC”，也称为“根上保守（conservative with respect to the roots）”。

为了支持半保守式GC，运行时需要在对象上带有足够的元数据。如果是JVM的话，这些数据可能在类加载器或者对象模型的模块里计算得到，但不需要JIT编译器的特别支持。

前面提到了Boehm GC，实际上它不但支持完全保守的方式，也可以支持半保守的方式。GCJ和Mono都是以半保守方式使用Boehm GC的例子。

Google Android的Dalvik VM的早期版本也是使用半保守式GC的一个例子。不过到2009年中的时候Dalvik VM的内部版本就已经开始支持准确式GC了——代价是优化过的DEX文件的体积膨胀了约9%。 
其实许多较老的JVM都选择这种实现方式。

由于半保守式GC在堆内部的数据是准确的，所以它可以在直接使用指针来实现引用的条件下支持部分对象的移动，方法是只将保守扫描能直接扫到的对象设置为不可移动（pinned），而从它们出发再扫描到的对象就可以移动了。 
完全保守的GC通常使用不移动对象的算法，例如mark-sweep。半保守方式的GC既可以使用mark-sweep，也可以使用移动部分对象的算法，例如Bartlett风格的mostly-copying GC。

半保守式GC对JNI方法调用的支持会比较容易：管它是不是JNI方法调用，是栈都扫过去…完事了。不需要对引用做任何额外的处理。当然代价跟完全保守式一样，会有“疑似指针”的问题。

与保守式GC相对的是“准确式GC”，原文可以是precise GC、exact GC、accurate GC或者type accurate GC。外国人也挺麻烦的，“准确”都统一不到一个词上⋯ 
是什么东西“准确”呢？关键就是“类型”，也就是说给定某个位置上的某块数据，要能知道它的准确类型是什么，这样才可以合理地解读数据的含义；GC所关心的含义就是“这块数据是不是指针”。 
要实现这样的GC，JVM就要能够判断出所有位置上的数据是不是指向GC堆里的引用，包括活动记录（栈+寄存器）里的数据。

有几种办法：

1、让数据自身带上标记（tag）。这种做法在JVM里不常见，但在别的一些语言实现里有体现。就不详细介绍了。打标记的方式在半保守式GC中倒是更常见一些，例如CRuby就是用打标记的半保守式GC。CLDC-HI比较有趣，栈上对每个slot都配对一个字长的tag来说明它的类型，通过这种方式来减少stack map的开销；类似的实现在别的地方没怎么见过，大家一般都不这么取舍。 
2、让编译器为每个方法生成特别的扫描代码。我还没见过JVM实现里这么做的，虽说在别的语言实现里有见过。 
3、从外部记录下类型信息，存成映射表。现在三种主流的高性能JVM实现，HotSpot、JRockit和J9都是这样做的。其中，HotSpot把这样的数据结构叫做OopMap，JRockit里叫做livemap，J9里叫做GC map。Apache Harmony的DRLVM也把它叫GCMap。 
要实现这种功能，需要虚拟机里的解释器和JIT编译器都有相应的支持，由它们来生成足够的元数据提供给GC。 
使用这样的映射表一般有两种方式： 
1、每次都遍历原始的映射表，循环的一个个偏移量扫描过去；这种用法也叫“解释式”； 
2、为每个映射表生成一块定制的扫描代码（想像扫描映射表的循环被展开的样子），以后每次要用映射表就直接执行生成的扫描代码；这种用法也叫“编译式”。

在HotSpot中，对象的类型信息里有记录自己的OopMap，记录了在该类型的对象内什么偏移量上是什么类型的数据。所以从对象开始向外的扫描可以是准确的；这些数据是在类加载过程中计算得到的。

可以把oopMap简单理解成是调试信息。 在源代码里面每个变量都是有类型的，但是编译之后的代码就只有变量在栈上的位置了。oopMap就是一个附加的信息，告诉你栈上哪个位置本来是个什么东西。 这个信息是在JIT编译时跟机器码一起产生的。因为只有编译器知道源代码跟产生的代码的对应关系。 每个方法可能会有好几个oopMap，就是根据safepoint把一个方法的代码分成几段，每一段代码一个oopMap，作用域自然也仅限于这一段代码。 循环中引用多个对象，肯定会有多个变量，编译后占据栈上的多个位置。那这段代码的oopMap就会包含多条记录。

每个被JIT编译过后的方法也会在一些特定的位置记录下OopMap，记录了执行到该方法的某条指令的时候，栈上和寄存器里哪些位置是引用。这样GC在扫描栈的时候就会查询这些OopMap就知道哪里是引用了。这些特定的位置主要在： 
1、循环的末尾 
2、方法临返回前 / 调用方法的call指令后 
3、可能抛异常的位置

这种位置被称为“安全点”（safepoint）。之所以要选择一些特定的位置来记录OopMap，是因为如果对每条指令（的位置）都记录OopMap的话，这些记录就会比较大，那么空间开销会显得不值得。选用一些比较关键的点来记录就能有效的缩小需要记录的数据量，但仍然能达到区分引用的目的。因为这样，HotSpot中GC不是在任意位置都可以进入，而只能在safepoint处进入。 
而仍然在解释器中执行的方法则可以通过解释器里的功能自动生成出OopMap出来给GC用。

平时这些OopMap都是压缩了存在内存里的；在GC的时候才按需解压出来使用。 
HotSpot是用“解释式”的方式来使用OopMap的，每次都循环变量里面的项来扫描对应的偏移量。

对Java线程中的JNI方法，它们既不是由JVM里的解释器执行的，也不是由JVM的JIT编译器生成的，所以会缺少OopMap信息。那么GC碰到这样的栈帧该如何维持准确性呢？ 
HotSpot的解决方法是：所有经过JNI调用边界（调用JNI方法传入的参数、从JNI方法传回的返回值）的引用都必须用“句柄”（handle）包装起来。JNI需要调用Java API的时候也必须自己用句柄包装指针。在这种实现中，JNI方法里写的“jobject”实际上不是直接指向对象的指针，而是先指向一个句柄，通过句柄才能间接访问到对象。这样在扫描到JNI方法的时候就不需要扫描它的栈帧了——只要扫描句柄表就可以得到所有从JNI方法能访问到的GC堆里的对象。 
但这也就意味着调用JNI方法会有句柄的包装/拆包装的开销，是导致JNI方法的调用比较慢的原因之一。


----------------------------------------------------------------------------------
什么是安全点 参考文章：https://www.jianshu.com/p/c79c5e02ebe6
1.safePoint是程序中的某些位置，线程执行到这些位置时，线程中的某些状态是确定的，在safePoint可以记录OopMap信息，线程在safePoint停顿，虚拟机进行GC。
2.一般会在发生GC，Deoptimization时候使用安全点暂停线程.
3.从线程角度看当线程执行到这些位置的时候说明虚拟机当前的状态是·安全的`，如果有需要，可以在这个位置暂停，比如发生GC时，需要暂停暂停所以活动线程，但是线程在这个时刻，还没有执行到一个安全点，所以该线程应该继续执行，到达下一个安全点的时候暂停，等待GC结束
什么地方可以放safepoint
下面以Hotspot为例，简单的说明一下什么地方会放置safepoint
1、理论上，在解释器的每条字节码的边界都可以放一个safepoint，不过挂在safepoint的调试符号信息要占用内存空间，如果每条机器码后面都加safepoint的话，需要保存大量的运行时数据，所以要尽量少放置safepoint，在safepoint会生成polling代码询问VM是否要“进入safepoint”，polling操作也是有开销的，polling操作会在后续解释。
2、通过JIT编译的代码里，会在所有方法的返回之前，以及所有非counted loop的循环（无界循环）回跳之前放置一个safepoint，为了防止发生GC需要STW时，该线程一直不能暂停。另外，JIT编译器在生成机器码的同时会为每个safepoint生成一些“调试符号信息”，为GC生成的符号信息是OopMap，指出栈上和寄存器里哪里有GC管理的指针
线程如何被挂起
1.如果触发GC动作，VM thread会在VMThread::loop()方法中调用SafepointSynchronize::begin()方法，最终使所有的线程都进入到safepoint。
2.在safepoint实现中Java threads可以有多种不同的状态，所以挂起的机制也不同，一共列举了5中情况：
1、执行Java code

在执行字节码时会检查safepoint状态，因为在begin方法中会调用Interpreter::notice_safepoints()方法，通知解释器更新dispatch table（JVM内部的一个数据结构，保存字节码和机器码）。
当线程在解释模式下执行的时候，让JVM发出请求之后，解释器会把指令跳转到检查safepoint的状态，比如检查某个内存页位置，从而让线程阻塞
2、执行native code

如果VM thread发现一个Java thread正在执行native code，并不会等待该Java thread阻塞，不过当该Java thread从native code返回时，必须检查safepoint状态，看是否需要进行阻塞。
这里涉及到两个状态：Java thread state和safepoint state，两者之间有着严格的读写顺序，一般可以通过内存屏障实现，但是性能开销比较大，Hotspot采用另一种方式，调用os::serialize_thread_states()把每个线程的状态依次写入到同一个内存页中.
当Java线程正在执行native code的时候，这种情况最复杂，篇幅也写的最多。当VM thread看到一个Java线程在执行native code，它不需要等待这个Java线程进入阻塞状态，因为当Java线程从执行native code返回的时候，Java线程会去检查safepoint看是否要block(When returning from the native code, a Java thread must check the safepoint _state to see if we must block)
后面说了一大堆关于如何让读写safepoint state和thread state按照严格顺序执行(serialized)，主要用两种做法，一种是加内存屏障(Memeory barrier)，一种是调用mprotected系统调用去强制Java的写操作按顺序执行（The VM thread performs a sequence of mprotect OS calls which forces all previous writes from all Java threads to be serialized. This is done in the os::serialize_thread_states() call）

3、执行complied code

如果想进入safepoint，则设置polling page不可读，当Java thread发现该内存页不可读时，最终会被阻塞挂起
当JVM以JIT编译模式运行的时候，就是最初说的在编译后代码插入一个检查全局的safepoint polling page，VM thread把它设置为不可读，让Java线程挂起
4、线程处于Block状态

即使线程已经满足了block condition，也要等到safepoint operation完成，如GC操作，才能返回。
当线程本来就是阻塞状态的时候，采用了safe region的方式，处于safe region的代码只有等到被允许的时候才能离开safe region.
5、线程正在转换状态

会去检查safepoint状态，如果需要阻塞，就把自己挂起。
当线程处在状态转化的时候，线程会去检查safepoint状态，如果要阻塞，就自己阻塞了
最终实现

当线程访问到被保护的内存地址时，会触发一个SIGSEGV信号，进而触发JVM的signal handler来阻塞这个线程.
那么线程到底是如何自己就阻塞了呢？在第2条的时候说了JVM可以使用mprotect 系统调用来保护一些所有线程可写的内存位置让他们不可写，当线程访问到这些被保护的内存位置时，会触发一个SIGSEGV信号,从而可以触发JVM的signal handler来阻塞这个线程
JVM要阻塞全部的Java线程的时候，要先检查所有的Java线程所处的状态，通过mprotect系统调用来保护一块全局的内存区域，然后让Java线程进入安全点去polling这个内存位置，当线程访问到这个forbidden内存位置的时候会触发JVM的signal handler来阻塞线程。
线程如何恢复

有了begin方法，自然有对应的end方法，在SafepointSynchronize::end()中，会最终唤醒所有挂起等待的线程，大概实现如下：
1、重新设置pooling page为可读
2、设置解释器为ignore_safepoints
3、唤醒所有挂起等待的线程
进入safePoint 慢的影响

一个大概率的原因是当发生GC时，有线程迟迟进入不到safepoint进行阻塞，导致其他已经停止的线程也一直等待，VM Thread也在等待所有的Java线程挂起才能开始GC，这里需要分析业务代码中是否存在有界的大循环逻辑，可能在JIT优化时，这些循环操作没有插入safepoint检查。
safepoint的总结
1.主要有GC safepoint和deoptimization safepoint
2.最简洁的定义是“A point in program where the state of execution is known by the VM”即可以被VM知道在这个点是一个执行状态。
3.GC safepoint需要知道在哪个程序位置上，调用栈、寄存器等一些重要的数据区域里什么地方包含了GC管理的指针；如果要触发一次GC，那么JVM里的所有Java线程都必须到达GC safepoint；
4.Deoptimization safepoint需要知道在那个程序位置上，原本抽象概念上的JVM的执行状态（所有局部变量、临时变量、锁，等等）到底分配到了什么地方，是在栈帧的具体某个slot还是在某个寄存器里，之类的。如果要执行一次deoptimization，那么需要执行deoptimization的线程要在到达deoptimization safepoint之后才可以开始deoptimize。
5.不同JVM实现会选用不同的位置放置safepoint。以hotspot为例子：在解释器里每条字节码的边界都可以是一个safepoint，因为HotSpot的解释器总是能很容易的找出完整的“state of execution”。
6.而在JIT编译的代码里，HotSpot会在所有方法的临返回之前，以及所有非counted loop的循环的回跳之前放置safepoint。
7.HotSpot的JIT编译器不但会生成机器码，还会额外在每个safepoint生成一些“调试符号信息”，以便VM能找到所需的“state of execution”。
8.为GC生成的符号信息是OopMap，指出栈上和寄存器里哪里有GC管理的指针；
9.为deoptimization生成的符号信息是debugInfo，指出如果要把当前栈帧从compiled frame转换为interpreted frame的话，要从哪里把相应的局部变量、临时变量、锁等信息找出来。
10.还有一种情况是当某个线程在执行native函数的时候。此时该线程在执行JVM管理之外的代码，不能对JVM的执行状态做任何修改，因而JVM要进入safepoint不需要关心它。所以也可以把正在执行native函数的线程看作“已经进入了safepoint”，或者把这种情况叫做“在safe-region里”。JVM外部要对JVM执行状态做修改必须要通过JNI。所有能修改JVM执行状态的JNI函数在入口处都有safepoint检查，一旦JVM已经发出通知说此时应该已经到达safepoint就会在这些检查的地方停下来把控制权交给JVM。
之所以只在选定的位置放置safepoint是因为：

挂在safepoint的调试符号信息要占用空间。如果允许每条机器码都可以是safepoint的话，需要存储的数据量会很大（当然这有办法解决，例如用delta存储和用压缩）
safepoint会影响优化。特别是deoptimization safepoint，会迫使JVM保留一些只有解释器可能需要的、JIT编译器认定无用的变量的值。本来JIT编译器可能可以发现某些值不需要而消除它们对应的运算，如果在safepoint需要这些值的话那就只好保留了。这才是更重要的地方，所以要尽量少放置safepoint
像HotSpot VM这样，在safepoint会生成polling代码询问VM是否要“进入safepoint”，polling也有开销所以要尽量减少。
safeRegion
safepoint只能处理正在运行的线程，它们可以主动运行到safepoint。而一些Sleep或者被blocked的线程不能主动运行到safepoint。这些线程也需要在GC的时候被标记检查，JVM引入了safe region的概念。safe region是指一块区域，这块区域中的引用都不会被修改，比如线程被阻塞了，那么它的线程堆栈中的引用是不会被修改的，JVM可以安全地进行标记。线程进入到safe region的时候先标识自己进入了safe region，等它被唤醒准备离开safe region的时候，先检查能否离开，如果GC已经完成，那么可以离开，否则就在safe region呆在。这可以理解，因为如果GC还没完成，那么这些在safe region中的线程也是被stop the world所影响的线程的一部分，如果让他们可以正常执行了，可能会影响标记的结果
什么是OopMap
1.JVM维护了一个专门的映射表（OopMap）记录哪些位置存放着对象引用，来快速完成根节点枚举过程。
2.为每一个操作记录OopMap不现实，HotSpot虚拟机引入了safePoint。
3.safePoint是程序中的某些位置，线程执行到这些位置时，线程中的某些状态是确定的，在safePoint可以记录OopMap信息，线程在safePoint停顿，虚拟机进行GC。
使用例子：JVM 通过JIT编译器为每个safepoint生成一些调试符号信息---OopMap（栈上和寄存器里哪里有GC管理的指针）。
jvm在安全点更新OopMap中的引用，而不需要一发生改变就去更新这个映射表，这也就是为什么我们说在安全点我们的状态是确定的。
当垃圾回收时，收集线程会对栈上的内存进行扫描，看看那些位置上存储了Reference类型。如果发现了某个位置上存储的是Reference类型，就意味着这个引用所指向的对象在这一次垃圾回收过程中不能够回收。
栈上的本地变量表里面只有一部分数据是Reference类型的，为了避免每次都扫描整个栈，所以采用空间换时间的策略。在某个时候(安全点)把栈上代表引用的位置记录下来，GC时直接读取，避免了全部扫描
RememberedSet
新生代 GC（发生得非常频繁）。一般来说， GC过程是这样的：首先枚举根节点。根节点有可能在新生代中，也有可能在老年代中。这里由于我们只想收集新生代（换句话说，不想收集老年代），所以没有必要对位于老年代的 GC Roots 做全面的可达性分析。但问题是，确实可能存在位于老年代的某个 GC Root，它引用了新生代的某个对象，这个对象你是不能清除的。那怎么办呢？
事实上，对于位于不同年代对象之间的引用关系，虚拟机会在程序运行过程中给记录下来。对应上面所举的例子，“老年代对象引用新生代对象”这种关系，会在引用关系发生时，在新生代边上专门开辟一块空间记录下来，这就是RememberedSet，RememberedSet记录的是新生代的对象被老年代引用的关系。所以“新生代的 GC Roots ” + “ RememberedSet 存储的内容”，才是新生代收集时真正的 GC Roots

---------------------------
描述特定PC的每个寄存器和帧堆栈插槽是否为：oop-当前帧的GC根


-----------------------------------------
// Get typed locals/expressions
  // FIXME: must figure out whether word swapping is necessary on x86
  public boolean   booleanAt(int slot)   { return (int)get(slot).getInteger() != 0; }
  public byte      byteAt(int slot)      { return (byte) get(slot).getInteger(); }
  public char      charAt(int slot)      { return (char) get(slot).getInteger(); }
  public short     shortAt(int slot)     { return (short) get(slot).getInteger(); }
  public int       intAt(int slot)       { return (int) get(slot).getInteger(); }
  public long      longAt(int slot)      { return VM.getVM().buildLongFromIntsPD((int) get(slot).getInteger(),
                                                                                 (int) get(slot+1).getInteger()); }
  **public OopHandle oopHandleAt(int slot) { return get(slot).getObject(); }**
  public float     floatAt(int slot)     { return Float.intBitsToFloat(intAt(slot)); }
  public double    doubleAt(int slot)    { return Double.longBitsToDouble(longAt(slot)); }

--------------------------------------------------
在Linux系统中，所有的进程是通过链表来管理的，对于每一个进程都有一个进程描述符 task_struct，其中包含了当前进程的所有信息。在系统启动初期链表中只有 init_task 这一个进程，它是内核的第一个进程，由于其在linux系统中的特殊性，可以再内核符号表中查找到它的逻辑地址，然后我们通过遍历进程链表便可以找到目标进程。

Linux中记录内核符号表的文件分为两个部分，首先是静态部分，也就是内核文件映像 vmlinux 部分的符号表，另外一部分是Linux可配置部分的符号表，两者分别对应 /proc/ksyms 和 System.map 这两个文件，其中 System.map 文件记录了 init_task 进程的逻辑地址，想要获取进程的详细信息我们还需要知道 task_struct 的结构，在Linux中，kernel-header 源码通过 dwarfdump 生成的 module.dwarf 文件中会包含很多内核数据结构，这个文件和 System.map 被我们一起打包在 profile 中供 Volatility 框架使用

-------------------------------------------------------
Java对象保存在堆内存中，一个Java对象包含3个部分：对象头、实例数据和对齐填充。
对象头中包含锁状态标志和线程持有的锁状态等标志
-------------------------------------------------------------
1、一个虚拟机中有多少个虚拟机栈？
线程数量个（一个线程一个虚拟机栈）
2、一个虚拟机栈中有多少个栈帧？
方法调用次数个（调用一次方法，创建一个栈帧）
栈帧是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区中的虚拟机栈的基本元素
栈帧中存储了 局部变量表、操作数栈、动态链接（直接地址）、返回地址（恢复现场）、附件信息
----------------------------------------------------------------------------------
HotSpot 虚拟机默认的分配顺序为

longs/doubles ——> ints ——> shorts/chars ——> bytes/booleans ——> oops(Ordinary Object Pointers)

其分配策略是：相同宽度的字段被分配到一起存放，在满足这个前提的条件下，在父类中定义的变量会出现在子类之前。如果 HotSpot 虚拟机的 +XX:CompactFields 参数值为 true（默认为 true)，则子类中较窄的变量也允许插入父类变量的空虚之中，以节省出一点点空间
----------------------------------------------------------------------
一个Java线程对应一个JavaThread->OSThread -> Native Thread
--------------------------------------------------------------------
传统的锁（也就是下文要说的重量级锁）依赖于系统的同步函数，在linux上使用mutex互斥锁，最底层实现依赖于futex，关于futex可以看我之前的文章，这些同步函数都涉及到用户态和内核态的切换、进程的上下文切换，成本较高。对于加了synchronized关键字但运行时并没有多线程竞争，或两个线程接近于交替执行的情况，使用传统锁机制无疑效率是会比较低的。
---------------------------------------------------------------------
synchronized的偏向锁、轻量级锁、重量级锁是通过Java对象头实现的

synchronized锁过程：

检测Mark Word中是不是当前线程ID，如果是，表示当前线程处于偏向锁；
如果不是，CAS将当前线程ID替换Mark Word，如果成功则表示当前线程获得偏向锁，置偏向标志为1；
如果失败，则说明有竞争，撤销偏向锁，升级而轻量级锁；
当前线程使用CAS将对象头的Mark Word替换为锁记录指针，如果成功，当前线程获得锁；
如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁；
如果自旋成功依然是轻量级状态
如果自旋失败，则升级为重量级锁。
---------------------------------------------------------------------
![](https://user-images.githubusercontent.com/12162133/51801033-67d35f80-2273-11e9-92f1-6ae082d3065e.png)

------------------------------------------------------------------
Thread State
NEW：初始状态，线程被创建，还未调用 start() 方法
RUNNABLE：运行状态，调用 start() 方法后。在 Java 线程中，将操作系统的就绪和运行统称为运行状态
BLOCKED：阻塞状态，线程等待进入 synchronized 代码块和方法中，等待获取锁
WAITING：等待状态，线程可调用 wait、join 等操作使得自己陷入等待状态
TIMED_WAITING：超时等待，线程调用 sleep(timeout)、wait(timeout) 后，处于这个状态
TERMINATED：线程退出，终止状态

![](https://user-images.githubusercontent.com/12162133/51788205-4a8c8b80-21b6-11e9-8965-faacb0bd150c.png)

--------------------------------------------------------
Wait Sets and Notification
每个对象，都关联一个监视器 monitor，monitor 有一个属性 wait set，wait set 包含一组线程。当对象刚刚创建时，wait set 是空的，wait set 中添加线程和移除线程都是原子性操作，通过 Object.wait()、Object.notify()、Object.notifyAll() 可以操作 wait set。
wait set 还可以通过线程中断状态，以及线程中断方法操作。除此之外，Thread.sleep()、Thread.join() 可以操作。

Wait
调用 wait()、wait(Long millisecs)、wait(long millisecs, int nanosecs) 方法。

wait method
当前线程必须拥有该对象的 monitor，调用该方法，当前线程释放该对象的 monitor，线程进入对象 monitor 的 wait set 里，并等待另外一个线程通知，在此对象上执行 notify() 或者 notifyAll()，然后线程等待直到重新获取监视器的锁权限，然后恢复执行。如果当前线程未拥有该对象的 monitor 则抛出异常，如果其他线程中断了当前线程则抛出异常。但是这个方法应该在一个循环中使用，这是为什么呢？
（1）一个对象锁可能用于保护多个状态变量
例如某个对象锁用于保护两个状态 stateA 和 stateB，当 stateA 条件不成立的时候发生 wait，当 stateB 条件不成立的时候发生 wait，两个线程加入到 obj 的 wait set 中，现在若改变 stateA 的操作发生，obj.notifyAll() 方法，obj wait set 里面的所有线程都被唤醒，之前 stateA 的某个线程条件已经成立，可以执行完，但是 stateB 的某个线程条件仍然不成立，仍然需要等待，所以加上循环可以避免这个行为。
（2）多个线程 wait 同一个状态
例如 BlockingQueue 场景，当前队列是空的，多个线程从里面获取元素，于是就 wait 了；此时，另外一个线程往队列里面添加元素，并调用 notifyAll() 方法，此时唤醒了所有线程，但是只有一个线程才能获取到这个元素，其他线程仍然需要等待
（3）虚假唤醒场景
Notification
先分析下 notify() 方法，唤醒正在此对象等待的单个线程的监控，如果有任何线程在此对象上等待，则为其中之一选择唤醒，选择是任意的。唤醒的线程将无法继续得到这个对象上的锁，直到这个线程释放该对象上的锁。唤醒的线程将和其他可能的线程竞争，唤醒的线程将和其他可能的线程等价竞争。
再分下下 notifyAll() 方法，唤醒所有正在此对象监视器上等待的线程，被唤醒的线程将无法继续得到这个对象上的锁，直到这个线程释放该对象上的锁，唤醒的所有线程将会继续竞争这个对象上的锁。

Interruptions
如果这个线程调用 wait()、join()、sleep() 、可中断通道 IO 方法阻塞的时候，其他线程调用了该线程的 interrupt() 方法，则会先检查该线程的中断标志位为 true，则会抛出 InterruptedException 异常，抛出异常后立即会清除自身的中断状态，即重新设置为 false。如果在线程方法里没有调用上述方法，如果有其他线程调用该线程 interrupt() 方法，则会设置线程的中断标志位为 true。至于中断线程是否感知标志位或者感知到标志位怎么处理全凭中断线程说了算。

------------------------------------------------------------

如果有多个线程操作，可能会出现并发问题，JMM 主要是围绕着如何在并发过程中如何处理原子性、可见性、有序性。

原子性：对基本数据类型的读取和赋值操作是原子性操作
可见性：利用 volatile 提供可见性
有序性：JMM 允许编译器和处理器对指令重排，但是规定了 as-if-serial 语义。JMM 也通过 happens-before 支持有序性。


Volatile
在多线程并发编程中 synchronized 和 volatile 都很重要，volatile 是轻量级的 synchronized，一旦一个共享变量（类的成员变量、类的静态成员变量、数组）被 volatile 修饰后，它在多处理器开发中保证了共享变量的可见性，可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值；还有一个特性是禁止指令重排。

将当前处理器缓存行的数据会写回到系统内存。
这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效。
处理器为了提高处理速度，不直接和内存进行通讯，而是先将系统内存的数据读到内部缓存（L1,L2或其他）后再进行操作，但操作完之后不知道何时会写到内存，如果对声明了Volatile变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。
Lock前缀指令会引起处理器缓存回写到内存
Lock前缀指令导致在执行指令期间，声言处理器的 LOCK# 信号，但在P6和最近的处理器中，如果访问的内存区域已经缓存在处理器内部，则不会声言LOCK#信号。相反地，它会锁定这块内存区域的缓存并回写到内存，并使用缓存一致性机制来确保修改的原子性，此操作被称为“缓存锁定”，缓存一致性机制会阻止同时修改被两个以上处理器缓存的内存区域数据。
一个处理器的缓存回写到内存会导致其他处理器的缓存无效
IA-32处理器和Intel 64处理器使用MESI（修改，独占，共享，无效）控制协议去维护内部缓存和其他处理器缓存的一致性。在多核处理器系统中进行操作的时候，IA-32 和Intel 64处理器能嗅探其他处理器访问系统内存和它们的内部缓存。它们使用嗅探技术保证它的内部缓存，系统内存和其他处理器的缓存的数据在总线上保持一致。例如在Pentium和P6 family处理器中，如果通过嗅探一个处理器来检测其他处理器打算写内存地址，而这个地址当前处理共享状态，那么正在嗅探的处理器将无效它的缓存行，在下次访问相同内存地址时，强制执行缓存行填充。
为了实现 volatile 的可见性，通过内存屏障来禁止指令重排：

第二个操作是 volatile 写时，不管第一个操作是什么，都不能重排序（保证 volatile 写之前的操作不会被编译重排到 volatile 写之后）
第一个操作是 volatile 读时，不管第二个操作是什么，都不能重排序（保证 volatile 读之后的操作不会被编译器重排到 volatile 读之后）
第一个操作是 volatile 读时，第二个操作是 volatile 读时，不能重排。
为了实现 volatile 的语义的时候，编译器在生成字节码时候，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

在每个 volatile 写操作的前面插入一个StoreStore屏障
在每个 volatile 写操作的后面插入一个StoreLoad屏障
在每个 volatile 读操作的后面插入一个LoadLoad屏障
在每个 volatile 读操作的后面插入一个LoadStore屏障

-----------------------------------------------
volatile本身不保证获取和设置操作的原子性，仅仅保持修改的**可见性**。但是java的内存模型保证声明为volatile的long和double变量的get和set操作是原子的
