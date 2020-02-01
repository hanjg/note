[toc]
## CAS ##
- mysql的UPDATE，hbase的checkAndPut提供CAS操作。
- redis基于watch和multi也可以实现CAS乐观锁。

### multi ###
- multi：multi开启事务（**非原子性**），包含多个命令。
- exec触发。
- 可以理解为打包的批量执行脚本，某条命令执行失败不会导致之前的命令回滚，后续命令也会继续执行。

### watch ###
- watch：监视key，如事务开始前key被改动，终止事务。
- client1完成操作后会将watch这个key的列表中其他client状态设置为CLIENT_DIRTY_CAS
- client2执行时状态为CLIENT_DIRTY_CAS直接终止事务。
- [原理](https://www.jianshu.com/p/ad273642b3bb)。


## Jedis实现 ##
- 根据 ```transaction.exec()``` 执行结果是否都是OK，判断操作是否成功。
  - 失败则重试或者抛异常。
  - 成功则继续操作。
```java
@Service
public class RedisCAS {

    @Autowired
    private JedisPool jedisPool;

    private String redisKey = "cas_key";

    private int threadNum = 5;

    private ExecutorService service = Executors.newFixedThreadPool(threadNum);

    private CountDownLatch latch = new CountDownLatch(threadNum);

    private AtomicInteger successCount = new AtomicInteger(0);

    private AtomicInteger failCount = new AtomicInteger(0);

    public void execute() {
        //初始化key
        Jedis cli = jedisPool.getResource();
        cli.set(redisKey, String.valueOf(0));
        //开启5个线程
        for (int i = 0; i < threadNum; i++) {
            service.submit(new AddThread("thread-" + i));
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(redisKey + ": " + cli.get(redisKey));
        System.out.println("success: " + successCount.get() + ", fail: " + failCount.get());
    }

    public class AddThread implements Runnable {

        private Jedis client;
        private String name;

        public AddThread(String name) {
            client = jedisPool.getResource();
            this.name = name;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    atomicAdd();
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                client.close();
                latch.countDown();
            }
        }

        private void atomicAdd() {
            while (true) {
                //开启watch
                client.watch(redisKey);
                int target = Integer.parseInt(client.get(redisKey)) + 1;
                //开启事务
                Transaction transaction = client.multi();
                transaction.set(redisKey, String.valueOf(target));
                List<Object> result = transaction.exec();
                //事务结果
                if (ok(result)) {
                    System.out.println(this.name + " multi succussful, value = " + target);
                    successCount.addAndGet(1);
                    break;
                }
                System.out.println(this.name + " multi fail.");
                failCount.addAndGet(1);
            }
        }

        private boolean ok(List<Object> result) {
            return CollectionUtils.isNotEmpty(result) && result.stream().allMatch("OK"::equals);
        }
    }
}

@Configuration
public class RedisConfig {

    @Bean
    public JedisPool jedisPool() {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxIdle(10);
        config.setMaxWaitMillis(1000);
        config.setMaxTotal(30);
        JedisPool jedisPool = new JedisPool(config, "127.0.0.1", 6379);
        return jedisPool;
    }
}

```

## 备选方案 ##
- [redis悲观锁](https://blog.csdn.net/qq_40369829/article/details/87560838#redis_33)。
- 原因：
  - 由于watch需要维护key的客户端观察者列表，key修改之后修改客户端状态会有一定的开销，不适合高并发场景。
  - 某些中间件不支持watch+multi
