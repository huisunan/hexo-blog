---
title: Jvm年轻代复制到Survivor To区时，对象存放不下会发生什么？
date: 2021-07-01 17:41
tags: java
categories: 
---

<!--more-->

如果To区放不下会直接晋升到老年代

```c++
oop DefNewGeneration::copy_to_survivor_space(oop old) {
  assert(is_in_reserved(old) && !old->is_forwarded(),
         "shouldn't be scavenging this oop");
  size_t s = old->size();
  oop obj = NULL;
 
  // Try allocating obj in to-space (unless too old)
  if (old->age() < tenuring_threshold()) {
    //如果对象的年龄低于tenuring_threshold，则该在to区申请一块同样大小的内存
    obj = (oop) to()->allocate_aligned(s);
  }
 
  // Otherwise try allocating obj tenured
  if (obj == NULL) {
    //如果如果对象的年龄大于tenuring_threshold或者to区申请内存失败
    //则尝试将该对象复制到老年代
    obj = _next_gen->promote(old, s);
    if (obj == NULL) {
      //复制失败
      handle_promotion_failure(old);
      return old;
    }
  } else {
    //to区中申请内存成功
    const intx interval = PrefetchCopyIntervalInBytes;
    Prefetch::write(obj, interval);
 
    //对象复制
    Copy::aligned_disjoint_words((HeapWord*)old, (HeapWord*)obj, s);
 
    //增加年龄，并修改age_table，增加对应年龄的总对象大小
    //注意此处是增加复制对象而非原来对象的分代年龄
    obj->incr_age();
    age_table()->add(obj, s);
  }
 
  //将对象头指针指向新地址
  old->forward_to(obj);
 
  return obj;
}
 
```