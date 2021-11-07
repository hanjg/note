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

。。。。。。。。。