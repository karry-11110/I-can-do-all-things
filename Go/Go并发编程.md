根据：`极客时间《GO语言第一课》专栏--白明` `极客时间《Go并发编程实战课》--鸟窝` `书籍《Go并发编程实战》--郝林`整理而来。

# 吃透Go并发编程

![image-20221126102117780](markdown%E5%9B%BE%E7%89%87/image-20221126102117780.png)

Go 中有一个大的方向，就是任务编排用 Channel，共享资源保护用传统并发原语。

针对同一种场景，也许存在很多并发原语都适用的情况，但是一定是有最合适的 那一个。所以，你必须非常清楚每种并发原语的实现机制和适用场景，千万不要被网上的 一些文章误导，万事皆用 Channel。

# 什么是并发

## 单核处理器时代

很久以前，面向大众消费者的主流处理器（CPU）都是单核的，**操作系统的基本调度与执行单元是进程（process）**。这个时候，用户层的应用有两种设计方式，一种是单进程应用，也就是每次启动一个应用，操作系统都只启动一个进程来运行这个应用。单进程应用的情况下，用户层应用、操作系统进程以及处理器之间的关系是这样的：

![image-20221124092655785](markdown%E5%9B%BE%E7%89%87/image-20221124092655785.png)

这个设计下，每个单进程应用对应一个操作系统进程，操作系统内的多个进程按时间片大小，被轮流调度到仅有的一颗单核处理器上执行。换句话说，这颗单核处理器在某个时刻只能执行一个进程对应的程序代码，两个进程不存在并行执行的可能。**这里说的并行（parallelism），指的就是在同一时刻，有两个或两个以上的任务（这里指进程）的代码在处理器上执行。从这个概念我们也可以知道，多个处理器或多核处理器是并行执行的必要条件。**



总的来说，单进程应用的设计比较简单，它的内部仅有一条代码执行流，代码从头执行到尾，不存在竞态，无需考虑同步问题。

用户层的另外一种设计方式，就是多进程应用，也就是应用通过 fork 等系统调用创建多个子进程，共同实现应用的功能。多进程应用的情况下，用户层应用、操作系统进程以及处理器之间的关系是这样的：

![image-20221124092718521](markdown%E5%9B%BE%E7%89%87/image-20221124092718521.png)

以图中的 App1 为例，这个应用设计者将应用内部划分为多个模块，每个模块用一个进程承载执行，每个模块都是一个单独的执行流，这样，App1 内部就有了多个独立的代码执行流。**但限于当前仅有一颗单核处理器，这些进程（执行流）依旧无法并行执行**，无论是 App1 内部的某个模块对应的进程，还是其他 App 对应的进程，**都得逐个按时间片**被操作系统调度到处理器上执行。

多进程应用与单进程应用相比并没有什么质的提升。那我们为什么还要将应用设计为多进程呢？这更多是**从应用的结构角度去考虑的**，多进程应用由于将功能职责做了划分，并指定专门的模块来负责，所以从结构上来看，要比单进程更为清晰简洁，可读性与可维护性也更好。这种将程序分成多个可独立执行的部分的结构化程序的设计方法，就是**并发设计**。采用了并发设计的应用也可以看成是一组独立执行的模块的组合。



## 多核处理器时代

**不过，进程并不适合用于承载采用了并发设计的应用的模块执行流。**因为进程是操作系统中资源拥有的基本单位，它不仅包含应用的代码和数据，还有系统级的资源，比如文件描述符、内存地址空间等等。进程的“包袱”太重，这导致它的创建、切换与撤销的代价都很大。于是线程便走入了人们的视野，**线程就是运行于进程上下文中的更轻量级的执行流。**同时随着处理器技术的发展，多核处理器硬件成为了主流，这让真正的并行成为了可能，于是主流的应用设计模型变成了这样：

![image-20221124092837162](markdown%E5%9B%BE%E7%89%87/image-20221124092837162.png)

**最终！！！我们看到，基于线程的应用通常采用单进程多线程的模型，一个应用对应一个进程，应用通过并发设计将自己划分为多个模块，每个模块由一个线程独立承载执行。多个线程共享这个进程所拥有的资源，但线程作为执行单元可被独立调度到处理器上运行。线程的创建、切换与撤销的代价相对于进程是要小得多。当这个应用的多个线程同时被调度到不同的处理器核上执行时，我们就说这个应用是并行的**



**并发与并行总结：**

​		就像 Go 语言之父 Rob Pike 曾说过那样**：并发不是并行，并发关乎结构，并行关乎执行**。**并发是在应用设计与实现阶段要考虑的问题。并发考虑的是如何将应用划分为多个互相配合的、可独立执行的模块的问题。**



# Go的并发方案(CSP模型)：goroutine

Go 并没有使用操作系统线程作为承载分解后的代码片段（模块）的基本执行单元，而是实 现了**goroutine这一由 Go 运行时（runtime）负责调度的、轻量的用户级线程**，为并发 程序设计提供**原生支持**

## **goroutine优势**

相比传统操作系统线程来说，goroutine 的优势主要 是：

- 资源占用小，每个 goroutine 的初始栈大小仅为 2k； 
- 由 Go 运行时而不是操作系统调度，goroutine 上下文切换在用户层完成，开销更小；
-  在语言层面而不是通过标准库提供。goroutine 由go关键字创建，一退出就会被回收或 销毁，开发体验更佳； 
- 语言内置 channel 作为 goroutine 间通信原语，为并发设计提供了强大支撑



Go编程语言思维：和传统编程语言不同的是，Go 语言是面向并发而生的，所以，在程序的结构设 计阶段，**Go 的惯例是优先考虑并发设计**。这样做的目的更多是考虑随着外界环境的变化， 通过并发设计的 Go 应用可以更好地、更自然地适应**规模化（scale）**



## **goroutine的基本用法**

并发是一种能力，它让你的程序可以由**若干个代码片段组合**而成，并且**每个片段都是独立 运行**的。goroutine 恰恰就是 Go 原生支持并发的一个具体实现。**无论是 Go 自身运行时 代码还是用户层 Go 代码，都无一例外地运行在 goroutine 中。**

> **创建**

Go 语言通过go关键字+函数/方法的方式创建一个 goroutine。创建后，新 goroutine 将 拥有独立的代码执行流，并与创建它的 goroutine 一起被 Go 运行时调度。

```go
go fmt.Println("I am a goroutine")
var c = make(chan int)
go func(a, b int) {
c <- a + b
}(3,4)
// $GOROOT/src/net/http/server.go
c := srv.newConn(rw)
go c.serve(connCtx)
```

通过 go 关键字，我们可以基于已有的具名函数 / 方法创建 goroutine，也可 以基于匿名函数 / 闭包创建 goroutine。

创建 goroutine 后，go 关键字不会返回 goroutine id 之 类的唯一标识 goroutine 的 id，你也不要尝试去得到这样的 id 并依赖它。

另外，和线程 一样，一个应用内部启动的所有 goroutine 共享进程空间的资源，如果多个 goroutine 访 问同一块内存数据，将会存在竞争，我们需要进行 **goroutine 间的同步。**



> **退出**

多数情况下， 我们不需要考虑对 goroutine 的退出进行控制：**goroutine 的执行函数的返回，就意味着 goroutine 退出。**

如果 main goroutine 退出了，那么也意味着整个应用程序的退出。

此外，你还要注意的 是，goroutine 执行的函数或方法即便有返回值，Go 也会忽略这些返回值。所以，如果你 要获取 goroutine 执行后的返回值，你需要另行考虑其他方法，比如通过 **goroutine 间的 通信**来实现。



## **goroutine间通信方式**

传统的编程语言（比如：C++、Java、Python 等）并非面向并发而生的，所以他们面对 并发的逻辑多是基于操作系统的线程。并发的执行单元（线程）之间的通信，利用的也是 操作系统提供的线程或进程间通信的原语，比如：共享内存、信号（signal）、管道 （pipe）、消息队列、套接字（socket）等，使用最多、最广泛的（也是最高效的）是结合了线程同步原语（比 如：锁以及更为低级的原子操作）的共享内存方式，因此，我们可以说传统语言的并发模 型是基于**对内存的共享**的。

- 难用
- 易错



Go 语言从设计伊始，就将解决上面这个传统并发模型的问题作为 Go 的一个目标，并在新并发模型设计中借鉴了著名计算机科学家Tony Hoare提出的**CSP（Communicationing Sequential Processes，通信顺序进程）并发模型。**在 Tony Hoare 眼中，**一个符合 CSP 模型的并发程序应该是一组通过输入输出原语 连接起来的 P 的集合。**

CSP 理论不仅是一个并发参考模型，也是一种并 发程序的程序组织方法。它的组合思想与 Go 的设计哲学不谋而合。![image-20221124094948104](markdown%E5%9B%BE%E7%89%87/image-20221124094948104.png)

注意了，这里的 P 并不一定与操作系统的进程或线程划等号。在 Go 中，与“Process”对 应的是 goroutine。



为了实现 CSP 并发模型中的输入和输出原语，Go 还引入了 goroutine（P）之间的**通信原语channel**。goroutine 可以从 channel 获取输入数据， 再将处理后得到的结果数据通过 channel 输出。通过 **channel 将 goroutine（P）组合连 接在一起**，让设计和编写大型并发系统变得更加**简单和清晰**



虽然 **CSP 模型已经成为 Go 语言支持的主流并发模型**，但 Go 也支持传统的、基于共享内 存的并发模型，并提供了基本的低级别同步原语（主要是 sync 包中的互斥锁、条件变量、 读写锁、原子操作等）。

- 从程序的整体结构来看，Go 始终推荐**以 CSP 并发模型风格构建并发程序**，尤 其是在复杂的业务层面，这能提升程序的逻辑清晰度，大大降低并发设计的复杂性，并让 程序更具可读性和可维护性。
- 对于局部情况，比如涉及性能敏感的区域或需要保护的结构体数据时，我们可以使 用更为高效的低级**同步原语**（如 mutex），保证 goroutine 对数据的同步访问。
- 当然还有Context!






# Goroutine调度(GPM)原理

## why调度

单进程-----多进程时代后：面临多进程/线程下的问题

1. 高cpu消耗：进程/线程数量越多，cpu消耗大，因为切换成本高，越浪费（比如显示的CPU利用率为100，但执行程序可能只利用了60，而切换线程占了40），所以尽量搞少一点进程/线程运行，提高cpu利用率才是软件层面要做的事，因此要调度。
2. 高内存占用：进程占用4GB, 线程占用4MB。
3. 设计复杂：伴随资源竞争，切换成本大等



## Goroutine 调度器 

​		提到“调度”，我们首先想到的就是操作系统对进程、线程的调度。**操作系统调度器会将系统中的多个线程按照一定算法调度到物理 CPU 上去运行。**

​		传统的编程语言，比如 C、C++ 等的并发实现，多是基于线程模型的，也就是应用程序负责创建线程（一般通过 libpthread 等库函数调用实现），操作系统负责调度线程。当然，我们也说过，这种传统支持并发的方式有很多不足。为了解决这些问题，Go 语言中的并发实现，使用了 Goroutine，代替了操作系统的线程，也不再依靠操作系统调度。

​		Goroutine 占用的资源非常小，每个 Goroutine 栈的大小默认是 2KB。而且，Goroutine 调度的切换也不用陷入（trap）操作系统内核层完成，代价很低。因此，一个 Go 程序中可以创建成千上万个并发的 Goroutine。而将这些 Goroutine 按照**一定算法**放到“CPU”上执行的程序，就被称为 **Goroutine 调度器（**Goroutine Scheduler），注意，这里说的“CPU”打了引号。

​		不过，一个 Go 程序对于操作系统来说只是一个用户层程序，操作系统眼中只有线程，它甚至不知道有一种叫 Goroutine 的事物存在。所以，Goroutine 的调度全要靠 Go 自己完成。那么，实现 Go 程序内 Goroutine 之间“公平”竞争“CPU”资源的任务，就落到了 Go 运行时（runtime）头上了。要知道在一个 Go 程序中，除了用户层代码，剩下的就是 Go 运行时了。于是，**Goroutine 的调度问题就演变为，Go 运行时如何将程序内的众多 Goroutine，按照一定算法调度到“CPU”资源上运行的问题了。**

**可是，在操作系统层面，线程竞争的“CPU”资源是真实的物理 CPU，但在 Go 程序层面，各个 Goroutine 要竞争的“CPU”资源又是什么呢？**

Go 程序是用户层程序，它本身就是整体运行在一个或多个操作系统线程上的。所以这个答案就出来了：Goroutine 们要竞争的“CPU”资源就是操作系统线程。这样，**Goroutine 调度器的任务也就明确了：Go运行时将 Goroutine 按照一定算法放到不同的操作系统线程中去执行。把大量的goroutine分配到少量线程上去执行，并利用多核并行，实现更强大的并发。**



**goroutine调度引发的问题：**

n :  1模型：无法利用多个cpu，出现阻塞瓶颈

1：1模型：更多线程/多进程模型无异

**m:  n模型： 能够利用多核，怎么优化协程调度器和算法才是关键**



## Goroutine 调度器模型与演化过程

Goroutine 调度器的实现不是一蹴而就的，它的调度模型与算法也是几经演化，从最初的 G-M 模型、到 G-P-M 模型，从不支持抢占，到支持协作式抢占，再到支持基于信号的异步抢占。

> **G-M模型**

2012 年 3 月 28 日，Go 1.0 正式发布。在这个版本中，Go 开发团队实现了一个简单的 Goroutine 调度器。在这个调度器中，每个 Goroutine 对应于运行时中的一个抽象结构： G ， 而被视作“物理 CPU”的操作系统线程，则被抽象为另外一个结构：M。**调度器的工作就是将 G 调度到 M 上去运行。**为了更好地控制程序中活跃的 M 的数量，调 度器引入了 GOMAXPROCS 变量来表示 Go 调度器可见的“处理器”的最大数量。

G-M 模型的一个重要不足：**限制了 Go 并发程序的伸 缩性，尤其是对那些有高吞吐或并行计算需求的服务程序。**

- 单一全局互斥锁(Sched.Lock) 和集中状态存储的存在，导致所有 Goroutine 相关操 作，比如创建、重新调度等，都要上锁； 
- Goroutine 传递问题：M 经常在 M 之间传递“可运行”的 Goroutine，这导致调度延 迟增大，也增加了额外的性能损耗；
-  每个 M 都做内存缓存，导致内存占用过高，数据局部性较差；
-  由于系统调用（syscall）而形成的频繁的工作线程阻塞和解除阻塞，导致额外的性能损 耗



> **G-P-M模型**

德米特里 - 维尤科夫又亲自操刀改进了 Go 调度器，在 Go 1.1 版本中 实现了 G-P-M 调度模型和work stealing 算法。

![image-20221124101518710](markdown%E5%9B%BE%E7%89%87/image-20221124101518710.png)

“**计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决**”，德 米特里 - 维尤科夫的 G-P-M 模型恰是这一理论的践行者。德米特里 - 维尤 科夫通过向 G-M 模型中增加了一个 P，让 Go 调度器具有很好的伸缩性。

P 是一个“逻辑 Proccessor”，每个 G要想真正运行起来，首先需要被分 配一个 P，也就是进入到 P 的本地运行队列中。对于 G 来说，**P 就是运行 它的“CPU”**，可以说：**在 G 的眼里只有 P**。但**从 Go 调度器的视角来看**，**真正 的“CPU”是 M**，只有将 P 和 M 绑定，才能让 P 的 本地队列中的 G 真正运行起来。

**缺点**：G-P-M 模型的实现算是Go调度器的一大进步，但调度器仍然有一个令人头疼的问题，那就 是**不支持抢占式调度**，这导致一旦某个 G 中出现死循环的代码逻辑，那么 G 将永久占用分 配给它的 P 和 M，而位于同一个 P 中的其他 G 将得不到调度，出现“**饿死**”的情况。更为严重的是，当只有一个 P（GOMAXPROCS=1）时，整个 Go 程序中的其他 G 都 将“饿死”

于是德米特里 - 维尤科夫又提出了《Go Preemptive Scheduler Design》并在 Go 1.2 中实现了**基于协作的“抢占式”调度**。 这个抢占式调度的**原理就是，Go 编译器在每个函数或方法的入口处加上了一段额外的代码 ，让Go运行时有机会在这段代码中检查是否需要执行抢占调 度。**

**缺点：**只能说局部解决了“饿死”问题，只在有函数调用的地方才能插入“抢 占”代码（埋点），对于没有函数调用而是纯算法循环计算的 G，Go 调度器依然无法抢 占。比如，死循环等并没有给编译器插入抢占代码的机会，这就会导致 GC 在等待所有 Goroutine 停止时的等待时间过长，从而导致 GC 延迟，内存占用瞬间冲高；甚至在一 些特殊情况下，导致在 STW（stop the world）时死锁

Go 在 1.14 版本中接受了奥斯汀 - 克莱门茨（Austin Clements）的 提案，增加了对**非协作的抢占式调度的支持，这种抢占式调度是基于系统信号的**，也就 是通过**向线程发送信号的方式来抢占正在运行的 Goroutine**。



## **深入G-P-M模型**

> **GPM**

**G**:  代表 Goroutine，存储了 Goroutine 的执行栈信息、Goroutine 状态以及 Goroutine 的任务函数等，而且 G 对象是可以重用的

**P**:  代表逻辑 processor，P 的数量决定了系统内最大可并行的 G 的数量，P 的最大作用还是其拥有的各种 G 对象队列、链表、一些缓存和状态；

**M:**  M 代表着真正的执行计算资源，当前操作系统分配到**当前Go程序**的内核线程数。在绑定有效的 P 后，进入一个调度循环，而**调度循环的机制大致是从 P 的本地运行队列以及全局队列中获取 G，切换到 G 的执行栈上并执行 G 的函数，调用 goexit 做清理工作并回到 M，如此反复**。**M 并不保留 G 状态，这是 G 可以跨 M 调度的基础。**

**全局队列：**存放等待运行的队列

**P的本地队列：**存放等待运行的G，有数量限制不超过256个G， 先将新创建的G放在P的本地队列中如果满了则放在全局队列中

**数量：**M与P的数量没有绝对关系，一个M阻塞，P就会去创建或者切换另一个M，所以，即使P的默认数量是1，也有可能会创建很多个M出来

- P: 环境变量￥GOMAXPROCS或者在程序中通过runtime.GOMAXPROCS()来设置。
- M： Go语言本身限定M的最大量是10000，runtime/debug包中SetMaxThreads函数来设置，有一个M阻塞，则会创建一个新的M，如果M有空闲，就会回收或者睡眠。

**Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了1个内核线程，OS调度器负责把内核线程分配到CPU的核上执行**。

**而 Goroutine 调度器的目标，就是公平合理地将各个 G 调度到 P 上“运行”**。



> **G 被抢占调度**

如果某个 G 没有进行系统调用（syscall）、没有进行 I/O 操 作、没有阻塞在一个 channel 操作上，调**度器是如何让 G 停下来并调度下一个可运行的 G 的呢**？ 答案就是：**G 是被抢占调度的。**

除非极端的无限循环，否则只要 G 调用函数，Go 运行时就有了抢占 G 的机 会。Go 程序启动时，运行时会去启动一个名为 sysmon 的 M（一般称为监控线程），这 个 M 的特殊之处在于它不需要绑定 P 就可以运行（以 g0 这个 G 的形式），这个 M 在整 个 Go 程序的运行过程中至关重要，**如果一个 G 任务运行 10ms，sysmon 就会认为它的运 行时间太久而发出抢占式调度的请求**。一旦 G 的抢占标志位被设为 true，那么等到这个 G 下一次调用函数或方法时，运行时就可以将 G 抢占并移出运行状态，放入队列中，等待下 一次被调度。

sysmon 主要完成了这些工作

- 释放闲置超过 5 分钟的 span 内存；
-  如果超过 2 分钟没有垃圾回收，强制执行；
-  将长时间未处理的 netpoll 结果添加到任务队列；
-  向长时间运行的 G 任务发出抢占调度； 
- 收回因 syscall 长时间阻塞的 P；



还有**两个特殊情况下 G 的调度方法**

- **第一种：channel 阻塞或网络 I/O 情况下的调度。**

  如果 G 被阻塞在某个 channel 操作或网络 I/O 操作上时，G 会被放置到某个等待（wait） 队列中，而 M 会尝试运行 P 的下一个可运行的 G。如果这个时候 P 没有可运行的 G 供 M 运行，那么 M 将解绑 P，并进入挂起状态。当 I/O 操作完成或 channel 操作完成，在等 待队列中的 G 会被唤醒，标记为可运行（runnable），并被放入到某 P 的队列中，绑定一 个 M 后继续执行。

- **第二种：系统调用阻塞情况下的调度。**

  如果 G 被阻塞在某个系统调用（system call）上，那么不光 G 会阻塞，执行这个 G 的 M 也会解绑 P，与 G 一起进入挂起状态。如果此时有空闲的 M，那么 P 就会和它绑定，并继 续执行其他 G；如果没有空闲的 M，但仍然有其他 G 要去执行，那么 Go 运行时就会创建 一个新 M（线程）。

  当系统调用返回后，阻塞在这个系统调用上的 G 会尝试获取一个可用的 P，如果没有可用 的 P，那么 G 会被标记为 runnable，之前的那个挂起的 M 将再次进入挂起状态。

  

> **GMP的设计思想**

1. **复用线程**： 避免频繁的创建，销毁线程，而是对线程的复用。

   ​				**work stealing机制**：当本线程无可运行的G时，尝试从其他线程绑定（无奈的时候从全局队列，但需要加锁解锁操作，消耗大）的P偷取G，而不是销毁线程

   ​				**hand off 机制：** 当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行。

2. **利用并行**：GOMAXPROCS设置P的数量，最多有GOMAXPROCS个线程分布在多个CPU上同时运行。

3. **抢占**：在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU10ms，防止其他goroutine被饿死，保证了并发，超过特定长的时间后，G 会被设置为**可抢占，**并在下一次执行函数或方法时被 Go 运行时移出运行状态。

4. **全局G队列**：当M执行work stealing从其他本地队列偷不到G时，可以从全局队列中获取G。



> **调度器生命周期**

**M0 :** M0是启动程序后编号为0的主线程，进程唯一。这个M对应的实例会在全局变量runtime.m0中，不需要在heap上分配，M0负责执行初始化操作和启动第一个G，之后就和其他的M一样了。

**G0：**G0是每次启动一个M都会第一个创建的goroutine,线程唯一。G0仅勇于负责调度的G，G0不指向任何可执行的函数，每个M都会有自己的G0，**全局变量的G0**是M0的G0。在调度或者系统调用时当前M会使用对应的G0的栈空间，即当前M会让G0这个特殊的goroutine来调度G。

调度器的生命周期几乎占满了一个Go程序的一生，`runtime.main`的goroutine执行之前都是为调度器做准备工作，`runtime.main`的goroutine运行，才是调度器的真正开始，直到`runtime.main`结束而结束。



> ##### go func() 经历过程

1. go关键字创建一个goroutine
2. 新创建的goroutine会优先保存在当前线程M的的P的本地队列中，如果P的本地队列满了的话会保存在全局队列中。
3. G只能运行在M中，M和P是1：1的关系。M会从P的本地队列中弹出一个可执行状态的G（**可抢占的G的状态不是可执行**）来执行，如果本地队列为空就会从其他MP组合偷一个，实在不行就偷全局队列的。 （work stealing机制）
4. M调度G执行的过程是循环机制。
   - 当M执行G发生了syscall或其他阻塞操作，M会阻塞，如果当前有一些G在执行，runtime**（sysmon机制的信号抢占调度）**会把这个线程M和P解绑，再创建一个新的操作系统的线程(如果**M队列有空闲的线程可用就复用空闲线程**)来服务P （hand off机制）
   - 当M系统调用结束，这个G会尝试获取一个空闲的P执行，并放入到P的本地队列中。如果获取不到，M线程变为休眠状态，加入到空闲线程中，G放到全局队列中



# GPM调度场景全过程分析

场景一：P拥有G1,M1获取P后开始运行G1,G1使用go func()创建了G2，为了局部性，G2优先加入到P1的本地队列。

![image-20221124200132786](markdown%E5%9B%BE%E7%89%87/image-20221124200132786.png)

场景二：G1运行完成（函数：goexit）,M上运行的goroutine切换为G0，G0负责调度时协程的切换（函数：schedule）。从P的本地队列取G2，从G0切换到G2，并开始运行G2（函数：execute），实现了线程M1的复用

![image-20221124200224863](markdown%E5%9B%BE%E7%89%87/image-20221124200224863.png)

场景三：假设每个P的本地队列只能存4个G。G2要创建了6个G，前3个G（G3, G4, G5）已经加入p1的本地队列，p1本地队列满了。

![image-20221124200435105](markdown%E5%9B%BE%E7%89%87/image-20221124200435105.png)

场景四：G2在创建G7的时候，发现P1的本地队列已满，需要执行**负载均衡**(把P1中本地队列中前一半的G，还有新创建G**转移**到全局队列)这些G被转移到全局队列时，会被打乱顺序。所以G3,G4,G7被转移到全局队列。

![image-20221124200611931](markdown%E5%9B%BE%E7%89%87/image-20221124200611931.png)

场景五：G2创建G8时，P1的本地队列未满，所以G8会被加入到P1的本地队列。

![image-20221124200648115](markdown%E5%9B%BE%E7%89%87/image-20221124200648115.png)

场景六：唤醒正在休眠的M，造成自旋线程。

​			规定：**再创建G时，运行的G会尝试唤醒其他空闲的P和M组合取执行**，这里假定G2唤醒了M2，M2绑定了P2，并运行G0，但P2本地队列没有G，M2此时为自旋线程**（没有G但为运行状态的线程，不断寻找G）**

![image-20221124201102062](markdown%E5%9B%BE%E7%89%87/image-20221124201102062.png)

场景七：M2会尝试从**全局队列（GQ）中取一批G放到P2的本地队列**（函数 ：findrunnable()）。数量符合公式：n=min(len(GQ)/GOMAXPROCS+1,len(GQ/2))

![image-20221124200812932](markdown%E5%9B%BE%E7%89%87/image-20221124200812932.png)

场景八：全局队列没有G，那么M就要执行work stealing，从其他有G的P那里偷取一半G过来，放到自己的P本地队列。P2从P1的本地队列尾部去一般的G。向下取整。

![image-20221124200839103](markdown%E5%9B%BE%E7%89%87/image-20221124200839103.png)

场景九：最多有GOMAXPROCS个自旋的线程，多余没事的线程会休眠。

![image-20221124200854140](markdown%E5%9B%BE%E7%89%87/image-20221124200854140.png)

场景十：

​			假设：当前除了M3和M4为自旋线程，还有M5和M6为空闲的线程（没有P的绑定，注意最多只能存在4个P，所以M>=P，大部分时间都是M再抢占需要运行的P），G8创建了G9，G8进行了阻塞的系统调用。M2和P2立即解绑。

​			P2会执行以下判断：如果P2本地队列或全局队列有G，或者有空闲的M，P2会立马唤醒一个M和它绑定，否则加入到空闲P列表，等待M来获取可用的P。

![image-20221124200908368](markdown%E5%9B%BE%E7%89%87/image-20221124200908368.png)

场景十一：M2会和P2解绑，但M2会记住P2。然后G8和M2会进入系统调用状态，当G8和M2退出系统调用非阻塞，会尝试获取P2,如无法获取，则获取空闲的的P，依然没有，G8标记为可运行状态并加入到全局队列。M2会因为没有P的绑定而变成休眠状态（长时间休眠等待GC回收销毁）

![image-20221124200924048](markdown%E5%9B%BE%E7%89%87/image-20221124200924048.png)



# channel

Go 语言实现了基于 CSP（Communicating Sequential Processes）理论的并发方案。

 Go 语言的 CSP 模型的实现包含两个主要组成部分：一个是 Goroutine，它是 Go 应用并 发设计的基本构建与执行单元；另一个就是 channel，它在并发模型中扮演着重要的角 色。channel 既可以用来实现 Goroutine 间的通信，还可以实现 Goroutine 间的同步。 它就好比 Go 并发设计这门“武功”的秘籍口诀，可以说，学会在 Go 并发设计时灵活运 用 channel，才能说真正掌握了 Go 并发设计的真谛。

## **一等公民：channel**的基本用法

Go 在语法层面将channel 作为一等公民意味着：**可以像使用普通变量那样使用 channel**：

- 定义 channel 类型变量
- 给 channel 变量赋值
- 将 channel 作为参数传递给函数 / 方法
- 将 channel 作为返回值从函数 / 方法中返回
- 甚至将 channel 发送到其他 channel 中

Go之父：**“执行业务处理的 goroutine 不要通过 共享内存的方式通信，而是要通过 Channel 通信的方式分享数据。”**



> **创建channel**

和切片、结构体、map 等一样，channel 也是一种复**合数据**类型。也就是说，我们在声明 一个 channel 类型变量时，必须给出其具体的元素类型

```go
var ch chan int
```

如果 channel 类型变量在声明时没有被赋予初值，那么它的默认值为 nil。**对值是 nil 的 chan 的发送接收调用者总是会阻塞，关闭nil channel则会发生panic。**

**channel 中的元素是任意的类型，所以也可能是 channel 类型**：当channel中元素也是channel类型，“<-”有个规则，总是尽量和左边的 chan 结合





并且，和其他复 合数据类型支持使用复合类型字面值作为变量初始值不同，**为 channel 类型变量赋初值的 唯一方法就是使用 make 这个 Go 预定义的函数**

```go
ch1 := make(chan int)
ch2 := make(chan int, 5)
```

第一行我们通过make(chan T)创建的、元素类型为 T 的 channel 类型，是**无缓冲 channel**，而第二行中通过带有 capacity 参数的make(chan T, capacity)创建的元素 类型为 T、缓冲区长度为 capacity 的 channel 类型，是**带缓冲 channel**。这两种类型的变量关于发送（send）与接收（receive）的特性是不同的



> **发送与接收**

**在理解 channel 的发送与接收操作时，你一定要始终牢记： channel 是用于 Goroutine 间通信的，所以绝大多数对 channel 的读写都被分别放在了 不同的 Goroutine 中。**

Go 提供了<-操作符用于对 channel 类型变量进行发送与接收操作：

```go
ch1 <- 13 // 将整型字面值13发送到无缓冲channel类型变量ch1中
n := <- ch1 // 从无缓冲channel类型变量ch1中接收一个整型值存储到整型变量n中
ch2 <- 17 // 将整型字面值17发送到带缓冲channel类型变量ch2中
m := <- ch2 // 从带缓冲channel类型变量ch2中接收一个整型值存储到整型变量m中
```

**无缓冲channel的发送与接收**：

Goroutine 对无缓冲 channel 的接收和发送操作是**同步**的。**对无缓冲 channel 类型的发送与接收操作，一定要放在两个不 同的 Goroutine 中进行，否则会导致 deadlock。**



**带缓冲 channel 的发送与接收：**

对带缓冲 channel 的发送操作在缓冲区未满、接收操作在缓冲区非空的情况下是**异步**的

- 但当缓冲区满了的情况下，对它进行发送操作的 Goroutine 就会阻塞挂起；
- 当缓冲区为空 的情况下，对它进行接收操作的 Goroutine 也会阻塞挂起。

**带缓冲 channel 与无缓冲 channel 在发送与接收行为上的差异，在具体使用 上，它们有各自的“用武之地”**



**只发送和只接受channel类型：**

```go
ch1 := make(chan<- int, 1) // 只发送channel类型
ch2 := make(<-chan int, 1) // 只接收channel类型
<-ch1 // invalid operation: <-ch1 (receive from send-only type chan<- in
ch2 <- 13 // invalid operation: ch2 <- 13 (send to receive-only type <-chan
```

通常只**发送 channel 类型和只接收 channel 类型**，会被用作函数的参数类型或返回值，用于限制对 channel 内的操作，或者是明确可对 channel 进行的操作的类型。**函数传参及任何赋值操作中可以将双向通道转换为单向通道，但反过来是不可以的**。**理解小技巧:**这个箭头总是射向左边的，元素类型总在最右边。如果箭头指向 chan，就表示可以往 chan 中塞数据；如果箭头远离 chan，就表示 chan 会往外吐数据



> **关闭channel**

调用 Go 内置的 close 函数关闭了 channel。channel 关闭后，所有等待从这个 channel 接收数据的操作都将返回。

```go
n := <- ch // 当ch被关闭后，n将被赋值为ch元素类型的零值
m, ok := <-ch // 当ch被关闭后，m将被赋值为ch元素类型的零值, ok值为false
for v := range ch { // 当ch被关闭后，for range循环结束
... ...
}
```

通过“comma, ok”惯用法或 for range 语句，我们可以准确地判定 channel 是否被关闭。而单纯采用n := <-ch形式的语句，我们就无法判定从 ch 返回的元素类型 零值，究竟是不是因为 channel 被关闭后才返回的。

channel 的一个使用惯例，那就是**发送端负责关闭 channel。** **因为发送端没有像接受端那样的、可以安全判断 channel 是否被关闭了的方法** 。

- 因为，如果在接收端关闭，发送端不知道的话，一旦向一个已经关闭的 channel 执行发送操作，这个操作就会引发 panic（或者说发送端关闭以及关闭的channel也会panic），而接受操作将会得到该channel**元素类型的零值**

```go
ch := make(chan int, 5)
close(ch)
ch <- 13 // panic: send on closed channel
```



**总结**：![image-20221126113310144](markdown%E5%9B%BE%E7%89%87/image-20221126113310144.png)



## channel的实现原理

> **channel的数据结构**

![image-20221126104326140](markdown%E5%9B%BE%E7%89%87/image-20221126104326140.png)

- **qcount**：代表 chan 中已经接收但还没被取走的元素的个数。内建函数 len 可以返回这 个字段的值。
-  **dataqsiz**：队列的大小。chan 使用一个循环队列来存放元素
- **elemtype 和 elemsize**：chan 中元素的类型和 size。因为 chan 一旦声明，它的元素 类型是固定的，即普通类型或者指针类型，所以元素大小也是固定的
- **sendx**：处理发送数据的指针在 buf 中的位置。一旦接收了新的数据，指针就会加上 elemsize，移向下一个位置。buf 的总大小是 elemsize 的整数倍，而且 buf 是一个循 环列表。
-  **recvx**：处理接收请求时的指针在 buf 中的位置。一旦取出数据，此指针会移动到下一 个位置。
-  **recvq**：chan 是多生产者多消费者的模式，如果消费者因为没有数据可读而被阻塞了， 就会被加入到 recvq 队列中。
-  **sendq**：如果生产者因为 buf 满了而阻塞，会被加入到 sendq 队列中。



> **初始化**

Go 在编译的时候，会根据容量的大小选择调用 makechan64，还是 makechan。只关注 makechan 就好了，**因为 makechan64 只是做了 size 检查，底层还是调用 makechan 实现的**。makechan 的目标就是生成 **hchan 对象**。

- 会根据 chan 的容量的大小和元素的类型不同，初始化不同的存储空间
- chan的size或者元素的size是0，不必创建buf
- 元素不是指针，分配一块连续的内存给hchan数据结构和buf
- 元素包含指针，那么单独分配buf
-  元素大小、类型、容量都记录下来



> **接收、发送、关闭操作实现**

**接收**

在处理从 chan 中接收数据时，Go 会把代码转换成 chanrecv1 函数，如果要返回两个返 回值，会转换成 chanrecv2，chanrecv1 函数和 chanrecv2 会调用 chanrecv。

- 第一部分是 **chan 为 nil** 的情况。和 send 一样，从 nil chan 中接收（读取、获取）数据 时，调用者会被永远阻塞。
- 第三部分是 **chan 已经被 close** 的情况。如果 chan 已经被 close 了，并且队列中没有缓存 的元素，那么返回 true、false。
- 第四部分是处理 **sendq 队列中有等待者**的情况。这个时候，如果 buf 中有数据，优先从 buf 中读取数据，否则直接从等待队列中弹出一个 sender，把它的数据复制给这个 receiver。
- 第五部分是处理**没有等待的 sender** 的情况。这个是和 chansend 共用一把大锁，所以不 会有并发的问题。如果 buf 有元素，就取出一个元素给 receiver。
-  第六部分是处理 **buf 中没有元素**的情况。如果没有元素，那么当前的 receiver 就会被阻 塞，直到它从 sender 中接收了数据，或者是 chan 被 close，才返回。



**发送**

Go 在编译发送数据给 chan 的时候，会把 send 语句转换成 chansend1 函数， chansend1 函数会调用 chansend

