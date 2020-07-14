# final 关键词
final指的是引用的不可变性，即只能指向初始化的那个对象，而不关心对象的内容变化，因此，***final修饰的变量必须被初始化***。
## final方法
子类不能重写final方法，但是可以使用。
## final参数
用来表示参数在函数体内不允许被修改
## final类
final类不能被继承，所有方法不能重写。但不意味着final类内部的成员变量不可修改。
# volatile
volatile是一个类型修饰符，用于修饰被不同线程访问和修改的变量。系统每次都直接从内存中获取volatile变量，而不是利用缓存。   
volatile不保证原子性。
## 两大特性
1. volatile变量的更新对所有线程可见。
2. 禁止指令重排序   
当程序执行到volatile变量的读或写时，在其前面的操作肯定全部已经进行，且结果已经对后面的操作可见；在其后面的操作肯定还没有进行；
在进行指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。
## 使用场景
1. 对变量的写操作不依赖于当前值。
2. 该变量没有包含在具有其他变量的不变式中。   
实际上，这些条件表明，可以被写入 volatile 变量的这些有效值独立于任何程序的状态，包括变量的当前状态。
## 为什么不保证原子性
比如自增操作，自增分为三步骤：读取变量的原始值、+1操作、写入主存，也就是说，这三个子操作可能会割开执行：   
假如volatile修饰的变量x原本为10，现线程A和B同时进行x++   
线程A对变量进行自增，取出变量，阻塞   
线程B对变量进行自增，取出变量，由于线程A仅仅取出变量，没有对变量进行操作，因此不会造成线程B中缓存变量x的缓存行无效，进行x++后x变为11，写入内存   
线程A因为已经读取出来，已经过了取变量这一步，此时会直接进行x++，x为11，写入内存
## 底层原理
### happen-before
happen-before原则保证了程序的“有序性”，它规定如果两个操作的执行顺序无法从happens-before原则中推到出来，那么他们就不能保证有序性，可以随意进行重排序。其定义如下：   
1. 同一个线程中的，前面的操作 happen-before 后续的操作。（即单线程内按代码顺序执行。但是，在不影响在单线程环境执行结果的前提下，编译器和处理器可以进行重排序，这是合法的。换句话说，这一是规则无法保证编译重排和指令重排）。
2. 监视器上的解锁操作 happen-before 其后续的加锁操作。（Synchronized 规则）
3. 对volatile变量的写操作 happen-before 后续的读操作。（volatile 规则）
4. 线程的start() 方法 happen-before 该线程所有的后续操作。（线程启动规则）
5. 线程所有的操作 happen-before 其他线程在该线程上调用 join 返回成功后的操作。
6. 如果 a happen-before b，b happen-before c，则a happen-before c（传递性）。
### 内存屏障（Meomory Barrier）
内存屏障又称内存栅栏，是一个CPU指令，有两个作用：
1. 保证特定操作的执行顺序；
2. 保证某些变量的内存可见性（利用该特性实现volatile的内存可见性）
由于编译器和处理器都能执行指令重排优化。如果在指令间插入一条内存屏障，则会告诉编译器和处理器，不管什么指令都不能和这条内存屏障指令重排序。也就是说通过插入内存屏障禁止在内存屏障前后的指令执行重排序优化。内存屏障的另一个作用是强制刷出CPU的缓存数据，因此CPU上的线程都能读取到数据的最新版本。
#### volatile写操作
会在写操作后加入一条store屏障指令，将工作内存中的共享变量值刷新回主存。
#### volatile读操作
会在读操作前加入一条load屏障指令，从主存中读取共享变量。

### volatile实现原理
计算机在运行程序时，每条指令都是在CPU中执行的，在执行过程中势必会涉及到数据的读写。我们知道程序运行的数据是存储在主存中，这时就会有一个问题，读写主存中的数据没有CPU中执行指令的速度快，如果任何的交互都需要与主存打交道则会大大影响效率，所以就有了CPU高速缓存。CPU高速缓存为某个CPU独有，只与在该CPU运行的线程有关。

有了CPU高速缓存虽然解决了效率问题，但是它会带来一个新的问题：数据一致性。在程序运行中，会将运行所需要的数据复制一份到CPU高速缓存中，在进行运算时CPU不再也主存打交道，而是直接从高速缓存中读写数据，只有当运行结束后才会将数据刷新到主存中。

有volatile修饰的共享变量进行写操作的时候会多出Lock前缀的指令，该指令在多核处理器下会引发两件事情。
* 将当前处理器缓存行数据刷写到系统主内存。
* 这个刷写回主内存的操作会使其他CPU缓存的该共享变量内存地址的数据无效。
这样就保证了多个处理器的缓存是一致的，对应的处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器缓存行设置无效状态，当处理器对这个数据进行修改操作的时候会重新从主内存中把数据读取到缓存里。


# wait() notify() notifyAll() sleep() yeild() join()
1. wait和notify用于多线程协调运行，在synchronized内部可以调用wait()使线程进入等待状态；
2. 必须在已获得的锁对象上调用wait()方法；
3. 在synchronized内部可以调用notify()或notifyAll()唤醒其他等待线程；
4. 必须在已获得的锁对象上调用notify()或notifyAll()方法；
5. 已唤醒的线程还需要重新获得锁后才能继续执行;
6. 使用notifyAll()将唤醒所有当前正在this锁等待的线程，而notify()只会唤醒其中一个（具体哪个依赖操作系统，有一定的随机性）。通常来说，notifyAll()更安全。有些时候，如果我们的代码逻辑考虑不周，用notify()会导致只唤醒了一个线程，而其他线程可能永远等待下去醒不过来了。
7. sleep()方法是Thread类的静态方法，而wait()是Object类的方法，用于线程间通信。sleep()不释放锁，到时间自动恢复执行。
8. 调用yield()方法的线程告诉虚拟机它乐意让其他线程占用自己的位置。yield告诉当前正在执行的线程把运行机会交给线程池中拥有相同优先级的线程。yield不能保证使得当前正在运行的线程迅速转换到可运行的状态，它仅能使一个线程从运行状态转到可运行状态，而不是等待或阻塞状态。
9. 线程实例的join()方法可以使得一个线程在另一个线程结束后再执行。如果join()方法在一个线程实例上调用，当前运行着的线程将阻塞直到这个线程实例完成了执行。

# 类加载过程
## 加载
由类加载器完成，做的事情有3个：
1. 根据权限定名获取此类的二进制字节流；
2. 将这个字节流代表的静态存储结构转化为方法区的运行时数据结构；
3. 在内存中生成一个代表这个类的Class对象，作为方法区这个类的各种数据的访问入口。
## 验证
检查待加载的class文件的正确性，目的是为了确保class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。格式检查（是否是class文件、） 元数据检查（是否继承了final的类，是否实现了接口中的所有方法） 符号引用验证（访问性 private protected）
## 准备
在方法区内为类中的静态变量分配存储空间。
准备阶段是正式为类变量(被static修饰的变量)分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中分配。
## 解析
将虚拟机常量池中的符号引用替换为直接引用的过程。

