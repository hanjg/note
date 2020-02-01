## 时间+随机数 ##
- 使用精确到毫秒级的系统时间，拼接上随机数作为id。
- 优点：
    - 实现简单。
- 缺点：
    - 有小概率会产生id的重复。

## UUID ##
- 通过jdk生成的全局唯一id。
- 优点：
    - **本地**生成，没有I/O，性能高。
- 缺点：
    - 128**长度**过长，存储空间和可读性较差。
    - 不能生成**递增**有序数字。
- 适用：
    - 不担心过多空间，不需要生成递增趋势的数据。

## 数据库自增主键 ##
- 例如mysql中 **AUTO_INCREMENT** 标识的列。
- 优点：
    - 数据库自动生成，生成**简单**，有序**递增**，方便排序和分页。
- 缺点：
    - 分库分表之后id很可能重复。
    - 数据备份恢复，id会改变。
    - 并发性能不高。
    - 数据库宕机不可用。
- 适用：
    - 数据量不多，并发量不大。
    - 对顺序递增强依赖。

## redis ##
- 使用redis的incr命令获得id。
- 优点：
    - 效率高。
    - 可递增。
- 缺点：
    - 可能存在数据丢失造成id重复。
    - 需要redis服务器的支持，依赖redis的稳定性。
- 适用：
    - 可接受id的重复和不稳定。
    - 隔一段时间刷新id（比如一天）。

## 数据库分段+服务缓存ID ##
- 数据库为每个业务分配id总量和缓存步长。**代理服务器**从数据库取出id，每个服务从代理服务器获取id。<br>![](https://mmbiz.qpic.cn/mmbiz_png/WLIGprPy3z4f85tXcyB4ySnN0xjn0azkLAFPs0I4USaOGLQxe3xa7g2kPt27Sbf08Zwex1uWib42NmSP240JxrQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
- 优点：
    - 自增趋势，且比数据库自增主键性能高。
    - db宕机缓存可支持一段时间。
- 缺点：
    - 主键递增容易被猜。
    - 依赖数据库。
- 适用：
    - 需要趋势递增，并且ID大小可控制。

## 雪花算法 ##

## 参考 ##
- [id生成方式](https://mp.weixin.qq.com/s?__biz=MzIxNjA5MTM2MA==&mid=2652435058&idx=1&sn=4b38ff7bd94734afd63acf3824b368b7&chksm=8c620dfdbb1584eb5af06c3bba4bfa09532f0d201295d68557172ec29adc5fd464298ed64d8d&mpshare=1&scene=23&srcid=1017MQwqiW8B6anIMVWvPmJW#rd)。