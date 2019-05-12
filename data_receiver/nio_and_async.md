## NIO和异步
晴朗的天空还飘着剩下一朵"乌云"。
通过监控系统我们注意到，每个事件上报请求的处理时延较高，数据接收服务器的吞吐能力偏低，但是操作系统CPU和IO的使用率却并不高。
进一步使用JVisualVM监测工具发现，主要是`doExtractCleanTransform()`函数处理时延偏高。
现在我们就来分析下`doExtractCleanTransform()`可能耗时的原因。

### CPU密集型任务
CPU密集型任务是指处理过程主要依靠CPU（这里不考虑协处理器，比如GPU、FPGA等）运算来完成的任务。
CPU密集型任务优化的方向主要是算法本身，除此以外优化的空间并不大。
因为如果CPU已经跑满，对于相同的计算量，是多线程还是单线程、是同步还是异步，并不会减少计算时间。
相反，线程过多时操作系统花更多时间在线程调度上，有效计算时间相应减少，反倒会降低性能、增加时延。
算法优化包括改进算法、合理使用内存资源、使用协处理器、针对JVM优化编程、针对CPU优化编程、甚至极端情况下使用汇编等等。
但这些内容与本书的主题实时流计算系统没有直接关系，在此不做详细讨论。

### IO密集型任务
IO密集型任务是指在处理过程中，有很多IO操作的任务。IO操作包括磁盘IO和网络IO。
磁盘IO发生在写日志或做数据持久化。
网络IO则发生在往消息中间件发送数据，或者是在调用外部服务，如数据库服务、其它微服务模块等等。
相比CPU密集型任务而言，IO密集型任务是大多数Java开发者面临更多、更现实的问题。
因为大部分业务系统和基础框架，都涉及数据持久化、数据库访问或对其它服务模块的调用。

IO密集型任务，如果计算逻辑简单，对CPU资源要求不多，最方便有效的方法是增加线程数。
让大量的线程同时触发更多的IO请求，可以将IO资源充分利用。
为什么这时可以使用大量线程？因为操作系统调度线程占用的是CPU资源，
如果计算逻辑本身比较简单，对CPU资源要求不高，那么这部分资源不妨就留给操作系统做线程调度。

### IO和CPU都比较密集
当IO和CPU都比较密集的时候，问题就复杂很多了。

我们在前边提到使用大量线程可以提高IO利用率。这是因为当进程执行到涉及IO操作或sleep之类的函数时，会引发系统调用。
进程执行系统调用，会从用户态进入内核态，之后在其准备从内核态返回用户态时，有一次操作系统进行进程调度的机会。
对于正在执行IO操作的进程，操作系统很有可能将其调度出去。
这是因为触发IO请求的进程，通常需要等待IO操作完成，操作系统就让其晾在一旁等等，先调度其它进程执行。
当IO请求数据准备好的时候，进程再次获得被调度执行的机会，然后继续之前的执行流程。

​从上面进程执行IO系统调用的过程可以看出，当进程执行IO操作时，操作系统本身并不会阻塞在等待IO返回，
而是通过调度让其它进程使用CPU。因此当大量进程进行IO请求时，这些IO请求都会被触发，使IO任务被安排得满满的，尽可能地充分利用IO资源。
操作系统采取这种调度策略的主要考虑是更充分地利用CPU资源，同时如果IO请求较多，IO资源也会被充分利用，这样做非常合理。

只不过如果进程过多，操作系统将频繁地进行进程调度和上下文切换，耗费过多的CPU时间，
使其用于执行有效程序逻辑的时间变少，造成另一种形式的CPU资源浪费。


### 纤程
​那有没有一种类似于线程，在碰到IO调用时不会阻塞，能够让出CPU执行其它计算，
等IO数据准备好了再继续执行，同时还不占用过多CPU在进程调度和上下文切换的办法呢？
有！这就是纤程，也叫协程。纤程是一种用户态的线程，其调度逻辑在用户态实现，从而避免了过多地进出内核态进行进程调度和上下文切换。
事实上，纤程才是最理想的线程！
那纤程是怎样实现的呢？就像线程一样，关键是要在执行过程中，能够在恰当时刻和恰当地方被中断，然后调度出去，让给其它线程使用CPU。
先来考虑IO，前面说到进程执行IO操作时，一定会不可避免地提供给操作系统一次调度它的机会。
但问题的关键不是避免IO操作，而是避免过多的线程调度和上下文切换。
我们可以将IO操作委托给少量固定线程，再使用另外少量线程负责IO状态检查和纤程调度，
再用适量线程执行纤程，这样就可以大量创建纤程，而且只需要少量线程即可。

回想下之前Tomcat NIO连接器的实现机制，是不是这种纤程和Tomcat NIO连接器的工作机制有异曲同工之妙。
事实上正是如此，理论上讲，纤程才是将异步执行过程封装得最好的方案。
因为它封装了所有异步复杂性，在提供同步API便利性的同时，还拥有非阻塞IO的性能！

更进一步地讲，最最理想的纤程应该完全像线程那样，连CPU的执行都不阻塞。
也就是说纤程在执行非IO操作的时候，也能够随时被调度出去，让CPU执行其它纤程。
这样做是可能的，但需要CPU中断支持。线程的调度在内核态完成，可以直接得到CPU中断支持。
但位于用户态的纤程，要得到中断支持相对会更繁琐，需要进出内核态。
再考虑CPU中断时的上下文切换过程，纤程实现CPU中断触发调度的复杂性和性能代价将非常大。
所以通常而言，用户态的纤程只会做到IO执行非阻塞，CPU执行依旧阻塞。
当然，有些纤程的实现方案，如Python里的绿色线程，提供了主动让出CPU给其它程序片段执行的方法。
这种在程序逻辑中主动让出CPU调度其它程序片段执行的方案，虽然只是由开发人员在编写代码时自行控制的，但也算是对实现CPU非阻塞执行的尝试了。

