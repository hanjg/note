[toc]
## 列表 ##
### 数组 ###
- slice对标java ArrayList，长度可变，复制扩容。<br>![210905.go.slice.png](https://img-blog.csdnimg.cn/9b106f46d909465ebf00d450646b53f4.png)
- range遍历的时候**第一个参数为下标**，第二个才是值。如果遍历值，需要忽略下标。
```go
	for index := range o.TicketIdList {
		//一个参数为下标
	}
	for _, ticketId := range o.TicketIdList {
		//两个参数为下标+值
		ticketIdList = append(ticketIdList, int64(ticketId))
	}
```


### 链表 ###
- List对标java LinkedList
- Ring循环链表

## 字典 ##
- 对标hashmap
- key需要支持判等操作，所以不能是函数、字典、切片，否则即使key是interface{}，也会抛panic
- 给key为nil的

## 通道 ##
- 对标java blockingList
- 特性
	- 同一个通道，发送操作之间是互斥的，接收操作之间也是互斥的。
	- 发送操作和接收操作中对元素值的处理都是不可分割的。
	- 发送操作在完全完成之前会被阻塞。接收操作也是如此。保证**操作的互斥和值完整**。
- select语句根据一套规则选择其中一个分支执行，没有命中则默认分支
- 对nil通道的发送和接收永久阻塞

## 函数 ##
- 函数可作为值传递
- 方法必须隶属于某个类（对标java类中的方法），不能作为值传递

## 异常 ##
- catch异常的方式
```go
func main() {
	defer func() {
		fmt.Println("Enter defer function.")
		if p := recover(); p != nil {
			fmt.Printf("eat panic: %s\n", p)
		}
		fmt.Println("Exit defer function.")
	}()

	panic("my panic")
}
```

## 线程模型 ##
- [Java和Golang的线程模型](https://lushunjian.gitee.io/2021/03/09/java-he-golang-de-xian-cheng-mo-xing/)
- [Golang 调度器 GMP 原理与调度全分析](https://learnku.com/articles/41728)

### 模型定义 ###
- **调度器**（Thread Scheduler）：内核通过操纵调度器对**内核线程**进行调度，并负责将线程的任务映射到各个处理器上。
- **内核线程**（Kernel Level Thread）：简称KLT
	- 每个内核线程可以视为**内核的一个分身**，这样操作系统就有能力同时处理多件事情。
- **轻量级进程**（Light Weight Process）：简称LWP
	- 是**内核线程的高级抽象**，内核线程在用户空间的代理线程，许多操作都要进行系统调用。
- **用户线程**（User Level Thread）：简称ULT
	- 用户线程的建立，同步，销毁，调度完全在**用户空间**完成，不需要内核的帮助。因此这种线程的操作是极其快速的且低消耗的
### 模型分类 ###
1. **N：1**线程模型。线程实现在用户空间中
	1. 一个进程维护多个用户线程，这**多个用户线程**绑定**一个内核线程**
	2. 线程调度只在用户态，无上下文切换开销
	3. 一个进程中只有一个线程能够运行，若阻塞影响这个进程<br>![211219.n-1.png](https://img-blog.csdnimg.cn/120f74c689054183836c8df0b64e00d6.png)
2. **1：1**线程模型。线程实现在内核空间中
	1. 一个进程维护多个用户线程，**一个用户线程**绑定**一个内核线程**
	2. 可以使用CPU多核能力
	3. 创建线程需要系统调用，代价高<br>![211219.1-1.png](https://img-blog.csdnimg.cn/018c213918fe4aa989482798b8f6547c.png)
3. **M：N**线程模型。用户线程加轻量级进程混合实现
	1. 一个进程维护多个用户线程，**多个用户线程**绑定**多个内核线程**
	2. 可多核加速，且创建用户线程在用户态，代价低
	3. 用户空间需要实现线程调度器，维护用户线程、内核线程的映射关系，实现复杂<br>![211219.m-n.png](https://img-blog.csdnimg.cn/60b0438d8c51498b937fec79d3c02a88.png)

### java ###
- 对于Sun JDK来说，它的Windows版与Linux版都是使用**一对一**的线程模型实现的。Java中线程的本质，其实就是操作系统中的线程
	- Linux下是基于pthread库实现的轻量级进程
	- Windows下是原生的系统Win32 API提供系统调用从而实现多线程

### go ###
> **GMP模型**
> ![211219.gmp.png](https://img-blog.csdnimg.cn/32648db65f1445249ab372d9a9988ab2.png)
- G: Goroutine，**用户线程**
	- 每个 Goroutine 对应一个 G 结构体，Goroutine 的运行堆栈、状态以及任务函数，可重用。
	- G 并非执行体，每个 G 需要绑定到 P 才能被调度执行
- M: Machine，**内核线程**，代表着真正执行计算的资源
	- 在绑定有效的 P 后，进入 schedule 循环
	- schedule 循环的机制大致是从 Global 队列、P 的 Local 队列以及 wait 队列中获取 G，切换到 G 的执行栈上并执行 G 的函数，调用 goexit 做清理工作并回到 M，如此反复。
	- M 并不保留 G 状态，这是 **G 可以跨 M 调度**的基础，M 的数量是不定的，由 Go Runtime 调整，为了防止创建过多 OS 线程导致系统调度不过来，目前默认最大限制为 10000 个。
- P: Processor，表示**线程调度器**
	- 对 G 来说，P 相当于 CPU 核，G 只有绑定到 P才能被调度。
	- 对 M 来说，P 提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等，
	- P 的数量决定了系统内最大可并行的 G 的数量（前提：物理 CPU 核数 >= P 的数量），P 的数量由用户设置的 GOMAXPROCS 决定，但是不论 GOMAXPROCS 设置为多大，P 的数量最大为 256。
- LRQ：Local Run Queue，P的**本地队列**
	- 本地等待运行的 G，LRQ存的数量有限，不超过 256 个。
	- 新建 G 时，G 优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列
- GRQ：Global Run Queue，**全局队列**


## 内存模型 ##
- [Go 语言内存管理三部曲](https://xie.infoq.cn/article/ee1d2416d884b229dfe57bbcc)
- [Go 并不需要 Java 风格的 GC](https://xie.infoq.cn/article/a2deef4194e0208f2bc03b99d)
- [GO GC 知识点整理](https://xie.infoq.cn/article/d5d9434a477ff1016d85e63d5)


### 内存管理组件 ###
- **mspan**。内存管理基本单元，每种mspan分配不同大小的内存，8B-32KB。
- **mcache**。管理本地缓存，和**P**线程调度器绑定。
- **mcentral**。管理全局mspan，分配内存时需要加锁
- **mheap**。管理go所有动态分配的内存，程序启动或者mheap内存不足时向操作系统申请。
- ![211224.go.mem.png](https://img-blog.csdnimg.cn/08036a8ffafd4db0827b7bca36844641.png)

### 分配规则 ###
- 小对象(<32KB)通过mspan分配内存
- 大对象则直接由mheap分配对应的数量的内存页（每页大小是 8KB）

## GC ##
- 无分代、不整理、并发的**三色标记清除法**。
- 无分代
	- Go会通过逃逸分析将**大部分新创建的对象存储在栈**上，需要**长期保存的对象存在于堆中**。
	- 栈会被回收，不需要 GC。回收堆内存时才GC。
- 不整理
	- 基于tcmalloc 分配算法，基本没有内存碎片问题。比如对象数组go分配连续的空间，java只有数组的引用连续。
- 三色标记清除
	- 解决标记清除时的STW问题，程序和GC基本并发

### 三色标记算法 ###
- 对象分类：
	- 白色对象：潜在垃圾对象，能被垃圾收集器回收。
	- 黑色对象：活跃对象，不存在指向外部的指针 Or 从根对象可达的对象，不会扫描子对象。
	- 灰色对象：活跃对象，存在指向白色对象的外部指针，会扫描子对象。
- 回收步骤：
	1. 初始所有对象都是白色的
	2. 遍历**GC ROOT**对象，将引用的对象标记为灰色<br>![211224.go.gc2.png](https://img-blog.csdnimg.cn/a2297914684e4da1abf6285538e2d2f6.png)
	3. 遍历**灰色集合**，灰色对象引用的对象标记白色，该灰色对象标记黑色<br>![211224.go.gc3.png](https://img-blog.csdnimg.cn/39b6a9e46af94d9790bd03190077402b.png)
	4. 重复3直到灰色集合中无对象
	5. 回收**白色集合**中的对象 

### 写屏障 ###
- 处理gc过程中创建的对象被黑对象引用的问题。
- 新创建的对象直接标记成黑色。

### GC步骤 ###
1. 标记准备：STW，状态切换，开启写屏障，助标记程序将gc root入队
2. 标记阶段：并行GC，三色标记法+写屏障
3. 标记终止：STW，状态切换，关闭辅助标记程序，清除相关缓存
4. 清理阶段：状态切换，关闭写屏障，并行清理