## 服务治理

在流计算系统中，流代表了业务执行的流程。在流计算过程中，可能会需要用到一些辅助服务。
比如，针对IP的分析会使用到IP解析服务，针对地理位置的分析会使用到地理位置解析服务。
在本章中，我们主要来讨论在流计算系统中，应该如何组织业务模块和独立服务模块。

### 流服务

当一个服务模块的输入和输出都是流的时候，我们称其为一种流服务。

流服务的好处在于其可以直观方便地描述一个业务流程。
流服务使用DAG来描述执行流程。DAG的每个节点代表了一个业务单元，每个业务单元负责一定的业务逻辑。
在业务单元中，可能会需要使用到一些不属于主业务流程的功能。
比如在特征提取这个业务步骤中，需要判断IP是否在黑名单中，需要解析GEO地理位置信息等等。

把这些辅助性功能集成到流计算的各个步骤中去或许是一个好方法，特别是在你很在乎性能的时候。
毕竟将这些功能集成到流里，会减少相当多的IO操作。

但是，这样做并不优雅。考虑下，如果你的流计算任务需要用到很多辅助性功能（这种情况其实相当常见），
而且这些辅助性功能中有些内部逻辑甚至相当复杂。
那么将这些功能一股脑地放到业务流程的流实现中来，
那势必会导致整个实现逻辑整体和细节不分、执行流程杂乱无章了。

所以，在整体的流服务中，我们还是应该将这些辅助性功能剥离出去，
让它们成为单独的服务，对外提供REST或RPC的访问接口。

这样，流服务负责整体的流计算业务逻辑，
而独立的辅助性功能服务，封装了内部的辅助性，对外提供友好的使用界面。
如此一来，整个系统架构清晰明确，灵活性也强。


### 微服务
微服务是一种服务组织架构。将复杂软件系统，按业务功能划分为一个个独立的服务模块。
每个服务模块独立开发、独立部署、独立提供服务，各个独立服务模块之间天然的是一种松耦合模式。

<div align="center">
<img src="../images/img10.1.流服务和微服务关系.png" width="50%"/>
<div style="text-align: center; font-size:50%">img10.1.流服务和微服务关系</div>
</div>

#### Spring Cloud
Spring Cloud可以说是Java领域最著名、最火热的微服务的开发框架了。
Spring Cloud以Spring Boot为基础，围绕微服务提供了一些列基础功能，
如服务注册和发现、服务API网关、服务负载均衡、服务配置管理、服务容错、服务链路追踪等等。

##### 微服务基础
在Spring Boot之前，当我们用Spring技术栈构建Java Web应用时，应用的开发与服务的部署是分开的。
也就是说，用Spring框架开发好Web应用后，将其打包成war包，然后部署到诸如Tomcat这样的web容器里。
后来，在受到其它许多其它语言的轻量级Web开发框架（比如Ruby on Rails, Node.js）的强烈冲击和启发后，
Spring技术栈（或者说Java世界里）终于迎来了一个轻便的Web开发框架，这就是Spring Boot。
当使用Spring Boot开发web服务时，只需要将Spring Boot应用构建为一个jar包，然后就可以将其作为一个普通java应用启动了。
Spring Boot针对微服务提供了一系列最佳实践的设计和实现，
对微服务从开发、到构建、再到部署和运维的整个DevOps环节提供了极大的便利，有效地提到了生产效率。
Spring Cloud的其它服务组件都是以Spring Boot为基础构建而来，因此说Spring Boot是整个Spring Cloud微服务架构的基石。

##### 服务治理
服务治理包含了服务注册和服务发现两个目标。
服务治理的存在是因为在业务逻辑复杂后，由于微服务架构松耦合的特点，会导致服务实例的组织会变得零散和杂乱。
这个时候我们就需要一个服务注册中心来统一管理这些微服务实例。
服务注册用于服务提供方向服务注册中心登记自己所提供的服务，包括服务端点（endpoint）和服务内容等信息。
而服务发现则是服务的使用方，从服务注册中心获取服务提供方的信息后，选择具体的服务提供方实例，然后发起服务请求的过程。
在Spring Cloud中，负责服务治理的组件是Spring Cloud Eureka。
其中Eureka是"找到了，发现了"的意思，据说阿基米德在洗澡发现浮力原理时，喊的就是这个单词。


##### 客户端负载均衡
为方便客户端的服务调用，Spring Cloud提供了一个带负载均衡能力的HTTP客户端工具，即Spring Cloud Ribbon。
Spring Cloud Ribbon并非一个服务，而是一个工具类框架，因此只需要集成在客户端使用即可，不需要另外启动服务。
 

