---
title: Hystrix架构解读
date: 2019-07-07 16:43:55
updated: 2019-07-07 16:43:55
desc: Hystrix架构解读
keywords: Hystrix, 架构, RxJava, 设计模式
categories: 技术
tags: [Hystrix, 架构, RxJava, 设计模式]
---

![hystrix-flow-chart](/images/hystrix-flow-chart.jpg)

Hystrix源自2011年Netflix的弹性工程项目，名字取自「豪猪」，寓意其会让系统拥有弹性、防御和容错的能力。经过了多年的发展，目前项目已进入停止更新的维护模式，官方给出的解释是他们的研究重点已从预先配置的方式转到实时自适应的方向，推荐新项目使用活跃的resilience4j，但Hystrix提出的概念和设计思想任然值得我们研究和借鉴。

<!--more-->

Hystrix为分布式系统提供了限流、熔断、降级的基础设施，避免了时延等网络问题或系统故障导致的级联故障甚至雪崩效应，使得系统可以快速失败、优雅降级和快速恢复。理解Hystrix的原理，首先就需要研究经过Hystrix封装的请求的执行过程。被调用方在Hystrix中被称为依赖项，利用HystrixCommand或HystrixObservableCommand封装对依赖项的请求，使用HystrixCommand只能获取一个结果，而使用HystrixObservableCommand返回Observable对象可以获取多个结果。我们知道命令模式可以将请求封装成对象，使发出请求的对象和执行请求的对象解耦，并支持排队、撤销和重做等操作，而Hystrix就利用命令模式将应用和依赖项进行了隔离。执行命令可以使用同步或异步的方式，获取异步结果可以使用Future或Observable的方式，但本质上最终都是通过Observable的方式实现的。如果允许使用请求结果缓存并且命中缓存，则直接从缓存中返回Observable形式的结果。否则检查「电路」是否跳闸，如果打开则进入Fallback回退逻辑。如果「电路」是闭合状态，继续检查与依赖相关的线程池和队列是否满了，或者信号量资源是否耗尽，如果是则拒绝执行命令并进入Fallback回退逻辑。否则继续执行依赖项的逻辑，如果执行命令超时或异常，则进入Fallback回退逻辑，否则返回成功的响应。在整个执行过程中，Hystrix向「断路器」报告成功、失败、拒绝和超时的Metrics信息，「断路器」维护了一组滚动计数器来计算统计数据，并利用这些统计信息控制「电路」的开闭。而对于依赖项出错有多种处理方式，如：Fail Fast、Fail Silent和Fallback等。

![hystrix-isolation](/images/hystrix-isolation.jpg)

Hystrix使用了多种隔离技术（舱壁隔离、泳道和断路器模式）来限制任何一个依赖项的影响。所谓舱壁隔离源自造船技术，货船为了防止漏水和火灾的扩散会将货仓分隔为多个，当发生灾害时将所在货仓进行隔离就可以降低整艘船的风险。Hystrix对不同的依赖项利用不同的线程池进行隔离，相同依赖项的请求使用不同的线程进行隔离，分别对应舱壁隔离和泳道。每个CommandKey代表一个依赖抽象，相同的依赖要使用相同的CommandKey名称，依赖隔离的根本就是对相同CommandKey的依赖做隔离。默认情况下，HystrixCommand使用线程池隔离，可以通过设置线程池的大小来限制请求数量，也可以通过修改配置项调整请求队列的大小，HystrixObservableCommand使用信号量隔离。信号量不需要线程切换，开销小，但不支持异步和超时，只有在非网络请求或者大量访问线程开销很大的情况下才考虑使用信号量隔离，否则默认的线程隔离已满足大部分使用场景的需求。

![hystrix-circuit-breaker](/images/hystrix-circuit-breaker.jpg)

断路器模式在Hystrix的架构中也起到了至关重要的作用，其设计了「滚桶」的数据结构，滑动窗口被分成N个桶，桶中保存成功、失败、拒绝、超时和百分位信息，而且总是从最新的N个桶中读取Metrics信息。Hystrix早期是利用`HystrixRollingNumber`和`HystrixRollingPercentile`来存放聚合数据，`1.5.x`版本以后变为利用响应式编程的一种实现RxJava和Stream来存放和处理聚合数据了。当统计数据达到错误率阈值时，「断路器」将短路某个依赖服务的后续请求，直到恢复期结束，若恢复期结束根据统计数据「断路器」判定「电路」仍然未恢复健康，「断路器」会再次关闭「电路」。

Hystrix在设计中使用继承抽象基类而不是接口，官方给出的原因主要是为了方便维护类库，如果使用接口每新增一个功能就需要新加一个接口，而使用继承抽象基类，用户可以重写方法但不破坏默认实现，这样就可以在不破坏已有实现的情况下添加新功能。然而我们知道面向对象设计有一条原则「基于接口而不是继承」，Hystrix的选择究竟是否正确，或者说是否是因为公共类库的设计和普通业务系统的设计的区别导致了不同的选择，这值得我们思考，当然Java8后的default方法给了我们另一个选择。

对Hystrix的深入理解还需要进一步阅读其源码，框架中涉及命令模式、观察者模式（反应器模式）、RxJava、范型、函数式接口、Lambda表达式、Stream流式编程等大量的使用。不为了读源码而读源码，我认为读源码原因包括：明确技术文档中原理的实现细节甚至发现文档错误，理解细节方便对框架进行扩展，耳濡目染修炼架构内功，这就是我从Hystrix研究中得来的些许感受。