符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时可以无歧义地定位到目标即可。符号引用和虚拟机实现的内存布局无关。
直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用和虚拟机实现的内存布局有关。
## 初始化
如果该类具有超类，则对其初始化，执行静态初始化变量和静态初始化代码块。

# 类加载的分类
1. 隐式加载指的是程序使用new关键字创建对象，则会隐式地调用类的加载器把对应的类加载到JVM中。
2. 显示加载指的是直接通过使用class.forName()方法来把所需的类加载到JVM中。

# 类加载器
在Java语言中，类的加载是动态的，不会一次性加载全部加载再运行，而是先把保证程序运行的基础类（如基类）完全加载到JVM中，其他类则是在需要时加载。从Java虚拟机的角度看，只存在两种不同的类加载器：
1. 启动类加载器，C++语言实现，是虚拟机自身的一部分；
2. 所有的其他的类加载器，Java语言实现，独立于虚拟机外部，并且全部都继承自抽象类java.lang.ClassLoader.
从Java开发人员的角度来看，类加载器可以划分为启动类加载器、扩展类加载器和应用程序类加载器。
1. 启动类加载器(Bootstrap ClassLoader)：这个类主要负责将存放在\<JAVA_HOME\>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中，并且可以被虚拟机识别的(如rt.jar)类库加载到虚拟机内存中。由于引导类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作。
2. 标准扩展(Extension)类加载器：是由Sun的ExtClassLoader (sun.misc.Launcher $ExtClassLoader)实现的，它负责将\<JAVA_HOME\>\lib\ext或者由系统变量java.ext.dirs指定位置中的类库加载到内存中，开发者可以直接使用标准扩展类加载器。
3. 应用程序类加载器(Application ClassLoader)：这个类加载器由sun.misc.Launcher $ApplicationClassLoader实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以一般也被称作是系统类加载器。它负责加载用户类路径(ClassPath)上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

# 自定义类加载器
## 自定义的好处
1. 加密：.class文件可以轻易的被反编译，如果你需要把自己的代码进行加密以防止反编译，可以先将编译后的代码用某种加密算法加密，类加密后就不能再用Java的ClassLoader去加载类了，这时就需要自定义ClassLoader在加载类的时候先解密类，然后再加载
2. 从非标准的来源加载代码：如果你的.class文件是放在数据库、甚至是在云端，就可以自定义类加载器，从指定的来源加载类。
## 自定义的实现方式
继承ClassLoader，并覆盖findClass方法。defineClass方法可以把二进制流字节组成的文件转换为一个java.lang.Class（只要二进制字节流的内容符合Class文件规范）。


# 双亲委派模型
双亲委派模型的工作过程：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是会把这个加载的请求委派给父类加载器去完成，每一个层次的类加载器都是这么做，只有当父类加载器无法完成类的加载工作时，子类加载器才会尝试自己去加载这个类。

使用双亲委派模型来组织类加载器之间的关系的一个好处是就是Java类随着它的加载器一起具备了一种带有优先级的层次关系，保证了java程序的稳定性。

# JVM 
## Java运行时数据区域
### 程序计数器
当前线程所执行的字节码的行号指示器。Pc(Program Counter Register)，正在执行的指令，当正在执行的是native方法是计数器为空。
### Java虚拟机栈
存放基本数据类型和引用变量。为执行java方法服务，描述的是Java方法执行的内存模型。

栈内存的管理是通过压栈和弹栈来实现的，以栈帧(每个方法开始运行时都会创建一个栈帧，栈帧是用于存储局部变量表、操作数栈、动态链接、方法出口等信息)为单位来进行管理方法之间的调用关系，当有方法调用时，会通过压栈方式进行创建新的栈帧，当方法调用结束时会通过弹栈的操作释放栈帧。
### 本地方法栈   
为虚拟机使用到的native本地方法服务。
### Java堆
几乎所有的对象实例以及数组都在堆上分配。线程之间共享堆内存，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例。

Java堆是垃圾收集器管理的主要区域，因此很多时候也被称为是”GC”堆。从内存分配的角度看，线程共享的Java堆可能划分出多个线程私有的分配缓冲区(Thread Local Allocation Buffer,TLAB)。从内存回收的角度来看，由于现在收集器基本都采用分代收集，所以Java堆还可以细分为：新生代和老年代；再细致一点新生代可以分为Eden空间、From Survivor空间、To Survivor空间等。不过无论如何划分，都与存放的内容无关，无论哪个区域，存放的都仍然是对象实例。

在实现时，堆可以实现成固定大小的，也可以实现成可以扩展的(当前主流虚拟机中都是按照可扩展实现的)。Java堆在物理上不要求存储在连续的内存空间中。
### 方法区
线程共享，用于存储已被虚拟机加载的类信息、常量、静态变量等。有时方法区也被称作永久代(但是本质上两者并不等价，仅仅因为是HotSpot虚拟机的设计团队选择把GC分代收集扩展至方法区，或者说使用永久代来实现方法区而已)，Java虚拟机规范对方法区的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集。方法区的内存回收目标主要是针对常量池的回收和对类型的卸载。

运行时常量池是方法区的一部分，用于存放编译期生成的各种字面量和符号引用，这些内容在类加载后进入方法区的运行时常量池中存放。当然并非class文件常量池中的内容才能进入运行时常量池，在运行期间也可将新的常量放入运行时常量池中，如String的intern方法。

## 运行异常
### StackOverflowError
虚拟机栈、本地方法栈在线程请求的栈深度大于虚拟机所允许的深度时抛出该错误。
### OutOfMemory
虚拟机栈、本地方法栈、Java堆在动态扩展的时候没有申请到足够的内存会抛出该异常。

## GC
### 堆内存的分配
1. 新生代：进一步划分为Eden空间、From Survivor空间、To Survivor空间
2. 老年代

### 垃圾回收算法
1. 标记清除：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。
2. 复制：可用的内存按容量大小分为两块大小相等的两块，每次只使用其中的一块。当某一块的内存使用完了，就把还存活的对象移到另一块内存上，然后把当前这块做清理掉。
3. 标记整理：标记过程与”标记-清除算法“一样，但是后续步骤不是直接对可回收的对象进行清理，而是把所有存活的对象都移到一端，然后直接清理掉端边界以外的内存。
4. 分代收集：根据对象存活周期的不同将内存划分为几块。一般是把堆分为新生代和老年代，在新生代采用复制算法回收内存，老年代采用标记-整理或标记-清除算法。算法在运行的过程中优先收集处于新生代的对象，如果一个对象经过多次收集还存活，那么就可以把这个对象移到高一级的堆里，减少对其扫描的次数。

