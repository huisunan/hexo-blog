---
title: synchronized锁
date: 2021-05-07 11:39
tags: java
categories: 
---

<!--more-->

# synchronized的实现

## Java对象头

synchronized用的锁是存在Java对象头里的。如果对象是数组类型，则虚拟机用3个字宽（Word）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，1字宽等于4字节，即32bit。

[![g1ozct.png](https://z3.ax1x.com/2021/05/07/g1ozct.png)](https://imgtu.com/i/g1ozct)

Java对象头里的Mark Word里默认存储对象的HashCode、分代年龄和锁标记位。32位JVM的Mark Word的默认存储结构：

[![g1TKBT.png](https://z3.ax1x.com/2021/05/07/g1TKBT.png)](https://imgtu.com/i/g1TKBT)

在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化。Mark Word可能变化为存储以下4种数据：

[![g1T1N4.png](https://z3.ax1x.com/2021/05/07/g1T1N4.png)](https://imgtu.com/i/g1T1N4)

## 锁的升级与对比

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。

### 偏向锁

HotSpot的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

- 偏向锁的撤销需要等待全局安全点，暂停持有该锁的线程，同时检查该线程是否还在执行该方法，如果是，则升级锁。反之则其他线程抢占。

- 即如果线程在全局安全点检查时，还需要使用该锁 则进行锁升级，如果线程已经不需要使用锁，并有其他线程需要使用时，将偏向锁的拥有者切换为另外线程。  
  [![g1TUu6.png](https://z3.ax1x.com/2021/05/07/g1TUu6.png)](https://imgtu.com/i/g1TUu6)

### 轻量级锁

轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。  
[![g1TD4H.png](https://z3.ax1x.com/2021/05/07/g1TD4H.png)](https://imgtu.com/i/g1TD4H)

因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时，都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

[![g1T6gI.png](https://z3.ax1x.com/2021/05/07/g1T6gI.png)](https://imgtu.com/i/g1T6gI)

## 锁的优缺点

[![g1T2KP.png](https://z3.ax1x.com/2021/05/07/g1T2KP.png)](https://imgtu.com/i/g1T2KP)

## synchronized修饰

### 修饰代码块\(实例\)

```java
public  class Test {
    private int i ;
    public void demo(){
        synchronized (this){
            i++;
        }
    }
}

```

通过javap \-verbose XXX.class命令查看class文件信息来具体分析两者实现上的差异。

```javascript
 public void demo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter   #进入同步代码块
         4: aload_0
         5: dup
         6: getfield      #2                  // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2                  // Field i:I
        14: aload_1
        15: monitorexit   #正常退出同步代码块
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit   #异常退出同步代码块
        22: aload_2
        23: athrow
        24: return
      Exception table:
         from    to  target type
             4    16    19   any
            19    22    19   any
```

### 修饰方法\(静态方法\)

class文件

```java
public  class Test {
    private int i ;
    public synchronized void demo(){
       i++;
    }
}
```

汇编文件

```javascript
public synchronized void demo();
    descriptor: ()V
    //ACC_SYNCHRONIZED 被修饰的方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
```

synchronized修饰方法并没有通过插入monitorentry和monitorexit指令来实现，而是在方法表结构中的访问标志（access\_flags\)设置ACC\_SYNCHRONIZED标志来实现。线程在执行方法前先判断access\_flags是否标记ACC\_SYNCHRONIZED，如果标记则在执行方法前先去获取monitor对象，获取成功则执行方法代码且执行完毕后释放monitor对象，获取失败则表示monitor对象被其他线程获取从而阻塞当前线程。