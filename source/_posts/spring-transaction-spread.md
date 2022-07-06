---
title: Spring 事务传播机制
date: 2021-08-09 18:54:14
updated: 2021-08-09 18:54:14
categories: Spring Framework
tags: 
  - spring
  - 事务
---

# 事务传播机制

首先我们看 [TransactionDefinition](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/TransactionDefinition.html) 接口的注释。

```plain
/**Interface that defines Spring-compliant transaction properties.
* Based on the propagation behavior definitions analogous to EJB CMT attributes.

大概说这个接口定义了 Spring 兼容的事务属性。基于类似于EJB CMT属性的传播行为定义。
*/
```

<!--more-->那什么是 EJB CMT 呢？参考[维基百科](https://en.wikipedia.org/wiki/Enterprise_JavaBeans)的解释，EJB 容器必须支持 Container-managed transaction（CMT）来管理 ACID 特性的事务和 Bean managed transaction（BMT）。

对应到 Spring 框架就是我们常说的 Spring 事务的传播机制以及隔离级别。

这两个容器的目的就是不需要显式复杂的配置，事务管理可以通过简单的注解进行。

Spring 通过 Spring AOP 来进行事务的管理，比如我们常用的 @Transactional 注解。

那当我们在方法中互相调用开启了事务功能的方法时，方法之间定义的事务是怎么起作用的呢？

 

比如 A 方法开启了事务，A 方法中调用了 B 方法，B 方法也开启了事务，那么这时候 A 和 B 之间的事务是独立进行的还是一起作用的？

Spring 在这方面为我们提供了管理的方法，也就是今天我们讨论的 Spring 的事务传播机制。

相关的定义在 Spring 的 TransactionDefinition 接口文件中。Spring 定义了 6 中传播行为。

## **REQUIRED**

支持当前的事务；如果当前没有，那么创建一个新的。这也是 Spring 默认的传播行为。

比如，方法 A 调用了方法 B，A B都开启 REQUIRED 级别的事务。那么，B 方法会发现有了当前事务 A，那么 B 就会加入到 A 的事务中。

如果 A 没有开启事务，B 仍旧开启自己的事务。

## **SUPPORTS**

支持当前事务；如果没有当前事务以非事务的方式运行。

也可以从字面上理解，supports 看起来就没有 required 那么强势。如果没有当前事务就非事务运行好了。

## **MANDATORY（强制性）**

支持当前事务；如果没有会抛出异常；

## **REQUIRES_NEW**

始终创建一个新的事务；如果有当前事务，会把当前事务挂起；

## **NOT_SUPPORTED**

不支持当前事务；总是以非事务方式执行。

## NEVER

不支持当前事务；如果有当前事务会抛出异常。

## NESTED（嵌套）

如果当前事务存在，就以嵌套的方式运行。行为和 REQUIRED 很像。

## 自调用的问题

Stackover Flow 上有一个相关的[问答](https://stackoverflow.com/questions/37217075/spring-nested-transactions)，大意是在方法中调用了同一个 service 的方法，比如方法 A （**REQUIRED**）调用了方法 B，即使 B 声明了 **REQUIRS_NEW**的传播行为（始终创建一个新事务执行），但在 A 方法抛出异常的时候，B 方法还是会回滚。

这个问题需要从 Spring AOP 中动态代理的角度来分析。

Spring AOP 使用 JDK / Cglib 进行动态代理，它会通过代理，织入增强代码。比如调用带 @Transactional 注解的方法，会通过调用**代理对象（增强后）**的该方法进行。

但当进行自调用的时候，方法内部调用的方法还是使用原对象进行。也就是说，Spring AOP 的代理对象只会在不同的 bean 之间相互调用的时候使用。

一般遇到自调用的问题时，一个解决办法是新建一个帮助类，然后去调用它。

大家也可以看看 [Spring doc](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/data-access.html#transaction-declarative-annotations) 中的 1.4.6 节，有关自调用方面的说明。

Spring 官方还建议把 @Transactional 注解使用在具体的类上，不建议用在 接口 上。

原因是，当你使用了基于 类 的代理方式时，使用在 接口 上的注解就失效了。

# 事务隔离级别

Spring 的事务隔离级别和数据库的没有什么区别。

* 读未提交
* 读提交
* 可重复读
* 串行化

Spring 有一个 ISOLATION_DEFAULT 级别，作用是采用和 数据库 一致的隔离级别。

# 异常回滚

@Transactional 还有一个 rollbackFor() 属性。

接口上的注释大意是，这个属性定义了发生什么类型的异常会触发事务的回滚。

By default，事务会因为发生非受检异常回滚，受检异常发生了不会回滚。

你可以这么用，来定义自己的异常回滚策略。

```plain
@Transactional(propagation = Propagation.REQUIRED,rollbackFor = Exception.class)
```

最后做一个总结，Spring 利用 Spring AOP 提供了方便的事务管理功能。在此基础上，通过配置事务的传播机制，进一步解决实际上使用事务时可能存在的问题。

因为 Spring AOP 动态代理的特性，在自调用的时候会出现事务失效的问题。除了刚才提到的使用帮助类来调用，还有一种方法是使用 AspectJ ，这是另一种 AOP 方案，其原理是通过在编译期在字节码层面织入代码实现增强功能。

最后就是事务的隔离级别。隔离级别这方面需要了解比如脏读，幻读等问题。

笔者也是一位初学者，如若文章有错误和疑惑的地方，也希望您可以指出，我们共同讨论和进步.

‍(=・ω・=)

