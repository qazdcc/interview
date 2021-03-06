[TOC]
#Java内存区域与溢出异常

溢出异常主要分为StackOverflowError和OutOfMemoryError,StackOverflowError主要发生在Java虚拟栈中，对于无法动态扩展栈深度的虚拟机来说，如果线程请求的栈深度大于JVM所允许的深度，将会抛出这个异常。对于可以扩大虚拟机栈深度的JVM来说，如果扩展时无法申请到足够的空间，则会抛出OutOfMemoryError
##运行时数据区域
Java虚拟机锁管理的内存将会包括以下几个运行时数据区：程序计数器，Java虚拟栈，本地方法栈，方法区，堆(heap)。
###程序计数器
他可以看做是当前线程所执行的字节码的行号指令器。在任何一个确定的时刻，一个处理器(对于多核处理来说是一个内核)都只会执行一个线程中的指令。因此，为了线程切换后能够恢复到正确的执行位置，每条线程都需要有一个独立的PC。如果执行的是Native方法，这个计数器为空(Undefine)。此内存区域是唯一一个在虚拟机规范中没有规定任何OutOfMemoryError情况的区域。

###Java虚拟机栈
与PC相同，也是私有的，他的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型，每个方法执行的同时都会创建一个栈帧，用于储存局部变量表，操作数栈，动态链表，方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中出入栈的过程。
局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部空间是完全确定的。在方法运行期间不会改变局部变量表的大小。
虚拟机栈会抛出StackOverflowError和OutOfMemoryError。

###本地方法栈
和Java虚拟栈类似，执行的是本地方法

###Java堆
Java堆是被所有线程共享的一块内存区域，几乎所有的对象实例都在这里分配内存。回收细节会在GC算法中描述。
主流虚拟机的Java堆都是被设计成可扩展的(通过-Xmx和-Xms控制)。如果在堆中没内存完成实例分配，并且堆也无法再扩展时，会抛出OutOfMemoryError异常。

###方法区
与Java堆一样，是各个线程共享的内存区域，他用于存储已被虚拟机加载的类信息，常亮，静态变量，即时编译器编译后的代码等数据。

