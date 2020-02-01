[toc]
## 类生命周期 ##
- java中，类型的加载、连接、初始化都是在程序**运行期**完成，而不是在编译期完成。
- 类的生命周期，包括7个阶段。<br>![180526.life.png](https://img-blog.csdn.net/20180526211557145)
- 为了支持**动态绑定**（根据实例的运行期类型调用相应的方法），**解析阶段**可以在初始化之后开始。


## 类加载时机 ##
- 类的**加载时机没有强制约束**，交由虚拟机把握，**初始化有且只有5种情况**，称为对一个类进行**主动引用**，其他行为不会触发初始化，称为**被动引用**。
- 接口初始化，不要求父接口全部初始化。

### 主动引用 ###
- 遇到new,getstatic,putstatic,invokestatic这4条指令码是，如类没有初始化，则进行初始化。场景包括：
    - new创建实例。
    - 读取或设置类的静态字段（被final static修饰在编译期放入常量池的静态字段除外）。
    - 调用一个类的静态方法。
- 对类进行**反射**调用，如类没有初始化，则进行初始化.
- 初始化一个类，如**父类**没有初始化，则父类进行初始化。
- 虚拟机启动时，用户指定的要执行的**主类**（main方法），需要先初始化这个主类。
- jdk1.7 **动态语言支持** ，MethodHandle实例的解析结果REF_getStatic,REF_putStatic,REF_invokeStatic的方法句柄，并且这个方法句柄对应的类没有初始化，需要对类进行初始化。[MethodHandle](https://blog.csdn.net/aesop_wubo/article/details/48858931)。

### 被动引用 ###
- 通过子类引用父类的静态字段，只会导致父类的初始化，不会导致子类的初始化。（也可以调节参数，使得子类同时被加载）。
- 通过数组定义引用类，不会初始化这个类。
```java
SuperClass[] a = new SuperClass[10];
```

- 常量编译阶段存入调用类的常量池，并没有直接引用到定义常量的类，不会触发类的初始化。

## 类加载过程 ##
### 加载 ###
- 是类加载的一个阶段。包括3件事。
    - 通过类的全限定名获取类的二进制**字节流**。
    - 将字节流代表的静态存储结构转化为方法区**运行时数据结构**。
    - 内存中生成一个该类的**Class对象**，在方法区中，作为这个类数据的访问入口。
- 字节流的来源包括：zip,jar等压缩包；网络；动态代理；jsp生成的Class类；数据库。
- 可以通过自定义的类加载器控制字节流的获取方式，即重写一个类加载器的loadClass方法。

### 验证 ###
- Class文件并不一定是从java源码编译而来，需要校验。
- 包括文件格式、元数据、字节码、符号引用的验证。

### 准备 ###
- 设置变量的初始值，通常是该数据类型的**零值**。
- 如果变量为final，则直接初始化为指定值。

### 解析 ###
- 将常量池中的符号引用替换为直接引用。
    - 符号引用：定义在Class文件中，以一组符号描述所引用的目标，与虚拟机内存布局无关，引用的目标不一定是加载到内存中。
    - 直接引用：可以是指向目标的指针、相对偏移量、间接定位目标的句柄等，与虚拟机内存布局相关，引用的是内存中的目标。
- 包括类或接口、字段、类方法、接口方法的解析。

### 初始化 ###
- 执行代码中的**类构造器**方法。
- 包括静态初始化块、静态变量。
- 虚拟机会保证类构造器在多线程环境中被正确的**加锁**、同步，即只有一个线程在某一时刻执行这个类的类构造器，其他线程阻塞。

## 类加载器 ##
- 两个类相等：加载的**类加载器**相同，**类本身**相同。
- 相同的类在不同加载器中加载，这两个类不同，instanceof的结果为 false 。
- java中的类加载器继承java.lang.ClassLoader这一抽象类。

## 双亲委派模型 ##
### 3种系统提供的类加载器 ###
- 启动类加载器。
    - 由HotSpot中由C++实现，用户无法获得实例。
    - 加载JAVE_HOME\lib目录，或者-Xbootclasspath参数的路径。
- 扩展类加载器。
    - sun.misc.Launcher$ExtClassLoader实现。
    - 加载JAVA_HOME\lib\ext目录，或者java.ext.dirs系统变量指定的路径。
- 应用程序类加载器（系统类加载器）。
    - sun.misc.Launcher$AppClassLoader实现。
    - 是ClassLoader中getSystemClassLoader()的返回值。
    - 加载用户类路径Classpath上的类。

### 委派行为 ###
- ![180529.classloader.png](https://img-blog.csdn.net/2018052917185317)
- 使用**组合**来实现父子关系。
- 如果一个类加载器收到类加载请求，**首先将请求委派给父类**，父类通向如此处理。只有当父类无法处理请求的时候，子类才会处理这个请求。所以所有的加载请求最后都会传递到顶层的启动类加载器。
- 可以使得java类和加载器一起具备了**优先级的层次关系**。例如java.lang.Object类最终都会由顶层的启动类加载器加载，从而保证了唯一性。
- java.lang.ClassLoader的核心方法。
```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   //父类抛出异常，表示父类无法完成加载请求
                }

                if (c == null) {
                    //父类无法加载，调用自身的findClass方法加载
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

### 委派模型的破坏 ###
#### 线程上下文类加载器 ####
- 基础类需要调用价更加具体的代码。如JNDI服务，代码由启动类加载器加载，但是需要调用应用程序类加载器的类提供的接口。
- 线程上下文类加载器可以使用**父类加载器请求子类加载器**去完成类加载。

#### OSGI热部署 ####
- 实现模块化热部署，每一个模块（Bundle）都有自己的类加载器。
- 类加载器发展为**网络**结构。<br>![180529.osgi.png](https://img-blog.csdn.net/20180529202931648)
- 收到类加载请求之后的操作：
    1. 将以java.*开头的类委派给父类加载器。
    2. 否则，将委派列表名单的类委派给父类加载器。
    3. 否则，将Import列表中的类委派给Export这个类的Bundle的类加载器加载。
    4. 否则，查找当前Bundle的ClassPath，使用自己的类加载器加载。
    5. 否则，查找类是否在自己的Fragment Bundle中，如果在，委派给Fragment Bundle的类加载器加载。
    6. 否则，查找Dynamic Import列表的Bundle，委派给对应的Bundle的类加载器加载。
    7. 否则，查找失败。

## 自定义类加载器 ##
- 需要继承抽象类ClassLoader，覆盖findClass()方法，自定义读取class文件的方法，并调用definClass()这一抽象类中的方法。
    - 不建议覆盖loadClass()方法，这样破坏委派模型。
```java
public class MyClassLoader extends ClassLoader {

    private String classpath;

    private String fileType = ".class";

    public MyClassLoader(String classpath) {
        super();
        this.classpath = classpath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] data = getData(name);
        return defineClass(name, data, 0, data.length);
    }

    private byte[] getData(String className) {
        InputStream in = null;
        ByteArrayOutputStream out = null;

        String path = getFilePath(className);
        byte[] data = null;

        try {
            in = new FileInputStream(path);
            out = new ByteArrayOutputStream();
            int c = 0;
            while (-1 != (c = in.read())) {
                out.write(c);
            }
            data = out.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                in.close();
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        return data;

    }

    private String getFilePath(String className) {
        return classpath + "\\" + className.replace('.', '\\') + fileType;
    }
}
```

- 需要加载的类和测试代码如下：
    - 注意加载的类的package名称和路径需要一致否则会报 **NoClassDefFoundError** 。
```java
package hjg.hjg;

public class LoaderTest {
    public LoaderTest() {
    }

    public void say() {
        System.out.println("hello!!!");
    }
}
```
```java
public class TestClassLoader {

    public static void main(String[] args)
            throws ClassNotFoundException, IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
        MyClassLoader classLoader = new MyClassLoader("A:");
        Class<?> loadTest = classLoader.loadClass("hjg.hjg.LoaderTest");

        Object object = loadTest.newInstance();
        Method method = loadTest.getMethod("say", null);
        method.invoke(object, null);
    }
}
```
