---
title: java代理的实现
date: 2021-05-07 11:22
tags: java
categories: 
---

<!--more-->

# Java代理的实现

## 静态代理

###### 一个接口，两个实现类，代理实现类组合真实实现类

静态代理，是一种根据上面的理论，很自然会想到的一种不依赖于其他技术的代理模式实现方式。

**接口**（动态代理也是该接口）

```java
public interface Work {
    void doWork();
}
```

**目标类**（动态代理也是该目标类）

```java
public class WorkImpl implements Work  {
    @Override
    public void doWork() {
        System.out.println("开始工作了");
    }
}
```

**委托类**

```java
public class WorkProxy implements Work{

    private Work work;

    public WorkProxy(Work work){
        this.work = work;
    }
    @Override
    public void doWork() {
        //执行前操作
        System.out.println("开始工作前");
        work.doWork();
        //执行后操作
        System.out.println("开始工作后");
    }
}
```

**测试类**

```java
public class WorkTest {
    public static void main(String[] args) {
        WorkImpl workImpl = new WorkImpl();
        Work work = new WorkProxy(workImpl);
        work.doWork();
    }
}
```

如果使用过静态代理，那么很容易理解，静态代理存在的缺陷。

因此，也就出现了动态代理。

**动态代理的动态, 就是可以动态的切换真实实现类, 也就是说可以一个代理类\(相同的代码, 相同的增强操作\)应对一堆不确定的真实实现类.**

[![g1fLDg.png](https://z3.ax1x.com/2021/05/07/g1fLDg.png)](https://imgtu.com/i/g1fLDg)

## 动态代理

## JDK动态代理

jdk动态代理目标类必须要实现接口

通过java.lang.reflect.Proxy类实现。

动态代理就是为了解决静态代理不灵活的缺陷而产生的。静态代理是固定的，一旦确定了代码，如果委托类新增一个方法，而这个方法又需要增强，那么就必须在代理类里重写一个带增强的方法。而动态代理可以灵活替换代理方法，动态就是体现在这里。

**委托类**

```java
public class Invocation implements InvocationHandler {
    /**
     * 实际目标类
     */
    Object realObject;

    public Invocation(Object realObject){
        this.realObject = realObject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("工作前");
        Object invoke = method.invoke(realObject, args);
        System.out.println("工作后");
        return invoke;
    }
}
```

**测试类**

```java
/**
 * jdk原生动态代理
 * 目标类必须要实现接口
 */
public class WorkTest {
    public static void main(String[] args) {
        WorkImpl work = new WorkImpl();
        Class<? extends WorkImpl> workClass = work.getClass();
        Work o = (Work) Proxy.newProxyInstance(workClass.getClassLoader(), workClass.getInterfaces(), new Invocation(work));
        o.doWork();
    }
}
```

## CGLib动态代理

CGLib动态代理是一个第三方实现的动态代理类库，不要求被代理类必须实现接口，它采用的是继承被代理类，使用其子类的方式，弥补了被代理类没有接口的不足。

**委托类**

```java
public class Interceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object realObject, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("工作前");
        Object result = methodProxy.invokeSuper(realObject, objects);//目标类和方法调用的参数
        System.out.println("工作后");
        return result;
    }
}
```

**测试类**

```java
/**
 * CGLib 编译时增强
 * 会生成目标类的子类作为代理类
 */
public class WorkTest {
    public static void main(String[] args) {
        //在指定目录下生成动态代理类，我们可以反编译看一下里面到底是一些什么东西
        String path = WorkTest.class.getResource("/").toString();
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY,
                path.substring(path.indexOf("/") + 1));
        Enhancer enhancer = new Enhancer();
        //设置目标类
        enhancer.setSuperclass(WorkImpl.class);
        //设置拦截器
        enhancer.setCallback(new Interceptor());
        //生成代理类
        Work work = (Work) enhancer.create();
        work.doWork();
    }
}

```

### Enhancer

Enhancer指定要代理的目标对象。通过create方法得到代理对象。通过代理实例调用非final方法，方法调用请求会首先转发给MethodInterceptor的intercept

### MethodInterceptor

通过代理实例调用方法，调用请求都会转发给intercept方法进行增强。

### 反编译查看增强的文件

```java
 public final void doWork() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$doWork$0$Method, CGLIB$emptyArgs, CGLIB$doWork$0$Proxy);
        } else {
            super.doWork();
        }
    }

```