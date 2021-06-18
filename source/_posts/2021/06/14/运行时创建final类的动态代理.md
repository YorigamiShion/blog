---
title: 运行时创建final类的动态代理
tags:
  - 研究
  - java
  - cglib
  - jvm
category: java
date: 2021-06-14 00:52:28
---


众所周知，jdk动态代理只能代理实现，而cglib代理需要创建对应类的子类因此不能代理final类。

那么有没有办法强行创建final类的子类呢？如果创建了会有什么后果？

<!-- more -->

## 环境

AdoptOpenJDK 1.8, spring boot 2.4.5, spring-boot-starter-aop, spring-boot-starter-test

```java
interface IBase {
    String func();
}

class Base implements IBase {
    @Override
    public String func() {
        return "From Base.class";
    }
}

final class FinalBase implements IBase {
    @Override
    public String func() {
        return "From FinalBase.class";
    }
}
```

## 代理非final类

```java
final MethodInterceptor mi = iv -> {
    if ("func".equals(iv.getMethod().getName())) {
        return "From Proxy";
    }
    return iv.proceed();
};

public void proxyBase() {
    Base target = new Base();
    ProxyFactory factory = new ProxyFactory(target);
    factory.setProxyTargetClass(true);
    factory.addAdvice(mi);
    Base proxy = (Base) factory.getProxy();

    Assertions.assertEquals("From Proxy", ((IBase) proxy).func());
    Assertions.assertEquals("From Proxy", proxy.func())
    Assertions.assertEquals("From Proxy", IBase.class.getMethod("func").invoke(proxy));;
    Assertions.assertEquals("From Proxy", Base.class.getMethod("func").invoke(proxy));
}
```

结果符合预期，无论是静态调用还是反射动态调用都成功走了动态代理的逻辑。

## 代理final类

### 生成子类时抛出异常

```java
public void proxyFinalBase() {
    FinalBase target = new FinalBase();
    ProxyFactory factory = new ProxyFactory(target);
    factory.setProxyTargetClass(true);
    factory.addAdvice(mi);
    FinalBase proxy = (FinalBase) factory.getProxy();
}
```

出现异常如下

```
Caused by: java.lang.IllegalArgumentException: Cannot subclass final class demo.FinalBase
	at org.springframework.cglib.proxy.Enhancer.generateClass(Enhancer.java:660)
	at org.springframework.cglib.core.DefaultGeneratorStrategy.generate(DefaultGeneratorStrategy.java:25)
	at org.springframework.cglib.core.ClassLoaderAwareGeneratorStrategy.generate(ClassLoaderAwareGeneratorStrategy.java:39)
	at org.springframework.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:358)
	at org.springframework.cglib.proxy.Enhancer.generate(Enhancer.java:585)
    ......
```

在 `TypeUtils.isFinal(sc.getModifiers())` 中因为检测到class的modifiers含有final标识抛出了这个异常。而modifiers的判断是通过查询target对象中klass指针指向的对象偏移一定字节后的某一位决定的，这就需要了解java对象和klass对象的结构。

### java对象和klass结构

#### java引用类型对象

对于非数组对象，指向地址的前8个字节为MarkWord，之后4/8个字节(根据是否开启指针压缩)为klass指针，而class对象的modifiers就在klass指针指向的对象之中。

#### klass结构

