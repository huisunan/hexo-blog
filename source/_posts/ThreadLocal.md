---
title: ThreadLocal
date: 2021-05-07 11:28
tags: java
categories: 
---

<!--more-->

## ThreadLocal

**ThreadLocal提供一个线程（Thread）局部变量，访问到某个变量的每一个线程都拥有自己的局部变量**

## 常用方法

```java
	//set方法
 	public void set(T value) {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取到ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        //使用的懒加载创建的方式
        //当前线程的ThreadLocalMap对象已经存在则将当前ThreadLocal对象和值放入Map当中
        if (map != null)
            map.set(this, value);
        else
        //创建对象并将当前ThreadLocal对象和值放入Map当中
            createMap(t, value);
    }

	ThreadLocal.ThreadLocalMap threadLocals = null;
	//获取ThreadLocalMap的方法，每个线程都维护了一个ThreaLocalMap对象
 	ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

## ThreadLocalMap类

**ThreadLocalMap类是ThreadLocal的静态内部类，Entry继承了弱引用，下一次GC就会被回收。**

```java
	static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            //TreadLocal对象存储的值
            Object value;
			//k为key
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

**ThreadLocalMap中调用的构造函数**

```java
	ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        	//entry数组的容量  默认为16
            table = new Entry[INITIAL_CAPACITY];
        	//根据hash与上数组的长度减一算出在数组的位置
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        	//将ThreadLocal存储到entry数组中
            table[i] = new Entry(firstKey, firstValue);
        	//记录数组的数据量
            size = 1;
        	//设置扩容阈值，为数组长度的三分之二  threshold = INITIAL_CAPACITY * 2 / 3;  
            setThreshold(INITIAL_CAPACITY);
        }
```

**set方法**

```java
	private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
        	//threadLocalHashCode使用的是斐波那契散列函数
            int i = key.threadLocalHashCode & (len-1);
			
        	//在解决hash冲突时，会依次向下遍历，指到找到插入的位置进行插入
            for (Entry e = tab[i];
                 e != null;
                 //i = nextIndex(i, len)  小于len 则返回 i + 1 大于len则返回0
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
        	//设置新的值
            int sz = ++size;
        	//满足条件进行扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

0x61c88647是斐波那契散列乘数,它的优点是通过它散列\(hash\)出来的结果分布会比较均匀，可以很大程度上避免hash冲突，已初始容量16为例，hash并与15位运算计算数组下标结果如下：

| hashCode | 数组下标 |
| --- | --- |
| 0x61c88647 | 7 |
| 0xc3910c8e | 14 |
| 0x255992d5 | 5 |
| 0x8722191c | 12 |
| 0xe8ea9f63 | 3 |
| 0x4ab325aa | 10 |
| 0xac7babf1 | 1 |
| 0xe443238 | 8 |
| 0x700cb87f | 15 |

总结如下：

1.  对于某一ThreadLocal来讲，他的索引值i是确定的，在不同线程之间访问时访问的是不同的table数组的同一位置即都为table\[i\]，只不过这个不同线程之间的table是独立的。
2.  对于同一线程的不同ThreadLocal来讲，这些ThreadLocal实例共享一个table数组，然后每个ThreadLocal实例在table中的索引i是不同的。

## 内存泄漏问题

[![g1Iu2q.png](https://z3.ax1x.com/2021/05/07/g1Iu2q.png)](https://imgtu.com/i/g1Iu2q)

我们知道Thread运行时，线程的的一些局部变量和引用使用的内存属于Stack（栈）区，而普通的对象是存储在Heap（堆）区。根据上图，基本分析如下：

- 线程运行时，我们定义的TheadLocal对象被初始化，存储在Heap，同时线程运行的栈区保存了指向该实例的引用，也就是图中的ThreadLocalRef
- 当ThreadLocal的set/get被调用时，虚拟机会根据当前线程的引用也就是CurrentThreadRef找到其对应在堆区的实例，然后查看其对用的TheadLocalMap实例是否被创建，如果没有，则创建并初始化。
- Map实例化之后，也就拿到了该ThreadLocalMap的句柄，然后如果将当前ThreadLocal对象作为key，进行存取操作
- 图中的虚线，表示key对ThreadLocal实例的引用是个弱引用

### 内存泄漏分析

根据上一节的内存模型图我们可以知道，由于ThreadLocalMap是以弱引用的方式引用着ThreadLocal，换句话说，就是**ThreadLocal是被ThreadLocalMap以弱引用的方式关联着，因此如果ThreadLocal没有被ThreadLocalMap以外的对象引用，则在下一次GC的时候，ThreadLocal实例就会被回收，那么此时ThreadLocalMap里的一组KV的K就是null**了，因此在没有额外操作的情况下，此处的V便不会被外部访问到，而且**只要Thread实例一直存在，Thread实例就强引用着ThreadLocalMap，因此ThreadLocalMap就不会被回收，那么这里K为null的V就一直占用着内存**。

综上，发生内存泄露的条件是

- ThreadLocal实例没有被外部强引用，比如我们假设在提交到线程池的task中实例化的ThreadLocal对象，当task结束时，ThreadLocal的强引用也就结束了
- ThreadLocal实例被回收，但是在ThreadLocalMap中的V没有被任何清理机制有效清理
- 当前Thread实例一直存在，则会一直强引用着ThreadLocalMap，也就是说ThreadLocalMap也不会被GC

也就是说，如果Thread实例还在，但是ThreadLocal实例却不在了，则ThreadLocal实例作为key所关联的value无法被外部访问，却还被强引用着，因此出现了内存泄露。

> 这里要额外说明一下，这里说的内存泄露，是因为对其内存模型和设计不了解，且编码时不注意导致的内存管理失联，而不是有意为之的一直强引用或者频繁申请大内存。比如如果编码时不停的人为塞一些很大的对象，而且一直持有引用最终导致OOM，不能算作ThreadLocal导致的“内存泄露”，只是代码写的不当而已！