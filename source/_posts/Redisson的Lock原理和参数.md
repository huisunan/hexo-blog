---
title: Redisson的Lock原理和参数
date: 2022-08-09 20:52
tags: java
categories: 
---

<!--more-->

# Reidsson的Lock原理和看门狗机制

## Lock

### 获取锁

调用RedissonClient的getLock方法获取锁

```java
    /**
     * Returns Lock instance by name.
     * <p>
     * Implements a <b>non-fair</b> locking so doesn't guarantees an acquire order by threads.
     * <p>
     * To increase reliability during failover, all operations wait for propagation to all Redis slaves.
     *
     * @param name - name of object
     * @return Lock object
     */
    RLock getLock(String name);
```

getLock方法的实现,会返回一个RedissonLock

```java
    @Override
    public RLock getLock(String name) {
        return new RedissonLock(commandExecutor, name);
    }
```

### ReidssonLock和RLock

ReidssonLock实现了RLock接口  
RLock.lock\(\); 是阻塞式等待的，默认加锁时间是30s；如果业务超长，运行期间会自动续期到30s。不用担心业务时间长，锁自动过期被删掉；加锁的业务只要运行完成，就不会给当前锁续期，即使不手动解锁，锁默认会在30s内自动过期，不会产生死锁问题；

也可以自己指定解锁时间RLock.lock\(10,TimeUnit.SECONDS\)，10秒钟自动解锁，自己指定解锁时间redis不会自动续期；

```
public interface RLock extends Lock, RLockAsync {

    /**
     * Returns name of object
     *
     * @return name - name of object
     */
    String getName();
    
    /**
     * Acquires the lock with defined <code>leaseTime</code>.
     * Waits if necessary until lock became available.
     *
     * Lock will be released automatically after defined <code>leaseTime</code> interval.
     *
     * @param leaseTime the maximum time to hold the lock after it's acquisition,
     *        if it hasn't already been released by invoking <code>unlock</code>.
     *        If leaseTime is -1, hold the lock until explicitly unlocked.
     * @param unit the time unit
     * @throws InterruptedException - if the thread is interrupted
     */
    void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException;

    /**
     * Tries to acquire the lock with defined <code>leaseTime</code>.
     * Waits up to defined <code>waitTime</code> if necessary until the lock became available.
     *
     * Lock will be released automatically after defined <code>leaseTime</code> interval.
     *
     * @param waitTime the maximum time to acquire the lock
     * @param leaseTime lease time
     * @param unit time unit
     * @return <code>true</code> if lock is successfully acquired,
     *          otherwise <code>false</code> if lock is already set.
     * @throws InterruptedException - if the thread is interrupted
     */
    boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;

    /**
     * Acquires the lock with defined <code>leaseTime</code>.
     * Waits if necessary until lock became available.
     *
     * Lock will be released automatically after defined <code>leaseTime</code> interval.
     *
     * @param leaseTime the maximum time to hold the lock after it's acquisition,
     *        if it hasn't already been released by invoking <code>unlock</code>.
     *        If leaseTime is -1, hold the lock until explicitly unlocked.
     * @param unit the time unit
     *
     */
    void lock(long leaseTime, TimeUnit unit);

    /**
     * Unlocks the lock independently of its state
     *
     * @return <code>true</code> if lock existed and now unlocked
     *          otherwise <code>false</code>
     */
    boolean forceUnlock();

    /**
     * Checks if the lock locked by any thread
     *
     * @return <code>true</code> if locked otherwise <code>false</code>
     */
    boolean isLocked();

    /**
     * Checks if the lock is held by thread with defined <code>threadId</code>
     *
     * @param threadId Thread ID of locking thread
     * @return <code>true</code> if held by thread with given id
     *          otherwise <code>false</code>
     */
    boolean isHeldByThread(long threadId);

    /**
     * Checks if this lock is held by the current thread
     *
     * @return <code>true</code> if held by current thread
     * otherwise <code>false</code>
     */
    boolean isHeldByCurrentThread();

    /**
     * Number of holds on this lock by the current thread
     *
     * @return holds or <code>0</code> if this lock is not held by current thread
     */
    int getHoldCount();

    /**
     * Remaining time to live of the lock
     *
     * @return time in milliseconds
     *          -2 if the lock does not exist.
     *          -1 if the lock exists but has no associated expire.
     */
    long remainTimeToLive();
    
}
```

执行加锁操作

 1.     获取线程id
 2.     尝试获取锁

```java
private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        long threadId = Thread.currentThread().getId();
        //先尝试一下获取锁
        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);
        // lock acquired
        //获取到锁直接返回
        if (ttl == null) {
            return;
        }

        CompletableFuture<RedissonLockEntry> future = subscribe(threadId);
        if (interruptibly) {
            commandExecutor.syncSubscriptionInterrupted(future);
        } else {
            commandExecutor.syncSubscription(future);
        }

        try {
            while (true) {
                ttl = tryAcquire(-1, leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    break;
                }

                // waiting for message
                if (ttl >= 0) {
                    try {
                        commandExecutor.getNow(future).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    } catch (InterruptedException e) {
                        if (interruptibly) {
                            throw e;
                        }
                        commandExecutor.getNow(future).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    }
                } else {
                    if (interruptibly) {
                        commandExecutor.getNow(future).getLatch().acquire();
                    } else {
                        commandExecutor.getNow(future).getLatch().acquireUninterruptibly();
                    }
                }
            }
        } finally {
            unsubscribe(commandExecutor.getNow(future), threadId);
        }
//        get(lockAsync(leaseTime, unit));
    }
```