- 第一部分是进行判断：如果 **chan 是 nil** 的话，就把调用者 goroutine park（阻 塞休眠）， 调用者就永远被阻塞住了
- 第二部分的逻辑是当你往一个已经满了的 chan 实例发送数据时，并且想不阻塞当前调 用，那么这里的逻辑是直接返回。chansend1 方法在调用 chansend 的时候设置了阻塞参 数，所以**不会执行到第二部分**的分支里。
- 第三部分显示的是，如果 **chan 已经被 close** 了，再往里面发送数据的话会 panic。
- 第四部分，如果**等待队列中有等待的 receiver**，那么这段代码就把它从队列中弹出，然后 直接把数据交给它（通过 memmove(dst, src, t.size)），而不需要放入到 buf 中，速度可 以更快一些。
- 第五部分说明**当前没有 receiver**，需要把数据放入到 buf 中，放入之后，就成功返回
- 第六部分是处理 **buf 满**的情况。如果 buf 满了，发送者的 goroutine 就会加入到发送者的 等待队列中，直到被唤醒。这个时候，数据或者被取走了，或者 chan 被 close 了。



**关闭**

通过 close 函数，可以把 chan 关闭，编译器会替换成 closechan 方法的调用。

- 如果 **chan 为 nil**，close 会 panic；
- 如果 **chan 已 经 closed**，再次 close 也会 panic。
- 如果 **chan 不为 nil，chan 也没有 closed**，就把等待队列中的 sender（writer）和 receiver（reader）从队列中全部移除并 唤醒。



## **channel的易错场景**

> **panic**

- close 为 nil 的 chan； 
-  send 已经 close 的 chan； 
- close 已经 close 的 chan。



> **goroutine泄露**

```go
 func process(timeout time.Duration) bool {
    ch := make(chan bool)
    go func() {
        // 模拟处理耗时的业务
        time.Sleep((timeout + time.Second))
        ch <- true // block
        fmt.Println("exit goroutine")
    }()
    select {
        case result := <-ch:
        	return result
        case <-time.After(timeout):
        	return false
    }
}

```

如果发生超时，process 函数就返回了，这就会导致 unbuffered 的 chan 从来就没有被读 取。我们知道，unbuffered chan 必须等 reader 和 writer 都准备好了才能交流，否则就 会阻塞。

解决这个 Bug 的办法很简单，就是将 unbuffered chan 改成容量为 1 的 chan，这样第 7 行就不会被阻塞了。



## **select**

当**涉及同时对多个 channel 进行操作时**，我们会结合 Go 为 CSP 并发模型提供的另外一个 原语 **select**，一起使用。通过 select，我们可以同时在多个 channel 上进行发送 / 接收操作

```go
select {
    case x := <-ch1: // 从channel ch1接收数据
    ... ...
    case y, ok := <-ch2: // 从channel ch2接收数据，并根据ok值判断ch2是否已经关闭
    ... ...
        case ch3 <- z: // 将z值发送到channel ch3中:
    ... ...
    default: // 当上面case中的channel通信均无法实施时，执行该默认分支
}
```

当 select 语句中没有 default 分支，而且所有 case 中的 channel 操作都阻塞了的时候， 整个 select 语句都将被阻塞，直到某一个 case 上的 channel 变成可发送，或者某个 case 上的 channel 变成可接收，select 语句才可以继续进行下去。

- 如果多个case同时满足，select会随机选择一个。
- **对于没有case的select{}会一直等待，可用于阻塞main函数。**



## channel or 并发原语？

Channel 并不是处理 并发问题的“银弹”，有时候使用并发原语更简单，而且不容易出错。所以，我给你提供 一套**选择的方法**: 

-  共享资源的并发访问使用传统并发原语；
- 复杂的任务编排和消息传递使用 Channel； 
- 消息通知机制使用 Channel，除非只想 signal 一个 goroutine，才使用 Cond
- 简单等待所有任务的完成用 WaitGroup，也有 Channel 的推崇者用 Channel，都可 以
- 需要和 Select 语句结合，使用 Channel
- 需要和超时配合时，使用 Channel 和 Context



## channel的应用场景

Go之父：**“执行业务处理的 goroutine 不要通过 共享内存的方式通信，而是要通过 Channel 通信的方式分享数据。”**

根据应用场景可以分为六类：

- **信号通知**：一个 goroutine 可以将信号 (closing、closed、data ready 等) 传递给另一 个或者另一组 goroutine  **`无缓冲的channel的同步性`**
- **锁**：利用 Channel 也可以实现互斥锁的机制   **`无缓冲的channel的同步性`**

- **数据传递**：一个 goroutine 将数据交给另一个 goroutine，相当于把数据的拥有权 (引 用) 托付出去 **`无缓冲的channel，紧密联系要实时性`**
- **任务编排**：可以让一组 goroutine 按照一定的**顺序并发或者串行**的执行，这就是编排的 功能 **`无缓冲的channel，紧密联系要实时性`**



- **数据交流**：当作并发的 buffer 或者 queue，解决生产者 - 消费者问题。多个 goroutine 可以并发当作生产者（Producer）和消费者（Consumer）      **`有缓冲的channel性能更好`**

- **计数信号量**：带缓冲 channel 中的当前数据个数代表的是，当前同时处于**活动状态**（处理业务）的 Goroutine 的数量带缓冲 channel 的容量（capacity），就代表了**允许同时处于活动状态的 Goroutine 的最大数量**。 **`有缓冲的channel`**

  

### nil channel 的妙用

如果一个 channel 类型变量的值为 nil，我们称它为 nil channel。nil channel 有一个特 性，那就是**对 nil channel 的读写都会发生阻塞**

```go
func main() {
	ch1, ch2 := make(chan int), make(chan int)
	go func() {
		time.Sleep(6 * time.Second)
		ch1 <- 1
		close(ch1)
	}()
	go func() {
		time.Sleep(7 * time.Second)
		ch2 <- 2
		close(ch2)
	}()
	var ok1, ok2 bool
	for {
		//select {
		//case x := <-ch1:
		//	ok1 = true
		//	fmt.Println(x)
		//case x := <-ch2:
		//	ok2 = true
		//	fmt.Println(x)
		//}
		select {
		case x, ok := <-ch1:
			if !ok {
				ch1 = nil
			} else {
				ok1 = true
				fmt.Println(x)
			}

		case x, ok := <-ch2:
			if !ok {
				ch1 = nil
			} else {
				ok2 = true
				fmt.Println(x)
			}
		}
		if ok1 && ok2 {
			break
		}
	}
}
```

期望程序在接收完 ch1 和 ch2 两个 channel 上的数据后就退出,期望上面这个在依次输出1 和 2两个数字后退出，但实际运行的输出结果却是在 输出 1之后，程序输出了许多的 0 值，之后才输出2 并退出。

- 因为：由于 ch1 被关闭，从一个已关闭的 channel 接收数据将永远不会被阻塞，于是新一轮 select 又把case x := <-ch1这个分支选出并执行。由于 ch1 处于关闭状态，从这个 channel 获取数据，我们会得到这个 channel 对应类型的零值，这里就是 0。于是程序 再次输出 0；程序按这个逻辑循环执行，一直输出 0 值；
- **对一个 nil channel 执行获取操作，这个操作将阻塞**。于是，这 里已经被置为 nil 的 c1 或 c2 的分支，将再也不会被 select 选中执行。



### len(channel) 的应用

len 是 Go 语言的一个内置函数，它支持接收数组、切片、map、字符串和 channel 类型 的参数，并返回对应类型的“长度”，也就是一个整型值。

针对 channel ch 的类型不同，len(ch) 有如下两种语义： 

- 当 ch 为无缓冲 channel 时，len(ch) 总是返回 0； 
- 当 ch 为带缓冲 channel 时，**len(ch) 返回当前 channel ch 中尚未被读取的元素个数**。

这样一来，针对带缓冲 channel 的 len 调用似乎才是有意义的。那我们是否可以使用 len 函数来实现带缓冲 channel 的“判满”、“判有”和“判空”逻辑呢？

```go
var ch chan T = make(chan T, capacity)
// 判空
if len(ch) == 0 {
// 此时channel ch空了?
}
// 判有
if len(ch) > 0 {
// 此时channel ch中有数据?
}
// 判满
if len(ch) == cap(ch) {
// 此时channel ch满了?
}
```

你可以看到，我在上面代码注释的“空了”、“有数据”和“满了”的后面都打上了问 号。这是为什么呢？ 这是因为，channel 原语用于多个 Goroutine 间的通信，一旦多个 Goroutine 共同对 channel 进行收发操作，len(channel) 就会在多个 Goroutine 间形成“竞态”。**单纯地依 靠 len(channel) 来判断 channel 中元素状态，是不能保证在后续对 channel 的收发时 channel 状态是不变的。** 比如：

![image-20221125102135466](markdown%E5%9B%BE%E7%89%87/image-20221125102135466.png)





### 无缓冲channel

无缓冲 channel 兼具**通信和同步**特性

> **第一种用法：用作信号通知**

​	chan 类型有这样一个特点：chan 如果为空，那么，receiver **接收端**数据的时候就会**阻塞等 待**，直到 chan 被**关闭或者有新的数据**到来。利用这个机制，我们可以实现 wait/notify 的 设计模式。无缓冲 channel 用作信号传递的时候，有两种情况，分别是 1 对 1 通知信号和 1 对 n 通知信号

​	**把无缓冲的接受动作放在那个goroutine就是阻塞它，发送或者关闭操作就是一个信号**。

**1 对 1 通知信号例子**：

```
type signal struct{}

func worker() {
	fmt.Println("worker is working")
	time.Sleep(1 * time.Second)
}

func spawn(f func()) <-chan signal {
	c := make(chan signal)
	go func() {
		println("worker start to work...")
		f()
		c <- signal{}
	}()
	return c
}

func main() {
	fmt.Println("start a worker")
	ch := spawn(worker)
	<-ch
	fmt.Println("worker work done!")
}
```

在这个例子中，spawn 函数返回的 channel，被用于承载**新 Goroutine 退出的“通知信号”，这个信号专门用作通知 main goroutine。**main goroutine 在调用 spawn 函数后一直阻塞在对这个**“通知信号”**也就是c的接收动作上.

**空结构体内存开销为零，所以go日常开发用空结构体类型元素，作为一种”事件“进行goroutine间信息传递**



 **1 对 n 的信号通知** : 

常被用于协调多个 Goroutine 一起工作。**关闭一个无缓冲 channel 会让所有阻塞在这个 channel 上的接收操作返 回，从而实现了一种 1 对 n 的“广播”机制。** 而发送数据只会是一对一的信号交流。

```go
type signal struct{}

func worker(i int) {
	fmt.Printf("worker: %d:is working...\n", i)
	time.Sleep(1 * time.Second)
	fmt.Printf("worker: %d: works done\n", i)
}

func spawnGroup(f func(i int), num int, groupSignal <-chan signal) <-chan signal {
	var wg sync.WaitGroup
	for i := 0; i < num; i++ {
		wg.Add(1)
		go func(i int) {
			<-groupSignal
			fmt.Printf("worker %d: start to work...\n", i)
			f(i)
			wg.Done()
		}(i + 1)
	}
	ch := make(chan signal)
	go func() {
		wg.Wait()
		ch <- signal{}
	}()
	return ch
}

func main() {
	fmt.Println("start a group of workers....")
	groupSignal := make(chan signal)
	ch := spawnGroup(worker, 5, groupSignal)
	fmt.Println("the group of workers start to work")
	close(groupSignal)
	<-ch
	fmt.Println("the group of worker work done")
}
```

这个例子中实际上：

- ch作为一对一信号交流
- groupSignal作为一对多信号交流

**这里还需要注意的**就是wg.Wait()一定只能放在一个子进程中，否则`wg.Wait()阻塞。``ch := spawnGroup(worker, 5, groupSignal)`无返回值造成主进程休眠，进行`groupSignal`信号通知不到，这样就会造成所有goroutine睡眠死锁



> **第二种用法：用于替代锁机制**

无缓冲 channel 具有同步特性，这让它在某些场合可以替代锁，让我们的程序更加清晰，可读性也更好。我们可以对比下两个方案，直观地感受一下。

一个传统的、基于**“共享内存”+“互斥锁”**的 Goroutine 安全的计数器的实现：

```
type counter struct {
    sync.Mutex
    i int
}
func main() {
	var c counter
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(i int) {
			c.Lock()
			defer c.Unlock()
			c.i++
			fmt.Println(i, c.i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

在这个示例中，我们使用了一个带有互斥锁保护的全局变量作为计数器，所有要操作计数器的 Goroutine 共享这个全局变量，并在互斥锁的同步下对计数器进行自增操作。接下来我们再看更符合 Go 设计惯例的实现，也就是使用无缓冲 channel 替代锁后的实现：

```
type Counter struct {
    c chan int
    i int
}

