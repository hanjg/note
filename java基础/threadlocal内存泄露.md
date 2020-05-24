## 原因 ##
- 泄漏原因：
  - 存在引用Thread->ThreadLocal.ThreadLocalMap->Entry->value。
  - 线程池场景下Thread回收复用，value永远无法被gc。[参考](https://blog.csdn.net/yanluandai1985/article/details/82590336)：<br>![](https://img-blog.csdn.net/20171020200142500?)
```java
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

```

## 使用方式 ##
- 结合方法注解作为线程缓存，校验、上下文传参。
- Threadlocal相关的操作最好放在同一个类中，最后使用remove清除ThreadLocalMap对Entry的引用。
- 校验方法可以传出入参给下一步使用。
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

```