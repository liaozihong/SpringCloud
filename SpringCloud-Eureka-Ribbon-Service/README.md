## SpringCloud核心组件Ribbon

Ribbon 是一个基于 HTTP 和 TCP 客户端的负载均衡器
它可以在客户端配置 ribbonServerList（服务端列表），然后轮询请求以实现均衡负载。

Ribbon是一个负载均衡客户端，可以很好的控制htt和tcp的一些行为。Feign默认集成了ribbon。

Ribbon的负载均衡默认使用的最经典的Round Robin轮询算法。这是啥？

简单来说，就是如果订单服务对库存服务发起10次请求，那就先让你请求第1台机器、然后是第2台机器、第3台机器、第4台机器、第5台机器，接着再来—个循环，第1台机器、第2台机器。。。以此类推。

 此外，Ribbon是和Feign以及Eureka紧密协作，完成工作的，具体如下：

* 首先Ribbon会从 Eureka Client里获取到对应的服务注册表，也就知道了所有的服务都部署在了哪些机器上，在监听哪些端口号。
* 然后Ribbon就可以使用默认的Round Robin算法，从中选择一台机器
* Feign就会针对这台机器，构造并发起请求。

流程图如下：
![](https://ws1.sinaimg.cn/large/006mOQRagy1fxnwvyy24sj30kr08640v.jpg)

## Spring Cloud核心组件：Hystrix
Hystrix是隔离、熔断以及降级的一个框架。

在微服务架构里，一个系统会有很多的服务。以本文的业务场景为例：订单服务在一个业务流程里需要调用三个服务。现在假设订单服务自己最多只有100个线程可以处理请求，然后呢，积分服务不幸的挂了，每次订单服务调用积分服务的时候，都会卡住几秒钟，然后抛出—个超时异常。

咱们一起来分析一下，这样会导致什么问题？

* 如果系统处于高并发的场景下，大量请求涌过来的时候，订单服务的100个线程都会卡在请求积分服务这块。导致订单服务没有一个线程可以处理请求
* 然后就会导致别人请求订单服务的时候，发现订单服务也挂了，不响应任何请求了

像这种情况就是传说中的微服务架构服务雪崩问题。
![](https://ws1.sinaimg.cn/large/006mOQRagy1fxnx71oybrj30js0d1t9z.jpg)
如上图，这么多服务互相调用，要是不做任何保护的话，某一个服务挂了，就会引起连锁反应，导致别的服务也挂。比如积分服务挂了，会导致订单服务的线程全部卡在请求积分服务这里，没有一个线程可以工作，瞬间导致订单服务也挂了，别人请求订单服务全部会卡住，无法响应。

但是我们思考一下，就算积分服务挂了，订单服务也可以不用挂啊！为什么？

* 我们结合业务来看：支付订单的时候，只要把库存扣减了，然后通知仓库发货就OK了
* 如果积分服务挂了，大不了等他恢复之后，慢慢人肉手工恢复数据！为啥一定要因为一个积分服务挂了，就直接导致订单服务也挂了呢？不可以接受！

** 解决 **

这时就轮到Hystrix闪亮登场了。Hystrix是隔离、熔断以及降级的一个框架。啥意思呢？说白了，Hystrix会搞很多个小小的线程池，比如订单服务请求库存服务是一个线程池，请求仓储服务是一个线程池，请求积分服务是一个线程池。每个线程池里的线程就仅仅用于请求那个服务。

打个比方：现在很不幸，积分服务挂了，会咋样？

当然会导致订单服务里的那个用来调用积分服务的线程都卡死不能工作了啊！但是由于订单服务调用库存服务、仓储服务的这两个线程池都是正常工作的，所以这两个服务不会受到任何影响。

这个时候如果别人请求订单服务，订单服务还是可以正常调用库存服务扣减库存，调用仓储服务通知发货。只不过调用积分服务的时候，每次都会报错。但是如果积分服务都挂了，每次调用都要去卡住几秒钟干啥呢？有意义吗？当然没有！所以我们直接对积分服务熔断不就得了，比如在5分钟内请求积分服务直接就返回了，不要去走网络请求卡住几秒钟，这个过程，就是所谓的熔断！

那人家又说，兄弟，积分服务挂了你就熔断，好歹你干点儿什么啊！别啥都不干就直接返回啊？没问题，咱们就来个降级：每次调用积分服务，你就在数据库里记录一条消息，说给某某用户增加了多少积分，因为积分服务挂了，导致没增加成功！这样等积分服务恢复了，你可以根据这些记录手工加一下积分。这个过程，就是所谓的降级。

![](https://ws1.sinaimg.cn/large/006mOQRagy1fxnx1k21wzj30kf0eaadz.jpg)

参考链接：
https://blog.csdn.net/forezp/article/details/83999882