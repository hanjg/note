[toc]
## 原理
- 核心对象Thread和ThreadLocal通过 **ThreadLocalMap** 关联
- ThreadLocalMap中，key为ThreadLocal对象，value为业务对象。
![](https://img-blog.csdn.net/20171020200142500?) 
### set
- 获取ThreadLocalMap变量
```java
public class ThreadLocal<T> {
    public void set(T value) {
        Thread t = Thread.currentThread();
		//获取当前线程的threadlocalmap成员变量
        ThreadLocalMap map = getMap(t);
        if (map != null)
			//存到ThreadLocalMap
            map.set(this, value);
        else
            createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
}

public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}

```

- 将存到threadLocalMap中
	- threadLocalMap类似hashmap，使用数组作为底层结构
	- Entry的key为弱引用
	- hash冲突则再寻址，而不是链表法
```java
    static class ThreadLocalMap {
        private Entry[] table;
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }


        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

			//找到key在table中的下标
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {

                ThreadLocal<?> k = e.get();

                if (k == key) {
					//覆盖已有的key
                    e.value = value;
                    return;
                }

                if (k == null) {
					//替换key为null的value，缓解内存泄露
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
			//未清理节点且容量超过阈值，rehash。如果清理过节点容量肯定变少，无须rehash
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
				//遍历删除key被gc的节点
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
        
}

```

### get
```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
			//获取ThreadLocalMap对象
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
					//清除key
                    e.clear();
					//清除脏节点
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```

### remove
```java
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```


## 内存泄露原因 ##
- ThreadLocalMap的Entry的key 弱引用：只被弱引用的对象会被gc，[key被回收后value无法访问](https://www.jianshu.com/p/dde92ec37bd1)
- ThreadLocalMap的Entry的value强引用：存在强引用Thread -> ThreadLocalMap -> Entry[] -> value，**value无法被回收**。
- jdk自身优化有限：
	- ThreadLocal的get,set会清除key为null的节点
	- 依赖代码触发，如果业务没有调用则会内存泄露，比如ThreadLocal对象被gc

## 使用方式 ##
- 结合方法注解作为线程缓存，校验、上下文传参。
- Threadlocal相关的操作最好放在同一个类中，最后使用 **remove** 清除ThreadLocalMap对Entry的引用。
- Threadlocal定义为static，保持对ThreadLocal的强引用，否则可能被gc无法找到无效对象的入口。
```java
    @Around("@annotation(validateEdit)")
    public Object validateEdit(ProceedingJoinPoint joinPoint, ValidateEdit validateEdit) throws Throwable {
        try {
            MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
            String methodName = methodSignature.getMethod().getName();
            Class<?>[] parameterTypes = methodSignature.getMethod().getParameterTypes();
            Annotation[][] annotations = joinPoint.getTarget().getClass().getMethod(methodName, parameterTypes).getParameterAnnotations();
            Object[] args = joinPoint.getArgs();

            for (int i = 0; i < args.length; ++i) {
                for (Annotation annotation : annotations[i]) {
                    //校验、设置上下文
                }
            }

            return joinPoint.proceed();
        } finally {
            ThreadCacheHolder.remove();
        }
    }

public class ThreadCacheHolder {
    private static final ThreadLocal<Map<Long, x>> CACHE = ThreadLocal.withInitial(HashMap::new);
}

```

- 如果子线程需要共享父线程的ThreadLocal，可使用 InheritableThreadLocal
## 参考
- [ThreadLocal解析](https://blog.csdn.net/zhengguofeng0328/article/details/123643732 "ThreadLocal解析")
- [InheritableThreadLocal](https://blog.csdn.net/windrui/article/details/105132387)