##### 客户端容错保护
在微服务架构中，存在着非常多的服务单元。如果某个单元出现故障，就有可能应为服务间的依赖关系而导致故障的蔓延，最终导致整个系统不可用。
比如某个微服务体系中，服务A需要调用服务B，服务B又需要调用服务C。现在服务C中的某个实例出现故障，响应非常缓慢，
当服务B的请求被轮询分配到这个故障实例后，服务B的这个实例也受到影响，服务也变得缓慢。
更有甚者，服务B的所有实例的请求都有一定概率被分配到这个有故障的C服务实例上，最终导致所有的服务B实例都出现处理缓慢的情况。
依次类推，最终服务A的所有实例也会变得响应缓慢。最终，整个系统就因为服务C一个实例的故障，导致了整个系统的不可用。
如果一开始，当服务C的实例在发生故障时，就将其剔除在服务提供者清单里，就可以避免这种故障蔓延的问题。
等到故障实例被修复后，再重新添加到服务提供者清单里来。

针对这类问题，在微服务架构体系中产生了"断路器"的概念。所谓断路器，就是通过故障监控，当服务实例发生故障时，
立刻向服务请求方返回一个错误响应，而不是让服务请求方长时间等待回应，卡死在调用的地方。

Spring Cloud Hystrix组件除了实现"断路器"的功能外，还提供服务降级、线程隔离、请求缓存、请求合并以及服务监控等一系列服务保护功能。
对于微服务架构系统的稳定性、可靠性及可用性提供有力保障。

##### 服务配置管理
配置是微服务系统非常重要的组成部分。特别是当存在大量的微服务实例时，配置会变得复杂。
而不同模块、不同环境（开发、测试和生产）、不同版本等因素的存在，更是极大地增加了配置管理的复杂度。
这个时候，一个统一管理配置的配置中心就变得十分重要。
使用Spring Cloud Config可以轻松地实现配置中心的功能。
Spring Cloud Config默认使用git来存储配置信息，天然支持了配置信息的版本控制。
不过目前Spring Cloud Config对动态配置更新的支持不是非常友好。
一方面Spring Cloud Config不支持配置变更后的自动通知。
另一方面配置刷新时的粒度太粗，容易造成系统配置和业务配置全都更新的问题。
针对更加精细的动态配置更新，需要结合Spring Cloud Bus二次开发才能实现。


##### API网关服务
当内部的服务需要对外提供服务时，会有许多问题需要考虑了。

* 反向代理
反向代理的重点在于将不同服务的请求，路由到提供各个服务的服务器上。
这样做可以将多个独立的功能模块，汇聚成一个完整的业务，并在同一域名下对外提供服务。
通过反向代理，即使在业务复杂、功能繁多、API层次丰富的情况下，也能轻松实现对外风格统一的服务接口。
对产品的迭代开发和演进，以及DevOps的实施都有极大的帮助。

* 负载均衡
负载均衡的重点在于将同一服务的请求，分发到提供该服务的不同实例上。
当单个的服务实例满足不了请求的性能要求时，可以通过横向扩展多个实例的方式来增加服务的整体性能。
特别是在线上生产系统环境下，必须能够实现服务实例的动态横向扩展。

* 认证和授权管理
认证是验证请求者身份是否真实的过程。授权则是验证请求者是否有权限访问相应资源的过程。
相比内网中的服务模块，在对外服务模块中，认证和授权是非常重要的功能，否则系统就存在信息安全的风险。

* 请求监控和日志统计
对来自外部请求进行监控和日志统计，对我们分析业务发展情况和系统性能状态都有非常大的帮助。
比如通过分析每天的全部请求数量，可以知道业务是在增长还是停滞。
通过分析请求响应的时延，可以知道内部服务是否健康，资源是否够用。
通过分析请求的失败比例，可以知道服务的稳定性及服务的质量，从而做出改进。

如果我们把上面这些具有共性的问题分散到每一个对外服务上实现，就会出现开发繁琐、实现冗余、运维复杂的情况，
可以说真的是一件吃亏不讨好的事。幸好，我们可以统一地解决这些问题，这就是API网关服务的功能。

Spring Cloud Zuul提供了API网关服务的功能，基于Spring Cloud Zuul，
定制开发专有的API网关服务是非常简单和方便的工作。


##### 消息总线
当系统中有配置变更，需要通知系统中各个模块刷新配置时，你会怎么做？
Spring Cloud Bus提供了消息总线的功能。
在使用消息总线的时候，先将各个微服务实例连接到消息总线上来，
然后当有诸如配置变更的消息需要在系统中通知时，只需要将消息通过消息总线广播出去即可。

消息总线为Spring Cloud中的各个服务模块提供了一个统一的消息通知和控制总线的功能，
可以方便地实现系统配置在线动态更新等功能。

##### 服务链路追踪
当业务变得复杂时，微服务间的调用关系势必也会变得复杂。
这时候当一个客户请求过来时，如果一切正常，客户会即时得到响应。这种情况下一切都好。
但是，如果系统某个环节发生故障，客户请求得不到正常响应，那我们该如何快速定位故障发生的地方？
这时候，对于请求调用的追踪就变得非常重要。
Spring Cloud Sleuth提供了全连路调用追踪的功能，通过它我们可以轻松追踪服务的调用链。
当发生故障时，通过调用链的执行情况，就能快速定位故障发生的服务模块所在。