### GC分类
1. Minor GC：从年轻代空间（包括 Eden 和 Survivor 区域）回收内存。当新生代无法为新生对象分配内存空间的时候，会触发Minor GC。因为新生代中大多数对象的生命周期都很短，所以发生Minor GC的频率很高，虽然它会触发stop-the-world，但是它的回收速度很快。
2. Major GC清理Tenured区，用于回收老年代，出现Major GC通常会出现至少一次Minor GC。
3. Full GC是针对整个新生代、老生代、元空间（metaspace，java8以上版本取代perm gen）的全局范围的GC。Full GC不等于Major GC，也不等于Minor GC+Major GC，发生Full GC需要看使用了什么垃圾收集器组合，才能解释是什么样的垃圾回收。

### 触发Full GC的原因
1. 当年轻代晋升到⽼年代的对象⼤⼩，并⽐⽬前⽼年代剩余的空间⼤⼩还要⼤时，会触发Full GC；
2. 当⽼年代的空间使⽤率超过某阈值时，会触发Full GC；
3. 当元空间不⾜时（JDK1.7永久代不足），也会触发Full GC；
4. 当调⽤System.gc()也会安排⼀次Full GC。

### 可达性分析
用于在GC时判断对象是否存活，主要有两种方法。
#### 引用计数法
每个对象都有一个引用计数器，每当该对象被引用时则计数器加1，当某个引用失效时，则引用计数器减1。任何时刻引用计数器的值为0的对象就是不可能再被引用的。

缺点是无法解决循环引用的问题，现在主流的java虚拟机没有采用引用计数法来管理内存。

#### 可达性分析算法
通过一系列称为”GC Roots“的对象作为起始点，从这些节点开始向下搜索与其相连的对象，搜索过程中经过的路径称为引用链(Reference Chain)，当一个对象到GC Roots没有任何引用链时，则证明该对象是不可用的。

