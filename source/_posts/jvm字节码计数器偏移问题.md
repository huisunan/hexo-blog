---
title: jvm字节码计数器偏移问题
date: 2021-06-11 10:30
tags: java
categories: 
---

<!--more-->

```class
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: bipush        10
         2: istore_1
         3: invokestatic  #7                  // Method getMessage:()Ljava/lang/String;
         6: astore_2
         7: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        10: aload_2
        11: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        14: bipush        10
        16: bipush        15
```

这里的0直接到2了，为什么不是递增的，不是1呢？  
这里的行号代表的是偏移量，bipush 后面还跟着了一个byte，所以对应的偏移量应该是1+1；类推invokestatic方法，后面跟着两个字节的方法地址，所以偏移量是1+2个字节