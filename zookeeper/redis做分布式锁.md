# Redis做分布式锁

关于分布式锁的相关知识，可参考另一篇博客，[zk做分布式锁原理详解](https://blog.csdn.net/weixin_40149557/article/details/117268491)。这里不做介绍

## setnx实现

### setnx命令介绍

`SETNX`是”**SET** if **N**ot e**X**ists”的简写。意思是如果`key`不存在，将`key`设置值为`value`，如果`key`已经存在，什么也不做。

```
# lockKey不存在，设置成功，返回1
127.0.0.1:6379> setnx lockKey redis
(integer) 1
# lockKey存在，设置失败，返回0
127.0.0.1:6379> setnx lockKey redis123
(integer) 0
127.0.0.1:6379> 
```

因此，根据这个特性可以用该命令实现一个简单的分布式锁，多线程并发的模式下，同时肯定只有一个能设置成功，模拟获取到了分布式锁，否则模拟获取锁失败。

### 代码

```java
@RestController
@Slf4j
public class ProductController {

    @Resource
    private MyLock myLock;

    @Resource
    private ProductService productService;

    @GetMapping("/stock/reduce")
    public String reduceStock(Integer id) throws Exception {
        String lockKey = "product:" + id;
        try {
            if (myLock.getLock(lockKey) {
                productService.reduceStock(id);
            }
        } finally {
            myLock.unLock(lockKey);
        }
        return "OK";
    }
}

@Service
@Slf4j
public class ProductService {

    @Resource
    private ProductMapper productMapper;

    @Transactional
    public void reduceStock(Integer id) throws InterruptedException {
        Product product = productMapper.getProduct(id);
        if (product.getStock() <= 0) {
            log.error("reduce stock failed, out of stock...");
            return;
        }
        int i = productMapper.reduceStock(id);
        if (i == 1) {
            log.info("deduct stock success...");
        } else {
            log.error("deduct stock failed, i = {}", i);
        }

    }
}
```

```java
@Component
@Data
public class MyLock {

    @Resource
    private RedisTemplate<String, String> redisTemplate;
    
    public boolean getLock(String lockKey) {
        // 设置10秒的过期时间，防止因为服务器宕机，导致释放锁失败而造成死锁。
        return redisTemplate.opsForValue().setIfAbsent(lockKey, "OK", 10, TimeUnit.SECONDS);
    }

    public void unLock(String lockKey) {
        redisTemplate.delete(lockKey);
    }
}
```

然而用JMeter做压力测试，发现并不能实现预期的分布式锁效果，却是为何呢？这里其实存在一个锁被误释放的问题。

假设现在多个线程并发执行获取锁的命令，其中线程1执行`setnx lockKey OK`成功拿到锁，并执行了`expire lockKey 5`设置了5秒的过期时间，然后开始干活，但是呢因为业务比较复杂，5秒时间线程1没有干完，然后`lockKey`过期了。这个时候线程2在调用`setnx lockKey OK`自然可以成功了，也设置了5秒的过期时间，然后也开始干活了。当他执行到第二秒的时候，线程1干完了，然后走到了`finally`代码块，将锁释放了，可是这个锁不是你线程1家加的啊，是人线程2加的啊。然后接下来就完全失控了。那么怎么办呢，有办法，就是在加锁的时候，把是那个线程加的锁记下来，等到释放的时候，校验一下，如果释放锁的线程和加锁的线程不是一个线程，那么不允许他释放。

## setnx+lua脚本实现

说干就干，有了下面的代码。

```java
@Component
@Data
public class MyLock {

    @Resource
    private RedisTemplate<String, String> redisTemplate;

    public boolean getLock(String lockKey, String threadName) {
        return redisTemplate.opsForValue().setIfAbsent(lockKey, threadName, 10, TimeUnit.SECONDS);
    }

    public void unLock(String lockKey, String threadName) {
        if (threadName.equals(redisTemplate.opsForValue().get(lockKey))) {
            redisTemplate.delete(lockKey);
        }
    }
}
```

那么这样写是不是可行呢，也是不行的，为什么呢？因为这13、14行是两个命令，不是原子操作啊。这可如何是好，好在redis他是支持lua脚本的。那我们就可以将两个命令整合在一个脚本里，然后将一个脚本丢给redis去执行，这样就保证了原子性。下面给出代码：

```java
@Component
@Data
public class MyLock {

    @Resource
    private RedisTemplate<String, String> redisTemplate;

    public boolean getLock(String lockKey, String threadName) {
        return redisTemplate.opsForValue().setIfAbsent(lockKey, threadName, 10, TimeUnit.SECONDS);
    }

    public void unLock(String lockKey, String threadName) {
        String luaScript = "if redis.call('get',KEYS[1]) == ARGV[1] then\n" +
                "    return redis.call('del',KEYS[1])\n" +
                "else\n" +
                "    return 0\n" +
                "end";
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>(luaScript, Long.class);
        redisTemplate.execute(redisScript, Collections.singletonList(lockKey), Collections.singleton(threadName));
    }
}
```

那么这种方案是不是就是完美的了，当然不是。他还有这样一个问题，就是我们现在设置的过期时间随便给了一个10秒，如果你可以保证所有的业务都能在10秒之内完成那固然没问题，但是你敢保证吗？所以你会想10秒这个时间短了，那么往大一点呢，那么多大合适呢，多大都不合适，因此我们有责任实现一个动态的给**锁续命**的机制。如何续命呢，我们可以在加锁成功后，在开启一个定时任务，然后定时通过`get`命令去获取锁，如果能获取到说明`finally`代码块里的代码还没执行，锁还没释放，业务还在执行中，然后我们就通过`expire`命令给他重新设置一个过期时间。如果获取不到锁，说明`finally`代码块的代码执行了，锁被释放了，业务执行完毕了，我们就结束这个定时任务就行了。

## Redisson实现

这样的逻辑，不用我们写了，已经有人帮我们封装好了，那就是大名鼎鼎的Redisson。我们先看一下如何使用，在看其原理。

### Redisson使用

```xml
<!-- 引入依赖 -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.15.4</version>
</dependency>
```

```java
// 注册redisson的bean
@Configuration
public class RedissonConfigure {

    @Value("${redisson.address}")
    private String address;

    @Bean
    public Redisson redisson() {
        Config config = new Config();
        config.useSingleServer().setAddress(address).setDatabase(0);
        return (Redisson) Redisson.create(config);
    }
}
```

```java
@RestController
@Slf4j
public class ProductController {

    @Resource
    private Redisson redisson;

    @Resource
    private ProductService productService;

    @GetMapping("/stock/reduce")
    public String reduceStock(Integer id) throws Exception {
        RLock lock = redisson.getLock("product:" + id);
        try {
            lock.lock();
            productService.reduceStock(id);
        } finally {
            lock.unlock();
        }
        return "OK";
    }
}
```

### Redisson源码分析

```java
private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
    long threadId = Thread.currentThread().getId();
    // 主要跟踪这行方法，核心参考下文：tryLockInnerAsync的介绍
    Long ttl = tryAcquire(-1, leaseTime, unit, threadId);
    // 如果ttl = null说明加锁成功，直接返回
    if (ttl == null) {
        return;
    }

    RFuture<RedissonLockEntry> future = subscribe(threadId);
    if (interruptibly) {
        commandExecutor.syncSubscriptionInterrupted(future);
    } else {
        commandExecutor.syncSubscription(future);
    }

    try {
        // 如果获取锁失败，就在这个死循环里不断尝试
        while (true) {
            ttl = tryAcquire(-1, leaseTime, unit, threadId);
            // lock acquired
            if (ttl == null) {
                break;
            }

            // waiting for message
            if (ttl >= 0) {
                try {
                    future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } catch (InterruptedException e) {
                    if (interruptibly) {
                        throw e;
                    }
                    future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                }
            } else {
                if (interruptibly) {
                    future.getNow().getLatch().acquire();
                } else {
                    future.getNow().getLatch().acquireUninterruptibly();
                }
            }
        }
    } finally {
        unsubscribe(future, threadId);
    }
    //        get(lockAsync(leaseTime, unit));
}
```

```java
// 跟踪tryAcquire最终走到这里
private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    RFuture<Long> ttlRemainingFuture;
    if (leaseTime != -1) {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    } else {
        // leaseTime = -1，会走到这里  internalLockLeaseTime默认为30秒，这个方法里实现了加锁的逻辑，详细看下文解析
        ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                                               TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    }
    
    // 这里注册了一个监听
    ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
        if (e != null) {
            return;
        }

        // lock acquired
        if (ttlRemaining == null) {
            if (leaseTime != -1) {
                internalLockLeaseTime = unit.toMillis(leaseTime);
            } else {
                // 核心逻辑在这个方法里，看这方法名字计划续订过期时间，结合上文说的大概就是给锁续命的时间了，后面详细解析看看怎么续命的
                scheduleExpirationRenewal(threadId);
            }
        }
    });
    return ttlRemainingFuture;
}

// 跟踪tryAcquire方法会发现核心代码其实在这里lua脚本
// waitTime = -1   leaseTime = 30000   threadId当前线程Id
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
                          										  // 这三行模拟加锁成功
              "if (redis.call('exists', KEYS[1]) == 0) then " +   // existx product:001 = 0 说明product:001不存在
              "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +    // hincrby product:001 9527 1 
              "redis.call('pexpire', KEYS[1], ARGV[1]); " +       // pexpire product:001 30000
              "return nil; " +
              "end; " +
                          													 // 这三行模拟可重入锁的情况
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +    // hexists product:001 9527 = 1 说明是9527这个线程第二次来加锁，可重入锁
              "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +               // hincrby product:001 9527 1 可重入次数加1
              "redis.call('pexpire', KEYS[1], ARGV[1]); " +                  // pexpire product:001 30000
              "return nil; " +
              "end; " +
              "return redis.call('pttl', KEYS[1]);",                         // 能走到这一步，说明加锁失败 获取product:001的过期时间
                          													 // pttl product:001
                          
              // getRawName()是我们在这里redisson.getLock("product:" + id);给的那个值，假设product:001  KEYS[1] = product:001
              // unit.toMillis(leaseTime)  ARGV[1] = 30000
              // getLockName(threadId)给当前线程ID加一个随机数前缀，我们这里假设为9527  ARGV[2] = 9527
              Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
}

// scheduleExpirationRenewal的核心逻辑，每隔internalLockLeaseTime / 3时间单位进行一次锁续命
private void renewExpiration() {
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (ee == null) {
        return;
    }

    Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
            if (ent == null) {
                return;
            }
            Long threadId = ent.getFirstThreadId();
            if (threadId == null) {
                return;
            }
			
            // 前面的可先忽略，重点看这个方法
            RFuture<Boolean> future = renewExpirationAsync(threadId);
            future.onComplete((res, e) -> {
                if (e != null) {
                    log.error("Can't update lock " + getRawName() + " expiration", e);
                    EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                    return;
                }

                if (res) {
                    // 不断的重新调用自身
                    renewExpiration();
                }
            });
        }
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);

    ee.setTimeout(task);
}

protected RFuture<Boolean> renewExpirationAsync(long threadId) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                          "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +   // hexists product:001 9527 = 1 说明锁还在
                          "redis.call('pexpire', KEYS[1], ARGV[1]); " +                 // pexpire product:001 30000  锁还在，就给续命
                          "return 1; " +
                          "end; " +
                          "return 0;",
                          Collections.singletonList(getRawName()),
                          internalLockLeaseTime, getLockName(threadId));
}
```

