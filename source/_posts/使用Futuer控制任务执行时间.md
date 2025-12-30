---
title: 使用Futuer控制任务执行时间
date: 2025-02-20 16:55:16
tags:
---

```java
public class ThreadTest {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        Future<Boolean> future = executorService.submit(() -> {
            try {
                TimeUnit.MINUTES.sleep(1);
            } catch (InterruptedException e) {
                System.out.println("Interrupted");
                e.printStackTrace();
            }
            return true;
        });

        try {
            Boolean result = future.get(10, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("timeout");
            future.cancel(true);

        }

        TimeUnit.HOURS.sleep(1);
    }
}
```

输出日志

```shell
java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:204)
	at io.github.hsn.bugtest.ThreadTest.main(ThreadTest.java:23)
java.lang.InterruptedException: sleep interrupted
	at java.base/java.lang.Thread.sleep(Native Method)
	at java.base/java.lang.Thread.sleep(Thread.java:344)
	at java.base/java.util.concurrent.TimeUnit.sleep(TimeUnit.java:446)
	at io.github.hsn.bugtest.ThreadTest.lambda$main$0(ThreadTest.java:14)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
	at java.base/java.lang.Thread.run(Thread.java:840)
timeout
Interrupted
```