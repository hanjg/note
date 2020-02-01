[toc]
## 静态代理和动态代理 ##
- 静态代理：代理类在**编译期**已经存在在.class文件中。
- 动态代理：程序运行时，通过**反射**机制动态创建而成。
- 如果需要为不同的主题类提供代理，需要一一增加代理类，导致类个数急剧增加，所以需要在运行时创建动态代理。

## 动态代理 ##
### 原理 ###
- jdk动态代理需要主题类实现接口，使用**实现相同接口**的代理类代理主题类。如果主题类没有实现接口，则需要使用cglib动态代理。
- cglib使用**继承**实现代理，针对指定的类创建一个子类，覆盖其中的方法实现增强。无法对final代理。

### jdk ###
- jdk动态代理通过Proxy和InvocationHandler接口实现。
- Proxy：提供创建动态代理类和对象的方法。
```java
    public static Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces) throws IllegalArgumentException

    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException
```

- InvocationHandler：实现增强的业务逻辑。
```java
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
```

- 实例。<br>![180510.AbstractUserDao.png](https://img-blog.csdn.net/20180510112311332)
```java
public class DaoHandler implements InvocationHandler {

    private Object subject;

    public DaoHandler(Object subject) {
        this.subject = subject;
    }

    public DaoHandler() {
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        beforeInvoke();
        Object result = method.invoke(subject, args);
        afterInvoke();
        return result;
    }

    private void beforeInvoke() {
        System.out.println("beforeInvoke");
    }

    private void afterInvoke() {
        System.out.println("afterInvoke");
    }
}

public class Client {

    public static void main(String[] args) {
        AbstractUserDao userDao = new UserDao();
        InvocationHandler handler = new DaoHandler(userDao);

        AbstractUserDao proxy = (AbstractUserDao) Proxy
                .newProxyInstance(AbstractUserDao.class.getClassLoader(), new Class[]{AbstractUserDao.class}, handler);

        System.out.println(proxy.findUser(""));
    }
}

```

### cglib ###
- cglib动态代理通过Enhancer和MethodInterceptor接口实现。
- Enhancer：创建代理对象。
```java
    public Object create() 
```

- MethodInterceptor：实现增强的业务逻辑。
```java
    Object intercept(Object var1, Method var2, Object[] var3, MethodProxy var4) throws Throwable;
```

- 实例。<br>![180510.Client2.png](https://img-blog.csdn.net/20180510122007595)
```java
public class UserDaoProxyHandler implements MethodInterceptor {

    private Object subject;

    public UserDaoProxyHandler(Object subject) {
        this.subject = subject;
    }

    public UserDaoProxyHandler() {
    }

    public Object getInstance() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(this.subject.getClass());
        //设置回调方法
        enhancer.setCallback(this);
        //创建代理对象
        return enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        beforeInvoke();
        Object result = methodProxy.invokeSuper(o, objects);
        afterInvoke();
        return result;
    }


    private void beforeInvoke() {
        System.out.println("beforeInvoke");
    }

    private void afterInvoke() {
        System.out.println("afterInvoke");
    }
}

public class Client {

    public static void main(String[] args) {
        UserDao userDao = new UserDao();
        UserDao proxy = (UserDao) new UserDaoProxyHandler(userDao).getInstance();

        System.out.println(proxy.findUser(""));
    }
}
```