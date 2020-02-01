## 参考链接 ##
- [Java方向如何准备BAT技术面试答案(汇总版)](https://zhuanlan.zhihu.com/p/26679704)

## 1、面向对象和面向过程的区别 ##
- 面向过程优点：性能高，因为类调用的时候需要实例化，开销比较大；
- 面向对象优点：易维护、易复用、易扩展，因为面向对象有封装、继承、多态的特性，可以设计出低耦合的系统，使系统更加灵活、更加易于维护。

## 2、java的四个基本特性 ##
### 1、抽象 ###
- 把现实中的某一类东西用代码表示，通常叫做类或者接口。
- 抽象包括两个方面：数据抽象--对象的域，过程抽象--对象的方法。
### 2、封装： ###
- 把客观事物抽象成类，类的域和方法只让可信的类或者对象操作，对不可信的隐藏。
- 封装分为域的封装和方法的封装。
### 3、 继承： ###
- 把拥有共同特性的多个类抽象成一个类，这个类是他们的父类，这些类继承这个父类。
- 父类的意义在于抽取多类事物的共性。
### 4、 多态： ###
- 允许不同类的对象对同一消息（方法调用）做出响应。即同一消息可以根据发送对象的不同而采取不同的行为方式。
- 实现技术：动态绑定--根据实例的运行期类型调用相应的方法。
- 作用：消除类型之间的耦合。
- 必要条件：继承、重写、父类引用指向子类的对象。

## 3、重载和重写的区别 ##
- 重载：在同一个类中，方法名相同，参数列表不同，返回值和访问修饰符可以不同也可以相同，发生在编译时。
- 重写：在父子类中，方法名和参数列表相同，子类返回值类型<=父类返回值类型（不包括基本类型），子类抛出异常<=父类抛出异常，子类访问修饰符>=父类访问修饰符（如果父类方法的访问修饰符为private，则子类中就不是重写）。

## 4、Constructor ##
- Constructor不能被override，不能用static修饰。

## 5、访问控制符的区别 ##
- private:本类。
- public:任何类。
- protected:同包的类、子类。
- 默认:同包的类。

## 6、String类不可继承 ##
String是final类，final修饰过的都不能继承。

## 7、String,StringBuffer,StringBuilder的区别 ##
### 1、可变性 ###
- String中使用final char value[]，不可变。
- StringBuffer,StringBuilder继承AbstractStringBuilder使用char[] value，可变。
### 2、线程安全性 ###
- String不可变，线程安全。
- StringBuffer对方法使用同步锁，线程安全。
- StringBuilder没有对方法加同步锁，非线程安全。
### 3、性能 ###
- String每次改变时都会生成一个新的String对象。
- StringBuffer,StringBuilder只是对象本身操作，StringBuilder性能比StringBuilder高10%。

## 8、hashCode和equals的关系 ##
- equals相等-->hashcode相等。
- 重写equals时一定要重写hashcode，否则在使用hashtable等需要使用hashcode的时候很有可能会出问题（contain函数在hashcode相同的bucket中查找是否存在equals的对象）。

## 9、抽象类和接口的区别 ##
### 1、语法 ###
- abstract class A {
	abstract void method();
}
- interface A {
	void method();
}
- interface某种意义上说是一种特殊的abstract class；只能够有static final的域（interface中一般不定义域）；所有方法都是abstract的。
- abstract class中可以赋予方法默认行为；interface中方法不能有默认行为。
- 一个类只能继承一个abstract class，但可以实现多个interface。
### 2、设计理念 ###
- abstarct class体现继承关系，父类和子类在概念本质上相同。
- interface的实现者仅仅实现interface定义的契约。

## 10、自动装箱和拆箱 ##
- 装箱：将基本类型用对应的包装类包类型。
- 拆箱：将包装类型转换为基本数据类型。
- java编译器会在编译器根据语法决定是否装箱和拆箱。

## 11、什么是泛型，为什么使用，什么是类型擦除 ##
- 泛型：参数化类型，适用于多种类型。
- 使用：创建集合时就指定元素的类型，该集合只能保存制定类型的元素。
- 类型擦除:java编译器生成的字节码不包含泛型类型信息，类型信息在编译处理时被擦除，用最顶级的父类类型替换。


## 12、java中集合类的关系 ##
- List,Set接口继承自Collection接口。
- Set无序、元素不重复，主要实现类有HashSet和TreeSet
- List有序、元素可重复，主要实现类有ArrayList,LinkedList,Vector。
- Map和Collection接口无关，Map是key对value的映射集合,key不能重复，value可以重复，主要实现类有HashMap,TreeMap,HashTable。

## 13、HashMap实现原理 ##
- [HashMap的工作原理](http://www.admin10000.com/document/3322.html)
- [深入Java集合学习系列：HashMap的实现原理](http://zhangshixi.iteye.com/blog/672697)

## 14、HashTable实现原理 ##
- [Java 集合系列11之 Hashtable详细介绍(源码解析)和使用示例](http://www.cnblogs.com/skywang12345/p/3310887.html)
- [Hashtable源码剖析](http://blog.csdn.net/chdjj/article/details/38581035)


## 15、HashMap和HashTable的区别 ##
- HashTable线程安全，HashMap非线程安全。
- HashTable不允许有null(key和value)，HashMap允许null(key和value)。
- HashTable多出一个contains方法，与containsValue功能一样，而不是containsKey。
- HashTable用Enumeration遍历；HashMap用Iterator遍历。
- HashTable中数组默认大小11，增加方式old*2+1；HashMap中数组默认大小16，增加方式<<1。
- HashTable直接使用对象的hashCode；HashMap重新计算hash值，并且用与代替求模。
### 注： ###
- Enumeration:只能读取不能修改，不支持fail-fast机制。
- Iterator：能读取能删除，支持fail-fast机制。
- fail-fast机制：当线程A通过iterator遍历集合时，若集合的内容被其他线程改变，那么线程A就会抛出异常，即产生fail-fast事件。

## 16、ArrayList和Vector区别 ##
- Vector线程安全，ArrayList非线程安全。
- 自动扩容时，Vector增长1倍，ArrayList增长1/2。

## 17、ArrayList和LinkedList的区别和使用场景 ##
### 区别 ###
- ArrayList底层数组实现，可以随机查找。
- LinkedList底层双向链表实现，增删速度快。实现Queue接口。
### 场景 ###
- LinkedList适合从中间插入或删除。
- ArrayList适合检索和在末尾插入或删除。

## 18、Collection和Collections的区别 ##
- Collection是一个接口，提供对集合对象基本操作的通用接口方法。
- Collections是一个类，包含集合操作的静态方法。该类不能实例化。

## 19、ConcurrenthashMap实现原理 ##
- Hashtable的synchronized是针对整张表，Concurrenthashmap使用多个锁控制对表不同部分的修改。
- [Java集合---ConcurrentHashMap原理分析](http://www.cnblogs.com/ITtangtang/p/3948786.html)
- [ConcurrentHashMap](http://ifeve.com/concurrenthashmap/)

## 20、Error、Exception区别 ##
- Error和Exception都继承tThrowable。
- Error类一般指与虚拟机相关的问题，比如系统崩溃，虚拟机错误，内存空间不足，方法调用栈溢出。这类错误导致的应用程序中断仅靠程序本身无法恢复和预防，应该终止程序。
- Exception类表示程序可以处理的异常，可以捕获并且可能恢复。这类异常应该尽可能处理。

## 21、checked和unchecked ##
### unchecked Exception ###
- 通常是自身的问题。
- 程序瑕疵或者逻辑错误，运行时无法恢复。
- 包括Error和RuntimeException及其子类。
- 语法上不需要抛出异常。
### checked Exception ###
- 程序不能直接控制的无效外界情况（用户输入，数据库问题，网络异常，文件丢失）。
- 除了Error和RuntimeException及其子类之外的类。
- 需要try catch处理或者throws抛出异常。

## 22、java如何实现代理机制 ##
- 代理模式：为其他对象提供一种代理，以控制对这个对象的访问。
- jdk动态代理：代理类和委托类实现了共同的接口，用到InvocationHandler接口。
- cglib动态代理：代理类是目标类的子类，用到MethodInterceptor接口。

## 23、多线程的实现方式 ##
- Thread + Runnable。
- ExecutorService + Runable，不返回值。
- ExecutorService + Callable，返回Future。

## 24、线程的状态转换 ##
resume,suspend，stop已经废止，因为可能导致死锁。
![](pic/threadState.jpg)

## 25、如何终止一个线程 ##
1. 线程里面是一个循环。设置一个标志位，在循环检查的时候跳出循环。
2. 线程因为sleep,wait,join等阻塞或者挂起的时候。使用interrupt，产生InterruptedException异常，从而跳出线程。

## 26、什么是线程安全 ##
- 多个线程可能会同时运行一段代码，每次运行的结果和单线程运行的结果是一样的，而且其他变量的值也和预期的是一样的。
- 《Java Concurrency In Practice》：当多个线程访问同一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者调用方法进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象就是线程安全的。

## 27、如何保证线程安全 ##
- 对非安全的代码进行加锁，synchronized,volatile,lock等。
- 使用线程安全的类。
- 多线程并发情况下，线程共享的变量改为方法级的局部变量。

## 28、synchronized如何使用 ##
### 使用synchronized同步方法 ###
- 静态方法，需要获得类锁。
- 非静态方法，需要获得对象锁。
### 使用synchronized同步代码段 ###
- synchronized(类.class){}，需要获得类锁。
- synchronized(this,其他对象)，需要获得对象锁。

## 29、synchronized和Lock的区别 ##
- Lock能完成synchronized所实现的所有功能。
- Lock的锁定是通过代码实现的，synchronized是在JVM层次上实现的
- Lock需要手动在finally从句中释放锁，synchronized自动释放锁。
- Lock可以通过tryLock方法用非阻塞方式去拿锁。
- Lock锁的范围：代码块；synchronized锁的范围：代码块，对象，类。

## 30、多线程如何进行信息交互 ##
- wait(long timeout)：挂起线程，释放对象锁，直到时间到期或其他线程调用该对象的notify(),notifyAll()。
- wait()：挂起线程，释放对象锁，直到其他线程调用该对象的notify(),notifyAll()。
- notify()：唤醒该对象上wait的单个线程。
- notifyAll()：唤醒该对象上wait的所有线程。

## 31、sleep和wait的区别 ##
- sleep是Thread中的方法，wait是Object中的方法。
- sleep：线程不会释放对象锁；wait：线程释放对象锁。

## 32、多线程和死锁 ##
### 定义 ###
- 任务A等待任务B，任务B等待任务C，直到这个链条上的任务等待任务A释放锁，这样就出现了一个任务之间相互等待的连续循环，若无外力作用他们都将无法继续。
### 原因 ###
- 系统资源不足。
- 进程推进的顺序不合适。
- 资源分配不当。

## 33、产生死锁的必要条件 ##
1. 互斥条件：进程使用的资源中至少有一个不能共享。
2. 请求与保持条件：进程因为请求资源阻塞时，保持已获得的资源。
3. 不剥夺条件：进程已获得的资源在未使用完之前不能被强行剥夺。
4. 循环等待条件：A等待B的资源，B等待C的资源，直到某个进程等待A的资源，使得所有进程都无法继续。

## 34、死锁的预防 ##
#### 打破产生死锁的四个必要条件中的一个或几个 ####
1. 互斥条件：允许进程同时访问某些资源，但是有的资源不允许同时访问，比如打印机。该方法无实用价值。
2. 请求与保持条件：实行资源预分配策略，即进程在运行前一次性的向系统申请所需要的全部资源。如果某个资源得不到满足，则不分配资源，进程阻塞；只有满足进程的全部资源需求时，才一次性的将所申请的资源全部分配给该进程。
3. 不剥夺条件：允许进程强行从占有者那里夺取资源，即一个进程拥有某些资源，但是新申请的资源不能立刻满足，该进程必须释放所占有的全部资源。该方法实现困难，会降低性能。
4. 循环等待条件：实行资源有序分配策略，即把资源事先编号，所有进程请求资源先编号小的再编号大的，这样就不会产生环路。

## 35、守护进程是什么，如何实现 ##
- 程序运行时在后台提供一种通用服务的线程，当非后台线程结束时，程序也就终止，同时杀死进程中的所有后台线程。
- setDaemon(true)。

## 36、java线程池技术及原理 ##
### 目的 ###
- 如果并发的线程数量很多，并且每个线程执行一个时间很短的任务就结束，这样频繁创建线程会大大降低系统的效率。
- 使用线程池可以使得线程复用，即线程执行完一个任务不被销毁而是继续执行其他任务。
### 任务处理策略 ###
- 线程池中的当前线程数poolSize。
- poolSize<corePoolSize，则每来一个任务，就创建一个线程执行这个任务。
- poolSize>=corePoolSize，则每来一个任务，则添加到任务缓存队列。添加成功（一般队列未满），则该任务会等待空闲线程将其取出执行；添加失败（一般是缓存队列已满）且poolSize<maximumPoolSize，则创建新的线程执行这个任务。
- poolSize>=maximumPoolSize，则采取任务拒绝策略。
- poolSize>corePoolSize，如果某线程空闲时间超过keepAliveTime,线程将被终止，直到poolSize<=corePoolSize；如果允许设置核心池中的存活时间，核心池中的线程空闲时间超过keepAliveTime，该线程也会被终止。
- [Java并发编程：线程池的使用](http://www.cnblogs.com/dolphin0520/p/3932921.html)

## 37、java并发包oncurrent及常用类 ##
- 线程池（36），[锁](http://www.cnblogs.com/dolphin0520/p/3923167.html)，[集合
](http://www.cnblogs.com/huangfox/archive/2012/08/16/2642666.html)

## 38、volatile关键字 ##
- volatile具有可见性：线程对变量作出的更改对于另一个线程是可见的，即线程能自动发现volatile的最新值。
- volatile不具有原子性：允许超过一个线程访问该数据。
- 要使volatile提供理想的线程安全，必须同时满足两个条件：1、对变量的写操作不依赖于当前值；2、该变量没有包含在具有其他变量的不变式中。

## 39、JAVA中的NIO,BIO,AIO分别是什么 ##
### 几个概念 ###
- 同步：使用同步IO时，java自己处理IO读写。
- 异步：使用异步IO时，java将IO读写委托给OS处理，需要将缓冲区地址和大小传给OS，OS需要支持异步IO操作API。
- 阻塞：使用阻塞IO时，java调用会一直阻塞到读写完成后才返回。
- 非阻塞：使用非阻塞IO时，当IO事件分发器通知可读写时继续读写，不可读写时进行其他操作，不断循环直到读写完成。
### BIO ###
- 同步阻塞。
- 服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，可以通过线程池机制改善。
- 适合链接数目较小且固定的架构。
### NIO ###
- 同步非阻塞。
- 服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有IO请求是才启动一个线程进行处理。
- 适合链接数目较多且比较短（轻操作）的架构，比如聊天服务器，jdk1.4开始支持。
### AIO ###
- 异步非阻塞。
- 服务器实现模式为一个有效请求一个线程，客户端的IO请求都是有OS完成后再通知服务器应用去启动线程处理。
- 适用于链接数目多且连接较长（重操作）的架构，比如相册服务器，jdk1.7开始。
### 详细 ###
[http://blog.csdn.net/skiof007/article/details/52873421](http://blog.csdn.net/skiof007/article/details/52873421)

## 40、IO和NIO区别 ##
1. IO面向流，NIO面向缓冲区。
2. IO阻塞，NIO非阻塞。
3. NIO的选择器允许一个单独的线程来监视多个输入通道。

## 41、序列化和反序列化 ##
### 定义 ###
- 序列化：把对象转换为字节序列。
- 反序列化：字节序列恢复为对象。
### 用途 ###
- 把对象的字节序列永久的保存在硬盘上，通常放在一个文件里。
- 在网络上传送对象的字节序列，因为无论何种类型的数据，都会以二进制序列的形式在网络上传输。

## 42、常见的序列化协议 ##
- Protobuf,Thrift,Hessian,Kryo

## java内存分配 ##
![](pic\memory.png)
### 程序计数器 ###
- 当前线程所执行的字节码的行号指示器。
### java虚拟机栈 ###
- 描述java方法执行的内存模型：每个方法被执行的时候都会创建一个栈帧，用于存储局部变量表，操作栈，动态链接，方法出口等信息。
- 每个方法被调用的过程就是对应一个栈帧在虚拟机栈中入栈到出栈的过程。
### 本地方法栈 ###
- 执行本地方法服务。
### 堆 ###
- 存放对象实例。
### 方法区 ###
- 存储已经被虚拟机加载的类信息，常量，静态变量，即时编译器（JIT)编译后的代码等数据。
- 运行时常量池： 运行时常量池是方法区的一部分，Class文件中除了有类的版本，字段，方法，接口等信息以外，还有一项信息是常量池用于存储编译器生成的各种字面量和符号引用，这部分信息将在类加载后存放到方法区的运行时常量池中。

## 43、内存溢出和内存泄漏 ##
- 内存溢出：程序在申请内存时，没有足够的内存空间。
- 内存泄漏：分配出去的内存不再使用，但是无法回收。

## 44、java内存模型及各个区域的OOM，如何重现OOM ##
[JVM内存管理：深入Java内存区域与OOM](http://hllvm.group.iteye.com/group/wiki/2857-JVM)

## 45、出现OOM如何解决 ##
1. 可通过命令定期抓取heap dump或者启动参数OOM时自动抓取heap dump文件。
2. 通过对比多个heap dump，以及heap dump的内容，分析代码找出内存占用最多的地方。
3. 分析占用的内存对象，是否是因为错误导致的内存未及时释放，或者数据过多导致的内存溢出。

## 46、javaGC机制 ##
#### “自适应的、分代的、停止-复制、标记清扫”式垃圾回收器 ####
### java内存分配 ###
- java堆分配的实现更像传送带，每份配一个新对象，就往前移动一格，“堆指针”只是简单的移动到尚未分配的区域。
- 垃圾回收器一面回收空间，一面使堆中的对象紧凑排列，从而实现了一种高速的、有无限空间可供分配的堆模型。
### 自适应的 ###
- JVM会进行监视，如果所有对象很稳定，切换到“标记-清扫”方式；如果堆空间出现很多碎片，切换到“停止-复制”方式。
### 分代的 ###
- 内存分配以块为单位，每个块用代数来记录是否存活（从旧堆复制到新堆会导致大量内存复制的行为）。如果块在某处被引用，其代数增加；垃圾回收器将对上次回收动作之后新分配的块整理。垃圾回收器会定期进行完整的清理--大型对象不会被复制（只是代数增加），内含小型对象的块则被复制并整理。
### 停止-复制 ###
- 暂停程序（不是后台回收），从堆栈和静态存储区出发，遍历所有引用，找出所有存活的对象，从当前堆复制到另一个堆，没有复制的都是垃圾。当对象被复制到新堆的时候，它们紧挨着，所以新堆保持紧凑排列。
### 标记-清扫 ###
- 从堆栈和静态存储区出发，遍历所有引用，找出所有存活的对象，给对象设一个标记，当标记工作全部完成时，释放没有标记的对象。不发生复制动作，所以剩下的堆空间是不连续的。
### 参考 ###
- [Java 内存区域和GC机制](http://www.cnblogs.com/hnrainll/archive/2013/11/06/3410042.html)

## 47、java类加载器如何加载类 ##
### 初始化顺序 ###
1. 父类的静态域和静态初始化块。
2. 子类的静态域和静态初始化块。
3. 父类的实例域和实例初始化块。
4. 父类的构造方法。
5. 子类的实例域和实例初始化块。
6. 子类的构造方法。
7. [深入理解Java类加载器(1)：Java类加载原理解析](http://blog.csdn.net/zhoudaxia/article/details/35824249)

## 48、Statement和PreparedStatement区别 ##
- PreparedStatement是预编译的，适合批处理，也叫JDBC存储过程。
- Statement每次执行sql语句都要编译，适合一次性的存取。
- PreparedStatement安全性更强，可以使传递的参数强制类型转换。