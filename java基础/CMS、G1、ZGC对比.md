[toc]
## CMS+ParNew ##
- 一块eden,两块survivor，一块old。

### YGC ###
- ParNew。一次stw——复制。<br>![](https://img-blog.csdn.net/20180526123542457)
- eden区和survivor1区活跃对象**复制**到survivor2，部分survivor1区对象晋升到老年代。
- 开始前。<br>![210317.cms.young.png](https://img-blog.csdnimg.cn/20210319003754232.png)
- 结束后。<br>![210317.cms.young.2.png](https://img-blog.csdnimg.cn/20210319003753542.png)

### OGC ###
- CMS。两次stw——初始标记、重新标记。<br>![](https://img-blog.csdn.net/2018052613362657)
- 老年代直接**标记清除**，没有复制压缩。
- 开始前。<br>![210317.cms.old.1.png](https://img-blog.csdnimg.cn/20210319003753512.png)
- 结束后。<br>![210317.cms.old.2.png](https://img-blog.csdnimg.cn/20210319003752329.png)

### 优缺点 ###
- 并发高效。
- 老年代并发清理碎片多。
- 停顿时间无上限。

## G1 ##
- 堆被分成多个大小的区域。映射为不连续的eden,survivor,old。
- YGC 1次stw——复制，Mixed GC 4次stw——初始标记、重新标记、清理、复制。<br>![210318.g1.png](https://img-blog.csdnimg.cn/20210319003912729.png)
- 停顿时间瓶颈在复制上，未能在转移过程中准确定位对象地址。
- 无法在堆中申请新的分区时，执行STW的单线程Full GC。

### YGC ###
- 年轻代**复制**存活对象到新的survivor。
  - 存活对象用 Rset(记录其他region中的对象对本region对象的引用)标识，GC时不用扫描整个堆。
- 开始前。<br>![210317.g1.young.1.png](https://img-blog.csdnimg.cn/20210319004006145.png)
- 结束后。<br>![210317.g1.young.2.png](https://img-blog.csdnimg.cn/2021031900400559.png)

### Mixed GC ###
- 并行**标记清除**不可达old，**复制**活跃的young和old。
- 标记清除old。<br>![210317.g1.old.1.png](https://img-blog.csdnimg.cn/20210319004043319.png)
- 复制old+young。<br>![210317.g1.old.2.png](https://img-blog.csdnimg.cn/20210319004043625.png)
- 结束后。<br>![210317.g1.old.3.png](https://img-blog.csdnimg.cn/20210319004042578.png)

### 优缺点 ###
- 最大停顿时间可预测。每次处理部分堆实现停顿时间限制。
- 老年代也复制，碎片有限。
- 写屏障，维护[RememberedSet](https://blog.csdn.net/FMC_WBL/article/details/107864334)，且Rset占据空间较多，5-20%。

## ZGC ##
- 不分代。
- 3次stw——初始标记、再标记、初始转移。
- 瓶颈为再标记，通常1ms以内。<br>![210318.zgc.png](https://img-blog.csdnimg.cn/20210319004133346.png)
- 2个技术点可在转移对象的过程中追踪对象最新的位置，从而做到并发转移。
  - 着色指针（引用）。
    - 对象存活信息记在**引用**中，而不是对象头。
    - 使用3个bit标识3种视图和存活信息：M0,M1,Remapped。<br>![210318.zgc.pointer.png](https://img-blog.csdnimg.cn/20210319004133187.png)
  - 读屏障。应用线程**从堆中读取对象引用**时触发。
    - ZGC用来标记和转移过程中确定对象引用地址是否满足条件，并作出修改对象视图等操作。

### 过程 ###
- 初始化。视图为Remapped。
- 标记阶段。
  - 视图为Mo，标记之后活跃对象视图为M0，不活跃的为Remapped。<br>![210321.zgc.mark.png](https://img-blog.csdnimg.cn/20210321235507326.png)
  - 活跃对象地址存储在**活跃对象信息表**。
- 转移阶段。视图为remapped。<br>![210321.zgc.copy.png](https://img-blog.csdnimg.cn/20210321235507281.png)
- 第二次GC标记阶段的活跃对象视图为M1，M0和Remapped均为不活跃对象。<br>![210318.zgc.2.png](https://img-blog.csdnimg.cn/2021031900413398.png)

### 优缺点 ###
- 停顿时间短，通常1ms左右。
- 吞吐量下降。
  - 单代回收每次处理的对象更多。
  - 读屏障cpu开销。

## 参考 ##
- [G1垃圾收集器入门](https://blog.csdn.net/renfufei/article/details/41897113)
- [新一代垃圾回收器ZGC的探索与实践](https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html)
- [Java最前沿技术——ZGC](https://mp.weixin.qq.com/s?__biz=MjM5NTEwMTAwNg==&mid=2650239962&idx=3&sn=5534fe8ad0ad9aa709084b77efd1f402&chksm=befe73fb8989faed4424cd9ba36176edf965d781dbdc2f1059d99b8b8f4c4b840e3e0d41edf2#rd)
- [三种收集器对比](https://www.cnblogs.com/cmt/p/14553189.html)
- [jvm一些参数收集和g1简单认知](https://www.jianshu.com/p/70273489db66)
- [垃圾优先型垃圾回收器调优](https://www.oracle.com/cn/technical-resources/articles/java/g1gc.html)
