## 错误日志 ##
```
严重: Servlet.service() for servlet [taotao-manager] in context with path [] threw exception [Request processing failed; nested exception is java.lang.IllegalStateException: Optional long parameter 'parentId' is present but cannot be translated into a null value due to being declared as a primitive type. Consider declaring it as object wrapper for the corresponding primitive type.] with root cause
java.lang.IllegalStateException: Optional long parameter 'parentId' is present but cannot be translated into a null value due to being declared as a primitive type. Consider declaring it as object wrapper for the corresponding primitive type.
	at org.springframework.web.method.annotation.AbstractNamedValueMethodArgumentResolver.handleNullValue(AbstractNamedValueMethodArgumentResolver.java:238)
	at org.springframework.web.method.annotation.AbstractNamedValueMethodArgumentResolver.resolveArgument(AbstractNamedValueMethodArgumentResolver.java:111)
	at org.springframework.web.method.support.HandlerMethodArgumentResolverComposite.resolveArgument(HandlerMethodArgumentResolverComposite.java:121)
	
......
```

## 错误原因 ##
- 前台只传递了一个参数id。<br>![5.moreparams.png](https://img-blog.csdn.net/20180404104940552)
- 但是后台却接受了两个参数id和parentId，并且这两个类型为基本类型long，不能将null赋值给long类型。
```java
    @RequestMapping("/delete")
    @ResponseBody
    public TaotaoResult deleteContentCategory(long id,long parentId)
```

## 解决方案 ##
- 后端删除不必要的parentId参数。这个参数可以通过id查询获得，这样可以少携带参数，也可以避免parentid和id不匹配的报文攻击。