func main() {
	var counter = &Counter{
		c: make(chan int),
	}
	go func() {
		for {
			counter.i = counter.i + 2
			counter.c <- counter.i
		}
	}()
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(i int) {
			counter.i = <-counter.c
			fmt.Println(i, counter.i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

在这个实现中，我们将计数器操作全部交给一个独立的 Goroutine 去处理，并通过**无缓冲 channel 的同步阻塞特性**，实现了计数器的控制。这样**其他 Goroutine 通过 Increase 函数试图增加计数器值的动作，实质上就转化为了一次无缓冲 channel 的接收动作**。

这种并发设计逻辑更符合 Go 语言所倡导的**“不要通过共享内存来通信，而是通过通信来共享内存”的原则。**



> **第三种用法：数据传递**

“击鼓传花”的游戏：花从一个人手中传给另外一个人，就有点类似**流水线** 的操作。

为了实现顺序的数据传递，我们可以定义一个令牌的变量，谁得到令牌，谁就可以打印一 次自己的编号，同时将令牌传递给下一个 goroutine，我们尝试使用 chan 来实现。

```go
type Token struct{}

func newWorker(id int, ch chan Token, nextCh chan Token) {
	for {
		token := <-ch         //取得令牌
		fmt.Println((id + 1)) //id从1开始
		nextCh <- token
	}
}

func main() {
	chs := []chan Token{make(chan Token), make(chan Token), make(chan Token), make(chan Token)}
	//创建4个worker
	for i := 0; i < 4; i++ {
		go newWorker(i, chs[i], chs[(i+1)%4])
	}
	//首先把令牌交给第一个worker
	chs[0] <- Token{}
	select {}
}
```

**这类场景有一个特点，就是当前持有数据的 goroutine 都有一个信箱，信箱使用 chan 实 现，goroutine 只需要关注自己的信箱中的数据，处理完毕后，就把结果发送到下一家的 信箱中。**



> **第四种 用法：任务编排**

这里的编排既**指安排 goroutine 按照指定的顺序执行，也指多个 chan 按照指定的方式组 合处理的方式**。

- goroutine 的编排类似“击鼓传花”的例子，我们通过编排数据在 chan 之间的流转，就可以控制 goroutine 的执行。
- 接下来，重点介绍下多个 chan 的编排 方式，总共 5 种，分别是 Or-Done 模式、扇入模式、扇出模式、Stream 和 MapReduce。

[14丨 Channel：透过代码看典型的应用模式.pdf](file:///E:/极客时间课程/2 高级Go语言工程师/2 Go并发编程实战/14丨 Channel：透过代码看典型的应用模式.pdf)



### 有缓冲channel的惯用法

带缓冲的 channel 与无缓冲的 channel 的最大不同之处，就在于它的**异步性**。

- 对一个带缓冲 channel，在缓冲区未满的情况下，对它进行发送操作的 Goroutine 不会阻塞挂起；
- 在缓冲区有数据的情况下，对它进行接收操作的 Goroutine 也不会阻塞挂起。

> **第一种用法：用作消息队列(消息交流)worker pool**

channel 的原生特性与我们认知中的消息队列十分相似，包括 Goroutine 安全、有 FIFO（first-in, first out）保证等。

- 从 channel 的内部实现看，它是以一个循环队列的方式存放数据，所以，它有时候也会被当成线程安全的队列和 buffer 使用。
- 一个goroutine 可以安全地往 Channel 中塞数据，另外一个 goroutine 可以安全地从 Channel 中读取数据，goroutine 就可以安全地实现信息交流了。

```go
func producer(intChan chan int) {
	for i := 0; i < cap(intChan); i++ {
		fmt.Println("生产者：", i)
		intChan <- i
	}
	close(intChan)
}
func consumer(intChan chan int, exitChan chan bool) {
	for {
		v, ok := <-intChan
		if ok {
			fmt.Println("消费者：", v)
		} else {
			break
		}
	}
	close(exitChan)
}
func main() {
	intChan := make(chan int, 10)
	exitChan := make(chan bool)
	go producer(intChan)
	go consumer(intChan, exitChan)
	<-exitChan
	fmt.Println("work done!")
}
```

综合前面这些结果数据，我们可以得出几个**初步结论**：

1. 无论是 1 收 1 发还是多收多发，带缓冲 channel 的收发性能都要好于无缓冲 channel；
2. 对于带缓冲 channel 而言，发送与接收的 Goroutine 数量越多，收发性能会有所下降；
3. 对于带缓冲 channel 而言，选择适当容量会在一定程度上提升收发性能。



**注意点**：Go 支持 channel 的初衷是将它作为 Goroutine 间的通信手段，它并不是专门用于消息队列场景的。如果你的项目需要专业消息队列的功能特性，比如支持优先级、支持权重、支持离线持久化等，那么 channel 就不合适了，可以使用第三方的专业的消息队列实现。



> **第二种用法：用作计数信号量**

Go 并发设计的一个惯用法，就是将带缓冲 channel 用作计数信号量（counting semaphore）。

- 带缓冲 channel 中的当前数据个数代表的是，当前同时处于活动状态（处理业务）的 Goroutine 的数量
- 带缓冲 channel 的容量（capacity），就代表了允许同时处于活动状态的 Goroutine 的最大数量。
- 向带缓冲 channel 的一个发送操作表示获取一个信号量
- 从 channel 的一个接收操作则表示释放一个信号量

```go
var active = make(chan struct{}, 3)
var jobs = make(chan int, 10)

func main() {
    go func() {
        for i := 0; i < 8; i++ {
            jobs <- (i + 1)
        }
        close(jobs)
    }()

    var wg sync.WaitGroup

    for j := range jobs {
        wg.Add(1)
        go func(j int) {
            active <- struct{}{}
            log.Printf("handle job: %d\n", j)
            time.Sleep(2 * time.Second)
            <-active
            wg.Done()
        }(j)
    }
    wg.Wait()
}
```

这个示例创建了一组 Goroutine 来处理 job，同一时间允许最多 3 个 Goroutine 处于活动状态。为了达成这一目标，我们看到这个示例使用了一个容量（capacity）为 3 的带缓冲 channel: active 作为计数信号量，这意味着允许同时处于活动状态的最大 Goroutine 数量为 3。

从示例运行结果中的时间戳中，我们可以看到，虽然我们创建了很多 Goroutine，但由于计数信号量的存在，同一时间内处于活动状态（正在处理 job）的 Goroutine 的数量最多为 3 个。



### 与select结合使用的一些惯用法

**channel 和 select 的结合使用能形成强大的表达能力**

> **第一种用法：利用 default 分支避免阻塞**

select 语句的 default 分支的语义，就是在其他非 default 分支因通信未就绪，而无法被选择的时候执行的，这就给 **default 分支赋予了一种“避免阻塞”的特性**。

```go
func tryRecv(c <-chan int) (int, bool) {
  select {
  case i := <-c:
    return i, true

  default: // channel为空
    return 0, false
  }
}

func trySend(c chan<- int, i int) bool {
  select {
  case c <- i:
    return true
  default: // channel满了
    return false
  }
}
```



> **第二种用法：实现超时机制**

带超时机制的 select，是 Go 中常见的一种 select 和 channel 的组合用法。通过超时事件，我们既可以避免长期陷入某种操作的等待中，也可以做一些异常处理工作。比如，下面示例代码实现了一次具有 30s 超时的 select：

```go
func worker() {
  deadline :=  heartbeat := time.After(30 * time.Second)
  defer deadline.Stop()
  select {
  case <-c:
       // ... do some stuff
  case <-deadline:
      return
  }
}
```

不过，在应用带有超时机制的 select 时，我们要特别注意 timer 使用后的释放，尤其在大量创建 timer 的时候。Go 语言标准库提供的 timer 实际上是由 Go 运行时自行维护的，而不是操作系统级的定时器资源，它的使用代价要比操作系统级的低许多。但即便如此，作为 time.Timer 的使用者，我们也要尽量减少在使用 Timer 时给 Go 运行时和 Go 垃圾回收带来的压力，要及时调用 timer 的 Stop 方法回收 Timer 资源。



> **第三种用法：实现心跳机制**

结合 time 包的 Ticker，我们可以实现带有心跳机制的 select。这种机制让我们可以在监听 channel 的同时，执行一些周期性的任务，比如下面这段代码：

```go
func worker() {
  heartbeat := time.NewTicker(30 * time.Second)
  defer heartbeat.Stop()
  for {
    select {
    case <-c:
      // ... do some stuff
    case <- heartbeat.C:
      //... do heartbeat stuff
    }
  }
}
```

这里我们使用 time.NewTicker，创建了一个 Ticker 类型实例 heartbeat。这个实例包含一个 channel 类型的字段 C，这个字段会按一定时间间隔持续产生事件，就像“心跳”一样。这样 for 循环在 channel c 无数据接收时，会每隔特定时间完成一次迭代，然后回到 for 循环进行下一次迭代。和 timer 一样，我们在使用完 ticker 之后，也不要忘记调用它的 Stop 方法，避免心跳事件在 ticker 的 channel（上面示例中的 heartbeat.C）中持续产生。



#  内存模型：Go如何保证并发读写的顺序

这里说的内存模型它并不 是指 Go 对象的内存分配、内存回收和内存整理的规范，**它描述的是并发环境中多 goroutine 读相同变量的时候，变量的可见性条件**。具体点说，就是指，在什么条件下， goroutine 在读取一个变量的值的时候，能够看到其它 goroutine 对这个变量进行的写的 结果。

## 重排和可见性的问题

**由于指令重排，代码并不一定会按照你写的顺序执行。**

比如：

```go
var a, b int
func f() {
    a = 1 // w之前的写操作
    b = 2 // 写操作w
}
func g() {
    print(b) // 读操作r
    print(a) // ???
}
func main() {
    go f() //g1
    g() //g2
}
```

第 9 行是要打印 b 的值。需要注意的是，即使这里打印出的值是 2，但是依然 可能在打印 a 的值时，打印出初始值 0，而不是 1。这是因为，程序运行的时候，**不能保 证 g2 看到的 a 和 b 的赋值有先后关系**。



## happen-before

**非常重要的保证！！！！！！！** **对于单个的 goroutine 来说，它有一个特殊的 happens-before 关系，Go 内存模型中这么说的：程序的执行顺序和它们的代码指定的顺序是一样的，即使编译器 或者 CPU 重排了读写顺序，从行为上来看，也和代码指定的顺序一样。**

但是，对于**另一个 goroutine 来说，重排却会产生非常大的影响**。因为 **Go 只保证 goroutine 内部重排对读写的顺序没有影响**，比如刚刚我们在讲“可见性”问题时提到的 三个例子，那该怎么办呢？这就要用到 happens-before 关系了。



**Go 内存模型**通过 happens-before 定义两个事件（读、写 action）的顺序：

- 如果事件 e1  happens before 事件 e2，那么，我们就可以说事件 e2 在事件 e1 **之后发生**（happens after）。
- 如果 e1 不是 happens before e2， 同时也不 happens after e2，那么，我们 就可以说事件 e1 和 e2 是**同时发生**的。



如果要保证对“变量 v 的读操作 r”能够观察到一个对“变量 v 的写操作 w”，并且 r 只 能观察到 w 对变量 v 的写，没有其它对 v 的写操作，也就是说，我们要保证 r 绝对能观察 到 w 操作的结果，那么就需要同时满足两个条件：

- w happens before r； 
- 其它对 v 的写操作（w2、w3、w4, …） 要么 happens before w，要么 happens after r，绝对不会和 w、r 同时发生，或者是在它们之间发生。



**三个 Go 语言中和内存模型有关的小知识**：

- 在 Go 语言中，对变量进行零值的初始化就是一个写操作。
-  如果对超过机器 word（64bit、32bit 或者其它）大小的值进行读写，那么，就可以看 作是对拆成 word 大小的几个读写无序进行。 
- Go 并不提供直接的 CPU 屏障（CPU fence）来提示编译器或者 CPU 保证顺序性，而 是使用不同架构的内存屏障指令来实现统一的并发原语。



## Go 语言中保证的 happens-before 关系

除了单个 goroutine 内部提供的 happens-before 保证，Go 语言中还提供了一些其它的 happens-before 关系的保证

""**你通过学习这节课来理解你的程序的行为是聪明的，但是， 不要自作聪明。**""比如，Go 里面已经有值得信赖的互斥锁了，如果没有额外的需求，就不要使用 Channel 创造出自己的互斥锁。

> **init函数**

应用程序的初始化是在单一的 goroutine 执行的。如果包 p 导入了包 q，那么，q 的 init 函数的执行一定 happens before p 的**任何初始化代码(包括p的init函数)**。

- **main 函数一定在导入的包的 init 函数之后执行。**
- 同一个包下可以有多个 init 函数，但是每个文件最多只能有一个 init 函数，多个 init 函数 按照它们的文件名顺序逐个初始化。



> **goroutine**

明确一个规则：**启动 goroutine 的 go 语句的执行，一定 happens before 此 goroutine 内的代码执行。**

```go
var a string
func f() {
	print(a)
}
func hello() {
	a = "hello, world"
	go f()
}
//肯定打印hello,world
```

goroutine 退出的时候，是没有任何 happens before 保证的。所以，如果你想观察某个 goroutine 的执行效果，你需要使用同步机制建 立 happens-before 关系，比如 Mutex 或者 Channel。



> **Channel**

**第 1 条规则：**往 Channel 中的发送操作，happens before 从该 Channel 接收相应数 据的动作完成之前，即第 n 个 send 一定 happens before 第 n 个 receive 的完成。

**第 2 条规则**：close 一个 Channel 的调用，肯定 happens before 从关闭的 Channel 中读取出一个零值。

**第 3 条规则**：对于 unbuffered 的 Channel，也就是容量是 0 的 Channel，从此 Channel 中读取数据的调用一定 happens before 往此 Channel 发送数据的调用完成。

**第 4 条规则**：如果 Channel 的容量是 m（m>0），那么，第 n 个 receive 一定 happens before 第 n+m 个 send 的完成。



> **Mutex/RWMutex**

**第一条：** 第 n 次的 m.Unlock 一定 happens before 第 n+1 m.Lock 方法的返回；

**第二条：**读写锁的 Lock 必须等待既有的读锁释放后才能获 取到。



> **WaitGroup**

**Wait 方法等到计数值归零之后才返回。**



> **Once**

它提供的保证是：**对于 once.Do(f) 调用，f 函数的那个单次调用一定 happens before 任何 once.Do(f) 调用的 返回**。换句话说，就是函数 f 一定会在 Do 方法返回之前执行。



# 基本并发原语



#### 5.1 引言

在前面的讲解中，我们学习了 Go 的并发实现方案，知道了 Go 基于 Tony Hoare 的 CSP 并发模型理论，实现了 Goroutine、channel 等并发原语。

并且，Go 语言之父 Rob Pike 还有一句经典名言：“不要通过共享内存来通信，**应该通过通信来共享内存**（Don’t communicate by sharing memory, share memory by communicating）”，这就奠定了 Go 应用并发设计的主流风格：**使用 channel 进行不同 Goroutine 间的通信。**

不过，Go 也并没有彻底放弃基于共享内存的并发模型，而是在提供 CSP 并发模型原语的同时，还通过标准库的 sync 包，提供了针对传统的、基于共享内存并发模型的低级同步原语，包括：互斥锁（sync.Mutex）、读写锁（sync.RWMutex）、条件变量（sync.Cond）等，并通过 atomic 包提供了原子操作原语等等。显然，基于共享内存的并发模型在 Go 语言中依然有它的“用武之地”。接下来，就围绕 sync 包中的几个同步结构与对应的方法，聊聊**基于共享内存的并发模型**在 Go 中的应用。



#### 5.2 sync 包同步原语和channel的抉择

一般情况下，我建议你优先使用 CSP 并发模型进行并发程序设计。但是在下面一些场景中，我们依然需要 sync 包提供的低级同步原语

- **共享资源：**1.并发读写共享临界区资源，会出现数据竞争（data race)，2.还有保证结构体内部状态数据的同步访问的场景。所以需要Mutex,RWMutex来保护。（sync.Mutex实现的同步机制的性能，都要比 channel 实现的高出三倍多。channel 并发原语也可以用于对数据对象访问的同步，我们可以把 channel 看成是一种高级的同步原语，它自身的实现也是建构在低级同步原语之上的。也正因为如此，channel 自身的性能与低级同步原语相比要略微逊色，开销要更大。）
- **任务编排**：需要goroutine按照一定的规律执行，而goroutine之间又相互等待或者依赖的顺序关系，Channel来实现。
- **消息传递和消息通知机制：**信息交流以及不同的goroutine之间的线程安全的数据交流，常用Channel来实现。除非只想 signal 一个 goroutine，才使用 Cond；
- 简单等待任务完成：用 WaitGroup，也有 Channel 的推崇者用 Channel，都可以；
- 和select结合：channel
- 和超时配合：channel和Context



#### 5.3 sync 包中同步原语使用的注意事项

如果对使用过的、sync 包中的类型的示例进行复制，并使用了复制后得到的副本，将导致不可预期的结果。所以，在使用 sync 包中的类型的时候，**我们推荐通过闭包方式，或者是传递类型实例（或包裹该类型的类型实例）的地址（指针）的方式进行**。这就是使用 sync 包时最值得我们注意的事项。

```
import (
	"fmt"
	"sync"
)

type Counter struct {
	sync.Mutex
	Count uint64
}

func main() {
	var counter Counter

	(&counter).Lock()
	defer (&counter).Unlock()
	(&counter).Count++
	fmt.Println("over")
}
```





#### 5.4  Mutex

常见的并发问题：计数器，秒杀系统，往同一个buffer中写入数据，用互斥锁来解决。在Go语言里就是Mutex

##### 实现机制

**临界区：**在并发编程中，如果程序中的 一部分会被并发访问或修改，为了避免并发访问导致意想不到的结果，这部分程序需要被保护起来，这部分程序就叫做临界区。可以说临界区就是一个被共享的资源，或者说是一个整体的一组共享资源。

如果很多线程同步访问临界区，就会造成访问或操作错误，我们可以**使用互斥锁，限定临界区只能同时由一个线程持有**

![image-20220228160258309](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220228160258309.png?lastModify=1668648666)

Mutex是使用最广泛的同步原语，也叫并发原语。并发原语指代范围更大，还包括任务编排类型。

##### 基本用法

在了解Mutex用法之前，先了解下：Locker接口。

Locker接口定义了锁同步原语的方法集：

```
type Locker interface{
	Lock()
	Unlock()
}
```

可以看到Go定义锁接口的方法集很简单，就是请求锁和释放锁两个方法，秉承了简洁风格。但这个接口实际项目应用的不多。一般使用具体的同步原语。

Mutex：互斥锁Mutex提供Lock和Unlock方法：进入临界区之前调用Lock方法，退出临界区调用Unlock方法。**当一个goroutine通过调用Lock方法获得了这个锁的拥有权后，其他请求锁的goroutine就会阻塞在Lock方法的调用上，直到锁被释放并且自己获得这个锁的拥有权。**

```
func(m *Mutex)Lock()
func(m *Mutex)Unlock()
```

为啥一定要加锁呢？来看一个并发访问场景中不使用锁的例子，看看实现起来会出现什么状况。在这个例子中，我们创建了 10 个 goroutine，同时不断地对一个变量（count）进行加 1 操作，每个 goroutine 负责执行 10 万次的加 1 操作，我们期望的最后计数的结果是 10 * 100000 = 1000000 (一百万)

```
import (
	"fmt"
	"sync"
)

func main() {
	var count = 0
	//使用waitgroup等待10个协程完成
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			//对变量执行10万次加1
			for j := 0; j < 100000; j++ {
				count++
			}

		}()
	}
	//等待10个协程完成
	wg.Wait()
	fmt.Println(count)
}
```

每次运行，你都可能得到不同的结果，基本上不会得到理想中的一百万的结果。这是因为，count++ 不是一个原子操作，它至少包含几个步骤，比如读取变量 count 的当前值，对这个值加 1，把结果再保存到 count 中。**因为不是原子操作，就可能有并发的问题。**

针对并发问题隐藏深的：Go 提供了一个**检测并发访问共享资源是否有问题的工具**： race detector，它可以帮助我们自动发现程序有没有 data race 的问题。

这个工具怎么用？在编译（compile）、测试（test）或者运行（run）Go 代码的时候，加上 race 参数，就有可能发现并发问题。比如在上面的例子中，我们可以加上 race 参数运行，检测一下是不是有并发问题。如果你 go run -race test.go，就会输出警告信息。这个警告不但会告诉你有并发问题，而且还会告诉你哪个 goroutine 在哪一行对哪个变量有写操作，同时，哪个 goroutine 在哪一行对哪个变量有读操作，就是这些并发的读写访问，引起了 data race。

虽然这个工具使用起来很方便，但是，因为它的实现方式，只能通过真正对实际地址进行读写访问的时候才能探测，所以它并不能在编译的时候发现 data race 的问题。而且，在运行的时候，只有在触发了 data race 之后，才能检测到，如果碰巧没有触发（比如一个 data race 问题只能在 2 月 14 号零点或者 11 月 11 号零点才出现），是检测不出来的。

**主角 Mutex** 就要登场了，它可以轻松地消除掉 data race：

这里的共享资源是 count 变量，临界区是 count++，只要在临界区前面获取锁，在离开临界区的时候释放锁，就能完美地解决 data race 的问题了

```
import (
	"fmt"
	"sync"
)

func main() {
	//互斥锁保护器
	var m sync.Mutex
	var count = 0
	//使用waitgroup等待10个协程完成
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			//对变量执行10万次加1
			for j := 0; j < 100000; j++ {
				m.Lock()
				count++
				m.Unlock()
			}

		}()
	}
	//等待10个协程完成
	wg.Wait()
	fmt.Println(count)
}
```

这里有**两点需要注意**：

- Mutex 的零值是还没有 goroutine 等待的未加锁的状态，所以你不需要额外的初始化，直接声明变量（如 var mu sync.Mutex）即可。零值可用类型。
- 如果 Mutex 已经被一个 goroutine 获取了锁，其它等待中的 goroutine 们只能一直等待。那么，等这个锁释放后，等待中的 goroutine 中哪一个会优先获取 Mutex 呢？等待的goroutine们以FIFO排队！！！

**使用互斥锁的两个原则**：

- **尽量减少在锁中的操作。**这可以减少其他因 Goroutine 阻塞而带来的损耗与延迟。
- **一定要记得调用 Unlock 解锁**。忘记解锁会导致程序局部死锁，甚至是整个程序死锁，会导致严重的后果。同时，我们也可以结合学习到的 defer，优雅地执行解锁操作。



##### 其他用法

- **嵌入到其他struct中使用**

```
type Counter struct {
    m sync.Mutex
    Count uint64
}
```

在初始化这个struct，也不必初始化Mutex字段。不会报错

- **采用嵌入字段方式，直接在这个struct上直接调用Lock/Unlock方法。**

```
type Counter struct {
	m     sync.Mutex
	Count uint64
}

func main() {
	var counter Counter
	//使用waitgroup等待10个协程完成
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			//对变量执行10万次加1
			for j := 0; j < 100000; j++ {
				counter.m.Lock()
				counter.Count++
				counter.m.Unlock()
			}

		}()
	}
	//等待10个协程完成
	wg.Wait()
	fmt.Println(counter.Count)
}
```

- **把获取锁、释放锁、计数加一的逻辑封装成一个方法**

```
//计数器类型
type Counter struct {
	m           sync.Mutex
	Count       uint64
	Name        string
	CounterType int
}
//得到计数器值
func (c *Counter) Incr() {
	c.m.Lock()
	c.Count++
	c.m.Unlock()
}
func (c *Counter) count() uint64 {
	c.m.Lock()
	defer c.m.Unlock()
	return c.Count
}

func main() {
	//封装好的计数器
	var counter Counter
	//使用waitgroup等待10个协程完成
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			//对变量执行10万次加1
			for j := 0; j < 100000; j++ {
				counter.Incr() //受到锁保护的方法
			}

		}()
	}
	//等待10个协程完成
	wg.Wait()
	fmt.Println(counter.count())
}
```



##### 演变历程

![image-20220228200309735](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220228200309735.png?lastModify=1668648666)

**Mutex 绝不容忍一个 goroutine 被落下，永远没有机会获取锁。不抛弃不放弃是它的宗旨，而且它也尽可能地让等待较长的 goroutine 更有机会获取到锁**

Mutex 可能处于两种操作模式下：**正常模式和饥饿模式**

正常模式下，waiter 都是进入先入先出队列，被唤醒的 waiter 并不会直接持有锁，而是要和新来的 goroutine 进行竞争。新来的 goroutine 有先天的优势，它们正在 CPU 中运行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。在饥饿模式下，Mutex 的拥有者将直接把锁交给队列最前面的 waiter。新来的 goroutine 不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会 spin，它会乖乖地加入到等待队列的尾部。如果拥有 Mutex 的 waiter 发现下面两种情况的其中之一，它就会把这个 Mutex 转换成正常模式:此 waiter 已经是队列中的最后一个 waiter 了，没有其它的等待锁的 goroutine 了；此 waiter 的等待时间小于 1 毫秒。正常模式拥有更好的性能，因为即使有等待抢锁的 waiter，goroutine 也可以连续多次获取到锁。饥饿模式是对公平性和性能的一种平衡，它避免了某些 goroutine 长时间的等待锁。在饥饿模式下，优先对待的是那些一直在等待的 waiter。



##### 4大易错场景



1. **Lock/Unlock不成对出现**

Lock/Unlock 没有成对出现，就意味着会出现死锁的情况，或者是因为 Unlock 一个未加锁的 Mutex 而导致 panic。

缺少 Unlock 的场景，常见的有三种情况：

- 代码中有太多的 if-else 分支，可能在某个分支中漏写了 Unlock；
- 在重构的时候把 Unlock 给删除了；
- Unlock 误写成了 Lock

缺少 Lock 的场景：

- 误操作删除了 Lock



1. **Copy 已使用的 Mutex**

   在使用 sync 包中的并发原语的时候，**我们推荐通过闭包方式，或者是传递类型实例（或包裹该类型的类型实例）的地址（指针）的方式进行**。sync 的同步原语在使用后是不能复制的，Mutex 是最常用的一个同步原语，那它也是不能复制的,因为拷贝后两变量互相独立，如果在lock的状态就复制就会造成不可预期结果。

   原因在于，Mutex 是一个有状态的对象，它的 state 字段记录这个锁的状态。如果你要复制一个已经加锁的 Mutex 给一个新的变量，那么新的刚初始化的变量居然被加锁了，这显然不符合你的期望，因为你期望的是一个零值的 Mutex。关键是在并发环境下，你根本不知道要复制的 Mutex 状态是什么，因为要复制的 Mutex 是由其它 goroutine 并发访问的，状态可能总是在变化。

   ```
   import (
   	"fmt"
   	"sync"
   )
   
   type Counter struct {
   	sync.Mutex
   	Count uint64
   }
   
   func main() {
   	var counter Counter
   
   	counter.Lock()
   	defer counter.Unlock()
   	counter.Count++
   	foo(counter)
   
   }
   func foo(c Counter) {
   	c.Lock()
   	defer c.Unlock()
   	fmt.Println("in foo")
   }
   ```

   在调用 foo 函数的时候，调用者会复制 Mutex 变量 c 作为 foo 函数的参数，不幸的是，复制之前已经使用了这个锁，这就导致，复制的 Counter 是一个带状态 Counter。

   怎么办呢？Go 在运行时，有死锁的检查机制（checkdead() 方法），它能够发现死锁的 goroutine。这个例子中因为复制了一个使用了的 Mutex，导致锁无法使用，程序处于死锁的状态。程序运行的时候，死锁检查机制能够发现这种死锁情况并输出错误信息，如下图中错误信息以及错误堆栈：

   ![image-20220228203933852](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220228203933852.png?lastModify=1668648666)

   肯定不想运行的时候才发现这个因为复制 Mutex 导致的死锁问题，那么你怎么能够及时发现问题呢？可以使用 vet 工具，把检查写在 Makefile 文件中，在持续集成的时候跑一跑，这样可以及时发现问题，及时修复。我们可以使用 go vet 检查这个 Go 文件：

![image-20220228204059970](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220228204059970.png?lastModify=1668648666)

​		使用这个工具就可以发现 Mutex 复制的问题，错误信息显示得很清楚，是在调用 foo 函数的时候发生了 lock value 复制的情况，还		告诉我们出问题的代码行数以及 copy lock 导致的错误。



1. **重入**

先了解下**可重入锁**：当一个线程获取锁时，如果没有其它线程拥有这个锁，那么，这个线程就成功获取到这个锁。之后，如果其它线程再请求这个锁，就会处于阻塞等待的状态。但是，如果拥有这把锁的线程再请求这把锁的话，不会阻塞，而是成功返回，所以叫可重入锁（有时候也叫做递归锁）。只要你拥有这把锁，你可以可着劲儿地调用，比如通过递归实现一些算法，调用者不会阻塞或者死锁。

画重点：**Mutex不是可重入的锁**。因为 Mutex 的实现中没有记录哪个 goroutine 拥有这把锁。旦误用 Mutex 的重入，就会导致报错，如下：

```
func foo(l sync.Locker) {
	fmt.Println("in foo")
	l.Lock()
	defer l.Unlock()
	bar(l)
}
func bar(l sync.Locker) {
	l.Lock()
	defer l.Unlock()
	fmt.Println("in bar")
}
func main() {
	l := &sync.Mutex{}
	foo(l)
}
```

![image-20220301145342719](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220301145342719.png?lastModify=1668648666)

写完这个 Mutex 重入的例子后，运行一下，你会发现类似下面的错误。程序一直在请求锁，但是一直没有办法获取到锁，结果就是 Go 运行时发现死锁了，没有其它地方能够释放锁让程序运行下去。

**那么如何实现可重入锁呢？**

虽然标准库 Mutex 不是可重入锁，但是想要实现一个可重入锁，可以吗？可以，自己实现一个。这里的关键就是，实现的锁要能记住当前是哪个 goroutine 持有这个锁。我来提供两个方案。

- 方案一：通过 hacker 的方式获取到 goroutine id，记录下获取锁的 goroutine id，它可以实现 Locker 接口。
- 方案二：调用 Lock/Unlock 方法时，由 goroutine 提供一个 token，用来标识它自己，而不是我们通过 hacker 的方式获取到 goroutine id，但是，这样一来，就不满足 Locker 接口了。

**可重入锁（递归锁）解决了代码重入或者递归调用带来的死锁问题，同时它也带来了另一个好处，就是我们可以要求，只有持有锁的 goroutine 才能 unlock 这个锁。这也很容易实现，因为在上面这两个方案中，都已经记录了是哪一个 goroutine 持有这个锁。**

**方案一：goroutine id**

第一步获取goroutine id有两种方式，分别为简单方式和hacker方式。

简单方式，就是通过 runtime.Stack 方法获取栈帧信息，栈帧信息里包含 goroutine id。你可以看看上面 panic 时候的贴图，goroutine id 明明白白地显示在那里。![image-20220301150414663](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220301150414663.png?lastModify=1668648666)

我们只要解析出这个id就行，解析的方法采用下面的代码；

```
func GoID() int64 {
	var buf [64]byte
	//获得当前goroutine信息
	n := runtime.Stack(buf[:], false)
	idField := bytes.TrimPrefix(buf[:n], []byte("goroutine "))
	idField = idField[:bytes.IndexByte(idField, ' ')]
	id, _ := strconv.ParseInt(string(idField), 10, 64)
	return id
}
```



hacker方式：也是方案一采用的方式

首先，我们获取运行时的 g 指针，反解出对应的 g 的结构。每个运行的 goroutine 结构的 g 指针保存在当前 goroutine 的一个叫做 TLS 对象中。

第一步：我们先获取到 TLS 对象；

第二步：再从 TLS 中获取 goroutine 结构的 g 指针；

第三步：再从 g 指针中取出 goroutine id。

以上步骤可以使用第三方的库petermattis/goid来获取 goroutine id，如下：

```
gid := goid.Get()
```



实现可重入锁：

```
type RecursiveMutex struct {
    sync.Mutex
    owner     int64 //当前持有锁的goroutine id
    recursion int32 //这个goroutine重入次数
}

func (m *RecursiveMutex) Lock() {
    gid := goid.Get()
    //如果当前持有锁的goroutine就是这次调用的goroutine，说明是重入
    if atomic.LoadInt64(&m.owner) == gid {
        m.recursion++
        return
    }
    m.Mutex.Lock()
    //获得锁的goroutien第一次调用，记录下他的id,调用次数加1
    atomic.StoreInt64(&m.owner, gid)
    m.recursion = 1
}

func (m *RecursiveMutex) Unlock() {
    gid := goid.Get()
    //非持有锁的goroutine尝试释放锁，错误地使用
    if atomic.LoadInt64(&m.owner) != gid {
        panic(fmt.Sprintf("wrong the owner(%d):%d!", m.owner, gid))
    }
    //调用次数-1
    m.recursion--
    if m.recursion != 0 { //这个goroutine还没完全释放
        return
    }
    //此goroutine最后一次调用，需要释放锁
    atomic.StoreInt64(&m.owner, -1)
    m.Mutex.Unlock()
}
func foo(l sync.Locker) {
	fmt.Println("in foo")
	l.Lock()
	defer l.Unlock()
	bar(l)
}
func bar(l sync.Locker) {
	l.Lock()
	defer l.Unlock()
	fmt.Println("in bar")
}
func main() {
	l := &RecursiveMutex{}
	foo(l)
}
```



**方案二：token**



方案一是用 goroutine id 做 goroutine 的标识，我们也可以让 goroutine 自己来提供标识。不管怎么说，Go 开发者不期望你利用 goroutine id 做一些不确定的东西，所以，他们没有暴露获取 goroutine id 的方法。下面的代码是第二种方案。调用者自己提供一个 token，获取锁的时候把这个 token 传入，释放锁的时候也需要把这个 token 传入。通过用户传入的 token 替换方案一中 goroutine id，其它逻辑和方案一一致。

```
type TokenRecursiveMutex struct {
	sync.Mutex
	token     int64
	recursion int32
}

func (m *TokenRecursiveMutex) Lock(token int64) {

	//如果当前持有锁的token就是这次调用的token，说明是重入
	if atomic.LoadInt64(&m.token) == token {
		m.recursion++
		return
	}
	m.Mutex.Lock() //传入token不一致，说明不是递归调用
	//获得锁的goroutien第一次调用，记录下他的token,调用次数加1
	atomic.StoreInt64(&m.token, token)
	m.recursion = 1
}

func (m *TokenRecursiveMutex) Unlock(token int64) {

	//非持有锁的goroutine尝试释放锁，错误地使用
	if atomic.LoadInt64(&m.token) != token {
		panic(fmt.Sprintf("wrong the owner(%d):%d!", m.token, token))
	}
	//当前持有锁的token调用次数-1
	m.recursion--
	if m.recursion != 0 { //这个goroutine还没完全释放
		return
	}
	//此goroutine最后一次调用，需要释放锁
	atomic.StoreInt64(&m.token, 0)
	m.Mutex.Unlock()
}
func foo(l *TokenRecursiveMutex) {
	fmt.Println("in foo")
	l.Lock(1)
	defer l.Unlock(1)
	bar(l)
}
func bar(l *TokenRecursiveMutex) {
	l.Lock(1)
	defer l.Unlock(1)
	fmt.Println("in bar")
}
func main() {
	l := &TokenRecursiveMutex{}
	foo(l)
}
```



1. **死锁**

先认识下死锁：两个或两个以上的进程（或线程，goroutine）在执行过程中，因争夺共享资源而处于一种互相等待的状态，如果没有外部干涉，它们都将无法推进下去，此时，我们称系统处于死锁状态或系统产生了死锁。

分析一下死锁产生的必要条件。如果你想避免死锁，只要破坏这四个条件中的一个或者几个，就可以了：

- ​		**互斥：** 至少一个资源是被排他性独享的，其他线程必须处于等待状态，直到资源被释放。
- **持有和等待：**goroutine 持有一个资源，并且还在请求其它 goroutine 持有的资源，也就是咱们常说的“吃着碗里，看着锅里”的意思。
- **不可剥夺：**资源只能由持有它的 goroutine 来释放。
- **环路等待：**一般来说，存在一组等待进程，P={P1，P2，…，PN}，P1 等待 P2 持有的资源，P2 等待 P3 持有的资源，依此类推，最后是 PN 等待 P1 持有的资源，这就形成了一个环路等待的死结。![image-20220301164120038](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220301164120038.png?lastModify=1668648666)

**经典死锁问题：**五个哲学家共用一张圆桌，分别坐在周围的五张椅子上，在圆桌上有五个碗和五只筷子，他们的生活方式是交替的进行思考和进餐。平时，一个哲学家进行思考，饥饿时便试图取用其左右最靠近他的筷子，只有在他拿到两只筷子时才能进餐。进餐完毕，放下筷子继续思考。

**简单一点的问题：**有一次我去派出所开证明，派出所要求物业先证明我是本物业的业主，但是，物业要我提供派出所的证明，才能给我开物业证明，结果就陷入了死锁状态。你可以把派出所和物业看成两个 goroutine，派出所证明和物业证明是两个资源，双方都持有自己的资源而要求对方的资源，而且自己的资源自己持有，不可剥夺。

这是一个最简单的只有两个 goroutine 相互等待的死锁的例子，转化成代码如下：

```
func main() {
	var (
		policeCertificate  sync.Mutex
		propertyCertificat sync.Mutex
	)
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		policeCertificate.Lock()
		defer policeCertificate.Unlock()
		time.Sleep(5 * time.Second)
		propertyCertificat.Lock()
		propertyCertificat.Unlock()
	}()
	go func() {
		defer wg.Done()
		propertyCertificat.Lock()
		defer propertyCertificat.Unlock()
		time.Sleep(5 * time.Second)
		policeCertificate.Lock()
		policeCertificate.Unlock()
	}()
	wg.Wait()
	fmt.Println("successs!!!")
}
```

![image-20220301170155465](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220301170155465.png?lastModify=1668648666)

**怎么解决？**

- Go 运行时，有死锁探测的功能，能够检查出是否出现了死锁的情况，如果出现了，这个时候你就需要调整策略来处理了。
- 你可以引入一个第三方的锁，大家都依赖这个锁进行业务处理，比如现在政府推行的一站式政务服务中心。或者是解决持有等待问题，物业不需要看到派出所的证明才给开物业证明，等等。



##### 扩展Mutex功能

1. **基于Mutex实现TryLock： 不希望追求某个锁的goroutine一直等待**

为 Mutex 添加一个 TryLock 的方法，也就是尝试获取排外锁。这里的逻辑：当一个 goroutine 调用这个 TryLock 方法请求锁的时候，如果这把锁没有被其他 goroutine 所持有，那么，这个 goroutine 就持有了这把锁，并返回 true；如果这把锁已经被其他 goroutine 所持有，或者是正在准备交给某个被唤醒的 goroutine，那么，这个请求锁的 goroutine 就直接返回 false，不会阻塞在方法调用上。如下图所示，如果 Mutex 已经被一个 goroutine 持有，调用 Lock 的 goroutine 阻塞排队等待，调用 TryLock 的 goroutine 直接得到一个 false 返回。![image-20220302103200177](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302103200177.png?lastModify=1668648666)

**在实际开发中**，如果要更新配置数据，我们通常需要加锁，这样可以避免同时有多个 goroutine 并发修改数据。有的时候，我们也会使用 TryLock。这样一来，当某个 goroutine 想要更改配置数据时，如果发现已经有 goroutine 在更改了，其他的 goroutine 调用 TryLock，返回了 false，这个 goroutine 就会放弃更改。

```
const (
	mutexLocked = 1 << iota // 1 0001 含义：用最后一位表示当前对象锁的状态，0-未锁住 1-已锁住

    mutexWoken // 2 0010 含义：用倒数第二位表示当前对象是否被唤醒 0-唤醒 1-未唤醒

    mutexStarving // 4 0100 含义：用倒数第三位表示当前对象是否为饥饿模式，0为正常模式，1为饥饿模式。

    mutexWaiterShift = iota // 3，从倒数第四位往前的bit位表示在排队等待的goroutine数
)

type Mutex struct {
	sync.Mutex
}

func (m *Mutex) Trylock() bool {
	//如果能成功抢到锁
	if atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), 0, mutexLocked) {
		return true
	}
	//如果处于唤醒，加锁或者饥饿状态，不参与竞争，返回false
	old := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
	if old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
		return false
	}
	//尝试在竞争状态下请求锁
	new := old | mutexLocked
	return atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), old, new)
}
```

写一个简单的测试程序，来测试我们的 TryLock 的机制是否工作。这个测试程序的工作机制是这样子的：程序运行时会启动一个 goroutine 持有这把我们自己实现的锁，经过随机的时间才释放。主 goroutine 会尝试获取这把锁。如果前一个 goroutine 一秒内释放了这把锁，那么，主 goroutine 就有可能获取到这把锁了，输出“got the lock”，否则没有获取到也不会被阻塞，会直接输出“can't get the lock”。

```
func main() {
	var m Mutex
	go func() {
		m.Lock()
		defer m.Unlock()
		time.Sleep(time.Duration(1 * time.Second))
	}()
	time.Sleep(time.Second)
	if m.Trylock() {
		fmt.Println("got the lock")
		m.Unlock()
		return
	}
	fmt.Println("can not get th lock")
}
```

![image-20220302105808293](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302105808293.png?lastModify=1668648666)



1. **通过unsafe字段读取到Mutex内部的state字段：监控锁**

首先来认识下Mutex结构：

```
type Mutex struct {
    state int32
    sema uint32
}
```

Mutex 结构中的 state 字段有很多个含义，通过 state 字段，你可以知道锁是否已经被某个 goroutine 持有、当前是否处于饥饿状态、是否有等待的 goroutine 被唤醒、等待者的数量等信息。但是，state 这个字段并没有暴露出来，所以，我们需要想办法获取到这个字段，并进行解析。

通过unsafe方式：

```
const (
	mutexLocked = 1 << iota // 1 0001 含义：用最后一位表示当前对象锁的状态，0-未锁住 1-已锁住

    mutexWoken // 2 0010 含义：用倒数第二位表示当前对象是否被唤醒 0-唤醒 1-未唤醒

    mutexStarving // 4 0100 含义：用倒数第三位表示当前对象是否为饥饿模式，0为正常模式，1为饥饿模式。

    mutexWaiterShift = iota // 3，从倒数第四位往前的bit位表示在排队等待的goroutine数
)

type Mutex struct {
	sync.Mutex
}
func (m *Mutex) Count() int {
	//获得state字段的值
	v := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
	v = v >> mutexWaiterShift //得到等待者数值
	v = v + (v & mutexLocked) // 如果当前的锁已经被其他 goroutine 持有，那么，我们就稍微调整一下这个值再加上锁持有者的数量 0或者 1
	return int(v)
}
```

state 这个字段的第一位是用来标记锁是否被持有，第二位用来标记是否已经唤醒了一个等待者，第三位标记锁是否处于饥饿状态，通过分析这个 state 字段我们就可以得到这些状态信息。我们可以为这些状态提供查询的方法，这样就可以实时地知道锁的状态了。

```
// 锁是否被持有
func (m *Mutex) IsLocked() bool {
	state := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
	return state & mutexLocked == mutexLocked
}
// 是否有等待者被唤醒
func (m *Mutex) IsWoken() bool {
	state := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
	return state & mutexWoken == mutexWoken
}
// 是否处于饥饿状态
func (m *Mutex) IsStarving() bool {
	state := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
	return state & mutexStarving == mutexStarving
}
```

我们可以写一个程序测试一下，比如，在 1000 个 goroutine 并发访问的情况下，我们可以把锁的状态信息输出出来：

```
func main() {
	var m Mutex
	for i := 0; i < 1000; i++ {
		go func() {
			m.Lock()
			time.Sleep(time.Second)
			m.Unlock()
		}()
	}
	time.Sleep(time.Second)
	fmt.Printf("waitings:%d, isLocked: %t, woken: %t, starving: %t\n", m.Count(), m.IsLocked(), m.IsWoken(), m.IsStarving())
}
```

![image-20220302135715083](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302135715083.png?lastModify=1668648666)



1. **使用Mutex组成更丰富的数据结构：线程安全的队列**

Mutex 经常会和其他非线程安全（对于 Go 来说，我们其实指的是 goroutine 安全）的数据结构一起，组合成一个线程安全的数据结构。新数据结构的业务逻辑由原来的数据结构提供，而 Mutex 提供了锁的机制，来保证线程安全。比如队列，我们可以通过 Slice 来实现，但是通过 Slice 实现的队列不是线程安全的，出队（Dequeue）和入队（Enqueue）会有 data race 的问题。这个时候，Mutex 就要隆重出场了，通过它，我们可以在出队和入队的时候加上锁的保护。

```
type SliceQueue struct {
	data []interface{}
	m    sync.Mutex
}

func NewSliceQueue(n int) *SliceQueue {
	return &SliceQueue{data: make([]interface{}, 0, n)}
}
func (q *SliceQueue) Enquque(v interface{}) {
	q.m.Lock()
	q.data = append(q.data, v)
	q.m.Unlock()
}
func (q *SliceQueue) Dequeue() interface{} {
	q.m.Lock()
	defer q.m.Unlock()
	if len(q.data) == 0 {
		q.m.Unlock()
		return nil
	}
	v := q.data[0]
	q.data = q.data[1:]
	return v
}
```





#### 5.5 RWMutex

 Mutex，我们使用它来保证读写共享资源的安全性。不管是读还是写，我们都通过 Mutex 来保证只有一个 goroutine 访问共享资源，这在某些情况下有点“浪费”。

比如说，在写少读多的情况下，即使一段时间内没有写操作，大量并发的读访问也不得不在 Mutex 的保护下变成了串行访问，这个时候，使用 Mutex，对性能的影响就比较大。

**readers-writers问题：**同时可能有多个读或者多个写，但是只要有一个线程在执行写操作，其它的线程都不能执行读写操作。



##### 使用场景

首先了解下什么是RWMutex?

 RWMutex 是一个 reader/writer 互斥锁。RWMutex 在某一时刻只能由任意数量的 reader 持有，或者是只被单个的 writer 持有。

RWMutex 的零值是未加锁的状态，所以，当你使用 RWMutex 的时候，无论是声明变量，还是嵌入到其它 struct 中，都不必显式地初始化。

RWMutex 的方法也很少，总共有 5 个:

- Lock/Unlock：写操作时调用的方法。如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。
- RLock/RUnlock：读操作时调用的方法。如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。
- RLocker：这个方法的作用是为读操作返回一个 Locker 接口的对象。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

接下来实现一个计数器例子：计数器的 count++操作是写操作，而获取 count 的值是读操作，这个场景非常适合读写锁，因为读操作可以并行执行，写操作时只允许一个线程执行，这正是 readers-writers 问题！在这个例子中，使用 10 个 goroutine 进行读操作，每读取一次，sleep 1 秒，同时，还有一个 gorotine 进行写操作，每一秒写一次，这是一个 1 writer-n reader 的读写场景，而且写操作还不是很频繁（一秒一次）：

```
type Counter struct {
	m     sync.RWMutex
	count uint64
}

func (c *Counter) Incr() {
	c.m.Lock()
	defer c.m.Unlock()
	c.count++
}
func (c *Counter) Count() uint64 {
	c.m.RLock()
	defer c.m.RUnlock()
	return c.count
}
func main() {
	var counter Counter
	start := time.Now()
	for i := 0; i < 1000000; i++ {
		go func() {
			counter.Count()
			time.Sleep(time.Millisecond)
		}()
	}
	counter.Incr()
	time.Sleep(time.Second)

	cost := time.Since(start)
	fmt.Printf("cost=[%s]", cost)
}
```

![image-20220302150948096](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302150948096.png?lastModify=1668648666)



Incr 方法每秒才调用一次，所以，writer 竞争锁的频次是比较低的，而 10 00000个 goroutine 每纳秒都要执行一次查询，通过读写锁，可以极大提升计数器的性能，因为在读取的时候，可以并发进行。如果使用 Mutex，性能就不会像读写锁这么好。因为多个 reader 并发读的时候，使用互斥锁导致了 reader 要排队读的情况，没有 RWMutex 并发读的性能好。就好比我们把上面代码的读锁操作改为如下，结果也会改变：

```
func (c *Counter) Count() uint64 {
	c.m.Lock()
	defer c.m.Unlock()
	return c.count
}
```



![image-20220302150917021](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302150917021.png?lastModify=1668648666)



**所以，如果你遇到可以明确区分 reader 和 writer goroutine 的场景，且有大量的并发读、少量的并发写，并且有强烈的性能需求，你就可以考虑使用读写锁 RWMutex 替换 Mutex。**

**在实际使用 RWMutex 的时候**：

- 如果我们在 struct 中使用 RWMutex 保护某个字段，一般会把它和这个字段放在一起，用来指示两个字段是一组字段。
- 除此之外，我们还可以采用匿名字段的方式嵌入 struct，这样，在使用这个 struct 时，我们就可以直接调用 Lock/Unlock、RLock/RUnlock 方法了





##### 实现原理

RWMutex是基于Mutex实现的。readers-writers 问题一般有三类，基于对读和写操作的优先级，读写锁的设计和实现也分成三类：

- **Read-preferring**：读优先的设计可以提供很高的并发性，但是，在竞争激烈的情况下可能会导致写饥饿。这是因为，如果有大量的读，这种设计会导致只有所有的读都释放了锁之后，写才可能获取到锁。
- **write-preferring：**写优先的设计意味着，如果已经有一个 writer 在等待请求锁的话，它会阻止新来的请求锁的 reader 获取到锁，所以优先保障 writer。当然，如果有一些 reader 已经请求了锁的话，新请求的 writer 也会等待已经存在的 reader 都释放锁之后才能获取。所以，写优先级设计中的优先权是**针对新来的请求**而言的。这种设计主要避免了 writer 的饥饿问题。
- **不指定优先级：**这种设计比较简单，不区分 reader 和 writer 优先级，某些场景下这种不指定优先级的设计反而更有效，因为第一类优先级会导致写饥饿，第二类优先级可能会导致读饥饿，这种不指定优先级的访问不再区分读写，大家都是同一个优先级，解决了饥饿的问题

**Go 标准库中的 RWMutex 设计是 Write-preferring 方案。一个正在阻塞的 Lock 调用会排除新的 reader 请求到锁**

```
type RWMutex struct {
    w Mutex 			// 互斥锁解决多个writer的竞争
    writerSem uint32	// writer信号量
    readerSem uint32	// reader信号量
    readerCount int32	// reader的数量,记录当前 reader 的数量（以及是否有 writer 竞争锁）
    readerWait int32	// writer等待完成的reader的数量,记录 writer 请求锁时需要等待 read 完成的 reader 的数量
}
const rwmutexMaxReaders = 1 << 30	//定义了最大的 reader 数量
```

我们再来看下几个方法的具体实现：

**RLock/RUnlock 的实现**

```
func (rw *RWMutex) RLock() { 
    if atomic.AddInt32(&rw.readerCount, 1) < 0 { // rw.readerCount是负值的时候，意味着此时有writer等待请求锁，因为writer优先 
        runtime_SemacquireMutex(&rw.readerSem, false, 0) 
    }
}
func (rw *RWMutex) RUnlock() { 
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        rw.rUnlockSlow(r) // 有等待的writer
    }
}
func (rw *RWMutex) rUnlockSlow(r int32) {
    if atomic.AddInt32(&rw.readerWait, -1) == 0 { 
        // 最后一个reader了，writer终于有机会获得锁了 
        runtime_Semrelease(&rw.writerSem, false, 1) 
    }
```

**Lock的实现：**

```
func (rw *RWMutex) Lock() {
    // 首先解决其他writer竞争问题 
    rw.w.Lock() 
    // 反转readerCount，告诉reader有writer竞争锁 
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxRead 
    // 如果当前有reader持有锁，那么需要等待 
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 { 
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```

**Unlock实现：**

```
func (rw *RWMutex) Unlock() { 
    // 告诉reader没有活跃的writer了
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders) 
    // 唤醒阻塞的reader们 
    for i := 0; i < int(r); i++ { 
        runtime_Semrelease(&rw.readerSem, false, 0) 
    } 
    // 释放内部的互斥锁 
    rw.w.Unlock()
}
```

**总之：RWMutex 的应用场景非常明确，就是解决 readers-writers 问题。**



##### 3个踩坑点

1. **不可复制**

不能复制的原因和互斥锁一样。一旦读写锁被使用，它的字段就会记录它当前的一些状态。这个时候你去复制这把锁，就会把它的状态也给复制过来。但是，原来的锁在释放的时候，并不会修改你复制出来的这个读写锁，这就会导致复制出来的读写锁的状态不对，可能永远无法释放锁。

那该怎么办呢？其实，解决方案也和互斥锁一样。你可以借助 vet 工具，在变量赋值、函数传参、函数返回值、遍历数据、struct 初始化等时，检查是否有读写锁隐式复制的情景。

1. **重入导致死锁**

写锁因为重入（或递归调用）导致死锁的情况更多：

- 因为读写锁内部基于互斥锁实现对 writer 的并发访问，而互斥锁本身是有重入问题的，所以，writer 重入调用 Lock 的时候，就会出现死锁的现象

```
func foo(l *sync.RWMutex) { 
    fmt.Println("in foo") 
    l.Lock()
    bar(l)
    l.Unlock()
}
func bar(l *sync.RWMutex) { 
    l.Lock() 
    fmt.Println("in bar")
    l.Unlock()
}
func main() {
    l := &sync.RWMutex{}
    foo(l)
}
```

运行这个程序，你就会得到死锁的错误输出，在 Go 运行的时候，很容易就能检测出来。

- 第二种死锁的场景有点隐蔽:有活跃 reader 的时候，writer 会等待，如果我们在 reader 的读操作时调用 writer 的写操作（它会调用 Lock 方法），那么，这个 reader 和 writer 就会形成互相依赖的死锁状态。Reader 想等待 writer 完成后再释放锁，而 writer 需要这个 reader 释放锁之后，才能不阻塞地继续执行。这是一个读写锁常见的死锁场景。
- 第三种死锁的场景更加隐蔽：当一个 writer 请求锁的时候，如果已经有一些活跃的 reader，它会等待这些活跃的 reader 完成，才有可能获取到锁，但是，如果之后活跃的 reader 再依赖新的 reader 的话，这些新的 reader 就会等待 writer 释放锁之后才能继续执行，这就形成了一个环形依赖： **writer 依赖活跃的 reader -> 活跃的 reader 依赖新来的 reader -> 新来的 reader 依赖 writer**![image-20220302154242482](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302154242482.png?lastModify=1668648666)

```
func factorial(m *sync.RWMutex, n int) int {
	if n < 1 {
		return 0
	}
	fmt.Println("Rlock")
	m.RLock()
	defer func() {
		fmt.Println("urlock")
		m.RUnlock()
	}()
	time.Sleep(100 * time.Millisecond)
	return factorial(m, n-1) * n
}
func main() {
	var m sync.RWMutex
	go func() { //writer，稍微等待，制造一个调用lock场景
		time.Sleep(200 * time.Millisecond)
		m.Lock()
		fmt.Println("lock")
		time.Sleep(100 * time.Millisecond)
		m.Unlock()
		fmt.Println("unlock")
	}()
	go func() {
		fmt.Println(factorial(&m, 10))
	}()
	select {}
}
```

![image-20220302155921113](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302155921113.png?lastModify=1668648666)

factoria 方法是一个递归计算阶乘的方法，我们用它来模拟 reader。为了更容易地制造出死锁场景，我在这里加上了 sleep 的调用，延缓逻辑的执行。这个方法会调用读锁（第 27 行），在第 33 行递归地调用此方法，每次调用都会产生一次读锁的调用，所以可以不断地产生读锁的调用，而且必须等到新请求的读锁释放，这个读锁才能释放。同时，我们使用另一个 goroutine 去调用 Lock 方法，来实现 writer，这个 writer 会等待 200 毫秒后才会调用 Lock，这样在调用 Lock 的时候，factoria 方法还在执行中不断调用 RLock。这两个 goroutine 互相持有锁并等待，谁也不会退让一步，满足了“writer 依赖活跃的 reader -> 活跃的 reader 依赖新来的 reader -> 新来的 reader 依赖 writer”的死锁条件，所以就导致了死锁的产生。

**所以，使用读写锁最需要注意的一点就是尽量避免重入，重入带来的死锁非常隐蔽，而且难以诊断**。



1. **释放未加锁的 RWMutex**

和互斥锁一样，Lock 和 Unlock 的调用总是成对出现的，**RLock 和 RUnlock 的调用也必须成对出现**。Lock 和 RLock 多余的调用会导致锁没有被释放，可能会出现死锁，而 Unlock 和 RUnlock 多余的调用会导致 panic。在生产环境中出现 panic 是大忌



#### 5.6 WaitGroup

**WaitGroup** :就是 package sync 用来做任务编排的一个并发原语。它要解决的就是并发 - 等待的问题：现在有一个 goroutine A 在检查点（checkpoint）等待一组 goroutine 全部完成，如果在执行任务的这些 goroutine 还没全部完成，那么 goroutine A 就会阻塞在检查点，直到所有 goroutine 都完成后才能继续执行

**一个使用 WaitGroup 的场景**。比如，我们要完成一个大的任务，需要使用并行的 goroutine 执行三个小任务，只有这三个小任务都完成，我们才能去执行后面的任务。如果通过轮询的方式定时询问三个小任务是否完成，会存在两个问题：一是，性能比较低，因为三个小任务可能早就完成了，却要等很长时间才被轮询到；二是，会有很多无谓的轮询，空耗 CPU 资源。那么，这个时候使用 WaitGroup 并发原语就比较有效了，它可以阻塞等待的 goroutine。等到三个小任务都完成了，再即时唤醒它们。

##### 基本用法

```
func (wg *WaitGroup) Add(delta int) 	//用来设置 WaitGroup 的计数值；
func (wg *WaitGroup) Done()				//用来将 WaitGroup 的计数值减 1，其实就是调用了 Add(-1)
func (wg *WaitGroup) Wait()				//调用这个方法的 goroutine 会一直阻塞{和select{}功能一样，直到 WaitGroup 的计数值变为 0
```

上计数器经典例子：

```
type Counter struct {
	m     sync.Mutex
	count uint64
}

func (c *Counter) Incr() {
	c.m.Lock()
	defer c.m.Unlock()
	c.count++
}
func (c *Counter) Count() uint64 {
	c.m.Lock()
	defer c.m.Unlock()
	return c.count
}
func main() {
	var counter Counter
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func(c *Counter, wg *sync.WaitGroup) {
			defer wg.Done()
			time.Sleep(time.Second)
			c.Incr()
		}(&counter, &wg)
	}
	wg.Wait()
    fmt.Println(counter.Count())
}
```

![image-20220302162010986](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302162010986.png?lastModify=1668648666)

如果我们把`fmt.Println(counter.Count())`放到`wg.Wait()`输出的就是0了



##### 实现原理

参考`E:\I can do all things!\Go\极客时间课程\2 高级Go语言工程师\2 Go并发编程实战\06`



##### 常见问题

1. **计数器设置为负值**

WaitGroup 的计数器的值必须大于等于 0。我们在更改这个计数值的时候，WaitGroup 会先做检查，如果计数值被设置为负数，就会导致 panic。

一般情况下，有两种方法会导致计数器设置为负数。

- 第一种方法是：调用 Add 的时候传递一个负数
- 第二个方法是：调用 Done 方法的次数过多，超过了 WaitGroup 的计数值。**使用 WaitGroup 的正确姿势是，预先确定好 WaitGroup 的计数值，然后调用相同次数的 Done 完成相应的任务。**

2**.不期望的Add时机**

使用 WaitGroup 的时候，你一定要遵循的原则就是，等所有的 Add 方法调用之后再调用 Wait，否则就可能导致 panic 或者不期望的结果。

这种场景就是：启动四个 goroutine，每个 goroutine 内部调用 Add(1) 然后调用 Done()，主 goroutine 调用 Wait 等待任务完成。

![image-20220302163257258](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302163257258.png?lastModify=1668648666)

在这个例子中，我们原本设想的是，等四个 goroutine 都执行完毕后输出 Done 的信息，但是它的错误之处在于，将 WaitGroup.Add 方法的调用放在了子 gorotuine 中。等主 goorutine 调用 Wait 的时候，因为四个任务 goroutine 一开始都休眠，所以可能 WaitGroup 的 Add 方法还没有被调用，WaitGroup 的计数还是 0，所以它并没有等待四个子 goroutine 执行完毕才继续执行，而是立刻执行了下一步。

**导致这个错误的原因是，没有遵循先完成所有的 Add 之后才 Wait**，代码中你只要有add,就要想它是否完成了

要解决这个问题，一个方法是，预先设置计数值：![image-20220302163425625](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302163425625.png?lastModify=1668648666)

另一种方法是在启动子 goroutine 之前才调用 Add

![image-20220302163510240](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302163510240.png?lastModify=1668648666)

![image-20220302163459758](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302163459758.png?lastModify=1668648666)

**总之：无论是怎么修复，都要保证所有的 Add 方法是在 Wait 方法之前被调用的**

1. **前一个 Wait 还没结束就重用 WaitGroup**

首先，我们要知道：**WaitGroup 是可以重用的**。只要 WaitGroup 的计数值恢复到零值的状态，那么它就可以被看作是新创建的 WaitGroup，被重复使用。

但是，如果我们在 WaitGroup 的计数值还没有恢复到零值的时候就重用，就会导致程序 panic。我们看一个例子，初始设置 WaitGroup 的计数值为 1，启动一个 goroutine 先调用 Done 方法，接着就调用 Add 方法，**Add 方法有可能和主 goroutine 并发执行。**

![image-20220302163931129](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302163931129.png?lastModify=1668648666)

第 6 行虽然让 WaitGroup 的计数恢复到 0，但是因为第 9 行有个 waiter 在等待，如果等待 Wait 的 goroutine，刚被唤醒就和 Add 调用（第 7 行）有并发执行的冲突，所以就会出现 panic。

**总结就是：**WaitGroup 虽然可以重用，但是是有一个前提的，那就是必须等到上一轮的 Wait 完成之后，才能重用 WaitGroup 执行下一轮的 Add/Wait，如果你**在 Wait 还没执行完的时候就调用下一轮 Add 方法，就有可能出现 panic。**



#### 5.7 Cond

我们可以把一个条件变量Cond原语理解为一个容器，这个容器中存放着一个或一组等待着某个条件成立的 Goroutine。当条件成立后，这些处于等待状态的 Goroutine 将得到通知，并被唤醒继续进行后续的工作。这与百米飞人大战赛场上，各位运动员等待裁判员的发令枪声的情形十分类似。条件变量是同步原语的一种，如果没有条件变量，开发人员可能需要在 Goroutine 中通过连续轮询的方式，检查某条件是否为真，这种连续轮询非常消耗资源，因为 Goroutine 在这个过程中是处于活动状态的，但它的工作又没有进展。

Cond 是为等待 / 通知场景下的并发问题提供支持的。它提供了条件变量的三个基本方法 Signal、Broadcast 和 Wait，为并发的 goroutine 提供等待 / 通知机制。使用 Cond 之所以容易出错，就是 Wait 调用需要加锁，以及被唤醒后一定要检查条件是否真的已经满足。你需要牢记这两点。

但是, Cond 在实际项目中被使用的机会比较少，原因总结起来有两个:

- 第一，同样的场景我们会使用其他的并发原语来替代。Go 特有的 Channel 类型，有一个应用很广泛的模式就是通知机制，这个模式使用起来也特别简单。所以很多情况下，我们会使用 Channel 而不是 Cond 实现 wait/notify 机制。
- 第二，对于简单的 wait/notify 场景，比如等待一组 goroutine 完成之后继续执行余下的代码，我们会使用 WaitGroup 来实现。因为 WaitGroup 的使用方法更简单，而且不容易出错。比如，上面百米赛跑的问题，就可以很方便地使用 WaitGroup 来实现。

##### 基本用法

```
func main() {
    c := sync.NewCond(&sync.Mutex{})
    var ready int
    for i := 0; i < 10; i++ {
        go func(i int) {
            time.Sleep(time.Duration(rand.Int63n(10)) * time.Second)
            // 加锁更改等待条件
            c.L.Lock()
            ready++
            c.L.Unlock()
            log.Printf("运动员#%d 已准备就绪\n", i) 
            // 广播唤醒所有的等待者
            c.Broadcast() 
        }(i) 
    }
    c.L.Lock() 
    for ready != 10 { 
        c.Wait()
        log.Println("裁判员被唤醒一次") 
    } 
    c.L.Unlock() //所有的运动员是否就绪
    log.Println("所有运动员都准备就绪。比赛开始，3，2，1, ......")
}
```



首先，Cond 关联的 Locker 实例可以通过 c.L 访问，它内部维护着一个先入先出的等待队列。

**Signal 方法，**允许调用者 Caller 唤醒一个等待此 Cond 的 goroutine。如果此时没有等待的 goroutine，显然无需通知 waiter；调用 Signal 方法时，不强求你一定要持有 c.L 的锁

**Broadcast 方法**，允许调用者 Caller 唤醒所有等待此 Cond 的 goroutine。如果此时没有等待的 goroutine，显然无需通知 waiter．同样地，调用 Broadcast 方法时，也不强求你一定持有 c.L 的锁

**Ｗait 方法**，会把调用者 Caller 放入 Cond 的等待队列中并阻塞，直到被 Signal 或者 Broadcast 的方法从等待队列中移除并唤醒。调用 Wait 方法时必须要持有 c.L 的锁



##### 实现原理

参考`E:\I can do all things!\Go\极客时间课程\2 高级Go语言工程师\2 Go并发编程实战\07`



##### 易错点：

1. **用 Wait 的时候没有加锁**

![image-20220303091424985](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220303091424985.png?lastModify=1668648666)

**2.只调用了一次 Wait，没有检查等待条件是否满足，结果条件没满足，程序就继续执行了**

![image-20220303091507084](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220303091507084.png?lastModify=1668648666)



#### 5.8 Once



**Once 可以用来执行且仅仅执行一次动作，常常用于单例对象的初始化场景。**在高并发的场景下只执行一次，例如只加载一次配置文件、只关闭一次通道等。

首先认识下对单例化对象初始化这件事：

- 定义package级别的变量，这样程序在启动的时候就可以初始化

```
package abc
```

- init函数中初始化

```
package abc
var startTime time.Time
func init(){
    startTime = time.Now()
}
```

- 在 main 函数开始执行的时候，执行一个初始化的函数

```
package abc
var startTime time.Time

func initApp() { 
    startTime = time.Now()
}
func main() {
    initApp()
}
```

这三种方法都是线程安全的，并且后两种方法还可以根据传入的参数实现定制化的初始化操作。但是很多时候我们是要延迟进行初始化的，所以有时候单例资源的初始化，我们会使用下面的方法：

![image-20220302170406347](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302170406347.png?lastModify=1668648666)

这种方式虽然实现起来简单，但是有性能问题。一旦连接创建好，每次请求的时候还是得竞争锁才能读取到这个连接，这是比较浪费资源的，因为连接如果创建好之后，其实就不需要锁的保护了。怎么办呢？这个时候就可以使用这一讲要介绍的 Once 并发原语了。



##### 使用场景

**sync.Once 只暴露了一个方法 Do，你可以多次调用 Do 方法，但是只有第一次调用 Do 方法时 f 参数才会执行，这里的 f 是一个无参数无返回值的函数。**

因为这里的 f 参数是一个无参数无返回的函数，所以你可能会通过闭包的方式引用外面的参数，比如：

![image-20220302170713438](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302170713438.png?lastModify=1668648666)

**而且在实际的使用中，绝大多数情况下，你会使用闭包的方式去初始化外部的一个资源。**

还有一种场景：![image-20220302171021343](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302171021343.png?lastModify=1668648666)

它将 sync.Once 和 *Float 封装成一个对象，提供了只初始化一次的值 v。 你看它的 three 方法的实现，虽然每次都调用 threeOnce.Do 方法，但是参数只会被调用一次。当你使用 Once 的时候，你也可以尝试采用这种结构，将值和 Once 封装成一个新的数据结构，提供只初始化一次的值。

总结一下 Once 并发原语解决的问题和使用场景：**Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。**



##### Once内部实现

参考`E:\I can do all things!\Go\极客时间课程\2 高级Go语言工程师\2 Go并发编程实战\06`

##### 两个易错点

1. **死锁**

 Do 方法会执行一次 f，但是如果 f 中再次调用这个 Once 的 Do 方法的话，就会导致死锁的情况出现。这还不是无限递归的情况，而是的的确确的 Lock 的递归调用导致的死锁。

```
func main() { 
    var once sync.Once
    once.Do(func() { 
        once.Do(func() { 
            fmt.Println("初始化")
        }) 
    })
}
```

1. ##### 未初始化

如果 f 方法执行的时候 panic，或者 f 执行初始化资源的时候失败了，这个时候，Once 还是会认为初次执行已经成功了，即使再次调用 Do 方法，也不会再次执行 f。

比如下面的例子，由于一些防火墙的原因，googleConn 并没有被正确的初始化，后面如果想当然认为既然执行了 Do 方法 googleConn 就已经初始化的话，会抛出空指针的错误：

```
func main() {
   var once sync.Once var googleConn net.Conn // 到Google网站的一个连接 
    once.Do(func() { 
        // 建立到google.com的连接，有可能因为网络的原因，googleConn并没有建立成功，此时它
        googleConn, _ = net.Dial("tcp", "google.com:80") 
    }) 
    // 发送http请求 
    googleConn.Write([]byte("GET / HTTP/1.1\r\nHost: google.com\r\n Accept: */ )
    io.Copy(os.Stdout, googleConn)
}
```

既然执行过 Once.Do 方法也可能因为函数执行失败的原因未初始化资源，并且以后也没机会再次初始化资源，那么这种初始化未完成的问题该怎么解决呢？

**独家秘笈** : 我们可以自己实现一个类似 Once 的并发原语，既可以返回当前调用 Do 方法是否正确完成，还可以在初始化失败后调用 Do 方法再次尝试初始化，直到初始化成功才不再初始化了。

#### 5.9 map

首先我们要知道Go内建的map对象**一定要显式初始化才能用，其次它不是线程安全的**，并发读写的时候运行时会检查，遇到并发问题就会导致panic。如下：

```
func main() {
	var m = make(map[int]int, 10) // 注意：make对slice,map,channel就是显式初始化了
	go func() {
		for {
			m[1] = 1 //设置key
		}
	}()
	go func() {
		for {
			_ = m[2] //访问这个map
		}
	}()
	select {} 
}
```

![image-20220302192717308](file://E:/I-can-do-all-things/Go/pics/Go%E8%AF%AD%E6%B3%95%E5%9F%BA%E7%A1%80/image-20220302192717308.png?lastModify=1668648666)



**那么如何实现一个线程安全的map呢？**

这里先说下有三种方法：**1. 加读写锁   2. 加分片式锁    3.用sync.Map**       这里用哪种，Go 开发者给的建议，就是通过性能测试，看看某种线程安全的 map 实现是否满足你的需求。

1. **加读写锁**

在正式发布泛型特性后，我们才能实现一个通用的支持泛型的加锁 map。但是，将要发布的泛型方案已经可以验证测试了，离发布也不远了，也许发布之后 sync.Map 就支持泛型了。

当然了，如果没有泛型支持，我们也能解决这个问题。我们可以通过 interface{}来模拟泛型，但还是要涉及接口和具体类型的转换，比较复杂，还不如将要发布的泛型方案更直接、性能更好。

这里我以一个**具体的 map 类型**为例，来演示利用**读写锁实现线程安全的 map[int]int** 类型：

```
type RWMap struct {
	sync.RWMutex
	m map[int]int
}

//新建一个RWWap
func NewRWMap(n int) *RWMap {
	return &RWMap{m: make(map[int]int, n)}
}

//从map中读取一个值
func (m *RWMap) Get(k int) (int, bool) {
	m.RLock()
	defer m.RUnlock()
	value, existed := m.m[k] //在所得保护下从map中读取
	return value, existed
}

//设置一个键值对
func (m *RWMap) Set(k, v int) {
	m.Lock()
	defer m.Unlock()
	m.m[k] = v
}

//删除一个键
func (m *RWMap) Delete(k int) {
	m.Lock()
	defer m.Unlock()
	delete(m.m, k)
}

//求map长度
func (m *RWMap) Len() int {
	m.RLock()
	defer m.RUnlock()
	return len(m.m)
}

//遍历map
func (m *RWMap) Each(f func(k, v int) bool) {
	m.RLock()
	defer m.RUnlock()
	for k, v := range m.m {
		if !f(k, v) {
			return
		}
	}
}
```

对 map 对象的操作，无非就是增删改查和遍历等几种常见操作。我们可以把这些操作分为读和写两类，其中，查询和遍历可以看做读操作，增加、修改和删除可以看做写操作。如例子所示，我们可以通过读写锁对相应的操作进行保护。



1. **分片加锁：更高效的并发map**

虽然使用读写锁可以提供线程安全的 map，但是在大量并发读写的情况下，锁的竞争会非常激烈。曾经提到过，锁是性能下降的万恶之源之一。

在并发编程中，我们的一条原则就是尽量减少锁的使用。一些单线程单进程的应用（比如 Redis 等），基本上不需要使用锁去解决并发线程访问的问题，所以可以取得很高的性能。但是对于 Go 开发的应用程序来说，并发是常用的一个特性，在这种情况下，我们能做的就是，**尽量减少锁的粒度和锁的持有时间。**

你可以优化业务处理的代码，以此来减少锁的持有时间，比如将串行的操作变成并行的子任务执行。不过，这就是另外的故事了，今天我们还是主要讲对同步原语的优化，所以这里我重点讲如何减少锁的粒度。

**减少锁的粒度常用的方法就是分片（Shard）**，将一把锁分成几把锁，每个锁控制一个分片。Go 比较知名的分片并发 map 的实现是orcaman/concurrent-map。它默认采用 32 个分片，**GetShard 是一个关键的方法，能够根据 key 计算出分片索引。**

```
var SHARD_COUNT = 32

//分成SHARD_COUNT个分片的map
type ConcurrentMap []*ConcurrentMapShared

//通过RWMutex保护的线程安全的分片，包含一个map
type ConcurrentMapShared struct {
	items map[string]interface{}
	sync.RWMutex
}

//创建并发map
func New() ConcurrentMap {
	m := make(ConcurrentMap, SHARD_COUNT)
	for i := 0; i < SHARD_COUNT; i++ {
		m[i] = &ConcurrentMapShared{items: make(map[string]interface{})}
	}
	return m
}

//根据key计算分片索引
func (m ConcurrentMap) GetShard(key string) *ConcurrentMapShared {
	return m[uint(fnv32(key))%uint(SHARD_COUNT)]
}

//增加或者查询的时候，首先根据分片索引得到分片对象，然后对分片对象加锁进行操作
//增加
func (m ConcurrentMap) Set(key string, value interface{}) {
	//根据key计算对应分片
	shard := m.GetShard(key)
	shard.Lock()
	defer shard.Unlock()
	shard.items[key] = value
}
func (m ConcurrentMap) Get(key string, value interface{}) (interface{}, bool) {
	//根据key计算对应分片
	shard := m.GetShard(key)
	shard.Lock()
	defer shard.Unlock()
	value, ok := shard.items[key]
	return value, ok
}
```

**在我个人使用并发 map 的过程中，加锁和分片加锁这两种方案都比较常用，如果是追求更高的性能，显然是分片加锁更好，因为它可以降低锁的粒度，进而提高访问此 map 对象的吞吐。如果并发性能要求不是那么高的场景，简单加锁方式更简单。**



1. ##### 应对特殊场景的sync.Map

sync.Map，这是 Go 官方线程安全 map 的标准实现。虽然是官方标准，反而是不常用的，为什么呢？一句话来说就是 map 要解决的场景很难描述，很多时候在做抉择时根本就不知道该不该用它。但是呢，确实有一些特定的场景，我们需要用到 sync.Map 来实现。sync.Map 并不是用来替换内建的 map 类型的，它只能被应用在一些特殊的场景里。

官方的文档中指出，在以下两个场景中使用 sync.Map，会比使用 map+RWMutex 的方式，性能要好得多：

- 1.只会增长的缓存系统中，一个 key 只写入一次而被读很多次；
- 2.多个 goroutine 为不相交的键集读、写和重写键值对。

我们可以把 sync.Map 看成一个生产环境中很少使用的同步原语。

核心的三个方法：

```
//适用于读多写少的场景
//创建 var 变量名 sync.Map{}
//写入 变量名.Store("key",value)
//读取 value,_=变量名.Load("key")
//删除 变量名.Delete("key")
//读取或写入 变量名.LoadOrStore("key",value)
//  遍历
// 变量名.Range(func(key, value interface{}) bool {
//     name := key.(string)
//     age := value.(int)
//     fmt.Println(name, age)
//     return true
// })
var m = sync.Map{}
func main() {
	wg := sync.WaitGroup{}
	for i := 0; i < 2000; i++ {
		wg.Add(1)
		go func(n int) {
			key := strconv.Itoa(n)
			m.Store(key, n)
			value, _ := m.Load(key)
			fmt.Printf("k=:%v,v:=%v\n", key, value)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```





#### 5.10 Pool

Go 是一个自动垃圾回收的编程语言，和其它没有自动垃圾回收的编程语言不同，使用 Go 语言创建对象的时候，我们没有回收 / 释放的心理负担，想用就用，想创建就创建。

但是，如果你想**使用 Go 开发一个高性能的应用程序的话，就必须考虑垃圾回收给性能带来的影响**，毕竟，Go 的自动垃圾回收机制还是有一个 STW（stop-the-world，程序暂停）的时间，而且，大量地创建在堆上的对象，也会影响垃圾回收标记的时间。

所以，一般我们做性能优化的时候，会采用对象池的方式，把不用的对象回收起来，避免被垃圾回收掉，这样使用的时候就不必在堆上重新创建了。不止如此，像数据库连接、TCP 的长连接，这些连接在创建的时候是一个非常耗时的操作。如果每次都创建一个新的连接对象，耗时较长，很可能整个业务的大部分耗时都花在了创建连接上。所以，如果我们能把这些连接保存下来，避免每次使用的时候都重新创建，不仅可以大大减少业务的耗时，还能提高应用程序的整体性能。

Go 标准库中提供了一个通用的 Pool 数据结构，也就是 sync.Pool，我们使用它可以创建池化的对象。不过，这个类型也有一些使用起来不太方便的地方，就是**它池化的对象可能会被垃圾回收掉，**这对于数据库长连接等场景是不合适的。



##### 基本用法

sync.Pool 数据类型用来保存一组可独立访问的**临时**对象。临时说明它池化的对象会在未来的某个时候被毫无预兆地移除掉。而且，如果没有别的对象引用这个被移除的对象的话，这个被移除的对象就会被垃圾回收掉。

有两个知识点你需要记住：

1.sync.Pool 本身就是线程安全的，多个 goroutine 可以并发地调用它的方法存取对象；

2.sync.Pool 不可在使用之后再复制使用。

**方法：**

**New:** 当调用Pool的Get方法从池中获取元素，没有更多的空闲元素可返回时，就会调用这个New方法来创建新的元素。如果你没有设置New字段，没有更多的空闲元素可返回时，Get方法将返回nil,表明当前没有可用的元素

**Get**: 如果调用这个方法，就会从Pool中取走一个元素

**Put:** 这个方法用于讲一个元素返回给Pool，Pool会把这个元素保存到池中，并可以复用。

了解下sync最常用的一个场景：buffer池（缓冲池）。因为 byte slice 是经常被创建销毁的一类对象，使用 buffer 池可以缓存已经创建的 byte slice，比如，著名的静态网站生成工具 Hugo 中，就包含这样的实现bufpool，你可以看一下下面这段代码：

```
var buffers = sync.Pool{ 
    New: func() interface{} { 
        return new(bytes.Buffer)
    },
}func GetBuffer() *bytes.Buffer { 
    return buffers.Get().(*bytes.Buffer)
}
func PutBuffer(buf *bytes.Buffer) { 
    buf.Reset() buffers.Put(buf)
}
```

请你注意了，这段代码是有问题的，你一定不要将上面的代码应用到实际的产品中。它可能会有内存泄漏的问题



##### 两个坑点



##### 第三方库



##### 常用场景



1. http client池
2. TCP连接池
3. 数据库连接池
4. memcached client连接池
5. Worker Pool:

**gammazero/workerpool**：gammazero/workerpool 可以无限制地提交任务，提供了更便利的 Submit 和 SubmitWait 方法提交任务，还可以提供当前的 worker 数和任务数以及关闭 Pool 的功能。

**ivpusic/grpool：**grpool 创建 Pool 的时候需要提供 Worker 的数量和等待执行的任务的最大数量，任务的提交是直接往 Channel 放入任务。

**dpaks/goworkers：**dpaks/goworkers 提供了更便利的 Submi 方法提交任务以及 Worker 数、任务数等查询方法、关闭 Pool 的方法。它的任务的执行结果需要在 ResultChan 和 ErrChan 中去获取，没有提供阻塞的方法，但是它可以在初始化的时候设置 Worker 的数量和任务数。

其实，你也可以自己去开发自己的 Worker Pool，但是，对于我这种“懒惰”的人来说，只要满足我的实际需求，我还是倾向于从这个几个常用的库中选择一个来使用

​	



#### 5.11 Context

##### 由来

**引言知识点：控制并发有三种经典的方式，一种是WaitGroup，一种是channel结合select，另外一种就是Context**	（WaitGroup:这种方式是控制**多个goroutine同时**完成.）

context由来(**面子很大**)：，在 Go1.9 中，还专门实现了一个叫做 **type alias** 的新特性，然后把 x/net/context 中的 Context 定义成标准库 Context 的别名，以解决新旧 Context 类型冲突问题：

```
package context
import "context"
type Context = context.Context
type CancelFunc = context.CancelFunc
```



上下文信息，把上下文传递给你，你接收后，就可以获取之前不了解的信息。上下文（Context）就是这么重要。在我们的开发场景中，上下文也是不可或缺的，缺少了它，我们就不能获取完整的程序信息。那到底啥是上下文呢？其实，这就是指，在 API 之间或者方法调用之间，所传递的除了业务参数之外的额外信息。比如，服务端接收到客户端的 HTTP 请求之后，可以把客户端的 IP 地址和端口、客户端的身份信息、请求接收的时间、Trace ID 等信息放入到上下文中，这个上下文可以在后端的方法调用中传递，后端的业务方法除了利用正常的参数做一些业务处理（如订单处理）之外，还可以从上下文读取到消息请求的时间、Trace ID 等信息，把服务处理的时间推送到 Trace 服务中。Trace 服务可以把同一 Trace ID 的不同方法的调用顺序和调用时间展示成流程图，方便跟踪。不过，Go 标准库中的 Context 功能还不止于此，它还提供了**超时（Timeout）和取消（Cancel）的机制**

在很多场景下，使用 Context 其实会很方便，所以现在它已经 在 Go 生态圈中传播开来了，包括很多的 Web 应用框架，都切换成了标准库的 Context。 标准库中

的 database/sql、os/exec、net、net/http 等包中都使用到了 Context。而 且，如果我们遇到了**下面的一些场景，也可以考虑使用 Context：**

- **上下文信息传递 （request-scoped），比如处理 http 请求、在请求处理链路上传递信 息**； 
- **控制子 goroutine 的运行**；嵌套的子goroutine
- 超时控制的方法调用； 
- 可以取消的方法调用。





##### Context接口类型

标准库`context`，它定义了`Context`类型，**专门用来简化 对于处理单个请求的多个 goroutine 之间传递上下文信息,如:数据、取消信号、截止时间等相关操作**，这些操作可能涉及多个 API 调用。

![image-20211122110839075](file://E:/I-can-do-all-things/Go/pics/Go%E5%9F%BA%E7%A1%80/image-20211122110839075.png?lastModify=1668648666)

```
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

- **Deadline 方法**会返回这个 Context 被取消的截止日期。如果没有设置截止日期，ok 的值 是 false。后续每次调用这个对象的 Deadline 方法时，都会返回和第一次调用相同的结果。
- **Done 方法**返回一个 Channel 对象。在 Context 被取消时，此 Channel 会被 close，如 果没被取消，可能会返回 nil。后续的 Done 调用总是返回相同的结果。当 Done 被 close 的时候，你可以通过 ctx.Err 获取错误信息。Done 这个方法名其实起得并不好，因为名字太过笼统，不能明确反映 Done 被 close 的原因，因为 cancel、timeout、deadline 都可 能导致 Done 被 close，不过，目前还没有一个更合适的方法名称。 关于 Done 方法，你必须要记住的知识点就是：**如果 Done 没有被 close，Err 方法返回 nil；如果 Done 被 close，Err 方法会返回 Done 被 close 的原因。**
- **Value** 返回此 ctx 中和指定的 key 相关联的 value





在使用Context时，有一些**约定俗成的规则**。

- 一般函数使用 Context 的时候，会把这个参数放在第一个参数的位置。
- 从来不把 nil 当做 Context 类型的参数值，可以使用 context.Background() 创建一个 空的上下文对象，也不要使用 nil。
- Context 只用来临时做函数之间的上下文透传，不能持久化 Context 或者把 Context 长久保存。把 Context 持久化到数据库、本地文件或者全局变量、缓存中都是错误的用 法。 
- key 的类型不应该是字符串类型或者其它内建类型，否则容易在包之间使用 Context 时 候产生冲突。使用 WithValue 时，key 的类型应该是自己定义的类型。 
- 常常使用 struct{}作为底层类型定义 key 的类型。对于 exported key 的静态类型，常 常是接口或者指针。这样可以尽量减少内存分配。



##### 四种具体实现

- emptyCtx

本质是个整型

- cancelCtx
- timerCtx
- valueCtx

```
type valueCtx struct {
    Context
    key, val interface{}
}
```

它持有一个 key-value 键值对，还持有 parent 的 Context。它覆盖了 Value 方法，优先 从自己的存储中检查这个 key，不存在的话会从 parent 中继续检查。 Go 标准库实现的 Context 还实现了链式查找。如果不存在，还会向 parent Context 去查 找，如果 parent 还是 valueCtx 的话，还是遵循相同的原则：valueCtx 会嵌入 parent， 所以还是会查找 parent 的 Value 方法的。



##### 六个生成特殊Context方法



- **Background** : 返回一个非 nil 的emptyCtx,初始化一个顶层父节点用它。不会被 cancel，不会超时，没有截止日期。一般用在主函数、初始化、测试以及创建根 Context 的时候。
- **TODO** ：和Background底层实现一样。可以直接使用 context.Background。
- **WithCancel**

WithCancel 方法返回 parent 的副本，只是副本中的 Done Channel 是新建的对象，它的类型是 cancelCtx。我们常常在一些需要主动取消长时间的任务时，创建这种类型的 Context，然后把这个 Context 传给长时间执行任务的 goroutine。当需要中止任务时，我们就可以 cancel 这个 Context，这样长时间执行任务的 goroutine，就可以通过检查这个 Context，知道 Context 已经被取消了。

当调用返回的cancel函数或当关闭父上下文的Done通道时，将关闭返回上下文的Done通道，无论先发生什么情况。取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func main() {
	ctx, clear := context.WithCancel(context.Background())
	message := make(chan int)
	go son(ctx, message)
	for i := 0; i < 10; i++ {
		message <- i
	}
	fmt.Println("1")
	clear() //// 当我们弄完之后调用cancel
	fmt.Println("2")
	time.Sleep(time.Second)
	fmt.Println("主进程结束")
}
func son(ctx context.Context, msg chan int) {
	t := time.Tick(time.Second)
	for _ = range t {
		select {
		case m := <-msg:
			fmt.Printf("收到值%d\n", m)
		case <-ctx.Done(): //Done方法需要返回一个Channel，这个Channel会在当前工作完成或者上下文被取消之后关闭
			fmt.Println("结束")
			return // return结束该goroutine，防止泄露
		}
	}
}
```

- **WithValue**

WithValue 基于 parent Context 生成一个新的 Context，保存了一个 key-value 键值 对。它常常用来传递上下文。 WithValue 方法其实是创建了一个类型为 valueCtx 的 Context.

```
func WithValue(parent Context, key, val interface{}) Context
```

所提供的键key必须是可比较的，并且不应该是`string`类型或任何其他内置类型，以避免使用上下文在包之间发生冲突,`WithValue`的用户应该为键定义自己的类型。

```
ctx = context.TODO()
ctx = context.WithValue(ctx, "key1", "0001")
ctx = context.WithValue(ctx, "key2", "0001")
ctx = context.WithValue(ctx, "key3", "0001")
ctx = context.WithValue(ctx, "key4", "0004")
fmt.Println(ctx.Value("key1"))//0001
```

- **WithDeadline**

返回父上下文的副本，并将deadline调整为不迟于d。当截止日过期时，当调用返回的cancel函数时，或者当父上下文的Done通道关闭时，返回上下文的Done通道将被关闭，以最先发生的情况为准。  取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。

```
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func main() {
	d := time.Now().Add(50 * time.Microsecond) //定义了一个50毫秒之后过期的deadline
	ctx, cancel := context.WithDeadline(context.Background(), d)
	defer cancel()
	select { //让主程序陷入等待：等待1秒后打印overslept退出或者等待ctx过期后退出。
	case <-time.After(time.Second):
		fmt.Println("overslepy")
	case <-ctx.Done(): //ctx 50毫秒后就会过期，所以ctx.Done()会先接收到context到期通知，并且会打印ctx.Err()的内容。
		fmt.Println(ctx.Err())
	}
}
```

- **WithTimeout**

WithTimeout 其实是和 WithDeadline 一样，只不过一个参数是超时时间，一个参数是截止时间。超时时间加上当前时间，其实就是截止时间，因此，WithTimeout 的实现是：return WithDeadline(parent, time.Now().Add(timeout))

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
var wg sync.WaitGroup
func worker(ctx context.Context) {
LOOP:
	for {
		fmt.Println("db connecting ...")
		time.Sleep(time.Millisecond * 10) // 假设正常连接数据库耗时10毫秒
		select {
		case <-ctx.Done(): // 50毫秒后自动调用
			break LOOP
		default:
		}
	}
	fmt.Println("worker done!")
	wg.Done()
}
func main() {
	// 设置一个50毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)
	wg.Add(1)
	go worker(ctx)
	time.Sleep(time.Second * 5)
	cancel() // 通知子goroutine结束
	wg.Wait()
	fmt.Println("over")
}
```



##### **总结**

我们经常使用 Context 来取消一个 goroutine 的运行，这是 Context 最常用的场景之 一，Context 也被称为 goroutine 生命周期范围（goroutine-scoped）的 Context，把 Context 传递给 goroutine。但是，goroutine 需要尝试检查 Context 的 Done 是否关闭 了

WithCancel 直接创建可以操作退出的子节点，WithTimeout 为子节点设置了超时 时间（还有多少时间结束），WithDeadline 为子节点设置了结束时间线（在什么时间结 束）。 但是这只是表层功能的不同，其实这三个库函数的**本质是一致**的,因为：

```
type Context interface {
// 当 Context 被取消或者到了 deadline，返回一个被关闭的 channel
Done() <-chan struct{}
...
}
//函数句柄
type CancelFunc func()
```

我们只理解核心的 Done() 方法和 CancelFunc 这两个函数就可以了。 在树形逻辑链条上，**一个节点其实有两个角色：一是下游树的管理者；二是上游树的被管 理者**，那么就对应需要有两个能力：

- 一个是能让整个下游树结束的能力，也就是函数句柄 CancelFunc；
- 另外一个是在上游树结束的时候被通知的能力，也就是 Done() 方法。同时因为通知是 需要不断监听的，所以 Done() 方法需要通过 channel 作为返回值让使用方进行监听。

```
func main() {
// 创建截止时间
d := time.Now().Add(shortDuration)
// 创建有截止时间的 Context
ctx, cancel := context.WithDeadline(context.Background(), d)
defer cancel()
// 使用 select 监听 1s 和有截止时间的 Context 哪个先结束
select {
case <-time.After(1 * time.Second):
fmt.Println("overslept")
case <-ctx.Done():
fmt.Println(ctx.Err())
}
```

主线程创建了一个 1 毫秒结束的定时器 Context，在定时器结束的时候，主线程会通过 Done() 函数收到事件结束通知，然后主动调用函数句柄 cancelFunc 来通知所有子 Context 结束（这个例子比较简单没有子 Context）。

**我打个更形象的比喻，CancelFunc 和 Done 方法就像是电话的话筒和听筒，话筒 CancelFunc，用来告诉管辖范围内的所有 Context 要进行自我终结，而通过监听听筒 Done 方法，我们就能听到上游父级管理者的终结命令。**

总之，**CancelFunc 是主动让下游结束，而 Done 是被上游通知结束。**

回过头来看这三个库函数 WithCancel / WithDeadline / WithTimeout 就很好理解了。 它们的本质就是**“通过定时器来自动触发终结通知,也就是Done方法被自动调用，而不用听CancelFunc的命令”**，WithTimeout 设置若干秒后通知触 发终结，WithDeadline 设置未来某个时间点触发终结。 对应到 Context 代码中，**它们的功能就是：为一个父节点生成一个带有 Done 方法的子节 点，并且返回子节点的 CancelFunc 函数句柄（这个的功能就是来通知生成的子节点的子孙们）。**



### 6 Go如何保证并发读写的顺序



#### 6.1 内存模型

向广大的程序员提供一种保证，以便他们在做设计和开发程序时，面对同一个数据同时被多个 goroutine 访问的情况，可以做一些串行化访问的控制，比如使用 Channel 或 sync 包和 sync/atomic 包中的并发原语

允许编译器和硬件对程序做一些优化。这一点其实主要是为编译器开发者提供的保证，这样可以方便他们对 Go 的编译器做优化



#### 6.2 重排和可见性

**由于指令重排，代码并不一定会按照你写的顺序执行。**

```
var a, b int
func f() {
 a = 1 // w之前的写操作
 b = 2 // 写操作w
}
func g() {
 print(b) // 读操作r
 print(a) // ???
}
func main() {
 go f() //g1
 g() //g2
}
```

可以看到，第 9 行是要打印 b 的值。需要注意的是，即使这里打印出的值是 2，但是依然可能在打印 a 的值时，打印出初始值 0，而不是 1。这是因为，程序运行的时候，不能保证 g2 看到的 a 和 b 的赋值有先后关系。

如果要保证对“变量 v 的读操作 r”能够观察到一个对“变量 v 的写操作 w”，并且 r 只能观察到 w 对变量 v 的写，没有其它对 v 的写操作，也就是说，我们要保证 r 绝对能观察到 w 操作的结果，那么就需要同时满足两个条件：

- w happens before r；
- 其它对 v 的写操作（w2、w3、w4, …） 要么 happens before w，要么 happens after r，绝对不会和 w、r 同时发生，或者是在它们之间发生。



#### 6.3 happens - before

**在一个 goroutine 内部，程序的执行顺序和它们的代码指定的顺序是一样的，即使编译器或者 CPU 重排了读写顺序，从行为上来看，也和代码指定的顺序一样**!!!!!!!!

**因为 Go 只保证 单个goroutine 内部重排对读写的顺序没有影响**

在 goroutine 内部对一个局部变量 v 的读，一定能观察到最近一次对这个局部变量 v 的写。如果要保证**多个 goroutine 之间对一个共享变量的读写顺序**，在 Go 语言中，可以使用**并发原语为读写操作建立 happens-before 关系，这样就可以保证顺序了**

先给你补充三个 Go 语言中和内存模型有关的小知识，掌握了这些，你就能更好地理解下面的内容：

- 在 Go 语言中，对变量进行零值的初始化就是一个写操作。
- 如果对超过机器 word（64bit、32bit 或者其它）大小的值进行读写，那么，就可以看作是对拆成 word 大小的几个读写无序进行。
- Go 并不提供直接的 CPU 屏障（CPU fence）来提示编译器或者 CPU 保证顺序性，而是使用不同架构的内存屏障指令来实现统一的并发原语



#### 6.4 Go语言保证的happens-before关系



> #### init 函数

应用程序的初始化是在单一的goroutine执行的。如果包p导入了包q,那么q的init函数的执行一定happens before p 的任何初始化代码。



> #### goroutine

启动goroutine的go语句一定happens before此goroutine内部的代码执行。在一个 goroutine 内部，程序的执行顺序和它们的代码指定的顺序是一样的，即使编译器或者 CPU 重排了读写顺序，从行为上来看，也和代码指定的顺序一样



> #### Channel

- 第 n 个 send 一定 happens before 第 n 个 receive 的完成
- close 一个 Channel 的调用，肯定 happens before 从关闭的 Channel 中读取出一个零值
- 对于 unbuffered 的 Channel，也就是容量是 0 的 Channel，从此 Channel 中读取数据的调用一定 happens before 往此 Channel **发送数据的调用完成**
- 如果 Channel 的容量是 m（m>0），那么，第 n 个 receive 一定 happens before 第 n+m 个 **send 的完成**：若第 n 个 receive 没发生。这时，channel 被填满了，send 就会被阻塞。那当第 n 个 receive 发生时，sender goroutine 会被唤醒，之后再继续发送过程。这样，第 n 个 `receive` 一定 `happened before` 第 n+m 个 `send finished`。若第 n 个 receive 已经发生过了，这直接就符合了要求

> #### Mutex/RWMutex



> #### WaitGroup

Wait 方法等到计数值归零之后才返回

> #### Once

对于  once.Do(f) 调用，f 函数的那个单次调用一定 happens before 任何 once.Do(f) 调用的返回。换句话说，就是函数 f 一定会在 Do 方法返回之前执行

> #### atomic



# 原子操作



# 分布式并发