| Index | Field Name | Size | Offset | Field Type |
| ----- | ------------ | ---- | ------ | ---------- |
| 1 | Header | 8 | +0 | Unknown |
| 2 | Klass | Pointer Size | +8 | `*Klass` to java/lang/Class |
| 3 | C++ Vtbl | Pointer Size | +16 | Unknown Pointer |
| 4 | Layout Helper | 4 | +24 | i32 |
| 5 | Super Offset | 4 | +28 | u32 |
| 6 | Name | Pointer Size | +32 | `*Symbol` (linked list type?)|
| 7 | Secondary Super Cache | Pointer Size | +40 | `*Klass` |
| 8 | Secondary Supers | Pointer Size | +48 | `Array<*Klass>` |
| 9 | Primary Supers | Pointer Size * 8 | +56 | `*Klass[]` | 
| 10 | Java Mirror | Pointer Size | +120 | `oop` (pointer type, representation of this class's `java/lang/Class` instance) |
| 11 | Super | Pointer Size | +128 | `*Klass` | 
| 12 | First Subclass | Pointer Size | +126 | `*Klass` or `NULL` |
| 13 | Sibling | Pointer Size | +144 | `*Klass` for linked list |
| 14 | Modifiers | 4 | +152 | i32 |
| 15 | Access Flags | 4 | +156 | u32 |

### 通过Unsafe修改内存移除final修饰后

```java
public void proxyFinalBase() {
    // 先通过反射获取unsafe实例。
    Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
    theUnsafe.setAccessible(true);
    Unsafe unsafe = (Unsafe) theUnsafe.get(null);


    FinalBase target = new FinalBase();
    // 从target对象起始位置偏移8字节开始读取一个int，得到压缩后的klass指针值，左移3位得到实际的地址
    long klassPtr = ((long) unsafe.getInt(target, 8L)) << 3;
    // 修改Modifiers 
    unsafe.putInt(klassPtr + 152, unsafe.getInt(klassPtr + 152) & ~Modifier.FINAL);
    // 修改Access Flags，ClassLoader.defineClass方法会检测这里
    unsafe.putInt(klassPtr + 156, unsafe.getInt(klassPtr + 156) & ~Modifier.FINAL);

    ProxyFactory factory = new ProxyFactory(target);
    factory.setProxyTargetClass(true);
    factory.addAdvice(mi);
    FinalBase proxy = (FinalBase) factory.getProxy();

    Assertions.assertEquals("From Proxy", ((IBase) proxy).func());
    Assertions.assertEquals("From FinalBase.class", proxy.func())
    Assertions.assertEquals("From Proxy", IBase.class.getMethod("func").invoke(proxy));;
    Assertions.assertEquals("From FinalBase.class", FinalBase.class.getMethod("func").invoke(proxy));
}
```

通过修改内存后的确是可以生成子类了，但是这里出现了一个很神奇的现象：根据proxy对象的静态类型不同，为IBase时的直接和反射调用都触发了动态代理逻辑，而以FinalBase为静态类型时都是没有触发动态代理逻辑的。这涉及到了jvm字节码和多态性的具体实现。

## jvm字节码和多态

### 方法调用的字节码

在jvm中，方法调用对应的字节码有一下几种：

- **invokespecial** 调用构造方法、private方法和super方法，直接跳转对应方法地址
- **invokevirtual** 调用一个class对象除了invokespecial的部分以外的所有方法，动态分发，最常用
- **invokeinterface** 调用接口的方法，动态分发
- **invokestatic** 调用静态方法
- **invokedymatic** 一般用于lambda表达式

### 多态的实现

和c++不同，java默认所有方法都是带有virtual的(当然virtual private方法是没意义的)，而实现方式也大体上一致，分为itable和itable两种。

**itable** 记录了一个class对象实现的每一个接口和实现这个接口的实际方法的地址。储存结构类似于`{interface:{method:address,...},...}`，会根据静态代码中的接口类型去查找当前对象所实现的方法地址。

**vtable** 实现方式和c++没什么差别，每当一个class对象构造时，会把`{method:address,...}`覆盖到他基类的储存结构中，在多层级继承的情况下，相应的方法层次覆盖，在调用时再去查找这个方法的实际地址。

### 例子中的问题

HotSpot VM的静态分析会观察到final class，尽管FinalBase对象的方法调用字节码是`INVOKEVIRTUAL demo/FinalBase.func ()Ljava/lang/String;`，但实际上会采用一个名为fast_invokevfinal的内部字节码，这个不需要查vtable获取方法地址而是直接跳转调用。所以当proxy对象的静态类型是FinalBase时，jvm会忽略proxy对象的实际方法也就是动态代理的地址，而是会让程序跳转到FinalBase中定义的方法，也就是导致了例子中的情况。

而当proxy对象的静态类型时IBase时，方法调用的字节码是`INVOKEINTERFACE demo/IBase.func ()Ljava/lang/String; (itf)`，查找了实际方法地址，也就触发了动态代理的逻辑，输出了修改后的结果。

## 感想

只是突发奇想试着继承一下final class，结果牵扯出那么多东西。
