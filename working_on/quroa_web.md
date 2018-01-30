---
layout: post
title: "WSGI web服务网关接口(4)"
date: 2018-01-13 22:20:15 +0800
comments: true
categories: python
tags: [python,wsgi,translation,PEP]
---

> 本文译自：https://www.python.org/dev/peps/pep-0333
>
> 前一部分内容请查看：http://10111000.com/2017/12/22/PEP333_3/
>
> **tips**: 以下， 服务端代指服务端和网关， 框架端代指应用端和框架端

Web Server Architecture at Quora
Qurao 的 Web 服务器架构

As we last mentioned in [**Improving Site Speed**](https://blog.quora.com/Improving-Site-Speed), we flipped over to a new parallel architecture for rendering web pages in August 2012. This architecture allowed us to render pages up to 3-10x faster than we did before. We have since made several improvements to the original design of this architecture, allowing for even greater speed-ups and cost savings, which I’ll share here.

正如我们在[提高网页速度](https://blog.quora.com/Improving-Site-Speed)一文中提到的一样，我们在 2012 年的 8 月份使用了一种新的并行架构去渲染页面。相比以前，它使得我们可以以 3-10 倍的速度来渲染页面。如今，我们又对原来的架构设计做了一些改进，从而进一步提高了速度并节省了成本，这里我将和大家一起分享。

#### Original Parellel Architecture: Webpara

#### 原来的并行架构：Webpara

The original parallel architecture followed the master-worker pattern, and its design and parameters were chosen after a detailed benchmarking in production.  There were separate tiers of master and worker machines: the master machines were called webpara, and each ran 6 master Python processes; the worker machines were called webworker, and each ran 6 worker Python processes and one C++ fastrouter process. The master and worker Python processes ran our application logic with the paste web server. The fastrouter process is a custom-built multi-threaded queue server written in C++ which supports very fast enqueues and dequeues of arbitrary data.  Both tiers used the c1.xlarge EC2 instances on Amazon Web Services, which have 8 cores and 6MB shared CPU cache.

原来的并行架构设计服从主从模式，它的设计和参数是在生产环境中经过一系列基准测试后选定的。该设计中的主结点和从结点在不同的层级里：主结点机器我们称之为 webpara，每个结点上跑有 6 个 Python 主进程；从结点机器我们称之为 webworker，每个结点上有 6 个工作进程和一个用 C++ 语言编写的 fastrouter 进程。主进程和工作进程中的应用逻辑跑在相同的 web 服务器中，fastrouter 进程则是跑在一个用 C++ 定制的多线程队列服务器中，支持任意数据的快速入队和出队。这两层都是使用了AWS的 c1.xlarge EC2 实例，它们有 8 核和 6MB 的共享 CPU 高速缓存。


![img](https://qph.ec.quoracdn.net/main-qimg-59c04ce0ba1d9d74b445655ca645ee5c.webp)

An inbound request from our load balancer would reach an idle master process on some webpara machine. This master process will perform the work to render a small portion of the page before distributing the rest of the work to worker machines as sub-requests through the fastrouters running on them. Each worker will either fully complete the work needed to render the response of its sub-request or will prepare a partial response and further distribute work to other workers, possibly on different machines. Finally, the original master process would merge all sub-responses to create the full combined response for the client requesting the page. Because of the divide-and-conquer approach, and the ability to perform rendering work in parallel across many different worker machines, we could render pages very quickly.

一个来自负载均衡器的请求会被分配给某个 webpara 机器中的空闲主进程，这个进程会先完成一部分的页面渲染工作，然后再把剩余的工作作为子请求通过工作结点机器上的快速路由分发给对应的工作进程。每个工作进程可以完成子请求需要的所有响应，或者是准备一部分响应并进一步将工作分配给别的工作进程，甚至是不同机器上的工作进程。最终，由原始的主进程把工作进程返回的所有子响应合并，并对客户的页面请求发出响应。因为这种分治的思想，使得我们有能力在许多不同的工作机器上并行的工作，从而快速的进行页面渲染。

However, since the workers and masters were on different machines, all this communication happened over the network. Each sub-request and sub-response were being transmitted as payloads across tiers spanning many machines in our EC2 environment. Around December 2012, we started facing some scaling problems which led Quora to slow down despite our improved parallel architecture.

不过，因为工作进程和主进程是在不同的机器上，所以所有的通信都依赖于网络。每个子请求和子响应都需要作为有效载荷在我们的 EC2 环境里跨机器进行传输。2012 年 12 月左右，我们开始面临一些规模扩展问题，尽管我们使用了这种并行的架构设计，但对于Quora的访问依然慢了下来。

Our web traffic had grown large enough that it was beginning to challenge the limits of such a design -- even adding more masters and workers weren’t able to improve the situation. We realized that calls to our external services like Memcached and Redis were getting slower over time and we were bound on network I/O rather than on CPU at this point. There were too many network calls resulting in too many context switches.  

我们的网络流量已经增长到足以挑战这种设计的极限 — 即使增加主结点机器和从结点机器，也依然不能很好改变这种情况。而像 Memcached 和 Redis 这样的外部服务调用也会随着时间的推移变得越来越慢。随后，我们意识到，我们的瓶颈在于网络 I/O 而不是 CPU。太多的网络调用导致了大量的上下文切换。

So we had to find a way to decrease our network volume as well as number of network calls, all this without compromising CPU efficiency or increasing costs. To achieve these goals, we re-arranged various pieces of our architecture, collected detailed metrics through several A/B tests and chose the best possible configuration that could achieve those goals. The resulting architecture is what we call Ultralisk. [1]

因此，我们不得不找到一种方法去减少网络容量和网络调用的数量，而所有的这些都不能影响到 CPU 的执行效率或是增加成本。为了达到这个目标，我们重新安排了架构的各个部分，通过几次A / B测试收集了详细的指标，并选择了可能实现这些目标的最佳配置。由此产生的架构就是我们所说的 Ultralisk。[1]

#### New Parallel Architecture: Ultralisk

#### 新的并行架构：Ultralisk

This new Ultralisk architecture is homogeneous and consists of just one machine type -- the cc2.8xlarge instance on EC2. These are beastmachines with 32 cores, 20 MB shared CPU cache, around 10% faster CPUs, and more efficient HVM virtualization (as compared to other EC2 instance types). In addition, these machines have very high network I/O capacity.

这种新的 Ultralisk 架构是由同一种机器类型组成 — EC2 的 cc2.8xlarge 实例，这种类型的机器在硬件虚拟化这一方面会更有效率（与其他的 EC2 实例类型相比）。另外，这些机器有着很高的网络I/O容量。

This architecture still has a master-worker pattern identical to what previous architecture had. The main improvement in the Ultralisk architecture is that each Ultralisk machine houses both masters and workers. All sub-requests generated by a master are now processed by workers on the same machine and so communication between masters and workers doesn't have to be over external network. Since such communication for our parallel architecture accounted for 1.5 times all other network traffic, this change has reduced our external network traffic by over 60%.  Communication between masters and workers can be done simply over local TCP connections or Unix sockets. In our experiments, both of these protocols give similar performance with Unix sockets being just a tad faster. 

这个架构仍然与之前的架构一样采用主从模式。Ultralisk 架构主要的改进在于其中的每台物理机器同时拥有主进程和工作进程。所有由主进程生成的子请求都将交给同一台机器上的工作进程处理。因此，主进程和工作进程之间的通信不再依赖于外部网络。因为我们这种类型的通信是并行架构中的其他网络流量的 1.5 倍，所以这种变化使我们的外部网络流量减少了超过 60% 。主进程和工作进程之前的通信可以很简单的基于 TCP 连接或者 Unix 套接字实现。在我们的实验中，这两种实现方式的性能很接近，Unix套接字只是稍快一些。

![img](https://qph.ec.quoracdn.net/main-qimg-0a9c167ab5989be459405ee953d07d93.webp)

The number of masters and workers on a machine is crucial to the overall performance. If there are too few workers, we won’t achieve a high degree of parallelism resulting in a slow down; and if they are too many, they would cause too much context switching. Similarly there is a tradeoff between cost and speed in deciding the number of masters. Through our experiments, we have found that the optimal configuration is to run 6 masters, 27 workers and 1 fastrouter on each machine. CPU affinities are set in a way such that each worker and fastrouter process are pinned to their own CPU core. Two masters share a core each (so 6 masters occupy 3 cores) and the last remaining core left is for kernel processes, including but not limited to, handling all I/O interrupts. [2]

每台物理机器上主进程和工作进程的个数是整体性能的关键。如果工作进程太少，我们就不能很好的利用并行从而导致系统变慢；如果工作进程太多，会导致太多的上下文切换。同样，在决定主进程的个数时，则需要在成本和速度之间进一步权衡。经过我们的实验，我们发现一个优化的配置是在每台物理机器上跑 6 个主进程，27 个工作进程和 1 个快速路由。CPU 的亲和性设置可以使得每个工作进程和快速路由进程跑在固定的 CPU 内核上。两个主进程共享一个核（因此 6 个主进程需要占用 3 个内核），剩下的核留给系统的内核进程，包括处理所有的 I/O中断，但并不仅限于此。[2]

This architecture reduced network I/O volume and contention for I/O resources making calls to external services much faster. Here is a graph which shows improvements in 99th percentile response time (in ms) for Memcached calls. 

这种架构减少了网络 I/O 数量和 I/O 资源的占用，从而使得外部服务的调用变得更快。下图给出了在 Memcached 调用时请求响应时间（毫秒）在 99% 分位处的改进。

![img](https://qph.ec.quoracdn.net/main-qimg-98a5bf77ff499eddc720d03d93d7f11b.webp)

Overall, we have found this architecture to perform 20-35% faster than our old one. Here is a graph which shows ratio of the median and the 90th percentile response times on both architectures. 

总体来看，新的架构比之前的要快上 20-35%。下图给出了两种架构体系中请求响应时间在中位数与 90% 分位处的比率。

![img](https://qph.ec.quoracdn.net/main-qimg-8379e9d6cef7ea5394350cf85dc8d1ca.webp)

To conclude, the Ultralisk architecture is faster, cheaper, less contentious on network resources, and much more scalable than our old Webpara architecture. Also, since each web server machine is now homogeneous and completely independent of all other machines, and we know exactly how many requests each machine can handle per second, we can now horizontally scale our web servers, too, in an automated way.  Furthermore, this new architecture has allowed us to reduce the number of web servers we need to manage by a factor of 4, allowing for greater ease in deployment and manageability.

总而言之，Ultralisk 架构更快，更便宜，对网络资源的占用更少，而且比我们之前的 Webpara 架构更有扩展性。另外，现在每台机器都是相同的配置，完全与其他所有的机器独立，而我们也精确的知道每台机器每秒可以处理多少请求，所以我们可以以自动的方式来水平扩展我们的 Web 服务器。此外，这种新架构将我们需要管理的机器数量减少为原来的四分之一，从而可以更轻松的管理和部署。

We fully deployed this new architecture to production in February and we haven’t seen any problems so far despite our exponentially increasing web traffic. As always, performance is of paramount importance to us and our speed and infrastructure teams are dedicated towards making Quora the best possible experience for all of  you.

我们在二月份已经把这种新架构完全部署到了生产环境，尽管我们的网络流量呈指数级增长，但目前还没有看到任何问题。一如既往，性能对我们至关重要，我们的速度和基础架构团队致力于让Quora给你更好的体验。

Nikhil Garg is a Quora engineer and works on the speed team.
Nikhil Garg 是一名 Quora 工程师，并供职于速度团队。



[1] Ultralisks are one of the largest and most powerful ground forces of Zerg. They are also very costly. Their genetic material has been subjected to countless tests and experiments and only the final viable code is used by zerg larvae. 

[1] Ultralisk 是虫族最大和最强力的地面部队之一。它们非常昂贵。它们的基因物质已经经过无数次的测试和实验，而最后的可行版本只能用于虫族的幼虫。

[2] Setting correct CPU affinities was crucial for performance gains. Since then, we've seen gains by setting correct CPU affinities on machines from different tiers as well.

[2] CPU亲和性的正确设置是性能收益的关键。从那以后，我们也开始在不同层次的机器上设置 CPU 亲和性，而这样也确实带来了一定的收益。