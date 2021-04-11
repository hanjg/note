## 原因 ##
- 泄漏原因：
  - ThreadLocalMap的key ThreadLocal对象为弱引用（只被弱引用的对象会被gc），[key被回收后value无法访问](https://www.jianshu.com/p/dde92ec37bd1)。<br>![](https://img-blog.csdn.net/20171020200142500?) 
  - 存在强引用Thread -> ThreadLocalMap -> Entry[]<ThreadLocal,?> -> value，**value无法被回收**。
  
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
	}
```

- jdk自身优化有限：get,set在hash冲突时可能会访问到key为null的entry，将其清除。但是也仅是可能。
```java
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
					//清除脏泄露对象
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

## 使用方式 ##
- 结合方法注解作为线程缓存，校验、上下文传参。
- Threadlocal相关的操作最好放在同一个类中，最后使用 **remove** 清除ThreadLocalMap对Entry的引用。
- Threadlocal定义为static，保持对ThreadLocal的强引用。
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

- 如果子线程需要共享父线程的ThreadLocal，可使用 [InheritableThreadLocal](https://blog.csdn.net/windrui/article/details/105132387)。