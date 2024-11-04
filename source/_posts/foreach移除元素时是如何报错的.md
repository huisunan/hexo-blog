---
title: foreach移除元素时是如何报错的
date: 2021-03-30 22:30
tags: java
categories: 
---

<!--more-->

```java
public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(2);
        list.add(3);
        list.add(4);
        list.add(5);
        for (Integer integer : list) {
            System.out.println(integer);
            if (integer == 5){
                list.remove(new Integer(5));
            }
        }
    }
```

使用javap \-v 工具可以看到foreach语法糖实际的工作原理是使用的是iterator

```java
         0: new           #2                  // class java/util/ArrayList
         3: dup
         4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
         7: astore_1
         8: aload_1
         9: iconst_1
        10: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        13: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
        18: pop
        19: aload_1
        20: iconst_2
        21: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        24: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
        29: pop
        30: aload_1
        31: iconst_3
        32: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        35: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
        40: pop
        41: aload_1
        42: iconst_4
        43: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        46: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
        51: pop
        52: aload_1
        53: iconst_5
        54: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        57: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
        62: pop
        63: aload_1
        64: invokeinterface #6,  1            // InterfaceMethod java/util/List.iterator:()Ljava/util/Iterator;
        69: astore_2
        70: aload_2
        71: invokeinterface #7,  1            // InterfaceMethod java/util/Iterator.hasNext:()Z
        76: ifeq          122
        79: aload_2
        80: invokeinterface #8,  1            // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
        85: checkcast     #9                  // class java/lang/Integer
        88: astore_3
        89: getstatic     #10                 // Field java/lang/System.out:Ljava/io/PrintStream;
        92: aload_3
        93: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        96: aload_3
        97: invokevirtual #12                 // Method java/lang/Integer.intValue:()I
       100: iconst_5
       101: if_icmpne     119
       104: aload_1
       105: new           #9                  // class java/lang/Integer
       108: dup
       109: iconst_5
       110: invokespecial #13                 // Method java/lang/Integer."<init>":(I)V
       113: invokeinterface #14,  2           // InterfaceMethod java/util/List.remove:(Ljava/lang/Object;)Z
       118: pop
       119: goto          70
       122: return
```

编译过来的代码应该是

```java
	while (iterator.hasNext()){
            Integer next = iterator.next();
            if (next == 5){
                list.remove(new Integer(5));
            }
        }
```

首先会调用hasNext\(\)方法进行判断,会判断当前的游标和集合的长度是否相等

```java
 	 public boolean hasNext() {
            return cursor != size;
        }
```

在进行next遍历时,会首先进行校验checkForComodification，这个方法会比较初始化迭代器时数组修改的次数和当前修改的次数是否一致，不一致则报错，初始化迭代器前进行了5次add方法此时modCount的为5，expectedModCount初始话为5，在进行list.remove\(new Integer\(5\)\)后modCount加一后成6，在调用hasNext的时候是返回的true，cursor为5，size是6，cursor \!= 6返回的是true，在进行next\(\)的checkForComodification方法时，modCount=6,expectedModCount =5,modCount\!=expectedModCount 报错

```java
 	public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
	final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
	//ArrayList迭代器初始话方法
 	private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;
	//......
	}
```