既然纤程有这么多好处，提供同步API的同时拥有非阻塞IO的性能，可以大量创建而不增加操作系统调度开销，
这样对于程序开发，不管多么复杂的逻辑只需要放在纤程里，然后起个几十万甚至上百万个纤程，
不就可以轻松做到高并发、高性能了？一切都很美好不是？
可是为什么到现在为止，我们大多数Java开发人员还没有用上纤程呢？
或者说，为什么至少在Java的世界里，时至今日纤程还没有大行其道呢？
这是个比较尴尬的现状。从前面的分析中我们知道，实现纤程的关键在于进程执行IO操作时拦截住CPU的执行流程。
那怎样拦截呢？这就用到我们常说的AOP技术了。在纤程上将所有与IO操作相关的函数进行AOP拦截，给调度器提供调度纤程的机会。
在JVM平台上，可以在三个层面进行拦截：
1. 修改JVM源码，在JVM层面拦截所有IO操作API；
2. 修改JDK源码，在JDK层面拦截所有IO操作API；
3. 采用动态字节码技术，动态拦截所有IO操作API。
其中第三种方案，已有开源实现Quasar，读者如果感兴趣可以自行研究，在此不再展开。
但是笔者认为，Quasar虽然确实实现了IO拦截，实现了纤程，但是对代码的侵入性还是太强，
如果读者要在生成环境使用，得做好严格的测试才行。

### Actor
在纤程之上，有一种称之为Actor的著名设计模式。
所谓Actor模式，是指用Actor来表示一个个的活动实体，这些活动实体之间以消息的方式进行通信和交互。
Actor模式非常适用的一种场景是游戏开发。比如DotA游戏里的每个小兵，就可以用一个个的Actor表示。
如果要小兵去攻击防御塔，就给这个小兵Actor发一条消息，让其移动到塔下，再发一条消息，让其攻击塔。
当然Actor设计模式本身不只是为了游戏开发而诞生，它是一种通用的应对大规模并发场景的系统设计方案。
最有名的Actor系统非Erlang莫属，而Erlang系统正是构建在纤程之上。再比如Quasar也有自己的Actor系统。

并非所有的Actor系统都构建在纤程之上，比如JVM平台Actor系统实现Akka。
由于Akka不是构建在纤程上，它在IO阻塞时也就只能依靠线程调度出去，所以Akka使用的线程也不宜过多。
虽然在Akka里面能够创建上万甚至上百万个Actor，但这些Actor被分配在少数线程里面执行。
如果Akka Actor的IO操作较多，势必分配在同一个线程里的Actor会出现排队和等待。
排在后面的Actor只能等到前面的Actor完成IO操作和计算后才能被执行，这极大地影响了程序的性能。
虽然Akka采用ForkJoinPool的work-stealing工作机制，可以让一个线程从其他线程的Actor队列获取任务执行，
对Actor的阻塞问题有一定缓解，但这并没有从本质上解决问题。究其原因，正是因为Akka使用的是线程而非纤程。
线程过多造成性能下降，限制了Akka系统不能像基于纤程的Actor系统那样给每个Actor分配一个纤程，而只能是多个Actor共用同一个线程。
不过如果Actor较少，每个Actor都能分配到一个线程的话，那么使用线程和使用纤程的差别就不是非常明显了。

必须强调的是，如果Actor是基于线程构建，那么在存在大量Actor时，Actor的代码逻辑就不宜做过多IO甚至是sleep操作。
当然大多数情况下，IO操作是难以避免的。为了减少IO对其它Actor的影响，应尽量将涉及IO操作的Actor与其它非IO操作的Actor隔离开。
给涉及IO操作的Actor分配专门的线程，不让这些Actor和其它非IO操作的Actor分配到相同的线程。

### NIO配合异步编程
那除了纤程外，还有没有方法能够同时保证CPU资源和IO资源都能高效使用呢？当然也是有的。
前面说到纤程是封装得最好的非阻塞IO方案。
所以如果不用纤程，那就直接使用非阻塞IO，再结合异步编程，可以充分发挥出CPU和IO的能力。

针对异步编程，在Java 8之前，`ExecutorService`类提供了初步的异步编程支持。
在`ExecutorService`的接口方法中，`execute()`用于异步执行任务，无需等待结果，属于完全异步方案。
而`submit()`则用于异步执行任务，同步等待结果，属于半异步半同步方案。

来自Google的第三方Java库Guava，提供了更好的异步编程方案。
特别是其中的`SettableFuture`类，在`Future`基础上提供了回调机制，初步实现了方便好用的链式异步编程API。

受到诸多优秀异步编程方案的启发后，在Java 8中，JVM平台迎来了全新的异步编程方法，即`CompletableFuture`类。
可以说`CompletableFuture`类汇集了各种异步编程的场景和需求，是异步编程API的集大成者，而且还在继续完善和增强。
强烈建议读者仔细阅读CompletableFuture类的API文档，会对理解和编写高性能程序有极大帮助。

除了这些偏底层的异步编程方案外，还有很多更高级和抽象的异步编程框架，比如Akka、Vert.x、Reactive、Netty等等。
这些框架大多基于事件或消息驱动。抛开各种不同的表现形式，从某种意义上来讲，异步和事件（或消息）驱动这两个概念是等同的。


### 总结
本节分析了异步能够提高CPU和IO使用效率，进而提升程序整体性能的原因。NIO正是将同步IO异步化以提升性能表现的结果。
在下节中，我们将结合实例初步体验NIO和异步编程的优势和乐趣所在。