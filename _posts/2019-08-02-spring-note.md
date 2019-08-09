---
layout: post
title: Spring笔记
date: 2019-08-02
author: Nicholas Huang
header-img:
categories: Spring
tags:
    - Spring
---
# 《Spring实战》
## 1.1简化java开发
4种策略：
    
    1.基于POJO的轻量级和最小侵入性编程
    2.通过依赖注入和面向接口实现松耦合
    3.基于切面和惯例进行声明式编程
    4.通过切面和模板减少样版式代码
    
 创建应用组件之间协作的行为通常称为装配（writing)
 Spring装配方式：
    
    1.XML——ClassPathXmlApplicationContext
    2.基于Java的配置
    
 依赖注入的方式：
    
    1.构造器注入——构造的时候传参
    
 Spring通过应用上下文（Application Context）装载bean的定义并把他们组装起来
 
 DI能够让相互协作的软件组件保持松耦合，面向切面编程（aspect-oriented programming, AOP）运行你把遍布应用各处的功能分离出来形成可重用的组件，能提高内聚，使组件关注自身的业务
 
 Spring容器（container）负责创建对象，装配它们，配置它们并管理它们的整个生命周期，从生存到死亡（可能就是new到finalize）
 
 两种容器：
    
    1.bean工厂，由org.springframework.beans.factory接口定义，是最简单的容器，提供基本的DI支持
    2.应用上下文，由org.springframework.context.Application接口定义，基于BeanFactory构建，并提供应用框架级别的服务。有下列几种：
        2.1 AnnotationConfigApplicationContext
        2.2 AnnotationConfigWebApplicationContext
        2.3 ClassPathXmlApplicationContext
        2.4 FileSystemXmlApplicationContext
        2.5 XmlWebApplicationContext
        
Spring中bean的生命周期：
    
    1.实例化，Spring对bean进行实例化
    2.填充属性，Spring将值和bean的引用注入到bean对应的属性中
    3.调用BeanNameAware的setBeanName()方法
    4.调用BeanFactoryAware的setBeanFactory()方法
    5.调用ApplicationContextAware的setApplicationContext()方法
    6.调用BeanPostProcessor的预初始化方法
    7.调用InitializingBean的afterPropertiesSet()方法
    8.调用自定义的初始化方法
    9.调用BeanPostProcessor的初始化后方法
    10.bean可以使用
    11.调用DisposableBean的destroy()方法
    12.调用自定义的销毁方法
    
@Component告诉Spring使用自动装配，而@Bean只能用在@Configuration注解的类中。两者都可以通过@Autowired装配，那为什么有了@Component还需要@Bean，比如对第三方库的组件的自动装配，只能通过@Bean或者XML配置

```
    @Component
    public class Student {
        
        private String name = "nicholas";
        
        public String getName() {
            return name;
        }
        
        public void setName(String name) {
            this.name = name;
        }
    }
    
    @Configuration
    public class WebSocketConfig {
        @Bean
        public Student student() {
            return new Student();
        }
    }
```    
## 第二章 装配Bean
[@Autowired,@Inject和@Resource的区别](http://www.dengshenyu.com/spring/2016/10/09/spring-inject.html)
[Wiring in Spring:@Autowired,@Resource and @inject](https://www.baeldung.com/spring-annotations-resource-inject-autowire)
总结如下:
@Resource：
    
    1. Match by Name
    2. Match by Type
    3. Match by Qualifier

@Autowired和@Inject：
    
    1. Match by Type
    2. Match by Qualifier
    3. Match by Name
    
## 第三章 高级装配
通过profile bean来实现不同环境的配置

Spring并不是在构建的时候做出决策，而是等到运行时再来确定

@Profile指定某个bean属于哪一个profile，可以在类上注解，也可以在方法上注解

没有指定profile的bean始终都会创建

[关于jndi](https://blog.csdn.net/zhaosg198312/article/details/3979435)，一句话总结就是jndi就是java为了获取外部资源而设立的一类接口

同一个xml配置中，可以配置不同的profile
    
profile的值，spring.profiles.active和spring.profiles.default

使用@ActiveProfiles来指定集成测试时激活哪个profile

@Conditional用到带有@Bean注解的方法上，如果条件为true，则生成该bean。设置给@Conditional的类必须要实现Condition接口，该接口有一个matches()函数

从Spring4开始@Profile进行了重构，使用了@Conditional

```
    /**
    *注意通过下面的代码，可以看出很多东西，比如@Profile为啥支持类和方法，以及为啥能在同一个
    *@Profile中声明多个profile
    */
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    @Documented
    @Conditional(ProfileCondition.class)
    public @interface Profile {
        String[] value();
    }
```   

解决自动装配歧义性的方法：
    
    1.将可选bean中的某一个设为首选（primary）的bean，@Primary可以和@Compoent、@Bean结合使用
    2.使用限定符（qualifier），@Qualifier可以和@Autowired、@Inject、@Resource、@Component和@Bean结合使用
    
使用自定义限定符注解：

```
    @Target(ElementType.CONSTRUCTOR, ElementType.FIELD, ElementType.METHOD, ElementType.TYPE)
    @Rentention(RetentionPolicy.RUNTIME)
    @Qualifier
    public @interface Cold {}
```    

默认情况下，Spring应用上下文中所有bean都是作为以单例（singleton）的形式创建的，不管一个bean被注入到其他bean多少次，每次所注入的都是同一个实例

Spring定义了多种作用域，选择其他作用域，使用@Scope，可以与@Component和@Bean结合使用
    
    1.单例（singleton），在整个 应用中，只创建bean的一个实例
    2.原型（prototype），每次注入或者通过Spring应用上下文获取的时候，都会创建一个新的实例
    3.会话（session），在Web应用中，为每个会话创建一个bean实例
    4.请求（request），在Web应用中，为每个请求创建一个bean实例
    
@Scope有一个proxyMode的属性，解决了将会话或请求作用域的bean注入到单例bean所遇到的问题——Spring会注入一个会话或请求作用域的bean的代理到单例bean中。如果带有@Scope的是接口，那么`proxyMode = ScopedProxyMode.INTERFACES`，但如果是类，那么只能用CGLib来生成基于类的代理，那么`proxyMode = ScopedProxyMode.TARGET_CLASS`

通过@PropertySource和Environment变量，可以获取外部的值。如果依赖于组件扫描和自动装配，那么使用@Value`@Value("${disc.title}") String title`

SpEL简单例子`@Value("#{systemProperties['disc.title']}")`
SpEL表示字面值：整数，浮点数，String值以及Boolean值
SpEL引用bean、属性和方法，类型安全运算符？，`#{artistSelector.selectArtist()?.toUpperCase()}`
SpEL是用类型，`#{T(java.lang.Math).random()}`
SpEL计算正则表达式，`#{admin.email matches '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.com'}`
SpEL使用集合，`#{jukebox.songs[T(java.lang.Math).random() * jukebox.songs.size()].title}`
       


