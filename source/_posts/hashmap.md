---
title: hashmap
date: 2021-05-07 11:20
tags: java
categories: 
---

<!--more-->

## **红黑树定义和性质**

红黑树是一种含有红黑结点并能自平衡的二叉查找树。它必须满足下面性质：

- 性质1：每个节点要么是黑色，要么是红色。
- 性质2：根节点是黑色。
- 性质3：每个叶子节点（NIL）是黑色。
- 性质4：每个红色结点的两个子结点一定都是黑色。
- **性质5：任意一结点到每个叶子结点的路径都包含数量相同的黑结点。**

## **红黑树与平衡二叉树的区别：**

1、红黑树放弃了追求完全平衡，追求大致平衡，在与平衡二叉树的时间复杂度相差不大的情况下，保证每次插入最多只需要三次旋转就能达到平衡，实现起来也更为简单。

2、平衡二叉树追求绝对平衡，条件比较苛刻，实现起来比较麻烦，每次插入新节点之后需要旋转的次数不能预知。

## HashMap 中带有初始容量的构造函数

```java
    public HashMap(int initialCapacity, float loadFactor) {
        //容量小于0报错
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        //取2的次幂
        this.threshold = tableSizeFor(initialCapacity);
    }
     public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }


```

**下面这个方法保证了 HashMap 总是使用2的幂作为哈希表的大小**。

```java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

比如 cap = 17
n  =  16  ---> 0001 0000
n |=  n >>>1;        n>>>1  0000 1000 ;   0001 0000 | 0000 1000    = 0001 1000  将第2位变为1    此时已经有两位为1  
n |=  n >>>2;        n>>>2  0000 0110 ;   0000 0110 | 0001 1000    = 0001 1110  将第3-4位变为1  此时已经有四位为1
.... 以次类推
最后返回32
```

这里会先-1然后向上比自己大的第一个2的次幂的数  
[tableSizeFor图解](https://www.cnblogs.com/xiyixiaodao/p/14483876.html)

| 输入 | 输出 |
| --- | --- |
| 5 | 8 |
| 15 | 16 |
| 16 | 16 |
| 17 | 32 |
| 33 | 64 |

**resize方法**

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //如果容量大于最大容量  则将临界值改为int的最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //新的容量等于旧的容量*2，新的容量要小于最大容量，并且旧的容量要大于等于默认容量（16） 
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
                 //新的临界值为旧的临界值2倍
            newThr = oldThr << 1; // double threshold
    }
    //如果容量为0，初始化容量等于临界值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //容量为0，并且临界也为0，则容量和临界值都重新赋值     
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //新的临界值为0 ，重新计算临界值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        //遍历每个节点
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 进行链表复制
                    // 方法比较特殊： 它并没有重新计算元素在数组中的位置
                    // 而是采用了 原始位置加原数组长度的方法计算得到位置
                    //不需要移动的节点
                    Node<K,V> loHead = null, loTail = null;
                    //需要移动的节点
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                            // 注意：不是(e.hash & (oldCap-1));而是(e.hash & oldCap)
                            // (e.hash & oldCap) 得到的是 元素的在数组中的位置是否需要移动,示例如下
                            // 示例1：
                            // e.hash=10 0000 1010
                            // oldCap=16 0001 0000
                            //   &   =0  0000 0000       比较高位的第一位 0
                            //结论：元素位置在扩容后数组中的位置没有发生改变
                            // 示例2：
                            // e.hash=17 0001 0001
                            // oldCap=16 0001 0000
                            //   &   =1  0001 0000      比较高位的第一位   1
                            //结论：元素位置在扩容后数组中的位置发生了改变，新的下标位置是原下标位置+原数组长度
                            // (e.hash & (oldCap-1)) 得到的是下标位置,示例如下
                            //   e.hash=10 0000 1010
                            // oldCap-1=15 0000 1111
                            //      &  =10 0000 1010
                            //   e.hash=17 0001 0001
                            // oldCap-1=15 0000 1111
                            //      &  =1  0000 0001
                            //新下标位置
                            //   e.hash=17 0001 0001
                            // newCap-1=31 0001 1111    newCap=32
                            //      &  =17 0001 0001    1+oldCap = 1+16
                            //元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：
                            // 0000 0001->0001 0001
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

**treeifyBin方法 链表 转二叉树的方法**

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果tab为空 或者 长度小于转换成红黑树的阈值
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //index   桶的索引
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        //hd头节点   tl尾节点
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
//将链表节点替换为树节点
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
//将新构建链表转为红黑树
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        //如果根据为null   将链表的第一个节点当作根节点
        if (root == null) {
            //根节点的parent为null
            x.parent = null;
            //根节点必须是黑色的
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            //当前节点的hash
            int h = x.hash;
           
            Class<?> kc = null;
            
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                //当前节点的hash值 小于根节点的hash值 将dir标记为-1 
                if ((ph = p.hash) > h)
                    dir = -1;
                //当前节点的hash值 大于根节点的hash值 将dir标记为1
                else if (ph < h)
                    dir = 1;
                //如果等于  comparableClassFor  如果没有实现CompareAble接口   则
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;
                //dir小于等于0   则放在左边   大于0 放在右边
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    //平衡红黑树
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}

static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
            //getGenericInterfaces()方法返回的是该对象的运行时类型“直接实现”的接口，这意味着： 
            //返回的一定是接口。
            //必然是该类型自己实现的接口，继承过来的不算。
            //c.getGenericInterfaces() Type[]
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) {
                // (t = ts[i]) instanceof ParameterizedType 判断ts[i] 是否是泛型类
                //(p = (ParameterizedType)t).getRawType() == Comparable.class 当前接口是否是Comparable接口
                //获取Comparable<>  里的泛型，里面泛型只能有一个  返回该泛型值c
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                     Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}
//如果x为null    或者Calss 不一样则不能比较
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}


static int tieBreakOrder(Object a, Object b) {
    int d;
    //如果a为null  或 b为null  或 类名相同时    此时无法比较
    //获取标识hashCode进行比较
    if (a == null || b == null ||
        (d = a.getClass().getName().
         compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
             -1 : 1);
    return d;
}
```

**put方法**

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table未初始化或者长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 桶中已经存在元素
    else {
        Node<K,V> e; K k;
        // 比较桶中第一个元素(数组中的结点)的hash值相等，key相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
                // 将第一个元素赋值给e，用e来记录
                e = p;
        // hash值不相等，即key不相等；为红黑树结点
        else if (p instanceof TreeNode)
            // 放入树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 为链表结点
        else {
            // 在链表最末插入结点
            for (int binCount = 0; ; ++binCount) {
                // 到达链表的尾部
                if ((e = p.next) == null) {
                    // 在尾部插入新结点
                    p.next = new Node(hash, key, value, null);
                    // 结点数量达到阈值，转化为红黑树, -1 算上当前节点
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    // 跳出循环
                    break;
                }
                // 判断链表中结点的key值与插入的元素的key值是否相等，Key指向的对象相同或者equals相同表示相同的key
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表
                p = e;
            }
        }
        // 表示在桶中找到key值、hash值与插入元素相等的结点
        if (e != null) { 
            // 记录e的value
            V oldValue = e.value;
            // onlyIfAbsent为false或者旧值为null
            if (!onlyIfAbsent || oldValue == null)
                //用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 结构性修改
    ++modCount;
    // 实际大小大于阈值则扩容
    if (++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
} 

```