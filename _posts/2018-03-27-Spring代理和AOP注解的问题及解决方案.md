---
layout: post
title: Spring代理和AOP注解的问题及解决方案
categories: 技术总结
author: whiteesky
date: 2018-03-27 21:01:10
tags:
  - Java
  - Spring
  - AOP
header-img: img/article/spring_aop/header.jpg
copyright: true
---
@Transactional @Async等注解不起作用
----------------------------

之前很多人在使用Spring中的@Transactional, @Async等注解时，都多少碰到过注解不起作用的情况。

为什么会出现这些情况呢？因为这些注解的功能实际上都是Spring AOP实现的，而其实现原理是通过代理实现的。

JDK动态代理
-------

以一个简单的例子理解一下JDK动态代理的基本原理：

```java
//目标类接口
public interface JDKProxyTestService {
    void run();
}

//目标类
public class JDKProxyTestServiceImpl implements JDKProxyTestService {
    public void run(){
        System.out.println("do something...");
    }
}

//代理类
public class TestJDKProxy implements InvocationHandler {

    private Object targetObject; //代理目标对象

    //构造代理对象
    public Object newProxy(Object targetObject) {
        this.targetObject = targetObject;
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(),
                targetObject.getClass().getInterfaces(), this);
    }

    //利用反射，在原逻辑上进行逻辑增强
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        //模拟事务开始
        assumeBeginTransaction();
        //原执行逻辑
        Object ret = method.invoke(targetObject, args);
        //模拟事务提交
        assumeCommitTransaction();
        return ret;
    }

    private void assumeBeginTransaction() {
        System.out.println("模拟事务开始...");
    }

    private void assumeCommitTransaction() {
        System.out.println("模拟事务提交...");
    }
}

//测试
public class Test {  
    public static void main(String[] args) {
        TestJDKProxy jdkProxy = new TestJDKProxy();
        JDKProxyTestService proxy = (JDKProxyTestService) jdkProxy.newProxy(new JDKProxyTestServiceImpl());
        proxy.run();
    }
}
```

上面的例子应该能够清楚的解释JDK动态代理的原理了。它利用反射机制，生成了一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。我们通过代理类对象调用方法时，实际上会先调用其invoke方法，里面再调用原方法。这样我们可以在原方法逻辑的前后统一添加处理逻辑。

Spring还有一种动态代理方式是CGLIB动态代理。它是把代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。虽然处理方式不一样，但是代理的思想都是一致的。

如果被代理的目标对象实现了接口，那么Spring会默认使用JDK动态代理。所有该目标类型实现的接口都将被代理。若该目标对象没有实现任何接口，则创建一个CGLIB代理。



Spring AOP注解失效及解决
-----------------

基于以上对于动态代理原理的分析，我们来看以下两个常见的问题：

#### 同一个类中，方法A调用方法B（方法B上加有注解），注解无效

针对所有的Spring AOP注解，Spring在扫描bean的时候如果发现有此类注解，那么会动态构造一个代理对象。

如果你想要通过类X的对象直接调用其中带注解的A方法，此注解是有效的。因为此时，Spring会判断你将要调用的方法上存在AOP注解，那么会使用类X的代理对象调用A方法。

但是假设类X中的A方法会调用带注解的B方法，而你依然想要通过类X对象调用A方法，那么B方法上的注解是无效的。因为此时Spring判断你调用的A并无注解，所以使用的还是原对象而非代理对象。接下来A再调用B时，在原对象内B方法的注解当然无效了。

**解决方法：**

最简单的方式当然是可以让方法A和B没有依赖，能够直接通过类X的对象调用B方法。
但是很多时候可能我们的逻辑拆成这样写并不好，那么就还有一种方法：想办法手动拿到代理对象。

AopContext类有一个currentProxy()方法，能够直接拿到当前类的代理对象。那么以上的例子，就可以这样解决：

```java
//加入注解，使得我们能够拿到代理对象
@EnableAspectJAutoProxy(exposeProxy = true)

// 在A方法内部调用B方法
// 1.直接调用B，注解失效。
B()
// 2.拿到代理类对象，再调用B。
((X)AopContext.currentProxy()).B()
```


#### AOP注解方法里使用@Autowired对象为null

在之前的使用中，出现过在加上注解的方法中，使用其他注入的对象时，发现对象并没有被注入进来，为null。

最终发现，导致这种情况的原因是因为方法为private。因为Spring不管使用的是JDK动态代理还是CGLIB动态代理，一个是针对实现接口的类，一个是通过子类实现。无论是接口还是父类，显然都不能出现private方法，否则子类或实现类都不能覆盖到。

如果方法为private，那么在代理过程中，根本找不到这个方法，引起代理对象创建出现问题，也导致了有的对象没有注入进去。

所以如果方法需要使用AOP注解，请把它设置为非private方法。
