---
title: Redis锁的简单实现
date: 2021-07-05 12:04
tags: java
categories: 
---

<!--more-->

## RedisLock

```java
@Component
public class RedisLock {
    @Autowired
    RedisTemplate<String, String> redisTemplate;
    ValueOperations<String, String> operations;

    @PostConstruct
    public void init() {
        operations = redisTemplate.opsForValue();
    }

    private String lockName = "redisLock";
    //尝试加锁，可以根据自己的需求加上过期时间，也可也设置看门狗来对锁进行延时
    public boolean tryLock() {
        return Boolean.TRUE.equals(operations.setIfAbsent(lockName, Thread.currentThread().getName()));
    }
    //尝试解锁，只能解自己线程的锁
    public boolean tryRelease() {
        String threadName = operations.get(lockName);
        if (Thread.currentThread().getName().equals(threadName)) {
            return Boolean.TRUE.equals(redisTemplate.delete(lockName));
        }
        return false;
    }

}
```

## 竞争线程

```java
@Slf4j
public class LockThread extends Thread {
    RedisLock redisLock;

    public LockThread(RedisLock redisLock) {
        this.redisLock = redisLock;
    }

    @SneakyThrows
    @Override
    public void run() {
        //尝试在3s内获取到锁
        for (int i = 0; i < 30; i++) {
            boolean lock = redisLock.tryLock();
            if (lock) {
                log.info("开始干活");
                TimeUnit.SECONDS.sleep(1);
                log.info("活干完了");
                redisLock.tryRelease();
                break;
            } else {
                log.info("枪锁失败");
                TimeUnit.MILLISECONDS.sleep(100);
            }
        }
    }
}
```

## 测试类

```java
@SpringBootTest
public class RedisLockTest {
    @Autowired
    RedisLock redisLock;

    @Test
    public void test() throws InterruptedException {
        LockThread lockThread = new LockThread(redisLock);
        LockThread lockThread2 = new LockThread(redisLock);
        lockThread.start();
        lockThread2.start();
        lockThread.join();
        lockThread2.join();
    }
}

```

```txt
2021-07-05 12:03:46.723  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:46.723  INFO 29540 --- [       Thread-2] com.study.redis.LockThread               : 开始干活
2021-07-05 12:03:46.825  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:46.936  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.043  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.153  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.261  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.363  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.480  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.588  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.696  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 抢锁失败
2021-07-05 12:03:47.727  INFO 29540 --- [       Thread-2] com.study.redis.LockThread               : 活干完了
2021-07-05 12:03:47.806  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 开始干活
2021-07-05 12:03:48.818  INFO 29540 --- [       Thread-3] com.study.redis.LockThread               : 活干完了

```