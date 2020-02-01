[toc]
## UML ##
- 统一建模语言（Unified Modeling Language， UML），在**面向对象**开发系统的过程中进行设计和分析。
- 可分为静态视图和动态视图，共九种。
    - 静态图：**用例图**，**类图**，对象图，构件图，部署图。
    - 动态图：**时序图**，协作图，状态图，活动图。

## EA12 ##
- 以Enterprise Architect 12作图，描述taotao-sso工程([https://github.com/hanjg/taotao](https://github.com/hanjg/taotao))。
- 使用到的9种图创建方式。<br>![180426.struct.png](https://img-blog.csdn.net/20180426192657157)<br>![180426.behav.png](https://img-blog.csdn.net/20180426192725672)

## 九种UML图 ##
### 用例图（UseCase Diagrams） ###
- 描述系统的使用者和功能。
    - 参与者：使用系统的**角色**，人或者系统。
    - 用例：系统提供的**功能**，通常需要用例的详细说明。
- 用例图。<br>![180426.usecase.png](https://img-blog.csdn.net/20180426120918708)
- 登录用例说明。<br>![180426.login.png](https://img-blog.csdn.net/20180426114840463)

### 类图（Class Diagrams） ###
- 描述系统中类的**内部结构**和类之间的**静态关系**，常见的类的关系有6种：依赖<关联<聚合<组合<泛化=实现，[类关系的详细说明](https://blog.csdn.net/qq_40369829/article/details/80093572)。
- 类图。<br>![180426.class.png](https://img-blog.csdn.net/20180426151611755)

### 对象图（Object Diagrams） ###
- 描述一组对象之间的联系，是系统状态的某一时刻的**快照**，使用相当有限。
- 对象图。<br>![180426.object.png](https://img-blog.csdn.net/20180426154519505)

### 构件图（Component Diagrams） ###
- 描述各种软件构件之间的依赖关系，可以用来帮助设计系统的整体构架。
- 构件图。<br>![180426.componnet.png](https://img-blog.csdn.net/20180426160951407)

### 部署图（Deployment Diagrams） ###
- 描述软件中的各个组件驻留在什么**硬件**位置，以及这些硬件之间的交互关系。
- 部署图。<br>![180426.deployment.png](https://img-blog.csdn.net/20180426162535926)

### 时序图（Sequence Diagrams） ###
- 描述对象之间的消息交互，强调消息的**时间顺序**，是对用例图的细化。[基本概念](https://blog.csdn.net/shulianghan/article/details/17927131)。
- 用户登录时序图。<br>![180426.sequence.png](https://img-blog.csdn.net/20180426172013174)

### 协作图（Collaboration Diagrams） ###
- 描述对象之间的消息交互，强调**对象的关系**。
- 用户登录协作图。<br>![180426.communication.png](https://img-blog.csdn.net/20180426175632507)

### 状态图（Statechart Diagrams） ###
- 描述对象的所有**状态和状态转移条件**。
- 用户登录状态。<br>![180426.state.png](https://img-blog.csdn.net/2018042619212213)

### 活动图（Activity Diagrams） ###
- 描述了活动之间的**控制流程**。本质上是一种流程图。
- 用户登录活动图。<br>![180426.activity.png](https://img-blog.csdn.net/2018042618493329)