在主流的商用程序语言(Java C#)的主流实现中，都是通过可达性分析算法来判定对象是否存活的。

在可达性分析算法中不可达的对象，也并非是”非死不可的“，要真正宣告一个对象是否死亡，需要经历两次被标记过程。第一次标记判断是否有必要执行finalize()方法，将有必要执行的对象放到F-Queue的队列之中；稍后GC会对F-Queue进行第二次标记，finalize()方法是对象逃脱死亡命运的最后一次机会，如果对象在finalize()方法中成功地与引用链上的任何一个对象相关联，那么在第二次标记时会把这个对象彻底移除”即将回收“的集合。

finalize()方法是Object类的一个方法，在垃圾回收器执行时会调用被回收对象的finalize()方法，可以覆盖此方法来实现对其他资源的回收，比如关闭文件等。需要注意的是，一旦垃圾回收器准备好释放对象占用的空间时，将首先调用其finalize()方法，并且在下一次垃圾回收动作发生时，才会真正回收对象占用的内存。

### GC Roots
在Java语言中，可作为GC Roots的对象主要包括以下四个方面。
1. 虚拟机栈中的引用对象；
2. 本地方法栈中本地方法引用的对象；
3. 方法区中类静态属性引用的对象；
4. 方法区内常量引用的对象。

### GC停顿
因为可达性分析工作必须在一个能确保一致性的快照中进行，这里的“一致性”的意思是指在整个分析期间整个执行系统看起来就像被冻结在某个时间点上，不可以出现分析过程中对象引用关系还在不断变化的情况。不然的话可达性分析的结果就无法得到保证，这是导致GC进行时必须停顿所有Java执行线程的一个重要原因。

### 垃圾回收器
#### Serial收集器
一个单线程的收集器，但它“单线程”的意义并不仅仅说明它只会使用一个CPU或一条收集线程去完成垃圾收集工作。更重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程。新生代收集器、采用复制算法。

优点：与其他收集器的单线程比，简单而高效。

Serial收集器对于运行在Client模式下的虚拟机来说是一个很好的选择。
#### ParNew收集器
其实就是Serival收集器的多线程版本。新生代收集器、采用复制算法。	适用于运行在server模式下的虚拟机中首选的新生代收集器。
#### Parallel Scavenge收集器
新生代收集器、采用复制算法、并行的多线程收集。主要适合在后台运算而不需要太多交互的任务，注重的是吞吐量。

如果用户对于收集器运作不是很了解，手工优化困难的时候，使用Parallel Scavenge收集器配合自适应调节策略，把内存管理的调节任务交给虚拟机去完成将是一个不错的选择。

自适应调节策略也是Parallel Scavenge收集器与ParNew收集器的一个不同。
#### SerialOld收集器
是Serial的老年代版本，同样是一个单线程收集器，使用“标记-整理”算法。主要给在Client模式下的虚拟机使用。如果是在Server模式下则主要有两大用途，一是在JDK1.5以及之前的版本中与Parallel Scavenge 收集器搭配使用，另一种用途就是作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用。
#### Parallel Old收集器
是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。在注重吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge收集器和Parallel Old收集器。
#### CMS(Conourrent Mark Sweep )收集器
以获取最短回收停顿时间为目标的收集器。基于”标记-清除”算法实现。

整个过程分为四步：初始标记、并发标记、重新标记、并发清除。

优点：并发收集、低停顿。

缺点：对CPU资源敏感；无法处理浮动垃圾；收集结束会有大量碎片。
#### G1收集器
面向服务端应用的垃圾收集器，标记整理。与其他GC收集器相比：G1具有如下特点：并发与并行；分代收集；空间整合；可预测的停顿。

运作的过程大致分为四个步骤：初始标记；并发标记；最终标记；筛选回收。

备注：并发(Parallel):指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态；
并行(Concurrent):指用户线程与垃圾收集线程同时执行(但不一定是并行的，可能会交替执行 )，用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。


## 内存泄漏和内存溢出
### 内存泄漏
内存泄露是指一个不再被程序使用的对象或变量还在内存中占用空间。在java语言中，判断一个内存空间是否符合垃圾回收的标准有两个：
1. 给对象赋予了空值null，以后再也没有不会被使用；
2. 给对象赋予了新值，重新分配了内存空间。

一般来说，内存泄露主要有两种情况：
1. 在堆中申请的空间没有被释放；
2. 对象已不再使用但是还仍然在内存中保留着。

垃圾回收机制的引入可以有效地解决第一种情况，但是对第二种情况却没有办法解决，因此java语言中内存泄露主要指的是第二种情况。

在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是可达的，即在有向图中，存在通路可以与其相连；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

在Java语言中，容易引起内存泄露的原因有很多，主要可以分为以下几种。
1. 静态集合类。例如HashMap和Vector，如果这些容器是静态的，那么它们的声明周期与程序一样，在程序结束之前这些容器所占的空间将得不到释放，从而造成内存泄露；
2. 各种连接。比如Connection、Statement、ResultSet等如果使用之后不显示地关闭，会造成大量的对象无法回收，造成内存泄露；
3. 监听器。在java中，往往一个程序使用多个监听器，但是在释放对象的时候却没有删除相应的监听器对象就会导致内存泄露；
4. 变量不合理的作用域。如果一个变量定义的作用范围大于其使用范围 ，就有可能造成内存泄露，另一个方面如果没有及时地把一个对象设置为null，也有可能导致内存泄露；
5. 单例模式可能会造成内存泄露。如果以静态的方式存储单例对象的话，那么它在JVM的整个生命周期中都存在，就会导致内存泄露。

### 内存溢出
内存溢出是指程序要求的内存，超出了系统所能分配的范围，从而发生溢出。

## JVM调优
### 常用JVM命令行工具
#### jps
只运行jps输出的是当前运行的java进程(Java程序的进程ID，Main函数)；   
运行jps –q只输出进程ID，而不输出类的名称；   
jps –m输出传递给Java进程(主函数)的短名称；   
jps –l输出主函数的完整路径；   
jps –v输出传递给JVM的参数。
#### jstat
虚拟机统计信息监视工具；可以用于观察Java应用程序运行时信息的工具。

jstat可以实时显示本地或远程JVM进程中类装载、内存、垃圾收集、JIT编译等数据（如果要显示远程JVM信息，需要远程主机开启RMI支持）。如果在服务启动时没有指定启动参数-verbose:gc，则可以用jstat实时查看gc情况。

在没有GUI图形界面，只提供了纯文本控制台环境的服务器上，jstat是运行期定位虚拟性能问题的首选工具。

如jstat –gc 2764 250 20表示查询进程2764垃圾收集状况，每250ms查询一次，一共查询20次。后面两个参数省略时表示只查询一次。

命令格式为jstat option vmid interval count.

其中option主要分为3类：类装载、垃圾收集、运行期编译。

#### jinfo
Java配置信息工具；用于查询当前运行时的JVM属性和参数的值。
#### jmap
Java内存映像工具；用于显示当前Java堆和永久代的详细信息（如当前使用的收集器，当前的空间使用率等）。

jmap –heap:显示java堆的相信信息，如使用哪种收集器、参数配置、分代状态等。

jmap -dump:生成java堆转储快照。

#### jhat
虚拟机堆转储快照分析工具；用于分析使用jmap生成的dump文件，是JDK自带的工具，使用方法为： jhat -J -Xmx512m [file]

#### jstack
Java堆栈跟踪工具

用于生成当前JVM的所有线程快照，线程快照是虚拟机每一条线程正在执行的方法,目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的原因。线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情或者是在等待什么资源。
 
### JDK可视化工具
#### JConsole
JConsole工具是JDK自带的图形化性能监控工具。通过JConsole工具，可以查看Java应用程序的运行概况，监控堆信息、永久区使用情况、类加载情况。

在JConsole中可以查看堆的详细信息，包括堆的大小、使用率、eden区大小、survivor区大小、永久区大小。

JConsole可以方便地查看系统内的线程信息，并且可以快速定位死锁问题。

JConsole的类页面可以显示系统已经装载的类数量。

JConsole的VM摘要显示了当前Java应用程序的基本信息，如虚拟机类型、虚拟机版本、系统线程信息、操作系统内存信息、堆信息、垃圾回收器信息以及路径信息等。

### JVM调优一般步骤
1. 监控GC的状态
使用各种JVM工具，查看当前日志，分析当前JVM参数设置，并且分析当前堆内存快照和gc日志，根据实际的各区域内存划分和GC执行时间，觉得是否进行优化；
2. 分析结果，判断虚拟机的配置参数是否需要优化   
如果各项参数设置合理，系统没有超时日志出现，GC频率不高，GC耗时不高，那么没有必要进行GC优化；如果GC时间超过1-3秒，或者频繁GC，则必须优化；   
注：如果满足下面的指标，则一般不需要进行GC：   
Minor GC执行时间不到50ms；   
Minor GC执行不频繁，约10秒一次；   
Full GC执行时间不到1s；   
Full GC执行频率不算频繁，不低于10分钟1次；
3. 调整GC类型和内存分配   
如果内存分配过大或过小，或者采用的GC收集器比较慢，则应该优先调整这些参数，并且先找1台或几台机器进行beta，然后比较优化过的机器和没有优化的机器的性能对比，并有针对性的做出最后选择；
4. 不断的分析和调整   
通过不断的试验和试错，分析并找到最合适的参数
5. 全面应用参数   
如果找到了最合适的参数，则将这些参数应用到所有服务器，并进行后续跟踪。

# Lock
## 乐观锁 VS 悲观锁
乐观锁与悲观锁是一种广义上的概念，体现了看待线程同步的不同角度。在Java和数据库中都有此概念对应的实际应用。

对于同一个数据的并发操作，悲观锁认为自己在使用数据的时候一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改。Java中，synchronized关键字和Lock的实现类都是悲观锁。

而乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据的时候去判断之前有没有别的线程更新了这个数据。如果这个数据没有被更新，当前线程将自己修改的数据成功写入。如果数据已经被其他线程更新，则根据不同的实现方式执行不同的操作（例如报错或者自动重试）。

乐观锁在Java中是通过使用无锁编程来实现，最常采用的是CAS算法，Java原子类中的递增操作就通过CAS自旋实现的。

特点：
* 悲观锁适合写操作多的场景，先加锁可以保证写操作时数据正确。
* 乐观锁适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。

悲观锁基本都是在显式的锁定之后再操作同步资源，而乐观锁则直接去操作同步资源。
### CAS
CAS全称 Compare And Swap（比较与交换），是一种无锁算法。在不使用锁（没有线程被阻塞）的情况下实现多线程之间的变量同步。

java.util.concurrent包中的原子类就是通过CAS来实现了乐观锁。

CAS算法涉及到三个操作数：
* 需要读写的内存值 V。
* 进行比较的值 A。
* 要写入的新值 B。
当且仅当 V 的值等于 A 时，CAS通过原子方式用新值B来更新V的值（“比较+更新”整体是一个原子操作），否则不会执行任何操作。一般情况下，“更新”是一个不断重试的操作。

之前提到java.util.concurrent包中的原子类，就是通过CAS来实现了乐观锁，那么我们进入原子类AtomicInteger的源码，看一下AtomicInteger的定义：

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

```
根据定义我们可以看出各属性的作用：
* unsafe： 获取并操作内存的数据。
* valueOffset： 存储value在AtomicInteger中的偏移量。
* value： 存储AtomicInteger的int值，该属性需要借助volatile关键字保证其在线程间是可见的。

CAS虽然很高效，但是它也存在三大问题，这里也简单说一下：
1. ABA问题。CAS需要在操作值的时候检查内存值是否发生变化，没有发生变化才会更新内存值。但是如果内存值原来是A，后来变成了B，然后又变成了A，那么CAS进行检查时会发现值没有发生变化，但是实际上是有变化的。ABA问题的解决思路就是在变量前面添加版本号，每次变量更新的时候都把版本号加一，这样变化过程就从“A－B－A”变成了“1A－2B－3A”。JDK从1.5开始提供了AtomicStampedReference类来解决ABA问题，具体操作封装在compareAndSet()中。compareAndSet()首先检查当前引用和当前标志与预期引用和预期标志是否相等，如果都相等，则以原子方式将引用值和标志的值设置为给定的更新值。
2. 循环时间长开销大。CAS操作如果长时间不成功，会导致其一直自旋，给CPU带来非常大的开销。
3. 只能保证一个共享变量的原子操作。对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量操作时，CAS是无法保证操作的原子性的。   

Java从1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。

## 自旋锁 VS 适应性自旋锁
阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态转换需要耗费处理器时间。如果同步代码块中的内容过于简单，状态转换消耗的时间有可能比用户代码执行的时间还要长。

在许多场景中，同步资源的锁定时间很短，为了这一小段时间去切换线程，线程挂起和恢复现场的花费可能会让系统得不偿失。如果物理机器有多个处理器，能够让两个或以上的线程同时并行执行，我们就可以让后面那个请求锁的线程不放弃CPU的执行时间，看看持有锁的线程是否很快就会释放锁。

而为了让当前线程“稍等一下”，我们需让当前线程进行自旋，如果在自旋完成后前面锁定同步资源的线程已经释放了锁，那么当前线程就可以不必阻塞而是直接获取同步资源，从而避免切换线程的开销。这就是自旋锁。

自旋锁本身是有缺点的，它不能代替阻塞。自旋等待虽然避免了线程切换的开销，但它要占用处理器时间。如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。所以，自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，可以使用-XX:PreBlockSpin来更改）没有成功获得锁，就应当挂起线程。

自旋锁的实现原理同样也是CAS，AtomicInteger中调用unsafe进行自增操作的源码中的do-while循环就是一个自旋操作，如果修改数值失败则通过循环来执行自旋，直至修改成功。
```java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```
自旋锁在JDK1.4.2中引入，使用-XX:+UseSpinning来开启。JDK 6中变为默认开启，并且引入了自适应的自旋锁（适应性自旋锁）。

自适应意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。

## 无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁
这四种锁是指锁的状态，专门针对synchronized的。锁状态只能升级不能降级。
### Synchronized实现线程同步原理
#### Java对象头
1. Mark Word：默认存储对象的HashCode，分代年龄和锁标志位信息。在运行期间MarkWord里存储的数据会随着锁标志位的变化而变化。
2. Klass Point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
#### Monitor
Monitor可以理解为一个同步工具或一种同步机制，通常被描述为一个对象。每一个Java对象就有一把看不见的锁，称为内部锁或者Monitor锁。

Monitor是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联，同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。

synchronized通过Monitor来实现线程同步，Monitor是依赖于底层的操作系统的Mutex Lock（互斥锁）来实
现的线程同步。

四种锁状态对应的的Mark Word：

|  锁状态   | 存储内容  |  存储内容 |
|  :----:  | :----  | :----:|
| 无锁  | 对象的hashcode，对象分代年龄，是否偏向锁（0） |00|
| 偏向锁  | 偏向线程ID，偏向时间戳，对象分代年龄，是否偏向锁（1） |01|
| 轻量级锁  | 指向栈中锁记录的指针 |00|
| 重量级锁  | 指向互斥量（重量级锁）的指针 |10|
### 无锁
无锁没有对资源进行锁定，所有的线程都能访问并修改同一个资源，但同时只有一个线程能修改成功。

无锁的特点就是修改操作在循环内进行，线程会不断的尝试修改共享资源。如果没有冲突就修改成功并退出，否则就会继续循环尝试。如果有多个线程修改同一个值，必定会有一个线程能修改成功，而其他修改失败的线程会不断重试直到修改成功。上面我们介绍的CAS原理及应用即是无锁的实现。无锁无法全面代替有锁，但无锁在某些场合下的性能是非常高的。

### 偏向锁
偏向锁是指一段同步代码一直被一个线程所访问，那么该线程会自动获取锁，降低获取锁的代价。

当一个线程访问同步代码块并获取锁时，会在Mark Word里存储锁偏向的线程ID。在线程进入和退出同步块时不再通过CAS操作来加锁和解锁，而是检测Mark Word里是否存储着指向当前线程的偏向锁。引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令即可。

偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动释放偏向锁。

### 轻量级锁
是指当锁是偏向锁的时候，被另外的线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，从而提高性能。

在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，然后拷贝对象头中的Mark Word复制到锁记录中。

拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock Record里的owner指针指向对象的Mark Word。

如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，表示此对象处于轻量级锁定状态。

如果轻量级锁的更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行，否则说明多个线程竞争锁。

若当前只有一个等待线程，则该线程通过自旋进行等待。但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁升级为重量级锁。

### 重量级锁
升级为重量级锁时，锁标志的状态值变为“10”，此时Mark Word中存储的是指向重量级锁的指针，此时等待锁的线程都会进入阻塞状态。

综上，偏向锁通过对比Mark Word解决加锁问题，避免执行CAS操作。而轻量级锁是通过用CAS操作和自旋来解决加锁问题，避免线程阻塞和唤醒而影响性能。重量级锁是将除了拥有锁的线程以外的线程都阻塞。

## 公平锁VS非公平锁
公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。

非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁。

#### ReentrantLock的公平锁和非公平锁实现
```java
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {...}
    
    static final class NonfairSync extends Sync {...}
        public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
根据代码可知，ReentrantLock里面有一个内部类Sync，Sync继承AQS（AbstractQueuedSynchronizer），添加锁和释放锁的大部分操作实际上都是在Sync中实现的。它有公平锁FairSync和非公平锁NonfairSync两个子类。ReentrantLock默认使用非公平锁，也可以通过构造器来显示的指定使用公平锁。

公平锁的加锁方法：
```java
/**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

非公平锁的加锁方法：
```java
        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

```
通过源代码对比，我们可以明显的看出公平锁与非公平锁的lock()方法唯一的区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors():
```java
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
可以看到该方法主要做一件事情：主要是判断当前线程是否位于同步队列中的第一个。如果是则返回true，否则返回false。

## 可重入锁 VS 非可重入锁
可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。Java中ReentrantLock和synchronized都是可重入锁，可重入锁的一个优点是可一定程度避免死锁。下面用示例代码来进行分析：
```java
    public class Widget{
        public synchronized void doSomething(){
            System.out.println("方法1执行...");
            doOthers();
        }

        public synchronized void doOthers(){
            System.out.println("方法2执行...");
        } 
    } 
```
在上面的代码中，类中的两个方法都是被内置锁synchronized修饰的，doSomething()方法中调用doOthers()方法。因为内置锁是可重入的，所以同一个线程在调用doOthers()时可以直接获得当前对象的锁，进入doOthers()进行操作。
如果是一个不可重入锁，那么当前线程在调用doOthers()之前需要将执行doSomething()时获取当前对象的锁释放掉，实际上该对象锁已被当前线程所持有，且无法释放。所以此时会出现死锁。

ReentrantLock和synchronized都是重入锁，那么我们通过重入锁ReentrantLock以及非可重入锁NonReentrantLock的源码来对比分析一下为什么非可重入锁在重复调用同步资源时会出现死锁。

ReentrantLock和NonReentrantLock都继承父类AQS，其父类AQS中维护了一个同步状态status来计数重入次数，status初始值为0。当线程尝试获取锁时，可重入锁先尝试获取并更新status值，如果status == 0表示没有其他线程在执行同步代码，则把status置为1，当前线程开始执行。如果status != 0，则判断当前线程是否是获取到这个锁的线程，如果是的话执行status+1，且当前线程可以再次获取锁。而非可重入锁是直接去获取并尝试更新当前status的值，如果status != 0的话会导致其获取锁失败，当前线程阻塞。

释放锁时，可重入锁同样先获取当前status的值，在当前线程是持有锁的线程的前提下。如果status-1 == 0，则表示当前线程所有重复获取锁的操作都已经执行完毕，然后该线程才会真正释放锁。而非可重入锁则是在确定当前线程是持有锁的线程之后，直接将status置为0，将锁释放。

```java
    final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

```

```java
    protected boolean tryAcquire(int acquires){
        if(this.compareAndSetState(0,1)){
            this.owner = Thread.currentThread();
            return true;
        }else{
            return false;
        }
    }

    protected boolean tryRelease(int releases){
        if(Thread.currentThread()!=this.owner){
            throw new IllegalMonitorStateException();
        }else{
            this.owner = null;
            this.setState(0);
            return true;
        }
    }
```

## 独享锁 VS 共享锁
独享锁和共享锁同样是一种概念。我们先介绍一下具体的概念，然后通过ReentrantLock和ReentrantReadWriteLock的源码来介绍独享锁和共享锁。

独享锁也叫排他锁，是指该锁一次只能被一个线程所持有。如果线程T对数据A加上排它锁后，则其他线程不能再对A加任何类型的锁。获得排它锁的线程即能读数据又能修改数据。JDK中的synchronized和JUC中Lock的实现类就是互斥锁。
共享锁是指该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A再加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。

独享锁与共享锁也是通过AQS来实现的，通过实现不同的方法，来实现独享或者共享。

下面为ReentrantReadWriteLock的部分源码：

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

```java
    public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;

        /**
         * Constructor for use by subclasses
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```

```java 
    public static class ReadLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -5992448646407690164L;
        private final Sync sync;

        /**
         * Constructor for use by subclasses
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
```
ReentrantReadWriteLock有两把锁：ReadLock和WriteLock，由词知意，一个读锁一个写锁，合称“读写锁”。再进一步观察可以发现ReadLock和WriteLock是靠内部类Sync实现的锁。Sync是AQS的一个子类，这种结构在CountDownLatch、ReentrantLock、Semaphore里面也都存在。在ReentrantReadWriteLock里面，读锁和写锁的锁主体都是Sync，但读锁和写锁的加锁方式不一样。读锁是共享锁，写锁是独享锁。读锁的共享锁可保证并发读非常高效，而读写、写读、写写的过程互斥，因为读锁和写锁是分离的。所以ReentrantReadWriteLock的并发性相比一般的互斥锁有了很大提升。

那读锁和写锁的具体加锁方式有什么区别呢？在了解源码之前我们需要回顾一下其他知识。

在最开始提及AQS的时候我们也提到了state字段（int类型，32位），该字段用来描述有多少线程获持有锁。在独享锁中这个值通常是0或者1（如果是重入锁的话state值就是重入的次数），在共享锁中state就是持有锁的数量。但是在
ReentrantReadWriteLock中有读、写两把锁，所以需要在一个整型变量state上分别描述读锁和写锁的数量（或者也可以叫状态）。于是将state变量“按位切割”切分成了两个部分，高16位表示读锁状态（读锁个数），低16位表示写锁状态（写锁个数）。

先看写锁的加锁源码：
```java
protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();                  //获取当前锁的个数
            int w = exclusiveCount(c);           //取写锁的个数w
            if (c != 0) {                        //如果已经有线程持有了锁（c!=0）
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //如果写线程数（w）为0（即存在读锁）或者持有锁的线程不是当前线程就返回失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //如果写入锁的数量大于最大数就抛出一个Error
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            //如果当前写线程数为0，且当前线程需要阻塞那么就返回失败；或者通过CAS增加写线程数失败也返回失败
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            //如果c=0, w=0或者c>0,w>0（重入），则设置当前线程为锁的拥有者
            setExclusiveOwnerThread(current);
            return true;
        }
```

* 这段代码首先取到当前锁的个数c，然后再通过c来获取写锁的个数w。因为写锁是低16位，所以取低16位的最大值与当前的c做与运算（ int w = exclusiveCount(c); ），高16位和0与运算后是0，剩下的就是低位运算的值，同时也是持有写锁的线程数目。
* 在取到写锁线程的数目后，首先判断是否已经有线程持有了锁。如果已经有线程持有了锁（c!=0），则查看当前写锁线程的数目，如果写线程数为0（即此时存在读锁）或者持有锁的线程不是当前线程就返回失败（涉及到公平锁和非公平锁的实现）。
* 如果写入锁的数量大于最大数（65535，2的16次方-1）就抛出一个Error。
* 如果当且写线程数为0（那么读线程也应该为0，因为上面已经处理c!=0的情况），并且当前线程需要阻塞那么就返回失败；如果通过CAS增加写线程数失败也返回失败。
* 如果c=0，w=0或者c>0，w>0（重入），则设置当前线程或锁的拥有者，返回成功！

tryAcquire()除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。如果存在读锁，则写锁不能被获取，原因在于：必须确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。

因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，然后等待的读写线程才能够继续访问读写锁，同时前次写线程的修改对后续的读写线程可见。

读锁的代码：
```java
protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;     //如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```
可以看到在tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS保证）增加读状态，成功获取读锁。读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的值是“1<<16”。所以读写锁才能实现读读的过程共享，而读写、写读、写写的过程互斥。

此时，我们再回头看一下互斥锁ReentrantLock中公平锁和非公平锁的加锁源码，发现在ReentrantLock虽然有公平锁和非公平锁两种，但是它们添加的都是独享锁。根据源码所示，当某一个线程调用lock方法获取锁时，如果同步资源没有被其他线程锁住，那么当前线程在使用CAS更新state成功后就会成功抢占该资源。而如果公共资源被占用且不是被当前线程占用，那么就会加锁失败。所以可以确定ReentrantLock无论读操作还是写操作，添加的锁都是都是独享锁。

## AQS
AQS，指的是AbstractQueuedSynchronizer，它提供了一种实现阻塞锁和一系列依赖FIFO等待队列的同步器的框架，ReentrantLock、Semaphore、CountDownLatch、CyclicBarrier等并发类均是基于AQS来实现的，具体用法是通过继承AQS实现其模板方法，然后将子类作为同步组件的内部类。

AQS维护了一个volatile语义(支持多线程下的可见性)的共享资源变量state和一个FIFO线程等待队列(多线程竞争state被阻塞时会进入此队列)。

### State
首先说一下共享资源变量state，它是int数据类型的，其访问方式有3种：
* getState()
* setState(int newState)
* compareAndSetState(int expect, int update)

上述3种方式均是原子操作，其中compareAndSetState()的实现依赖于Unsafe的compareAndSwapInt()方法。

资源的共享方式分为2种：
* 独占式(Exclusive)：只有单个线程能够成功获取资源并执行，如ReentrantLock。
* 共享式(Shared)：多个线程可成功获取资源并执行，如Semaphore/CountDownLatch等。

AQS将大部分的同步逻辑均已经实现好，继承的自定义同步器只需要实现state的获取(acquire)和释放(release)的逻辑代码就可以，主要包括下面方法：
* tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
* tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
* tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
* tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。
* isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。

AQS需要子类复写的方法均没有声明为abstract，目的是避免子类需要强制性覆写多个方法，因为一般自定义同步器要么是独占方法，要么是共享方法，只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。

当然，AQS也支持子类同时实现独占和共享两种模式，如ReentrantReadWriteLock。

### CLH队列（FIFO）
AQS是通过内部类Node来实现FIFO队列的，源代码解析如下：
```java
static final class Node {
    
    // 表明节点在共享模式下等待的标记
    static final Node SHARED = new Node();
    // 表明节点在独占模式下等待的标记
    static final Node EXCLUSIVE = null;

    // 表征等待线程已取消的
    static final int CANCELLED =  1;
    // 表征需要唤醒后续线程
    static final int SIGNAL    = -1;
    // 表征线程正在等待触发条件(condition)
    static final int CONDITION = -2;
    // 表征下一个acquireShared应无条件传播
    static final int PROPAGATE = -3;

    /**
     *   SIGNAL: 当前节点释放state或者取消后，将通知后续节点竞争state。
     *   CANCELLED: 线程因timeout和interrupt而放弃竞争state，当前节点将与state彻底拜拜
     *   CONDITION: 表征当前节点处于条件队列中，它将不能用作同步队列节点，直到其waitStatus被重置为0
     *   PROPAGATE: 表征下一个acquireShared应无条件传播
     *   0: None of the above
     */
    volatile int waitStatus;
    
    // 前继节点
    volatile Node prev;
    // 后继节点
    volatile Node next;
    // 持有的线程
    volatile Thread thread;
    // 链接下一个等待条件触发的节点
    Node nextWaiter;

    // 返回节点是否处于Shared状态下
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 返回前继节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    
    // Shared模式下的Node构造函数
    Node() {  
    }

    // 用于addWaiter
    Node(Thread thread, Node mode) {  
        this.nextWaiter = mode;
        this.thread = thread;
    }
    
    // 用于Condition
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```
可以看到，waitStatus非负的时候，表征不可用，正数代表处于等待状态，所以waitStatus只需要检查其正负符号即可，不用太多关注特定值。

## Lock()和Syncronized的区别
1. 首先synchronized是java内置关键字，在jvm层面，Lock是个java类；
2. synchronized无法判断是否获取锁的状态，Lock可以判断是否获取到锁；
3. synchronized会自动释放锁(a 线程执行完同步代码会释放锁 ；b 线程执行过程中发生异常会释放锁)，Lock需在finally中手工释放锁（unlock()方法释放锁），否则容易造成线程死锁；
4. 用synchronized关键字的两个线程1和线程2，如果当前线程1获得锁，线程2线程等待。如果线程1阻塞，线程2则会一直等待下去，而Lock锁就不一定会等待下去，如果尝试获取不到锁，线程可以不用一直等待就结束了；
5. synchronized的锁可重入、不可中断、非公平，而Lock锁可重入、可判断、可公平（两者皆可）
6. Lock锁适合大量同步的代码的同步问题，synchronized锁适合代码少量的同步问题。

# 容器
## HashMap
JDK1.8 之前HashMap底层是 数组和链表 结合在一起使用也就是链表散列。HashMap 通过 key 的 hashCode 经过扰动函数处理过后得到 hash 值，然后通过 (n - 1) & hash 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

相比于之前的版本，JDK1.8之后在解决哈希冲突时：当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。
### put方法
```java
1.	public V put(K key, V value) {  
2.	    // HashMap允许存放null键和null值。  
3.	    // 当key为null时，调用putForNullKey方法，将value放置在数组第一个位置。  
4.	    if (key == null)  
5.	        return putForNullKey(value);  
6.	    // 根据key的keyCode重新计算hash值。  
7.	    int hash = hash(key.hashCode());  
8.	    // 搜索指定hash值在对应table中的索引。  
9.	    int i = indexFor(hash, table.length);  
10.	    // 如果 i 索引处的 Entry 不为 null，通过循环不断遍历 e 元素的下一个元素。  
11.	    for (Entry<K,V> e = table[i]; e != null; e = e.next) {  
12.	        Object k;  
13.	        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  
14.	            V oldValue = e.value;  
15.	            e.value = value;  
16.	            e.recordAccess(this);  
17.	            return oldValue;  
18.	        }  
19.	    }  
20.	    // 如果i索引处的Entry为null，表明此处还没有Entry。  
21.	    modCount++;  
22.	    // 将key、value添加到i索引处。  
23.	    addEntry(hash, key, value, i);  
24.	    return null;  
25.	}  
```

往HashMap中put元素的时候，先根据key的hashCode重新计算hash值，根据hash值得到这个元素在数组中的位置（即下标），如果数组该位置上已经存放有其他元素了，那么在这个位置上的元素将以链表的形式存放，新加入的放在链头，最先加入的放在链尾。如果数组该位置上没有元素，就直接将该元素放到此数组中的该位置上。

addEntry(hash, key, value, i)方法根据计算出的hash值，将key-value对放在数组table的i索引处。addEntry 是 HashMap 提供的一个包访问权限的方法，代码如下：
```java
1.	void addEntry(int hash, K key, V value, int bucketIndex) {  
2.	    // 获取指定 bucketIndex 索引处的 Entry   
3.	    Entry<K,V> e = table[bucketIndex];  
4.	    // 将新创建的 Entry 放入 bucketIndex 索引处，并让新的 Entry 指向原来的 Entry  
5.	    table[bucketIndex] = new Entry<K,V>(hash, key, value, e);  
6.	    // 如果 Map 中的 key-value 对的数量超过了极限  
7.	    if (size++ >= threshold)  
8.	    // 把 table 对象的长度扩充到原来的2倍。  
9.	        resize(2 * table.length);  
10.	}  
```
当系统决定存储HashMap中的key-value对时，完全没有考虑Entry中的value，仅仅只是根据key来计算并决定每个Entry的存储位置。我们完全可以把 Map 集合中的 value 当成 key 的附属，当系统决定了 key 的存储位置之后，value 随之保存在那里即可。

hash(int h)方法根据key的hashCode重新计算一次散列。此算法加入了高位计算，防止低位不变，高位变化时，造成的hash冲突。
```java
1.	static int hash(int h) {  
2.	    h ^= (h >>> 20) ^ (h >>> 12);  
3.	    return h ^ (h >>> 7) ^ (h >>> 4);  
4.	}  
```
在HashMap中要找到某个元素，需要根据key的hash值来求得对应数组中的位置。如何计算这个位置就是hash算法。前面说过HashMap的数据结构是数组和链表的结合，所以我们当然希望这个HashMap里面的元素位置尽量的分布均匀些，尽量使得每个位置上的元素数量只有一个，那么当我们用hash算法求得这个位置的时候，马上就可以知道对应位置的元素就是我们要的，而不用再去遍历链表，这样就大大优化了查询的效率。

对于任意给定的对象，只要它的 hashCode() 返回值相同，那么程序调用 hash(int h) 方法所计算得到的 hash 码值总是相同的。我们首先想到的就是把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是，“模”运算的消耗还是比较大的，在HashMap中是这样做的：调用 indexFor(int h, int length) 方法来计算该对象应该保存在 table 数组的哪个索引处。

indexFor(int h, int length) 方法的代码如下：
```java
1.	static int indexFor(int h, int length) {  
2.	    return h & (length-1);  
3.	}  
```
这个方法非常巧妙，它通过 h & (table.length -1) 来得到该对象的保存位，而HashMap底层数组的长度总是 2 的 n 次方，这是HashMap在速度上的优化。

我们回到indexFor方法，该方法仅有一条语句：h&(length - 1)，这句话除了上面的取模运算外还有一个非常重要的责任：均匀分布table数据和充分利用空间。

```java
1.	int capacity = 1;  
2.	    while (capacity < initialCapacity)  
3.	        capacity <<= 1;  
```
这段代码保证初始化时HashMap的容量总是2的n次方，即底层数组的长度总是为2的n次方。当数组长度为2的n次幂的时候，不同的key算得得index相同的几率较小，那么数据在数组上分布就比较均匀，也就是说碰撞的几率小，相对的，查询的时候就不用遍历某个位置上的链表，这样查询效率也就较高了。

### get方法
```java
1.	public V get(Object key) {  
2.	    if (key == null)  
3.	        return getForNullKey();  
4.	    int hash = hash(key.hashCode());  
5.	    for (Entry<K,V> e = table[indexFor(hash, table.length)];  
6.	        e != null;  
7.	        e = e.next) {  
8.	        Object k;  
9.	        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))  
10.	            return e.value;  
11.	    }  
12.	    return null;  
13.	}  
```

从HashMap中get元素时，首先计算key的hashCode，找到数组中对应位置的某一元素，然后通过key的equals方法在对应位置的链表中找到需要的元素。

### Fail-Fast机制
java.util.HashMap不是线程安全的，因此如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。

这一策略在源码中的实现是通过modCount域，modCount顾名思义就是修改次数，对HashMap内容的修改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的expectedModCount。

```java
1.	HashIterator() {  
2.	    expectedModCount = modCount;  
3.	    if (size > 0) { // advance to first entry  
4.	    Entry[] t = table;  
5.	    while (index < t.length && (next = t[index++]) == null)  
6.	        ;  
7.	    }  
8.	}  
```
在迭代过程中，判断modCount跟expectedModCount是否相等，如果不相等就表示已经有其他线程修改了Map。注意到modCount声明为volatile，保证线程之间修改的可见性。

```java
1.	final Entry<K,V> nextEntry() {     
2.	    if (modCount != expectedModCount)     
3.	        throw new ConcurrentModificationException();  
```

## ConcurrentHashMap
JDK1.7的 ConcurrentHashMap 底层采用 分段的数组+链表 实现，JDK1.8 采用的数据结构跟HashMap1.8的结构一样，数组+链表/红黑二叉树。

在JDK1.7的时候，ConcurrentHashMap（分段锁） 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 到了 JDK1.8 的时候已经摒弃了Segment的概念（但是为了兼容之前的版本，实现上还是保留了segment），而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作，加锁的时候也不会去锁住整个segment，只会锁住链表的首节点。（JDK1.6以后 对 synchronized锁做了很多优化） 整个看起来就像是优化过且线程安全的 HashMap，虽然在JDK1.8中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本。

### 线程安全的具体方式
#### JDK1.7
首先将数据分为一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。
ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成。
Segment 实现了 ReentrantLock,所以 Segment 是一种可重入锁，扮演锁的角色。HashEntry 用于存储键值对数据。
```java
static class Segment<K,V> extends ReentrantLock implements Serializable {
}
```
一个 ConcurrentHashMap 里包含一个 Segment 数组。Segment 的结构和HashMap类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个HashEntry数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment的锁。

几个成员变量：
* static final int DEFAULT_INITIAL_CAPACITY = 16; map的默认容量
* static final float DEFAULT_LOAD_FACTOR = 0.75f;默认负载因子
* static final int DEFAULT_CONCURRENCY_LEVEL = 16;并发级别，就是segment数组的大小，一旦确定，不会再改变，默认大小16（面试有问过这个值会不会再变大，如果要变智能通过构造函数，也就是根据这个对象创建一个新的对象）
* static final int MAX_SEGMENTS = 1 << 16;允许的最大segment数量
* static final int MAXIMUM_CAPACITY = 1 << 30;map最大容量，在构造时如果指定的大小超过该值，就用该值替换
* static final int MIN_SEGMENT_TABLE_CAPACITY = 2; Entry[]数组最小容量
* static final int RETRIES_BEFORE_LOCK = 2; 默认自旋次数，超过这个次数直接加锁，防止在size方法中由于不停有线程在更新map导致无限进行自旋影响性能

#### JDK 1.8
ConcurrentHashMap取消了Segment分段锁，采用CAS和synchronized来保证并发安全。数据结构跟HashMap1.8的结构类似，数组+链表/红黑二叉树。Java 8在链表长度超过一定阈值（8）时将链表（寻址时间复杂度为O(N)）转换为红黑树（寻址时间复杂度为O(log(N))）
synchronized只锁定当前链表或红黑二叉树的首节点，这样只要hash不冲突，就不会产生并发，效率又提升N倍。
链表长度大于8时并不一定会转换成红黑树，在大于8时是调用treeifyBin，在这个方法中会判断整个map的容量是否大于MIN_TREEIFY_CAPACITY（默认64），大于才会转为树，否则会将调用tryPresize()对table进行扩容，然后调整位置。
几个成员变量：
* private static final int MAXIMUM_CAPACITY = 1 << 30;最大容量
* private static final int DEFAULT_CAPACITY = 16;默认容量，在第一次插入时才会进行Node数组的初始化
* private static final int DEFAULT_CONCURRENCY_LEVEL = 16;默认的并发度，也就是segment的数量，这是为了兼容旧版保留的值，在1.8中已经没有了segment的概念
* private static final float LOAD_FACTOR = 0.75f;默认负载因子
* static final int TREEIFY_THRESHOLD = 8;转为红黑树的阈值，默认8，大于该值会调用方法，但是不一定会转树
* static final int UNTREEIFY_THRESHOLD = 6;树转链表的阈值
* static final int MIN_TREEIFY_CAPACITY = 64;在调用转树的方法时，如果table的总元素数量不够这个值，就对table进行扩容，然后重新调整位置，而不是直接转树


