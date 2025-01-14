# 从类库到服务

:::tip 通过服务来实现组件

Microservice architectures will use libraries, but their primary way of componentizing their own software is by breaking down into services.

微服务架构也会使用到类库，但构成软件系统组件的主要方式是将其拆分为一个个服务。

:::right

—— [Martin Fowler](https://martinfowler.com/) / [James Lewis](https://twitter.com/boicy), [Microservices](https://martinfowler.com/articles/microservices.html), 2014

:::

微服务架构其中一个重要设计原则是“通过服务来实现独立自治的组件”（Componentization via Services），强调应采用“服务”（Service）而不再是“类库”（Library）来构建组件化的程序，这两者的差别在于类库是在编译期静态链接到程序中的，通过调用本地方法来使用其中的功能，而服务是进程外组件，通过调用远程方法来使用其中的功能。

采用服务来构建程序，获得的收益是软件系统“整体”与“部分”在物理层面的真正隔离，这对构筑可靠的大型软件系统来说无比珍贵，但另一面，其付出的代价也同样无可忽视，微服务架构在复杂性与执行性能方面做出了极大的让步。一套由多个微服务相互调用才能正常运作的分布式系统中，每个节点都互相扮演着服务的生产者与消费者的多重角色，形成了一套复杂的网状调用关系，此时，至少有（但不限于）以下三个问题是必须考虑并得到妥善解决的：

- **对消费者来说，外部的服务由谁提供？具体在什么网络位置？**
- **对生产者来说，内部哪些服务需要暴露？哪些应当隐藏？应当以何种形式暴露服务？以什么规则在集群中分配请求？**
- **对调用过程来说，如何保证每个远程服务都接收到相对平均的流量，获得尽可能高的服务质量与可靠性？**

这三个问题的解决方案，**在微服务架构中通常被称为“服务发现”、“服务的网关路由”和“服务的负载均衡”**。
