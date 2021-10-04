---
title: Project Reactor 响应式编程分享
date: 2021-08-27 21:54:14
updated: 2021-08-27 21:54:14
categories: Java
tags: 
  - reactive programming
  - java
---

# Reactive Programming

在计算机领域中，响应式编程是一种面向**数据流和变化**传播的编程范式。这意味着可以在编程语言中很方便地表达静态或动态的数据流，而相关的计算模型会自动将变化的值通过数据流进行传播。<!--more-->

举个例子来说，对于表达式 a = b + c，a 的计算结果由 b 和 c 的值确定下来后，之后 b 或 c 的值改变不会影响到 a 的值。如果是套用响应式的概念，对于 a = b + c，a 的值会随着数据流的变化实时发生变化，也就是说 b 和 c 的值改变，会影响表达式的结果 a。

之后，从微软在 .NET 上实现 Rreactive Extensions（Rx）库，随之 JVM 也出现了 RxJava 实现了响应式的编程范式那一套东西。

如今，Java 已经将响应式相关的内容纳入规范（**Reactive Streams**），定义了响应式库的接口的交互规则。

>The specification developed with the intent of future inclusion in the official Java standard library, if proven successful and adopted by enough libraries and vendors.
>Reactive Streams were proposed to become part of [Java](https://en.wikipedia.org/wiki/Java_(software_platform)) 9 by [Doug Lea](https://en.wikipedia.org/wiki/Doug_Lea), leader of [JSR 166](https://en.wikipedia.org/wiki/JSR_166)[[7]](https://en.wikipedia.org/wiki/Reactive_Streams#cite_note-7) as a new Flow class[[8]](https://en.wikipedia.org/wiki/Reactive_Streams#cite_note-8) that would include the interfaces currently provided by Reactive Streams.[[4]](https://en.wikipedia.org/wiki/Reactive_Streams#cite_note-infoq-4)[[9]](https://en.wikipedia.org/wiki/Reactive_Streams#cite_note-jep266-9) After a successful 1.0 release of Reactive Streams and growing adoption, the proposal was accepted and Reactive Streams was included in [JDK9](https://en.wikipedia.org/wiki/Java_(software_platform)) via the [JEP](https://en.wikipedia.org/wiki/JDK_Enhancement_Proposal)-266.[[9]](https://en.wikipedia.org/wiki/Reactive_Streams#cite_note-jep266-9)
>
>>-- [https://en.wikipedia.org/wiki/Reactive_Streams](https://en.wikipedia.org/wiki/Reactive_Streams)

## 响应式编程的优势

比如常见的 Web 服务，在传统的阻塞编程模型里，每个请求都需要一个线程去处理，请求过程可能需要调用数据库 I/O，远程调用服务等等，都需要线程阻塞等待结果返回。如果是非阻塞的编程模型，请求调用某些服务后，可以保存相关的信息后直接返回，线程继续处理其他新的请求。线程无需一直阻塞在那些耗时的操作上，而是等那些耗时的操作有结果后主动通知线程就绪。这样做的好处是线程不必频繁阻塞，可以利用少量的线程处理大量的请求。

## 响应式编程为何不够普及？

* 基础服务不够普及。前面提到的响应式编程的优势在于调用某些 API 后可以直接返回，但是就目前来说，响应式的 API 普及度远远不够。比如 JDBC，没有提供非阻塞 API ，线程调用后需要等待响应，响应式的优势难以体现（目前也出现了一些响应式的数据库客户端 API，比如 R2DBC，但都不普及）。
* 应用学习的成本问题。

# Project Reactor 

Project Reactor 基于 Reactive Stream 规范，提供了可以构造非阻塞（异步）应用的响应库。

## 非阻塞（异步）

Java 对于异步编程也提供了支持，我们可以为方法传入一个 **Callable**参数或者立即返回一个 **Future<T>** ，当前线程则不用一直等待直到有结果返回，而是当有结果可以返回的时候再获取。相比于同步的阻塞调用，线程不用阻塞等待结果返回，可以先服务其他请求，在结果返回后再去获取。

## Reactor 与 Callback、Future<T>

假设我们要异步实现这样一个服务：通过 userService 获取用户前五个点赞列表（如果有的话通过 favoriteService 获取详细的点赞信息），如果没有点赞列表则调用 suggestionService 获取推荐列表。

### 回调写法

![image(11)](reactor-project/image(11).png)

可以看到每个服务调用都需要 onSuccess，onError 处理来处理服务调用的问题，调用正确时，调用其他服务时又需要 Callback 代码。

### 使用 Reactor 

![image(12)](reactor-project/image(12).png)

1. 根据 userId 获取 Favorites 列表
2. 从 Favorites 列表获取对应的 Detail 信息
3. 如果 Favorites 是 empty 的话，改成获取推荐列表
4. 只需要 5 个信息
5. 通过 UI thread 处理数据
6. 最后，通过 subscribe 触发数据流，然后通过 show 方法展示数据。

Reactor 提供了需许多易用的操作，可以应对复杂的业务。

## Reactor 一些核心的概念

### Publisher-Subscriber

Publisher 负责生产数据；但是默认情况下 Publisher 不会做任何事情，直到有 Subscriber 订阅了它，Publisher 被订阅后，就会主动向 Subscriber **push** 数据。

### Nothing Happens Until You subscribe()

定义了 Publisher，以及一系列对数据流的操作后，默认地数据还没有被进行任何操作。只有调用了 subscribe 后，才会触发数据流开始操作。

### Flux - [0...N]，Mono - [0...1]

Reactor 中，Flux 和 Mono 就是标准的 Publiser<T> 。Flux 的含义是可能有 0 到 N 个数据的流。Mono 则表示至多只有一个。

### Backpressure

当 Publisher 生产的数据 Subscriber 来不及消费的时候，数据过多得积压可能发生错误，比如内存占用变大等等。Reactor 可以通过订阅者的 onRequest,onCancle 来指定需要处理的数据量避免 Backpressure 问题。

## 运用 Reactor

### 核心概念的简单示例

![image(13)](reactor-project/image(13).png)

* Flux.range 生成数据流

* ### R2DBC、WebFlux

![image(14)](reactor-project/image(14).png)

![image(15)](reactor-project/image(15).png)

# debug

# 相关链接

* [https://projectreactor.io/docs/core/3.2.2.RELEASE/reference/index.html#about-doc](https://projectreactor.io/docs/core/3.2.2.RELEASE/reference/index.html#about-doc)
* [https://projectreactor.io/](https://projectreactor.io/)
* [https://wiki.jikexueyuan.com/project/android-weekly/issue-145/introduction-to-RP.html](https://wiki.jikexueyuan.com/project/android-weekly/issue-145/introduction-to-RP.html)
* [https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#spring-webflux](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#spring-webflux)
* https://r2dbc.io/