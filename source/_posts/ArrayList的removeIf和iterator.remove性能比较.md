---
title: ArrayList的removeIf和iterator.remove性能比较
date: 2021-06-22 20:09
tags: java
categories: 
---

<!--more-->

## 测试代码

```java
package com.example.springtestsuanfa;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class ArrayListTest {
    List<Integer> list = new ArrayList<>();
    long start;
    @BeforeEach
    public void init(){
        for (int i = 0; i < 100_0000; i++) {
            list.add(i);
        }
        start = System.currentTimeMillis();
    }

    @AfterEach
    public void end(){
        System.out.println("用时:" + (System.currentTimeMillis() - start));
    }

    @Test
    public void removeArrayList(){
        list.removeIf((item)->item % 2 == 0);
    }

    @Test void removeIterator(){
        Iterator<Integer> iterator = list.iterator();
        while (iterator.hasNext()) {
            Integer next = iterator.next();
            if (next % 2 == 0){
                iterator.remove();
            }
        }
    }
}

```

100\_0000份数据，删除一半数据  
removeIf\(\) 22ms  
iterator.remove 39962ms

## removeIf源码分析

removeIf分为两部，先标记后整理  
使用BitSet标记要删除的位置

```java
    @Override
    public boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        // figure out which elements are to be removed
        // any exception thrown from the filter predicate at this stage
        // will leave the collection unmodified
        int removeCount = 0;
        final BitSet removeSet = new BitSet(size);
        final int expectedModCount = modCount;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            @SuppressWarnings("unchecked")
            final E element = (E) elementData[i];
            if (filter.test(element)) {
                //使用位图标记删除掉的元素
                removeSet.set(i);
                removeCount++;
            }
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

        // shift surviving elements left over the spaces left by removed elements
        final boolean anyToRemove = removeCount > 0;
        if (anyToRemove) {
            final int newSize = size - removeCount;
            //整理,并将值交换，将未被删除的值覆盖掉已删除的值
            for (int i=0, j=0; (i < size) && (j < newSize); i++, j++) {
                i = removeSet.nextClearBit(i);
                elementData[j] = elementData[i];
            }
            //去掉引用，让gc回收掉被删除的对象
            for (int k=newSize; k < size; k++) {
                elementData[k] = null;  // Let gc do its work
            }
            this.size = newSize;
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            modCount++;
        }

        return anyToRemove;
    }
```

## iterator

```java
    @Test
    void removeIterator() {
        Iterator<Integer> iterator = list.iterator();
        while (iterator.hasNext()) {
            Integer next = iterator.next();
            if (next % 2 == 0) {
                iterator.remove();
            }
        }
    }


        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                //调用ArrayList的remove方法
                ArrayList.this.remove(lastRet);
                //设置游标的位置
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        //数组拷贝
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
    
```

## 时间复杂度

removeIf的时间复杂度是O\(n\)  
iterator.remove的时间复杂度是O\(n²\)