尝试获取锁的代码  
如果没有指定锁的持有时间，则默认使用**internalLockLeaseTime\(30s\)**作为锁的持有时间，该参数在RedissonLock初始化的时候从Reidsson的配置文件中获取

```java
    private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        RFuture<Long> ttlRemainingFuture;
        if (leaseTime != -1) {
            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        } else {
            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                    TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        }
        CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
            // lock acquired
            // ttlRemaining代表获取到了锁，从下面的方法可以看出
            if (ttlRemaining == null) {
                if (leaseTime != -1) {
                    internalLockLeaseTime = unit.toMillis(leaseTime);
                } else {
                    //没有指定释放时间，会进行锁续约
                    scheduleExpirationRenewal(threadId);
                }
            }
            return ttlRemaining;
        });
        return new CompletableFutureWrapper<>(f);
    }
```

加锁方法，这里的加锁和setnx不一样  
使用的lua脚本，一样有原子性  
KEY\[1\]: 锁的key名称  
ARGV\[2\]: 为线程id  
ARGV\[1\]: 为设置的过期时间

 1.     判断key是否存在  
    1.1 当key不存在时，调用hincrby,创建一个hash结构，并对该hash结构的以线程id为key的值进行自增1  
    1.2 设置该key的过期时间，pexpire（pexpire是毫秒级的过期时间，expire是秒级的过期时间）
 2.     如果key值存在  
    2.1 查找该hash结构对应的线程id是否存在，如果存在则进行自增  
    2.2 并重新设置过期时间
 3.     如果key值存在，但对应的线程id不存在，则返回该key的过期时间，代表改锁已被其他线程占有

```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "return redis.call('pttl', KEYS[1]);",
                Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
    }
```

### 看门狗机制

触发条件，在调用RLock.lock时会触发，如果指定了锁的释放时间，则不会自动需要

续约方法的入口  
会判断当前要续约的对象是否已经存在，如果已经存在，则把当前id加入；如果不存在创建一个新的，并调用新的续约操作

```
protected void scheduleExpirationRenewal(long threadId) {
        ExpirationEntry entry = new ExpirationEntry();
        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);//EXPIRATION_RENEWAL_MAP是属于类`RedissonBaseLock`的变量
        if (oldEntry != null) {
            oldEntry.addThreadId(threadId);
        } else {
            entry.addThreadId(threadId);
            try {
                //新的续约操作
                renewExpiration();
            } finally {
                if (Thread.currentThread().isInterrupted()) {
                    cancelExpirationRenewal(threadId);
                }
            }
        }
    }
```

EXPIRATION\_RENEWAL\_MAP是属于类的变量

```
   private static final ConcurrentMap<String, ExpirationEntry> EXPIRATION_RENEWAL_MAP = new ConcurrentHashMap<>();
```

#### renewExpiration

```
private void renewExpiration() {
        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (ee == null) {
            return;
        }
        //添加延时任务，这里的定时任务使用的是hash轮
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                //进行一些参数校验
                if (ent == null) {
                    return;
                }
                //进行一些参数校验
                Long threadId = ent.getFirstThreadId();
                if (threadId == null) {
                    return;
                }
                //进行续约操作，判断当前锁和对应的线程是否存在，如果存在则执行续约操作
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                //成功后的回调
                future.whenComplete((res, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getRawName() + " expiration", e);
                        //如果报错了则从续约操作中移除
                        EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                        return;
                    }
                    
                    if (res) {
                        // reschedule itself
                        //继续调用，添加延时任务
                        renewExpiration();
                    } else {
                        //从延时任务中移除
                        cancelExpirationRenewal(null);
                    }
                });
            }
        //续约间隔，为1/3
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
        
        ee.setTimeout(task);
    }
```

### UnLock释放锁操作

释放锁的实际调用方法

```
    @Override
    public RFuture<Void> unlockAsync(long threadId) {
        RFuture<Boolean> future = unlockInnerAsync(threadId);

        CompletionStage<Void> f = future.handle((opStatus, e) -> {
            //取消续约，将该线程id从map中移除，如果map中的锁也为空了，则将锁记录也移除
            cancelExpirationRenewal(threadId);

            if (e != null) {
                throw new CompletionException(e);
            }
            if (opStatus == null) {
                IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                        + id + " thread-id: " + threadId);
                throw new CompletionException(cause);
            }

            return null;
        });

        return new CompletableFutureWrapper<>(f);
    }
```

## 面试题

### Reidsson的锁用的是setnx吗

不是，Reidsson使用的是lua脚本，使用的结构是hash结构，方便其实现一些可重入锁，读写锁的特性

### Redisson的默认锁持有时间是多久

30s

### Redisson看门狗锁的续约时间是多久

锁持有时间的三分之一

### Redssion什么时候会进行续约

在没有指定锁持有时间的时候会自动续约