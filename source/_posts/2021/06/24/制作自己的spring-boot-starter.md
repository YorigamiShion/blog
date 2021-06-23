---
title: 制作自己的spring-boot-starter
tags:
  - 应用
  - java
  - spring
category: java
date: 2021-06-24 01:57:07
---

编写spring服务时，除了用到官方提供的starter(例如spring-boot-starter-aop)以外，还会用到一些第三方提供的(例如mybatis-spring-boot-starter)。这一篇就将讲述如果制作打包一份自定义的spring-boot-starter，其中包含条件装配和@Enable模式的装配。

<!-- more -->

## 命名

俗话说，计算机科学有两大难题：命名和缓存失效，起一个好名字对于自定义的包来说非常重要。在springboot推荐的命名规范中，官方提供的starter命名为spring-boot-starter-xxx，而第三方提供的命名为yyy-spring-boot-starter，这里就将项目命名为`demo-spring-boot-starter`。

## 项目结构

新建maven项目，只依赖一个spring boot starter即可，内容为

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <packaging>pom</packaging>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.4.7</version>
    <relativePath/>
  </parent>
  <groupId>org.home</groupId>
  <artifactId>demo-spring-boot-starter</artifactId>
  <version>0.1.0</version>
  <name>demo-spring-boot-starter</name>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
  </dependencies>
</project>
```

新建2个接口作为2种自动装配的对外内容。

```java
public interface DemoConditionalService {
    String getValue();
}

public interface DemoEnableService {
    int getResult();
}
```

## spring boot自动装配

spring boot中提供了一系列的自动装配bean，为很多组件所需的bean提供了一个默认值，这些作为默认值的bean之上有很多@Conditional类的注解，其中一个是`@ConditionalOnMissingBean`，这个注解是最好理解的，当它和`@Bean`放在一起的时候，仅会在缺失这个bean的情况下将被指定bean加入到spring容器之中，这样就完成了为bean提供默认值的逻辑。

另一类有代表性的便是@Enable系列，例如`@EnableCaching`会自动载入spring-cache的相关bean，`@EnableAspectJAutoProxy`会根据参数来选择载入jdk proxy的bean还是cglib的bean。这种是通过注解显式地载入了对应的功能。

## @Conditional

`@Conditional`可以作为元注解，也就是加在注解上的注解，例如`@ConditionalOnMissingBean`；或者直接使用。这个注解可以被放置在`@Configuration`类上或者`@Bean`方法上，根据其指定的`Condition`接口的实现，来决定是否载入被注解的配置类或者bean方法。

### @ConditionalOnMissingBean

当指定的Bean不存在时进行载入逻辑。这里为`DemoConditionalService`提供了一个默认值，当其他途径都没有提供bean的实现时就会载入当前的实现。

```java
@ConditionalOnMissingBean
@Bean
public DemoConditionalService defaultDemoConditionalService() {
    return () -> "missing";
}
```

### @ConditionalOnProperty

当指定的配置(`application.yml`或者`-Dkey=value`)符合条件时进行载入，这里根据demo.value的值为one或者two时载入实现，当demo.value不存在时载入two的实现，当demo.value存在且不是以上两个值时不载入任何一个，而是采用默认实现返回missing。

```java
@ConditionalOnProperty(name = "demo.value", havingValue = "one")
@Bean
public DemoConditionalService defaultDemoConditionalService() {
    return () -> "oneone";
}

@ConditionalOnProperty(name = "demo.value", havingValue = "two", matchIfMissing = true)
@Bean
public DemoConditionalService defaultDemoConditionalService() {
    return () -> "twotwo";
}
```

### 引入自动装配

将以上三个方法放在同一个配置类中，命名为`org.home.demo.spring.boot.starter.DemoConfiguration`。在`resources/META-INF`中新建文件`spring.factories`(这个由`SpringFactoriesLoader.FACTORIES_RESOURCE_LOCATION`硬编码)，文件内容为：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.home.demo.spring.boot.starter.DemoConfiguration
```

这是spirng设计的一种spi扩展机制，和java的`ServiceLoader`、dubbo的扩展机制类似可以由spring自动地读取并载入相应的配置类，从而达到减少了用户的心智负担。

## enable方式载入

注解的命名为`@EnableDemoService`，有一个int属性value，这个值乘以10作为`DemoEnableService`的结果。

通常这类enable注解都会带有一个`@Import`注解，可以import一个配置类或者import一个`ImportBeanDefinitionRegistrar`实现用于动态注册bean等等。这里采用`ImportBeanDefinitionRegistrar`是因为需要取到注解中的值。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
@Import(EnableDemoServiceRegistrar.class)
public @interface EnableDemoService {
    int value() default 1;
}

public class EnableDemoServiceRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        int value = importingClassMetadata.getAnnotations().get(EnableDemoService.class).getInt("value");
        int result = 10 * value;
        BeanDefinition bd = new RootBeanDefinition(DemoEnableService.class, () -> () -> result);
        registry.registerBeanDefinition("demoEnableService", bd);
    }
}

```

`AnnotationMetadata`是spring提供的获取配置类元数据的接口，spring的注解中有很多`@AliasFor`作为等价别名，这个接口取到的信息会对这些进行处理。获得注解的值之后，通过`BeanDefinition`动态地注册一个bean的实现。这样就达成了根据注解值的不同载入不同实现的效果。

## 最后工作

执行`mvn install`安装到本地仓库，在其他的spring项目中引入`org.home:demo-spring-boot-starter`即可享受自动装配的魅力了。