###运行时常量池
Runtime Constant Pool是方法区的一部分。Class文件中除了有类的版本，字段，方法，接口等描述信息外，还有一项是常量池(Constant Pool Table）用于存放编译期间生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池存放。
常量池具有动态性，并非常量只有在编译期才能产生，运行期间也能将新的常量放入常量池中，如String类的intern()方法。

###直接内存
Direct Memory并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域，但是这部分内存也频繁地被使用，也会引发OutOfMemoryError。
在JDK1.4中新加入了NIO，引入了一种基于通道(Channel)和缓冲区(Buffer)的I/O方式，可以使用Native函数库直接分配堆外内存，然后通过这个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，这样能在一些场景中避免在Java堆和Native堆中来回复制数据。

##对象的创建
虚拟机遇到一条New指令时，首先将去检查这个符号引用代表的类是否已经被加载，解析和初始化过，如果有，那必须先执行相应的类加载过程。
类加载检查通后，虚拟机将为新生对象分配内存，对象书序内存的大小在类加载完成后便可完全确定，为对象分配空间的任务等同于把一块确定大小的内存从Java堆中划分出来。
需要考虑的是，对象创建在虚拟机中是非常频繁地行为，即使仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的。
解决这个问题有两种方案
- 对分配内存空间的动作进行同步处理
- 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲(Thread Local Allocation Buffer，TLAB)。只有TLAB用完并分配新的TLAB时，才需要同步锁定。虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB参数来设定。

内存分配完成后，虚拟机将分配到的内存空间都初始化为零值，保证对象的实例字段在Java中不赋初值就可以直接使用，程序能访问到这些字段的数据类型所对应的零值。


##对象的内存布局
在HotSpot虚拟机中，对象在内存中存储的布局可以分为3块区域，对象头，实例数据，和对齐填充。

##对象的访问定位
建立对象是为了访问对象，Java程序需要通过栈上的reference数据来操作堆上的具体对象。主流的访问方式有使用句柄和直接指针两种。
- 使用句柄，在堆中会划分出一块内存来作为句柄池，reference中存储的信息就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。
- 使用指针，那么Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息。而reference中存储的直接就是对象地址。


#GC与内存分配策略
三个问题
- 哪些内存需要回收
- 什么时候回收
- 如何回收

##对象是否该被回收
###引用计数法(Reference Counting)
该算法实现简单，效率也很高，但是无法解决对象循环引用的问题。

###可达性分析算法
通过一系列的称为GC Roots的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的。

###几种引用
- 强引用 只要强引用还存在，垃圾收集器就永远不会回收掉被引用的对象。
- 软引用是用来描述一些还有用但非必须的对象。对于软引用关联的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。
- 弱引用 被弱引用关联的对象只能生存到下一次垃圾收集发生之前，当垃圾收集器工作时，无论当前内存是否足够，都会回收掉之被弱引用关联的对象。
- 虚引用 不会对对象的生存时间构成影响，亦无法通过虚引用来取得一个对象的实例。唯一目的就是在这个对象在收集器回收时能收到一个系统通知。

###回收方法区
在大量使用反射，动态代理，CGLib等ByteCode框架、动态生成JSP以及OSGi这里频繁自定义ClassLoader的场景都需要虚拟机具备类卸载的功能，以保证永久代不会溢出。

##垃圾收集算法
###标记-清除算法 (Mark-Sweep)
首先标记出需要回收的对象，在标记完成后统一回收所有被标记的对象。
主要不足有两个:
- 效率问题 标记和清除的效率都不高
- 空间问题 清除后会产生大量不连续的内存碎片

###复制算法
将可用内存容量划分为大小相等的两块，每次只使用其中的一块，当这一块的内存用完时，就将还存活的对象复制到另一块上面，然后再把已使用过的内存空间**一次清理掉**。

###标记-整理算法
根据老年代的特点，有人提出了一种标记整理(Mark-Compat)算法，标记过程仍然与“标记-清除”算法一样，但后续过程不是直接对可回收对象进行清理，而是让所有存活的对象都想一端移动，然后清理掉端便捷意外的内存。

###分代收算法
在新生代，每次垃圾收集都有大批对象死去，只有少量存活，那就选择复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高，没有额外空间对他进行分配担保，就必须使用***标记-清理***或是***标志整理***算法来进行回收。

##垃圾收集器

//TODO:待补充

##内存分配与回收策略
对象主要分配在新生代的Edne区上，如果启动了本地线程分配缓冲，则按线程优先在TLAB上分配，少数情况下也可能会直接分配在老年代中，分配的规则并不是百分之百固定的。
###对象优先在Eden分配
对象在新生代Eden区分配，当Eden区没有足够内存进行分配时，虚拟机将发起一次Minor GC。
注意:Minor GC会将Eden区存活的对象移动到Survivor区，当Survivor区不足时，只能通过分配担保机制提前转移到老年代去。
大对象直接进入老年代 参数(-XX:PretenureSizeThreshold)，令大于这个设置值的对象直接在老年代分配。

###长期存活的对象进入老年代
如果对象在Eden出生并且经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设置为一，对象在Survivor区每熬过一次Minor GC，年龄就加一岁，当它的年龄增加到一定程度(默认为15岁，可以通过-XX:MaxTenuringThreshold设置)，就会被晋升到老年代中。


###空间分配担保
在发生Minor GC前，***虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，则可以锁Minor GC是安全的，如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试进行一次Minor GC。如果小于，或者HandlePromotionFailure设置为不允许冒险，那这是要改为进行一次Full GC*** JDK6之后的规则变为，***只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行Minor GC，否则将进行Full GC***。


#JVM类加载机制
//TODO:待补充


#Java内存模型
Java内存模型的主要目标是定义程序中各个变量的访问规则，即载虚拟机中将变量存储到内存和从内存取出变量这样的底层细节。此处的变量和Java编程中所说的变量有所区别，它包含了实例字段，静态字段，和构成数组对象的元素，但不包括局部变量与方法参数，因为后者(reference本身在Java栈的局部变量表中)是线程私有的，不会被共享。

Java内存模型规定了所有的变量都储存在主内存种，每条线程还有自己的工作内存，线程的工作内存中保存了被该线程使用到的变量的主内存副本拷贝，线程对变量的所有操作必须在工作内存中进行，而不能直接读写主内存中的变量，不同的线程之间也无法访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成。 

##volatile总结
锁提供了两种主要特性，互斥和可见性。
- 互斥即一次只允许一个线程持有某个特定的锁，因此可以使用该特性实现对共享数据的访问，一次只有一个线程可以访问该共享数据。
- 可见性较为复杂，必须保持释放锁之前对共享数据做出的更改对后一个使用该数据的线程是可见的，如果没有同步机制提供这种可见性保证，线程看到的该变量可能是修改前的值或者不一致的值，这将引发许多问题。

volatile具有synchronized的可见性，但是不具备原子性，线程可以自动发现volatile的最新值。volatile变量可以提供线程安全，但是只能应用于非常有限的一种用例：多个变量之间或者某个变量的当前值和修改后的值之间没有约束。
- 可见性。对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
- 原子性：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性。
（监视器锁）happens before。

##volatile写-读的内存语义

volatile写的内存语义如下：
当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存。

volatile读的内存语义如下：
当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。


##volatile内存语义的实现

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能，为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略：

- 在每个volatile写操作的前面插入一个StoreStore屏障。
- 在每个volatile写操作的后面插入一个StoreLoad屏障。
- 在每个volatile读操作的后面插入一个LoadLoad屏障。
- 在每个volatile读操作的后面插入一个LoadStore屏障。

volatile不会像锁一样造成阻塞，因此，在能够安全使用volatile的前提下，他可以提供一些优于锁的可伸缩特性。


###已有的Volatile优化
LinkedTransferQueue，在使用volatile时，使用了追加字节的方式来优化队列出队和入队性能。
LinkedTransferQueue使用一个内部类类型来定义队列的头队列（Head）和尾节点（tail），而这个内部类PaddedAtomicReference相对于父类AtomicReference只做了一件事情，就将共享变量追加到64字节。我们可以来计算下，一个对象的引用占4个字节，它追加了15个变量共占60个字节，再加上父类的Value变量，一共64个字节。
如果头结点和尾结点都不足64个字节的话，处理器会将他们都读到同一个高速缓存行之中，在多处理器下每个处理器都会缓存同样的头尾结点，当一个处理器试图修改头结点的时候会将整个缓存行锁定，在缓存一致性的作用下，会使其他处理器都不能访问高速缓存中的尾结点，而队列的出队入队需要不停地修改头尾结点，所以在多处理器的情况下会严重影响效率，将64个字节填满缓存行，避免头尾结点位于同一个缓存行，使得头尾结点在修改时不会相互锁定。

#Java与线程

##线程的实现
每个已经执行start()且还未结束的Thread类就代表了一个线程。Thread关键方法都是声明成Native的。
实现线程主要有三种方式
- 使用内核线程实现
- 使用用户线程实现
- 使用用户线程加轻量级进程混合实现

####使用内核线程实现
内核线程就是直接由操作系统内核支持的线程，这种线程由内核来完成切换，内核通过调度器对线程进行调度，并负责将线程的任务映射到各个处理器上。程序一般不会去使用内核线程，而是使用内核线程的一种高级接口，轻量级进程(Light Weight Process)。

####使用用户线程实现
从广义上讲，一个线程不是内核进程就可以认为是用户线程。因此，从这个定义上来讲，轻量级进程就属于用户线程，但轻量级进程的实现始终是基于内核的，许多操作都需要系统调用，效率会受到限制。
而狭义的用户线程指的是完全建立在用户空间的线程库上。系统内核不能感知线程存在的实现。用户线程的建立同步销毁和调度完全在用户态中完成，不需要内核的帮助。

####使用用户线程加轻量级进程混合实现
用户线程的建立同步销毁和调度操作依然廉价，并且可以支持大规模的用户线程并发。而操作系统提供支持的轻量级进程则作为用户线程和内核线程之间的桥梁，这样可以使用内核提供的线程调度功能以及处理器映射。

##状态转换
Java定义了5种线程状态，在任意一个时间点，一个线程有且只有其中的一种状态，如下
- 新建
- 运行
- 无限期等待
- 限期等待
- 阻塞
- 结束

#线程安全
##Java中的线程安全
- 不可变(Immutable)
- 绝对线程安全
- 相对线程安全：通常意义上的线程安全，如Vector,HashTable,Collections的synchornizedCollection()方法包装的集合。
- 线程兼容：大部分类属于这种情况，如ArrayList,HashMap等
- 线程对立：无论是否采用了同步，都无法在多线程环境中并发使用，如Thread的suspend()和resume()方法，System.setIn(),System.SetOut(),System.runFinalizerOnExit()等

##线程安全的实现方法
###互斥同步
同步是指在多个线程并发访问共享数据时，保证共享数据在同一个时刻只被一个(或者是一些，信号量时)线程使用，而互斥是实现同步的一个手段，临界区(Cirtical Section)，互斥量(Mutex)和信号量(Semaphore)都是主要的互斥实现方式。
Java中，最基本的互斥同步手段就是synchornized关键字，关键字经过编译后，会在同步块的前后分别行程monitorenter和monitorexit这两个字节码指令。这两个字节码都需要一个reference类型的参数对象来指明要锁定和解锁的对象。如果Java程序的synchornized明确指定了参数对象，那就是这个对象的reference，如果没有明确指定，那就根据synchronized修饰的是实例方法还是类方法，去去对于的对象实例或者`Class`对象来作为锁对象。
在执行monitorenter时，要先尝试获取对象的锁，如果这个对象没有被锁定，或者当前线程已经拥有了这个对象的锁，把锁的计数器加1，相应的，在执行monitorexit指令时，会将计数器减1，当计数器为0时，锁就被释放。如果获取对象失败，那么当前线程就阻塞等待，直到对象锁被另一个线程释放为止。
###非阻塞同步
基于通途检测的乐观并发策略，通俗地说，就是先进行操作，如果没有其他线程争用共享数据，那操作就成功了，如果共享数据有争用，产生了冲突，那就再采取别的补偿措施，这种乐观的并发策略许多实现都不需要挂起线程，因此成为非阻塞同步。
###无同步方案
- 可重入代码：特征，不依赖储存在堆上的数据和公共的系统资源，用到的状态量都由参数传入，不调用非可重入的方法等。
- 线程本地储存：比如`ThreadLocal`。每一个线程的Thread对象中都有一个ThreadLoaclMap对象，这个对象储存了以`ThreadLocal.threadLocalHashCode`为键，以本地线程变量为值的键值对，`ThreadLocal`对象就是当前线程`ThreadLocalMap`的访问入口，每一个`ThreadLocal`对象都包含了一个独一无二的`threadLocalHashCode`值，使用这个值就可以在键值中找回对应的本地线程变量。

##锁优化
###自旋锁与自适应自旋
互斥同步对性能最大的影响是阻塞的实现，挂起线程和恢复线程的操作都需要转入到内核态中完成，这些操作给系统的并发带来了很大的压力。很多时候，共享数据的锁定状态只会持续一段很短的时间，为了这段时间去挂起恢复线程并不值得，所以可以让等待的线程稍等一下，进行一个忙循环(自旋)，结束后就可以获得已经没有处于锁定状态的共享数据。
###锁消除
将其他线程无法到方法内的锁消去
###锁粗化
如果一个连续的操作都在对一个对象进行反复的加锁解锁，那么就可以在这个操作的外部进行加锁解锁，这样只需要加锁一次就可以了。
###轻量级锁
轻量级锁是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。具体实现有关对象头和CAS操作

###偏向锁
目的是消除数据在无竞争情况下的同步原语，进一步提高同步性能。