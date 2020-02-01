[toc]
## 分类 ##
### compile ###
- 默认就是compile。compile表示被依赖项目需要参与当前项目的编译，当然后续的测试，运行周期也参与其中，是一个比较强的依赖。打包的时候通常需要包含进去。

### test ###
- test依赖项目仅仅参与**测试相关的工作**，包括测试代码的编译，执行。比较典型的如junit。

### runntime ###
- runntime表示被依赖项目**无需参与项目的编译**，不过后期的测试和运行周期需要其参与。与compile相比，跳过编译而已，说实话在终端的项目（非开源，企业内部系统）中，和compile区别不是很大。比较常见的如JSR×××的实现，对应的API jar是compile的，具体实现是runtime的，compile只需要知道接口就足够了。oracle jdbc驱动架包就是一个很好的例子，一般scope为runntime。另外runntime的依赖通常和optional搭配使用，optional为true。我可以用A实现，也可以用B实现。

### provided ###
- provided意味着**打包的时候可以不用包进去**，别的设施(Web Container)会提供。事实上该依赖理论上可以参与编译，测试，运行等周期。相当于compile，但是在打包阶段做了exclude的动作。

### system ###
- 从参与度来说，也provided相同，不过被**依赖项不会从maven仓库抓，而是从本地文件系统拿**，一定需要配合systemPath属性使用。

## 依赖传递 ##
- A–>B–>C。当前项目为A，A依赖于B，B依赖于C。
- 当**C是test或者provided时，C直接被丢弃**，A不依赖C； 否则A依赖C，C的scope继承于B的scope。

[http://blog.csdn.net/kimylrong/article/details/50353161](http://blog.csdn.net/kimylrong/article/details/50353161)