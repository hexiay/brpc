# Table of Content

       atomic_instructions.md
       auto_concurrency_limiter.md
       avalanche.md
       backup_request.md
       baidu_std.md
       benchmark.md
       benchmark_http.md
       bthread.md
       bthread_id.md
       bthread_or_not.md
       builtin_service.md
       bvar.md
       bvar_c++.md
       case_apicontrol.md
       case_baidu_dsp.md
       case_elf.md
       case_ubrpc.md
       client.md
       combo_channel.md
       connections.md
       consistent_hashing.md
       contention_profiler.md
       cpu_profiler.md
       dummy_server.md
       error_code.md
       execution_queue.md
       flags.md
       flatmap.md
       getting_started.md
       heap_profiler.md
       http_client.md
       http_derivatives.md
       http_service.md
       io.md
       iobuf.md
       json2pb.md
       lalb.md
       load_balancing.md
       memcache_client.md
       memory_management.md
       new_protocol.md
       nshead_service.md
       overview.md
       parallel_http.md
       redis_client.md
       rpc_press.md
       rpc_replay.md
       rpc_view.md
       rpcz.md
       server.md
       server_debugging.md
       server_push.md
       status.md
       streaming_log.md
       streaming_rpc.md
       thread_local.md
       threading_overview.md
       thrift.md
       timer_keeping.md
       ub_client.md
       vars.md

# ========= ./atomic_instructions.md ========
[English version](../en/atomic_instructions.md)

我们都知道多核编程常用锁避免多个线程在修改同一个数据时产生[race condition](http://en.wikipedia.org/wiki/Race_condition)。当锁成为性能瓶颈时，我们又总想试着绕开它，而不可避免地接触了原子指令。但在实践中，用原子指令写出正确的代码是一件非常困难的事，琢磨不透的race condition、[ABA problem](https://en.wikipedia.org/wiki/ABA_problem)、[memory fence](https://en.wikipedia.org/wiki/Memory_barrier)很烧脑，这篇文章试图通过介绍[SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)架构下的原子指令帮助大家入门。C++11正式引入了[原子指令](http://en.cppreference.com/w/cpp/atomic/atomic)，我们就以其语法描述。

顾名思义，原子指令是**对软件**不可再分的指令，比如x.fetch_add(n)指原子地给x加上n，这个指令**对软件**要么没做，要么完成，不会观察到中间状态。常见的原子指令有：

| 原子指令 (x均为std::atomic<int>)               | 作用                                       |
| ---------------------------------------- | ---------------------------------------- |
| x.load()                                 | 返回x的值。                                   |
| x.store(n)                               | 把x设为n，什么都不返回。                            |
| x.exchange(n)                            | 把x设为n，返回设定之前的值。                          |
| x.compare_exchange_strong(expected_ref, desired) | 若x等于expected_ref，则设为desired，返回成功；否则把最新值写入expected_ref，返回失败。 |
| x.compare_exchange_weak(expected_ref, desired) | 相比compare_exchange_strong可能有[spurious wakeup](http://en.wikipedia.org/wiki/Spurious_wakeup)。 |
| x.fetch_add(n), x.fetch_sub(n)           | 原子地做x += n, x-= n，返回修改之前的值。              |

你已经可以用这些指令做原子计数，比如多个线程同时累加一个原子变量，以统计这些线程对一些资源的操作次数。但是，这可能会有两个问题：

- 这个操作没有你想象地快。
- 如果你尝试通过看似简单的原子操作控制对一些资源的访问，你的程序有很大几率会crash。

# Cacheline

没有任何竞争或只被一个线程访问的原子操作是比较快的，“竞争”指的是多个线程同时访问同一个[cacheline](https://en.wikipedia.org/wiki/CPU_cache#Cache_entries)。现代CPU为了以低价格获得高性能，大量使用了cache，并把cache分了多级。百度内常见的Intel E5-2620拥有32K的L1 dcache和icache，256K的L2 cache和15M的L3 cache。其中L1和L2 cache为每个核心独有，L3则所有核心共享。一个核心写入自己的L1 cache是极快的(4 cycles, ~2ns)，但当另一个核心读或写同一处内存时，它得确认看到其他核心中对应的cacheline。对于软件来说，这个过程是原子的，不能在中间穿插其他代码，只能等待CPU完成[一致性同步](https://en.wikipedia.org/wiki/Cache_coherence)，这个复杂的硬件算法使得原子操作会变得很慢，在E5-2620上竞争激烈时fetch_add会耗费700纳秒左右。访问被多个线程频繁共享的内存往往是比较慢的。比如像一些场景临界区看着很小，但保护它的spinlock性能不佳，因为spinlock使用的exchange, fetch_add等指令必须等待最新的cacheline，看上去只有几条指令，花费若干微秒并不奇怪。

要提高性能，就要避免让CPU频繁同步cacheline。这不单和原子指令本身的性能有关，还会影响到程序的整体性能。最有效的解决方法很直白：**尽量避免共享**。

- 一个依赖全局多生产者多消费者队列(MPMC)的程序难有很好的多核扩展性，因为这个队列的极限吞吐取决于同步cache的延时，而不是核心的个数。最好是用多个SPMC或多个MPSC队列，甚至多个SPSC队列代替，在源头就规避掉竞争。
- 另一个例子是计数器，如果所有线程都频繁修改一个计数器，性能就会很差，原因同样在于不同的核心在不停地同步同一个cacheline。如果这个计数器只是用作打打日志之类的，那我们完全可以让每个线程修改thread-local变量，在需要时再合并所有线程中的值，性能可能有[几十倍的差别](bvar.md)。

一个相关的编程陷阱是false sharing：对那些不怎么被修改甚至只读变量的访问，由于同一个cacheline中的其他变量被频繁修改，而不得不经常等待cacheline同步而显著变慢了。多线程中的变量尽量按访问规律排列，频繁被其他线程修改的变量要放在独立的cacheline中。要让一个变量或结构体按cacheline对齐，可以include \<butil/macros.h\>后使用BAIDU_CACHELINE_ALIGNMENT宏，请自行grep brpc的代码了解用法。

# Memory fence

仅靠原子技术实现不了对资源的访问控制，即使简单如[spinlock](https://en.wikipedia.org/wiki/Spinlock)或[引用计数](https://en.wikipedia.org/wiki/Reference_counting)，看上去正确的代码也可能会crash。这里的关键在于**重排指令**导致了读写顺序的变化。只要没有依赖，代码中在后面的指令就可能跑到前面去，[编译器](http://preshing.com/20120625/memory-ordering-at-compile-time/)和[CPU](https://en.wikipedia.org/wiki/Out-of-order_execution)都会这么做。

这么做的动机非常自然，CPU要尽量塞满每个cycle，在单位时间内运行尽量多的指令。如上节中提到的，访存指令在等待cacheline同步时要花费数百纳秒，最高效地自然是同时同步多个cacheline，而不是一个个做。一个线程在代码中对多个变量的依次修改，可能会以不同的次序同步到另一个线程所在的核心上。不同线程对数据的需求不同，按需同步也会导致cacheline的读序和写序不同。

如果其中第一个变量扮演了开关的作用，控制对后续变量的访问。那么当这些变量被一起同步到其他核心时，更新顺序可能变了，第一个变量未必是第一个更新的，然而其他线程还认为它代表着其他变量有效，去访问了实际已被删除的变量，从而导致未定义的行为。比如下面的代码片段：

```c++
// Thread 1
// ready was initialized to false
p.init();
ready = true;
```

```c++
// Thread2
if (ready) {
    p.bar();
}
```
从人的角度，这是对的，因为线程2在ready为true时才会访问p，按线程1的逻辑，此时p应该初始化好了。但对多核机器而言，这段代码可能难以正常运行：

- 线程1中的ready = true可能会被编译器或cpu重排到p.init()之前，从而使线程2看到ready为true时，p仍然未初始化。这种情况同样也会在线程2中发生，p.bar()中的一些代码可能被重排到检查ready之前。
- 即使没有重排，ready和p的值也会独立地同步到线程2所在核心的cache，线程2仍然可能在看到ready为true时看到未初始化的p。

注：x86/x64的load带acquire语意，store带release语意，上面的代码刨除编译器和CPU因素可以正确运行。

通过这个简单例子，你可以窥见原子指令编程的复杂性了吧。为了解决这个问题，CPU和编译器提供了[memory fence](http://en.wikipedia.org/wiki/Memory_barrier)，让用户可以声明访存指令间的可见性(visibility)关系，boost和C++11对memory fence做了抽象，总结为如下几种[memory order](http://en.cppreference.com/w/cpp/atomic/memory_order).

| memory order         | 作用                                       |
| -------------------- | ---------------------------------------- |
| memory_order_relaxed | 没有fencing作用                              |
| memory_order_consume | 后面依赖此原子变量的访存指令勿重排至此条指令之前                 |
| memory_order_acquire | 后面访存指令勿重排至此条指令之前                         |
| memory_order_release | 前面访存指令勿重排至此条指令之后。当此条指令的结果对其他线程可见后，之前的所有指令都可见 |
| memory_order_acq_rel | acquire + release语意                      |
| memory_order_seq_cst | acq_rel语意外加所有使用seq_cst的指令有严格地全序关系        |

有了memory order，上面的例子可以这么更正：

```c++
// Thread1
// ready was initialized to false
p.init();
ready.store(true, std::memory_order_release);
```

```c++
// Thread2
if (ready.load(std::memory_order_acquire)) {
    p.bar();
}
```

线程2中的acquire和线程1的release配对，确保线程2在看到ready==true时能看到线程1 release之前所有的访存操作。

注意，memory fence不等于可见性，即使线程2恰好在线程1在把ready设置为true后读取了ready也不意味着它能看到true，因为同步cache是有延时的。memory fence保证的是可见性的顺序：“假如我看到了a的最新值，那么我一定也得看到b的最新值”。

一个相关问题是：如何知道看到的值是新还是旧？一般分两种情况：

- 值是特殊的。比如在上面的例子中，ready=true是个特殊值，只要线程2看到ready为true就意味着更新了。只要设定了特殊值，读到或没有读到特殊值都代表了一种含义。
- 总是累加。一些场景下没有特殊值，那我们就用fetch_add之类的指令累加一个变量，只要变量的值域足够大，在很长一段时间内，新值和之前所有的旧值都会不相同，我们就能区分彼此了。

原子指令的例子可以看boost.atomic的[Example](http://www.boost.org/doc/libs/1_56_0/doc/html/atomic/usage_examples.html)，atomic的官方描述可以看[这里](http://en.cppreference.com/w/cpp/atomic/atomic)。

# wait-free & lock-free

原子指令能为我们的服务赋予两个重要属性：[wait-free](http://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom)和[lock-free](http://en.wikipedia.org/wiki/Non-blocking_algorithm#Lock-freedom)。前者指不管OS如何调度线程，每个线程都始终在做有用的事；后者比前者弱一些，指不管OS如何调度线程，至少有一个线程在做有用的事。如果我们的服务中使用了锁，那么OS可能把一个刚获得锁的线程切换出去，这时候所有依赖这个锁的线程都在等待，而没有做有用的事，所以用了锁就不是lock-free，更不会是wait-free。为了确保一件事情总在确定时间内完成，实时系统的关键代码至少是lock-free的。在百度广泛又多样的在线服务中，对时效性也有着严苛的要求，如果RPC中最关键的部分满足wait-free或lock-free，就可以提供更稳定的服务质量。事实上，brpc中的读写都是wait-free的，具体见[IO](io.md)。

值得提醒的是，常见想法是lock-free或wait-free的算法会更快，但事实可能相反，因为：

- lock-free和wait-free必须处理更多更复杂的race condition和ABA problem，完成相同目的的代码比用锁更复杂。代码越多，耗时就越长。
- 使用mutex的算法变相带“后退”效果。后退(backoff)指出现竞争时尝试另一个途径以临时避免竞争，mutex出现竞争时会使调用者睡眠，使拿到锁的那个线程可以很快地独占完成一系列流程，总体吞吐可能反而高了。

mutex导致低性能往往是因为临界区过大（限制了并发度），或竞争过于激烈（上下文切换开销变得突出）。lock-free/wait-free算法的价值在于其保证了一个或所有线程始终在做有用的事，而不是绝对的高性能。但在一种情况下lock-free和wait-free算法的性能多半更高：就是算法本身可以用少量原子指令实现。实现锁也是要用原子指令的，当算法本身用一两条指令就能完成的时候，相比额外用锁肯定是更快了。


# ========= ./auto_concurrency_limiter.md ========
# 自适应限流

服务的处理能力是有客观上限的。当请求速度超过服务的处理速度时，服务就会过载。

如果服务持续过载，会导致越来越多的请求积压，最终所有的请求都必须等待较长时间才能被处理，从而使整个服务处于瘫痪状态。

与之相对的，如果直接拒绝掉一部分请求，反而能够让服务能够"及时"处理更多的请求。对应的方法就是[设置最大并发](https://github.com/brpc/brpc/blob/master/docs/cn/server.md#%E9%99%90%E5%88%B6%E6%9C%80%E5%A4%A7%E5%B9%B6%E5%8F%91)。

自适应限流能动态调整服务的最大并发，在保证服务不过载的前提下，让服务尽可能多的处理请求。

## 使用场景
通常情况下要让服务不过载，只需在上线前进行压力测试，并通过little's law计算出best_max_concurrency就可以了。但在服务数量多，拓扑复杂，且处理能力会逐渐变化的局面下，使用固定的最大并发会带来巨大的测试工作量，很不方便。自适应限流就是为了解决这个问题。

使用自适应限流前建议做到：
1. 客户端开启了重试功能。

2. 服务端有多个节点。

这样当一个节点返回过载时，客户端可以向其他的节点发起重试，从而尽量不丢失流量。

## 开启方法
目前只有method级别支持自适应限流。如果要为某个method开启自适应限流，只需要将它的最大并发设置为"auto"即可。

```c++
// Set auto concurrency limiter for all methods
brpc::ServerOptions options;
options.method_max_concurrency = "auto";

// Set auto concurrency limiter for specific method
server.MaxConcurrencyOf("example.EchoService.Echo") = "auto";
```

## 原理

### 名词
**concurrency**: 同时处理的请求数，又被称为“并发度”。

**max_concurrency**: 设置的最大并发度。超过并发的请求会被拒绝（返回ELIMIT错误），在集群层面，client应重试到另一台server上去。

**best_max_concurrency**: 并发的物理含义是任务处理槽位，天然存在上限，这个上限就是best_max_concurrency。若max_concurrency设置的过大，则concurrency可能大于best_max_concurrency，任务将无法被及时处理而暂存在各种队列中排队，系统也会进入拥塞状态。若max_concurrency设置的过小，则concurrency总是会小于best_max_concurrency，限制系统达到本可以达到的更高吞吐。

**noload_latency**: 单纯处理任务的延时，不包括排队时间。另一种解释是低负载的延时。由于正确处理任务得经历必要的环节，其中会耗费cpu或等待下游返回，noload_latency是一个服务固有的属性，但可能随时间逐渐改变（由于内存碎片，压力变化，业务数据变化等因素）。

**min_latency**: 实际测定的latency中的较小值的ema，当concurrency不大于best_max_concurrency时，min_latency和noload_latency接近(可能轻微上升）。

**peak_qps**: qps的上限。注意是处理或回复的qps而不是接收的qps。值取决于best_max_concurrency / noload_latency，这两个量都是服务的固有属性，故peak_qps也是服务的固有属性，和拥塞状况无关，但可能随时间逐渐改变。

**max_qps**: 实际测定的qps中的较大值。由于qps具有上限，max_qps总是会小于peak_qps，不论拥塞与否。

### Little's Law
在服务处于稳定状态时: concurrency = latency * qps。 这是自适应限流的理论基础。

当服务没有超载时，随着流量的上升，latency基本稳定(接近noload_latency)，qps和concurrency呈线性关系一起上升。

当流量超过服务的peak_qps时，则concurrency和latency会一起上升，而qps会稳定在peak_qps。

假如一个服务的peak_qps和noload_latency都比较稳定，那么它的best_max_concurrency = noload_latency * peak_qps。

自适应限流就是要找到服务的noload_latency和peak_qps， 并将最大并发设置为靠近两者乘积的一个值。

### 计算公式

自适应限流会不断的对请求进行采样，当采样窗口的样本数量足够时，会根据样本的平均延迟和服务当前的qps计算出下一个采样窗口的max_concurrency:

> max_concurrency = max_qps * ((2+alpha) * min_latency - latency)

alpha为可接受的延时上升幅度，默认0.3。

latency是当前采样窗口内所有请求的平均latency。

max_qps是最近一段时间测量到的qps的极大值。

min_latency是最近一段时间测量到的latency较小值的ema，是noload_latency的估算值。

当服务处于低负载时，min_latency约等于noload_latency，此时计算出来的max_concurrency会高于concurrency，但低于best_max_concurrency，给流量上涨留探索空间。而当服务过载时，服务的qps约等于max_qps，同时latency开始明显超过min_latency，此时max_concurrency则会接近concurrency，并通过定期衰减避免远离best_max_concurrency，保证服务不会过载。


### 估算noload_latency
服务的noload_latency并非是一成不变的，自适应限流必须能够正确的探测noload_latency的变化。当noload_latency下降时，是很容感知到的，因为这个时候latency也会下降。难点在于当latency上涨时，需要能够正确的辨别到底是服务过载了，还是noload_latency上涨了。

可能的方案有：
1. 取最近一段时间的最小latency来近似noload_latency
2. 取最近一段时间的latency的各种平均值来预测noload_latency
3. 收集请求的平均排队等待时间，使用latency - queue_time作为noload_latency
4. 每隔一段时间缩小max_concurrency，过一小段时间后以此时的latency作为noload_latency

方案1和方案2的问题在于：假如服务持续处于高负载，那么最近的所有latency都会高出noload_latency，从而使得算法估计的noload_latency不断升高。

方案3的问题在于，假如服务的性能瓶颈在下游服务，那么请求在服务本身的排队等待时间无法反应整体的负载情况。

方案4是最通用的，也经过了大量实验的考验。缩小max_concurrency和公式中的alpha存在关联。让我们做个假想实验，若latency极为稳定并都等于min_latency，那么公式简化为max_concurrency = max_qps * latency * (1 + alpha)。根据little's law，qps最多为max_qps * (1 + alpha). alpha是qps的"探索空间"，若alpha为0，则qps被锁定为max_qps，算法可能无法探索到peak_qps。但在qps已经达到peak_qps时，alpha会使延时上升（已拥塞），此时测定的min_latency会大于noload_latency，一轮轮下去最终会导致min_latency不收敛。定期降低max_concurrency就是阻止这个过程，并给min_latency下降提供"探索空间"。

#### 减少重测时的流量损失

每隔一段时间，自适应限流算法都会缩小max_concurrency，并持续一段时间，然后将此时的latency作为服务的noload_latency，以处理noload_latency上涨了的情况。测量noload_latency时，必须让先服务处于低负载的状态，因此对max_concurrency的缩小是难以避免的。

由于max_concurrency < concurrency时，服务会拒绝掉所有的请求，限流算法将"排空所有的经历过排队等待的请求的时间" 设置为 latency * 2 ，以确保用于计算min_latency的样本绝大部分都是没有经过排队等待的。

由于服务的latency通常都不会太长，这种做法所带来的流量损失也很小。

#### 应对抖动
即使服务自身没有过载，latency也会发生波动，根据Little's Law，latency的波动会导致server的concurrency发生波动。

我们在设计自适应限流的计算公式时，考虑到了latency发生抖动的情况:
当latency与min_latency很接近时，根据计算公式会得到一个较高max_concurrency来适应concurrency的波动，从而尽可能的减少“误杀”。同时，随着latency的升高，max_concurrency会逐渐降低，以保护服务不会过载。

从另一个角度来说，当latency也开始升高时，通常意味着某处(不一定是服务本身，也有可能是下游服务)消耗了大量CPU资源，这个时候缩小max_concurrency也是合理的。

#### 平滑处理
为了减少个别窗口的抖动对限流算法的影响，同时尽量降低计算开销，计算min_latency时会通过使用[EMA](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average)来进行平滑处理：

```
if latency > min_latency:
    min_latency = latency * ema_alpha  + (1 - ema_alpha) * min_latency
else:
    do_nothing
```

### 估算peak_qps

#### 提高qps增长的速度
当服务启动时，由于服务本身需要进行一系列的初始化，tcp本身也有慢启动等一系列原因。服务在刚启动时的qps一定会很低。这就导致了服务启动时的max_concurrency也很低。而按照上面的计算公式，当max_concurrency很低的时候，预留给qps增长的冗余concurrency也很低(即：alpha * max_qps * min_latency)。从而会影响当流量增加时，服务max_concurrency的增加速度。

假如从启动到打满qps的时间过长，这期间会损失大量流量。在这里我们采取的措施有两个，

1. 采样方面，一旦采到的请求数量足够多，直接提交当前采样窗口，而不是等待采样窗口的到时间了才提交
2. 计算公式方面，当current_qps > 保存的max_qps时，直接进行更新，不进行平滑处理。

在进行了这两个处理之后，绝大部分情况下都能够在2秒左右将qps打满。

#### 平滑处理
为了减少个别窗口的抖动对限流算法的影响，同时尽量降低计算开销，在计算max_qps时，会通过使用[EMA](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average)来进行平滑处理：

```    
if current_qps > max_qps:
    max_qps = current_qps
else: 
    max_qps = current_qps * ema_alpha / 10 + (1 - ema_alpha / 10) * max_qps
```
将max_qps的ema参数置为min_latency的ema参数的十分之一的原因是: max_qps 下降了通常并不意味着极限qps也下降了。而min_latency下降了，通常意味着noload_latency确实下降了。

### 与netflix gradient算法的对比

netflix中的gradient算法公式为：max_concurrency = min_latency / latency * max_concurrency + queue_size。

其中latency是采样窗口的最小latency，min_latency是最近多个采样窗口的最小latency。min_latency / latency就是算法中的"梯度"，当latency大于min_latency时，max_concurrency会逐渐减少；反之，max_concurrency会逐渐上升，从而让max_concurrency围绕在best_max_concurrency附近。

这个公式可以和本文的算法进行类比：

* gradient算法中的latency和本算法的不同，前者的latency是最小值，后者是平均值。netflix的原意是最小值能更好地代表noload_latency，但实际上只要不对max_concurrency做定期衰减，不管最小值还是平均值都有可能不断上升使算法不收敛。最小值并不能带来额外的好处，反而会使算法更不稳定。
* gradient算法中的max_concurrency / latency从概念上和qps有关联（根据little's law)，但可能严重脱节。比如在重测
min_latency前，若所有latency都小于min_latency，那么max_concurrency会不断下降甚至到0；但按照本算法，max_qps和min_latency仍然是稳定的，它们计算出的max_concurrency也不会剧烈变动。究其本质，gradient算法在迭代max_concurrency时，latency并不能代表实际并发为max_concurrency时的延时，两者是脱节的，所以max_concurrency / latency的实际物理含义不明，与qps可能差异甚大，最后导致了很大的偏差。
* gradient算法的queue_size推荐为sqrt(max_concurrency)，这是不合理的。netflix对queue_size的理解大概是代表各种不可控环节的缓存，比如socket里的，和max_concurrency存在一定的正向关系情有可原。但在我们的理解中，这部分queue_size作用微乎其微，没有或用常量即可。我们关注的queue_size是给concurrency上升留出的探索空间: max_concurrency的更新是有延迟的，在并发从低到高的增长过程中，queue_size的作用就是在max_concurrency更新前不限制qps上升。而当concurrency高时，服务可能已经过载了，queue_size就应该小一点，防止进一步恶化延时。这里的queue_size和并发是反向关系。


# ========= ./avalanche.md ========
“雪崩”指的是访问服务集群时绝大部分请求都超时，且在流量减少时仍无法恢复的现象。下面解释这个现象的来源。

当流量超出服务的最大qps时，服务将无法正常服务；当流量恢复正常时（小于服务的处理能力），积压的请求会被处理，虽然其中很大一部分可能会因为处理的不及时而超时，但服务本身一般还是会恢复正常的。这就相当于一个水池有一个入水口和一个出水口，如果入水量大于出水量，水池子终将盛满，多出的水会溢出来。但如果入水量降到出水量之下，一段时间后水池总会排空。雪崩并不是单一服务能产生的。

如果一个请求经过两个服务，情况就有所不同了。比如请求访问A服务，A服务又访问了B服务。当B被打满时，A处的client会大量超时，如果A处的client在等待B返回时也阻塞了A的服务线程（常见），且使用了固定个数的线程池（常见），那么A处的最大qps就从**线程数 / 平均延时**，降到了**线程数 / 超时**。由于超时往往是平均延时的3~4倍，A处的最大qps会相应地下降3~4倍，从而产生比B处更激烈的拥塞。如果A还有类似的上游，拥塞会继续传递上去。但这个过程还是可恢复的。B处的流量终究由最前端的流量触发，只要最前端的流量回归正常，B处的流量总会慢慢降下来直到能正常回复大多数请求，从而让A恢复正常。

但有两个例外：

1. A可能对B发起了过于频繁的基于超时的重试。这不仅会让A的最大qps降到**线程数 / 超时**，还会让B处的qps翻**重试次数**倍。这就可能陷入恶性循环了：只要**线程数 / 超时 \* 重试次数**大于B的最大qps**，**B就无法恢复 -> A处的client会继续超时 -> A继续重试 -> B继续无法恢复。
2. A或B没有限制某个缓冲或队列的长度，或限制过于宽松。拥塞请求会大量地积压在那里，要恢复就得全部处理完，时间可能长得无法接受。由于有限长的缓冲或队列需要在填满时解决等待、唤醒等问题，有时为了简单，代码可能会假定缓冲或队列不会满，这就埋下了种子。即使队列是有限长的，恢复时间也可能很长，因为清空队列的过程是个追赶问题，排空的时间取决于**积压的请求数 / (最大qps - 当前qps)**，如果当前qps和最大qps差的不多，积压的请求又比较多，那排空时间就遥遥无期了。

了解这些因素后可以更好的理解brpc中相关的设计。

1. 拥塞时A服务最大qps的跳变是因为线程个数是**硬限**，单个请求的处理时间很大程度上决定了最大qps。而brpc server端默认在bthread中处理请求，个数是软限，单个请求超时只是阻塞所在的bthread，并不会影响为新请求建立新的bthread。brpc也提供了完整的异步接口，让用户可以进一步提高io-bound服务的并发度，降低服务被打满的可能性。
2. brpc中[重试](client.md#重试)默认只在连接出错时发起，避免了流量放大，这是比较有效率的重试方式。如果需要基于超时重试，可以设置[backup request](client.md#重试)，这类重试最多只有一次，放大程度降到了最低。brpc中的RPC超时是deadline，超过后RPC一定会结束，这让用户对服务的行为有更好的预判。在之前的一些实现中，RPC超时是单次超时*重试次数，在实践中容易误判。
3. brpc server端的[max_concurrency选项](server.md#限制最大并发)控制了server的最大并发：当同时处理的请求数超过max_concurrency时，server会回复client错误，而不是继续积压。这一方面在服务开始的源头控制住了积压的请求数，尽量避免延生到用户缓冲或队列中，另一方面也让client尽快地去重试其他server，对集群来说是个更好的策略。

对于brpc的用户来说，要防止雪崩，主要注意两点：

1. 评估server的最大并发，设置合理的max_concurrency值。这个默认是不设的，也就是不限制。无论程序是同步还是异步，用户都可以通过 **最大qps \* 非拥塞时的延时**（秒）来评估最大并发，原理见[little's law](https://en.wikipedia.org/wiki/Little%27s_law)，这两个量都可以在brpc中的内置服务中看到。max_concurrency与最大并发相等或大一些就行了。
2. 注意考察重试发生时的行为，特别是在定制RetryPolicy时。如果你只是用默认的brpc重试，一般是安全的。但用户程序也常会自己做重试，比如通过一个Channel访问失败后，去访问另外一个Channel，这种情况下要想清楚重试发生时最差情况下请求量会放大几倍，服务是否可承受。


# ========= ./backup_request.md ========
有时为了保证可用性，需要同时访问两路服务，哪个先返回就取哪个。在brpc中，这有多种做法：

# 当后端server可以挂在一个命名服务内时

Channel开启backup request。这个Channel会先向其中一个server发送请求，如果在ChannelOptions.backup_request_ms后还没回来，再向另一个server发送。之后哪个先回来就取哪个。在设置了合理的backup_request_ms后，大部分时候只会发一个请求，对后端服务只有一倍压力。

示例代码见[example/backup_request_c++](https://github.com/brpc/brpc/blob/master/example/backup_request_c++)。这个例子中，client设定了在2ms后发送backup request，server在碰到偶数位的请求后会故意睡眠20ms以触发backup request。

运行后，client端和server端的日志分别如下，“index”是请求的编号。可以看到server端在收到第一个请求后会故意sleep 20ms，client端之后发送另一个同样index的请求，最终的延时并没有受到故意sleep的影响。

![img](../images/backup_request_1.png)

![img](../images/backup_request_2.png)

/rpcz也显示client在2ms后触发了backup超时并发出了第二个请求。

![img](../images/backup_request_3.png)

## 选择合理的backup_request_ms

可以观察brpc默认提供的latency_cdf图，或自行添加。cdf图的y轴是延时（默认微秒），x轴是小于y轴延时的请求的比例。在下图中，选择backup_request_ms=2ms可以大约覆盖95.5%的请求，选择backup_request_ms=10ms则可以覆盖99.99%的请求。

![img](../images/backup_request_4.png)

自行添加的方法：

```c++
#include <bvar/bvar.h>
#include <butil/time.h>
...
bvar::LatencyRecorder my_func_latency("my_func");
...
butil::Timer tm;
tm.start();
my_func();
tm.stop();
my_func_latency << tm.u_elapsed();  // u代表微秒，还有s_elapsed(), m_elapsed(), n_elapsed()分别对应秒，毫秒，纳秒。
 
// 好了，在/vars中会显示my_func_qps, my_func_latency, my_func_latency_cdf等很多计数器。
```

# 当后端server不能挂在一个命名服务内时

【推荐】建立一个开启backup request的SelectiveChannel，其中包含两个sub channel。访问这个SelectiveChannel和上面的情况类似，会先访问一个sub channel，如果在ChannelOptions.backup_request_ms后没返回，再访问另一个sub channel。如果一个sub channel对应一个集群，这个方法就是在两个集群间做互备。SelectiveChannel的例子见[example/selective_echo_c++](https://github.com/brpc/brpc/tree/master/example/selective_echo_c++)，具体做法请参考上面的过程。

【不推荐】发起两个异步RPC后Join它们，它们的done内是相互取消的逻辑。示例代码见[example/cancel_c++](https://github.com/brpc/brpc/tree/master/example/cancel_c++)。这种方法的问题是总会发两个请求，对后端服务有两倍压力，这个方法怎么算都是不经济的，你应该尽量避免用这个方法。


# ========= ./baidu_std.md ========
# 介绍

baidu_std是一种基于TCP协议的二进制RPC通信协议。它以Protobuf作为基本的数据交换格式，并基于Protobuf内置的RPC Service形式，规定了通信双方之间的数据交换协议，以实现完整的RPC调用。

baidu_std不考虑跨TCP连接的情况。

# 基本协议

## 服务

所有RPC服务都在某个IP地址上通过某个端口发布。

一个端口可以同时发布多个服务。服务以名字标识。服务名必须是UpperCamelCase，由大小写字母和数字组成，长度不超过64个字符。

一个服务可以包含一个或多个方法。每个方法以名字标识，由大小写字母、数字和下划线组成，长度不超过64个字符。考虑到不同语言风格差异较大，这里对方法命名格式不做强制规定。

四元组唯一地标识了一个RPC方法。

调用方法所需参数应放在一个Protobuf消息内。如果方法有返回结果，也同样应放在一个Protobuf消息内。具体定义由通信双方自行约定。特别地，可以使用空的Protobuf消息来表示请求/响应为空的情况。

## 包

包是baidu_std的基本数据交换单位。每个包由包头和包体组成，其中包体又分为元数据、数据、附件三部分。具体的参数和返回结果放在数据部分。

包分为请求包和响应包两种。它们的包头格式一致，但元数据部分的定义不同。

## 包头

包头长度固定为12字节。前四字节为协议标识PRPC，中间四字节是一个32位整数，表示包体长度（不包括包头的12字节），最后四字节是一个32位整数，表示包体中的元数据包长度。整数均采用网络字节序表示。

## 元数据

元数据用于描述请求/响应。

```
message RpcMeta {
    optional RpcRequestMeta request = 1;
    optional RpcResponseMeta response = 2;
    optional int32 compress_type = 3;
    optional int64 correlation_id = 4;
    optional int32 attachment_size = 5;
    optional ChunkInfo chuck_info = 6;
    optional bytes authentication_data = 7;
};
```

请求包中只有request，响应包中只有response。实现可以根据域的存在性来区分请求包和响应包。

| 参数                  | 说明                                       |
| ------------------- | ---------------------------------------- |
| request             | 请求包元数据                                   |
| response            | 响应包元数据                                   |
| compress_type       | 详见附录[压缩算法](#compress_algorithm)          |
| correlation_id      | 请求包中的该域由请求方设置，用于唯一标识一个RPC请求。请求方有义务保证其唯一性，协议本身对此不做任何检查。响应方需要在对应的响应包里面将correlation_id设为同样的值。 |
| attachment_size     | 附件大小，详见[附件](#attachment)                 |
| chuck_info          | 详见[Chunk模式](#chunk-mode)                 |
| authentication_data | 用于存放身份认证相关信息                             |

### 请求包元数据

请求包的元数据主要描述了需要调用的RPC方法信息，Protobuf如下

```
message RpcRequestMeta {
    required string service_name = 1; 
    required string method_name = 2;
    optional int64 log_id = 3; 
};
```

| 参数           | 说明                           |
| ------------ | ---------------------------- |
| service_name | 服务名，约束见上文                    |
| method_name  | 方法名，约束见上文                    |
| log_id       | 用于打印日志。可用于存放BFE_LOGID。该参数可选。 |

### 响应包元数据

响应包的元数据是对返回结果的描述。如果出现任何异常，错误也会放在元数据中。其Protobuf描述如下

```
message RpcResponseMeta { 
    optional int32 error_code = 1; 
    optional string error_text = 2; 
};
```

| 参数         | 说明                                   |
| ---------- | ------------------------------------ |
| error_code | 发生错误时的错误号，0表示正常，非0表示错误。具体含义由应用方自行定义。 |
| error_text | 错误的文本描述                              |

### 对元数据的扩展

某些实现需要在元数据中增加自己专有的字段。为了避免冲突，并保证不同实现之间相互调用的兼容性，所有实现都需要向接口规范委员会申请一个专用的序号用于存放自己的扩展字段。

以Hulu为例，被分配的序号为100。因此Hulu可以使用这样的proto定义：

```
message RpcMeta {
    optional RpcRequestMeta request = 1;
    optional RpcResponseMeta response = 2;
    optional int32 compress_type = 3;
    optional int64 correlation_id = 4;
    optional int32 attachment_size = 5;
    optional ChunkInfo chuck_info = 6;
    optional HuluMeta hulu_meta = 100;
};
message RpcRequestMeta {
    required string service_name = 1; 
    required string method_name = 2;
    optional int64 log_id = 3; 
    optional HuluRequestMeta hulu_request_meta = 100;
};
message RpcResponseMeta { 
    optional int32 error_code = 1; 
    optional string error_text = 2; 
    optional HuluResponseMeta hulu_response_meta = 100;
};
```

因为只是将100这个序号保留给Hulu使用，因此Hulu可以自由决定是否添加这些字段，以及使用什么样的名字。其余实现使用的proto中不存在这些定义，会直接作为Unknown字段忽略。

当前分配的序号如下

| 序号   | 实现   |
| ---- | ---- |
| 100  | Hulu |
| 101  | Sofa |

## 数据

自定义的Protobuf Message。用于存放参数或返回结果。

## Attachment

某些场景下需要通过RPC来传递二进制数据，例如文件上传下载，多媒体转码等等。将这些二进制数据打包在Protobuf内会增加不必要的内存拷贝。因此协议允许使用附件的方式直接传送二进制数据。

附件总是放在包体的最后，紧跟数据部分。消息包需要携带附件时，应将RpcMeta中的attachment_size设为附件的实际字节数。

## 压缩算法

可以使用指定的压缩算法来压缩消息包中的数据部分。

| 值    | 含义       |
| ---- | -------- |
| 0    | 不压缩      |
| 1    | 使用Snappy |
| 2    | 使用gzip   |

# HTTP接口

服务应以标准的HTTP协议对外发布接口。

数据交换格式默认应使用JSON。Content-Type使用application/json。有特殊需求的接口不受此限制。例如上传文件可以使用multipart/form-data；下载文件可以根据实际内容选用合适的Content-Type。

URL和JSON中的字符编码一律使用UTF-8。

建议使用RESTful形式的Web Service接口。由于RESTful并非一个严格的规范，本规范对此不做强制规定。

# ========= ./benchmark.md ========
NOTE: following tests were done in 2015, which may not reflect latest status of the package.

# 序言

在多核的前提下，性能和线程是紧密联系在一起的。线程间的跳转对高频IO操作的性能有决定性作用: 一次跳转意味着至少3-20微秒的延时，由于每个核心的L1 cache独立（我们的cpu L2 cache也是独立的），随之而来是大量的cache miss，一些变量的读取、写入延时会从纳秒级上升几百倍至微秒级: 等待cpu把对应的cacheline同步过来。有时这带来了一个出乎意料的结果，当每次的处理都很简短时，一个多线程程序未必比一个单线程程序更快。因为前者可能在每次付出了大的切换代价后只做了一点点“正事”，而后者在不停地做“正事”。不过单线程也是有代价的，它工作良好的前提是“正事”都很快，否则一旦某次变慢就使后续的所有“正事”都被延迟了。在一些处理时间普遍较短的程序中，使用（多个不相交的）单线程能最大程度地”做正事“，由于每个请求的处理时间确定，延时表现也很稳定，各种http server正是这样。但我们的检索服务要做的事情可就复杂多了，有大量的后端服务需要访问，广泛存在的长尾请求使每次处理的时间无法确定，排序策略也越来越复杂。如果还是使用（多个不相交的）单线程的话，一次难以预计的性能抖动，或是一个大请求可能导致后续一堆请求被延迟。

为了避免请求之间相互影响，请求级的线程跳转是brpc必须付出的代价，我们能做的是使[线程跳转最优化](io.md#the-full-picture)。不过，对服务的性能测试还不能很好地体现这点。测试中的处理往往极为简单，使得线程切换的影响空前巨大，通过控制多线程和单线程处理的比例，我们可以把一个测试服务的qps从100万到500万操纵自如（同机），这损伤了性能测试结果的可信度。要知道，真实的服务并不是在累加一个数字，或者echo一个字符串，一个qps几百万的echo程序没有指导意义。鉴于此，在发起性能测试一年后（15年底），在brpc又经历了1200多次改动后，我们需要review所有的测试，加强其中的线程因素，以获得对真实场景有明确意义的结果。具体来说: 

- 请求不应等长，要有长尾。这能考察RPC能否让请求并发，否则一个慢请求会影响大量后续请求。
- 要有多级server的场景。server内用client访问下游server，这能考察server和client的综合表现。
- 要有一个client访问多个server的场景。这能考察负载均衡是否足够并发，真实场景中很少一个client只访问一个server。

我们希望这套测试场景对其他服务的性能测试有借鉴意义。

# 测试目标

## UB

百度在08年开发的RPC框架，在百度产品线广泛使用，已被brpc代替。UB的每个请求独占一个连接(连接池)，在大规模服务中每台机器都需要保持大量的连接，限制了其使用场景，像百度的分布式系统没有用UB。UB只支持nshead+mcpack协议，也没怎么考虑扩展性，所以增加新协议和新功能往往要调整大段代码，在实践中大部分人“知难而退”了。UB缺乏调试和运维接口，服务的运行状态对用户基本是黑盒，只能靠低效地打日志来追踪问题，服务出现问题时常要拉上维护者一起排查，效率很低。UB有多个变种: 

* ubrpc: 百度在10年基于UB开发的RPC框架，用.idl文件(类似.proto)描述数据的schema，而不是手动打包。这个RPC有被使用，但不广泛。

- nova_pbrpc: 百度网盟团队在12年基于UB开发的RPC框架，用protobuf代替mcpack作为序列化方法，协议是nshead + user's protobuf。
- public_pbrpc: 百度在13年初基于UB开发的RPC框架，用protobuf代替mcpack作为序列化方法，但协议与nova_pbrpc不同，大致是nshead + meta protobuf。meta protobuf中有个string字段包含user's protobuf。由于用户数据要序列化两次，这个RPC的性能很差，没有被推广开来。

我们以在百度网盟团队广泛使用的nova_pbrpc为UB的代表。测试时其代码为r10500。早期的UB支持CPOOL和XPOOL，分别使用[select](http://linux.die.net/man/2/select)和[leader-follower模型](http://kircher-schwanninger.de/michael/publications/lf.pdf)，后来提供了EPOLL，使用[epoll](http://man7.org/linux/man-pages/man7/epoll.7.html)处理多路连接。鉴于产品线大都是用EPOLL模型，我们的UB配置也使用EPOLL。UB只支持[连接池](client.md#连接方式)，结果用“**ubrpc_mc**"指代（mc代表"multiple
connection"）。虽然这个名称不太准确（见上文对ubrpc的介绍），但在本文的语境下，请默认ubrpc = UB。

## hulu-pbrpc

百度在13年基于saber(kylin变种)和protobuf实现的RPC框架，hulu在多线程实现上有较多问题，已被brpc代替，测试时其代码为`pbrpc_2-0-15-27959_PD_BL`。hulu-pbrpc只支持单连接，结果用“**hulu-pbrpc**"指代。

## brpc

INF在2014年底开发至今的rpc产品，支持百度内所有协议（不限于protobuf），并第一次统一了百度主要分布式系统和业务线的RPC框架。测试时代码为r31906。brpc既支持单连接也支持连接池，前者的结果用"**baidu-rpc**"指代，后者用“**baidu-rpc_mc**"指代。

## sofa-pbrpc

百度大搜团队在13年基于boost::asio和protobuf实现的RPC框架，有多个版本，咨询相关同学后，确认ps/opensource下的和github上的较新，且会定期同步。故测试使用使用ps/opensource下的版本。测试时其代码为`sofa-pbrpc_1-0-2_BRANCH`。sofa-pbrpc只支持单连接，结果用“**sofa-pbrpc**”指代。

## apache thrift

thrift是由facebook最早在07年开发的序列化方法和rpc框架，包含独特的序列化格式和IDL，支持很多编程语言。开源后改名[apache thrift](https://thrift.apache.org/)，fb自己有一个[fbthrift分支](https://github.com/facebook/fbthrift)，我们使用的是apache thrift。测试时其代码为`thrift_0-9-1-400_PD_BL`。thrift的缺点是: 代码看似分层清晰，client和server选择很多，但没有一个足够通用，每个server实现都只能解决很小一块场景，每个client都线程不安全，实际使用很麻烦。由于thrift没有线程安全的client，所以每个线程中都得建立一个client，使用独立的连接。在测试中thrift其实是占了其他实现的便宜: 它的client不需要处理多线程问题。thrift的结果用"**thrift_mc**"指代。

## gRPC

由google开发的rpc框架，使用http/2和protobuf 3.0，测试时其代码为<https://github.com/grpc/grpc/tree/release-0_11>。gRPC并不是stubby，定位更像是为了推广http/2和protobuf 3.0，但鉴于很多人对它的表现很感兴趣，我们也（很麻烦地）把它加了进来。gRPC的结果用"**grpc**"指代。

# 测试方法

如序言中解释的那样，性能数字有巨大的调整空间。这里的关键在于，我们对RPC的底线要求是什么，脱离了这个底线，测试中的表现就严重偏离真实环境中的了。

这个底线我们认为是**RPC必须能处理长尾**。

在百度的环境中，这是句大白话，哪个产品线，哪个系统没有长尾呢？作为承载大部分服务的RPC框架自然得处理好长尾，减少长尾对正常请求的影响。但在实现层面，这个问题对设计的影响太大了。如果测试中没有长尾，那么RPC实现就可以假设每个请求都差不多快，这时候最优的方法是用多个线程独立地处理请求。由于没有上下文切换和cache一致性同步，程序的性能会显著高于多个线程协作时的表现。

比如简单的echo程序，处理一个请求只需要200-300纳秒，单个线程可以达到300-500万的吞吐。但如果多个线程协作，即使在极其流畅的系统中，也要付出3-5微秒的上下文切换代价和1微秒的cache同步代价，这还没有考虑多个线程间的其他互斥逻辑，一般来说单个线程的吞吐很难超过10万，即使24核全部用满，吞吐也只有240万，不及一个线程。这正是以http server为典型的服务选用[单线程模型](threading_overview.md#单线程reactor)的原因（多个线程独立运行eventloop）: 大部分http请求的处理时间是可预测的，对下游的访问也不会有任何阻塞代码。这个模型可以最大化cpu利用率，同时提供可接受的延时。

多线程付出这么大的代价是为了**隔离请求间的影响**。一个计算复杂或索性阻塞的过程不会影响到其他请求，1%的长尾最终只会影响到1%的性能。而多个独立的线程是保证不了这点的，一个请求进入了一个线程就等于“定了终生”，如果前面的请求慢了一下，那也只能跟着慢了。1%的长尾会影响远超1%的请求，最终表现不佳。换句话说，乍看上去多线程模型“慢”了，但在真实应用中反而会获得更好的综合性能。

延时能精确地体现出长尾的干扰作用，如果普通请求的延时没有被长尾请求干扰，就说明RPC成功地隔离了请求。而QPS无法体现这点，只要CPU都在忙，即使一个正常请求进入了挤满长尾的队列而被严重延迟，最终的QPS也变化不大。为了测量长尾的干扰作用，我们在涉及到延时的测试中都增加了1%的长尾请求。

# 开始测试

## 环境

性能测试使用的机器配置为: 

- 单机1: CPU开超线程24核，E5-2620 @ 2.00GHz；64GB内存；OS linux 2.6.32_1-15-0-0
- 多机1（15台+8台）: CPU均未开超线程12核，其中15台的CPU为E5-2420 @ 1.90GHz.，64GB内存，千兆网卡，无法开启多队列。其余8台为E5-2620 2.0GHz，千兆网卡，绑定多队列到前8个核。这些长期测试机器比较杂，跨了多个机房，测试中延时在1ms以上的就是这批机器。
- 多机2（30台）: CPU未开超线程12核，E5-2620 v3 @ 2.40GHz.；96GB内存；OS linux 2.6.32_1-17-0-0；万兆网卡，绑定多队列到前8个核。这是临时借用的新机器，配置非常好，都在广州机房，延时非常短，测试中延时在几百微秒的就是这批机器。

测试代码: <https://svn.baidu.com/com-test/trunk/public/rpc-perf/>

下面所有的曲线图是使用brpc开发的dashboard程序绘制的，去掉路径后可以看到和所有brpc
server一样的[内置服务](builtin_service.md)。

## 配置

如无特殊说明，所有测试中的配置只是数量差异（线程数，请求大小，client个数etc），而不是模型差异。我们确保用户看到的qps和延时是同一个场景的不同维度，而不是无法统一的两个场景。

所有RPC server都配置了24个工作线程，这些线程一般运行用户的处理逻辑。关于每种RPC的特殊说明: 

- UB: 配置了12个reactor线程，使用EPOOL模型。连接池限制数配置为线程个数（24）
- hulu-pbrpc: 额外配置了12个IO线程。这些线程会处理fd读取，请求解析等任务。hulu有个“共享队列“的配置项，默认不打开，作用是把fd静态散列到多个线程中，由于线程间不再争抢，hulu的qps会显著提高，但会明显地被长尾影响（原因见[测试方法](#测试方法)）。考虑到大部分使用者并不会去改配置，我们也选择不打开。
- thrift: 额外配置了12个IO线程。这些线程会处理fd读取，请求解析等任务。thrift的client不支持多线程，每个线程得使用独立的client，连接也都是分开的。
- sofa-pbrpc: 按照sofa同学的要求，把io_service_pool_size配置为24，work_thread_num配置为1。大概含义是使用独立的24组线程池，每组1个worker thread。和hulu不打开“共享队列”时类似，这个配置会显著提高sofa-pbrpc的QPS，但同时使它失去了处理长尾的能力。如果你在真实产品中使用，我们不建议这个配置。（而应该用io_service_pool_size=1, work_thread_num=24)
- brpc: 尽管brpc的client运行在bthread中时会获得10%~20%的QPS提升和更低的延时，但测试中的client都运行统一的pthread中。

所有的RPC client都以多个线程同步方式发送，这种方法最接近于真实系统中的情况，在考察QPS时也兼顾了延时因素。

一种流行的方案是client不停地往连接中写入数据看server表现，这个方法的弊端在于: server一下子能读出大量请求，不同RPC的比拼变成了“for循环执行用户代码”的比拼，而不是分发请求的效率。在真实系统中server很少能同时读到超过4个请求。这个方法也完全放弃了延时，client其实是让server陷入了雪崩时才会进入的状态，所有请求都因大量排队而超时了。

## 同机单client→单server在不同请求下的QPS（越高越好）

本测试运行在[单机1](#环境)上。图中的数值均为用户数据的字节数，实际的请求尺寸还要包括协议头，一般会增加40字节左右。

（X轴是用户数据的字节数，Y轴是对应的QPS）

![img](../images/qps_vs_reqsize.png)

以_mc结尾的曲线代表client和server保持多个连接（线程数个），在本测试中会有更好的表现。

**分析**

 * brpc: 当请求包小于16KB时，单连接下的吞吐超过了多连接的ubrpc_mc和thrift_mc，随着请求包变大，内核对单个连接的写入速度成为瓶颈。而多连接下的brpc则达到了测试中最高的2.3GB/s。注意: 虽然使用连接池的brpc在发送大包时吞吐更高，但也会耗费更多的CPU（UB和thrift也是这样）。下图中的单连接brpc已经可以提供800多兆的吞吐，足以打满万兆网卡，而使用的CPU可能只有多链接下的1/2(写出过程是[wait-free的](io.md#发消息))，真实系统中请优先使用单链接。
* thrift: 初期明显低于brpc，随着包变大超过了单连接的brpc。
* UB:和thrift类似的曲线，但平均要低4-5万QPS，在32K包时超过了单连接的brpc。整个过程中QPS几乎没变过。
* gRPC: 初期几乎与UB平行，但低1万左右，超过8K开始下降。
* hulu-pbrpc和sofa-pbrpc: 512字节前高于UB和gRPC，但之后就急转直下，相继垫底。这个趋势是写不够并发的迹象。

## 同机单client→单server在不同线程数下的QPS（越高越好）

本测试运行在[单机1](#环境)上。

（X轴是线程数，Y轴是对应的QPS）

![img](../images/qps_vs_threadnum.png)

**分析**

brpc: 随着发送线程增加，QPS在快速增加，有很好的多线程扩展性。

UB和thrift: 8个线程下高于brpc，但超过8个线程后被brpc迅速超过，thrift继续“平移”，UB出现了明显下降。

gRPC，hulu-pbrpc，sofa-pbrpc: 几乎重合，256个线程时相比1个线程时只有1倍的提升，多线程扩展性不佳。

## 同机单client→单server在固定QPS下的延时[CDF](vars.md#统计和查看分位值)（越左越好，越直越好）
本测试运行在[单机1](#环境)上。考虑到不同RPC的处理能力，我们选择了一个较低、在不少系统中会达到的的QPS: 1万。

本测试中有1%的长尾请求耗时5毫秒，长尾请求的延时不计入结果，因为我们考察的是普通请求是否被及时处理了。

（X轴是延时（微秒），Y轴是小于X轴延时的请求比例）

![img](../images/latency_cdf.png)

**分析**
- brpc: 平均延时短，几乎没有被长尾影响。
- UB和thrift: 平均延时比brpc高1毫秒，受长尾影响不大。
- hulu-pbrpc: 走向和UB和thrift类似，但平均延时进一步增加了1毫秒。
- gRPC : 初期不错，到长尾区域后表现糟糕，直接有一部分请求超时了。（反复测试都是这样，像是有bug）
- sofa-pbrpc: 30%的普通请求（上图未显示）被长尾严重干扰。

## 跨机多client→单server的QPS（越高越好）

本测试运行在[多机1](#环境)上。

（X轴是client数，Y轴是对应的QPS）

![img](../images/qps_vs_multi_client.png)

**分析**
* brpc: 随着cilent增加，server的QPS在快速增加，有不错的client扩展性。
* sofa-pbrpc: 随着client增加，server的QPS也在快速增加，但幅度不如brpc，client扩展性也不错。从16个client到32个client时的提升较小。
* hulu-pbrpc: 随着client增加，server的QPS在增加，但幅度进一步小于sofa-pbrpc。
* UB: 增加client几乎不能增加server的QPS。
* thrift: 平均QPS低于UB，增加client几乎不能增加server的QPS。
* gRPC: 垫底、增加client几乎不能增加server的QPS。

## 跨机多client→单server在固定QPS下的延时[CDF](vars.md#统计和查看分位值)（越左越好，越直越好）

本测试运行在[多机1](#环境)上。负载均衡算法为round-robin或RPC默认提供的。由于有32个client且一些RPC的单client能力不佳，我们为每个client仅设定了2500QPS，这是一个真实业务系统能达到的数字。

本测试中有1%的长尾请求耗时15毫秒，长尾请求的延时不计入结果，因为我们考察的是普通请求是否被及时处理了。

（X轴是延时（微秒），Y轴是小于X轴延时的请求比例）

![img](../images/multi_client_latency_cdf.png)

**分析**
- brpc: 平均延时短，几乎没有被长尾影响。
- UB和thrift: 平均延时短，受长尾影响小，平均延时高于brpc
- sofa-pbrpc: 14%的普通请求被长尾严重干扰。
- hulu-pbrpc: 15%的普通请求被长尾严重干扰。
- gRPC: 已经完全失控，非常糟糕。

## 跨机多client→多server在固定QPS下的延时[CDF](vars.md#统计和查看分位值)（越左越好，越直越好）

本测试运行在[多机2](#环境)上。20台每台运行4个client，多线程同步访问10台server。负载均衡算法为round-robin或RPC默认提供的。由于gRPC访问多server较麻烦且有很大概率仍表现不佳，这个测试不包含gRPC。

本测试中有1%的长尾请求耗时10毫秒，长尾请求的延时不计入结果，因为我们考察的是普通请求是否被及时处理了。

（X轴是延时（微秒），Y轴是小于X轴延时的请求比例）

![img](../images/multi_server_latency_cdf.png)

**分析**
- brpc和UB: 平均延时短，几乎没有被长尾影响。
- thrift: 平均延时显著高于brpc和UB。
- sofa-pbrpc: 2.5%的普通请求被长尾严重干扰。
- hulu-pbrpc: 22%的普通请求被长尾严重干扰。

## 跨机多client→多server→多server在固定QPS下的延时[CDF](vars.md#统计和查看分位值)（越左越好，越直越好）

本测试运行在[多机2](#环境)上。14台每台运行4个client，多线程同步访问8台server，这些server还会同步访问另外8台server。负载均衡算法为round-robin或RPC默认提供的。由于gRPC访问多server较麻烦且有很大概率仍表现不佳，这个测试不包含gRPC。

本测试中有1%的长尾请求耗时10毫秒，长尾请求的延时不计入结果，因为我们考察的是普通请求是否被及时处理了。

（X轴是延时（微秒），Y轴是小于X轴延时的请求比例）

![img](../images/twolevel_server_latency_cdf.png)

**分析**
- brpc: 平均延时短，几乎没有被长尾影响。
- UB: 平均延时短，长尾区域略差于brpc。
- thrift: 平均延时显著高于brpc和UB。
- sofa-pbrpc: 17%的普通请求被长尾严重干扰，其中2%的请求延时极长。
- hulu-pbrpc: 基本消失在视野中，已无法正常工作。

# 结论

brpc: 在吞吐，平均延时，长尾处理上都表现优秀。

UB: 平均延时和长尾处理的表现都不错，吞吐的扩展性较差，提高线程数和client数几乎不能提升吞吐。

thrift: 单机的平均延时和吞吐尚可，多机的平均延时明显高于brpc和UB。吞吐的扩展性较差，提高线程数和client数几乎不能提升吞吐。

sofa-pbrpc: 处理小包的吞吐尚可，大包的吞吐显著低于其他RPC，延时受长尾影响很大。

hulu-pbrpc: 单机表现和sofa-pbrpc类似，但多机的延时表现极差。

gRPC: 几乎在所有参与的测试中垫底，可能它的定位是给google cloud platform的用户提供一个多语言，对网络友好的实现，性能还不是要务。



# ========= ./benchmark_http.md ========
可代替[ab](https://httpd.apache.org/docs/2.2/programs/ab.html)测试http server极限性能。ab功能较多但年代久远，有时本身可能会成为瓶颈。benchmark_http基本上就是一个brpc http client，性能很高，功能较少，一般压测够用了。

使用方法：

首先你得[下载和编译](getting_started.md)了brpc源码，然后去example/http_c++目录编译，成功后应该能看到benchmark_http。


# ========= ./bthread.md ========
[bthread](https://github.com/brpc/brpc/tree/master/src/bthread)是brpc使用的M:N线程库，目的是在提高程序的并发度的同时，降低编码难度，并在核数日益增多的CPU上提供更好的scalability和cache locality。”M:N“是指M个bthread会映射至N个pthread，一般M远大于N。由于linux当下的pthread实现([NPTL](http://en.wikipedia.org/wiki/Native_POSIX_Thread_Library))是1:1的，M个bthread也相当于映射至N个[LWP](http://en.wikipedia.org/wiki/Light-weight_process)。bthread的前身是Distributed Process(DP)中的fiber，一个N:1的合作式线程库，等价于event-loop库，但写的是同步代码。

# Goals

- 用户可以延续同步的编程模式，能在数百纳秒内建立bthread，可以用多种原语同步。
- bthread所有接口可在pthread中被调用并有合理的行为，使用bthread的代码可以在pthread中正常执行。
- 能充分利用多核。
- better cache locality, supporting NUMA is a plus.

# NonGoals

- 提供pthread的兼容接口，只需链接即可使用。**拒绝理由**: bthread没有优先级，不适用于所有的场景，链接的方式容易使用户在不知情的情况下误用bthread，造成bug。
- 覆盖各类可能阻塞的glibc函数和系统调用，让原本阻塞系统线程的函数改为阻塞bthread。**拒绝理由**: 
  - bthread阻塞可能切换系统线程，依赖系统TLS的函数的行为未定义。
  - 和阻塞pthread的函数混用时可能死锁。
  - 这类hook函数本身的效率一般更差，因为往往还需要额外的系统调用，如epoll。但这类覆盖对N:1合作式线程库(fiber)有一定意义：虽然函数本身慢了，但若不覆盖会更慢（系统线程阻塞会导致所有fiber阻塞）。
- 修改内核让pthread支持同核快速切换。**拒绝理由**: 拥有大量pthread后，每个线程对资源的需求被稀释了，基于thread-local cache的代码效果会很差，比如tcmalloc。而独立的bthread不会有这个问题，因为它最终还是被映射到了少量的pthread。bthread相比pthread的性能提升很大一部分来自更集中的线程资源。另一个考量是可移植性，bthread更倾向于纯用户态代码。

# FAQ

##### Q：bthread是协程(coroutine)吗？

不是。我们常说的协程特指N:1线程库，即所有的协程运行于一个系统线程中，计算能力和各类eventloop库等价。由于不跨线程，协程之间的切换不需要系统调用，可以非常快(100ns-200ns)，受cache一致性的影响也小。但代价是协程无法高效地利用多核，代码必须非阻塞，否则所有的协程都被卡住，对开发者要求苛刻。协程的这个特点使其适合写运行时间确定的IO服务器，典型如http server，在一些精心调试的场景中，可以达到非常高的吞吐。但百度内大部分在线服务的运行时间并不确定，且很多检索由几十人合作完成，一个缓慢的函数会卡住所有的协程。在这点上eventloop是类似的，一个回调卡住整个loop就卡住了，比如ub**a**server（注意那个a，不是ubserver）是百度对异步框架的尝试，由多个并行的eventloop组成，真实表现糟糕：回调里打日志慢一些，访问redis卡顿，计算重一点，等待中的其他请求就会大量超时。所以这个框架从未流行起来。

bthread是一个M:N线程库，一个bthread被卡住不会影响其他bthread。关键技术两点：work stealing调度和butex，前者让bthread更快地被调度到更多的核心上，后者让bthread和pthread可以相互等待和唤醒。这两点协程都不需要。更多线程的知识查看[这里](threading_overview.md)。

##### Q: 我应该在程序中多使用bthread吗？

不应该。除非你需要在一次RPC过程中[让一些代码并发运行](bthread_or_not.md)，你不应该直接调用bthread函数，把这些留给brpc做更好。

##### Q：bthread和pthread worker如何对应？

pthread worker在任何时间只会运行一个bthread，当前bthread挂起时，pthread worker先尝试从本地runqueue弹出一个待运行的bthread，若没有，则随机偷另一个worker的待运行bthread，仍然没有才睡眠并会在有新的待运行bthread时被唤醒。

##### Q：bthread中能调用阻塞的pthread或系统函数吗？

可以，只阻塞当前pthread worker。其他pthread worker不受影响。

##### Q：一个bthread阻塞会影响其他bthread吗？

不影响。若bthread因bthread API而阻塞，它会把当前pthread worker让给其他bthread。若bthread因pthread API或系统函数而阻塞，当前pthread worker上待运行的bthread会被其他空闲的pthread worker偷过去运行。

##### Q：pthread中可以调用bthread API吗？

可以。bthread API在bthread中被调用时影响的是当前bthread，在pthread中被调用时影响的是当前pthread。使用bthread API的代码可以直接运行在pthread中。

##### Q：若有大量的bthread调用了阻塞的pthread或系统函数，会影响RPC运行么？

会。比如有8个pthread worker，当有8个bthread都调用了系统usleep()后，处理网络收发的RPC代码就暂时无法运行了。只要阻塞时间不太长, 这一般**没什么影响**, 毕竟worker都用完了, 除了排队也没有什么好方法.
在brpc中用户可以选择调大worker数来缓解问题, 在server端可设置[ServerOptions.num_threads](server.md#worker线程数)或[-bthread_concurrency](http://brpc.baidu.com:8765/flags/bthread_concurrency), 在client端可设置[-bthread_concurrency](http://brpc.baidu.com:8765/flags/bthread_concurrency).

那有没有完全规避的方法呢?

- 一个容易想到的方法是动态增加worker数. 但实际未必如意, 当大量的worker同时被阻塞时,
  它们很可能在等待同一个资源(比如同一把锁), 增加worker可能只是增加了更多的等待者. 
- 那区分io线程和worker线程? io线程专门处理收发, worker线程调用用户逻辑, 即使worker线程全部阻塞也不会影响io线程. 但增加一层处理环节(io线程)并不能缓解拥塞, 如果worker线程全部卡住, 程序仍然会卡住,
  只是卡的地方从socket缓冲转移到了io线程和worker线程之间的消息队列. 换句话说, 在worker卡住时,
  还在运行的io线程做的可能是无用功. 事实上, 这正是上面提到的**没什么影响**真正的含义. 另一个问题是每个请求都要从io线程跳转至worker线程, 增加了一次上下文切换, 在机器繁忙时, 切换都有一定概率无法被及时调度, 会导致更多的延时长尾.
- 一个实际的解决方法是[限制最大并发](server.md#限制最大并发), 只要同时被处理的请求数低于worker数, 自然可以规避掉"所有worker被阻塞"的情况.
- 另一个解决方法当被阻塞的worker超过阈值时(比如8个中的6个), 就不在原地调用用户代码了, 而是扔到一个独立的线程池中运行. 这样即使用户代码全部阻塞, 也总能保留几个worker处理rpc的收发. 不过目前bthread模式并没有这个机制, 但类似的机制在[打开pthread模式](server.md#pthread模式)时已经被实现了. 那像上面提到的, 这个机制是不是在用户代码都阻塞时也在做"无用功"呢? 可能是的. 但这个机制更多是为了规避在一些极端情况下的死锁, 比如所有的用户代码都lock在一个pthread mutex上, 并且这个mutex需要在某个RPC回调中unlock, 如果所有的worker都被阻塞, 那么就没有线程来处理RPC回调了, 整个程序就死锁了. 虽然绝大部分的RPC实现都有这个潜在问题, 但实际出现频率似乎很低, 只要养成不在锁内做RPC的好习惯, 这是完全可以规避的. 

##### Q：bthread会有[Channel](https://gobyexample.com/channels)吗？

不会。channel代表的是两点间的关系，而很多现实问题是多点的，这个时候使用channel最自然的解决方案就是：有一个角色负责操作某件事情或某个资源，其他线程都通过channel向这个角色发号施令。如果我们在程序中设置N个角色，让它们各司其职，那么程序就能分类有序地运转下去。所以使用channel的潜台词就是把程序划分为不同的角色。channel固然直观，但是有代价：额外的上下文切换。做成任何事情都得等到被调用处被调度，处理，回复，调用处才能继续。这个再怎么优化，再怎么尊重cache locality，也是有明显开销的。另外一个现实是：用channel的代码也不好写。由于业务一致性的限制，一些资源往往被绑定在一起，所以一个角色很可能身兼数职，但它做一件事情时便无法做另一件事情，而事情又有优先级。各种打断、跳出、继续形成的最终代码异常复杂。

我们需要的往往是buffered channel，扮演的是队列和有序执行的作用，bthread提供了[ExecutionQueue](execution_queue.md)，可以完成这个目的。


# ========= ./bthread_id.md ========
bthread_id是一个特殊的同步结构，它可以互斥RPC过程中的不同环节，也可以O(1)时间内找到RPC上下文(即Controller)。注意，这里我们谈论的是bthread_id_t，不是bthread_t（bthread的tid），这个名字起的确实不太好，容易混淆。

具体来说，bthread_id解决的问题有：

- 在发送RPC过程中response回来了，处理response的代码和发送代码产生竞争。
- 设置timer后很快触发了，超时处理代码和发送代码产生竞争。
- 重试产生的多个response同时回来产生的竞争。
- 通过correlation_id在O(1)时间内找到对应的RPC上下文，而无需建立从correlation_id到RPC上下文的全局哈希表。
- 取消RPC。

上文提到的bug在其他rpc框架中广泛存在，下面我们来看下brpc是如何通过bthread_id解决这些问题的。

bthread_id包括两部分，一个是用户可见的64位id，另一个是对应的不可见的bthread::Id结构体。用户接口都是操作id的。从id映射到结构体的方式和brpc中的[其他结构](memory_management.md)类似：32位是内存池的位移，32位是version。前者O(1)时间定位，后者防止ABA问题。

bthread_id的接口不太简洁，有不少API：

- create
- lock
- unlock
- unlock_and_destroy
- join
- error

这么多接口是为了满足不同的使用流程。

- 发送request的流程：create -> lock -> ... register timer and send RPC ... -> unlock
- 接收response的流程：lock -> ..process response -> call done


# ========= ./bthread_or_not.md ========
brpc提供了[异步接口](client.md#异步访问)，所以一个常见的问题是：我应该用异步接口还是bthread？

短回答：延时不高时你应该先用简单易懂的同步接口，不行的话用异步接口，只有在需要多核并行计算时才用bthread。

# 同步或异步

异步即用回调代替阻塞，有阻塞的地方就有回调。虽然在javascript这种语言中回调工作的很好，接受度也非常高，但只要用过，就会发现这和我们需要的回调是两码事，这个区别不是[lambda](https://en.wikipedia.org/wiki/Anonymous_function)，也不是[future](https://en.wikipedia.org/wiki/Futures_and_promises)，而是javascript是单线程的。javascript的回调放到多线程下可能没有一个能跑过，竞争太多，单线程的同步方法和多线程的同步方法是完全不同的。那是不是服务能搞成类似的形式呢？多个线程，每个都是独立的eventloop。可以，ub**a**server就是（注意带a)，但实际效果糟糕，因为阻塞改回调可不简单，当阻塞发生在循环，条件分支，深层子函数中时，改造特别困难，况且很多老代码、第三方代码根本不可能去改造。结果是代码中会出现不可避免的阻塞，导致那个线程中其他回调都被延迟，流量超时，server性能不符合预期。如果你说，”我想把现在的同步代码改造为大量的回调，除了我其他人都看不太懂，并且性能可能更差了”，我猜大部分人不会同意。别被那些鼓吹异步的人迷惑了，他们写的是从头到尾从下到上全异步且不考虑多线程的代码，和你要写的完全是两码事。

brpc中的异步和单线程的异步是完全不同的，异步回调会运行在与调用处不同的线程中，你会获得多核扩展性，但代价是你得意识到多线程问题。你可以在回调中阻塞，只要线程够用，对server整体的性能并不会有什么影响。不过异步代码还是很难写的，所以我们提供了[组合访问](combo_channel.md)来简化问题，通过组合不同的channel，你可以声明式地执行复杂的访问，而不用太关心其中的细节。

当然，延时不长，qps不高时，我们更建议使用同步接口，这也是创建bthread的动机：维持同步代码也能提升交互性能。

**判断使用同步或异步**：计算qps * latency(in seconds)，如果和cpu核数是同一数量级，就用同步，否则用异步。

比如：

- qps = 2000，latency = 10ms，计算结果 = 2000 * 0.01s = 20。和常见的32核在同一个数量级，用同步。
- qps = 100, latency = 5s, 计算结果 = 100 * 5s = 500。和核数不在同一个数量级，用异步。
- qps = 500, latency = 100ms，计算结果 = 500 * 0.1s = 50。基本在同一个数量级，可用同步。如果未来延时继续增长，考虑异步。

这个公式计算的是同时进行的平均请求数（你可以尝试证明一下），和线程数，cpu核数是可比的。当这个值远大于cpu核数时，说明大部分操作并不耗费cpu，而是让大量线程阻塞着，使用异步可以明显节省线程资源（栈占用的内存）。当这个值小于或和cpu核数差不多时，异步能节省的线程资源就很有限了，这时候简单易懂的同步代码更重要。

# 异步或bthread

有了bthread这个工具，用户甚至可以自己实现异步。以“半同步”为例，在brpc中用户有多种选择：

- 发起多个异步RPC后挨个Join，这个函数会阻塞直到RPC结束。（这儿是为了和bthread对比，实现中我们建议你使用[ParallelChannel](combo_channel.md#parallelchannel)，而不是自己Join）
- 启动多个bthread各自执行同步RPC后挨个join bthreads。

哪种效率更高呢？显然是前者。后者不仅要付出创建bthread的代价，在RPC过程中bthread还被阻塞着，不能用于其他用途。

**如果仅仅是为了并发RPC，别用bthread。**

不过当你需要并行计算时，问题就不同了。使用bthread可以简单地构建树形的并行计算，充分利用多核资源。比如检索过程中有三个环节可以并行处理，你可以建立两个bthread运行两个环节，在原地运行剩下的环节，最后join那两个bthread。过程大致如下：
```c++
bool search() {
  ...
  bthread th1, th2;
  if (bthread_start_background(&th1, NULL, part1, part1_args) != 0) {
    LOG(ERROR) << "Fail to create bthread for part1";
    return false;
  }
  if (bthread_start_background(&th2, NULL, part2, part2_args) != 0) {
    LOG(ERROR) << "Fail to create bthread for part2";
    return false;
  }
  part3(part3_args);
  bthread_join(th1);
  bthread_join(th2);
  return true;
}
```
这么实现的point：
- 你当然可以建立三个bthread分别执行三个部分，最后join它们，但相比这个方法要多耗费一个线程资源。
- bthread从建立到执行是有延时的（调度延时），在不是很忙的机器上，这个延时的中位数在3微秒左右，90%在10微秒内，99.99%在30微秒内。这说明两点：
  - 计算时间超过1ms时收益比较明显。如果计算非常简单，几微秒就结束了，用bthread是没有意义的。
  - 尽量让原地运行的部分最慢，那样bthread中的部分即使被延迟了几微秒，最后可能还是会先结束，而消除掉延迟的影响。并且join一个已结束的bthread时会立刻返回，不会有上下文切换开销。

另外当你有类似线程池的需求时，像执行一类job的线程池时，也可以用bthread代替。如果对job的执行顺序有要求，你可以使用基于bthread的[ExecutionQueue](execution_queue.md)。


# ========= ./builtin_service.md ========
[English version](../en/builtin_service.md)

# 什么是内置服务？

内置服务以多种形式展现服务器内部状态，提高你开发和调试服务的效率。brpc通过HTTP协议提供内置服务，可通过浏览器或curl访问，服务器会根据User-Agent返回纯文本或html，你也可以添加?console=1要求返回纯文本。我们在自己的开发机上启动了[一个长期运行的例子](http://brpc.baidu.com:8765/)(只能百度内访问)，你可以点击后随便看看。如果服务端口被限（比如百度内不是所有的端口都能被笔记本访问到），可以使用[rpc_view](rpc_view.md)转发。

下面是分别从浏览器和终端访问的截图，注意其中的logo是百度内部的名称，在开源版本中是brpc。

**从浏览器访问**

![img](../images/builtin_service_more.png)

**从命令行访问** ![img](../images/builtin_service_from_console.png)

# 安全模式

出于安全考虑，直接对外服务需要隐藏内置服务（包括经过nginx或其他http server转发流量的），具体方法请阅读[这里](server.md#安全模式)。

# 主要服务

[/status](status.md): 显示所有服务的主要状态。

[/vars](vars.md): 用户可定制的，描绘各种指标的计数器。

[/connections](connections.md): 所有连接的统计信息。

[/flags](flags.md): 所有gflags的状态，可动态修改。

[/rpcz](rpcz.md): 查看所有的RPC的细节。

[cpu profiler](cpu_profiler.md): 分析cpu热点。

[heap profiler](heap_profiler.md): 分析内存占用。

[contention profiler](contention_profiler.md): 分析锁竞争。

# 其他服务

[/version](http://brpc.baidu.com:8765/version): 查看服务器的版本。用户可通过Server::set_version()设置Server的版本，如果用户没有设置，框架会自动为用户生成，规则：`brpc_server_<service-name1>_<service-name2> ...`

![img](../images/version_service.png)

[/health](http://brpc.baidu.com:8765/health): 探测服务的存活情况。

![img](../images/health_service.png)

[/protobufs](http://brpc.baidu.com:8765/protobufs): 查看程序中所有的protobuf结构体。

![img](../images/protobufs_service.png)

[/vlog](http://brpc.baidu.com:8765/vlog): 查看程序中当前可开启的[VLOG](streaming_log.md#VLOG) (对glog无效)。

![img](../images/vlog_service.png)

/dir: 浏览服务器上的所有文件，方便但非常危险，默认关闭。

/threads: 查看进程内所有线程的运行状况，调用时对程序性能影响较大，默认关闭。

# ========= ./bvar.md ========
[English version](../en/bvar.md)

# 什么是bvar？

[bvar](https://github.com/brpc/brpc/tree/master/src/bvar/)是多线程环境下的计数器类库，方便记录和查看用户程序中的各类数值，它利用了thread local存储减少了cache bouncing，相比UbMonitor(百度内的老计数器库)几乎不会给程序增加性能开销，也快于竞争频繁的原子操作。brpc集成了bvar，[/vars](http://brpc.baidu.com:8765/vars)可查看所有曝光的bvar，[/vars/VARNAME](http://brpc.baidu.com:8765/vars/rpc_socket_count)可查阅某个bvar，在brpc中的使用方法请查看[vars](vars.md)。brpc大量使用了bvar提供统计数值，当你需要在多线程环境中计数并展现时，应该第一时间想到bvar。但bvar不能代替所有的计数器，它的本质是把写时的竞争转移到了读：读得合并所有写过的线程中的数据，而不可避免地变慢了。当你读写都很频繁或得基于最新值做一些逻辑判断时，你不应该用bvar。

为了理解bvar的原理，你得先阅读[Cacheline这节](atomic_instructions.md#cacheline)，其中提到的计数器例子便是bvar。当很多线程都在累加一个计数器时，每个线程只累加私有的变量而不参与全局竞争，在读取时累加所有线程的私有变量。虽然读比之前慢多了，但由于这类计数器的读多为低频的记录和展现，慢点无所谓。而写就快多了，极小的开销使得用户可以无顾虑地使用bvar监控系统，这便是我们设计bvar的目的。

下图是bvar和原子变量，静态UbMonitor，动态UbMonitor在被多个线程同时使用时的开销。可以看到bvar的耗时基本和线程数无关，一直保持在极低的水平（~20纳秒）。而动态UbMonitor在24核时每次累加的耗时达7微秒，这意味着使用300次bvar的开销才抵得上使用一次动态UbMonitor变量。

![img](../images/bvar_perf.png)

# 监控bvar

下图是监控bvar的示意图：

![img](../images/bvar_flow.png)

其中：

- APP代表用户服务，使用bvar API定义监控各类指标。
- bvar定期把被监控的项目打入$PWD/monitor/目录下的文件（用log指代）。此处的log和普通的log的不同点在于是bvar导出是覆盖式的，而不是添加式的。
- 监控系统（用noah指代）收集导出的文件，汇总至全局并生成曲线。

建议APP做到如下监控要求：

- **Error**: 系统中可能出现的error个数 
- **Latency**: 系统对外的每个RPC接口的latency(平均和分位值)，系统依赖的每个后台的每个RPC接口的latency
- **QPS**: 系统对外的每个RPC接口的QPS信息，系统依赖的每个后台的每个RPC接口的QPS信息

增加C++ bvar的方法请看[快速介绍](bvar_c++.md#quick-introduction). bvar默认统计了进程、系统的一些变量，以process\_, system\_等开头，比如：

```
process_context_switches_involuntary_second : 14
process_context_switches_voluntary_second : 15760
process_cpu_usage : 0.428
process_cpu_usage_system : 0.142
process_cpu_usage_user : 0.286
process_disk_read_bytes_second : 0
process_disk_write_bytes_second : 260902
process_faults_major : 256
process_faults_minor_second : 14
process_memory_resident : 392744960
system_core_count : 12
system_loadavg_15m : 0.040
system_loadavg_1m : 0.000
system_loadavg_5m : 0.020
```

还有像brpc内部的各类计数器：

```
bthread_switch_second : 20422
bthread_timer_scheduled_second : 4
bthread_timer_triggered_second : 4
bthread_timer_usage : 2.64987e-05
bthread_worker_count : 13
bthread_worker_usage : 1.33733
bvar_collector_dump_second : 0
bvar_collector_dump_thread_usage : 0.000307385
bvar_collector_grab_second : 0
bvar_collector_grab_thread_usage : 1.9699e-05
bvar_collector_pending_samples : 0
bvar_dump_interval : 10
bvar_revision : "34975"
bvar_sampler_collector_usage : 0.00106495
iobuf_block_count : 89
iobuf_block_count_hit_tls_threshold : 0
iobuf_block_memory : 729088
iobuf_newbigview_second : 10
```

打开bvar的[dump功能](bvar_c++.md#export-all-variables)以导出所有的bvar到文件，格式就入上文一样，每行是一对"名字:值"。打开dump功能后应检查monitor/下是否有数据，比如：

```
$ ls monitor/
bvar.echo_client.data  bvar.echo_server.data
 
$ tail -5 monitor/bvar.echo_client.data
process_swaps : 0
process_time_real : 2580.157680
process_time_system : 0.380942
process_time_user : 0.741887
process_username : "gejun"
```

监控系统会把定期把单机导出数据汇总到一起，并按需查询。这里以百度内的noah为例，bvar定义的变量会出现在noah的指标项中，用户可勾选并查看历史曲线。

![img](../images/bvar_noah2.png)

![img](../images/bvar_noah3.png)


# ========= ./bvar_c++.md ========
# Quick introduction

```c++
#include <bvar/bvar.h>

namespace foo {
namespace bar {

// bvar::Adder<T>用于累加，下面定义了一个统计read error总数的Adder。
bvar::Adder<int> g_read_error;
// 把bvar::Window套在其他bvar上就可以获得时间窗口内的值。
bvar::Window<bvar::Adder<int> > g_read_error_minute("foo_bar", "read_error", &g_read_error, 60);
//                                                     ^          ^                         ^
//                                                    前缀       监控项名称                  60秒,
// 忽略则为10秒

// bvar::LatencyRecorder是一个复合变量，可以统计：总量、qps、平均延时，延时分位值，最大延时。
bvar::LatencyRecorder g_write_latency(“foo_bar", "write”);
//                                      ^          ^
//                                     前缀       监控项，别加latency！LatencyRecorder包含多个bvar，
//它们会加上各自的后缀，比如write_qps, write_latency等等。

// 定义一个统计“已推入task”个数的变量。
bvar::Adder<int> g_task_pushed("foo_bar", "task_pushed");
// 把bvar::PerSecond套在其他bvar上可以获得时间窗口内*平均每秒*的值，这里是每秒内推入task的个数。
bvar::PerSecond<bvar::Adder<int> > g_task_pushed_second("foo_bar", "task_pushed_second",
 &g_task_pushed);
//       ^                                                                                             ^
//    和Window不同，PerSecond会除以时间窗口的大小.                                   
//时间窗口是最后一个参数，这里没填，就是默认10秒。

}  // bar
}  // foo
```

在应用的地方：

```c++
// 碰到read error
foo::bar::g_read_error << 1;

// write_latency是23ms
foo::bar::g_write_latency << 23;

// 推入了1个task
foo::bar::g_task_pushed << 1;
```

注意Window<>和PerSecond<>都是衍生变量，会自动更新，你不用给它们推值。你当然也可以把bvar作为成员变量或局部变量。

常用的bvar有：

- `bvar::Adder<T>` : 计数器，默认0，varname << N相当于varname += N。
- `bvar::Maxer<T>` : 求最大值，默认std::numeric_limits<T>::min()，varname << N相当于varname = max(varname, N)。
- `bvar::Miner<T>` : 求最小值，默认std::numeric_limits<T>::max()，varname << N相当于varname = min(varname, N)。
- `bvar::IntRecorder` : 求自使用以来的平均值。注意这里的定语不是“一段时间内”。一般要通过Window衍生出时间窗口内的平均值。
- `bvar::Window<VAR>` : 获得某个bvar在一段时间内的累加值。Window衍生于已存在的bvar，会自动更新。
- `bvar::PerSecond<VAR>` : 获得某个bvar在一段时间内平均每秒的累加值。PerSecond也是会自动更新的衍生变量。
- `bvar::LatencyRecorder` : 专用于记录延时和qps的变量。输入延时，平均延时/最大延时/qps/总次数 都有了。

**确认变量名是全局唯一的！**否则会曝光失败，如果-bvar_abort_on_same_name为true，程序会直接abort。

程序中有来自各种模块不同的bvar，为避免重名，建议如此命名：**模块_类名_指标**

- **模块**一般是程序名，可以加上产品线的缩写，比如inf_ds，ecom_retrbs等等。
- **类名**一般是类名或函数名，比如storage_manager, file_transfer, rank_stage1等等。
- **指标**一般是count，qps，latency这类。

一些正确的命名如下：

```
iobuf_block_count : 29                          # 模块=iobuf   类名=block  指标=count
iobuf_block_memory : 237568                     # 模块=iobuf   类名=block  指标=memory
process_memory_resident : 34709504              # 模块=process 类名=memory 指标=resident
process_memory_shared : 6844416                 # 模块=process 类名=memory 指标=shared
rpc_channel_connection_count : 0                # 模块=rpc     类名=channel_connection  指标=count
rpc_controller_count : 1                        # 模块=rpc     类名=controller 指标=count
rpc_socket_count : 6                            # 模块=rpc     类名=socket     指标=count
```

目前bvar会做名字归一化，不管你打入的是foo::BarNum, foo.bar.num, foo bar num , foo-bar-num，最后都是foo_bar_num。

关于指标：

- 个数以_count为后缀，比如request_count, error_count。
- 每秒的个数以_second为后缀，比如request_second, process_inblocks_second，已经足够明确，不用写成_count_second或_per_second。
- 每分钟的个数以_minute为后缀，比如request_minute, process_inblocks_minute

如果需要使用定义在另一个文件中的计数器，需要在头文件中声明对应的变量。

```c++
namespace foo {
namespace bar {
// 注意g_read_error_minute和g_task_pushed_per_second都是衍生的bvar，会自动更新，不要声明。
extern bvar::Adder<int> g_read_error;
extern bvar::LatencyRecorder g_write_latency;
extern bvar::Adder<int> g_task_pushed;
}  // bar
}  // foo
```

**不要跨文件定义全局Window或PerSecond**。不同编译单元中全局变量的初始化顺序是[未定义的](https://isocpp.org/wiki/faq/ctors#static-init-order)。在foo.cpp中定义`Adder<int> foo_count`，在foo_qps.cpp中定义`PerSecond<Adder<int> > foo_qps(&foo_count);`是**错误**的做法。

About thread-safety:

- bvar是线程兼容的。你可以在不同的线程里操作不同的bvar。比如你可以在多个线程中同时expose或hide**不同的**bvar，它们会合理地操作需要共享的全局数据，是安全的。
- **除了读写接口**，bvar的其他函数都是线程不安全的：比如说你不能在多个线程中同时expose或hide**同一个**bvar，这很可能会导致程序crash。一般来说，读写之外的其他接口也没有必要在多个线程中同时操作。

计时可以使用butil::Timer，接口如下：

```c++
#include <butil/time.h>
namespace butil {
class Timer {
public:
    enum TimerType { STARTED };

    Timer();

    // butil::Timer tm(butil::Timer::STARTED);  // tm is already started after creation.
    explicit Timer(TimerType);

    // Start this timer
    void start();

    // Stop this timer
    void stop();

    // Get the elapse from start() to stop().
    int64_t n_elapsed() const;  // in nanoseconds
    int64_t u_elapsed() const;  // in microseconds
    int64_t m_elapsed() const;  // in milliseconds
    int64_t s_elapsed() const;  // in seconds
};
}  // namespace butil
```

# bvar::Variable

Variable是所有bvar的基类，主要提供全局注册，列举，查询等功能。

用户以默认参数建立一个bvar时，这个bvar并未注册到任何全局结构中，在这种情况下，bvar纯粹是一个更快的计数器。我们称把一个bvar注册到全局表中的行为为”曝光“，可通过**expose**函数曝光：
```c++
// Expose this variable globally so that it's counted in following functions:
//   list_exposed
//   count_exposed
//   describe_exposed
//   find_exposed
// Return 0 on success, -1 otherwise.
int expose(const butil::StringPiece& name);
int expose(const butil::StringPiece& prefix, const butil::StringPiece& name);
```
全局曝光后的bvar名字便为name或prefix + name，可通过以_exposed为后缀的static函数查询。比如Variable::describe_exposed(name)会返回名为name的bvar的描述。

当相同名字的bvar已存在时，expose会打印FATAL日志并返回-1。如果选项**--bvar_abort_on_same_name**设为true (默认是false)，程序会直接abort。

下面是一些曝光bvar的例子：
```c++
bvar::Adder<int> count1;

count1 << 10 << 20 << 30;   // values add up to 60.
count1.expose("count1");  // expose the variable globally
CHECK_EQ("60", bvar::Variable::describe_exposed("count1"));
count1.expose("another_name_for_count1");  // expose the variable with another name
CHECK_EQ("", bvar::Variable::describe_exposed("count1"));
CHECK_EQ("60", bvar::Variable::describe_exposed("another_name_for_count1"));

bvar::Adder<int> count2("count2");  // exposed in constructor directly
CHECK_EQ("0", bvar::Variable::describe_exposed("count2"));  // default value of Adder<int> is 0

bvar::Status<std::string> status1("count2", "hello");  // the name conflicts. 
if -bvar_abort_on_same_name is true,
                                                       // program aborts, 
                                                       otherwise a fatal log is printed.
```

为避免重名，bvar的名字应加上前缀，建议为<namespace>_<module>_<name>。为了方便使用，我们提供了**expose_as**函数，接收一个前缀。
```c++
// Expose this variable with a prefix.
// Example:
//   namespace foo {
//   namespace bar {
//   class ApplePie {
//       ApplePie() {
//           // foo_bar_apple_pie_error
//           _error.expose_as("foo_bar_apple_pie", "error");
//       }
//   private:
//       bvar::Adder<int> _error;
//   };
//   }  // foo
//   }  // bar
int expose_as(const butil::StringPiece& prefix, const butil::StringPiece& name);
```

# Export all variables

最常见的导出需求是通过HTTP接口查询和写入本地文件。前者在brpc中通过[/vars](vars.md)服务提供，后者则已实现在bvar中，
默认不打开。有几种方法打开这个功能：

- 用[gflags](flags.md)解析输入参数，在程序启动时加入-bvar_dump，或在brpc中也可通过[/flags](flags.md)服
务在启动后动态修改。gflags的解析方法如下，在main函数处添加如下代码:

```c++
  #include <gflags/gflags.h>
  ...
  int main(int argc, char* argv[]) {
      google::ParseCommandLineFlags(&argc, &argv, true/*表示把识别的参数从argc/argv中删除*/);
      ...
  }
```

- 不想用gflags解析参数，希望直接在程序中默认打开，在main函数处添加如下代码：

```c++
#include <gflags/gflags.h>
...
int main(int argc, char* argv[]) {
    if (google::SetCommandLineOption("bvar_dump", "true").empty()) {
        LOG(FATAL) << "Fail to enable bvar dump";
    }
    ...
}
```

dump功能由如下gflags控制：

| 名称                 | 默认值                     | 作用                                       |
| ------------------ | ----------------------- | ---------------------------------------- |
| bvar_dump          | false                   | Create a background thread dumping all bvar periodically, all bvar_dump_* flags are not effective when this flag is off |
| bvar_dump_exclude  | ""                      | Dump bvar excluded from these wildcards(separated by comma), empty means no exclusion |
| bvar_dump_file     | monitor/bvar.<app>.data | Dump bvar into this file                 |
| bvar_dump_include  | ""                      | Dump bvar matching these wildcards(separated by comma), empty means including all |
| bvar_dump_interval | 10                      | Seconds between consecutive dump         |
| bvar_dump_prefix   | \<app\>                 | Every dumped name starts with this prefix |
| bvar_dump_tabs     | \<check the code\>      | Dump bvar into different tabs according to the filters (seperated by semicolon), format: *(tab_name=wildcards) |

当bvar_dump_file不为空时，程序会启动一个后台导出线程以bvar_dump_interval指定的间隔更新bvar_dump_file，其中包含了被bvar_dump_include匹配且不被bvar_dump_exclude匹配的所有bvar。

比如我们把所有的gflags修改为下图：

![img](../images/bvar_dump_flags_2.png)

导出文件为：

```
$ cat bvar.echo_server.data
rpc_server_8002_builtin_service_count : 20
rpc_server_8002_connection_count : 1
rpc_server_8002_nshead_service_adaptor : brpc::policy::NovaServiceAdaptor
rpc_server_8002_service_count : 1
rpc_server_8002_start_time : 2015/07/24-21:08:03
rpc_server_8002_uptime_ms : 14740954
```

像”`iobuf_block_count : 8`”被bvar_dump_include过滤了，“`rpc_server_8002_error : 0`”则被bvar_dump_exclude排除了。

如果你的程序没有使用brpc，仍需要动态修改gflag（一般不需要），可以调用google::SetCommandLineOption()，如下所示：
```c++
#include <gflags/gflags.h>
...
if (google::SetCommandLineOption("bvar_dump_include", "*service*").empty()) {
    LOG(ERROR) << "Fail to set bvar_dump_include";
    return -1;
}
LOG(INFO) << "Successfully set bvar_dump_include to *service*";
```
请勿直接设置FLAGS_bvar_dump_file / FLAGS_bvar_dump_include / FLAGS_bvar_dump_exclude。
一方面这些gflag类型都是std::string，直接覆盖是线程不安全的；另一方面不会触发validator（检查正确性的回调），所以也不会启动后台导出线程。

用户也可以使用dump_exposed函数自定义如何导出进程中的所有已曝光的bvar：
```c++
// Implement this class to write variables into different places.
// If dump() returns false, Variable::dump_exposed() stops and returns -1.
class Dumper {
public:
    virtual bool dump(const std::string& name, const butil::StringPiece& description) = 0;
};

// Options for Variable::dump_exposed().
struct DumpOptions {
    // Contructed with default options.
    DumpOptions();
    // If this is true, string-type values will be quoted.
    bool quote_string;
    // The ? in wildcards. Wildcards in URL need to use another character
    // because ? is reserved.
    char question_mark;
    // Separator for white_wildcards and black_wildcards.
    char wildcard_separator;
    // Name matched by these wildcards (or exact names) are kept.
    std::string white_wildcards;
    // Name matched by these wildcards (or exact names) are skipped.
    std::string black_wildcards;
};

class Variable {
    ...
    ...
    // Find all exposed variables matching `white_wildcards' but
    // `black_wildcards' and send them to `dumper'.
    // Use default options when `options' is NULL.
    // Return number of dumped variables, -1 on error.
    static int dump_exposed(Dumper* dumper, const DumpOptions* options);
};
```

# bvar::Reducer

Reducer用二元运算符把多个值合并为一个值，运算符需满足结合律，交换律，没有副作用。只有满足这三点，我们才能确保合并的结果不受线程私有数据如何分布的影响。像减法就不满足结合律和交换律，它无法作为此处的运算符。
```c++
// Reduce multiple values into one with `Op': e1 Op e2 Op e3 ...
// `Op' shall satisfy:
//   - associative:     a Op (b Op c) == (a Op b) Op c
//   - commutative:     a Op b == b Op a;
//   - no side effects: a Op b never changes if a and b are fixed.
// otherwise the result is undefined.
template <typename T, typename Op>
class Reducer : public Variable;
```
reducer << e1 << e2 << e3的作用等价于reducer = e1 op e2 op e3。

常见的Redcuer子类有bvar::Adder, bvar::Maxer, bvar::Miner。

## bvar::Adder

顾名思义，用于累加，Op为+。
```c++
bvar::Adder<int> value;
value<< 1 << 2 << 3 << -4;
CHECK_EQ(2, value.get_value());

bvar::Adder<double> fp_value;  // 可能有warning
fp_value << 1.0 << 2.0 << 3.0 << -4.0;
CHECK_DOUBLE_EQ(2.0, fp_value.get_value());
```
Adder<>可用于非基本类型，对应的类型至少要重载`T operator+(T, T)`。一个已经存在的例子是std::string，下面的代码会把string拼接起来：
```c++
// This is just proof-of-concept, don't use it for production code because it makes a
// bunch of temporary strings which is not efficient, use std::ostringstream instead.
bvar::Adder<std::string> concater;
std::string str1 = "world";
concater << "hello " << str1;
CHECK_EQ("hello world", concater.get_value());
```

## bvar::Maxer
用于取最大值，运算符为std::max。
```c++
bvar::Maxer<int> value;
value<< 1 << 2 << 3 << -4;
CHECK_EQ(3, value.get_value());
```
Since Maxer<> use std::numeric_limits<T>::min() as the identity, it cannot be applied to generic types unless you specialized std::numeric_limits<> (and overloaded operator<, yes, not operator>).

## bvar::Miner

用于取最小值，运算符为std::min。
```c++
bvar::Maxer<int> value;
value<< 1 << 2 << 3 << -4;
CHECK_EQ(-4, value.get_value());
```
Since Miner<> use std::numeric_limits<T>::max() as the identity, it cannot be applied to generic types unless you specialized std::numeric_limits<> (and overloaded operator<).

# bvar::IntRecorder

用于计算平均值。
```c++
// For calculating average of numbers.
// Example:
//   IntRecorder latency;
//   latency << 1 << 3 << 5;
//   CHECK_EQ(3, latency.average());
class IntRecorder : public Variable;
```

# bvar::LatencyRecorder

专用于计算latency和qps的计数器。只需填入latency数据，就能获得latency / max_latency / qps / count。统计窗口是最后一个参数，不填为bvar_dump_interval（这里没填）。

注意：LatencyRecorder没有继承Variable，而是多个bvar的组合。
```c++
LatencyRecorder write_latency("table2_my_table_write");  // produces 4 variables:
                                                         //   table2_my_table_write_latency
                                                         //   table2_my_table_write_max_latency
                                                         //   table2_my_table_write_qps
                                                         //   table2_my_table_write_count
// In your write function
write_latency << the_latency_of_write;
```

# bvar::Window

获得之前一段时间内的统计值。Window不能独立存在，必须依赖于一个已有的计数器。Window会自动更新，不用给它发送数据。出于性能考虑，Window的数据来自于每秒一次对原计数器的采样，在最差情况下，Window的返回值有1秒的延时。
```c++
// Get data within a time window.
// The time unit is 1 second fixed.
// Window relies on other bvar which should be constructed before this window and destructs after this window.
// R must:
// - have get_sampler() (not require thread-safe)
// - defined value_type and sampler_type
template <typename R>
class Window : public Variable;
```

# bvar::PerSecond

获得之前一段时间内平均每秒的统计值。它和Window基本相同，除了返回值会除以时间窗口之外。
```c++
bvar::Adder<int> sum;

// sum_per_second.get_value()是sum在之前60秒内*平均每秒*的累加值，省略最后一个时间窗口的话默认为bvar_dump_interval。
bvar::PerSecond<bvar::Adder<int> > sum_per_second(&sum, 60);
```
**PerSecond并不总是有意义**

上面的代码中没有Maxer，因为一段时间内的最大值除以时间窗口是没有意义的。
```c++
bvar::Maxer<int> max_value;

// 错误！最大值除以时间是没有意义的
bvar::PerSecond<bvar::Maxer<int> > max_value_per_second_wrong(&max_value);

// 正确，把Window的时间窗口设为1秒才是正确的做法
bvar::Window<bvar::Maxer<int> > max_value_per_second(&max_value, 1);
```

## 和Window的差别

比如要统计内存在上一分钟内的变化，用Window<>的话，返回值的含义是”上一分钟内存增加了18M”，用PerSecond<>的话，返回值的含义是“上一分钟平均每秒增加了0.3M”。

Window的优点是精确值，适合一些比较小的量，比如“上一分钟的错误数“，如果这用PerSecond的话，得到可能是”上一分钟平均每秒产生了0.0167个错误"，这相比于”上一分钟有1个错误“显然不够清晰。另外一些和时间无关的量也要用Window，比如统计上一分钟cpu占用率的方法是用一个Adder同时累加cpu时间和真实时间，然后用Window获得上一分钟的cpu时间和真实时间，两者相除就得到了上一分钟的cpu占用率，这和时间无关，用PerSecond会产生错误的结果。

# bvar::Status

记录和显示一个值，拥有额外的set_value函数。
```c++

// Display a rarely or periodically updated value.
// Usage:
//   bvar::Status<int> foo_count1(17);
//   foo_count1.expose("my_value");
//
//   bvar::Status<int> foo_count2;
//   foo_count2.set_value(17);
//
//   bvar::Status<int> foo_count3("my_value", 17);
//
// Notice that Tp needs to be std::string or acceptable by boost::atomic<Tp>.
template <typename Tp>
class Status : public Variable;
```

# bvar::PassiveStatus

按需显示值。在一些场合中，我们无法set_value或不知道以何种频率set_value，更适合的方式也许是当需要显示时才打印。用户传入打印回调函数实现这个目的。
```c++
// Display a updated-by-need value. This is done by passing in an user callback
// which is called to produce the value.
// Example:
//   int print_number(void* arg) {
//      ...
//      return 5;
//   }
//
//   // number1 : 5
//   bvar::PassiveStatus status1("number1", print_number, arg);
//
//   // foo_number2 : 5
//   bvar::PassiveStatus status2(typeid(Foo), "number2", print_number, arg);
template <typename Tp>
class PassiveStatus : public Variable;
```
虽然很简单，但PassiveStatus是最有用的bvar之一，因为很多统计量已经存在，我们不需要再次存储它们，而只要按需获取。比如下面的代码声明了一个在linux下显示进程用户名的bvar：
```c++

static void get_username(std::ostream& os, void*) {
    char buf[32];
    if (getlogin_r(buf, sizeof(buf)) == 0) {
        buf[sizeof(buf)-1] = '\0';
        os << buf;
    } else {
        os << "unknown";
    }
}
PassiveStatus<std::string> g_username("process_username", get_username, NULL);
```

# bvar::GFlag

Expose important gflags as bvar so that they're monitored (in noah).
```c++
DEFINE_int32(my_flag_that_matters, 8, "...");

// Expose the gflag as *same-named* bvar so that it's monitored (in noah).
static bvar::GFlag s_gflag_my_flag_that_matters("my_flag_that_matters");
//                                                ^
//                                            the gflag name

// Expose the gflag as a bvar named "foo_bar_my_flag_that_matters".
static bvar::GFlag s_gflag_my_flag_that_matters_with_prefix("foo_bar", "my_flag_that_matters");
```


# ========= ./case_apicontrol.md ========
# 进展

| 时间          | 内容                                    | 说明             |
| ----------- | ------------------------------------- | -------------- |
| 8.11 - 8.28 | 调研 + 研发 + 自测                          | 自测性能报告见附件      |
| 9.8 - 9.22  | QA测试                                  | QA测试报告见附件      |
| 10.8        | 北京机房1台机器上线                            |                |
| 10.14       | 北京机房1台机器上线                            | 修复URL编码问题      |
| 10.19       | 北京机房7/35机器上线杭州和南京各2台机器上线              | 开始小流量上线        |
| 10.22       | 北京机房10/35机器上线杭州机房5/26机器上线南京机房5/19机器上线 | 修复http响应数据压缩问题 |
| 11.3        | 北京机房10/35机器上线                         | 修复RPC内存泄露问题    |
| 11.6        | 杭州机房5/26机器上线南京机房5/19机器上线              | 同北京机房版本        |
| 11.9        | 北京机房全流量上线                             |                |

截止目前，线上服务表现稳定。

# QA测试结论

1. 【性能测试】单机支持最大QPS：**9000+**。可以有效解决原来hulu_pbrpc中一个慢服务拖垮所有服务的问题。性能很好。
2. 【稳定性测试】长时间压测没问题。

QA测试结论：通过

# 性能提升实时统计

统计时间2015.11.3 15:00 – 2015.11.9 14:30，共**143.5**小时（近6天）不间断运行。北京机房升级前和升级后同机房各6台机器，共**12**台线上机器的Noah监控数据。

| 指标       | 升级**前**均值hulu_pbrpc | 升级**后**均值brpc | 收益对比         | 说明                       |
| -------- | ------------------- | ------------- | ------------ | ------------------------ |
| CPU占用率   | 67.35%              | 29.28%        | 降低**56.53**% |                          |
| 内存占用     | 327.81MB            | 336.91MB      | 基本持平         |                          |
| 鉴权平响(ms) | 0.605               | 0.208         | 降低**65.62**% |                          |
| 转发平响(ms) | 22.49               | 23.18         | 基本持平         | 依赖后端各个服务的性能              |
| 总线程数     | 193                 | 132           | 降低**31.61**% | Baidu RPC版本线程数使用率较低，还可降低 |
| 极限QPS    | 3000                | 9000          | 提升**3**倍     | 线下使用Geoconv和Geocoder服务测试 |

**CPU使用率(%)**（红色为升级前，蓝色为升级后）
![img](../images/apicontrol_compare_1.png)

**内存使用量(KB)**（红色为升级前，蓝色为升级后）
![img](../images/apicontrol_compare_2.png)

**鉴权平响(ms)**（红色为升级前，蓝色为升级后）
![img](../images/apicontrol_compare_3.png)

**转发平响(ms)**（红色为升级前，蓝色为升级后）
![img](../images/apicontrol_compare_4.png)

**总线程数(个)**（红色为升级前，蓝色为升级后）
![img](../images/apicontrol_compare_5.png)


# ========= ./case_baidu_dsp.md ========
# 背景

baidu-dsp是联盟基于Ad Exchange和RTB模式的需求方平台，服务大客户、代理的投放产品体系。我们改造了多个模块，均取得了显著的效果。本文只介绍其中关于super-nova-as的改动。super-nova-as是的baidu-dsp的AS，之前使用ub-aserver编写，为了尽量减少改动，我们没有改造整个as，而只是把super-nova-as连接下游（ctr-server、cvr-server、super-nova-bs）的client从ubrpc升级为brpc。

# 结论

1. as的吞吐量有显著提升（不到1500 -> 2500+）
2. cpu优化：从1500qps 50%cpu_idle提高到2000qps 50% cpu_idle；
3. 超时率改善明显。

# 测试过程

1. 环境：1个as，1个bs，1个ctr，1个cvr；部署情况为：bs单机部署，as+ctr+cvr混布；ctr和cvr为brpc版本
2. 分别采用1000,1500压力对ubrpc版本的as进行压测，发现1500压力下，as对bs有大量的超时，as到达瓶颈；
3. 分别采用2000,2500压力对brpc版本的as进行压测，发现2500压力下，as机器的cpu_idle低于30%，as到达瓶颈。brpc对资源利用充分。

|          | ubrpc                                    | brpc                                |
| -------- | ---------------------------------------- | ---------------------------------------- |
| 流量       | ![img](../images/baidu_dsp_compare_1.png) | ![img](../images/baidu_dsp_compare_2.png) |
| bs成功率    | ![img](../images/baidu_dsp_compare_3.png) | ![img](../images/baidu_dsp_compare_4.png) |
| cpu_idle | ![img](../images/baidu_dsp_compare_5.png) | ![img](../images/baidu_dsp_compare_6.png) |
| ctr成功率   | ![img](../images/baidu_dsp_compare_7.png) | ![img](../images/baidu_dsp_compare_8.png) |
| cvr成功率   | ![img](../images/baidu_dsp_compare_9.png) | ![img](../images/baidu_dsp_compare_10.png) |


# ========= ./case_elf.md ========
# 背景

ELF(Essential/Extreme/Excellent Learning Framework) 框架为公司内外的大数据应用提供学习/挖掘算法开发支持。 平台主要包括数据迭代处理的框架支持，并行计算过程中的通信支持和用于存储大规模参数的分布式、快速、高可用参数服务器。应用于fcr-model，公有云bml，大数据实验室，语音技术部门等等。之前是基于[zeromq](http://zeromq.org/)封装的rpc，这次改用brpc。

# 结论

单个rpc-call以及单次请求所有rpc-call的提升非常显著，延时都缩短了**50%**以上，总训练的耗时除了ftrl-sync-no-shuffle提升不明显外，其余三个算法训练总体性能都有**15%**左右的提升。现在瓶颈在于计算逻辑，所以相对单个的rpc的提升没有那么多。该成果后续会推动凤巢模型训练的上线替换。详细耗时见下节。

# 性能对比报告

## 算法总耗时

ftrl算法: 替换前总耗时2:4:39, 替换后总耗时1:42:48, 提升18%

ftrl-sync-no-shuffle算法: 替换前总耗时3:20:47, 替换后总耗时3:15:28, 提升2.5%

ftrl-sync算法: 替换前总耗时4:28:47, 替换后总耗时3:45:57, 提升16%

fm-sync算法: 替换前总耗时6:16:37, 替换后总耗时5:21:00, 提升14.6%

## 子任务耗时

单个rpc-call(针对ftrl算法)

|      | Average   | Max       | Min        |
| ---- | --------- | --------- | ---------- |
| 替换前  | 164.946ms | 7938.76ms | 0.249756ms |
| 替换后  | 10.4198ms | 2614.38ms | 0.076416ms |
| 缩短   | 93%       | 67%       | 70%        |

单次请求所有rpc-call(针对ftrl算法)

|      | Average  | Max      | Min       |
| ---- | -------- | -------- | --------- |
| 替换前  | 658.08ms | 7123.5ms | 1.88159ms |
| 替换后  | 304.878  | 2571.34  | 0         |
| 缩短   | 53.7%    | 63.9%    |           |

## 结论

单个rpc-call以及单次请求所有rpc-call的提升非常显著，提升都在50%以上，总任务的耗时除了ftrl-sync-no-shuffle提升不明显外，其余三个算法都有15%左右的提升，现在算法的瓶颈在于对计算逻辑，所以相对单个的rpc的提升没有那么多。

附cpu profiling结果, top 40没有rpc相关函数。
```
Total: 8664 samples
     755   8.7%   8.7%      757   8.7% baidu::elf::Partitioner
     709   8.2%  16.9%      724   8.4% baidu::elf::GlobalShardWriterClient::local_aggregator::{lambda#1}::operator [clone .part.1341]
     655   7.6%  24.5%      655   7.6% boost::detail::lcast_ret_unsigned
     582   6.7%  31.2%      642   7.4% baidu::elf::RobinHoodLinkedHashMap
     530   6.1%  37.3%     2023  23.4% std::vector
     529   6.1%  43.4%      529   6.1% std::binary_search
     406   4.7%  48.1%      458   5.3% tc_delete
     405   4.7%  52.8%     2454  28.3% baidu::elf::GlobalShardWriterClient
     297   3.4%  56.2%      297   3.4% __memcpy_sse2_unaligned
     256   3.0%  59.2%      297   3.4% tc_new
     198   2.3%  61.5%      853   9.8% std::__introsort_loop
     157   1.8%  63.3%      157   1.8% baidu::elf::GrowableArray
     152   1.8%  65.0%      177   2.0% calculate_grad
     142   1.6%  66.7%      699   8.1% baidu::elf::BlockTableReaderManager
     137   1.6%  68.2%      656   7.6% baidu::elf::DefaultWriterServer
     127   1.5%  69.7%      127   1.5% _init
     122   1.4%  71.1%      582   6.7% __gnu_cxx::__normal_iterator
     117   1.4%  72.5%      123   1.4% baidu::elf::GrowableArray::emplace_back
     116   1.3%  73.8%      116   1.3% baidu::elf::RobinHoodHashMap::insert
     101   1.2%  75.0%      451   5.2% baidu::elf::NoCacheReaderClient
      99   1.1%  76.1%     3614  41.7% parse_ins
      97   1.1%  77.2%       97   1.1% std::basic_string::_Rep::_M_dispose [clone .part.12]
      96   1.1%  78.3%      154   1.8% std::basic_string
      91   1.1%  79.4%      246   2.8% boost::algorithm::split_iterator
      87   1.0%  80.4%      321   3.7% boost::function2
      76   0.9%  81.3%      385   4.4% boost::detail::function::functor_manager
      69   0.8%  82.1%       69   0.8% std::locale::~locale
      63   0.7%  82.8%      319   3.7% std::__unguarded_linear_insert
      58   0.7%  83.5%     2178  25.2% boost::algorithm::split [clone .constprop.2471]
      54   0.6%  84.1%      100   1.2% std::vector::_M_emplace_back_aux
      49   0.6%  84.7%       49   0.6% boost::algorithm::detail::is_any_ofF
      47   0.5%  85.2%       79   0.9% baidu::elf::DefaultReaderServer
      41   0.5%  85.7%       41   0.5% std::locale::_S_initialize
      39   0.5%  86.1%      677   7.8% boost::detail::function::function_obj_invoker2
      39   0.5%  86.6%       39   0.5% memset
      39   0.5%  87.0%       39   0.5% std::locale::locale
      38   0.4%  87.5%       50   0.6% FTRLAggregator::serialize
      36   0.4%  87.9%       67   0.8% tcmalloc::CentralFreeList::ReleaseToSpans
      34   0.4%  88.3%       34   0.4% madvise
      34   0.4%  88.7%       38   0.4% tcmalloc::CentralFreeList::FetchFromOneSpans
      32   0.4%  89.0%       32   0.4% std::__insertion_sort
```


# ========= ./case_ubrpc.md ========
# 背景

云平台部把使用ubrpc的模块改造为使用brpc。由于使用了mcpack2pb的转换功能，这个模块既能被老的ubrpc client访问，也可以通过protobuf类的协议访问（baidu_std，sofa_pbrpc等）。

原有使用43台机器（对ubrpc也有富余），brpc使用3台机器即可（此时访问redis的io达到瓶颈）。当前流量4w qps，支持流量增长，考虑跨机房冗余，避免redis和vip瓶颈，brpc实际使用8台机器提供服务。 

brpc改造后的connecter收益明显，可以用较少的机器提供更优质的服务。收益分3个方面：

# 相同配置的机器qps和latency的比较 

通过逐渐缩容，不断增加connecter的压力，获得单机qps和latency的对应数据如下： 
![img](../images/ubrpc_compare_1.png)

机器配置：cpu: 24 Intel(R) Xeon(R) CPU  E5645  @ 2.40GHz || mem: 64G 

混布情况：同机部署了逻辑层2.0/3.0和C逻辑层，均有流量 

图中可以看到随着压力的增大：
* brpc的延时，增加微乎其微，提供了较为一致的延时体验
* ubrpc的延时，快速增大，到了6000~8000qps的时候，出现*queue full*，服务不可用。

# 不同配置机器qps和延时的比较
qps固定为6500，观察延时。

| 机器名称  | 略                                        | 略                                        |
| ----- | ---------------------------------------- | ---------------------------------------- |
| cpu   | 24 Intel(R) Xeon(R) CPU E5645  @ 2.40GHz | 24 Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz |
| ubrpc | 8363.46（us）                              | 12649.5（us）                              |
| brpc  | 3364.66（us）                              | 3382.15（us）                              |

有此可见： 

* ubrpc在不同配置下性能表现差异大，在配置较低的机器下表现较差。
* brpc表现的比ubrpc好，在较低配置的机器上也能有好的表现，因机器不同带来的差异不大。

# 相同配置机器idle分布的比较 

机器配置：cpu： 24 Intel(R) Xeon(R) CPU  E5645  @ 2.40GHz || mem：64G 

![img](../images/ubrpc_compare_2.png)

在线上缩容 不断增大压力过程中：

* ubrpc cpu idle分布在35%~60%，在55%最集中，最低30%； 
* brpc cpu idle分布在60%~85%，在75%最集中，最低50%； brpc比ubrpc对cpu的消耗低。


# ========= ./client.md ========
[English version](../en/client.md)

# 示例程序

Echo的[client端代码](https://github.com/brpc/brpc/blob/master/example/echo_c++/client.cpp)。

# 事实速查

- Channel.Init()是线程不安全的。
- Channel.CallMethod()是线程安全的，一个Channel可以被所有线程同时使用。
- Channel可以分配在栈上。
- Channel在发送异步请求后可以析构。
- 没有brpc::Client这个类。

# Channel
Client指发起请求的一端，在brpc中没有对应的实体，取而代之的是[brpc::Channel](https://github.com/brpc/brpc/blob/master/src/brpc/channel.h)，它代表和一台或一组服务器的交互通道，Client和Channel在角色上的差别在实践中并不重要，你可以把Channel视作Client。

Channel可以**被所有线程共用**，你不需要为每个线程创建独立的Channel，也不需要用锁互斥。不过Channel的创建和Init并不是线程安全的，请确保在Init成功后再被多线程访问，在没有线程访问后再析构。

一些RPC实现中有ClientManager的概念，包含了Client端的配置信息和资源管理。brpc不需要这些，以往在ClientManager中配置的线程数、长短连接等等要么被加入了brpc::ChannelOptions，要么可以通过gflags全局配置，这么做的好处：

1. 方便。你不需要在创建Channel时传入ClientManager，也不需要存储ClientManager。否则不少代码需要一层层地传递ClientManager，很麻烦。gflags使一些全局行为的配置更加简单。
2. 共用资源。比如server和channel可以共用后台线程。(bthread的工作线程)
3. 生命周期。析构ClientManager的过程很容易出错，现在由框架负责则不会有问题。

就像大部分类那样，Channel必须在**Init**之后才能使用，options为NULL时所有参数取默认值，如果你要使用非默认值，这么做就行了：
```c++
brpc::ChannelOptions options;  // 包含了默认值
options.xxx = yyy;
...
channel.Init(..., &options);
```
注意Channel不会修改options，Init结束后不会再访问options。所以options一般就像上面代码中那样放栈上。Channel.options()可以获得channel在使用的所有选项。

Init函数分为连接一台服务器和连接服务集群。

# 连接一台服务器

```c++
// options为NULL时取默认值
int Init(EndPoint server_addr_and_port, const ChannelOptions* options);
int Init(const char* server_addr_and_port, const ChannelOptions* options);
int Init(const char* server_addr, int port, const ChannelOptions* options);
```
这类Init连接的服务器往往有固定的ip地址，不需要命名服务和负载均衡，创建起来相对轻量。但是**请勿频繁创建使用域名的Channel**。这需要查询dns，可能最多耗时10秒(查询DNS的默认超时)。重用它们。

合法的“server_addr_and_port”：
- 127.0.0.1:80
- www.foo.com:8765
- localhost:9000

不合法的"server_addr_and_port"：
- 127.0.0.1:90000     # 端口过大
- 10.39.2.300:8000   # 非法的ip

# 连接服务集群

```c++
int Init(const char* naming_service_url,
         const char* load_balancer_name,
         const ChannelOptions* options);
```
这类Channel需要定期从`naming_service_url`指定的命名服务中获得服务器列表，并通过`load_balancer_name`指定的负载均衡算法选择出一台机器发送请求。

你**不应该**在每次请求前动态地创建此类（连接服务集群的）Channel。因为创建和析构此类Channel牵涉到较多的资源，比如在创建时得访问一次命名服务，否则便不知道有哪些服务器可选。由于Channel可被多个线程共用，一般也没有必要动态创建。

当`load_balancer_name`为NULL或空时，此Init等同于连接单台server的Init，`naming_service_url`应该是"ip:port"或"域名:port"。你可以通过这个Init函数统一Channel的初始化方式。比如你可以把`naming_service_url`和`load_balancer_name`放在配置文件中，要连接单台server时把`load_balancer_name`置空，要连接服务集群时则设置一个有效的算法名称。

## 命名服务

命名服务把一个名字映射为可修改的机器列表，在client端的位置如下：

![img](../images/ns.png)

有了命名服务后client记录的是一个名字，而不是每一台下游机器。而当下游机器变化时，就只需要修改命名服务中的列表，而不需要逐台修改每个上游。这个过程也常被称为“解耦上下游”。当然在具体实现上，上游会记录每一台下游机器，并定期向命名服务请求或被推送最新的列表，以避免在RPC请求时才去访问命名服务。使用命名服务一般不会对访问性能造成影响，对命名服务的压力也很小。

`naming_service_url`的一般形式是"**protocol://service_name**"

### bns://\<bns-name\>

BNS是百度内常用的命名服务，比如bns://rdev.matrix.all，其中"bns"是protocol，"rdev.matrix.all"是service-name。相关一个gflag是-ns_access_interval: ![img](../images/ns_access_interval.png)

如果BNS中显示不为空，但Channel却说找不到服务器，那么有可能BNS列表中的机器状态位（status）为非0，含义为机器不可用，所以不会被加入到server候选集中．状态位可通过命令行查看：

`get_instance_by_service [bns_node_name] -s`

### file://\<path\>

服务器列表放在`path`所在的文件里，比如"file://conf/machine_list"中的“conf/machine_list”对应一个文件:
 * 每行是一台服务器的地址。
 * \#之后的是注释会被忽略
 * 地址后出现的非注释内容被认为是tag，由一个或多个空格与前面的地址分隔，相同的地址+不同的tag被认为是不同的实例。
 * 当文件更新时, brpc会重新加载。
```
# 此行会被忽略
10.24.234.17 tag1  # 这是注释，会被忽略
10.24.234.17 tag2  # 此行和上一行被认为是不同的实例
10.24.234.18
10.24.234.19
```

优点: 易于修改，方便单测。

缺点: 更新时需要修改每个上游的列表文件，不适合线上部署。

### list://\<addr1\>,\<addr2\>...

服务器列表直接跟在list://之后，以逗号分隔，比如"list://db-bce-81-3-186.db01:7000,m1-bce-44-67-72.m1:7000,cp01-rd-cos-006.cp01:7000"中有三个地址。

地址后可以声明tag，用一个或多个空格分隔，相同的地址+不同的tag被认为是不同的实例。

优点: 可在命令行中直接配置，方便单测。

缺点: 无法在运行时修改，完全不能用于线上部署。

### http://\<url\>

连接一个域名下所有的机器, 例如http://www.baidu.com:80 ，注意连接单点的Init（两个参数）虽然也可传入域名，但只会连接域名下的一台机器。

优点: DNS的通用性，公网内网均可使用。

缺点: 受限于DNS的格式限制无法传递复杂的meta数据，也无法实现通知机制。

### https://\<url\>

和http前缀类似，只是会自动开启SSL。

### consul://\<service-name\>

通过consul获取服务名称为service-name的服务列表。consul的默认地址是localhost:8500，可通过gflags设置-consul\_agent\_addr来修改。consul的连接超时时间默认是200ms，可通过-consul\_connect\_timeout\_ms来修改。

默认在consul请求参数中添加[stale](https://www.consul.io/api/index.html#consistency-modes)和passing（仅返回状态为passing的服务列表），可通过gflags中-consul\_url\_parameter改变[consul请求参数](https://www.consul.io/api/health.html#parameters-2)。

除了对consul的首次请求，后续对consul的请求都采用[long polling](https://www.consul.io/api/index.html#blocking-queries)的方式，即仅当服务列表更新或请求超时后consul才返回结果，这里超时时间默认为60s，可通过-consul\_blocking\_query\_wait\_secs来设置。

若consul返回的服务列表[响应格式](https://www.consul.io/api/health.html#sample-response-2)有错误，或者列表中所有服务都因为地址、端口等关键字段缺失或无法解析而被过滤，consul naming server会拒绝更新服务列表，并在一段时间后（默认500ms，可通过-consul\_retry\_interval\_ms设置）重新访问consul。

如果consul不可访问，服务可自动降级到file naming service获取服务列表。此功能默认关闭，可通过设置-consul\_enable\_degrade\_to\_file\_naming\_service来打开。服务列表文件目录通过-consul \_file\_naming\_service\_dir来设置，使用service-name作为文件名。该文件可通过consul-template生成，里面会保存consul不可用之前最新的下游服务节点。当consul恢复时可自动恢复到consul naming service。

### 更多命名服务
用户可以通过实现brpc::NamingService来对接更多命名服务，具体见[这里](https://github.com/brpc/brpc/blob/master/docs/cn/load_balancing.md#%E5%91%BD%E5%90%8D%E6%9C%8D%E5%8A%A1)

### 命名服务中的tag
每个地址可以附带一个tag，在常见的命名服务中，如果地址后有空格，则空格之后的内容均为tag。
相同的地址配合不同的tag被认为是不同的实例，brpc会建立不同的连接。用户可利用这个特性更灵活地控制与单个地址的连接方式。
如果你需要"带权重的轮询"，你应当优先考虑使用[wrr算法](#wrr)，而不是用tag来模拟。

### VIP相关的问题
VIP一般是4层负载均衡器的公网ip，背后有多个RS。当客户端连接至VIP时，VIP会选择一个RS建立连接，当客户端连接断开时，VIP也会断开与对应RS的连接。

如果客户端只与VIP建立一个连接(brpc中的单连接)，那么来自这个客户端的所有流量都会落到一台RS上。如果客户端的数量非常多，至少在集群的角度，所有的RS还是会分到足够多的连接，从而基本均衡。但如果客户端的数量不多，或客户端的负载差异很大，那么可能在个别RS上出现热点。另一个问题是当有多个VIP可选时，客户端分给它们的流量与各自后面的RS数量可能不一致。

解决这个问题的一种方法是使用连接池模式(pooled)，这样客户端对一个VIP就可能建立多个连接(约为一段时间内的最大并发度)，从而让负载落到多个RS上。如果有多个VIP，可以用[wrr负载均衡](#wrr)给不同的VIP声明不同的权重从而分到对应比例的流量，或给相同的VIP后加上多个不同的tag而被认为是多个不同的实例。

如果对性能有更高的要求，或要限制大集群中连接的数量，可以使用单连接并给相同的VIP加上不同的tag以建立多个连接。相比连接池一般连接数量更小，系统调用开销更低，但如果tag不够多，仍可能出现RS热点。

### 命名服务过滤器

当命名服务获得机器列表后，可以自定义一个过滤器进行筛选，最后把结果传递给负载均衡：

![img](../images/ns_filter.jpg)

过滤器的接口如下：
```c++
// naming_service_filter.h
class NamingServiceFilter {
public:
    // Return true to take this `server' as a candidate to issue RPC
    // Return false to filter it out
    virtual bool Accept(const ServerNode& server) const = 0;
};
 
// naming_service.h
struct ServerNode {
    butil::EndPoint addr;
    std::string tag;
};
```
常见的业务策略如根据server的tag进行过滤。

自定义的过滤器配置在ChannelOptions中，默认为NULL（不过滤）。

```c++
class MyNamingServiceFilter : public brpc::NamingServiceFilter {
public:
    bool Accept(const brpc::ServerNode& server) const {
        return server.tag == "main";
    }
};
 
int main() {
    ...
    MyNamingServiceFilter my_filter;
    ...
    brpc::ChannelOptions options;
    options.ns_filter = &my_filter;
    ...
}
```

## 负载均衡

当下游机器超过一台时，我们需要分割流量，此过程一般称为负载均衡，在client端的位置如下图所示：

![img](../images/lb.png)

理想的算法是每个请求都得到及时的处理，且任意机器crash对全局影响较小。但由于client端无法及时获得server端的延迟或拥塞，而且负载均衡算法不能耗费太多的cpu，一般来说用户得根据具体的场景选择合适的算法，目前rpc提供的算法有（通过load_balancer_name指定）：

### rr

即round robin，总是选择列表中的下一台服务器，结尾的下一台是开头，无需其他设置。比如有3台机器a,b,c，那么brpc会依次向a, b, c, a, b, c, ...发送请求。注意这个算法的前提是服务器的配置，网络条件，负载都是类似的。

### wrr

即weighted round robin, 根据服务器列表配置的权重值来选择服务器。服务器被选到的机会正比于其权重值，并且该算法能保证同一服务器被选到的结果较均衡的散开。

### random

随机从列表中选择一台服务器，无需其他设置。和round robin类似，这个算法的前提也是服务器都是类似的。

### la

locality-aware，优先选择延时低的下游，直到其延时高于其他机器，无需其他设置。实现原理请查看[Locality-aware load balancing](lalb.md)。

### c_murmurhash or c_md5

一致性哈希，与简单hash的不同之处在于增加或删除机器时不会使分桶结果剧烈变化，特别适合cache类服务。

发起RPC前需要设置Controller.set_request_code()，否则RPC会失败。request_code一般是请求中主键部分的32位哈希值，**不需要和负载均衡使用的哈希算法一致**。比如用c_murmurhash算法也可以用md5计算哈希值。

[src/brpc/policy/hasher.h](https://github.com/brpc/brpc/blob/master/src/brpc/policy/hasher.h)中包含了常用的hash函数。如果用std::string key代表请求的主键，controller.set_request_code(brpc::policy::MurmurHash32(key.data(), key.size()))就正确地设置了request_code。

注意甄别请求中的“主键”部分和“属性”部分，不要为了偷懒或通用，就把请求的所有内容一股脑儿计算出哈希值，属性的变化会使请求的目的地发生剧烈的变化。另外也要注意padding问题，比如struct Foo { int32_t a; int64_t b; }在64位机器上a和b之间有4个字节的空隙，内容未定义，如果像hash(&foo, sizeof(foo))这样计算哈希值，结果就是未定义的，得把内容紧密排列或序列化后再算。

实现原理请查看[Consistent Hashing](consistent_hashing.md)。

## 健康检查

连接断开的server会被暂时隔离而不会被负载均衡算法选中，brpc会定期连接被隔离的server，以检查他们是否恢复正常，间隔由参数-health_check_interval控制:

| Name                      | Value | Description                              | Defined At              |
| ------------------------- | ----- | ---------------------------------------- | ----------------------- |
| health_check_interval （R） | 3     | seconds between consecutive health-checkings | src/brpc/socket_map.cpp |

一旦server被连接上，它会恢复为可用状态。如果在隔离过程中，server从命名服务中删除了，brpc也会停止连接尝试。

# 发起访问

一般来说，我们不直接调用Channel.CallMethod，而是通过protobuf生成的桩XXX_Stub，过程更像是“调用函数”。stub内没什么成员变量，建议在栈上创建和使用，而不必new，当然你也可以把stub存下来复用。Channel::CallMethod和stub访问都是**线程安全**的，可以被所有线程同时访问。比如：
```c++
XXX_Stub stub(&channel);
stub.some_method(controller, request, response, done);
```
甚至
```c++
XXX_Stub(&channel).some_method(controller, request, response, done);
```
一个例外是http/h2 client。访问http服务和protobuf没什么关系，直接调用CallMethod即可，除了Controller和done均为NULL，详见[访问http/h2服务](http_client.md)。

## 同步访问

指的是：CallMethod会阻塞到收到server端返回response或发生错误（包括超时）。

同步访问中的response/controller不会在CallMethod后被框架使用，它们都可以分配在栈上。注意，如果request/response字段特别多字节数特别大的话，还是更适合分配在堆上。
```c++
MyRequest request;
MyResponse response;
brpc::Controller cntl;
XXX_Stub stub(&channel);
 
request.set_foo(...);
cntl.set_timeout_ms(...);
stub.some_method(&cntl, &request, &response, NULL);
if (cntl->Failed()) {
    // RPC失败了. response里的值是未定义的，勿用。
} else {
    // RPC成功了，response里有我们想要的回复数据。
}
```

## 异步访问

指的是：给CallMethod传递一个额外的回调对象done，CallMethod在发出request后就结束了，而不是在RPC结束后。当server端返回response或发生错误（包括超时）时，done->Run()会被调用。对RPC的后续处理应该写在done->Run()里，而不是CallMethod后。

由于CallMethod结束不意味着RPC结束，response/controller仍可能被框架及done->Run()使用，它们一般得创建在堆上，并在done->Run()中删除。如果提前删除了它们，那当done->Run()被调用时，将访问到无效内存。

你可以独立地创建这些对象，并使用[NewCallback](#使用NewCallback)生成done，也可以把Response和Controller作为done的成员变量，[一起new出来](#继承google::protobuf::Closure)，一般使用前一种方法。

**发起异步请求后Request和Channel也可以立刻析构**。这两样和response/controller是不同的。注意:这是说Channel的析构可以立刻发生在CallMethod**之后**，并不是说析构可以和CallMethod同时发生，删除正被另一个线程使用的Channel是未定义行为（很可能crash）。

### 使用NewCallback
```c++
static void OnRPCDone(MyResponse* response, brpc::Controller* cntl) {
    // unique_ptr会帮助我们在return时自动删掉response/cntl，防止忘记。gcc 3.4下的unique_ptr是模拟版本。
    std::unique_ptr<MyResponse> response_guard(response);
    std::unique_ptr<brpc::Controller> cntl_guard(cntl);
    if (cntl->Failed()) {
        // RPC失败了. response里的值是未定义的，勿用。
    } else {
        // RPC成功了，response里有我们想要的数据。开始RPC的后续处理。    
    }
    // NewCallback产生的Closure会在Run结束后删除自己，不用我们做。
}
 
MyResponse* response = new MyResponse;
brpc::Controller* cntl = new brpc::Controller;
MyService_Stub stub(&channel);
 
MyRequest request;  // 你不用new request,即使在异步访问中.
request.set_foo(...);
cntl->set_timeout_ms(...);
stub.some_method(cntl, &request, response, google::protobuf::NewCallback(OnRPCDone, response, cntl));
```
由于protobuf 3把NewCallback设置为私有，r32035后brpc把NewCallback独立于[src/brpc/callback.h](https://github.com/brpc/brpc/blob/master/src/brpc/callback.h)（并增加了一些重载）。如果你的程序出现NewCallback相关的编译错误，把google::protobuf::NewCallback替换为brpc::NewCallback就行了。

### 继承google::protobuf::Closure

使用NewCallback的缺点是要分配三次内存：response, controller, done。如果profiler证明这儿的内存分配有瓶颈，可以考虑自己继承Closure，把response/controller作为成员变量，这样可以把三次new合并为一次。但缺点就是代码不够美观，如果内存分配不是瓶颈，别用这种方法。
```c++
class OnRPCDone: public google::protobuf::Closure {
public:
    void Run() {
        // unique_ptr会帮助我们在return时自动delete this，防止忘记。gcc 3.4下的unique_ptr是模拟版本。
        std::unique_ptr<OnRPCDone> self_guard(this);
          
        if (cntl->Failed()) {
            // RPC失败了. response里的值是未定义的，勿用。
        } else {
            // RPC成功了，response里有我们想要的数据。开始RPC的后续处理。
        }
    }
 
    MyResponse response;
    brpc::Controller cntl;
}
 
OnRPCDone* done = new OnRPCDone;
MyService_Stub stub(&channel);
 
MyRequest request;  // 你不用new request,即使在异步访问中.
request.set_foo(...);
done->cntl.set_timeout_ms(...);
stub.some_method(&done->cntl, &request, &done->response, done);
```

### 如果异步访问中的回调函数特别复杂会有什么影响吗?

没有特别的影响，回调会运行在独立的bthread中，不会阻塞其他的逻辑。你可以在回调中做各种阻塞操作。

### rpc发送处的代码和回调函数是在同一个线程里执行吗?

一定不在同一个线程里运行，即使该次rpc调用刚进去就失败了，回调也会在另一个bthread中运行。这可以在加锁进行rpc（不推荐）的代码中避免死锁。

## 等待RPC完成
注意：当你需要发起多个并发操作时，可能[ParallelChannel](combo_channel.md#parallelchannel)更方便。

如下代码发起两个异步RPC后等待它们完成。
```c++
const brpc::CallId cid1 = controller1->call_id();
const brpc::CallId cid2 = controller2->call_id();
...
stub.method1(controller1, request1, response1, done1);
stub.method2(controller2, request2, response2, done2);
...
brpc::Join(cid1);
brpc::Join(cid2);
```
**在发起RPC前**调用Controller.call_id()获得一个id，发起RPC调用后Join那个id。

Join()的行为是等到RPC结束**且done->Run()运行后**，一些Join的性质如下：

- 如果对应的RPC已经结束，Join将立刻返回。
- 多个线程可以Join同一个id，它们都会醒来。
- 同步RPC也可以在另一个线程中被Join，但一般不会这么做。 

Join()在之前的版本叫做JoinResponse()，如果你在编译时被提示deprecated之类的，修改为Join()。

在RPC调用后`Join(controller->call_id())`是**错误**的行为，一定要先把call_id保存下来。因为RPC调用后controller可能被随时开始运行的done删除。下面代码的Join方式是**错误**的。

```c++
static void on_rpc_done(Controller* controller, MyResponse* response) {
    ... Handle response ...
    delete controller;
    delete response;
}
 
Controller* controller1 = new Controller;
Controller* controller2 = new Controller;
MyResponse* response1 = new MyResponse;
MyResponse* response2 = new MyResponse;
...
stub.method1(controller1, &request1, response1, google::protobuf::NewCallback(on_rpc_done, controller1, response1));
stub.method2(controller2, &request2, response2, google::protobuf::NewCallback(on_rpc_done, controller2, response2));
...
brpc::Join(controller1->call_id());   // 错误，controller1可能被on_rpc_done删除了
brpc::Join(controller2->call_id());   // 错误，controller2可能被on_rpc_done删除了
```

## 半同步

Join可用来实现“半同步”访问：即等待多个异步访问完成。由于调用处的代码会等到所有RPC都结束后再醒来，所以controller和response都可以放栈上。
```c++
brpc::Controller cntl1;
brpc::Controller cntl2;
MyResponse response1;
MyResponse response2;
...
stub1.method1(&cntl1, &request1, &response1, brpc::DoNothing());
stub2.method2(&cntl2, &request2, &response2, brpc::DoNothing());
...
brpc::Join(cntl1.call_id());
brpc::Join(cntl2.call_id());
```
brpc::DoNothing()可获得一个什么都不干的done，专门用于半同步访问。它的生命周期由框架管理，用户不用关心。

注意在上面的代码中，我们在RPC结束后又访问了controller.call_id()，这是没有问题的，因为DoNothing中并不会像上节中的on_rpc_done中那样删除Controller。

## 取消RPC

brpc::StartCancel(call_id)可取消对应的RPC，call_id必须**在发起RPC前**通过Controller.call_id()获得，其他时刻都可能有race condition。

注意：是brpc::StartCancel(call_id)，不是controller->StartCancel()，后者被禁用，没有效果。后者是protobuf默认提供的接口，但是在controller对象的生命周期上有严重的竞争问题。

顾名思义，StartCancel调用完成后RPC并未立刻结束，你不应该碰触Controller的任何字段或删除任何资源，它们自然会在RPC结束时被done中对应逻辑处理。如果你一定要在原地等到RPC结束（一般不需要），则可通过Join(call_id)。

关于StartCancel的一些事实：

- call_id在发起RPC前就可以被取消，RPC会直接结束（done仍会被调用）。
- call_id可以在另一个线程中被取消。
- 取消一个已经取消的call_id不会有任何效果。推论：同一个call_id可以被多个线程同时取消，但最多一次有效果。
- 这里的取消是纯client端的功能，**server端未必会取消对应的操作**，server cancelation是另一个功能。

## 获取Server的地址和端口

remote_side()方法可知道request被送向了哪个server，返回值类型是[butil::EndPoint](https://github.com/brpc/brpc/blob/master/src/butil/endpoint.h)，包含一个ip4地址和端口。在RPC结束前调用这个方法都是没有意义的。

打印方式：
```c++
LOG(INFO) << "remote_side=" << cntl->remote_side();
printf("remote_side=%s\n", butil::endpoint2str(cntl->remote_side()).c_str());
```
## 获取Client的地址和端口

r31384后通过local_side()方法可**在RPC结束后**获得发起RPC的地址和端口。

打印方式：
```c++
LOG(INFO) << "local_side=" << cntl->local_side(); 
printf("local_side=%s\n", butil::endpoint2str(cntl->local_side()).c_str());
```
## 应该重用brpc::Controller吗?

不用刻意地重用，但Controller是个大杂烩，可能会包含一些缓存，Reset()可以避免反复地创建这些缓存。

在大部分场景下，构造Controller(snippet1)和重置Controller(snippet2)的性能差异不大。
```c++
// snippet1
for (int i = 0; i < n; ++i) {
    brpc::Controller controller;
    ...
    stub.CallSomething(..., &controller);
}
 
// snippet2
brpc::Controller controller;
for (int i = 0; i < n; ++i) {
    controller.Reset();
    ...
    stub.CallSomething(..., &controller);
}
```
但如果snippet1中的Controller是new出来的，那么snippet1就会多出“内存分配”的开销，在一些情况下可能会慢一些。

# 设置

Client端的设置主要由三部分组成：

- brpc::ChannelOptions: 定义在[src/brpc/channel.h](https://github.com/brpc/brpc/blob/master/src/brpc/channel.h)中，用于初始化Channel，一旦初始化成功无法修改。
- brpc::Controller: 定义在[src/brpc/controller.h](https://github.com/brpc/brpc/blob/master/src/brpc/controller.h)中，用于在某次RPC中覆盖ChannelOptions中的选项，可根据上下文每次均不同。
- 全局gflags：常用于调节一些底层代码的行为，一般不用修改。请自行阅读服务[/flags页面](flags.md)中的说明。

Controller包含了request中没有的数据和选项。server端和client端的Controller结构体是一样的，但使用的字段可能是不同的，你需要仔细阅读Controller中的注释，明确哪些字段可以在server端使用，哪些可以在client端使用。

一个Controller对应一次RPC。一个Controller可以在Reset()后被另一个RPC复用，但一个Controller不能被多个RPC同时使用（不论是否在同一个线程发起）。

Controller的特点：
1. 一个Controller只能有一个使用者，没有特殊说明的话，Controller中的方法默认线程不安全。
2. 因为不能被共享，所以一般不会用共享指针管理Controller，如果你用共享指针了，很可能意味着出错了。
3. Controller创建于开始RPC前，析构于RPC结束后，常见几种模式：
   - 同步RPC前Controller放栈上，出作用域后自行析构。注意异步RPC的Controller绝对不能放栈上，否则其析构时异步调用很可能还在进行中，从而引发未定义行为。
   - 异步RPC前new Controller，done中删除。

## 线程数

和大部分的RPC框架不同，brpc中并没有独立的Client线程池。所有Channel和Server通过[bthread](http://wiki.baidu.com/display/RPC/bthread)共享相同的线程池. 如果你的程序同样使用了brpc的server, 仅仅需要设置Server的线程数。 或者可以通过[gflags](http://wiki.baidu.com/display/RPC/flags)设置[-bthread_concurrency](http://brpc.baidu.com:8765/flags/bthread_concurrency)来设置全局的线程数.

## 超时

**ChannelOptions.timeout_ms**是对应Channel上所有RPC的总超时，Controller.set_timeout_ms()可修改某次RPC的值。单位毫秒，默认值1秒，最大值2^31（约24天），-1表示一直等到回复或错误。

**ChannelOptions.connect_timeout_ms**是对应Channel上所有RPC的连接超时，单位毫秒，默认值1秒。-1表示等到连接建立或出错，此值被限制为不能超过timeout_ms。注意此超时独立于TCP的连接超时，一般来说前者小于后者，反之则可能在connect_timeout_ms未达到前由于TCP连接超时而出错。

注意1：brpc中的超时是deadline，超过就意味着RPC结束，超时后没有重试。其他实现可能既有单次访问的超时，也有代表deadline的超时。迁移到brpc时请仔细区分。

注意2：RPC超时的错误码为**ERPCTIMEDOUT (1008)**，ETIMEDOUT的意思是连接超时，且可重试。

## 重试

ChannelOptions.max_retry是该Channel上所有RPC的默认最大重试次数，Controller.set_max_retry()可修改某次RPC的值，默认值3，0表示不重试。

r32111后Controller.retried_count()返回重试次数。

r34717后Controller.has_backup_request()获知是否发送过backup_request。

**重试时框架会尽量避开之前尝试过的server。**

重试的触发条件有(条件之间是AND关系）：
### 连接出错

如果server一直没有返回，但连接没有问题，这种情况下不会重试。如果你需要在一定时间后发送另一个请求，使用backup request。

工作机制如下：如果response没有在backup_request_ms内返回，则发送另外一个请求，哪个先回来就取哪个。新请求会被尽量送到不同的server。注意如果backup_request_ms大于超时，则backup request总不会被发送。backup request会消耗一次重试次数。backup request不意味着server端cancel。

ChannelOptions.backup_request_ms影响该Channel上所有RPC，单位毫秒，默认值-1（表示不开启），Controller.set_backup_request_ms()可修改某次RPC的值。

### 没到超时

超时后RPC会尽快结束。

### 有剩余重试次数

Controller.set_max_retry(0)或ChannelOptions.max_retry=0关闭重试。

### 错误值得重试

一些错误重试是没有意义的，就不会重试，比如请求有错时(EREQUEST)不会重试，因为server总不会接受,没有意义。

用户可以通过继承[brpc::RetryPolicy](https://github.com/brpc/brpc/blob/master/src/brpc/retry_policy.h)自定义重试条件。比如brpc默认不重试http/h2相关的错误，而你的程序中希望在碰到HTTP_STATUS_FORBIDDEN (403)时重试，可以这么做：

```c++
#include <brpc/retry_policy.h>
 
class MyRetryPolicy : public brpc::RetryPolicy {
public:
    bool DoRetry(const brpc::Controller* cntl) const {
        if (cntl->ErrorCode() == brpc::EHTTP && // http/h2错误
            cntl->http_response().status_code() == brpc::HTTP_STATUS_FORBIDDEN) {
            return true;
        }
        // 把其他情况丢给框架。
        return brpc::DefaultRetryPolicy()->DoRetry(cntl);
    }
};
...
 
// 给ChannelOptions.retry_policy赋值就行了。
// 注意：retry_policy必须在Channel使用期间保持有效，Channel也不会删除retry_policy，所以大部分情况下RetryPolicy都应以单例模式创建。
brpc::ChannelOptions options;
static MyRetryPolicy g_my_retry_policy;
options.retry_policy = &g_my_retry_policy;
...
```

一些提示：

* 通过cntl->response()可获得对应RPC的response。
* 对ERPCTIMEDOUT代表的RPC超时总是不重试，即使你继承的RetryPolicy中允许。

### 重试应当保守

由于成本的限制，大部分线上server的冗余度是有限的，主要是满足多机房互备的需求。而激进的重试逻辑很容易导致众多client对server集群造成2-3倍的压力，最终使集群雪崩：由于server来不及处理导致队列越积越长，使所有的请求得经过很长的排队才被处理而最终超时，相当于服务停摆。默认的重试是比较安全的: 只要连接不断RPC就不会重试，一般不会产生大量的重试请求。用户可以通过RetryPolicy定制重试策略，但也可能使重试变成一场“风暴”。当你定制RetryPolicy时，你需要仔细考虑client和server的协作关系，并设计对应的异常测试，以确保行为符合预期。

## 协议

Channel的默认协议是baidu_std，可通过设置ChannelOptions.protocol换为其他协议，这个字段既接受enum也接受字符串。

目前支持的有：

- PROTOCOL_BAIDU_STD 或 “baidu_std"，即[百度标准协议](baidu_std.md)，默认为单连接。
- PROTOCOL_HTTP 或 ”http", http/1.0或http/1.1协议，默认为连接池(Keep-Alive)。
  - 访问普通http服务的方法见[访问http/h2服务](http_client.md)
  - 通过http:json或http:proto访问pb服务的方法见[http/h2衍生协议](http_derivatives.md)
- PROTOCOL_H2 或 ”h2", http/2.0协议，默认是单连接。
  - 访问普通h2服务的方法见[访问http/h2服务](http_client.md)。
  - 通过h2:json或h2:proto访问pb服务的方法见[http/h2衍生协议](http_derivatives.md)
- "h2:grpc", [gRPC](https://grpc.io)的协议，也是h2的衍生协议，默认为单连接，具体见[h2:grpc](http_derivatives.md#h2grpc)。
- PROTOCOL_THRIFT 或 "thrift"，[apache thrift](https://thrift.apache.org)的协议，默认为连接池, 具体方法见[访问thrift](thrift.md)。
- PROTOCOL_MEMCACHE 或 "memcache"，memcached的二进制协议，默认为单连接。具体方法见[访问memcached](memcache_client.md)。
- PROTOCOL_REDIS 或 "redis"，redis 1.2后的协议(也是hiredis支持的协议)，默认为单连接。具体方法见[访问Redis](redis_client.md)。
- PROTOCOL_HULU_PBRPC 或 "hulu_pbrpc"，hulu的协议，默认为单连接。
- PROTOCOL_NOVA_PBRPC 或 ”nova_pbrpc“，网盟的协议，默认为连接池。
- PROTOCOL_SOFA_PBRPC 或 "sofa_pbrpc"，sofa-pbrpc的协议，默认为单连接。
- PROTOCOL_PUBLIC_PBRPC 或 "public_pbrpc"，public_pbrpc的协议，默认为连接池。
- PROTOCOL_UBRPC_COMPACK 或 "ubrpc_compack"，public/ubrpc的协议，使用compack打包，默认为连接池。具体方法见[ubrpc (by protobuf)](ub_client.md)。相关的还有PROTOCOL_UBRPC_MCPACK2或ubrpc_mcpack2，使用mcpack2打包。
- PROTOCOL_NSHEAD_CLIENT 或 "nshead_client"，这是发送baidu-rpc-ub中所有UBXXXRequest需要的协议，默认为连接池。具体方法见[访问UB](ub_client.md)。
- PROTOCOL_NSHEAD 或 "nshead"，这是发送NsheadMessage需要的协议，默认为连接池。具体方法见[nshead+blob](ub_client.md#nshead-blob) 。
- PROTOCOL_NSHEAD_MCPACK 或 "nshead_mcpack", 顾名思义，格式为nshead + mcpack，使用mcpack2pb适配，默认为连接池。
- PROTOCOL_ESP 或 "esp"，访问使用esp协议的服务，默认为连接池。

## 连接方式

brpc支持以下连接方式：

- 短连接：每次RPC前建立连接，结束后关闭连接。由于每次调用得有建立连接的开销，这种方式一般用于偶尔发起的操作，而不是持续发起请求的场景。没有协议默认使用这种连接方式，http 1.0对连接的处理效果类似短链接。
- 连接池：每次RPC前取用空闲连接，结束后归还，一个连接上最多只有一个请求，一个client对一台server可能有多条连接。http 1.1和各类使用nshead的协议都是这个方式。
- 单连接：进程内所有client与一台server最多只有一个连接，一个连接上可能同时有多个请求，回复返回顺序和请求顺序不需要一致，这是baidu_std，hulu_pbrpc，sofa_pbrpc协议的默认选项。

|                     | 短连接                                      | 连接池                   | 单连接                 |
| ------------------- | ---------------------------------------- | --------------------- | ------------------- |
| 长连接                 | 否                                        | 是                     | 是                   |
| server端连接数(单client) | qps*latency (原理见[little's law](https://en.wikipedia.org/wiki/Little%27s_law)) | qps*latency           | 1                   |
| 极限qps               | 差，且受限于单机端口数                              | 中等                    | 高                   |
| latency             | 1.5RTT(connect) + 1RTT + 处理时间            | 1RTT + 处理时间           | 1RTT + 处理时间         |
| cpu占用               | 高, 每次都要tcp connect                       | 中等, 每个请求都要一次sys write | 低, 合并写出在大流量时减少cpu占用 |

框架会为协议选择默认的连接方式，用户**一般不用修改**。若需要，把ChannelOptions.connection_type设为：

- CONNECTION_TYPE_SINGLE 或 "single" 为单连接

- CONNECTION_TYPE_POOLED 或 "pooled" 为连接池, 单个远端对应的连接池最多能容纳的连接数由-max_connection_pool_size控制。注意,此选项不等价于“最大连接数”。需要连接时只要没有闲置的，就会新建；归还时，若池中已有max_connection_pool_size个连接的话，会直接关闭。max_connection_pool_size的取值要符合并发，否则超出的部分会被频繁建立和关闭，效果类似短连接。若max_connection_pool_size为0，就近似于完全的短连接。

  | Name                         | Value | Description                              | Defined At          |
  | ---------------------------- | ----- | ---------------------------------------- | ------------------- |
  | max_connection_pool_size (R) | 100   | Max number of pooled connections to a single endpoint | src/brpc/socket.cpp |

- CONNECTION_TYPE_SHORT 或 "short" 为短连接

- 设置为“”（空字符串）则让框架选择协议对应的默认连接方式。

brpc支持[Streaming RPC](streaming_rpc.md)，这是一种应用层的连接，用于传递流式数据。

## 关闭连接池中的闲置连接

当连接池中的某个连接在-idle_timeout_second时间内没有读写，则被视作“闲置”，会被自动关闭。默认值为10秒。此功能只对连接池(pooled)有效。打开-log_idle_connection_close在关闭前会打印一条日志。

| Name                      | Value | Description                              | Defined At              |
| ------------------------- | ----- | ---------------------------------------- | ----------------------- |
| idle_timeout_second       | 10    | Pooled connections without data transmission for so many seconds will be closed. No effect for non-positive values | src/brpc/socket_map.cpp |
| log_idle_connection_close | false | Print log when an idle connection is closed | src/brpc/socket.cpp     |

## 延迟关闭连接

多个channel可能通过引用计数引用同一个连接，当引用某个连接的最后一个channel析构时，该连接将被关闭。但在一些场景中，channel在使用前才被创建，用完立刻析构，这时其中一些连接就会被无谓地关闭再被打开，效果类似短连接。

一个解决办法是用户把所有或常用的channel缓存下来，这样自然能避免channel频繁产生和析构，但目前brpc没有提供这样一个utility，用户自己（正确）实现有一些工作量。

另一个解决办法是设置全局选项-defer_close_second

| Name               | Value | Description                              | Defined At              |
| ------------------ | ----- | ---------------------------------------- | ----------------------- |
| defer_close_second | 0     | Defer close of connections for so many seconds even if the connection is not used by anyone. Close immediately for non-positive values | src/brpc/socket_map.cpp |

设置后引用计数清0时连接并不会立刻被关闭，而是会等待这么多秒再关闭，如果在这段时间内又有channel引用了这个连接，它会恢复正常被使用的状态。不管channel创建析构有多频率，这个选项使得关闭连接的频率有上限。这个选项的副作用是一些fd不会被及时关闭，如果延时被误设为一个大数值，程序占据的fd个数可能会很大。

## 连接的缓冲区大小

-socket_recv_buffer_size设置所有连接的接收缓冲区大小，默认-1（不修改）

-socket_send_buffer_size设置所有连接的发送缓冲区大小，默认-1（不修改）

| Name                    | Value | Description                              | Defined At          |
| ----------------------- | ----- | ---------------------------------------- | ------------------- |
| socket_recv_buffer_size | -1    | Set the recv buffer size of socket if this value is positive | src/brpc/socket.cpp |
| socket_send_buffer_size | -1    | Set send buffer size of sockets if this value is positive | src/brpc/socket.cpp |

## log_id

通过set_log_id()可设置64位整型log_id。这个id会和请求一起被送到服务器端，一般会被打在日志里，从而把一次检索经过的所有服务串联起来。字符串格式的需要转化为64位整形才能设入log_id。

## 附件

baidu_std和hulu_pbrpc协议支持附件，这段数据由用户自定义，不经过protobuf的序列化。站在client的角度，设置在Controller::request_attachment()的附件会被server端收到，response_attachment()则包含了server端送回的附件。附件不受压缩选项影响。

在http/h2协议中，附件对应[message body](http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html)，比如要POST的数据就设置在request_attachment()中。

## 开启SSL

要开启SSL，首先确保代码依赖了最新的openssl库。如果openssl版本很旧，会有严重的安全漏洞，支持的加密算法也少，违背了开启SSL的初衷。
然后设置`ChannelOptions.mutable_ssl_options()`，具体选项见[ssl_options.h](https://github.com/brpc/brpc/blob/master/src/brpc/ssl_options.h)。ChannelOptions.has_ssl_options()可查询是否设置过ssl_options, ChannelOptions.ssl_options()可访问到设置过的只读ssl_options。

```c++
// 开启客户端SSL并使用默认值。
options.mutable_ssl_options();

// 开启客户端SSL并定制选项。
options.mutable_ssl_options()->ciphers_name = "...";
options.mutable_ssl_options()->sni_name = "...";
```
- 连接单点和集群的Channel均可以开启SSL访问（初始实现曾不支持集群）。
- 开启后，该Channel上任何协议的请求，都会被SSL加密后发送。如果希望某些请求不加密，需要额外再创建一个Channel。
- 针对HTTPS做了些易用性优化：Channel.Init能自动识别https://前缀并自动开启SSL；开启-http_verbose也会输出证书信息。

## 认证

client端的认证一般分为2种：

1. 基于请求的认证：每次请求都会带上认证信息。这种方式比较灵活，认证信息中可以含有本次请求中的字段，但是缺点是每次请求都会需要认证，性能上有所损失
2. 基于连接的认证：当TCP连接建立后，client发送认证包，认证成功后，后续该连接上的请求不再需要认证。相比前者，这种方式灵活度不高（一般认证包里只能携带本机一些静态信息），但性能较好，一般用于单连接/连接池场景

针对第一种认证场景，在实现上非常简单，将认证的格式定义加到请求结构体中，每次当做正常RPC发送出去即可；针对第二种场景，brpc提供了一种机制，只要用户继承实现：

```c++
class Authenticator {
public:
    virtual ~Authenticator() {}

    // Implement this method to generate credential information
    // into `auth_str' which will be sent to `VerifyCredential'
    // at server side. This method will be called on client side.
    // Returns 0 on success, error code otherwise
    virtual int GenerateCredential(std::string* auth_str) const = 0;
};
```

那么当用户并发调用RPC接口用单连接往同一个server发请求时，框架会自动保证：建立TCP连接后，连接上的第一个请求中会带有上述`GenerateCredential`产生的认证包，其余剩下的并发请求不会带有认证信息，依次排在第一个请求之后。整个发送过程依旧是并发的，并不会等第一个请求先返回。若server端认证成功，那么所有请求都能成功返回；若认证失败，一般server端则会关闭连接，这些请求则会收到相应错误。

目前自带协议中支持客户端认证的有：[baidu_std](baidu_std.md)(默认协议), HTTP, hulu_pbrpc, ESP。对于自定义协议，一般可以在组装请求阶段，调用Authenticator接口生成认证串，来支持客户端认证。

## 重置

调用Reset方法可让Controller回到刚创建时的状态。

别在RPC结束前重置Controller，行为是未定义的。

## 压缩

set_request_compress_type()设置request的压缩方式，默认不压缩。

注意：附件不会被压缩。

http/h2 body的压缩方法见[client压缩request body](http_client#压缩request-body)。

支持的压缩方法有：

- brpc::CompressTypeSnappy : [snanpy压缩](http://google.github.io/snappy/)，压缩和解压显著快于其他压缩方法，但压缩率最低。
- brpc::CompressTypeGzip : [gzip压缩](http://en.wikipedia.org/wiki/Gzip)，显著慢于snappy，但压缩率高
- brpc::CompressTypeZlib : [zlib压缩](http://en.wikipedia.org/wiki/Zlib)，比gzip快10%~20%，压缩率略好于gzip，但速度仍明显慢于snappy。

下表是多种压缩算法应对重复率很高的数据时的性能，仅供参考。

| Compress method | Compress size(B) | Compress time(us) | Decompress time(us) | Compress throughput(MB/s) | Decompress throughput(MB/s) | Compress ratio |
| --------------- | ---------------- | ----------------- | ------------------- | ------------------------- | --------------------------- | -------------- |
| Snappy          | 128              | 0.753114          | 0.890815            | 162.0875                  | 137.0322                    | 37.50%         |
| Gzip            | 10.85185         | 1.849199          | 11.2488             | 66.01252                  | 47.66%                      |                |
| Zlib            | 10.71955         | 1.66522           | 11.38763            | 73.30581                  | 38.28%                      |                |
| Snappy          | 1024             | 1.404812          | 1.374915            | 695.1555                  | 710.2713                    | 8.79%          |
| Gzip            | 16.97748         | 3.950946          | 57.52106            | 247.1718                  | 6.64%                       |                |
| Zlib            | 15.98913         | 3.06195           | 61.07665            | 318.9348                  | 5.47%                       |                |
| Snappy          | 16384            | 8.822967          | 9.865008            | 1770.946                  | 1583.881                    | 4.96%          |
| Gzip            | 160.8642         | 43.85911          | 97.13162            | 356.2544                  | 0.78%                       |                |
| Zlib            | 147.6828         | 29.06039          | 105.8011            | 537.6734                  | 0.71%                       |                |
| Snappy          | 32768            | 16.16362          | 19.43596            | 1933.354                  | 1607.844                    | 4.82%          |
| Gzip            | 229.7803         | 82.71903          | 135.9995            | 377.7849                  | 0.54%                       |                |
| Zlib            | 240.7464         | 54.44099          | 129.8046            | 574.0161                  | 0.50%                       |                |

下表是多种压缩算法应对重复率很低的数据时的性能，仅供参考。

| Compress method | Compress size(B) | Compress time(us) | Decompress time(us) | Compress throughput(MB/s) | Decompress throughput(MB/s) | Compress ratio |
| --------------- | ---------------- | ----------------- | ------------------- | ------------------------- | --------------------------- | -------------- |
| Snappy          | 128              | 0.866002          | 0.718052            | 140.9584                  | 170.0021                    | 105.47%        |
| Gzip            | 15.89855         | 4.936242          | 7.678077            | 24.7294                   | 116.41%                     |                |
| Zlib            | 15.88757         | 4.793953          | 7.683384            | 25.46339                  | 107.03%                     |                |
| Snappy          | 1024             | 2.087972          | 1.06572             | 467.7087                  | 916.3403                    | 100.78%        |
| Gzip            | 32.54279         | 12.27744          | 30.00857            | 79.5412                   | 79.79%                      |                |
| Zlib            | 31.51397         | 11.2374           | 30.98824            | 86.90288                  | 78.61%                      |                |
| Snappy          | 16384            | 12.598            | 6.306592            | 1240.276                  | 2477.566                    | 100.06%        |
| Gzip            | 537.1803         | 129.7558          | 29.08707            | 120.4185                  | 75.32%                      |                |
| Zlib            | 519.5705         | 115.1463          | 30.07291            | 135.697                   | 75.24%                      |                |
| Snappy          | 32768            | 22.68531          | 12.39793            | 1377.543                  | 2520.582                    | 100.03%        |
| Gzip            | 1403.974         | 258.9239          | 22.25825            | 120.6919                  | 75.25%                      |                |
| Zlib            | 1370.201         | 230.3683          | 22.80687            | 135.6524                  | 75.21%                      |                |

# FAQ

### Q: brpc能用unix domain socket吗

不能。同机TCP socket并不走网络，相比unix domain socket性能只会略微下降。一些不能用TCP socket的特殊场景可能会需要，以后可能会扩展支持。

### Q: Fail to connect to xx.xx.xx.xx:xxxx, Connection refused

一般是对端server没打开端口（很可能挂了）。 

### Q: 经常遇到至另一个机房的Connection timedout

![img](../images/connection_timedout.png)

这个就是连接超时了，调大连接和RPC超时：

```c++
struct ChannelOptions {
    ...
    // Issue error when a connection is not established after so many
    // milliseconds. -1 means wait indefinitely.
    // Default: 200 (milliseconds)
    // Maximum: 0x7fffffff (roughly 30 days)
    int32_t connect_timeout_ms;
    
    // Max duration of RPC over this Channel. -1 means wait indefinitely.
    // Overridable by Controller.set_timeout_ms().
    // Default: 500 (milliseconds)
    // Maximum: 0x7fffffff (roughly 30 days)
    int32_t timeout_ms;
    ...
};
```

注意: 连接超时不是RPC超时，RPC超时打印的日志是"Reached timeout=..."。

### Q: 为什么同步方式是好的，异步就crash了

重点检查Controller，Response和done的生命周期。在异步访问中，RPC调用结束并不意味着RPC整个过程结束，而是在进入done->Run()时才会结束。所以这些对象不应在调用RPC后就释放，而是要在done->Run()里释放。你一般不能把这些对象分配在栈上，而应该分配在堆上。详见[异步访问](client.md#异步访问)。

### Q: 怎么确保请求只被处理一次

这不是RPC层面的事情。当response返回且成功时，我们确认这个过程一定成功了。当response返回且失败时，我们确认这个过程一定失败了。但当response没有返回时，它可能失败，也可能成功。如果我们选择重试，那一个成功的过程也可能会被再执行一次。一般来说带副作用的RPC服务都应当考虑[幂等](http://en.wikipedia.org/wiki/Idempotence)问题，否则重试可能会导致多次叠加副作用而产生意向不到的结果。只有读的检索服务大都没有副作用而天然幂等，无需特殊处理。而带写的存储服务则要在设计时就加入版本号或序列号之类的机制以拒绝已经发生的过程，保证幂等。

### Q: Invalid address=`bns://group.user-persona.dumi.nj03'
```
FATAL 04-07 20:00:03 7778 src/brpc/channel.cpp:123] Invalid address=`bns://group.user-persona.dumi.nj03'. You should use Init(naming_service_name, load_balancer_name, options) to access multiple servers.
```
访问命名服务要使用三个参数的Init，其中第二个参数是load_balancer_name，而这里用的是两个参数的Init，框架认为是访问单点，就会报这个错。

### Q: 两端都用protobuf，为什么不能互相访问

**协议 !=protobuf**。protobuf负责一个包的序列化，协议中的一个消息可能会包含多个protobuf包，以及额外的长度、校验码、magic number等等。打包格式相同不意味着协议可以互通。在brpc中写一份代码就能服务多协议的能力是通过把不同协议的数据转化为统一的编程接口完成的，而不是在protobuf层面。

### Q: 为什么C++ client/server 能够互相通信， 和其他语言的client/server 通信会报序列化失败的错误

检查一下C++ 版本是否开启了压缩 (Controller::set_compress_type), 目前其他语言的rpc框架还没有实现压缩，互相返回会出现问题。 

# 附:Client端基本流程

![img](../images/client_side.png)

主要步骤：

1. 创建一个[bthread_id](https://github.com/brpc/brpc/blob/master/src/bthread/id.h)作为本次RPC的correlation_id。
2. 根据Channel的创建方式，从进程级的[SocketMap](https://github.com/brpc/brpc/blob/master/src/brpc/socket_map.h)中或从[LoadBalancer](https://github.com/brpc/brpc/blob/master/src/brpc/load_balancer.h)中选择一台下游server作为本次RPC发送的目的地。
3. 根据连接方式（单连接、连接池、短连接），选择一个[Socket](https://github.com/brpc/brpc/blob/master/src/brpc/socket.h)。
4. 如果开启验证且当前Socket没有被验证过时，第一个请求进入验证分支，其余请求会阻塞直到第一个包含认证信息的请求写入Socket。server端只对第一个请求进行验证。
5. 根据Channel的协议，选择对应的序列化函数把request序列化至[IOBuf](https://github.com/brpc/brpc/blob/master/src/butil/iobuf.h)。
6. 如果配置了超时，设置定时器。从这个点开始要避免使用Controller对象，因为在设定定时器后随时可能触发超时->调用到用户的超时回调->用户在回调中析构Controller。
7. 发送准备阶段结束，若上述任何步骤出错，会调用Channel::HandleSendFailed。
8. 将之前序列化好的IOBuf写出到Socket上，同时传入回调Channel::HandleSocketFailed，当连接断开、写失败等错误发生时会调用此回调。
9. 如果是同步发送，Join correlation_id；否则至此CallMethod结束。
10. 网络上发消息+收消息。
11. 收到response后，提取出其中的correlation_id，在O(1)时间内找到对应的Controller。这个过程中不需要查找全局哈希表，有良好的多核扩展性。
12. 根据协议格式反序列化response。
13. 调用Controller::OnRPCReturned，可能会根据错误码判断是否需要重试，或让RPC结束。如果是异步发送，调用用户回调。最后摧毁correlation_id唤醒Join着的线程。


# ========= ./combo_channel.md ========
[English version](../en/combo_channel.md)

随着服务规模的增大，对下游的访问流程会越来越复杂，其中往往包含多个同时发起的RPC或有复杂的层次结构。但这类代码的多线程陷阱很多，用户可能写出了bug也不自知，复现和调试也比较困难。而且实现要么只能支持同步的情况，要么要么得为异步重写一套。以"在多个异步RPC完成后运行一些代码"为例，它的同步实现一般是异步地发起多个RPC，然后逐个等待各自完成；它的异步实现一般是用一个带计数器的回调，每当一个RPC完成时计数器减一，直到0时调用回调。可以看到它的缺点：

- 同步和异步代码不一致。用户无法轻易地从一个模式转为另一种模式。从设计的角度，不一致暗示了没有抓住本质。
- 往往不能被取消。正确及时地取消一个操作不是一件易事，何况是组合访问。但取消对于终结无意义的等待是很必要的。
- 不能继续组合。比如你很难把一个上述实现变成“更大"的访问模式的一部分。换个场景还得重写一套。

我们需要更好的抽象。如果我们能以不同的方式把一些Channel组合为更大的Channel，并把不同的访问模式置入其中，那么用户可以便用统一接口完成同步、异步、取消等操作。这种channel在brpc中被称为组合channel。

# ParallelChannel

ParallelChannel (有时被称为“pchan”)同时访问其包含的sub channel，并合并它们的结果。用户可通过CallMapper修改请求，通过ResponseMerger合并结果。ParallelChannel看起来就像是一个Channel：

- 支持同步和异步访问。
- 发起异步操作后可以立刻删除。
- 可以取消。
- 支持超时。

示例代码见[example/parallel_echo_c++](https://github.com/brpc/brpc/tree/master/example/parallel_echo_c++/)。

任何brpc::ChannelBase的子类都可以加入ParallelChannel，包括ParallelChannel和其他组合Channel。用户可以设置ParallelChannelOptions.fail_limit来控制访问的最大失败次数，当失败的访问达到这个数目时，RPC会立刻结束而不等待超时。

一个sub channel可多次加入同一个ParallelChannel。当你需要对同一个服务发起多次异步访问并等待它们完成的话，这很有用。

ParallelChannel的内部结构大致如下：

![img](../images/pchan.png)

## 插入sub channel

可通过如下接口把sub channel插入ParallelChannel：

```c++
int AddChannel(brpc::ChannelBase* sub_channel,
               ChannelOwnership ownership,
               CallMapper* call_mapper,
               ResponseMerger* response_merger);
```

当ownership为brpc::OWNS_CHANNEL时，sub_channel会在ParallelChannel析构时被删除。一个sub channel可能会多次加入一个ParallelChannel，如果其中一个指明了ownership为brpc::OWNS_CHANNEL，那个sub channel会在ParallelChannel析构时被最多删除一次。

访问ParallelChannel时调用AddChannel是**线程不安全**的。

## CallMapper

用于把对ParallelChannel的调用转化为对sub channel的调用。如果call_mapper是NULL，sub channel的请求就是ParallelChannel的请求，而response则New()自ParallelChannel的response。如果call_mapper不为NULL，则会在ParallelChannel析构时被删除。call_mapper内含引用计数，一个call_mapper可与多个sub channel关联。

```c++
class CallMapper {
public:
    virtual ~CallMapper();
 
    virtual SubCall Map(int channel_index/*starting from 0*/,
                        const google::protobuf::MethodDescriptor* method,
                        const google::protobuf::Message* request,
                        google::protobuf::Message* response) = 0;
};
```

channel_index：该sub channel在ParallelChannel中的位置，从0开始计数。

method/request/response：ParallelChannel.CallMethod()的参数。

返回的SubCall被用于访问对应sub channel，SubCall有两个特殊值：

- 返回SubCall::Bad()则对ParallelChannel的该次访问立刻失败，Controller.ErrorCode()为EREQUEST。
- 返回SubCall::Skip()则跳过对该sub channel的访问，如果所有的sub channel都被跳过了，该次访问立刻失败，Controller.ErrorCode()为ECANCELED。

常见的Map()实现有：

- 广播request。这也是call_mapper为NULL时的行为：
```c++
  class Broadcaster : public CallMapper {
  public:
      SubCall Map(int channel_index/*starting from 0*/,
                  const google::protobuf::MethodDescriptor* method,
                  const google::protobuf::Message* request,
                  google::protobuf::Message* response) {
          // method/request和pchan保持一致.
          // response是new出来的，最后的flag告诉pchan在RPC结束后删除Response。
          return SubCall(method, request, response->New(), DELETE_RESPONSE);
      }
  };
```
- 修改request中的字段后再发。
```c++
  class ModifyRequest : public CallMapper {
  public:
    SubCall Map(int channel_index/*starting from 0*/,
                const google::protobuf::MethodDescriptor* method,
                const google::protobuf::Message* request,
                google::protobuf::Message* response) {
        FooRequest* copied_req = brpc::Clone<FooRequest>(request);
        copied_req->set_xxx(...);
        // 拷贝并修改request，最后的flag告诉pchan在RPC结束后删除Request和Response。
        return SubCall(method, copied_req, response->New(), DELETE_REQUEST | DELETE_RESPONSE);
    }
  };
```
- request和response已经包含了sub request/response，直接取出来。
```c++
  class UseFieldAsSubRequest : public CallMapper {
  public:
    SubCall Map(int channel_index/*starting from 0*/,
                const google::protobuf::MethodDescriptor* method,
                const google::protobuf::Message* request,
                google::protobuf::Message* response) {
        if (channel_index >= request->sub_request_size()) {
            // sub_request不够，说明外面准备数据的地方和pchan中sub channel的个数不符.
            // 返回Bad()让该次访问立刻失败
            return SubCall::Bad();
        }
        // 取出对应的sub request，增加一个sub response，最后的flag为0告诉pchan什么都不用删
        return SubCall(sub_method, request->sub_request(channel_index), response->add_sub_response(), 0);
    }
  };
```

## ResponseMerger

response_merger把sub channel的response合并入总的response，其为NULL时，则使用response->MergeFrom(*sub_response)，MergeFrom的行为可概括为“除了合并repeated字段，其余都是覆盖”。如果你需要更复杂的行为，则需实现ResponseMerger。response_merger是一个个执行的，所以你并不需要考虑多个Merge同时运行的情况。response_merger在ParallelChannel析构时被删除。response_merger内含引用计数，一个response_merger可与多个sub channel关联。

Result的取值有：
- MERGED: 成功合并。
- FAIL: sub_response没有合并成功，会被记作一次失败。比如有10个sub channels且fail_limit为4，只要有4个合并结果返回了FAIL，这次RPC就会达到fail_limit并立刻结束。
- FAIL_ALL: 使本次RPC直接结束。


## 获得访问sub channel时的controller

有时访问者需要了解访问sub channel时的细节，通过Controller.sub(i)可获得访问sub channel的controller.

```c++
// Get the controllers for accessing sub channels in combo channels.
// Ordinary channel:
//   sub_count() is 0 and sub() is always NULL.
// ParallelChannel/PartitionChannel:
//   sub_count() is #sub-channels and sub(i) is the controller for
//   accessing i-th sub channel inside ParallelChannel, if i is outside
//    [0, sub_count() - 1], sub(i) is NULL.
//   NOTE: You must test sub() against NULL, ALWAYS. Even if i is inside
//   range, sub(i) can still be NULL:
//   * the rpc call may fail and terminate before accessing the sub channel
//   * the sub channel was skipped
// SelectiveChannel/DynamicPartitionChannel:
//   sub_count() is always 1 and sub(0) is the controller of successful
//   or last call to sub channels.
int sub_count() const;
const Controller* sub(int index) const;
```

# SelectiveChannel

[SelectiveChannel](https://github.com/brpc/brpc/blob/master/src/brpc/selective_channel.h) (有时被称为“schan”)按负载均衡算法访问其包含的Channel，相比普通Channel它更加高层：把流量分给sub channel，而不是具体的Server。SelectiveChannel主要用来支持机器组之间的负载均衡，它具备Channel的主要属性：

- 支持同步和异步访问。
- 发起异步操作后可以立刻删除。
- 可以取消。
- 支持超时。

示例代码见[example/selective_echo_c++](https://github.com/brpc/brpc/tree/master/example/selective_echo_c++/)。

任何brpc::ChannelBase的子类都可加入SelectiveChannel，包括SelectiveChannel和其他组合Channel。

SelectiveChannel的重试独立于其中的sub channel，当SelectiveChannel访问某个sub channel失败后（本身可能重试），它会重试另外一个sub channel。

目前SelectiveChannel要求**request必须在RPC结束前有效**，其他channel没有这个要求。如果你使用SelectiveChannel发起异步操作，确保request在done中才被删除。

## 使用SelectiveChannel

SelectiveChannel的初始化和普通Channel基本一样，但Init不需要指定命名服务，因为SelectiveChannel通过AddChannel动态添加sub channel，而普通Channel通过命名服务动态管理server。

```c++
#include <brpc/selective_channel.h>
...
brpc::SelectiveChannel schan;
brpc::ChannelOptions schan_options;
schan_options.timeout_ms = ...;
schan_options.backup_request_ms = ...;
schan_options.max_retry = ...;
if (schan.Init(load_balancer, &schan_options) != 0) {
    LOG(ERROR) << "Fail to init SelectiveChannel";
    return -1;
}
```

初始化完毕后通过AddChannel加入sub channel。

```c++
if (schan.AddChannel(sub_channel, NULL/*ChannelHandle*/) != 0) {  
    // 第二个参数ChannelHandle用于删除sub channel，不用删除可填NULL
    LOG(ERROR) << "Fail to add sub_channel";
    return -1;
}
```

注意：

- 和ParallelChannel不同，SelectiveChannel的AddChannel可在任意时刻调用，即使该SelectiveChannel正在被访问（下一次访问时生效）
- SelectiveChannel总是own sub channel，这和ParallelChannel可选择ownership是不同的。
- 如果AddChannel第二个参数不为空，会填入一个类型为brpc::SelectiveChannel::ChannelHandle的值，这个handle可作为RemoveAndDestroyChannel的参数来动态删除一个channel。
- SelectiveChannel会用自身的超时覆盖sub channel初始化时指定的超时。比如某个sub channel的超时为100ms，SelectiveChannel的超时为500ms，实际访问时的超时是500ms。

访问SelectiveChannel的方式和普通Channel是一样的。

## 例子: 往多个命名服务分流

一些场景中我们需要向多个命名服务下的机器分流，原因可能有：

- 完成同一个检索功能的机器被挂载到了不同的命名服务下。
- 机器被拆成了多个组，流量先分流给一个组，再分流到组内机器。组间的分流方式和组内有所不同。

这都可以通过SelectiveChannel完成。

下面的代码创建了一个SelectiveChannel，并插入三个访问不同bns的普通Channel。

```c++
brpc::SelectiveChannel channel;
brpc::ChannelOptions schan_options;
schan_options.timeout_ms = FLAGS_timeout_ms;
schan_options.max_retry = FLAGS_max_retry;
if (channel.Init("c_murmurhash", &schan_options) != 0) {
    LOG(ERROR) << "Fail to init SelectiveChannel";
    return -1;
}
 
for (int i = 0; i < 3; ++i) {
    brpc::Channel* sub_channel = new brpc::Channel;
    if (sub_channel->Init(ns_node_name[i], "rr", NULL) != 0) {
        LOG(ERROR) << "Fail to init sub channel " << i;
        return -1;
    }
    if (channel.AddChannel(sub_channel, NULL/*handle for removal*/) != 0) {
        LOG(ERROR) << "Fail to add sub_channel to channel";
        return -1;
    } 
}
...
XXXService_Stub stub(&channel);
stub.FooMethod(&cntl, &request, &response, NULL);
...
```

# PartitionChannel

[PartitionChannel](https://github.com/brpc/brpc/blob/master/src/brpc/partition_channel.h)是特殊的ParallelChannel，它会根据命名服务中的tag自动建立对应分库的sub channel。这样用户就可以把所有的分库机器挂在一个命名服务内，通过tag来指定哪台机器对应哪个分库。示例代码见[example/partition_echo_c++](https://github.com/brpc/brpc/tree/master/example/partition_echo_c++/)。

ParititonChannel只能处理一种分库方法，当用户需要多种分库方法共存，或从一个分库方法平滑地切换为另一种分库方法时，可以使用DynamicPartitionChannel，它会根据不同的分库方式动态地建立对应的sub PartitionChannel，并根据容量把请求分配给不同的分库。示例代码见[example/dynamic_partition_echo_c++](https://github.com/brpc/brpc/tree/master/example/dynamic_partition_echo_c++/)。

如果分库在不同的命名服务内，那么用户得自行用ParallelChannel组装，即每个sub channel对应一个分库（使用不同的命名服务）。ParellelChannel的使用方法见[上面](#ParallelChannel)。

## 使用PartitionChannel

首先定制PartitionParser。这个例子中tag的形式是N/M，N代表分库的index，M是分库的个数。比如0/3代表一共3个分库，这是第一个。

```c++
#include <brpc/partition_channel.h>
...
class MyPartitionParser : public brpc::PartitionParser {
public:
    bool ParseFromTag(const std::string& tag, brpc::Partition* out) {
        // "N/M" : #N partition of M partitions.
        size_t pos = tag.find_first_of('/');
        if (pos == std::string::npos) {
            LOG(ERROR) << "Invalid tag=" << tag;
            return false;
        }
        char* endptr = NULL;
        out->index = strtol(tag.c_str(), &endptr, 10);
        if (endptr != tag.data() + pos) {
            LOG(ERROR) << "Invalid index=" << butil::StringPiece(tag.data(), pos);
            return false;
        }
        out->num_partition_kinds = strtol(tag.c_str() + pos + 1, &endptr, 10);
        if (endptr != tag.c_str() + tag.size()) {
            LOG(ERROR) << "Invalid num=" << tag.data() + pos + 1;
            return false;
        }
        return true;
    }
};
```

然后初始化PartitionChannel。

```c++
#include <brpc/partition_channel.h>
...
brpc::PartitionChannel channel;
 
brpc::PartitionChannelOptions options;
options.protocol = ...;   // PartitionChannelOptions继承了ChannelOptions，后者有的前者也有
options.timeout_ms = ...; // 同上
options.fail_limit = 1;   // PartitionChannel自己的选项，意思同ParalellChannel中的fail_limit。
//这里为1的意思是只要有1个分库访问失败，这次RPC就失败了。
 
if (channel.Init(num_partition_kinds, new MyPartitionParser(),
                 server_address, load_balancer, &options) != 0) {
    LOG(ERROR) << "Fail to init PartitionChannel";
    return -1;
}
// 访问方法和普通Channel是一样的
```

## 使用DynamicPartitionChannel

DynamicPartitionChannel的使用方法和PartitionChannel基本上是一样的，先定制PartitionParser再初始化，但Init时不需要num_partition_kinds，因为DynamicPartitionChannel会为不同的分库方法动态建立不同的sub PartitionChannel。

下面演示一下使用DynamicPartitionChannel平滑地从3库变成4库。

首先分别在8004, 8005, 8006端口启动三个server。

```
$ ./echo_server -server_num 3
TRACE: 09-06 10:40:39:   * 0 server.cpp:159] EchoServer is serving on port=8004
TRACE: 09-06 10:40:39:   * 0 server.cpp:159] EchoServer is serving on port=8005
TRACE: 09-06 10:40:39:   * 0 server.cpp:159] EchoServer is serving on port=8006
TRACE: 09-06 10:40:40:   * 0 server.cpp:192] S[0]=0 S[1]=0 S[2]=0 [total=0]
TRACE: 09-06 10:40:41:   * 0 server.cpp:192] S[0]=0 S[1]=0 S[2]=0 [total=0]
TRACE: 09-06 10:40:42:   * 0 server.cpp:192] S[0]=0 S[1]=0 S[2]=0 [total=0]
```

启动后每个Server每秒会打印上一秒收到的流量，目前都是0。

在本地启动使用DynamicPartitionChannel的Client，初始化代码如下：

```c++
    ...
    brpc::DynamicPartitionChannel channel;
    brpc::PartitionChannelOptions options;
    // 访问任何分库失败都认为RPC失败。调大这个数值可以使访问更宽松，比如等于2的话表示至少两个分库失败才算失败。
    options.fail_limit = 1;
    if (channel.Init(new MyPartitionParser(), "file://server_list", "rr", &options) != 0) {
        LOG(ERROR) << "Fail to init channel";
        return -1;
    }
    ...
```

命名服务"file://server_list"的内容是：
```
0.0.0.0:8004  0/3  # 表示3分库中的第一个分库，其他依次类推
0.0.0.0:8004  1/3
0.0.0.0:8004  2/3
```

3分库方案的3个库都在8004端口对应的server上。启动Client后Client发现了8004，并向其发送流量。

```
$ ./echo_client            
TRACE: 09-06 10:51:10:   * 0 src/brpc/policy/file_naming_service.cpp:83] Got 3 unique addresses from `server_list'
TRACE: 09-06 10:51:10:   * 0 src/brpc/socket.cpp:779] Connected to 0.0.0.0:8004 via fd=3 SocketId=0 self_port=46544
TRACE: 09-06 10:51:11:   * 0 client.cpp:226] Sending EchoRequest at qps=132472 latency=371
TRACE: 09-06 10:51:12:   * 0 client.cpp:226] Sending EchoRequest at qps=132658 latency=370
TRACE: 09-06 10:51:13:   * 0 client.cpp:226] Sending EchoRequest at qps=133208 latency=369
```

同时Server端收到了3倍的流量：因为访问一次Client端要访问三次8004，分别对应每个分库。

```
TRACE: 09-06 10:51:11:   * 0 server.cpp:192] S[0]=398866 S[1]=0 S[2]=0 [total=398866]
TRACE: 09-06 10:51:12:   * 0 server.cpp:192] S[0]=398117 S[1]=0 S[2]=0 [total=398117]
TRACE: 09-06 10:51:13:   * 0 server.cpp:192] S[0]=398873 S[1]=0 S[2]=0 [total=398873]
```

开始修改分库，在server_list中加入4分库的8005：

```
0.0.0.0:8004  0/3
0.0.0.0:8004  1/3   
0.0.0.0:8004  2/3 
 
0.0.0.0:8005  0/4        
0.0.0.0:8005  1/4        
0.0.0.0:8005  2/4        
0.0.0.0:8005  3/4
```

观察Client和Server的输出变化。Client端发现了server_list的变化并重新载入，但qps并没有什么变化。

```
TRACE: 09-06 10:57:10:   * 0 src/brpc/policy/file_naming_service.cpp:83] Got 7 unique addresses from `server_list'
TRACE: 09-06 10:57:10:   * 0 src/brpc/socket.cpp:779] Connected to 0.0.0.0:8005 via fd=7 SocketId=768 self_port=39171
TRACE: 09-06 10:57:11:   * 0 client.cpp:226] Sending EchoRequest at qps=135346 latency=363
TRACE: 09-06 10:57:12:   * 0 client.cpp:226] Sending EchoRequest at qps=134201 latency=366
TRACE: 09-06 10:57:13:   * 0 client.cpp:226] Sending EchoRequest at qps=137627 latency=356
TRACE: 09-06 10:57:14:   * 0 client.cpp:226] Sending EchoRequest at qps=136775 latency=359
TRACE: 09-06 10:57:15:   * 0 client.cpp:226] Sending EchoRequest at qps=139043 latency=353
```

server端的变化比较大。8005收到了流量，并且和8004的流量比例关系约为4:3。

```
TRACE: 09-06 10:57:09:   * 0 server.cpp:192] S[0]=398597 S[1]=0 S[2]=0 [total=398597]
TRACE: 09-06 10:57:10:   * 0 server.cpp:192] S[0]=392839 S[1]=0 S[2]=0 [total=392839]
TRACE: 09-06 10:57:11:   * 0 server.cpp:192] S[0]=334704 S[1]=83219 S[2]=0 [total=417923]
TRACE: 09-06 10:57:12:   * 0 server.cpp:192] S[0]=206215 S[1]=273873 S[2]=0 [total=480088]
TRACE: 09-06 10:57:13:   * 0 server.cpp:192] S[0]=204520 S[1]=270483 S[2]=0 [total=475003]
TRACE: 09-06 10:57:14:   * 0 server.cpp:192] S[0]=207055 S[1]=273725 S[2]=0 [total=480780]
TRACE: 09-06 10:57:15:   * 0 server.cpp:192] S[0]=208453 S[1]=276803 S[2]=0 [total=485256]
```

一次RPC要访问三次8004或四次8005，8004和8005流量比是3:4，说明Client以1:1的比例访问了3分库和4分库。这个比例关系取决于其容量。容量的计算是递归的：

- 普通Channel的容量等于它其中所有server的容量之和。如果命名服务没有配置权值，单个server的容量为1。
- ParallelChannel或PartitionChannel的容量等于它其中Sub Channel容量的最小值。
- SelectiveChannel的容量等于它其中Sub Channel的容量之和。
- DynamicPartitionChannel的容量等于它其中Sub PartitionChannel的容量之和。

在这儿的场景中，3分库和4分库的容量都是1，因为所有的3库都在8004一台server上，所有的4库都在8005一台server上。

在4分库方案加入加入8006端口的server:

```
0.0.0.0:8004  0/3
0.0.0.0:8004  1/3   
0.0.0.0:8004  2/3
 
0.0.0.0:8005  0/4   
0.0.0.0:8005  1/4   
0.0.0.0:8005  2/4   
0.0.0.0:8005  3/4    
 
0.0.0.0:8006 0/4
0.0.0.0:8006 1/4
0.0.0.0:8006 2/4
0.0.0.0:8006 3/4
```

Client的变化仍旧不大：

```
TRACE: 09-06 11:11:51:   * 0 src/brpc/policy/file_naming_service.cpp:83] Got 11 unique addresses from `server_list'
TRACE: 09-06 11:11:51:   * 0 src/brpc/socket.cpp:779] Connected to 0.0.0.0:8006 via fd=8 SocketId=1280 self_port=40759
TRACE: 09-06 11:11:51:   * 0 client.cpp:226] Sending EchoRequest at qps=131799 latency=372
TRACE: 09-06 11:11:52:   * 0 client.cpp:226] Sending EchoRequest at qps=136217 latency=361
TRACE: 09-06 11:11:53:   * 0 client.cpp:226] Sending EchoRequest at qps=133531 latency=368
TRACE: 09-06 11:11:54:   * 0 client.cpp:226] Sending EchoRequest at qps=136072 latency=361
```

Server端可以看到8006收到了流量。三台server的流量比例约为3:4:4。这是因为3分库的容量仍为1，而4分库由于8006的加入变成了2。3分库和4分库的流量比例是3:8。4分库中的每个分库在8005和8006上都有实例，同一个分库的不同实例使用round robin分流，所以8005和8006平摊了流量。最后的效果就是3:4:4。

```
TRACE: 09-06 11:11:51:   * 0 server.cpp:192] S[0]=199625 S[1]=263226 S[2]=0 [total=462851]
TRACE: 09-06 11:11:52:   * 0 server.cpp:192] S[0]=143248 S[1]=190717 S[2]=159756 [total=493721]
TRACE: 09-06 11:11:53:   * 0 server.cpp:192] S[0]=133003 S[1]=178328 S[2]=178325 [total=489656]
TRACE: 09-06 11:11:54:   * 0 server.cpp:192] S[0]=135534 S[1]=180386 S[2]=180333 [total=496253]
```

尝试去掉3分库中的一个分库: (你可以在file://server_list中使用#注释一行)

```
 0.0.0.0:8004  0/3
 0.0.0.0:8004  1/3
#0.0.0.0:8004  2/3
 
 0.0.0.0:8005  0/4   
 0.0.0.0:8005  1/4   
 0.0.0.0:8005  2/4   
 0.0.0.0:8005  3/4    
 
 0.0.0.0:8006 0/4
 0.0.0.0:8006 1/4
 0.0.0.0:8006 2/4
 0.0.0.0:8006 3/4
```

Client端发现了这点。

```
TRACE: 09-06 11:17:47:   * 0 src/brpc/policy/file_naming_service.cpp:83] Got 10 unique addresses from `server_list'
TRACE: 09-06 11:17:47:   * 0 client.cpp:226] Sending EchoRequest at qps=131653 latency=373
TRACE: 09-06 11:17:48:   * 0 client.cpp:226] Sending EchoRequest at qps=120560 latency=407
TRACE: 09-06 11:17:49:   * 0 client.cpp:226] Sending EchoRequest at qps=124100 latency=395
TRACE: 09-06 11:17:50:   * 0 client.cpp:226] Sending EchoRequest at qps=123743 latency=397
```

Server端更明显，8004很快没有了流量。这是因为去掉的分库已经是3分库中最后的2/3分库，去掉后3分库的容量变为了0，导致8004分不到任何流量了。

```
TRACE: 09-06 11:17:47:   * 0 server.cpp:192] S[0]=130864 S[1]=174499 S[2]=174548 [total=479911]
TRACE: 09-06 11:17:48:   * 0 server.cpp:192] S[0]=20063 S[1]=230027 S[2]=230098 [total=480188]
TRACE: 09-06 11:17:49:   * 0 server.cpp:192] S[0]=0 S[1]=245961 S[2]=245888 [total=491849]
TRACE: 09-06 11:17:50:   * 0 server.cpp:192] S[0]=0 S[1]=250198 S[2]=250150 [total=500348]
```

在真实的线上环境中，我们会逐渐地增加4分库的server，同时下掉3分库中的server。DynamicParititonChannel会按照每种分库方式的容量动态切分流量。当某个时刻3分库的容量变为0时，我们便平滑地把Server从3分库变为了4分库，同时并没有修改Client的代码。


# ========= ./connections.md ========
[connections服务](http://brpc.baidu.com:8765/connections)可以查看所有的连接。一个典型的页面如下：

server_socket_count: 5

| CreatedTime                | RemoteSide          | SSL  | Protocol  | fd   | BytesIn/s | In/s | BytesOut/s | Out/s | BytesIn/m | In/m | BytesOut/m | Out/m | SocketId |
| -------------------------- | ------------------- | ---- | --------- | ---- | --------- | ---- | ---------- | ----- | --------- | ---- | ---------- | ----- | -------- |
| 2015/09/21-21:32:09.630840 | 172.22.38.217:51379 | No   | http      | 19   | 1300      | 1    | 269        | 1     | 68844     | 53   | 115860     | 53    | 257      |
| 2015/09/21-21:32:09.630857 | 172.22.38.217:51380 | No   | http      | 20   | 1308      | 1    | 5766       | 1     | 68884     | 53   | 129978     | 53    | 258      |
| 2015/09/21-21:32:09.630880 | 172.22.38.217:51381 | No   | http      | 21   | 1292      | 1    | 1447       | 1     | 67672     | 52   | 143414     | 52    | 259      |
| 2015/09/21-21:32:01.324587 | 127.0.0.1:55385     | No   | baidu_std | 15   | 1480      | 20   | 880        | 20    | 88020     | 1192 | 52260      | 1192  | 512      |
| 2015/09/21-21:32:01.325969 | 127.0.0.1:55387     | No   | baidu_std | 17   | 4016      | 40   | 1554       | 40    | 238879    | 2384 | 92660      | 2384  | 1024     |

channel_socket_count: 1

| CreatedTime                | RemoteSide     | SSL  | Protocol  | fd   | BytesIn/s | In/s | BytesOut/s | Out/s | BytesIn/m | In/m | BytesOut/m | Out/m | SocketId |
| -------------------------- | -------------- | ---- | --------- | ---- | --------- | ---- | ---------- | ----- | --------- | ---- | ---------- | ----- | -------- |
| 2015/09/21-21:32:01.325870 | 127.0.0.1:8765 | No   | baidu_std | 16   | 1554      | 40   | 4016       | 40    | 92660     | 2384 | 238879     | 2384  | 0        |

channel_short_socket_count: 0

上述信息分为三段：

- 第一段是server接受(accept)的连接。
- 第二段是server与下游的单连接（使用brpc::Channel建立），fd为-1的是虚拟连接，对应第三段中所有相同RemoteSide的连接。
- 第三段是server与下游的短连接或连接池(pooled connections)，这些连接从属于第二段中的相同RemoteSide的虚拟连接。

表格标题的含义：

- RemoteSide : 远端的ip和端口。
- SSL：是否使用SSL加密，若为Yes的话，一般是HTTPS连接。
- Protocol : 使用的协议，可能为baidu_std hulu_pbrpc sofa_pbrpc memcache http public_pbrpc nova_pbrpc nshead_server等。
- fd : file descriptor（文件描述符），可能为-1。
- BytesIn/s : 上一秒读入的字节数
- In/s : 上一秒读入的消息数（消息是对request和response的统称）
- BytesOut/s : 上一秒写出的字节数
- Out/s : 上一秒写出的消息数
- BytesIn/m: 上一分钟读入的字节数
- In/m: 上一分钟读入的消息数
- BytesOut/m: 上一分钟写出的字节数
- Out/m: 上一分钟写出的消息数
- SocketId ：内部id，用于debug，用户不用关心。



典型截图分别如下所示：

单连接：![img](../images/single_conn.png)

连接池：![img](../images/pooled_conn.png)

短连接：![img](../images/short_conn.png)



# ========= ./consistent_hashing.md ========
# 概述

一些场景希望同样的请求尽量落到一台机器上，比如访问缓存集群时，我们往往希望同一种请求能落到同一个后端上，以充分利用其上已有的缓存，不同的机器承载不同的稳定working set。而不是随机地散落到所有机器上，那样的话会迫使所有机器缓存所有的内容，最终由于存不下形成颠簸而表现糟糕。 我们都知道hash能满足这个要求，比如当有n台服务器时，输入x总是会发送到第hash(x) % n台服务器上。但当服务器变为m台时，hash(x) % n和hash(x) % m很可能都不相等，这会使得几乎所有请求的发送目的地都发生变化，如果目的地是缓存服务，所有缓存将失效，继而对原本被缓存遮挡的数据库或计算服务造成请求风暴，触发雪崩。一致性哈希是一种特殊的哈希算法，在增加服务器时，发向每个老节点的请求中只会有一部分转向新节点，从而实现平滑的迁移。[这篇论文](http://blog.phpdr.net/wp-content/uploads/2012/08/Consistent-Hashing-and-Random-Trees.pdf)中提出了一致性hash的概念。

一致性hash满足以下四个性质：

- 平衡性 (Balance) : 每个节点被选到的概率是O(1/n)。
- 单调性 (Monotonicity) : 当新节点加入时， 不会有请求在老节点间移动， 只会从老节点移动到新节点。当有节点被删除时，也不会影响落在别的节点上的请求。
- 分散性 (Spread) : 当上游的机器看到不同的下游列表时(在上线时及不稳定的网络中比较常见),  同一个请求尽量映射到少量的节点中。
- 负载 (Load) : 当上游的机器看到不同的下游列表的时候， 保证每台下游分到的请求数量尽量一致。



# 实现方式

所有server的32位hash值在32位整数值域上构成一个环(Hash Ring)，环上的每个区间和一个server唯一对应，如果一个key落在某个区间内， 它就被分流到对应的server上。 

![img](../images/chash.png)

当删除一个server的， 它对应的区间会归属于相邻的server，所有的请求都会跑过去。当增加一个server时，它会分割某个server的区间并承载落在这个区间上的所有请求。单纯使用Hash Ring很难满足我们上节提到的属性，主要两个问题：

- 在机器数量较少的时候， 区间大小会不平衡。
- 当一台机器故障的时候， 它的压力会完全转移到另外一台机器， 可能无法承载。

为了解决这个问题，我们为每个server计算m个hash值，从而把32位整数值域划分为n*m个区间，当key落到某个区间时，分流到对应的server上。那些额外的hash值使得区间划分更加均匀，被称为Virtual Node。当删除一个server时，它对应的m个区间会分别合入相邻的区间中，那个server上的请求会较为平均地转移到其他server上。当增加server时，它会分割m个现有区间，从对应server上分别转移一些请求过来。

由于节点故障和变化不常发生， 我们选择了修改复杂度为O(n)的有序数组来存储hash ring，每次分流使用二分查找来选择对应的机器， 由于存储是连续的，查找效率比基于平衡二叉树的实现高。 线程安全性请参照[Double Buffered Data](lalb.md#doublybuffereddata)章节.

# 使用方式

我们内置了分别基于murmurhash3和md5两种hash算法的实现， 使用要做两件事：

- 在Channel.Init 时指定*load_balancer_name*为 "c_murmurhash" 或 "c_md5"。

- 发起rpc时通过Controller::set_request_code()填入请求的hash code。

> request的hash算法并不需要和lb的hash算法保持一致，只需要hash的值域是32位无符号整数。由于memcache默认使用md5，访问memcached集群时请选择c_md5保证兼容性， 其他场景可以选择c_murmurhash以获得更高的性能和更均匀的分布。


# ========= ./contention_profiler.md ========
brpc可以分析花在等待锁上的时间及发生等待的函数。

# 开启方法

按需开启。无需配置，不依赖tcmalloc，不需要链接frame pointer或libunwind。如果只是brpc client或没有使用brpc，看[这里](dummy_server.md)。 

# 图示

当很多线程争抢同一把锁时，一些线程无法立刻获得锁，而必须睡眠直到某个线程退出临界区。这个争抢过程我们称之为**contention**。在多核机器上，当多个线程需要操作同一个资源却被一把锁挡住时，便无法充分发挥多个核心的并发能力。现代OS通过提供比锁更底层的同步原语，使得无竞争锁完全不需要系统调用，只是一两条wait-free，耗时10-20ns的原子操作，非常快。而锁一旦发生竞争，一些线程就要陷入睡眠，再次醒来触发了OS的调度代码，代价至少为3-5us。所以让锁尽量无竞争，让所有线程“一起飞”是需要高性能的server的永恒话题。

r31906后brpc支持contention profiler，可以分析在等待锁上花费了多少时间。等待过程中线程是睡着的不会占用CPU，所以contention profiler中的时间并不是cpu时间，也不会出现在[cpu profiler](cpu_profiler.md)中。cpu profiler可以抓到特别频繁的锁（以至于花费了很多cpu），但耗时真正巨大的临界区往往不是那么频繁，而无法被cpu profiler发现。**contention profiler和cpu profiler好似互补关系，前者分析等待时间（被动），后者分析忙碌时间。**还有一类由用户基于condition或sleep发起的主动等待时间，无需分析。

目前contention profiler支持pthread_mutex_t（非递归）和bthread_mutex_t，开启后每秒最多采集1000个竞争锁，这个数字由参数-bvar_collector_expected_per_second控制（同时影响rpc_dump）。

| Name                               | Value | Description                              | Defined At         |
| ---------------------------------- | ----- | ---------------------------------------- | ------------------ |
| bvar_collector_expected_per_second | 1000  | Expected number of samples to be collected per second | bvar/collector.cpp |

如果一秒内竞争锁的次数Ｎ超过了1000，那么每把锁会有1000/N的概率被采集。在我们的各类测试场景中（qps在10万-60万不等）没有观察到被采集程序的性能有明显变化。

我们通过实际例子来看下如何使用contention profiler，点击“contention”按钮（more左侧）后就会开启默认10秒的分析过程。下图是libraft中的一个示例程序的锁状况，这个程序是3个节点复制组的leader，qps在10-12万左右。左上角的**Total seconds: 2.449**是采集时间内（10秒）在锁上花费的所有等待时间。注意是“等待”，无竞争的锁不会被采集也不会出现在下图中。顺着箭头往下走能看到每份时间来自哪些函数。

![img](../images/raft_contention_1.png)

 上图有点大，让我们放大一个局部看看。下图红框中的0.768是这个局部中最大的数字，它代表raft::LogManager::get_entry在等待涉及到bvar::detail::UniqueLockBase的函数上共等待了0.768秒（10秒内）。我们如果觉得这个时间不符合预期，就可以去排查代码。

![img](../images/raft_contention_2.png)

点击上方的count选择框，可以查看锁的竞争次数。选择后左上角变为了**Total samples: 439026**，代表采集时间内总共的锁竞争次数（估算）。图中箭头上的数字也相应地变为了次数，而不是时间。对比同一份结果的时间和次数，可以更深入地理解竞争状况。

![img](../images/raft_contention_3.png)


# ========= ./cpu_profiler.md ========
brpc可以分析程序中的热点函数。

# 开启方法

1. 链接`libtcmalloc_and_profiler.a`
   1. 这么写也开启了tcmalloc，不建议单独链接cpu profiler而不链接tcmalloc，可能越界访问导致[crash](https://github.com/gperftools/gperftools/blob/master/README#L226).可能由于tcmalloc不及时归还内存，越界访问不会crash。
   2. 如果tcmalloc使用frame pointer而不是libunwind回溯栈，请确保在CXXFLAGS或CFLAGS中加上`-fno-omit-frame-pointer`，否则函数间的调用关系会丢失，最后产生的图片中都是彼此独立的函数方框。
2. 定义宏BRPC_ENABLE_CPU_PROFILER, 一般加入编译参数-DBRPC_ENABLE_CPU_PROFILER。
3. 如果只是brpc client或没有使用brpc，看[这里](dummy_server.md)。 

 注意要关闭Server端的认证，否则可能会看到这个：

```
$ tools/pprof --text localhost:9002/pprof/profile
Use of uninitialized value in substitution (s///) at tools/pprof line 2703.
http://localhost:9002/profile/symbol doesn't exist
```

server端可能会有这样的日志：

```
FATAL: 12-26 10:01:25:   * 0 [src/brpc/policy/giano_authenticator.cpp:65][4294969345] Giano fails to verify credentical, 70003
WARNING: 12-26 10:01:25:   * 0 [src/brpc/input_messenger.cpp:132][4294969345] Authentication failed, remote side(127.0.0.1:22989) of sockfd=5, close it
```

# 图示

下图是一次运行cpu profiler后的结果：

- 左上角是总体信息，包括时间，程序名，总采样数等等。
- View框中可以选择查看之前运行过的profile结果，Diff框中可选择查看和之前的结果的变化量，重启后清空。
- 代表函数调用的方框中的字段从上到下依次为：函数名，这个函数本身（除去所有子函数）占的采样数和比例，这个函数及调用的所有子函数累计的采样数和比例。采样数越大框越大。
- 方框之间连线上的数字表示被采样到的上层函数对下层函数的调用数，数字越大线越粗。

热点分析一般开始于找到最大的框最粗的线考察其来源及去向。

cpu profiler的原理是在定期被调用的SIGPROF handler中采样所在线程的栈，由于handler（在linux 2.6后）会被随机地摆放于活跃线程的栈上运行，cpu profiler在运行一段时间后能以很大的概率采集到所有活跃线程中的活跃函数，最后根据栈代表的函数调用关系汇总为调用图，并把地址转换成符号，这就是我们看到的结果图了。采集频率由环境变量CPUPROFILE_FREQUENCY控制，默认100，即每秒钟100次或每10ms一次。在实践中cpu profiler对原程序的影响不明显。

![img](../images/echo_cpu_profiling.png)

在Linux下，你也可以使用[pprof](https://github.com/brpc/brpc/blob/master/tools/pprof)或gperftools中的pprof进行profiling。

比如`pprof --text localhost:9002 --seconds=5`的意思是统计运行在本机9002端口的server的cpu情况，时长5秒。一次运行的例子如下：

```
$ tools/pprof --text 0.0.0.0:9002 --seconds=5
Gathering CPU profile from http://0.0.0.0:9002/pprof/profile?seconds=5 for 5 seconds to
  /home/gejun/pprof/echo_server.1419501210.0.0.0.0
Be patient...
Wrote profile to /home/gejun/pprof/echo_server.1419501210.0.0.0.0
Removing funlockfile from all stack traces.
Total: 2946 samples
    1161  39.4%  39.4%     1161  39.4% syscall
     248   8.4%  47.8%      248   8.4% bthread::TaskControl::steal_task
     227   7.7%  55.5%      227   7.7% writev
      87   3.0%  58.5%       88   3.0% ::cpp_alloc
      74   2.5%  61.0%       74   2.5% __read_nocancel
      46   1.6%  62.6%       48   1.6% tc_delete
      42   1.4%  64.0%       42   1.4% brpc::Socket::Address
      41   1.4%  65.4%       41   1.4% epoll_wait
      35   1.2%  66.6%       35   1.2% memcpy
      33   1.1%  67.7%       33   1.1% __pthread_getspecific
      33   1.1%  68.8%       33   1.1% brpc::Socket::Write
      33   1.1%  69.9%       33   1.1% epoll_ctl
      28   1.0%  70.9%       42   1.4% brpc::policy::ProcessRpcRequest
      27   0.9%  71.8%       27   0.9% butil::IOBuf::_push_back_ref
      27   0.9%  72.7%       27   0.9% bthread::TaskGroup::ending_sched
```

省略–text进入交互模式，如下图所示：

```
$ tools/pprof localhost:9002 --seconds=5       
Gathering CPU profile from http://0.0.0.0:9002/pprof/profile?seconds=5 for 5 seconds to
  /home/gejun/pprof/echo_server.1419501236.0.0.0.0
Be patient...
Wrote profile to /home/gejun/pprof/echo_server.1419501236.0.0.0.0
Removing funlockfile from all stack traces.
Welcome to pprof!  For help, type 'help'.
(pprof) top
Total: 2954 samples
    1099  37.2%  37.2%     1099  37.2% syscall
     253   8.6%  45.8%      253   8.6% bthread::TaskControl::steal_task
     240   8.1%  53.9%      240   8.1% writev
      90   3.0%  56.9%       90   3.0% ::cpp_alloc
      67   2.3%  59.2%       67   2.3% __read_nocancel
      47   1.6%  60.8%       47   1.6% butil::IOBuf::_push_back_ref
      42   1.4%  62.2%       56   1.9% brpc::policy::ProcessRpcRequest
      41   1.4%  63.6%       41   1.4% epoll_wait
      38   1.3%  64.9%       38   1.3% epoll_ctl
      37   1.3%  66.1%       37   1.3% memcpy
      35   1.2%  67.3%       35   1.2% brpc::Socket::Address
```

# MacOS的额外配置

在MacOS下，gperftools中的perl pprof脚本无法将函数地址转变成函数名，解决办法是：

1. 安装[standalone pprof](https://github.com/google/pprof)，并把下载的pprof二进制文件路径写入环境变量GOOGLE_PPROF_BINARY_PATH中
2. 安装llvm-symbolizer（将函数符号转化为函数名），直接用brew安装即可：`brew install llvm`


# ========= ./dummy_server.md ========
如果你的程序只使用了brpc的client或根本没有使用brpc，但你也想使用brpc的内置服务，只要在程序中启动一个空的server就行了，这种server我们称为**dummy server**。

# 使用了brpc的client

只要在程序运行目录建立dummy_server.port文件，填入一个端口号（比如8888），程序会马上在这个端口上启动一个dummy server。在浏览器中访问它的内置服务，便可看到同进程内的所有bvar。
![img](../images/dummy_server_1.png) ![img](../images/dummy_server_2.png) 

![img](../images/dummy_server_3.png)

# 没有使用brpc

你必须手动加入dummy server。你得先查看[Getting Started](getting_started.md)如何下载和编译brpc，然后在程序入口处加入如下代码片段：

```c++
#include <brpc/server.h>
 
...
 
int main() {
    ...
    brpc::StartDummyServerAt(8888/*port*/);
    ...
}
```


# ========= ./error_code.md ========
[English version](../en/error_code.md)

brpc使用[brpc::Controller](https://github.com/brpc/brpc/blob/master/src/brpc/controller.h)设置和获取一次RPC的参数，`Controller::ErrorCode()`和`Controller::ErrorText()`则分别是该次RPC的错误码和错误描述，RPC结束后才能访问，否则结果未定义。ErrorText()由Controller的基类google::protobuf::RpcController定义，ErrorCode()则是brpc::Controller定义的。Controller还有个Failed()方法告知该次RPC是否失败，这三者的关系是：

- 当Failed()为true时，ErrorCode()一定为非0，ErrorText()则为非空。
- 当Failed()为false时，ErrorCode()一定为0，ErrorText()未定义（目前在brpc中会为空，但你最好不要依赖这个事实）

# 标记RPC为错误

brpc的client端和server端都有Controller，都可以通过SetFailed()修改其中的ErrorCode和ErrorText。当多次调用一个Controller的SetFailed时，ErrorCode会被覆盖，ErrorText则是**添加**而不是覆盖。在client端，框架会额外加上第几次重试，在server端，框架会额外加上server的地址信息。

client端Controller的SetFailed()常由框架调用，比如发送request失败，接收到的response不符合要求等等。只有在进行较复杂的访问操作时用户才可能需要设置client端的错误，比如在访问后端前做额外的请求检查，发现有错误时把RPC设置为失败。

server端Controller的SetFailed()常由用户在服务回调中调用。当处理过程发生错误时，一般调用SetFailed()并释放资源后就return了。框架会把错误码和错误信息按交互协议填入response，client端的框架收到后会填入它那边的Controller中，从而让用户在RPC结束后取到。需要注意的是，**server端在SetFailed()时默认不打印送往client的错误**。打日志是比较慢的，在繁忙的线上磁盘上，很容易出现巨大的lag。一个错误频发的client容易减慢整个server的速度而影响到其他的client，理论上来说这甚至能成为一种攻击手段。对于希望在server端看到错误信息的场景，可以打开gflag **-log_error_text**(可动态开关)，server会在每次失败的RPC后把对应Controller的ErrorText()打印出来。

# brpc的错误码

brpc使用的所有ErrorCode都定义在[errno.proto](https://github.com/brpc/brpc/blob/master/src/brpc/errno.proto)中，*SYS_*开头的来自linux系统，与/usr/include/errno.h中定义的精确一致，定义在proto中是为了跨语言。其余的是brpc自有的。

[berror(error_code)](https://github.com/brpc/brpc/blob/master/src/butil/errno.h)可获得error_code的描述，berror()可获得当前[system errno](http://www.cplusplus.com/reference/cerrno/errno/)的描述。**ErrorText() != berror(ErrorCode())**，ErrorText()会包含更具体的错误信息。brpc默认包含berror，你可以直接使用。

brpc中常见错误的打印内容列表如下：

 

| 错误码            | 数值   | 重试   | 说明                                       | 日志                                       |
| -------------- | ---- | ---- | ---------------------------------------- | ---------------------------------------- |
| EAGAIN         | 11   | 是    | 同时发送的请求过多。软限，很少出现。                       | Resource temporarily unavailable         |
| ETIMEDOUT      | 110  | 是    | 连接超时。                                    | Connection timed out                     |
| EHOSTDOWN      | 112  | 是    | 找不到可用的server。server可能停止服务了，也可能正在退出中(返回了ELOGOFF)。 | "Fail to select server from …"  "Not connected to … yet" |
| ENOSERVICE     | 1001 | 否    | 找不到服务，不太出现，一般会返回ENOMETHOD。               |                                          |
| ENOMETHOD      | 1002 | 否    | 找不到方法。                                   | 形式广泛，常见如"Fail to find method=..."        |
| EREQUEST       | 1003 | 否    | request序列化错误，client端和server端都可能设置        | 形式广泛："Missing required fields in request: …" "Fail to parse request message, …"  "Bad request" |
| EAUTH          | 1004 | 否    | 认证失败                                     | "Authentication failed"                  |
| ETOOMANYFAILS  | 1005 | 否    | ParallelChannel中太多子channel失败             | "%d/%d channels failed, fail_limit=%d"   |
| EBACKUPREQUEST | 1007 | 是    | 触发backup request时设置，不会出现在ErrorCode中，但可在/rpcz里看到 | “reached backup timeout=%dms"            |
| ERPCTIMEDOUT   | 1008 | 否    | RPC超时                                    | "reached timeout=%dms"                   |
| EFAILEDSOCKET  | 1009 | 是    | RPC进行过程中TCP连接出现问题                        | "The socket was SetFailed"               |
| EHTTP          | 1010 | 否    | 非2xx状态码的HTTP访问结果均认为失败并被设置为这个错误码。默认不重试，可通过RetryPolicy定制 | Bad http call                            |
| EOVERCROWDED   | 1011 | 是    | 连接上有过多的未发送数据，常由同时发起了过多的异步访问导致。可通过参数-socket_max_unwritten_bytes控制，默认8MB。 | The server is overcrowded                |
| EINTERNAL      | 2001 | 否    | Server端Controller.SetFailed没有指定错误码时使用的默认错误码。 | "Internal Server Error"                  |
| ERESPONSE      | 2002 | 否    | response解析错误，client端和server端都可能设置        | 形式广泛"Missing required fields in response: ...""Fail to parse response message, ""Bad response" |
| ELOGOFF        | 2003 | 是    | Server已经被Stop了                           | "Server is going to quit"                |
| ELIMIT         | 2004 | 是    | 同时处理的请求数超过ServerOptions.max_concurrency了 | "Reached server's limit=%d on concurrent requests" |

# 自定义错误码

在C++/C中你可以通过宏、常量、enum等方式定义ErrorCode:
```c++
#define ESTOP -114                // C/C++
static const int EMYERROR = 30;   // C/C++
const int EMYERROR2 = -31;        // C++ only
```
如果你需要用berror返回这些新错误码的描述，你可以在.cpp或.c文件的全局域中调用BAIDU_REGISTER_ERRNO(error_code, description)进行注册，比如：
```c++
BAIDU_REGISTER_ERRNO(ESTOP, "the thread is stopping")
BAIDU_REGISTER_ERRNO(EMYERROR, "my error")
```
strerror和strerror_r不认识使用BAIDU_REGISTER_ERRNO定义的错误码，自然地，printf类的函数中的%m也不能转化为对应的描述，你必须使用%s并配以berror()。
```c++
errno = ESTOP;
printf("Describe errno: %m\n");                              // [Wrong] Describe errno: Unknown error -114
printf("Describe errno: %s\n", strerror_r(errno, NULL, 0));  // [Wrong] Describe errno: Unknown error -114
printf("Describe errno: %s\n", berror());                    // [Correct] Describe errno: the thread is stopping
printf("Describe errno: %s\n", berror(errno));               // [Correct] Describe errno: the thread is stopping
```
当同一个error code被重复注册时，那么会出现链接错误：

```
redefinition of `class BaiduErrnoHelper<30>'
```
或者在程序启动时会abort：
```
Fail to define EMYERROR(30) which is already defined as `Read-only file system', abort
```

你得确保不同的模块对ErrorCode的理解是相同的，否则当两个模块把一个错误码理解为不同的错误时，它们之间的交互将出现无法预计的行为。为了防止这种情况出现，你最好这么做：
- 优先使用系统错误码，它们的值和含义一般是固定不变的。
- 多个交互的模块使用同一份错误码定义，防止后续修改时产生不一致。
- 使用BAIDU_REGISTER_ERRNO描述新错误码，以确保同一个进程内错误码是互斥的。 


# ========= ./execution_queue.md ========
# 概述

类似于kylin的ExecMan, [ExecutionQueue](https://github.com/brpc/brpc/blob/master/src/bthread/execution_queue.h)提供了异步串行执行的功能。ExecutionQueue的相关技术最早使用在RPC中实现[多线程向同一个fd写数据](io.md#发消息). 在r31345之后加入到bthread。 ExecutionQueue 提供了如下基本功能:

- 异步有序执行: 任务在另外一个单独的线程中执行, 并且执行顺序严格和提交顺序一致.
- Multi Producer: 多个线程可以同时向一个ExecutionQueue提交任务
- 支持cancel一个已经提交的任务
- 支持stop
- 支持高优任务插队

和ExecMan的主要区别:
- ExecutionQueue的任务提交接口是[wait-free](https://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom)的, ExecMan依赖了lock, 这意味着当机器整体比较繁忙的时候，使用ExecutionQueue不会因为某个进程被系统强制切换导致所有线程都被阻塞。
- ExecutionQueue支持批量处理: 执行线程可以批量处理提交的任务, 获得更好的locality. ExecMan的某个线程处理完某个AsyncClient的AsyncContext之后下一个任务很可能是属于另外一个AsyncClient的AsyncContex, 这时候cpu cache会在不同AsyncClient依赖的资源间进行不停的切换。
- ExecutionQueue的处理函数不会被绑定到固定的线程中执行, ExecMan中是根据AsyncClient hash到固定的执行线程，不同的ExecutionQueue之间的任务处理完全独立，当线程数足够多的情况下，所有非空闲的ExecutionQueue都能同时得到调度。同时也意味着当线程数不足的时候，ExecutionQueue无法保证公平性, 当发生这种情况的时候需要动态增加bthread的worker线程来增加整体的处理能力.
- ExecutionQueue运行线程为bthread, 可以随意的使用一些bthread同步原语而不用担心阻塞pthread的执行. 而在ExecMan里面得尽量避免使用较高概率会导致阻塞的同步原语.

# 背景

在多核并发编程领域， [Message passing](https://en.wikipedia.org/wiki/Message_passing)作为一种解决竞争的手段得到了比较广泛的应用，它按照业务依赖的资源将逻辑拆分成若干个独立actor，每个actor负责对应资源的维护工作，当一个流程需要修改某个资源的时候，
就转化为一个消息发送给对应actor，这个actor(通常在另外的上下文中)根据命令内容对这个资源进行相应的修改，之后可以选择唤醒调用者(同步)或者提交到下一个actor(异步)的方式进行后续处理。

![img](http://web.mit.edu/6.005/www/fa14/classes/20-queues-locks/figures/producer-consumer.png)

# ExecutionQueue Vs Mutex

ExecutionQueue和mutex都可以用来在多线程场景中消除竞争. 相比较使用mutex,
使用ExecutionQueue有着如下几个优点:

- 角色划分比较清晰, 概念理解上比较简单, 实现中无需考虑锁带来的问题(比如死锁)
- 能保证任务的执行顺序，mutex的唤醒顺序不能得到严格保证.
- 所有线程各司其职，都能在做有用的事情，不存在等待.
- 在繁忙、卡顿的情况下能更好的批量执行，整体上获得较高的吞吐.

但是缺点也同样明显:

- 一个流程的代码往往散落在多个地方，代码理解和维护成本高。
- 为了提高并发度， 一件事情往往会被拆分到多个ExecutionQueue进行流水线处理，这样会导致在多核之间不停的进行切换，会付出额外的调度以及同步cache的开销, 尤其是竞争的临界区非常小的情况下， 这些开销不能忽略.
- 同时原子的操作多个资源实现会变得复杂, 使用mutex可以同时锁住多个mutex, 用了ExeuctionQueue就需要依赖额外的dispatch queue了。
- 由于所有操作都是单线程的，某个任务运行慢了就会阻塞同一个ExecutionQueue的其他操作。
- 并发控制变得复杂，ExecutionQueue可能会由于缓存的任务过多占用过多的内存。

不考虑性能和复杂度，理论上任何系统都可以只使用mutex或者ExecutionQueue来消除竞争.
但是复杂系统的设计上，建议根据不同的场景灵活决定如何使用这两个工具:

- 如果临界区非常小，竞争又不是很激烈，优先选择使用mutex, 之后可以结合[contention profiler](contention_profiler.md)来判断mutex是否成为瓶颈。
- 需要有序执行，或者无法消除的激烈竞争但是可以通过批量执行来提高吞吐， 可以选择使用ExecutionQueue。

总之，多线程编程没有万能的模型，需要根据具体的场景，结合丰富的profliling工具，最终在复杂度和性能之间找到合适的平衡。

**特别指出一点**，Linux中mutex无竞争的lock/unlock只有需要几条原子指令，在绝大多数场景下的开销都可以忽略不计.

# 使用方式

### 实现执行函数

```
// Iterate over the given tasks
//
// Example:
//
// #include <bthread/execution_queue.h>
//
// int demo_execute(void* meta, TaskIterator<T>& iter) {
//     if (iter.is_stopped()) {
//         // destroy meta and related resources
//         return 0;
//     }
//     for (; iter; ++iter) {
//         // do_something(meta, *iter)
//         // or do_something(meta, iter->a_member_of_T)
//     }
//     return 0;
// }
template <typename T>
class TaskIterator;
```

### 启动一个ExecutionQueue:

```
// Start a ExecutionQueue. If |options| is NULL, the queue will be created with
// default options.
// Returns 0 on success, errno otherwise
// NOTE: type |T| can be non-POD but must be copy-constructible
template <typename T>
int execution_queue_start(
        ExecutionQueueId<T>* id,
        const ExecutionQueueOptions* options,
        int (*execute)(void* meta, TaskIterator<T>& iter),
        void* meta);
```

创建的返回值是一个64位的id, 相当于ExecutionQueue实例的一个[弱引用](https://en.wikipedia.org/wiki/Weak_reference), 可以wait-free的在O(1)时间内定位一个ExecutionQueue, 你可以到处拷贝这个id， 甚至可以放在RPC中，作为远端资源的定位工具。
你必须保证meta的生命周期，在对应的ExecutionQueue真正停止前不会释放.

### 停止一个ExecutionQueue:

```
// Stop the ExecutionQueue.
// After this function is called:
//  - All the following calls to execution_queue_execute would fail immediately.
//  - The executor will call |execute| with TaskIterator::is_queue_stopped() being
//    true exactly once when all the pending tasks have been executed, and after
//    this point it's ok to release the resource referenced by |meta|.
// Returns 0 on success, errno othrwise
template <typename T>
int execution_queue_stop(ExecutionQueueId<T> id);
 
// Wait until the the stop task (Iterator::is_queue_stopped() returns true) has
// been executed
template <typename T>
int execution_queue_join(ExecutionQueueId<T> id);
```

stop和join都可以多次调用， 都会又合理的行为。stop可以随时调用而不用当心线程安全性问题。

和fd的close类似，如果stop不被调用, 相应的资源会永久泄露。

安全释放meta的时机: 可以在execute函数中收到iter.is_queue_stopped()==true的任务的时候释放，也可以等到join返回之后释放. 注意不要double-free

### 提交任务

```
struct TaskOptions {
    TaskOptions();
    TaskOptions(bool high_priority, bool in_place_if_possible);
 
    // Executor would execute high-priority tasks in the FIFO order but before
    // all pending normal-priority tasks.
    // NOTE: We don't guarantee any kind of real-time as there might be tasks still
    // in process which are uninterruptible.
    //
    // Default: false
    bool high_priority;
 
    // If |in_place_if_possible| is true, execution_queue_execute would call
    // execute immediately instead of starting a bthread if possible
    //
    // Note: Running callbacks in place might cause the dead lock issue, you
    // should be very careful turning this flag on.
    //
    // Default: false
    bool in_place_if_possible;
};
 
const static TaskOptions TASK_OPTIONS_NORMAL = TaskOptions(/*high_priority=*/ false, /*in_place_if_possible=*/ false);
const static TaskOptions TASK_OPTIONS_URGENT = TaskOptions(/*high_priority=*/ true, /*in_place_if_possible=*/ false);
const static TaskOptions TASK_OPTIONS_INPLACE = TaskOptions(/*high_priority=*/ false, /*in_place_if_possible=*/ true);
 
// Thread-safe and Wait-free.
// Execute a task with defaut TaskOptions (normal task);
template <typename T>
int execution_queue_execute(ExecutionQueueId<T> id,
                            typename butil::add_const_reference<T>::type task);
 
// Thread-safe and Wait-free.
// Execute a task with options. e.g
// bthread::execution_queue_execute(queue, task, &bthread::TASK_OPTIONS_URGENT)
// If |options| is NULL, we will use default options (normal task)
// If |handle| is not NULL, we will assign it with the hanlder of this task.
template <typename T>
int execution_queue_execute(ExecutionQueueId<T> id,
                            typename butil::add_const_reference<T>::type task,
                            const TaskOptions* options);
template <typename T>
int execution_queue_execute(ExecutionQueueId<T> id,
                            typename butil::add_const_reference<T>::type task,
                            const TaskOptions* options,
                            TaskHandle* handle); 
```

high_priority的task之间的执行顺序也会**严格按照提交顺序**, 这点和ExecMan不同, ExecMan的QueueExecEmergent的AsyncContex执行顺序是undefined. 但是这也意味着你没有办法将任何任务插队到一个high priority的任务之前执行.

开启inplace_if_possible, 在无竞争的场景中可以省去一次线程调度和cache同步的开销. 但是可能会造成死锁或者递归层数过多(比如不停的ping-pong)的问题，开启前请确定你的代码中不存在这些问题。

### 取消一个已提交任务

```
/// [Thread safe and ABA free] Cancel the corresponding task.
// Returns:
//  -1: The task was executed or h is an invalid handle
//  0: Success
//  1: The task is executing
int execution_queue_cancel(const TaskHandle& h);
```

返回非0仅仅意味着ExecutionQueue已经将对应的task递给过execute, 真实的逻辑中可能将这个task缓存在另外的容器中，所以这并不意味着逻辑上的task已经结束，你需要在自己的业务上保证这一点.


# ========= ./flags.md ========
brpc使用gflags管理配置。如果你的程序也使用gflags，那么你应该已经可以修改和brpc相关的flags，你可以浏览[flags服务](http://brpc.baidu.com:8765/flags)了解每个flag的具体功能。如果你的程序还没有使用gflags，我们建议你使用，原因如下：

- 命令行和文件均可传入，前者方便做测试，后者适合线上运维。放在文件中的gflags可以reload。而configure只支持从文件读取配置。
- 你可以在浏览器中查看brpc服务器中所有gflags，并对其动态修改（如果允许的话）。configure不可能做到这点。
- gflags分散在和其作用紧密关联的文件中，更好管理。而使用configure需要聚集到一个庞大的读取函数中。

# Usage of gflags

gflags一般定义在需要它的源文件中。#include <gflags/gflags.h>后在全局scope加入DEFINE_*<type>*(*<name>*, *<default-value>*, *<description>*); 比如：

```c++
#include <gflags/gflags.h>
...
DEFINE_bool(hex_log_id, false, "Show log_id in hexadecimal");
DEFINE_int32(health_check_interval, 3, "seconds between consecutive health-checkings");
```

一般在main函数开头用ParseCommandLineFlags处理程序参数：

```c++
#include <gflags/gflags.h>
...
int main(int argc, char* argv[]) {
    google::ParseCommandLineFlags(&argc, &argv, true/*表示把识别的参数从argc/argv中删除*/);
    ...
}
```

如果要从conf/gflags.conf中加载gflags，则可以加上参数-flagfile=conf/gflags.conf。如果希望默认（什么参数都不加）就从文件中读取，则可以在程序中直接给flagfile赋值，一般这么写

```c++
google::SetCommandLineOption("flagfile", "conf/gflags.conf");
```

程序启动时会检查conf/gflags.conf是否存在，如果不存在则会报错：

```
$ ./my_program
conf/gflags.conf: No such file or directory
```

更具体的使用指南请阅读[官方文档](http://gflags.github.io/gflags/)。

# flagfile

在命令行中参数和值之间可不加等号，而在flagfile中一定要加。比如`./myapp -param 7`是ok的，但在`./myapp -flagfile=./gflags.conf`对应的gflags.conf中一定要写成**-param=7**或**--param=7**，否则就不正确且不会报错。

在命令行中字符串可用单引号或双引号包围，而在flagfile中不能加。比如`./myapp -name="tom"`或`./myapp -name='tom'`都是ok的，但在`./myapp -flagfile=./gflags.conf`对应的gflags.conf中一定要写成**-name=tom**或**--name=tom**，如果写成-name="tom"的话，引号也会作为值的一部分。配置文件中的值可以有空格，比如gflags.conf中写成-name=value with spaces是ok的，参数name的值就是value with spaces，而在命令行中要用引号括起来。

flagfile中参数可由单横线(如-foo)或双横线(如--foo)打头，但不能以三横线或更多横线打头，否则的话是无效参数且不会报错!

flagfile中以`#开头的行被认为是注释。开头的空格和空白行都会被忽略。`

flagfile中可以使用`--flagfile包含另一个flagfile。`

# Change gflag on-the-fly

[flags服务](http://brpc.baidu.com:8765/flags)可以查看服务器进程中所有的gflags。修改过的flags会以红色高亮。“修改过”指的是修改这一行为，即使再改回默认值，仍然会显示为红色。

/flags：列出所有的gflags

/flags/NAME：查询名字为NAME的gflag

/flags/NAME1,NAME2,NAME3：查询名字为NAME1或NAME2或NAME3的gflag

/flags/foo*,b$r：查询名字与某一统配符匹配的gflag，注意用$代替?匹配单个字符，因为?在url中有特殊含义。

访问/flags/NAME?setvalue=VALUE即可动态修改一个gflag的值，validator会被调用。

为了防止误修改，需要动态修改的gflag必须有validator，显示此类gflag名字时有(R)后缀。

![img](../images/reloadable_flags.png)

*修改成功后会显示如下信息*：

![img](../images/flag_setvalue.png)

*尝试修改不允许修改的gflag会显示如下错误信息*：

![img](../images/set_flag_reject.png)

*设置一个不允许的值会显示如下错误（flag值不会变化）*：

![img](../images/set_flag_invalid_value.png)

 

r31658之后支持可视化地修改，在浏览器上访问时将看到(R)下多了下划线：

![img](../images/the_r_after_flag.png)

点击后在一个独立页面可视化地修改对应的flag：

![img](../images/set_flag_with_form.png)

填入true后确定：

![img](../images/set_flag_with_form_2.png)

返回/flags可以看到对应的flag已经被修改了：

![img](../images/set_flag_with_form_3.png)

 

关于重载gflags，重点关注：

- 避免在一段代码中多次调用同一个gflag，应把该gflag的值保存下来并调用该值。因为gflag的值随时可能变化，而产生意想不到的结果。
- 使用google::GetCommandLineOption()访问string类型的gflag，直接访问是线程不安全的。
- 处理逻辑和副作用应放到validator里去。比如修改FLAGS_foo后得更新另一处的值，如果只是写在程序初始化的地方，而不是validator里，那么重载时这段逻辑就运行不到了。

如果你确认某个gflag不需要额外的线程同步和处理逻辑就可以重载，那么可以用如下方式为其注册一个总是返回true的validator：

```c++
DEFINE_bool(hex_log_id, false, "Show log_id in hexadecimal");
BAIDU_RPC_VALIDATE_GFLAG(hex_log_id, brpc::PassValidate/*always true*/);
```

这个flag是单纯的开关，修改后不需要更新其他数据（没有处理逻辑），代码中前面看到true后面看到false也不会产生什么后果（不需要线程同步），所以我们让其默认可重载。

对于int32和int64类型，有一个判断是否为正数的常用validator：

```c++
DEFINE_int32(health_check_interval, 3, "seconds between consecutive health-checkings");
BAIDU_RPC_VALIDATE_GFLAG(health_check_interval, brpc::PositiveInteger);
```

以上操作都可以在命令行中进行：

```shell
$ curl brpc.baidu.com:8765/flags/health_check_interval
Name | Value | Description | Defined At
---------------------------------------
health_check_interval (R) | 3 | seconds between consecutive health-checkings | src/brpc/socket_map.cpp
```

1.0.251.32399后增加了-immutable_flags，打开后所有的gflags将不能被动态修改。当一个服务对某个gflag值比较敏感且不希望在线上被误改，可打开这个开关。打开这个开关的同时也意味着你无法动态修改线上的配置，每次修改都要重启程序，对于还在调试阶段或待收敛阶段的程序不建议打开。


# ========= ./flatmap.md ========
# NAME

FlatMap - Maybe the fastest hashmap, with tradeoff of space.

# EXAMPLE

`#include <butil/containers/flat_map.h>`

# DESCRIPTION

[FlatMap](https://github.com/brpc/brpc/blob/master/src/butil/containers/flat_map.h)可能是最快的哈希表，但当value较大时它需要更多的内存，它最适合作为检索过程中需要极快查找的小字典。

原理：把开链桶中第一个节点的内容直接放桶内。由于在实践中，大部分桶没有冲突或冲突较少，所以大部分操作只需要一次内存跳转：通过哈希值访问对应的桶。桶内两个及以上元素仍存放在链表中，由于桶之间彼此独立，一个桶的冲突不会影响其他桶，性能很稳定。在很多时候，FlatMap的查找性能和原生数组接近。

# BENCHMARK

下面是FlatMap和其他key/value容器的比较：

- [AlignHashMap](https://svn.baidu.com/app/ecom/nova/trunk/public/util/container/alignhash.h)：闭链中较快的实现。
- [CowHashMap](https://svn.baidu.com/app/ecom/nova/trunk/afs/smalltable/cow_hash_map.hpp)：smalltable中的开链哈希表，和普通开链不同的是带Copy-on-write逻辑。
- [std::map](http://www.cplusplus.com/reference/map/map/)：非哈希表，一般是红黑树。

```
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:474] [ value = 8 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting   100 into FlatMap/AlignHashMap/CowHashMap/std::map takes 15/19/30/102ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing     100 from FlatMap/AlighHashMap/CowHashMap/std::map takes 7/11/33/146ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting  1000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 10/28/26/93ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing    1000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 6/9/29/100ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting 10000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 10/21/26/130ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing   10000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 5/10/30/104ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:474] [ value = 32 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting   100 into FlatMap/AlignHashMap/CowHashMap/std::map takes 23/31/31/130ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing     100 from FlatMap/AlighHashMap/CowHashMap/std::map takes 9/11/72/104ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting  1000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 20/53/28/112ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing    1000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 7/10/29/101ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting 10000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 20/46/28/137ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing   10000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 7/10/29/112ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:474] [ value = 128 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting   100 into FlatMap/AlignHashMap/CowHashMap/std::map takes 34/109/91/179ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing     100 from FlatMap/AlighHashMap/CowHashMap/std::map takes 8/11/33/112ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting  1000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 28/76/86/169ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing    1000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 8/9/30/110ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Sequentially inserting 10000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 28/68/87/201ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Sequentially erasing   10000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 9/9/30/125ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:474] [ value = 8 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting   100 into FlatMap/AlignHashMap/CowHashMap/std::map takes 14/56/29/157ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing     100 from FlatMap/AlighHashMap/CowHashMap/std::map takes 9/11/31/181ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting  1000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 11/17/27/156ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing    1000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 6/10/30/204ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting 10000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 13/26/27/212ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing   10000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 7/11/38/309ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:474] [ value = 32 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting   100 into FlatMap/AlignHashMap/CowHashMap/std::map takes 24/32/32/181ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing     100 from FlatMap/AlighHashMap/CowHashMap/std::map takes 10/12/32/182ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting  1000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 21/46/35/168ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing    1000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 7/10/36/209ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting 10000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 24/46/31/240ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing   10000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 8/11/40/314ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:474] [ value = 128 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting   100 into FlatMap/AlignHashMap/CowHashMap/std::map takes 36/114/93/231ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing     100 from FlatMap/AlighHashMap/CowHashMap/std::map takes 9/12/35/190ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting  1000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 44/94/88/224ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing    1000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 8/10/34/236ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:521] Randomly inserting 10000 into FlatMap/AlignHashMap/CowHashMap/std::map takes 46/92/93/314ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:558] Randomly erasing   10000 from FlatMap/AlighHashMap/CowHashMap/std::map takes 12/11/42/362ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:576] [ value = 8 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking   100 from FlatMap/AlignHashMap/CowHashMap/std::map takes 4/7/12/54ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking  1000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 3/7/11/78ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking 10000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 4/8/13/172ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:576] [ value = 32 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking   100 from FlatMap/AlignHashMap/CowHashMap/std::map takes 5/8/12/55ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking  1000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 4/8/11/82ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking 10000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 6/10/14/164ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:576] [ value = 128 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking   100 from FlatMap/AlignHashMap/CowHashMap/std::map takes 7/9/13/56ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking  1000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 6/10/12/93ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking 10000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 9/12/21/166ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:576] [ value = 8 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking   100 from FlatMap/AlignHashMap/CowHashMap/std::map takes 4/7/11/56ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking  1000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 3/7/11/79ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking 10000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 4/9/13/173ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:576] [ value = 32 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking   100 from FlatMap/AlignHashMap/CowHashMap/std::map takes 5/8/12/54ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking  1000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 4/8/11/100ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking 10000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 6/10/14/165ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:576] [ value = 128 bytes ]
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking   100 from FlatMap/AlignHashMap/CowHashMap/std::map takes 7/9/12/56ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking  1000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 6/10/12/88ns
TRACE: 12-30 13:19:53:   * 0 [test_flat_map.cpp:637] Seeking 10000 from FlatMap/AlignHashMap/CowHashMap/std::map takes 9/14/20/169ns
```
# Overview of hashmaps

哈希表是最常用的数据结构，它的基本原理是通过[计算哈希值](http://en.wikipedia.org/wiki/Hash_function)把不同的key分散到不同的区间，在查找时通过key的哈希值能快速地缩小查找区间。在使用恰当参数的前提下，哈希表在大部分时候能在O(1)时间内把一个key映射为value。像其他算法一样，这个“O(1)”在不同的实现中差异很大。哈希表的实现一般有两部分：

## 计算哈希值(非加密型)

即把key散列开的方法，最常见的莫过于线性同余，但一个好的哈希算法(非加密型)要考虑很多因素：

- 结果是确定的。
- [雪崩效应](http://en.wikipedia.org/wiki/Avalanche_effect)：输入中一个bit的变化应该尽量影响输出所有bit的变化。
- 输出应尽量在值域中均匀分布。
- 充分利用现代cpu特性：成块计算，减少分支，循环展开等等。

大部分哈希算法针对的只是一个key，不会耗用太多的cpu。影响主要来自哈希表的整体数据分布，对于工程师来说，选用何种算法得看实践效果，一些最简单的方法也许就有很好的效果。通用算法可选择Murmurhash。

## 解决冲突

哈希值可能重合，解决冲突是哈希表性能的另一关键因素。常见的冲突解决方法有：

- 开链哈希(open hashing, closed addressing): 开链哈希表是链表的数组，其中链表一般称为桶。当若干个key落到同一个桶时，做链表插入。这是最通用的结构，有很多优点：占用内存为O(NumElement * (KeySize + ValueSize + SomePointers))，resize时候不会使之前的存放key/value的内存失效。桶之间是独立的，一个桶的冲突不会影响到其他桶，平均查找时间较为稳定，独立的桶也易于高并发。缺点是至少要两次内存跳转：先跳到桶入口，再跳到桶中的第一个节点。对于一些很小的表这个问题不明显，因为当表很小时，节点内存是接近的，但当表变大时，访存就愈发随机。如果一次访存在50ns左右（2G左右主频），开链哈希的查找时间往往就在100ns以上。在检索端的层层ranking过程中，对一些热点字典的查找1秒内可能有几百万次以上，开链哈希有时会成为热点。一些产品线可能对开链哈希的内存也有诟病，因为每对key/value都需要额外的指针。
- [闭链哈希](http://en.wikipedia.org/wiki/Open_addressing)(closed hashing or open addressing): 闭链的初衷是减少内存跳转，桶不再是链表入口，而只需要记录一对key/value与一些标记，当桶被占时，按照不同的探查方法直到找到空桶为止。比如线性探查就是查找下一个桶，二次探查是按1,2,4,9...平方数位移查找。优点是：当表很空时或冲突较少时，查找只需要一次访存，也不需要管理节点内存池。但仅此而已，这个方法带来了更多缺点：桶个数必须大于元素个数，resize后之前的内存全部失效，难以并发. 更关键的是聚集效应：当区域内元素较多时（超过70%，其实不算多），大量元素的实际桶和它们应在的桶有较大位移。这使哈希表的主要操作都要扫过一大片内存才能找到元素，性能不稳定难以预测。闭链哈希表在很多人的印象中“很快”，但在复杂的应用中往往不如开链哈希表，并且可能是数量级的慢。闭链有一些衍生版本试图解决这个问题，比如[Hopscotch hashing](http://en.wikipedia.org/wiki/Hopscotch_hashing)。
- 混合开链和闭链：一般是把桶数组中的一部分拿出来作为容纳冲突元素的空间，典型如[Coalesced hashing](http://en.wikipedia.org/wiki/Coalesced_hashing)，但这种结构没有解决开链的内存跳转问题，结构又比闭链复杂很多，工程效果并不好。
- 多次哈希：一般用多个哈希表代替一个哈希表，当发生冲突时（用另一个哈希值）尝试另一个哈希表。典型如[Cuckoo hashing](http://en.wikipedia.org/wiki/Cuckoo_hashing)，这个结构也没有解决内存跳转。


# ========= ./getting_started.md ========
# BUILD

brpc prefers static linkages of deps, so that they don't have to be installed on every machine running the app.

brpc depends on following packages:

* [gflags](https://github.com/gflags/gflags): Extensively used to define global options.
* [protobuf](https://github.com/google/protobuf): Serializations of messages, interfaces of services.
* [leveldb](https://github.com/google/leveldb): Required by [/rpcz](rpcz.md) to record RPCs for tracing.

# Supported Environment

* [Ubuntu/LinuxMint/WSL](#ubuntulinuxmintwsl)
* [Fedora/CentOS](#fedoracentos)
* [Linux with self-built deps](#linux-with-self-built-deps)
* [MacOS](#macos)

## Ubuntu/LinuxMint/WSL
### Prepare deps

Install common deps:
```shell
sudo apt-get install git g++ make libssl-dev
```

Install [gflags](https://github.com/gflags/gflags), [protobuf](https://github.com/google/protobuf), [leveldb](https://github.com/google/leveldb):
```shell
sudo apt-get install libgflags-dev libprotobuf-dev libprotoc-dev protobuf-compiler libleveldb-dev
```

If you need to statically link leveldb:
```shell
sudo apt-get install libsnappy-dev
```

If you need to enable cpu/heap profilers in examples:
```shell
sudo apt-get install libgoogle-perftools-dev
```

If you need to run tests, install and compile libgtest-dev (which is not compiled yet):
```shell
sudo apt-get install libgtest-dev && cd /usr/src/gtest && sudo cmake . && sudo make && sudo mv libgtest* /usr/lib/ && cd -
```
The directory of gtest source code may be changed, try `/usr/src/googletest/googletest` if `/usr/src/gtest` is not there.

### Compile brpc with config_brpc.sh
git clone brpc, cd into the repo and run
```shell
$ sh config_brpc.sh --headers=/usr/include --libs=/usr/lib
$ make
```
To change compiler to clang, add `--cxx=clang++ --cc=clang`.

To not link debugging symbols, add `--nodebugsymbols` and compiled binaries will be much smaller.

To use brpc with glog, add `--with-glog`.

To enable [thrift support](../en/thrift.md), install thrift first and add `--with-thrift`.

**Run example**

```shell
$ cd example/echo_c++
$ make
$ ./echo_server &
$ ./echo_client
```

Examples link brpc statically, if you need to link the shared version, `make clean` and `LINK_SO=1 make`

**Run tests**
```shell
$ cd test
$ make
$ sh run_tests.sh
```

### Compile brpc with cmake
```shell
mkdir build && cd build && cmake .. && make
```
To change compiler to clang, overwrite environment variable CC and CXX to clang and clang++.

To not link debugging symbols, use `cmake -DWITH_DEBUG_SYMBOLS=OFF ..` and compiled binaries will be much smaller.

To use brpc with glog, add `-DWITH_GLOG=ON`.

To enable [thrift support](../en/thrift.md), install thrift first and add `-DWITH_THRIFT=ON`.

**Run example with cmake**
```shell
$ cd example/echo_c++
$ mkdir build && cd build && cmake .. && make
$ ./echo_server &
$ ./echo_client
```
Examples link brpc statically, if you need to link the shared version, use `cmake -DEXAMPLE_LINK_SO=ON ..`

**Run tests**
```shell
$ mkdir build && cd build && cmake -DBUILD_UNIT_TESTS=ON .. && make
$ cd test
$ sh run_tests.sh
```

## Fedora/CentOS

### Prepare deps

CentOS needs to install EPEL generally otherwise many packages are not available by default.
```shell
sudo yum install epel-release
```

Install common deps:
```shell
sudo yum install git gcc-c++ make openssl-devel
```

Install [gflags](https://github.com/gflags/gflags), [protobuf](https://github.com/google/protobuf), [leveldb](https://github.com/google/leveldb):
```shell
sudo yum install gflags-devel protobuf-devel protobuf-compiler leveldb-devel
```

If you need to enable cpu/heap profilers in examples:
```shell
sudo yum install gperftools-devel
```

If you need to run tests, install and compile gtest-devel (which is not compiled yet):
```shell
sudo yum install gtest-devel
```

### Compile brpc with config_brpc.sh

git clone brpc, cd into the repo and run

```shell
$ sh config_brpc.sh --headers=/usr/include --libs=/usr/lib64
$ make
```
To change compiler to clang, add `--cxx=clang++ --cc=clang`.

To not link debugging symbols, add `--nodebugsymbols` and compiled binaries will be much smaller.

To use brpc with glog, add `--with-glog`.

To enable [thrift support](../en/thrift.md), install thrift first and add `--with-thrift`.

**Run example**

```shell
$ cd example/echo_c++
$ make
$ ./echo_server &
$ ./echo_client
```

Examples link brpc statically, if you need to link the shared version, `make clean` and `LINK_SO=1 make`

**Run tests**
```shell
$ cd test
$ make
$ sh run_tests.sh
```

### Compile brpc with cmake
```shell
mkdir build && cd build && cmake .. && make
```
To change compiler to clang, overwrite environment variable CC and CXX to clang and clang++.

To not link debugging symbols, use `cmake -DWITH_DEBUG_SYMBOLS=OFF ..` and compiled binaries will be much smaller.

To use brpc with glog, add `-DWITH_GLOG=ON`.

To enable [thrift support](../en/thrift.md), install thrift first and add `-DWITH_THRIFT=ON`.

**Run example**

```shell
$ cd example/echo_c++
$ mkdir build && cd build && cmake .. && make
$ ./echo_server &
$ ./echo_client
```
Examples link brpc statically, if you need to link the shared version, use `cmake -DEXAMPLE_LINK_SO=ON ..`

**Run tests**
```shell
$ mkdir build && cd build && cmake -DBUILD_UNIT_TESTS=ON .. && make
$ cd test
$ sh run_tests.sh
```

## Linux with self-built deps

### Prepare deps

brpc builds itself to both static and shared libs by default, so it needs static and shared libs of deps to be built as well.

Take [gflags](https://github.com/gflags/gflags) as example, which does not build shared lib by default, you need to pass options to `cmake` to change the behavior:
```shell
$ cmake . -DBUILD_SHARED_LIBS=1 -DBUILD_STATIC_LIBS=1
$ make
```

### Compile brpc

Keep on with the gflags example, let `../gflags_dev` be where gflags is cloned.

git clone brpc. cd into the repo and run

```shell
$ sh config_brpc.sh --headers="../gflags_dev /usr/include" --libs="../gflags_dev /usr/lib64"
$ make
```

Here we pass multiple paths to `--headers` and `--libs` to make the script search for multiple places. You can also group all deps and brpc into one directory, then pass the directory to --headers/--libs which actually search all subdirectories recursively and will find necessary files.

To change compiler to clang, add `--cxx=clang++ --cc=clang`.

To not link debugging symbols, add `--nodebugsymbols` and compiled binaries will be much smaller.

To use brpc with glog, add `--with-glog`.

To enable [thrift support](../en/thrift.md), install thrift first and add `--with-thrift`.

```shell
$ ls my_dev
gflags_dev protobuf_dev leveldb_dev brpc_dev
$ cd brpc_dev
$ sh config_brpc.sh --headers=.. --libs=..
$ make
```

### Compile brpc with cmake

git clone brpc. cd into the repo and run

```shell
mkdir build && cd build && cmake -DCMAKE_INCLUDE_PATH="/path/to/dep1/include;/path/to/dep2/include" -DCMAKE_LIBRARY_PATH="/path/to/dep1/lib;/path/to/dep2/lib" .. && make
```

To change compiler to clang, overwrite environment variable CC and CXX to clang and clang++.

To not link debugging symbols, use `cmake -DWITH_DEBUG_SYMBOLS=OFF ..` and compiled binaries will be much smaller.

To use brpc with glog, add `-DWITH_GLOG=ON`.

To enable [thrift support](../en/thrift.md), install thrift first and add `-DWITH_THRIFT=ON`.

## MacOS

Note: In the same running environment, the performance of the current Mac version is about 2.5 times worse than the Linux version. If your service is performance-critical, do not use MacOS as your production environment.

### Prepare deps

Install common deps:
```shell
brew install openssl git gnu-getopt coreutils
```

Install [gflags](https://github.com/gflags/gflags), [protobuf](https://github.com/google/protobuf), [leveldb](https://github.com/google/leveldb):
```shell
brew install gflags protobuf leveldb
```

If you need to enable cpu/heap profilers in examples:
```shell
brew install gperftools
```

If you need to run tests, install and compile googletest (which is not compiled yet):
```shell
git clone https://github.com/google/googletest && cd googletest/googletest && mkdir build && cd build && cmake .. && make && sudo mv libgtest* /usr/lib/ && cd -
```

### Compile brpc with config_brpc.sh
git clone brpc, cd into the repo and run
```shell
$ sh config_brpc.sh --headers=/usr/local/include --libs=/usr/local/lib --cc=clang --cxx=clang++
$ make
```
To not link debugging symbols, add `--nodebugsymbols` and compiled binaries will be much smaller.

To use brpc with glog, add `--with-glog`.

To enable [thrift support](../en/thrift.md), install thrift first and add `--with-thrift`.

**Run example**

```shell
$ cd example/echo_c++
$ make
$ ./echo_server &
$ ./echo_client
```

Examples link brpc statically, if you need to link the shared version, `make clean` and `LINK_SO=1 make`

**Run tests**
```shell
$ cd test
$ make
$ sh run_tests.sh
```

### Compile brpc with cmake
```shell
mkdir build && cd build && cmake .. && make
```

To not link debugging symbols, use `cmake -DWITH_DEBUG_SYMBOLS=OFF ..` and compiled binaries will be much smaller.

To use brpc with glog, add `-DWITH_GLOG=ON`.

To enable [thrift support](../en/thrift.md), install thrift first and add `-DWITH_THRIFT=ON`.

**Run example with cmake**
```shell
$ cd example/echo_c++
$ mkdir build && cd build && cmake .. && make
$ ./echo_server &
$ ./echo_client
```
Examples link brpc statically, if you need to link the shared version, use `cmake -DEXAMPLE_LINK_SO=ON ..`

**Run tests**
```shell
$ mkdir build && cd build && cmake -DBUILD_UNIT_TESTS=ON .. && make
$ cd test
$ sh run_tests.sh
```

# Supported deps

## GCC: 4.8-7.1

c++11 is turned on by default to remove dependencies on boost (atomic).

The over-aligned issues in GCC7 is suppressed temporarily now.

Using other versions of gcc may generate warnings, contact us to fix.

Adding `-D__const__=` to cxxflags in your makefiles is a must to avoid [errno issue in gcc4+](thread_local.md).

## Clang: 3.5-4.0

no known issues.

## glibc: 2.12-2.25

no known issues.

## protobuf: 2.4-3.4

Be compatible with pb 3.x and pb 2.x with the same file:
Don't use new types in proto3 and start the proto file with `syntax="proto2";`
[tools/add_syntax_equal_proto2_to_all.sh](https://github.com/brpc/brpc/blob/master/tools/add_syntax_equal_proto2_to_all.sh)can add `syntax="proto2"` to all proto files without it.

Arena in pb 3.x is not supported yet.

## gflags: 2.0-2.21

no known issues.

## openssl: 0.97-1.1

required by https.

## tcmalloc: 1.7-2.5

brpc does **not** link [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html) by default. Users link tcmalloc on-demand.

Comparing to ptmalloc embedded in glibc, tcmalloc often improves performance. However different versions of tcmalloc may behave really differently. For example, tcmalloc 2.1 may make multi-threaded examples in brpc perform significantly worse(due to a spinlock in tcmalloc) than the one using tcmalloc 1.7 and 2.5. Even different minor versions may differ. When you program behave unexpectedly, remove tcmalloc or try another version.

Code compiled with gcc 4.8.2 and linked to a tcmalloc compiled with earlier GCC may crash or deadlock before main(), E.g:

![img](../images/tcmalloc_stuck.png)

When you meet the issue, compile tcmalloc with the same GCC.

Another common issue with tcmalloc is that it does not return memory to system as early as ptmalloc. So when there's an invalid memory access, the program may not crash directly, instead it crashes at a unrelated place, or even not crash. When you program has weird memory issues, try removing tcmalloc.

If you want to use [cpu profiler](cpu_profiler.md) or [heap profiler](heap_profiler.md), do link `libtcmalloc_and_profiler.a`. These two profilers are based on tcmalloc.[contention profiler](contention_profiler.md) does not require tcmalloc.

When you remove tcmalloc, not only remove the linkage with tcmalloc but also the macro `-DBRPC_ENABLE_CPU_PROFILER`.

## glog: 3.3+

brpc implements a default [logging utility](../../src/butil/logging.h) which conflicts with glog. To replace this with glog, add *--with-glog* to config_brpc.sh or add `-DWITH_GLOG=ON` to cmake.

## valgrind: 3.8+

brpc detects valgrind automatically (and registers stacks of bthread). Older valgrind(say 3.2) is not supported.

## thrift: 0.9.3-0.11.0

no known issues.

# Track instances

We provide a program to help you to track and monitor all brpc instances. Just run [trackme_server](https://github.com/brpc/brpc/tree/master/tools/trackme_server/) somewhere and launch need-to-be-tracked instances with -trackme_server=SERVER. The trackme_server will receive pings from instances periodically and print logs when it does. You can aggregate instance addresses from the log and call builtin services of the instances for further information.


# ========= ./heap_profiler.md ========
brpc可以分析内存是被哪些函数占据的。heap profiler的原理是每分配满一些内存就采样调用处的栈，“一些”由环境变量TCMALLOC_SAMPLE_PARAMETER控制，默认524288，即512K字节。根据栈表现出的函数调用关系汇总为我们看到的结果图。在实践中heap profiler对原程序的影响不明显。

# 开启方法

1. 链接`libtcmalloc_and_profiler.a`

   1. 如果tcmalloc使用frame pointer而不是libunwind回溯栈，请确保在CXXFLAGS或CFLAGS中加上`-fno-omit-frame-pointer`，否则函数间的调用关系会丢失，最后产生的图片中都是彼此独立的函数方框。

2. 在shell中`export TCMALLOC_SAMPLE_PARAMETER=524288`。该变量指每分配这么多字节内存时做一次统计，默认为0，代表不开启内存统计。[官方文档](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)建议设置为524288。这个变量也可在运行前临时设置，如`TCMALLOC_SAMPLE_PARAMETER=524288 ./server`。如果没有这个环境变量，可能会看到这样的结果：

   ```
   $ tools/pprof --text localhost:9002/pprof/heap           
   Fetching /pprof/heap profile from http://localhost:9002/pprof/heap to
     /home/gejun/pprof/echo_server.1419559063.localhost.pprof.heap
   Wrote profile to /home/gejun/pprof/echo_server.1419559063.localhost.pprof.heap
   /home/gejun/pprof/echo_server.1419559063.localhost.pprof.heap: header size >= 2**16
   ```

3. 如果只是brpc client或没有使用brpc，看[这里](dummy_server.md)。 

注意要关闭Server端的认证，否则可能会看到这个：

```
$ tools/pprof --text localhost:9002/pprof/heap
Use of uninitialized value in substitution (s///) at tools/pprof line 2703.
http://localhost:9002/pprof/symbol doesn't exist
```

server端可能会有这样的日志：

```
FATAL: 12-26 10:01:25:   * 0 [src/brpc/policy/giano_authenticator.cpp:65][4294969345] Giano fails to verify credentical, 70003
WARNING: 12-26 10:01:25:   * 0 [src/brpc/input_messenger.cpp:132][4294969345] Authentication failed, remote side(127.0.0.1:22989) of sockfd=5, close it
```

# 图示

![img](../images/heap_profiler_1.png)

左上角是当前程序通过malloc分配的内存总量，顺着箭头上的数字可以看到内存来自哪些函数。

点击左上角的text选择框可以查看文本格式的结果，有时候这种按分配量排序的形式更方便。

![img](../images/heap_profiler_2.png)

左上角的两个选择框作用分别是：

- View：当前正在看的profile。选择\<new profile\>表示新建一个。新建完毕后，View选择框中会出现新profile，URL也会被修改为对应的地址。这意味着你可以通过粘贴URL分享结果，点击链接的人将看到和你一模一样的结果，而不是重做profiling的结果。你可以在框中选择之前的profile查看。历史profiie保留最近的32个，可通过[--max_profiles_kept](http://brpc.baidu.com:8765/flags/max_profiles_kept)调整。
- Diff：和选择的profile做对比。<none>表示什么都不选。如果你选择了之前的某个profile，那么将看到View框中的profile相比Diff框中profile的变化量。

下图演示了勾选Diff和Text的效果。

![img](../images/heap_profiler_3.gif)

在Linux下，你也可以使用pprof脚本（tools/pprof）在命令行中查看文本格式结果：

```
$ tools/pprof --text db-rpc-dev00.db01:8765/pprof/heap    
Fetching /pprof/heap profile from http://db-rpc-dev00.db01:8765/pprof/heap to
  /home/gejun/pprof/play_server.1453216025.db-rpc-dev00.db01.pprof.heap
Wrote profile to /home/gejun/pprof/play_server.1453216025.db-rpc-dev00.db01.pprof.heap
Adjusting heap profiles for 1-in-524288 sampling rate
Heap version 2
Total: 38.9 MB
    35.8  92.0%  92.0%     35.8  92.0% ::cpp_alloc
     2.1   5.4%  97.4%      2.1   5.4% butil::FlatMap
     0.5   1.3%  98.7%      0.5   1.3% butil::IOBuf::append
     0.5   1.3% 100.0%      0.5   1.3% butil::IOBufAsZeroCopyOutputStream::Next
     0.0   0.0% 100.0%      0.6   1.5% MallocExtension::GetHeapSample
     0.0   0.0% 100.0%      0.5   1.3% ProfileHandler::Init
     0.0   0.0% 100.0%      0.5   1.3% ProfileHandlerRegisterCallback
     0.0   0.0% 100.0%      0.5   1.3% __do_global_ctors_aux
     0.0   0.0% 100.0%      1.6   4.2% _end
     0.0   0.0% 100.0%      0.5   1.3% _init
     0.0   0.0% 100.0%      0.6   1.5% brpc::CloseIdleConnections
     0.0   0.0% 100.0%      1.1   2.9% brpc::GlobalUpdate
     0.0   0.0% 100.0%      0.6   1.5% brpc::PProfService::heap
     0.0   0.0% 100.0%      1.9   4.9% brpc::Socket::Create
     0.0   0.0% 100.0%      2.9   7.4% brpc::Socket::Write
     0.0   0.0% 100.0%      3.8   9.7% brpc::Span::CreateServerSpan
     0.0   0.0% 100.0%      1.4   3.5% brpc::SpanQueue::Push
     0.0   0.0% 100.0%      1.9   4.8% butil::ObjectPool
     0.0   0.0% 100.0%      0.8   2.0% butil::ResourcePool
     0.0   0.0% 100.0%      1.0   2.6% butil::iobuf::tls_block
     0.0   0.0% 100.0%      1.0   2.6% bthread::TimerThread::Bucket::schedule
     0.0   0.0% 100.0%      1.6   4.1% bthread::get_stack
     0.0   0.0% 100.0%      4.2  10.8% bthread_id_create
     0.0   0.0% 100.0%      1.1   2.9% bvar::Variable::describe_series_exposed
     0.0   0.0% 100.0%      1.0   2.6% bvar::detail::AgentGroup
     0.0   0.0% 100.0%      0.5   1.3% bvar::detail::Percentile::operator
     0.0   0.0% 100.0%      0.5   1.3% bvar::detail::PercentileSamples
     0.0   0.0% 100.0%      0.5   1.3% bvar::detail::Sampler::schedule
     0.0   0.0% 100.0%      6.5  16.8% leveldb::Arena::AllocateNewBlock
     0.0   0.0% 100.0%      0.5   1.3% leveldb::VersionSet::LogAndApply
     0.0   0.0% 100.0%      4.2  10.8% pthread_mutex_unlock
     0.0   0.0% 100.0%      0.5   1.3% pthread_once
     0.0   0.0% 100.0%      0.5   1.3% std::_Rb_tree
     0.0   0.0% 100.0%      1.5   3.9% std::basic_string
     0.0   0.0% 100.0%      3.5   9.0% std::string::_Rep::_S_create
```

brpc还提供一个类似的growth profiler分析内存的分配去向（不考虑释放）。 

![img](../images/growth_profiler.png)

# MacOS的额外配置

1. 安装[standalone pprof](https://github.com/google/pprof)，并把下载的pprof二进制文件路径写入环境变量GOOGLE_PPROF_BINARY_PATH中
2. 安装llvm-symbolizer（将函数符号转化为函数名），直接用brew安装即可：`brew install llvm`


# ========= ./http_client.md ========
[English version](../en/http_client.md)

# 示例

[example/http_c++](https://github.com/brpc/brpc/blob/master/example/http_c++/http_client.cpp)

# 关于h2

brpc把HTTP/2协议统称为"h2"，不论是否加密。然而未开启ssl的HTTP/2连接在/connections中会按官方名称h2c显示，而开启ssl的会显示为h2。

brpc中http和h2的编程接口基本没有区别。除非特殊说明，所有提到的http特性都同时对h2有效。

# 创建Channel

brpc::Channel可访问http/h2服务，ChannelOptions.protocol须指定为PROTOCOL_HTTP或PROTOCOL_H2。

设定好协议后，`Channel::Init`的第一个参数可为任意合法的URL。注意：允许任意URL是为了省去用户取出host和port的麻烦，`Channel::Init`只用其中的host及port，其他部分都会丢弃。

```c++
brpc::ChannelOptions options;
options.protocol = brpc::PROTOCOL_HTTP;  // or brpc::PROTOCOL_H2
if (channel.Init("www.baidu.com" /*any url*/, &options) != 0) {
     LOG(ERROR) << "Fail to initialize channel";
     return -1;
}
```

http/h2 channel也支持bns地址或其他NamingService。

# GET

```c++
brpc::Controller cntl;
cntl.http_request().uri() = "www.baidu.com/index.html";  // 设置为待访问的URL
channel.CallMethod(NULL, &cntl, NULL, NULL, NULL/*done*/);
```

HTTP/h2和protobuf关系不大，所以除了Controller和done，CallMethod的其他参数均为NULL。如果要异步操作，最后一个参数传入done。

`cntl.response_attachment()`是回复的body，类型也是butil::IOBuf。IOBuf可通过to_string()转化为std::string，但是需要分配内存并拷贝所有内容，如果关注性能，处理过程应直接支持IOBuf，而不要求连续内存。

# POST

默认的HTTP Method为GET，可设置为POST或[更多http method](https://github.com/brpc/brpc/blob/master/src/brpc/http_method.h)。待POST的数据应置入request_attachment()，它([butil::IOBuf](https://github.com/brpc/brpc/blob/master/src/butil/iobuf.h))可以直接append std::string或char*。

```c++
brpc::Controller cntl;
cntl.http_request().uri() = "...";  // 设置为待访问的URL
cntl.http_request().set_method(brpc::HTTP_METHOD_POST);
cntl.request_attachment().append("{\"message\":\"hello world!\"}");
channel.CallMethod(NULL, &cntl, NULL, NULL, NULL/*done*/);
```

需要大量打印过程的body建议使用butil::IOBufBuilder，它的用法和std::ostringstream是一样的。对于有大量对象要打印的场景，IOBufBuilder简化了代码，效率也可能比c-style printf更高。

```c++
brpc::Controller cntl;
cntl.http_request().uri() = "...";  // 设置为待访问的URL
cntl.http_request().set_method(brpc::HTTP_METHOD_POST);
butil::IOBufBuilder os;
os << "A lot of printing" << printable_objects << ...;
os.move_to(cntl.request_attachment());
channel.CallMethod(NULL, &cntl, NULL, NULL, NULL/*done*/);
```
# 控制HTTP版本

brpc的http行为默认是http 1.1。

http 1.0相比1.1缺少长连接功能，brpc client与一些古老的http server通信时可能需要按如下方法设置为1.0。
```c++
cntl.http_request().set_version(1, 0);
```

设置http版本对h2无效，但是client收到的h2 response和server收到的h2 request中的version会被设置为(2, 0)。

brpc server会自动识别HTTP版本，并相应回复，无需用户设置。

# URL

URL的一般形式如下图：

```
// URI scheme : http://en.wikipedia.org/wiki/URI_scheme
//
//  foo://username:password@example.com:8042/over/there/index.dtb?type=animal&name=narwhal#nose
//  \_/   \_______________/ \_________/ \__/            \___/ \_/ \______________________/ \__/
//   |           |               |       |                |    |            |                |
//   |       userinfo           host    port              |    |          query          fragment
//   |    \________________________________/\_____________|____|/ \__/        \__/
// schema                 |                          |    |    |    |          |
//                    authority                      |    |    |    |          |
//                                                 path   |    |    interpretable as keys
//                                                        |    |
//        \_______________________________________________|____|/       \____/     \_____/
//                             |                          |    |          |           |
//                     hierarchical part                  |    |    interpretable as values
//                                                        |    |
//                                   interpretable as filename |
//                                                             |
//                                                             |
//                                               interpretable as extension
```

在上面例子中可以看到，Channel.Init()和cntl.http_request().uri()被设置了相同的URL。为什么Channel为什么不直接利用Init时传入的URL，而需要给uri()再设置一次？

确实，在简单使用场景下，这两者有所重复，但在复杂场景中，两者差别很大，比如：

- 访问命名服务(如BNS)下的多个http/h2 server。此时Channel.Init传入的是对该命名服务有意义的名称（如BNS中的节点名称），对uri()的赋值则是包含Host的完整URL(比如"www.foo.com/index.html?name=value")。
- 通过http/h2 proxy访问目标server。此时Channel.Init传入的是proxy server的地址，但uri()填入的是目标server的URL。

## Host字段

若用户自己填写了"host"字段(大小写不敏感)，框架不会修改。

若用户没有填且URL中包含host，比如http://www.foo.com/path，则http request中会包含"Host: www.foo.com"。

若用户没有填且URL不包含host，比如"/index.html?name=value"，则框架会以目标server的ip和port为Host，地址为10.46.188.39:8989的http server将会看到"Host: 10.46.188.39:8989"。

对应的字段在h2中叫":authority"。

# 常见设置

以http request为例 (对response的操作自行替换), 常见操作方式如下所示：

访问名为Foo的header
```c++
const std::string* value = cntl->http_request().GetHeader("Foo"); //不存在为NULL
```
设置名为Foo的header
```c++
cntl->http_request().SetHeader("Foo", "value");
```
访问名为Foo的query
```c++
const std::string* value = cntl->http_request().uri().GetQuery("Foo"); // 不存在为NULL
```
设置名为Foo的query
```c++
cntl->http_request().uri().SetQuery("Foo", "value");
```
设置HTTP Method
```c++
cntl->http_request().set_method(brpc::HTTP_METHOD_POST);
```
设置url
```c++
cntl->http_request().uri() = "http://www.baidu.com";
```
设置content-type
```c++
cntl->http_request().set_content_type("text/plain");
```
访问body
```c++
butil::IOBuf& buf = cntl->request_attachment();
std::string str = cntl->request_attachment().to_string(); // 有拷贝
```
设置body
```c++
cntl->request_attachment().append("....");
butil::IOBufBuilder os; os << "....";
os.move_to(cntl->request_attachment());
```

Notes on http header:

- 根据[rfc2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html#sec4.2)，header的field_name部分不区分大小写。brpc支持大小写不敏感，同时还能在打印时保持field_name大小写与用户设定的相同。
- 如果HTTP头中出现了相同的field_name, 根据[rfc2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html#sec4.2)，value应合并到一起，用逗号(,)分隔，用户自己确定如何理解和处理此类value.
- query之间用"&"分隔, key和value之间用"="分隔, value可以省略，比如key1=value1&key2&key3=value3中key2是合理的query，值为空字符串。

# 查看HTTP消息

打开[-http_verbose](http://brpc.baidu.com:8765/flags/http_verbose)即可看到所有的http/h2 request和response，注意这应该只用于线下调试，而不是线上程序。 

# HTTP错误

当Server返回的http status code不是2xx时，该次http/h2访问被视为失败，client端会把`cntl->ErrorCode()`设置为EHTTP，用户可通过`cntl->http_response().status_code()`获得具体的http错误。同时server端可以把代表错误的html或json置入`cntl->response_attachment()`作为http body传递回来。

# 压缩request body

调用Controller::set_request_compress_type(brpc::COMPRESS_TYPE_GZIP)将尝试用gzip压缩http body。

“尝试”指的是压缩有可能不发生，条件有：

- body尺寸小于-http_body_compress_threshold指定的字节数，默认是512。这是因为gzip并不是一个很快的压缩算法，当body较小时，压缩增加的延时可能比网络传输省下的还多。

# 解压response body

出于通用性考虑brpc不会自动解压response body，解压代码并不复杂，用户可以自己做，方法如下：

```c++
#include <brpc/policy/gzip_compress.h>
...
const std::string* encoding = cntl->http_response().GetHeader("Content-Encoding");
if (encoding != NULL && *encoding == "gzip") {
    butil::IOBuf uncompressed;
    if (!brpc::policy::GzipDecompress(cntl->response_attachment(), &uncompressed)) {
        LOG(ERROR) << "Fail to un-gzip response body";
        return;
    }
    cntl->response_attachment().swap(uncompressed);
}
// cntl->response_attachment()中已经是解压后的数据了
```

# 持续下载

http client往往需要等待到body下载完整才结束RPC，这个过程中body都会存在内存中，如果body超长或无限长（比如直播用的flv文件），那么内存会持续增长，直到超时。这样的http client不适合下载大文件。

brpc client支持在读取完body前就结束RPC，让用户在RPC结束后再读取持续增长的body。注意这个功能不等同于“支持http chunked mode”，brpc的http实现一直支持解析chunked mode，这里的问题是如何让用户处理超长或无限长的body，和body是否以chunked mode传输无关。

使用方法如下：

1. 首先实现ProgressiveReader，接口如下：

   ```c++
   #include <brpc/progressive_reader.h>
   ...
   class ProgressiveReader {
   public:
       // Called when one part was read.
       // Error returned is treated as *permenant* and the socket where the
       // data was read will be closed.
       // A temporary error may be handled by blocking this function, which
       // may block the HTTP parsing on the socket.
       virtual butil::Status OnReadOnePart(const void* data, size_t length) = 0;
    
       // Called when there's nothing to read anymore. The `status' is a hint for
       // why this method is called.
       // - status.ok(): the message is complete and successfully consumed.
       // - otherwise: socket was broken or OnReadOnePart() failed.
       // This method will be called once and only once. No other methods will
       // be called after. User can release the memory of this object inside.
       virtual void OnEndOfMessage(const butil::Status& status) = 0;
   };
   ```
   OnReadOnePart在每读到一段数据时被调用，OnEndOfMessage在数据结束或连接断开时调用，实现前仔细阅读注释。

2. 发起RPC前设置`cntl.response_will_be_read_progressively();`
   这告诉brpc在读取http response时只要读完header部分RPC就可以结束了。

3. RPC结束后调用`cntl.ReadProgressiveAttachmentBy(new MyProgressiveReader);`
   MyProgressiveReader就是用户实现ProgressiveReader的实例。用户可以在这个实例的OnEndOfMessage接口中删除这个实例。

# 持续上传

目前Post的数据必须是完整生成好的，不适合POST超长的body。

# 访问带认证的Server

根据Server的认证方式生成对应的auth_data，并设置为http header "Authorization"的值。比如用的是curl，那就加上选项`-H "Authorization : <auth_data>"。`

# 发送https请求
https是http over SSL的简称，SSL并不是http特有的，而是对所有协议都有效。开启客户端SSL的一般性方法见[这里](client.md#开启ssl)。为方便使用，brpc会对https://开头的uri自动开启SSL。


# ========= ./http_derivatives.md ========
[English version](../en/http_derivatives.md)

http/h2协议的基本用法见[http_client](http_client.md)和[http_service](http_service.md)

下文中的小节名均为可填入ChannelOptions.protocol中的协议名。冒号后的内容是协议参数，用于动态选择衍生行为，但基本协议仍然是http/1.x或http/2，故这些协议在服务端只会被显示为http或h2/h2c。

# http:json, h2:json

如果pb request不为空，则会被转为json并作为http/h2请求的body，此时Controller.request_attachment()必须为空否则报错。

如果pb response不为空，则会以json解析http/h2回复的body，并转填至pb response的对应字段。

http/1.x默认是这个行为，所以"http"和"http:json"等价。

# http:proto, h2:proto

如果pb request不为空，则会把pb序列化结果作为http/h2请求的body，此时Controller.request_attachment()必须为空否则报错。

如果pb response不为空，则会以pb序列化格式解析http/h2回复的body，并填至pb response。

http/2默认是这个行为，所以"h2"和"h2:proto"等价。

# h2:grpc

[gRPC](https://github.com/grpc)的默认协议，具体格式可阅读[gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)

使用brpc的客户端把ChannelOptions.protocol设置为"h2:grpc"一般就能连通gRPC。

使用brpc的服务端一般无需修改代码即可自动被gRPC客户端访问。

gRPC默认序列化是pb二进制格式，所以"h2:grpc"和"h2:grpc+proto"等价。

TODO: gRPC其他配置

# h2:grpc+json

这个协议相比h2:grpc就是用json序列化结果代替pb序列化结果。gRPC未必直接支持这个格式，如grpc-go用户可参考[这里](https://github.com/johanbrandhorst/grpc-json-example/blob/master/codec/json.go)注册相应的codec后才支持。


# ========= ./http_service.md ========
[English version](../en/http_service.md)

这里指我们通常说的http/h2服务，而不是可通过http/h2访问的pb服务。

虽然用不到pb消息，但brpc中的http/h2服务接口也得定义在proto文件中，只是request和response都是空的结构体。这确保了所有的服务声明集中在proto文件中，而不是散列在proto文件、程序、配置等多个地方。

#示例
[http_server.cpp](https://github.com/brpc/brpc/blob/master/example/http_c++/http_server.cpp)。

# 关于h2

brpc把HTTP/2协议统称为"h2"，不论是否加密。然而未开启ssl的HTTP/2连接在/connections中会按官方名称h2c显示，而开启ssl的会显示为h2。

brpc中http和h2的编程接口基本没有区别。除非特殊说明，所有提到的http特性都同时对h2有效。

# URL类型

## 前缀为/ServiceName/MethodName

定义一个service名为ServiceName(不包含package名), method名为MethodName的pb服务，且让request和reponse定义为空，则该服务默认在/ServiceName/MethodName上提供http/h2服务。

request和response可为空是因为数据都在Controller中：

* http/h2 request的header在Controller.http_request()中，body在Controller.request_attachment()中。
* http/h2 response的header在Controller.http_response()中，body在Controller.response_attachment()中。

实现步骤如下：

1. 填写proto文件。

```protobuf
option cc_generic_services = true;
 
message HttpRequest { };
message HttpResponse { };
 
service HttpService {
      rpc Echo(HttpRequest) returns (HttpResponse);
};
```

2. 实现Service接口。和pb服务一样，也是继承定义在.pb.h中的service基类。

```c++
class HttpServiceImpl : public HttpService {
public:
    ...
    virtual void Echo(google::protobuf::RpcController* cntl_base,
                      const HttpRequest* /*request*/,
                      HttpResponse* /*response*/,
                      google::protobuf::Closure* done) {
        brpc::ClosureGuard done_guard(done);
        brpc::Controller* cntl = static_cast<brpc::Controller*>(cntl_base);
 
        // body是纯文本
        cntl->http_response().set_content_type("text/plain");
       
        // 把请求的query-string和body打印结果作为回复内容。
        butil::IOBufBuilder os;
        os << "queries:";
        for (brpc::URI::QueryIterator it = cntl->http_request().uri().QueryBegin();
                it != cntl->http_request().uri().QueryEnd(); ++it) {
            os << ' ' << it->first << '=' << it->second;
        }
        os << "\nbody: " << cntl->request_attachment() << '\n';
        os.move_to(cntl->response_attachment());
    }
};
```

3. 把实现好的服务插入Server后可通过如下URL访问，/HttpService/Echo后的部分在 cntl->http_request().unresolved_path()中。

| URL                        | 访问方法             | cntl->http_request().uri().path() | cntl->http_request().unresolved_path() |
| -------------------------- | ---------------- | --------------------------------- | -------------------------------------- |
| /HttpService/Echo          | HttpService.Echo | "/HttpService/Echo"               | ""                                     |
| /HttpService/Echo/Foo      | HttpService.Echo | "/HttpService/Echo/Foo"           | "Foo"                                  |
| /HttpService/Echo/Foo/Bar  | HttpService.Echo | "/HttpService/Echo/Foo/Bar"       | "Foo/Bar"                              |
| /HttpService//Echo///Foo// | HttpService.Echo | "/HttpService//Echo///Foo//"      | "Foo"                                  |
| /HttpService               | 访问错误             |                                   |                                        |

## 前缀为/ServiceName

资源类的http/h2服务可能需要这样的URL，ServiceName后均为动态内容。比如/FileService/foobar.txt代表./foobar.txt，/FileService/app/data/boot.cfg代表./app/data/boot.cfg。

实现方法：

1. proto文件中应以FileService为服务名，以default_method为方法名。

```protobuf
option cc_generic_services = true;

message HttpRequest { };
message HttpResponse { };

service FileService {
      rpc default_method(HttpRequest) returns (HttpResponse);
}
```

2. 实现Service。

```c++
class FileServiceImpl: public FileService {
public:
    ...
    virtual void default_method(google::protobuf::RpcController* cntl_base,
                                const HttpRequest* /*request*/,
                                HttpResponse* /*response*/,
                                google::protobuf::Closure* done) {
        brpc::ClosureGuard done_guard(done);
        brpc::Controller* cntl = static_cast<brpc::Controller*>(cntl_base);
        cntl->response_attachment().append("Getting file: ");
        cntl->response_attachment().append(cntl->http_request().unresolved_path());
    }
};
```

3. 实现完毕插入Server后可通过如下URL访问，/FileService之后的路径在cntl->http_request().unresolved_path()中i。

| URL                             | 访问方法                       | cntl->http_request().uri().path() | cntl->http_request().unresolved_path() |
| ------------------------------- | -------------------------- | --------------------------------- | -------------------------------------- |
| /FileService                    | FileService.default_method | "/FileService"                    | ""                                     |
| /FileService/123.txt            | FileService.default_method | "/FileService/123.txt"            | "123.txt"                              |
| /FileService/mydir/123.txt      | FileService.default_method | "/FileService/mydir/123.txt"      | "mydir/123.txt"                        |
| /FileService//mydir///123.txt// | FileService.default_method | "/FileService//mydir///123.txt//" | "mydir/123.txt"                        |

## Restful URL

brpc支持为service中的每个方法指定一个URL。API如下：

```c++
// 如果restful_mappings不为空, service中的方法可通过指定的URL被http/h2协议访问，而不是/ServiceName/MethodName.
// 映射格式："PATH1 => NAME1, PATH2 => NAME2 ..."
// PATHs是有效的路径, NAMEs是service中的方法名.
int AddService(google::protobuf::Service* service,
               ServiceOwnership ownership,
               butil::StringPiece restful_mappings);
```

下面的QueueService包含多个方法。如果我们像之前那样把它插入server，那么只能通过`/QueueService/start, /QueueService/stop`等url来访问。

```protobuf
service QueueService {
    rpc start(HttpRequest) returns (HttpResponse);
    rpc stop(HttpRequest) returns (HttpResponse);
    rpc get_stats(HttpRequest) returns (HttpResponse);
    rpc download_data(HttpRequest) returns (HttpResponse);
};
```

而在调用AddService时指定第三个参数(restful_mappings)就能定制URL了，如下所示：

```c++
if (server.AddService(&queue_svc,
                      brpc::SERVER_DOESNT_OWN_SERVICE,
                      "/v1/queue/start   => start,"
                      "/v1/queue/stop    => stop,"
                      "/v1/queue/stats/* => get_stats") != 0) {
    LOG(ERROR) << "Fail to add queue_svc";
    return -1;
}
 
// 星号可出现在中间
if (server.AddService(&queue_svc,
                      brpc::SERVER_DOESNT_OWN_SERVICE,
                      "/v1/*/start   => start,"
                      "/v1/*/stop    => stop,"
                      "*.data        => download_data") != 0) {
    LOG(ERROR) << "Fail to add queue_svc";
    return -1;
}
```

上面代码中AddService的第三个参数分了三行，但实际上是一个字符串。这个字符串包含以逗号(,)分隔的三个映射关系，每个映射告诉brpc：在遇到箭头左侧的URL时调用右侧的方法。"/v1/queue/stats/*"中的星号可匹配任意字串。

关于映射规则：

- 多个路径可映射至同一个方法。
- service不要求是纯http/h2，pb service也支持。
- 没有出现在映射中的方法仍旧通过/ServiceName/MethodName访问。出现在映射中的方法不再能通过/ServiceName/MethodName访问。
- ==> ===> ...都是可以的。开头结尾的空格，额外的斜杠(/)，最后多余的逗号，都不要紧。
- PATH和PATH/*两者可以共存。
- 支持后缀匹配: 星号后可以有更多字符。
- 一个路径中只能出现一个星号。

`cntl.http_request().unresolved_path()` 对应星号(*)匹配的部分，保证normalized：开头结尾都不包含斜杠(/)，中间斜杠不重复。比如：

![img](../images/restful_1.png)

或

![img](../images/restful_2.png)

unresolved_path都是`"foo/bar"`，左右、中间多余的斜杠被移除了。

 注意：`cntl.http_request().uri().path()`不保证normalized，这两个例子中分别为`"//v1//queue//stats//foo///bar//////"`和`"//vars///foo////bar/////"`

/status页面上的方法名后会加上所有相关的URL，形式是：@URL1 @URL2 ...

![img](../images/restful_3.png)

# HTTP参数

## HTTP headers

http header是一系列key/value对，有些由HTTP协议规定有特殊含义，其余则由用户自由设定。

query string也是key/value对，http headers与query string的区别:

* 虽然http headers由协议准确定义操作方式，但由于也不易在地址栏中被修改，常用于传递框架或协议层面的参数。
* query string是URL的一部分，**常见**形式是key1=value1&key2=value2&…，易于阅读和修改，常用于传递应用层参数。但query string的具体格式并不是HTTP规范的一部分，只是约定成俗。

```c++
// 获得header中"User-Agent"的值，大小写不敏感。
const std::string* user_agent_str = cntl->http_request().GetHeader("User-Agent");
if (user_agent_str != NULL) {  // has the header
    LOG(TRACE) << "User-Agent is " << *user_agent_str;
}
...
 
// 在header中增加"Accept-encoding: gzip"，大小写不敏感。
cntl->http_response().SetHeader("Accept-encoding", "gzip");
// 覆盖为"Accept-encoding: deflate"
cntl->http_response().SetHeader("Accept-encoding", "deflate");
// 增加一个value，逗号分隔，变为"Accept-encoding: deflate,gzip"
cntl->http_response().AppendHeader("Accept-encoding", "gzip");
```

## Content-Type

Content-type记录body的类型，是一个使用频率较高的header。它在brpc中被特殊处理，需要通过cntl->http_request().content_type()来访问，cntl->GetHeader("Content-Type")是获取不到的。

```c++
// Get Content-Type
if (cntl->http_request().content_type() == "application/json") {
    ...
}
...
// Set Content-Type
cntl->http_response().set_content_type("text/html");
```

如果RPC失败(Controller被SetFailed), Content-Type会框架强制设为text/plain，而response body设为Controller::ErrorText()。

## Status Code

status code是http response特有的字段，标记http请求的完成情况。可能的值定义在[http_status_code.h](https://github.com/brpc/brpc/blob/master/src/brpc/http_status_code.h)中。

```c++
// Get Status Code
if (cntl->http_response().status_code() == brpc::HTTP_STATUS_NOT_FOUND) {
    LOG(FATAL) << "FAILED: " << controller.http_response().reason_phrase();
}
...
// Set Status code
cntl->http_response().set_status_code(brpc::HTTP_STATUS_INTERNAL_SERVER_ERROR);
cntl->http_response().set_status_code(brpc::HTTP_STATUS_INTERNAL_SERVER_ERROR, 
"My explanation of the error...");
```

比如，以下代码以302错误实现重定向：

```c++
cntl->http_response().set_status_code(brpc::HTTP_STATUS_FOUND);
cntl->http_response().SetHeader("Location", "http://bj.bs.bae.baidu.com/family/image001(4979).jpg");
```

![img](../images/302.png)

## Query String

如上面的[HTTP headers](#http-headers)中提到的那样，我们按约定成俗的方式来理解query string，即key1=value1&key2=value2&...。只有key而没有value也是可以的，仍然会被GetQuery查询到，只是值为空字符串，这常被用做bool型的开关。接口定义在[uri.h](https://github.com/brpc/brpc/blob/master/src/brpc/uri.h)。

```c++
const std::string* time_value = cntl->http_request().uri().GetQuery("time");
if (time_value != NULL) {  // the query string is present
    LOG(TRACE) << "time = " << *time_value;
}

...
cntl->http_request().uri().SetQuery("time", "2015/1/2");
```

# 调试

打开[-http_verbose](http://brpc.baidu.com:8765/flags/http_verbose)即可看到所有的http/h2 request和response，注意这应该只用于线下调试，而不是线上程序。

# 压缩response body

http服务常对http body进行压缩，可以有效减少网页的传输时间，加快页面的展现速度。

设置Controller::set_response_compress_type(brpc::COMPRESS_TYPE_GZIP)后将**尝试**用gzip压缩http body。“尝试“指的是压缩有可能不发生，条件有：

- 请求中没有设置Accept-encoding或不包含gzip。比如curl不加--compressed时是不支持压缩的，这时server总是会返回不压缩的结果。

- body尺寸小于-http_body_compress_threshold指定的字节数，默认是512。gzip并不是一个很快的压缩算法，当body较小时，压缩增加的延时可能比网络传输省下的还多。当包较小时不做压缩可能是个更好的选项。

  | Name                         | Value | Description                              | Defined At                            |
  | ---------------------------- | ----- | ---------------------------------------- | ------------------------------------- |
  | http_body_compress_threshold | 512   | Not compress http body when it's less than so many bytes. | src/brpc/policy/http_rpc_protocol.cpp |

# 解压request body

出于通用性考虑且解压代码不复杂，brpc不会自动解压request body，用户可以自己做，方法如下：

```c++
#include <brpc/policy/gzip_compress.h>
...
const std::string* encoding = cntl->http_request().GetHeader("Content-Encoding");
if (encoding != NULL && *encoding == "gzip") {
    butil::IOBuf uncompressed;
    if (!brpc::policy::GzipDecompress(cntl->request_attachment(), &uncompressed)) {
        LOG(ERROR) << "Fail to un-gzip request body";
        return;
    }
    cntl->request_attachment().swap(uncompressed);
}
// cntl->request_attachment()中已经是解压后的数据了
```

# 处理https请求
https是http over SSL的简称，SSL并不是http特有的，而是对所有协议都有效。开启服务端SSL的一般性方法见[这里](server.md#开启ssl)。

# 性能

没有极端性能要求的产品都有使用HTTP协议的倾向，特别是移动产品，所以我们很重视HTTP的实现质量，具体来说：

- 使用了node.js的[http parser](https://github.com/brpc/brpc/blob/master/src/brpc/details/http_parser.h)解析http消息，这是一个轻量、优秀、被广泛使用的实现。
- 使用[rapidjson](https://github.com/miloyip/rapidjson)解析json，这是一个主打性能的json库。
- 在最差情况下解析http请求的时间复杂度也是O(N)，其中N是请求的字节数。反过来说，如果解析代码要求http请求是完整的，那么它可能会花费O(N^2)的时间。HTTP请求普遍较大，这一点意义还是比较大的。
- 来自不同client的http消息是高度并发的，即使相当复杂的http消息也不会影响对其他客户端的响应。其他rpc和[基于单线程reactor](threading_overview.md#单线程reactor)的各类http server往往难以做到这一点。

# 持续发送

brpc server支持发送超大或无限长的body。方法如下:

1. 调用Controller::CreateProgressiveAttachment()创建可持续发送的body。返回的ProgressiveAttachment对象需要用intrusive_ptr管理。
  ```c++
  #include <brpc/progressive_attachment.h>
  ...
  butil::intrusive_ptr<brpc::ProgressiveAttachment> pa(cntl->CreateProgressiveAttachment());
  ```

2. 调用ProgressiveAttachment::Write()发送数据。

   * 如果写入发生在server-side done调用前，发送的数据将会被缓存直到回调结束后才会开始发送。
   * 如果写入发生在server-side done调用后，发送的数据将立刻以chunked mode写出。

3. 发送完毕后确保所有的`butil::intrusive_ptr<brpc::ProgressiveAttachment>`都析构以释放资源。

# 持续接收

目前brpc server不支持在收齐http请求的header部分后就调用服务回调，即brpc server不适合接收超长或无限长的body。

# FAQ

### Q: brpc前的nginx报了final fail

这个错误在于brpc server直接关闭了http连接而没有发送任何回复。

brpc server一个端口支持多种协议，当它无法解析某个http请求时无法说这个请求一定是HTTP。server会对一些基本可确认是HTTP的请求返回HTTP 400错误并关闭连接，但如果是HTTP method错误(在http包开头)或严重的格式错误（可能由HTTP client的bug导致），server仍会直接断开连接，导致nginx的final fail。

解决方案: 在使用Nginx转发流量时，通过指定$HTTP_method只放行允许的方法或者干脆设置proxy_method为指定方法。

### Q: brpc支持http chunked方式传输吗

支持。

### Q: HTTP请求的query string中含有BASE64编码过的value，为什么有时候无法正常解析

根据[HTTP协议](http://tools.ietf.org/html/rfc3986#section-2.2)中的要求，以下字符应该使用%编码

```
       reserved    = gen-delims / sub-delims

       gen-delims  = ":" / "/" / "?" / "#" / "[" / "]" / "@"

       sub-delims  = "!" / "$" / "&" / "'" / "(" / ")"
                   / "*" / "+" / "," / ";" / "="
```

但Base64 编码后的字符串中，会以"="或者"=="作为结尾，比如: ?wi=NDgwMDB8dGVzdA==&anothorkey=anothervalue。这个字段可能会被正确解析，也可能不会，取决于具体实现，原则上不应做任何假设.

一个解决方法是删除末尾的"=", 不影响Base64的[正常解码](http://en.wikipedia.org/wiki/Base64#Padding); 第二个方法是在对这个URI做[percent encoding](https://en.wikipedia.org/wiki/Percent-encoding)，解码时先做percent decoding再用Base64.


# ========= ./io.md ========
[English version](../en/io.md)

一般有三种操作IO的方式：

- blocking IO: 发起IO操作后阻塞当前线程直到IO结束，标准的同步IO，如默认行为的posix [read](http://linux.die.net/man/2/read)和[write](http://linux.die.net/man/2/write)。
- non-blocking IO: 发起IO操作后不阻塞，用户可阻塞等待多个IO操作同时结束。non-blocking也是一种同步IO：“批量的同步”。如linux下的[poll](http://linux.die.net/man/2/poll),[select](http://linux.die.net/man/2/select), [epoll](http://linux.die.net/man/4/epoll)，BSD下的[kqueue](https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2)。
- asynchronous IO: 发起IO操作后不阻塞，用户得递一个回调待IO结束后被调用。如windows下的[OVERLAPPED](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684342(v=vs.85).aspx) + [IOCP](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365198(v=vs.85).aspx)。linux的native AIO只对文件有效。

linux一般使用non-blocking IO提高IO并发度。当IO并发度很低时，non-blocking IO不一定比blocking IO更高效，因为后者完全由内核负责，而read/write这类系统调用已高度优化，效率显然高于一般得多个线程协作的non-blocking IO。但当IO并发度愈发提高时，blocking IO阻塞一个线程的弊端便显露出来：内核得不停地在线程间切换才能完成有效的工作，一个cpu core上可能只做了一点点事情，就马上又换成了另一个线程，cpu cache没得到充分利用，另外大量的线程会使得依赖thread-local加速的代码性能明显下降，如tcmalloc，一旦malloc变慢，程序整体性能往往也会随之下降。而non-blocking IO一般由少量event dispatching线程和一些运行用户逻辑的worker线程组成，这些线程往往会被复用（换句话说调度工作转移到了用户态），event dispatching和worker可以同时在不同的核运行（流水线化），内核不用频繁的切换就能完成有效的工作。线程总量也不用很多，所以对thread-local的使用也比较充分。这时候non-blocking IO就往往比blocking IO快了。不过non-blocking IO也有自己的问题，它需要调用更多系统调用，比如[epoll_ctl](http://man7.org/linux/man-pages/man2/epoll_ctl.2.html)，由于epoll实现为一棵红黑树，epoll_ctl并不是一个很快的操作，特别在多核环境下，依赖epoll_ctl的实现往往会面临棘手的扩展性问题。non-blocking需要更大的缓冲，否则就会触发更多的事件而影响效率。non-blocking还得解决不少多线程问题，代码比blocking复杂很多。

# 收消息

“消息”指从连接读入的有边界的二进制串，可能是来自上游client的request或来自下游server的response。brpc使用一个或多个[EventDispatcher](https://github.com/brpc/brpc/blob/master/src/brpc/event_dispatcher.h)(简称为EDISP)等待任一fd发生事件。和常见的“IO线程”不同，EDISP不负责读取。IO线程的问题在于一个线程同时只能读一个fd，当多个繁忙的fd聚集在一个IO线程中时，一些读取就被延迟了。多租户、复杂分流算法，[Streaming RPC](streaming_rpc.md)等功能会加重这个问题。高负载下常见的某次读取卡顿会拖慢一个IO线程中所有fd的读取，对可用性的影响幅度较大。

由于epoll的[一个bug](https://patchwork.kernel.org/patch/1970231/)(开发brpc时仍有)及epoll_ctl较大的开销，EDISP使用Edge triggered模式。当收到事件时，EDISP给一个原子变量加1，只有当加1前的值是0时启动一个bthread处理对应fd上的数据。在背后，EDISP把所在的pthread让给了新建的bthread，使其有更好的cache locality，可以尽快地读取fd上的数据。而EDISP所在的bthread会被偷到另外一个pthread继续执行，这个过程即是bthread的work stealing调度。要准确理解那个原子变量的工作方式可以先阅读[atomic instructions](atomic_instructions.md)，再看[Socket::StartInputEvent](https://github.com/brpc/brpc/blob/master/src/brpc/socket.cpp)。这些方法使得brpc读取同一个fd时产生的竞争是[wait-free](http://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom)的。

[InputMessenger](https://github.com/brpc/brpc/blob/master/src/brpc/input_messenger.h)负责从fd上切割和处理消息，它通过用户回调函数理解不同的格式。Parse一般是把消息从二进制流上切割下来，运行时间较固定；Process则是进一步解析消息(比如反序列化为protobuf)后调用用户回调，时间不确定。若一次从某个fd读取出n个消息(n > 1)，InputMessenger会启动n-1个bthread分别处理前n-1个消息，最后一个消息则会在原地被Process。InputMessenger会逐一尝试多种协议，由于一个连接上往往只有一种消息格式，InputMessenger会记录下上次的选择，而避免每次都重复尝试。

可以看到，fd间和fd内的消息都会在brpc中获得并发，这使brpc非常擅长大消息的读取，在高负载时仍能及时处理不同来源的消息，减少长尾的存在。

# 发消息

"消息”指向连接写出的有边界的二进制串，可能是发向上游client的response或下游server的request。多个线程可能会同时向一个fd发送消息，而写fd又是非原子的，所以如何高效率地排队不同线程写出的数据包是这里的关键。brpc使用一种wait-free MPSC链表来实现这个功能。所有待写出的数据都放在一个单链表节点中，next指针初始化为一个特殊值(Socket::WriteRequest::UNCONNECTED)。当一个线程想写出数据前，它先尝试和对应的链表头(Socket::_write_head)做原子交换，返回值是交换前的链表头。如果返回值为空，说明它获得了写出的权利，它会在原地写一次数据。否则说明有另一个线程在写，它把next指针指向返回的头以让链表连通。正在写的线程之后会看到新的头并写出这块数据。

这套方法可以让写竞争是wait-free的，而获得写权利的线程虽然在原理上不是wait-free也不是lock-free，可能会被一个值仍为UNCONNECTED的节点锁定（这需要发起写的线程正好在原子交换后，在设置next指针前，仅仅一条指令的时间内被OS换出），但在实践中很少出现。在当前的实现中，如果获得写权利的线程一下子无法写出所有的数据，会启动一个KeepWrite线程继续写，直到所有的数据都被写出。这套逻辑非常复杂，大致原理如下图，细节请阅读[socket.cpp](https://github.com/brpc/brpc/blob/master/src/brpc/socket.cpp)。

![img](../images/write.png)

由于brpc的写出总能很快地返回，调用线程可以更快地处理新任务，后台KeepWrite写线程也能每次拿到一批任务批量写出，在大吞吐时容易形成流水线效应而提高IO效率。

# Socket

和fd相关的数据均在[Socket](https://github.com/brpc/brpc/blob/master/src/brpc/socket.h)中，是rpc最复杂的结构之一，这个结构的独特之处在于用64位的SocketId指代Socket对象以方便在多线程环境下使用fd。常用的三个方法：

- Create：创建Socket，并返回其SocketId。
- Address：取得id对应的Socket，包装在一个会自动释放的unique_ptr中(SocketUniquePtr)，当Socket被SetFailed后，返回指针为空。只要Address返回了非空指针，其内容保证不会变化，直到指针自动析构。这个函数是wait-free的。
- SetFailed：标记一个Socket为失败，之后所有对那个SocketId的Address会返回空指针（直到健康检查成功）。当Socket对象没人使用后会被回收。这个函数是lock-free的。

可以看到Socket类似[shared_ptr](http://en.cppreference.com/w/cpp/memory/shared_ptr)，SocketId类似[weak_ptr](http://en.cppreference.com/w/cpp/memory/weak_ptr)，但Socket独有的SetFailed可以在需要时确保Socket不能被继续Address而最终引用计数归0，单纯使用shared_ptr/weak_ptr则无法保证这点，当一个server需要退出时，如果请求仍频繁地到来，对应Socket的引用计数可能迟迟无法清0而导致server无法退出。另外weak_ptr无法直接作为epoll的data，而SocketId可以。这些因素使我们设计了Socket，这个类的核心部分自14年完成后很少改动，非常稳定。

存储SocketUniquePtr还是SocketId取决于是否需要强引用。像Controller贯穿了RPC的整个流程，和Socket中的数据有大量交互，它存放的是SocketUniquePtr。epoll主要是提醒对应fd上发生了事件，如果Socket回收了，那这个事件是可有可无的，所以它存放了SocketId。

由于SocketUniquePtr只要有效，其中的数据就不会变，这个机制使用户不用关心麻烦的race conditon和ABA problem，可以放心地对共享的fd进行操作。这种方法也规避了隐式的引用计数，内存的ownership明确，程序的质量有很好的保证。brpc中有大量的SocketUniquePtr和SocketId，它们确实简化了我们的开发。

事实上，Socket不仅仅用于管理原生的fd，它也被用来管理其他资源。比如SelectiveChannel中的每个Sub Channel都被置入了一个Socket中，这样SelectiveChannel可以像普通channel选择下游server那样选择一个Sub Channel进行发送。这个假Socket甚至还实现了健康检查。Streaming RPC也使用了Socket以复用wait-free的写出过程。

# The full picture

![img](../images/rpc_flow.png)


# ========= ./iobuf.md ========
[English version](../en/iobuf.md)

brpc使用[butil::IOBuf](https://github.com/brpc/brpc/blob/master/src/butil/iobuf.h)作为一些协议中的附件或http body的数据结构，它是一种非连续零拷贝缓冲，在其他项目中得到了验证并有出色的性能。IOBuf的接口和std::string类似，但不相同。

如果你之前使用Kylin中的BufHandle，你将更能感受到IOBuf的便利性：前者几乎没有实现完整，直接暴露了内部结构，用户得小心翼翼地处理引用计数，极易出错。

# IOBuf能做的：

- 默认构造不分配内存。
- 可以拷贝，修改拷贝不影响原IOBuf。拷贝的是IOBuf的管理结构而不是数据。
- 可以append另一个IOBuf，不拷贝数据。
- 可以append字符串，拷贝数据。
- 可以从fd读取，可以写入fd。
- 可以解析或序列化为protobuf messages.
- IOBufBuilder可以把IOBuf当std::ostream用。

# IOBuf不能做的：

- 程序内的通用存储结构。IOBuf应保持较短的生命周期，以避免一个IOBuf锁定了多个block (8K each)。

# 切割

从source_buf头部切下16字节放入dest_buf：

```c++
source_buf.cut(&dest_buf, 16); // 当source_buf不足16字节时，切掉所有字节。
```

从source_buf头部弹掉16字节：

```c++
source_buf.pop_front(16); // 当source_buf不足16字节时，清空它
```

# 拼接

在尾部加入另一个IOBuf：

```c++
buf.append(another_buf);  // no data copy
```

在尾部加入std::string

```c++
buf.append(str);  // copy data of str into buf
```

# 解析

解析IOBuf为protobuf message

```c++
IOBufAsZeroCopyInputStream wrapper(&iobuf);
pb_message.ParseFromZeroCopyStream(&wrapper);
```

解析IOBuf为自定义结构

```c++
IOBufAsZeroCopyInputStream wrapper(&iobuf);
CodedInputStream coded_stream(&wrapper);
coded_stream.ReadLittleEndian32(&value);
...
```

# 序列化

protobuf序列化为IOBuf

```c++
IOBufAsZeroCopyOutputStream wrapper(&iobuf);
pb_message.SerializeToZeroCopyStream(&wrapper);
```

用可打印数据创建IOBuf

```c++
IOBufBuilder os;
os << "anything can be sent to std::ostream";
os.buf();  // IOBuf
```

# 打印

可直接打印至std::ostream. 注意这个例子中的iobuf必需只包含可打印字符。

```c++
std::cout << iobuf << std::endl;
// or
std::string str = iobuf.to_string(); // 注意: 会分配内存
printf("%s\n", str.c_str());
```

# 性能

IOBuf有不错的综合性能：

| 动作                                       | 吞吐          | QPS     |
| ---------------------------------------- | ----------- | ------- |
| 文件读入->切割12+16字节->拷贝->合并到另一个缓冲->写出到/dev/null | 240.423MB/s | 8586535 |
| 文件读入->切割12+128字节->拷贝->合并到另一个缓冲->写出到/dev/null | 790.022MB/s | 5643014 |
| 文件读入->切割12+1024字节->拷贝->合并到另一个缓冲->写出到/dev/null | 1519.99MB/s | 1467171 |


# ========= ./json2pb.md ========
brpc支持json和protobuf间的**双向**转化，实现于[json2pb](https://github.com/brpc/brpc/tree/master/src/json2pb/)，json解析使用[rapidjson](https://github.com/miloyip/rapidjson)。此功能对pb2.x和3.x均有效。pb3内置了[转换json](https://developers.google.com/protocol-buffers/docs/proto3#json)的功能。

by design, 通过HTTP + json访问protobuf服务是对外服务的常见方式，故转化必须精准，转化规则列举如下。

## message

 对应rapidjson Object, 以花括号包围，其中的元素会被递归地解析。

```protobuf
// protobuf
message Foo {
    required string field1 = 1;
    required int32 field2 = 2;  
}
message Bar { 
    required Foo foo = 1; 
    optional bool flag = 2;
    required string name = 3;
}

// rapidjson
{"foo":{"field1":"hello", "field2":3},"name":"Tom" }
```

## repeated field

对应rapidjson Array, 以方括号包围，其中的元素会被递归地解析，和message不同，每个元素的类型相同。

```protobuf
// protobuf
repeated int32 numbers = 1;

// rapidjson
{"numbers" : [12, 17, 1, 24] }
```

## map

满足如下条件的repeated MSG被视作json map :

- MSG包含一个名为key的字段，类型为string，tag为1。
- MSG包含一个名为value的字段，tag为2。
- 不包含其他字段。

这种"map"的属性有：

- 自然不能确保key有序或不重复，用户视需求自行检查。
- 与protobuf 3.x中的map二进制兼容，故3.x中的map使用pb2json也会正确地转化为json map。

如果符合所有条件的repeated MSG并不需要被认为是json map，打破上面任一条件就行了: 在MSG中加入optional int32 this_message_is_not_map_entry = 3; 这个办法破坏了“不包含其他字段”这项，且不影响二进制兼容。也可以调换key和value的tag值，让前者为2后者为1，也使条件不再满足。

## integers

rapidjson会根据值打上对应的类型标记，比如：

* 对于3，rapidjson中的IsUInt, IsInt, IsUint64, IsInt64等函数均会返回true。
* 对于-1，则IsUInt和IsUint64会返回false。
* 对于5000000000，IsUInt和IsInt是false。

这使得我们不用特殊处理，转化代码就可以自动地把json中的UInt填入protobuf中的int64，而不是机械地认为这两个类型不匹配。相应地，转化代码自然能识别overflow和underflow，当出现时会转化失败。

```protobuf
// protobuf
int32 uint32 int64 uint64

// rapidjson
Int UInt Int64 UInt64
```

## floating point

json的整数类型也可以转至pb的浮点数类型。浮点数(IEEE754)除了普通数字外还接受"NaN", "Infinity", "-Infinity"三个字符串，分别对应Not A Number，正无穷，负无穷。

```protobuf
// protobuf
float double

// rapidjson
Float Double Int Uint Int64 Uint64
```

## enum

enum可转化为整数或其名字对应的字符串，可由Pb2JsonOptions.enum_options控制。默认后者。

## string

默认同名转化。但当json中出现非法C++变量名（pb的变量名规则）时，允许转化，规则是:

`illegal-char <-> **_Z**<ASCII-of-the-char>**_**`

## bytes

和string不同，可能包含\0的bytes默认以base64编码。

```protobuf
// protobuf
"Hello, World!"

// json
"SGVsbG8sIFdvcmxkIQo="
```

## bool

对应json的true false

## unknown fields

unknown_fields → json目前不支持，未来可能支持。json → unknown_fields目前也未支持，即protobuf无法透传json中不认识的字段。原因在于protobuf真正的key是proto文件中每个字段后的数字:

```protobuf
...
required int32 foo = 3; <-- the real key
...
```

这也是unknown_fields的key。当一个protobuf不认识某个字段时，其proto中必然不会有那个数字，所以没办法插入unknown_fields。

可行的方案有几种：

- 确保被json访问的服务的proto文件最新。这样就不需要透传了，但越前端的服务越类似proxy，可能并不现实。
- protobuf中定义特殊透传字段。比如名为unknown_json_fields，在解析对应的protobuf时特殊处理。此方案修改面广且对性能有一定影响，有明确需求时再议。


# ========= ./lalb.md ========
# 概述

LALB全称Locality-aware load balancing，是一个能把请求及时、自动地送到延时最低的下游的负载均衡算法，特别适合混合部署环境。该算法产生自DP系统，现已加入brpc！

LALB可以解决的问题：

- 下游的机器配置不同，访问延时不同，round-robin和随机分流效果不佳。
- 下游服务和离线服务或其他服务混部，性能难以预测。
- 自动地把大部分流量送给同机部署的模块，当同机模块出问题时，再跨机器。
- 优先访问本机房服务，出问题时再跨机房。

**…**

# 背景

最常见的分流算法是round robin和随机。这两个方法的前提是下游的机器和网络都是类似的，但在目前的线上环境下，特别是混部的产品线中，已经很难成立，因为：

- 每台机器运行着不同的程序组合，并伴随着一些离线任务，机器的可用资源在持续动态地变化着。
- 机器配置不同。
- 网络延时不同。

这些问题其实一直有，但往往被OP辛勤的机器监控和替换给隐藏了。框架层面也有过一些努力，比如UB中的[WeightedStrategy](https://svn.baidu.com/public/trunk/ub/ub_client/ubclient_weightstrategy.h)是根据下游的cpu占用率来进行分流，但明显地它解决不了延时相关的问题，甚至cpu的问题也解决不了：因为它被实现为定期reload一个权值列表，可想而知更新频率高不了，等到负载均衡反应过来，一大堆请求可能都超时了。并且这儿有个数学问题：怎么把cpu占用率转为权值。假设下游差异仅仅由同机运行的其他程序导致，机器配置和网络完全相同，两台机器权值之比是cpu idle之比吗？假如是的，当我们以这个比例给两台机器分流之后，它们的cpu idle应该会更接近对吧？而这会导致我们的分流比例也变得接近，从而使两台机器的cpu idle又出现差距。你注意到这个悖论了吗？这些因素使得这类算法的实际效果和那两个基本算法没什么差距，甚至更差，用者甚少。

我们需要一个能自适应下游负载、规避慢节点的通用分流算法。

# Locality-aware

在DP 2.0中我们使用了一种新的算法: Locality-aware load balancing，能根据下游节点的负载分配流量，还能快速规避失效的节点，在很大程度上，这种算法的延时也是全局最优的。基本原理非常简单：

> 以下游节点的吞吐除以延时作为分流权值。

比如只有两台下游节点，W代表权值，QPS代表吞吐，L代表延时，那么W1 = QPS1 / L1和W2 = QPS2 / L2分别是这两个节点的分流权值，分流时随机数落入的权值区间就是流量的目的地了。

一种分析方法如下：

- 稳定状态时的QPS显然和其分流权值W成正比，即W1 / W2 ≈ QPS1 / QPS2。
- 根据分流公式又有：W1 / W2 = QPS1 / QPS2 * (L2 / L1)。

故稳定状态时L1和L2应当是趋同的。当L1小于L2时，节点1会更获得相比其QPS1更大的W1，从而在未来获得更多的流量，直到**其延时高于平均值或没有更多的流量。**

注意这个算法并不是按照延时的比例来分流，不是说一个下游30ms，另一个60ms，它们的流量比例就是60 / 30。而是30ms的节点会一直获得流量直到它的延时高于60ms，或者没有更多流量了。以下图为例，曲线1和曲线2分别是节点1和节点2的延时与吞吐关系图，随着吞吐增大延时会逐渐升高，接近极限吞吐时，延时会飙升。左下的虚线标记了QPS=400时的延时，此时虽然节点1的延时有所上升，但还未高于节点2的基本延时（QPS=0时的延时），所以所有流量都会分给节点1，而不是按它们基本延时的比例（图中大约2:1）。当QPS继续上升达到1600时，分流比例会在两个节点延时相等时平衡，图中为9 : 7。很明显这个比例是高度非线性的，取决于不同曲线的组合，和单一指标的比例关系没有直接关联。在真实系统中，延时和吞吐的曲线也在动态变化着，分流比例更加动态。

![img](../images/lalb_1.png)

我们用一个例子来看一下具体的分流过程。启动3台server，逻辑分别是sleep 1ms，2ms，3ms，对于client来说这些值就是延时。启动client（50个同步访问线程）后每秒打印的分流结果如下：

![img](../images/lalb_2.png)

S[n]代表第n台server。由于S[1]和S[2]的平均延时大于1ms，LALB会发现这点并降低它们的权值。它们的权值会继续下降，直到被算法设定的最低值拦住。这时停掉server，反转延时并重新启动，即逻辑分别为sleep 3ms，2ms，1ms，运行一段时候后分流效果如下：

![img](../images/lalb_3.png)

刚重连上server时，client还是按之前的权值把大部分流量都分给了S[0]，但由于S[0]的延时从1ms上升到了3ms，client的qps也降到了原来的1/3。随着数据积累，LALB逐渐发现S[2]才是最快的，而把大部分流量切换了过去。同样的服务如果用rr或random访问，则qps会显著下降：

"rr" or "random": ![img](../images/lalb_4.png)

"la" :                       ![img](../images/lalb_5.png)

真实的场景中不会有这么显著的差异，但你应该能看到差别了。

这有很多应用场景：

- 如果本机有下游服务，LALB会优先访问这些最近的节点。比如CTR应用中有一个计算在1ms左右的receiver模块，被model模块访问，很多model和receiver是同机部署的，以前的分流算法必须走网络，使得receiver的延时开销较大(3-5ms)，特别是在晚上由于离线任务起来，很不稳定，失败率偏高，而LALB会优先访问本机或最近的receiver模块，很多流量都不走网络了，成功率一下子提升了很多。
- 如果同rack有下游服务，LALB也会优先访问，减少机房核心路由器的压力。甚至不同机房的服务可能不再需要隔离，LALB会优先走本机房的下游，当本机房下游出问题时再自动访问另一些机房。

但我们也不能仅看到“基本原理”，这个算法有它复杂的一面：

- 传统的经验告诉我们，不能把所有鸡蛋放一个篮子里，而按延时优化不可避免地会把很多流量送到同一个节点，如果这个节点出问题了，我们如何尽快知道并绕开它。
- 对吞吐和延时的统计都需要统计窗口，窗口越大数据越可信，噪声越少，但反应也慢了，一个异常的请求可能对统计值造不成什么影响，等我们看到统计值有显著变化时可能已经太晚了。
- 我们也不能只统计已经回来的，还得盯着路上的请求，否则我们可能会向一个已经出问题（总是不回）的节点傻傻地浪费请求。
- ”按权值分流”听上去好简单，但你能写出多线程和可能修改节点的前提下，在O(logN)时间内尽量不互斥的查找算法吗？

这些问题可以归纳为以下几个方面。

## DoublyBufferedData

LoadBalancer是一个读远多于写的数据结构：大部分时候，所有线程从一个不变的server列表中选取一台server。如果server列表真是“不变的”，那么选取server的过程就不用加锁，我们可以写更复杂的分流算法。一个方法是用读写锁，但当读临界区不是特别大时（毫秒级），读写锁并不比mutex快，而实用的分流算法不可能到毫秒级，否则开销也太大了。另一个方法是双缓冲，很多检索端用类似的方法实现无锁的查找过程，它大概这么工作：

- 数据分前台和后台。
- 检索线程只读前台，不用加锁。
- 只有一个写线程：修改后台数据，切换前后台，睡眠一段时间，以确保老前台（新后台）不再被检索线程访问。

这个方法的问题在于它假定睡眠一段时间后就能避免和前台读线程发生竞争，这个时间一般是若干秒。由于多次写之间有间隔，这儿的写往往是批量写入，睡眠时正好用于积累数据增量。

但这套机制对“server列表”不太好用：总不能插入一个server就得等几秒钟才能插入下一个吧，即使我们用批量插入，这个"冷却"间隔多少会让用户觉得疑惑：短了担心安全性，长了觉得没有必要。我们能尽量降低这个时间并使其安全么？

我们需要**写以某种形式和读同步，但读之间相互没竞争**。一种解法是，读拿一把thread-local锁，写需要拿到所有的thread-local锁。具体过程如下：

- 数据分前台和后台。
- 读拿到自己所在线程的thread-local锁，执行查询逻辑后释放锁。
- 同时只有一个写：修改后台数据，切换前后台，**挨个**获得所有thread-local锁并立刻释放，结束后再改一遍新后台（老前台）。

我们来分析下这个方法的基本原理：

- 当一个读正在发生时，它会拿着所在线程的thread-local锁，这把锁会挡住同时进行的写，从而保证前台数据不会被修改。
- 在大部分时候thread-local锁都没有竞争，对性能影响很小。
- 逐个获取thread-local锁并立刻释放是为了**确保对应的读线程看到了切换后的新前台**。如果所有的读线程都看到了新前台，写线程便可以安全地修改老前台（新后台）了。

其他特点：

- 不同的读之间没有竞争，高度并发。
- 如果没有写，读总是能无竞争地获取和释放thread-local锁，一般小于25ns，对延时基本无影响。如果有写，由于其临界区极小（拿到立刻释放），读在大部分时候仍能快速地获得锁，少数时候释放锁时可能有唤醒写线程的代价。由于写本身就是少数情况，读整体上几乎不会碰到竞争锁。

完成这些功能的数据结构是[DoublyBufferedData<>](https://github.com/brpc/brpc/blob/master/src/butil/containers/doubly_buffered_data.h)，我们常简称为DBD。brpc中的所有load balancer都使用了这个数据结构，使不同线程在分流时几乎不会互斥。而其他rpc实现往往使用了全局锁，这使得它们无法写出复杂的分流算法：否则分流代码将会成为竞争热点。

这个结构有广泛的应用场景：

- reload词典。大部分时候词典都是只读的，不同线程同时查询时不应互斥。
- 可替换的全局callback。像butil/logging.cpp支持配置全局LogSink以重定向日志，这个LogSink就是一个带状态的callback。如果只是简单的全局变量，在替换后我们无法直接删除LogSink，因为可能还有都写线程在用。用DBD可以解决这个问题。

## weight tree

LALB的查找过程是按权值分流，O(N)方法如下：获得所有权值的和total，产生一个间于[0, total-1]的随机数R，逐个遍历权值，直到当前权值之和不大于R，而下一个权值之和大于R。

这个方法可以工作，也好理解，但当N达到几百时性能已经很差，这儿的主要因素是cache一致性：LALB是一个基于反馈的算法，RPC结束时信息会被反馈入LALB，被遍历的数据结构也一直在被修改。这意味着前台的O(N)读必须刷新每一行cacheline。当N达到数百时，一次查找过程可能会耗时百微秒，更别提更大的N了，LALB（将）作为brpc的默认分流算法，这个性能开销是无法接受的。

另一个办法是用完全二叉树。每个节点记录了左子树的权值之和，这样我们就能在O(logN)时间内完成查找。当N为1024时，我们最多跳转10次内存，总耗时可控制在1微秒内，这个性能是可接受的。这个方法的难点是如何和DoublyBufferedData结合。

- 我们不考虑不使用DoublyBufferedData，那样要么绕不开锁，要么写不出正确的算法。
- 前台后必须共享权值数据，否则切换前后台时，前台积累的权值数据没法同步到后台。
- “左子树权值之和”也被前后台共享，但和权值数据不同，它和位置绑定。比如权值结构的指针可能从位置10移动到位置5，但“左子树权值之和”的指针不会移动，算法需要从原位置减掉差值，而向新位置加上差值。
- 我们不追求一致性，只要最终一致即可，这能让我们少加锁。这也意味着“权值之和”，“左子树权值之和”，“节点权值”未必能精确吻合，查找算法要能适应这一点。

最困难的部分是增加和删除节点，它们需要在整体上对前台查找不造成什么影响，详细过程请参考代码。

## base_weight

QPS和latency使用一个循环队列统计，默认容量128。我们可以使用这么小的统计窗口，是因为inflight delay能及时纠正过度反应，而128也具备了一定的统计可信度。不过，这么计算latency的缺点是：如果server的性能出现很大的变化，那么我们需要积累一段时间才能看到平均延时的变化。就像上节例子中那样，server反转延时后client需要积累很多秒的数据才能看到的平均延时的变化。目前我们并么有处理这个问题，因为真实生产环境中的server不太会像例子中那样跳变延时，大都是缓缓变慢。当集群有几百台机器时，即使我们反应慢点给个别机器少分流点也不会导致什么问题。如果在产品线中确实出现了性能跳变，并且集群规模不大，我们再处理这个问题。

权值的计算方法是base_weight = QPS * WEIGHT_SCALE / latency ^ p。其中WEIGHT_SCALE是一个放大系数，为了能用整数存储权值，又能让权值有足够的精度，类似定点数。p默认为2，延时的收敛速度大约为p=1时的p倍，选项quadratic_latency=false可使p=1。

权值计算在各个环节都有最小值限制，为了防止某个节点的权值过低而使其完全没有访问机会。即使一些延时远大于平均延时的节点，也应该有足够的权值，以确保它们可以被定期访问，否则即使它们变快了，我们也不会知道。

> 除了待删除节点，所有节点的权值绝对不会为0。

这也制造了一个问题：即使一个server非常缓慢（但没有断开连接），它的权值也不会为0，所以总会有一些请求被定期送过去而铁定超时。当qps不高时，为了降低影响面，探测间隔必须拉长。比如为了把对qps=1000的影响控制在1%%内，故障server的权值必须低至使其探测间隔为10秒以上，这降低了我们发现server变快的速度。这个问题的解决方法有：

- 什么都不干。这个问题也许没有想象中那么严重，由于持续的资源监控，线上服务很少出现“非常缓慢”的情况，一般性的变慢并不会导致请求超时。
- 保存一些曾经发向缓慢server的请求，用这些请求探测。这个办法的好处是不浪费请求。但实现起来耦合很多，比较麻烦。
- 强制backup request。
- 再选一次。

## inflight delay

我们必须追踪还未结束的RPC，否则我们就必须等待到超时或其他错误发生，而这可能会很慢（超时一般会是正常延时的若干倍），在这段时间内我们可能做出了很多错误的分流。最简单的方法是统计未结束RPC的耗时：

- 选择server时累加发出时间和未结束次数。
- 反馈时扣除发出时间和未结束次数。
- 框架保证每个选择总对应一次反馈。

这样“当前时间 - 发出时间之和 / 未结束次数”便是未结束RPC的平均耗时，我们称之为inflight delay。当inflight delay大于平均延时时，我们就线性地惩罚节点权值，即weight = base_weight * avg_latency / inflight_delay。当发向一个节点的请求没有在平均延时内回来时，它的权值就会很快下降，从而纠正我们的行为，这比等待超时快多了。不过这没有考虑延时的正常抖动，我们还得有方差，方差可以来自统计，也可简单线性于平均延时。不管怎样，有了方差bound后，当inflight delay > avg_latency + max(bound * 3, MIN_BOUND)时才会惩罚权值。3是正态分布中的经验数值。


# ========= ./load_balancing.md ========
上游一般通过命名服务发现所有的下游节点，并通过多种负载均衡方法把流量分配给下游节点。当下游节点出现问题时，它可能会被隔离以提高负载均衡的效率。被隔离的节点定期被健康检查，成功后重新加入正常节点。

# 命名服务

在brpc中，[NamingService](https://github.com/brpc/brpc/blob/master/src/brpc/naming_service.h)用于获得服务名对应的所有节点。一个直观的做法是定期调用一个函数以获取最新的节点列表。但这会带来一定的延时（定期调用的周期一般在若干秒左右），作为通用接口不太合适。特别当命名服务提供事件通知时(比如zk)，这个特性没有被利用。所以我们反转了控制权：不是我们调用用户函数，而是用户在获得列表后调用我们的接口，对应[NamingServiceActions](https://github.com/brpc/brpc/blob/master/src/brpc/naming_service.h)。当然我们还是得启动进行这一过程的函数，对应NamingService::RunNamingService。下面以三个实现解释这套方式：

- bns：没有事件通知，所以我们只能定期去获得最新列表，默认间隔是[5秒](http://brpc.baidu.com:8765/flags/ns_access_interval)。为了简化这类定期获取的逻辑，brpc提供了[PeriodicNamingService](https://github.com/brpc/brpc/blob/master/src/brpc/periodic_naming_service.h) 供用户继承，用户只需要实现单次如何获取（GetServers）。获取后调用NamingServiceActions::ResetServers告诉框架。框架会对列表去重，和之前的列表比较，通知对列表有兴趣的观察者(NamingServiceWatcher)。这套逻辑会运行在独立的bthread中，即NamingServiceThread。一个NamingServiceThread可能被多个Channel共享，通过intrusive_ptr管理ownership。
- file：列表即文件。合理的方式是在文件更新后重新读取。[该实现](https://github.com/brpc/brpc/blob/master/src/brpc/policy/file_naming_service.cpp)使用[FileWatcher](https://github.com/brpc/brpc/blob/master/src/butil/files/file_watcher.h)关注文件的修改时间，当文件修改后，读取并调用NamingServiceActions::ResetServers告诉框架。
- list：列表就在服务名里（逗号分隔）。在读取完一次并调用NamingServiceActions::ResetServers后就退出了，因为列表再不会改变了。

如果用户需要建立这些对象仍然是不够方便的，因为总是需要一些工厂代码根据配置项建立不同的对象，鉴于此，我们把工厂类做进了框架，并且是非常方便的形式：

```
"protocol://service-name"
 
e.g.
bns://<node-name>            # baidu naming service
file://<file-path>           # load addresses from the file
list://addr1,addr2,...       # use the addresses separated by comma
http://<url>                 # Domain Naming Service, aka DNS.
```

这套方式是可扩展的，实现了新的NamingService后在[global.cpp](https://github.com/brpc/brpc/blob/master/src/brpc/global.cpp)中依葫芦画瓢注册下就行了，如下图所示：

![img](../images/register_ns.png)

看到这些熟悉的字符串格式，容易联想到ftp:// zk:// galileo://等等都是可以支持的。用户在新建Channel时传入这类NamingService描述，并能把这些描述写在各类配置文件中。

# 负载均衡

brpc中[LoadBalancer](https://github.com/brpc/brpc/blob/master/src/brpc/load_balancer.h)从多个服务节点中选择一个节点，目前的实现见[负载均衡](client.md#负载均衡)。

Load balancer最重要的是如何让不同线程中的负载均衡不互斥，解决这个问题的技术是[DoublyBufferedData](lalb.md#doublybuffereddata)。

和NamingService类似，我们使用字符串来指代一个load balancer，在global.cpp中注册：

![img](../images/register_lb.png)

# 健康检查

对于那些无法连接却仍在NamingService的节点，brpc会定期连接它们，成功后对应的Socket将被”复活“，并可能被LoadBalancer选择上，这个过程就是健康检查。注意：被健康检查或在LoadBalancer中的节点一定在NamingService中。换句话说，只要一个节点不从NamingService删除，它要么是正常的（会被LoadBalancer选上），要么在做健康检查。

传统的做法是使用一个线程做所有连接的健康检查，brpc简化了这个过程：为需要的连接动态创建一个bthread专门做健康检查（Socket::HealthCheckThread）。这个线程的生命周期被对应连接管理。具体来说，当Socket被SetFailed后，健康检查线程就可能启动（如果SocketOptions.health_check_interval为正数的话）：

- 健康检查线程先在确保没有其他人在使用Socket了后关闭连接。目前是通过对Socket的引用计数判断的。这个方法之所以有效在于Socket被SetFailed后就不能被Address了，所以引用计数只减不增。
- 定期连接直到远端机器被连接上，在这个过程中，如果Socket析构了，那么该线程也就随之退出了。
- 连上后复活Socket(Socket::Revive)，这样Socket就又能被其他地方，包括LoadBalancer访问到了（通过Socket::Address）。


# ========= ./memcache_client.md ========
[English version](../en/memcache_client.md)

[memcached](http://memcached.org/)是常用的缓存服务，为了使用户更快捷地访问memcached并充分利用bthread的并发能力，brpc直接支持memcache协议。示例程序：[example/memcache_c++](https://github.com/brpc/brpc/tree/master/example/memcache_c++/)

**注意**：brpc只支持memcache的二进制协议。memcached在1.3前只有文本协议，但在当前看来支持的意义甚微。如果你的memcached早于1.3，升级版本。

相比使用[libmemcached](http://libmemcached.org/libMemcached.html)(官方client)的优势有：

- 线程安全。用户不需要为每个线程建立独立的client。
- 支持同步、异步、半同步等访问方式，能使用[ParallelChannel等](combo_channel.md)组合访问方式。
- 支持多种[连接方式](client.md#连接方式)。支持超时、backup request、取消、tracing、内置服务等一系列brpc提供的福利。
- 有明确的request和response。而libmemcached是没有的，收到的消息不能直接和发出的消息对应上，用户得做额外开发，而且并没有那么容易做对。

当前实现充分利用了RPC的并发机制并尽量避免了拷贝。一个client可以轻松地把一个同机memcached实例(版本1.4.15)压到极限：单连接9万，多连接33万。在大部分情况下，brpc client能充分发挥memcached的性能。

# 访问单台memcached

创建一个访问memcached的Channel：

```c++
#include <brpc/memcache.h>
#include <brpc/channel.h>
 
brpc::ChannelOptions options;
options.protocol = brpc::PROTOCOL_MEMCACHE;
if (channel.Init("0.0.0.0:11211", &options) != 0) {  // 11211是memcached的默认端口
   LOG(FATAL) << "Fail to init channel to memcached";
   return -1;
}
... 
```

往memcached中设置一份数据。

```c++
// 写入key="hello" value="world" flags=0xdeadbeef，10秒失效，无视cas。
brpc::MemcacheRequest request;
brpc::MemcacheResponse response;
brpc::Controller cntl;
if (!request.Set("hello", "world", 0xdeadbeef/*flags*/, 10/*expiring seconds*/, 0/*ignore cas*/)) {
    LOG(FATAL) << "Fail to SET request";
    return -1;
} 
channel.CallMethod(NULL, &cntl, &request, &response, NULL/*done*/);
if (cntl.Failed()) {
    LOG(FATAL) << "Fail to access memcached, " << cntl.ErrorText();
    return -1;
}  
if (!response.PopSet(NULL)) {
    LOG(FATAL) << "Fail to SET memcached, " << response.LastError();
    return -1;   
}
...
```

上述代码的说明：

- 请求类型必须为MemcacheRequest，回复类型必须为MemcacheResponse，否则CallMethod会失败。不需要stub，直接调用channel.CallMethod，method填NULL。
- 调用request.XXX()增加操作，本例XXX=Set，一个request多次调用不同的操作，这些操作会被同时送到memcached（常被称为pipeline模式）。
- 依次调用response.PopXXX()弹出操作结果，本例XXX=Set，成功返回true，失败返回false，调用response.LastError()可获得错误信息。XXX必须和request的依次对应，否则失败。本例中若用PopGet就会失败，错误信息为“not a GET response"。
- Pop结果独立于RPC结果。即使“不能把某个值设入memcached”，RPC可能还是成功的。RPC失败指连接断开，超时之类的。如果业务上认为要成功操作才算成功，那么你不仅要判RPC成功，还要判PopXXX是成功的。

目前支持的请求操作有：

```c++
bool Set(const Slice& key, const Slice& value, uint32_t flags, uint32_t exptime, uint64_t cas_value);
bool Add(const Slice& key, const Slice& value, uint32_t flags, uint32_t exptime, uint64_t cas_value);
bool Replace(const Slice& key, const Slice& value, uint32_t flags, uint32_t exptime, uint64_t cas_value);
bool Append(const Slice& key, const Slice& value, uint32_t flags, uint32_t exptime, uint64_t cas_value);
bool Prepend(const Slice& key, const Slice& value, uint32_t flags, uint32_t exptime, uint64_t cas_value);
bool Delete(const Slice& key);
bool Flush(uint32_t timeout);
bool Increment(const Slice& key, uint64_t delta, uint64_t initial_value, uint32_t exptime);
bool Decrement(const Slice& key, uint64_t delta, uint64_t initial_value, uint32_t exptime);
bool Touch(const Slice& key, uint32_t exptime);
bool Version();
```

对应的回复操作：

```c++
// Call LastError() of the response to check the error text when any following operation fails.
bool PopGet(IOBuf* value, uint32_t* flags, uint64_t* cas_value);
bool PopGet(std::string* value, uint32_t* flags, uint64_t* cas_value);
bool PopSet(uint64_t* cas_value);
bool PopAdd(uint64_t* cas_value);
bool PopReplace(uint64_t* cas_value);
bool PopAppend(uint64_t* cas_value);
bool PopPrepend(uint64_t* cas_value);
bool PopDelete();
bool PopFlush();
bool PopIncrement(uint64_t* new_value, uint64_t* cas_value);
bool PopDecrement(uint64_t* new_value, uint64_t* cas_value);
bool PopTouch();
bool PopVersion(std::string* version);
```

# 访问memcached集群

建立一个使用c_md5负载均衡算法的channel就能访问挂载在对应命名服务下的memcached集群了。注意每个MemcacheRequest应只包含一个操作或确保所有的操作是同一个key。如果request包含了多个操作，在当前实现下这些操作总会送向同一个server，假如对应的key分布在多个server上，那么结果就不对了，这个情况下你必须把一个request分开为多个，每个包含一个操作。

或者你可以沿用常见的[twemproxy](https://github.com/twitter/twemproxy)方案。这个方案虽然需要额外部署proxy，还增加了延时，但client端仍可以像访问单点一样的访问它。


# ========= ./memory_management.md ========
内存管理总是程序中的重要一环，在多线程时代，一个好的内存分配大都在如下两点间权衡：

- 线程间竞争少。内存分配的粒度大都比较小，对性能敏感，如果不同的线程在大多数分配时会竞争同一份资源或同一把锁，性能将会非常糟糕，原因无外乎和cache一致性有关，已被大量的malloc方案证明。
- 浪费的空间少。如果每个线程各申请各的，速度也许不错，但万一一个线程总是申请，另一个线程总是释放，内存就爆炸了。线程之间总是要共享内存的，如何共享就是方案的关键了。

一般的应用可以使用[tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)、[jemalloc](https://github.com/jemalloc/jemalloc)等成熟的内存分配方案，但这对于较为底层，关注性能长尾的应用是不够的。多线程框架广泛地通过传递对象的ownership来让问题异步化，如何让分配这些小对象的开销变的更小是值得研究的问题。其中的一个特点较为显著：

- 大多数结构是等长的。

这个属性可以大幅简化内存分配的过程，获得比通用malloc更稳定、快速的性能。brpc中的ResourcePool<T>和ObjectPool<T>即提供这类分配。

> 这篇文章不鼓励用户使用ResourcePool<T>或ObjectPool<T>，事实上我们反对用户在程序中使用这两个类。因为”等长“的副作用是某个类型独占了一部分内存，这些内存无法再被其他类型使用，如果不加控制的滥用，反而会在程序中产生大量彼此隔离的内存分配体系，既浪费内存也不见得会有更好的性能。

# ResourcePool<T>

创建一个类型为T的对象并返回一个偏移量，这个偏移量可以在O(1)时间内转换为对象指针。这个偏移量相当于指针，但它的值在一般情况下小于2^32，所以我们可以把它作为64位id的一部分。对象可以被归还，但归还后对象并没有删除，也没有被析构，而是仅仅进入freelist。下次申请时可能会取到这种使用过的对象，需要重置后才能使用。当对象被归还后，通过对应的偏移量仍可以访问到对象，即ResourcePool只负责内存分配，并不解决ABA问题。但对于越界的偏移量，ResourcePool会返回空。

由于对象等长，ResourcePool通过批量分配和归还内存以避免全局竞争，并降低单次的开销。每个线程的分配流程如下：

1. 查看thread-local free block。如果还有free的对象，返回。没有的话步骤2。
2. 尝试从全局取一个free block，若取到的话回到步骤1，否则步骤3。
3. 从全局取一个block，返回其中第一个对象。

原理是比较简单的。工程实现上数据结构、原子变量、memory fence等问题会复杂一些。下面以bthread_t的生成过程说明ResourcePool是如何被应用的。

# ObjectPool<T>

这是ResourcePool<T>的变种，不返回偏移量，而直接返回对象指针。内部结构和ResourcePool类似，一些代码更加简单。对于用户来说，这就是一个多线程下的对象池，brpc里也是这么用的。比如Socket::Write中把每个待写出的请求包装为WriteRequest，这个对象就是用ObjectPool<WriteRequest>分配的。

# 生成bthread_t

用户期望通过创建bthread获得更高的并发度，所以创建bthread必须很快。 在目前的实现中创建一个bthread的平均耗时小于200ns。如果每次都要从头创建，是不可能这么快的。创建过程更像是从一个bthread池子中取一个实例，我们又同时需要一个id来指代一个bthread，所以这儿正是ResourcePool的用武之地。bthread在代码中被称作Task，其结构被称为TaskMeta，定义在[task_meta.h](https://github.com/brpc/brpc/blob/master/src/bthread/task_meta.h)中，所有的TaskMeta由ResourcePool<TaskMeta>分配。

bthread的大部分函数都需要在O(1)时间内通过bthread_t访问到TaskMeta，并且当bthread_t失效后，访问应返回NULL以让函数做出返回错误。解决方法是：bthread_t由32位的版本和32位的偏移量组成。版本解决[ABA问题](http://en.wikipedia.org/wiki/ABA_problem)，偏移量由ResourcePool<TaskMeta>分配。查找时先通过偏移量获得TaskMeta，再检查版本，如果版本不匹配，说明bthread失效了。注意：这只是大概的说法，在多线程环境下，即使版本相等，bthread仍可能随时失效，在不同的bthread函数中处理方法都是不同的，有些函数会加锁，有些则能忍受版本不相等。

![img](../images/resource_pool.png)

这种id生成方式在brpc中应用广泛，brpc中的SocketId，bthread_id_t也是用类似的方法分配的。

# 栈

使用ResourcePool加快创建的副作用是：一个pool中所有bthread的栈必须是一样大的。这似乎限制了用户的选择，不过基于我们的观察，大部分用户并不关心栈的具体大小，而只需要两种大小的栈：尺寸普通但数量较少，尺寸小但数量众多。所以我们用不同的pool管理不同大小的栈，用户可以根据场景选择。两种栈分别对应属性BTHREAD_ATTR_NORMAL（栈默认为1M）和BTHREAD_ATTR_SMALL（栈默认为32K）。用户还可以指定BTHREAD_ATTR_LARGE，这个属性的栈大小和pthread一样，由于尺寸较大，bthread不会对其做caching，创建速度较慢。server默认使用BTHREAD_ATTR_NORMAL运行用户代码。

栈使用[mmap](http://linux.die.net/man/2/mmap)分配，bthread还会用mprotect分配4K的guard page以检测栈溢出。由于mmap+mprotect不能超过max_map_count（默认为65536），当bthread非常多后可能要调整此参数。另外当有很多bthread时，内存问题可能不仅仅是栈，也包括各类用户和系统buffer。

goroutine在1.3前通过[segmented stacks](https://gcc.gnu.org/wiki/SplitStacks)动态地调整栈大小，发现有[hot split](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)问题后换成了变长连续栈（类似于vector resizing，只适合内存托管的语言）。由于bthread基本只会在64位平台上使用，虚存空间庞大，对变长栈需求不明确。加上segmented stacks的性能有影响，bthread暂时没有变长栈的计划。


# ========= ./new_protocol.md ========
# server端多协议

brpc server一个端口支持多种协议，大部分时候这对部署和运维更加方便。由于不同协议的格式大相径庭，严格地来说，一个端口很难无二义地支持所有协议。出于解耦和可扩展性的考虑，也不太可能集中式地构建一个针对所有协议的分类器。我们的做法就是把协议归三类后逐个尝试：

- 第一类协议：标记或特殊字符在最前面，比如[baidu_std](baidu_std.md)，hulu_pbrpc的前4个字符分别是PRPC和HULU，解析代码只需要检查前4个字节就可以知道协议是否匹配，最先尝试这类协议。这些协议在同一个连接上也可以共存。
- 第二类协议：有较为复杂的语法，没有固定的协议标记或特殊字符，可能在解析一段输入后才能判断是否匹配，目前此类协议只有http。
- 第三类协议：协议标记或特殊字符在中间，比如nshead的magic_num在第25-28字节。由于之前的字段均为二进制，难以判断正确性，在没有读取完28字节前，我们无法判定消息是不是nshead格式的，所以处理起来很麻烦，若其解析排在http之前，那么<=28字节的http消息便可能无法被解析，因为程序以为是“还未完整的nshead消息”。

考虑到大多数链接上只会有一种协议，我们会记录前一次的协议选择结果，下次首先尝试。对于长连接，这几乎把甄别协议的开销降到了0；虽然短连接每次都得运行这段逻辑，但由于短连接的瓶颈也往往不在于此，这套方法仍旧是足够快的。在未来如果有大量的协议加入，我们可能得考虑一些更复杂的启发式的区分方法。

# client端多协议

不像server端必须根据连接上的数据动态地判定协议，client端作为发起端，自然清楚自己的协议格式，只要某种协议只通过连接池或短链接发送，即独占那个连接，那么它可以是任意复杂（或糟糕的）格式。因为client端发出时会记录所用的协议，等到response回来时直接调用对应的解析函数，没有任何甄别代价。像memcache，redis这类协议中基本没有magic number，在server端较难和其他协议区分开，但让client端支持却没什么问题。

# 支持新协议

brpc就是设计为可随时扩展新协议的，步骤如下：

> 以nshead开头的协议有统一支持，看[这里](nshead_service.md)。

## 增加ProtocolType

在[options.proto](https://github.com/brpc/brpc/blob/master/src/brpc/options.proto)的ProtocolType中增加新协议类型，如果你需要的话可以联系我们增加，以确保不会和其他人的需求重合。

目前的ProtocolType（18年中）:
```c++
enum ProtocolType {
    PROTOCOL_UNKNOWN = 0;
    PROTOCOL_BAIDU_STD = 1;
    PROTOCOL_STREAMING_RPC = 2;
    PROTOCOL_HULU_PBRPC = 3;
    PROTOCOL_SOFA_PBRPC = 4;
    PROTOCOL_RTMP = 5;
    PROTOCOL_HTTP = 6;
    PROTOCOL_PUBLIC_PBRPC = 7;
    PROTOCOL_NOVA_PBRPC = 8;
    PROTOCOL_NSHEAD_CLIENT = 9;        // implemented in brpc-ub
    PROTOCOL_NSHEAD = 10;
    PROTOCOL_HADOOP_RPC = 11;
    PROTOCOL_HADOOP_SERVER_RPC = 12;
    PROTOCOL_MONGO = 13;               // server side only
    PROTOCOL_UBRPC_COMPACK = 14;
    PROTOCOL_DIDX_CLIENT = 15;         // Client side only
    PROTOCOL_REDIS = 16;               // Client side only
    PROTOCOL_MEMCACHE = 17;            // Client side only
    PROTOCOL_ITP = 18;
    PROTOCOL_NSHEAD_MCPACK = 19;
    PROTOCOL_DISP_IDL = 20;            // Client side only
    PROTOCOL_ERSDA_CLIENT = 21;        // Client side only
    PROTOCOL_UBRPC_MCPACK2 = 22;       // Client side only
    // Reserve special protocol for cds-agent, which depends on FIFO right now
    PROTOCOL_CDS_AGENT = 23;           // Client side only
    PROTOCOL_ESP = 24;                 // Client side only
    PROTOCOL_THRIFT = 25;              // Server side only
}
```
## 实现回调

均定义在struct Protocol中，该结构定义在[protocol.h](https://github.com/brpc/brpc/blob/master/src/brpc/protocol.h)。其中的parse必须实现，除此之外server端至少要实现process_request，client端至少要实现serialize_request，pack_request，process_response。

实现协议回调还是比较困难的，这块的代码不会像供普通用户使用的那样，有较好的提示和保护，你得先靠自己搞清楚其他协议中的类似代码，然后再动手，最后发给我们做code review。

### parse

```c++
typedef ParseResult (*Parse)(butil::IOBuf* source, Socket *socket, bool read_eof, const void *arg);
```
用于把消息从source上切割下来，client端和server端使用同一个parse函数。返回的消息会被递给process_request(server端)或process_response(client端)。

参数：source是读取到的二进制内容，socket是对应的连接，read_eof为true表示连接已被对端关闭，arg在server端是对应server的指针，在client端是NULL。

ParseResult可能是错误，也可能包含一个切割下来的message，可能的值有：

- PARSE_ERROR_TRY_OTHERS ：不是这个协议，框架会尝试下一个协议。source不能被消费。
- PARSE_ERROR_NOT_ENOUGH_DATA : 到目前为止数据内容不违反协议，但不构成完整的消息。等到连接上有新数据时，新数据会被append入source并重新调用parse。如果不确定数据是否一定属于这个协议，source不应被消费，如果确定数据属于这个协议，也可以把source的内容转移到内部的状态中去。比如http协议解析中即使source不包含一个完整的http消息，它也会被http parser消费掉，以避免下一次重复解析。
- PARSE_ERROR_TOO_BIG_DATA : 消息太大，拒绝掉以保护server，连接会被关闭。
- PARSE_ERROR_NO_RESOURCE  : 内部错误，比如资源分配失败。连接会被关闭。
- PARSE_ERROR_ABSOLUTELY_WRONG  : 应该是这个协议（比如magic number匹配了），但是格式不符合预期。连接会被关闭。

### serialize_request
```c++
typedef bool (*SerializeRequest)(butil::IOBuf* request_buf,
                                 Controller* cntl,
                                 const google::protobuf::Message* request);
```
把request序列化进request_buf，client端必须实现。发生在pack_request之前，一次RPC中只会调用一次。cntl包含某些协议（比如http）需要的信息。成功返回true，否则false。

### pack_request
```c++
typedef int (*PackRequest)(butil::IOBuf* msg, 
                           uint64_t correlation_id,
                           const google::protobuf::MethodDescriptor* method,
                           Controller* controller,
                           const butil::IOBuf& request_buf,
                           const Authenticator* auth);
```
把request_buf打包入msg，每次向server发送消息前（包括重试）都会调用。当auth不为空时，需要打包认证信息。成功返回0，否则-1。

### process_request
```c++
typedef void (*ProcessRequest)(InputMessageBase* msg_base);
```
处理server端parse返回的消息，server端必须实现。可能会在和parse()不同的线程中运行。多个process_request可能同时运行。

在r34386后必须在处理结束时调用msg_base->Destroy()，为了防止漏调，考虑使用DestroyingPtr<>。

### process_response
```c++
typedef void (*ProcessResponse)(InputMessageBase* msg_base);
```
处理client端parse返回的消息，client端必须实现。可能会在和parse()不同的线程中运行。多个process_response可能同时运行。

在r34386后必须在处理结束时调用msg_base->Destroy()，为了防止漏调，考虑使用DestroyingPtr<>。

### verify
```c++
typedef bool (*Verify)(const InputMessageBase* msg);
```
处理连接的认证，只会对连接上的第一个消息调用，需要支持认证的server端必须实现，不需要认证或仅支持client端的协议可填NULL。成功返回true，否则false。

### parse_server_address
```c++
typedef bool (*ParseServerAddress)(butil::EndPoint* out, const char* server_addr_and_port);
```
把server_addr_and_port(Channel.Init的一个参数)转化为butil::EndPoint，可选。一些协议对server地址的表达和理解可能是不同的。

### get_method_name
```c++
typedef const std::string& (*GetMethodName)(const google::protobuf::MethodDescriptor* method,
                                            const Controller*);
```
定制method name，可选。

### supported_connection_type

标记支持的连接方式。如果支持所有连接方式，设为CONNECTION_TYPE_ALL。如果只支持连接池和短连接，设为CONNECTION_TYPE_POOLED_AND_SHORT。

### name

协议的名称，会出现在各种配置和显示中，越简短越好，必须是字符串常量。

## 注册到全局

实现好的协议要调用RegisterProtocol[注册到全局](https://github.com/brpc/brpc/blob/master/src/brpc/global.cpp)，以便brpc发现。就像这样：
```c++
Protocol http_protocol = { ParseHttpMessage,
                           SerializeHttpRequest, PackHttpRequest,
                           ProcessHttpRequest, ProcessHttpResponse,
                           VerifyHttpRequest, ParseHttpServerAddress,
                           GetHttpMethodName,
                           CONNECTION_TYPE_POOLED_AND_SHORT,
                           "http" };
if (RegisterProtocol(PROTOCOL_HTTP, http_protocol) != 0) {
    exit(1);
}
```


# ========= ./nshead_service.md ========
ub是百度内广泛使用的老RPC框架，在迁移ub服务时不可避免地需要[访问ub-server](ub_client.md)或被ub-client访问。ub使用的协议种类很多，但都以nshead作为二进制包的头部，这类服务在brpc中统称为**“nshead service”**。

nshead后大都使用mcpack/compack作为序列化格式，注意这不是“协议”。"协议"除了序列化格式，还涉及到各种特殊字段的定义，一种序列化格式可能会衍生出很多协议。ub没有定义标准协议，所以即使都使用mcpack或compack，产品线的通信协议也是五花八门，无法互通。鉴于此，我们提供了一套接口，让用户能够灵活的处理自己产品线的协议，同时享受brpc提供的builtin services等一系列框架福利。

# 使用ubrpc的服务

ubrpc协议的基本形式是nshead+compack或mcpack2，但compack或mcpack2中包含一些RPC过程需要的特殊字段。

在brpc r31687之后，用protobuf写的服务可以通过mcpack2pb被ubrpc client访问，步骤如下：

## 把idl文件转化为proto文件

使用脚本[idl2proto](https://github.com/brpc/brpc/blob/master/tools/idl2proto)把idl文件自动转化为proto文件，下面是转化后的proto文件。

```protobuf
// Converted from echo.idl by brpc/tools/idl2proto
import "idl_options.proto";
option (idl_support) = true;
option cc_generic_services = true;
message EchoRequest {
  required string message = 1; 
}
message EchoResponse {
  required string message = 1; 
}
 
// 对于有多个参数的idl方法，需要定义一个包含所有request或response的消息，作为对应方法的参数。
message MultiRequests {
  required EchoRequest req1 = 1;
  required EchoRequest req2 = 2;
}
message MultiResponses {
  required EchoRequest res1 = 1;
  required EchoRequest res2 = 2;
}
 
service EchoService {
  // 对应idl中的void Echo(EchoRequest req, out EchoResponse res);
  rpc Echo(EchoRequest) returns (EchoResponse);
 
  // 对应idl中的EchoWithMultiArgs(EchoRequest req1, EchoRequest req2, out EchoResponse res1, out EchoResponse res2);
  rpc EchoWithMultiArgs(MultiRequests) returns (MultiResponses);
}
```

原先的echo.idl文件如下：

```protobuf
struct EchoRequest {
    string message;
};
 
struct EchoResponse {
    string message;
};
 
service EchoService {
    void Echo(EchoRequest req, out EchoResponse res);
    uint32_t EchoWithMultiArgs(EchoRequest req1, EchoRequest req2, out EchoResponse res1, out EchoResponse res2);
};
```

## 以插件方式运行protoc

BRPC_PATH代表brpc产出的路径（包含bin include等目录），PROTOBUF_INCLUDE_PATH代表protobuf的包含路径。
注意--mcpack_out要和--cpp_out一致。

```shell
protoc --plugin=protoc-gen-mcpack=$BRPC_PATH/bin/protoc-gen-mcpack --cpp_out=. 
--mcpack_out=. --proto_path=$BRPC_PATH/include --proto_path=PROTOBUF_INCLUDE_PATH
```

## 实现生成的Service基类

```c++
class EchoServiceImpl : public EchoService {
public:
 
    ...
    // 对应idl中的void Echo(EchoRequest req, out EchoResponse res);
    virtual void Echo(google::protobuf::RpcController* cntl_base,
                      const EchoRequest* request,
                      EchoResponse* response,
                      google::protobuf::Closure* done) {
        brpc::ClosureGuard done_guard(done);
        brpc::Controller* cntl = static_cast<brpc::Controller*>(cntl_base);
 
        // 填充response。
        response->set_message(request->message());
 
        // 对应的idl方法没有返回值，不需要像下面方法中那样set_idl_result()。
        // 可以看到这个方法和其他protobuf服务没有差别，所以这个服务也可以被ubrpc之外的协议访问。
    }
 
    virtual void EchoWithMultiArgs(google::protobuf::RpcController* cntl_base,
                                   const MultiRequests* request,
                                   MultiResponses* response,
                                   google::protobuf::Closure* done) {
        brpc::ClosureGuard done_guard(done);
        brpc::Controller* cntl = static_cast<brpc::Controller*>(cntl_base);
 
        // 填充response。response是我们定义的包含所有idl response的消息。
        response->mutable_res1()->set_message(request->req1().message());
        response->mutable_res2()->set_message(request->req2().message());
 
        // 告诉RPC有多个request和response。
        cntl->set_idl_names(brpc::idl_multi_req_multi_res);
 
        // 对应idl方法的返回值。
        cntl->set_idl_result(17);
    }
};
```

## 设置ServerOptions.nshead_service

```c++
#include <brpc/ubrpc2pb_protocol.h>
...
brpc::ServerOptions option;
option.nshead_service = new brpc::policy::UbrpcCompackAdaptor; // mcpack2用UbrpcMcpack2Adaptor
```

例子见[example/echo_c++_ubrpc_compack](https://github.com/brpc/brpc/blob/master/example/echo_c++_ubrpc_compack/)。

# 使用nshead+blob的服务

[NsheadService](https://github.com/brpc/brpc/blob/master/src/brpc/nshead_service.h)是brpc中所有处理nshead打头协议的基类，实现好的NsheadService实例得赋值给ServerOptions.nshead_service才能发挥作用。不赋值的话，默认是NULL，代表不支持任何nshead开头的协议，这个server被nshead开头的数据包访问时会报错。明显地，**一个Server只能处理一种以nshead开头的协议。**

NsheadService的接口如下，基本上用户只需要实现`ProcessNsheadRequest`这个函数。

```c++
// 代表一个nshead请求或回复。
struct NsheadMessage {
    nshead_t head;
    butil::IOBuf body;
};
 
// 实现这个类并复制给ServerOptions.nshead_service来让brpc处理nshead请求。
class NsheadService : public Describable {
public:
    NsheadService();
    NsheadService(const NsheadServiceOptions&);
    virtual ~NsheadService();
 
    // 实现这个方法来处理nshead请求。注意这个方法可能在调用时controller->Failed()已经为true了。
    // 原因可能是Server.Stop()被调用正在退出(错误码是brpc::ELOGOFF)
    // 或触发了ServerOptions.max_concurrency(错误码是brpc::ELIMIT)
    // 在这种情况下，这个方法应该通过返回一个代表错误的response让客户端知道这些错误。
    // Parameters:
    //   server      The server receiving the request.
    //   controller  Contexts of the request.
    //   request     The nshead request received.
    //   response    The nshead response that you should fill in.
    //   done        You must call done->Run() to end the processing, brpc::ClosureGuard is preferred.
    virtual void ProcessNsheadRequest(const Server& server,
                                      Controller* controller,
                                      const NsheadMessage& request,
                                      NsheadMessage* response,
                                      NsheadClosure* done) = 0;
};
```

完整的example在[example/nshead_extension_c++](https://github.com/brpc/brpc/tree/master/example/nshead_extension_c++/)。

# 使用nshead+mcpack/compack/idl的服务

idl是mcpack/compack的前端，用户只要在idl文件中描述schema，就可以生成一些C++结构体，这些结构体可以打包为mcpack/compack。如果你的服务仍在大量地使用idl生成的结构体，且短期内难以修改，同时想要使用brpc提升性能和开发效率的话，可以实现[NsheadService](https://github.com/brpc/brpc/blob/master/src/brpc/nshead_service.h)，其接口接受nshead + 二进制包为request，用户填写自己的处理逻辑，最后的response也是nshead+二进制包。流程与protobuf方法保持一致，但过程中不涉及任何protobuf的序列化和反序列化，用户可以自由地理解nshead后的二进制包，包括用idl加载mcpack/compack数据包。

不过，你应当充分意识到这么改造的坏处：

> **这个服务在继续使用mcpack/compack作为序列化格式，相比protobuf占用成倍的带宽和打包时间。**

为了解决这个问题，我们提供了[mcpack2pb](mcpack2pb.md)，允许把protobuf作为mcpack/compack的前端。你只要写一份proto文件，就可以同时解析mcpack/compack和protobuf格式的请求。使用这个方法，使用idl描述的服务的可以平滑地改造为使用proto文件描述，而不用修改上游client（仍然使用mcpack/compack）。你产品线的服务可以逐个地从mcpack/compack/idl切换为protobuf，从而享受到性能提升，带宽节省，全新开发体验等好处。你可以自行在NsheadService使用src/mcpack2pb，也可以联系我们，提供更高质量的协议支持。

# 使用nshead+protobuf的服务

如果你的协议已经使用了nshead + protobuf，或者你想把你的协议适配为protobuf格式，那可以使用另一种模式：实现[NsheadPbServiceAdaptor](https://github.com/brpc/brpc/blob/master/src/brpc/nshead_pb_service_adaptor.h)（NsheadService的子类）。

工作步骤：

- Call ParseNsheadMeta() to understand the nshead header, user must tell RPC which pb method to call in the callback.
- Call ParseRequestFromIOBuf() to convert the body after nshead header to pb request, then call the pb method.
- When user calls server's done to end the RPC, SerializeResponseToIOBuf() is called to convert pb response to binary data that will be appended after nshead header and sent back to client.

这样做的好处是，这个服务还可以被其他使用protobuf的协议访问，比如baidu_std，hulu_pbrpc，sofa_pbrpc协议等等。NsheadPbServiceAdaptor的主要接口如下。完整的example在[这里](https://github.com/brpc/brpc/tree/master/example/nshead_pb_extension_c++/)。

```c++
class NsheadPbServiceAdaptor : public NsheadService {
public:
    NsheadPbServiceAdaptor() : NsheadService(
        NsheadServiceOptions(false, SendNsheadPbResponseSize)) {}
    virtual ~NsheadPbServiceAdaptor() {}
 
    // Fetch meta from `nshead_req' into `meta'.
    // Params:
    //   server: where the RPC runs.
    //   nshead_req: the nshead request that server received.
    //   controller: If something goes wrong, call controller->SetFailed()
    //   meta: Set meta information into this structure. `full_method_name'
    //         must be set if controller is not SetFailed()-ed
    // FIXME: server is not needed anymore, controller->server() is same
    virtual void ParseNsheadMeta(const Server& server,
                                 const NsheadMessage& nshead_req,
                                 Controller* controller,
                                 NsheadMeta* meta) const = 0;
    // Transform `nshead_req' to `pb_req'.
    // Params:
    //   meta: was set by ParseNsheadMeta()
    //   nshead_req: the nshead request that server received.
    //   controller: you can set attachment into the controller. If something
    //               goes wrong, call controller->SetFailed()
    //   pb_req: the pb request should be set by your implementation.
    virtual void ParseRequestFromIOBuf(const NsheadMeta& meta,
                                       const NsheadMessage& nshead_req,
                                       Controller* controller,
                                       google::protobuf::Message* pb_req) const = 0;
    // Transform `pb_res' (and controller) to `nshead_res'.
    // Params:
    //   meta: was set by ParseNsheadMeta()
    //   controller: If something goes wrong, call controller->SetFailed()
    //   pb_res: the pb response that returned by pb method. [NOTE] `pb_res'
    //           can be NULL or uninitialized when RPC failed (indicated by
    //           Controller::Failed()), in which case you may put error
    //           information into `nshead_res'.
    //   nshead_res: the nshead response that will be sent back to client.
    virtual void SerializeResponseToIOBuf(const NsheadMeta& meta,
                                          Controller* controller,
                                          const google::protobuf::Message* pb_res,
                                          NsheadMessage* nshead_res) const = 0;
};
```


# ========= ./overview.md ========
[English version](../en/overview.md)

# 什么是RPC?

互联网上的机器大都通过[TCP/IP协议](http://en.wikipedia.org/wiki/Internet_protocol_suite)相互访问，但TCP/IP只是往远端发送了一段二进制数据，为了建立服务还有很多问题需要抽象：

- 数据以什么格式传输？不同机器间，网络间可能是不同的字节序，直接传输内存数据显然是不合适的；随着业务变化，数据字段往往要增加或删减，怎么兼容前后不同版本的格式？
- 一个TCP连接可以被多个请求复用以减少开销么？多个请求可以同时发往一个TCP连接么?
- 如何管理和访问很多机器？
- 连接断开时应该干什么？
- 万一server不发送回复怎么办？
- ...

[RPC](http://en.wikipedia.org/wiki/Remote_procedure_call)可以解决这些问题，它把网络交互类比为“client访问server上的函数”：client向server发送request后开始等待，直到server收到、处理、回复client后，client又再度恢复并根据response做出反应。

![rpc.png](../images/rpc.png)

我们来看看上面的一些问题是如何解决的：

- 数据需要序列化，[protobuf](https://github.com/google/protobuf)在这方面做的不错。用户填写protobuf::Message类型的request，RPC结束后，从同为protobuf::Message类型的response中取出结果。protobuf有较好的前后兼容性，方便业务调整字段。http广泛使用[json](http://www.json.org/)作为序列化方法。
- 用户无需关心连接如何建立，但可以选择不同的[连接方式](client.md#连接方式)：短连接，连接池，单连接。
- 大量机器一般通过命名服务被发现，可基于[DNS](https://en.wikipedia.org/wiki/Domain_Name_System), [ZooKeeper](https://zookeeper.apache.org/), [etcd](https://github.com/coreos/etcd)等实现。在百度内，我们使用BNS (Baidu Naming Service)。brpc也提供["list://"和"file://"](client.md#命名服务)。用户可以指定负载均衡算法，让RPC每次选出一台机器发送请求，包括: round-robin, randomized, [consistent-hashing](consistent_hashing.md)(murmurhash3 or md5)和 [locality-aware](lalb.md).
- 连接断开时可以重试。
- 如果server没有在给定时间内回复，client会返回超时错误。

# 哪里可以使用RPC?

几乎所有的网络交互。

RPC不是万能的抽象，否则我们也不需要TCP/IP这一层了。但是在我们绝大部分的网络交互中，RPC既能解决问题，又能隔离更底层的网络问题。

对于RPC常见的质疑有：

- 我的数据非常大，用protobuf序列化太慢了。首先这可能是个伪命题，你得用[profiler](cpu_profiler.md)证明慢了才是真的慢，其次很多协议支持携带二进制数据以绕过序列化。
- 我传输的是流数据，RPC表达不了。事实上brpc中很多协议支持传递流式数据，包括[http中的ProgressiveReader](http_client.md#持续下载), h2的streams, [streaming rpc](streaming_rpc.md), 和专门的流式协议RTMP。
- 我的场景不需要回复。简单推理可知，你的场景中请求可丢可不丢，可处理也可不处理，因为client总是无法感知，你真的确认这是OK的？即使场景真的不需要，我们仍然建议用最小的结构体回复，因为这不大会是瓶颈，并且追查复杂bug时可能是很有价值的线索。

# 什么是![brpc](../images/logo.png)?

百度内最常使用的工业级RPC框架, 有1,000,000+个实例(不包含client)和上千种服务, 在百度内叫做"**baidu-rpc**". 目前只开源C++版本。

你可以使用它：

* 搭建能在**一个端口**支持多协议的服务, 或访问各种服务
  * restful http/https, [h2](https://http2.github.io/http2-spec)/[gRPC](https://grpc.io)。使用brpc的http实现比[libcurl](https://curl.haxx.se/libcurl/)方便多了。从其他语言通过HTTP/h2+json访问基于protobuf的协议.
  * [redis](redis_client.md)和[memcached](memcache_client.md), 线程安全，比官方client更方便。
  * [rtmp](https://github.com/brpc/brpc/blob/master/src/brpc/rtmp.h)/[flv](https://en.wikipedia.org/wiki/Flash_Video)/[hls](https://en.wikipedia.org/wiki/HTTP_Live_Streaming), 可用于搭建[流媒体服务](https://github.com/brpc/media-server).
  * hadoop_rpc(可能开源)
  * 支持[rdma](https://en.wikipedia.org/wiki/Remote_direct_memory_access)(即将开源)
  * 支持[thrift](thrift.md) , 线程安全，比官方client更方便
  * 各种百度内使用的协议: [baidu_std](baidu_std.md), [streaming_rpc](streaming_rpc.md), hulu_pbrpc, [sofa_pbrpc](https://github.com/baidu/sofa-pbrpc), nova_pbrpc, public_pbrpc, ubrpc和使用nshead的各种协议.
  * 基于工业级的[RAFT算法](https://raft.github.io)实现搭建[高可用](https://en.wikipedia.org/wiki/High_availability)分布式系统，已在[braft](https://github.com/brpc/braft)开源。
* Server能[同步](server.md)或[异步](server.md#异步service)处理请求。
* Client支持[同步](client.md#同步访问)、[异步](client.md#异步访问)、[半同步](client.md#半同步)，或使用[组合channels](combo_channel.md)简化复杂的分库或并发访问。
* [通过http界面](builtin_service.md)调试服务, 使用[cpu](cpu_profiler.md), [heap](heap_profiler.md), [contention](contention_profiler.md) profilers.
* 获得[更好的延时和吞吐](#更好的延时和吞吐).
* 把你组织中使用的协议快速地[加入brpc](new_protocol.md)，或定制各类组件, 包括[命名服务](load_balancing.md#命名服务) (dns, zk, etcd), [负载均衡](load_balancing.md#负载均衡) (rr, random, consistent hashing)

# brpc的优势

### 更友好的接口

只有三个(主要的)用户类: [Server](https://github.com/brpc/brpc/blob/master/src/brpc/server.h), [Channel](https://github.com/brpc/brpc/blob/master/src/brpc/channel.h), [Controller](https://github.com/brpc/brpc/blob/master/src/brpc/controller.h), 分别对应server端，client端，参数集合. 你不必推敲诸如"如何初始化XXXManager", "如何组合各种组件",  "XXXController的XXXContext间的关系是什么"。要做的很简单:

* 建服务? 包含[brpc/server.h](https://github.com/brpc/brpc/blob/master/src/brpc/server.h)并参考注释或[示例](https://github.com/brpc/brpc/blob/master/example/echo_c++/server.cpp).
* 访问服务? 包含[brpc/channel.h](https://github.com/brpc/brpc/blob/master/src/brpc/channel.h)并参考注释或[示例](https://github.com/brpc/brpc/blob/master/example/echo_c++/client.cpp).
* 调整参数? 看看[brpc/controller.h](https://github.com/brpc/brpc/blob/master/src/brpc/controller.h). 注意这个类是Server和Channel共用的，分成了三段，分别标记为Client-side, Server-side和Both-side methods。

我们尝试让事情变得更加简单，以命名服务为例，在其他RPC实现中，你也许需要复制一长段晦涩的代码才可使用，而在brpc中访问BNS可以这么写"bns://node-name"，DNS是`Init("http://domain-name", ...)`，本地文件列表是"file:///home/work/server.list"，相信不用解释，你也能明白这些代表什么。

### 使服务更加可靠

brpc在百度内被广泛使用:

* map-reduce服务和table存储
* 高性能计算和模型训练
* 各种索引和排序服务
* ….

它是一个经历过考验的实现。

brpc特别重视开发和维护效率, 你可以通过浏览器或curl[查看server内部状态](builtin_service.md), 分析在线服务的[cpu热点](cpu_profiler.md), [内存分配](heap_profiler.md)和[锁竞争](contention_profiler.md), 通过[bvar](bvar.md)统计各种指标并通过[/vars](vars.md)查看。

### 更好的延时和吞吐

虽然大部分RPC实现都声称“高性能”，但数字仅仅是数字，要在广泛的场景中做到高性能仍是困难的。为了统一百度内的通信架构，brpc在性能方面比其他RPC走得更深。

- 对不同客户端请求的读取和解析是完全并发的，用户也不用区分”IO线程“和”处理线程"。其他实现往往会区分“IO线程”和“处理线程”，并把[fd](http://en.wikipedia.org/wiki/File_descriptor)（对应一个客户端）散列到IO线程中去。当一个IO线程在读取其中的fd时，同一个线程中的fd都无法得到处理。当一些解析变慢时，比如特别大的protobuf message，同一个IO线程中的其他fd都遭殃了。虽然不同IO线程间的fd是并发的，但你不太可能开太多IO线程，因为这类线程的事情很少，大部分时候都是闲着的。如果有10个IO线程，一个fd能影响到的”其他fd“仍有相当大的比例（10个即10%，而工业级在线检索要求99.99%以上的可用性）。这个问题在fd没有均匀地分布在IO线程中，或在多租户(multi-tenacy)环境中会更加恶化。在brpc中，对不同fd的读取是完全并发的，对同一个fd中不同消息的解析也是并发的。解析一个特别大的protobuf message不会影响同一个客户端的其他消息，更不用提其他客户端的消息了。更多细节看[这里](io.md#收消息)。
- 对同一fd和不同fd的写出是高度并发的。当多个线程都要对一个fd写出时（常见于单连接），第一个线程会直接在原线程写出，其他线程会以[wait-free](http://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom)的方式托付自己的写请求，多个线程在高度竞争下仍可以在1秒内对同一个fd写入500万个16字节的消息。更多细节看[这里](io.md#发消息)。
- 尽量少的锁。高QPS服务可以充分利用一台机器的CPU。比如为处理请求[创建bthread](memory_management.md), [设置超时](timer_keeping.md), 根据回复[找到RPC上下文](bthread_id.md), [记录性能计数器](bvar.md)都是高度并发的。即使服务的QPS超过50万，用户也很少在[contention profiler](contention_profiler.md))中看到框架造成的锁竞争。
- 服务器线程数自动调节。传统的服务器需要根据下游延时的调整自身的线程数，否则吞吐可能会受影响。在brpc中，每个请求均运行在新建立的[bthread](bthread.md)中，请求结束后线程就结束了，所以天然会根据负载自动调节线程数。

brpc和其他实现的性能对比见[这里](benchmark.md)。


# ========= ./parallel_http.md ========
parallel_http能同时访问大量的http服务（几万个），适合在命令行中查询线上所有server的内置信息，供其他工具进一步过滤和聚合。curl很难做到这点，即使多个curl以后台的方式运行，并行度一般也只有百左右，访问几万台机器需要等待极长的时间。


# ========= ./redis_client.md ========
[English version](../en/redis_client.md)

[redis](http://redis.io/)是最近几年比较火的缓存服务，相比memcached在server端提供了更多的数据结构和操作方法，简化了用户的开发工作。为了使用户更快捷地访问redis并充分利用bthread的并发能力，brpc直接支持redis协议。示例程序：[example/redis_c++](https://github.com/brpc/brpc/tree/master/example/redis_c++/)

相比使用[hiredis](https://github.com/redis/hiredis)(官方client)的优势有：

- 线程安全。用户不需要为每个线程建立独立的client。
- 支持同步、异步、批量同步、批量异步等访问方式，能使用ParallelChannel等组合访问方式。
- 支持多种[连接方式](client.md#连接方式)。支持超时、backup request、取消、tracing、内置服务等一系列RPC基本福利。
- 一个进程中的所有brpc client和一个redis-server只有一个连接。多个线程同时访问一个redis-server时更高效（见[性能](#性能)）。无论reply的组成多复杂，内存都会连续成块地分配，并支持短串优化(SSO)进一步提高性能。

像http一样，brpc保证在最差情况下解析redis reply的时间复杂度也是O(N)，N是reply的字节数，而不是O($N^2$)。当reply是个较大的数组时，这是比较重要的。

加上[-redis_verbose](#查看发出的请求和收到的回复)后会打印出所有的redis request和response供调试。

# 访问单台redis

创建一个访问redis的Channel：

```c++
#include <brpc/redis.h>
#include <brpc/channel.h>
  
brpc::ChannelOptions options;
options.protocol = brpc::PROTOCOL_REDIS;
brpc::Channel redis_channel;
if (redis_channel.Init("0.0.0.0:6379", &options) != 0) {  // 6379是redis-server的默认端口
   LOG(ERROR) << "Fail to init channel to redis-server";
   return -1;
}
...
```

执行SET后再INCR：

```c++
std::string my_key = "my_key_1";
int my_number = 1;
...
// 执行"SET <my_key> <my_number>"
brpc::RedisRequest set_request;
brpc::RedisResponse response;
brpc::Controller cntl;
set_request.AddCommand("SET %s %d", my_key.c_str(), my_number);
redis_channel.CallMethod(NULL, &cntl, &set_request, &response, NULL/*done*/);
if (cntl.Failed()) {
    LOG(ERROR) << "Fail to access redis-server";
    return -1;
}
// 可以通过response.reply(i)访问某个reply
if (response.reply(0).is_error()) {
    LOG(ERROR) << "Fail to set";
    return -1;
}
// 可用多种方式打印reply
LOG(INFO) << response.reply(0).c_str()  // OK
          << response.reply(0)          // OK
          << response;                  // OK
...
 
// 执行"INCR <my_key>"
brpc::RedisRequest incr_request;
incr_request.AddCommand("INCR %s", my_key.c_str());
response.Clear();
cntl.Reset();
redis_channel.CallMethod(NULL, &cntl, &incr_request, &response, NULL/*done*/);
if (cntl.Failed()) {
    LOG(ERROR) << "Fail to access redis-server";
    return -1;
}
if (response.reply(0).is_error()) {
    LOG(ERROR) << "Fail to incr";
    return -1;
}
// 可用多种方式打印结果
LOG(INFO) << response.reply(0).integer()  // 2
          << response.reply(0)            // (integer) 2
          << response;                    // (integer) 2
```

批量执行incr或decr

```c++
brpc::RedisRequest request;
brpc::RedisResponse response;
brpc::Controller cntl;
request.AddCommand("INCR counter1");
request.AddCommand("DECR counter1");
request.AddCommand("INCRBY counter1 10");
request.AddCommand("DECRBY counter1 20");
redis_channel.CallMethod(NULL, &cntl, &request, &response, NULL/*done*/);
if (cntl.Failed()) {
    LOG(ERROR) << "Fail to access redis-server";
    return -1;
}
CHECK_EQ(4, response.reply_size());
for (int i = 0; i < 4; ++i) {
    CHECK(response.reply(i).is_integer());
    CHECK_EQ(brpc::REDIS_REPLY_INTEGER, response.reply(i).type());
}
CHECK_EQ(1, response.reply(0).integer());
CHECK_EQ(0, response.reply(1).integer());
CHECK_EQ(10, response.reply(2).integer());
CHECK_EQ(-10, response.reply(3).integer());
```

# RedisRequest

一个[RedisRequest](https://github.com/brpc/brpc/blob/master/src/brpc/redis.h)可包含多个Command，调用AddCommand*增加命令，成功返回true，失败返回false**并会打印调用处的栈**。

```c++
bool AddCommand(const char* fmt, ...);
bool AddCommandV(const char* fmt, va_list args);
bool AddCommandByComponents(const butil::StringPiece* components, size_t n);
```

格式和hiredis基本兼容：即%b对应二进制数据（指针+length)，其他和printf的参数类似。对一些细节做了改进：当某个字段包含空格时，使用单引号或双引号包围起来会被视作一个字段。比如AddCommand("Set 'a key with space' 'a value with space as well'")中的key是a key with space，value是a value with space as well。在hiredis中必须写成redisvCommand(..., "SET %s %s", "a key with space", "a value with space as well");

AddCommandByComponents类似hiredis中的redisCommandArgv，用户通过数组指定命令中的每一个部分。这个方法对AddCommand和AddCommandV可能发生的转义问题免疫，且效率最高。如果你在使用AddCommand和AddCommandV时出现了“Unmatched quote”，“无效格式”等问题且无法定位，可以试下这个方法。

如果AddCommand\*失败，后续的AddCommand\*和CallMethod都会失败。一般来说不用判AddCommand*的结果，失败后自然会通过RPC失败体现出来。

command_size()可获得（成功）加入的命令个数。

调用Clear()后可重用RedisRequest

# RedisResponse

[RedisResponse](https://github.com/brpc/brpc/blob/master/src/brpc/redis.h)可能包含一个或多个[RedisReply](https://github.com/brpc/brpc/blob/master/src/brpc/redis_reply.h)，reply_size()可获得reply的个数，reply(i)可获得第i个reply的引用（从0计数）。注意在hiredis中，如果请求包含了N个command，获取结果也要调用N次redisGetReply。但在brpc中这是不必要的，RedisResponse已经包含了N个reply，通过reply(i)获取就行了。只要RPC成功，response.reply_size()应与request.command_size()相等，除非redis-server有bug，redis-server工作的基本前提就是reply和command按序一一对应。

每个reply可能是：

- REDIS_REPLY_NIL：redis中的NULL，代表值不存在。可通过is_nil()判定。
- REDIS_REPLY_STATUS：在redis文档中称为Simple String。一般是操作的返回状态，比如SET返回的OK。可通过is_string()判定（和string相同），c_str()或data()获得值。
- REDIS_REPLY_STRING：在redis文档中称为Bulk String。大多数值都是这个类型，包括incr返回的。可通过is_string()判定，c_str()或data()获得值。
- REDIS_REPLY_ERROR：操作出错时的返回值，包含一段错误信息。可通过is_error()判定，error_message()获得错误信息。
- REDIS_REPLY_INTEGER：一个64位有符号数。可通过is_integer()判定，integer()获得值。
- REDIS_REPLY_ARRAY：另一些reply的数组。可通过is_array()判定，size()获得数组大小，[i]获得对应的子reply引用。

如果response包含三个reply，分别是integer，string和一个长度为2的array。那么可以分别这么获得值：response.reply(0).integer()，response.reply(1).c_str(), repsonse.reply(2)[0]和repsonse.reply(2)[1]。如果类型对不上，调用处的栈会被打印出来，并返回一个undefined的值。

response中的所有reply的ownership属于response。当response析构时，reply也析构了。

调用Clear()后RedisResponse可以重用。

# 访问redis集群

建立一个使用一致性哈希负载均衡算法(c_md5或c_murmurhash)的channel就能访问挂载在对应命名服务下的redis集群了。注意每个RedisRequest应只包含一个操作或确保所有的操作是同一个key。如果request包含了多个操作，在当前实现下这些操作总会送向同一个server，假如对应的key分布在多个server上，那么结果就不对了，这个情况下你必须把一个request分开为多个，每个包含一个操作。

或者你可以沿用常见的[twemproxy](https://github.com/twitter/twemproxy)方案。这个方案虽然需要额外部署proxy，还增加了延时，但client端仍可以像访问单点一样的访问它。

# 查看发出的请求和收到的回复

 打开[-redis_verbose](http://brpc.baidu.com:8765/flags/redis_verbose)即看到所有的redis request和response，注意这应该只用于线下调试，而不是线上程序。

打开[-redis_verbose_crlf2space](http://brpc.baidu.com:8765/flags/redis_verbose_crlf2space)可让打印内容中的CRLF (\r\n)变为空格，方便阅读。

| Name                     | Value | Description                              | Defined At                         |
| ------------------------ | ----- | ---------------------------------------- | ---------------------------------- |
| redis_verbose            | false | [DEBUG] Print EVERY redis request/response | src/brpc/policy/redis_protocol.cpp |
| redis_verbose_crlf2space | false | [DEBUG] Show \r\n as a space             | src/brpc/redis.cpp                 |

# 性能

redis版本：2.6.14

分别使用1，50，200个bthread同步压测同机redis-server，延时单位均为微秒。

```
$ ./client -use_bthread -thread_num 1
TRACE: 02-13 19:42:04:   * 0 client.cpp:180] Accessing redis server at qps=18668 latency=50
TRACE: 02-13 19:42:05:   * 0 client.cpp:180] Accessing redis server at qps=17043 latency=52
TRACE: 02-13 19:42:06:   * 0 client.cpp:180] Accessing redis server at qps=16520 latency=54

$ ./client -use_bthread -thread_num 50
TRACE: 02-13 19:42:54:   * 0 client.cpp:180] Accessing redis server at qps=301212 latency=164
TRACE: 02-13 19:42:55:   * 0 client.cpp:180] Accessing redis server at qps=301203 latency=164
TRACE: 02-13 19:42:56:   * 0 client.cpp:180] Accessing redis server at qps=302158 latency=164

$ ./client -use_bthread -thread_num 200
TRACE: 02-13 19:43:48:   * 0 client.cpp:180] Accessing redis server at qps=411669 latency=483
TRACE: 02-13 19:43:49:   * 0 client.cpp:180] Accessing redis server at qps=411679 latency=483
TRACE: 02-13 19:43:50:   * 0 client.cpp:180] Accessing redis server at qps=412583 latency=482
```

200个线程后qps基本到极限了。这里的极限qps比hiredis高很多，原因在于brpc默认以单链接访问redis-server，多个线程在写出时会[以wait-free的方式合并](io.md#发消息)，从而让redis-server就像被批量访问一样，每次都能从那个连接中读出一批请求，从而获得远高于非批量时的qps。下面通过连接池访问redis-server时qps的大幅回落是另外一个证明。

分别使用1，50，200个bthread一次发送10个同步压测同机redis-server，延时单位均为微秒。

```
$ ./client -use_bthread -thread_num 1 -batch 10  
TRACE: 02-13 19:46:45:   * 0 client.cpp:180] Accessing redis server at qps=15880 latency=59
TRACE: 02-13 19:46:46:   * 0 client.cpp:180] Accessing redis server at qps=16945 latency=57
TRACE: 02-13 19:46:47:   * 0 client.cpp:180] Accessing redis server at qps=16728 latency=57

$ ./client -use_bthread -thread_num 50 -batch 10
TRACE: 02-13 19:47:14:   * 0 client.cpp:180] Accessing redis server at qps=38082 latency=1307
TRACE: 02-13 19:47:15:   * 0 client.cpp:180] Accessing redis server at qps=38267 latency=1304
TRACE: 02-13 19:47:16:   * 0 client.cpp:180] Accessing redis server at qps=38070 latency=1305
 
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND 
16878 gejun     20   0 48136 2436 1004 R 93.8  0.0  12:48.56 redis-server   // thread_num=50

$ ./client -use_bthread -thread_num 200 -batch 10
TRACE: 02-13 19:49:09:   * 0 client.cpp:180] Accessing redis server at qps=29053 latency=6875
TRACE: 02-13 19:49:10:   * 0 client.cpp:180] Accessing redis server at qps=29163 latency=6855
TRACE: 02-13 19:49:11:   * 0 client.cpp:180] Accessing redis server at qps=29271 latency=6838
 
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND 
16878 gejun     20   0 48136 2508 1004 R 99.9  0.0  13:36.59 redis-server   // thread_num=200
```

注意redis-server实际处理的qps要乘10。乘10后也差不多在40万左右。另外在thread_num为50或200时，redis-server的CPU已打满。注意redis-server是[单线程reactor](threading_overview.md#单线程reactor)，一个核心打满就意味server到极限了。

使用50个bthread通过连接池方式同步压测同机redis-server。

```
$ ./client -use_bthread -connection_type pooled
TRACE: 02-13 18:07:40:   * 0 client.cpp:180] Accessing redis server at qps=75986 latency=654
TRACE: 02-13 18:07:41:   * 0 client.cpp:180] Accessing redis server at qps=75562 latency=655
TRACE: 02-13 18:07:42:   * 0 client.cpp:180] Accessing redis server at qps=75238 latency=657
 
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
16878 gejun     20   0 48136 2520 1004 R 99.9  0.0   9:52.33 redis-server
```

可以看到qps相比单链接时有大幅回落，同时redis-server的CPU打满了。原因在于redis-server每次只能从一个连接中读到一个请求，IO开销大幅增加。这也是单个hiredis client的极限性能。

# Command Line Interface

[example/redis_c++/redis_cli](https://github.com/brpc/brpc/blob/master/example/redis_c%2B%2B/redis_cli.cpp)是一个类似于官方CLI的命令行工具，以展示brpc对redis协议的处理能力。当使用brpc访问redis-server出现不符合预期的行为时，也可以使用这个CLI进行交互式的调试。

和官方CLI类似，`redis_cli <command>`也可以直接运行命令，-server参数可以指定redis-server的地址。

```
$ ./redis_cli 
     __          _     __
    / /_  ____ _(_)___/ /_  __      _________  _____
   / __ \/ __ `/ / __  / / / /_____/ ___/ __ \/ ___/
  / /_/ / /_/ / / /_/ / /_/ /_____/ /  / /_/ / /__  
 /_.___/\__,_/_/\__,_/\__,_/     /_/  / .___/\___/  
                                     /_/            
This command-line tool mimics the look-n-feel of official redis-cli, as a 
demostration of brpc's capability of talking to redis server. The 
output and behavior is not exactly same with the official one.
 
redis 127.0.0.1:6379> mset key1 foo key2 bar key3 17
OK
redis 127.0.0.1:6379> mget key1 key2 key3
["foo", "bar", "17"]
redis 127.0.0.1:6379> incrby key3 10
(integer) 27
redis 127.0.0.1:6379> client setname brpc-cli
OK
redis 127.0.0.1:6379> client getname
"brpc-cli"
```


# ========= ./rpc_press.md ========
rpc_press无需写代码就压测各种rpc server，目前支持的协议有：

- baidu_std
- hulu-pbrpc
- sofa-pbrpc
- public_pbrpc
- nova_pbrpc

# 获取工具

先按照[Getting Started](getting_started.md)编译好brpc，再去tools/rpc_press编译。

在CentOS 6.3上如果出现找不到libssl.so.4的错误，可执行`ln -s /usr/lib64/libssl.so.6 libssl.so.4临时解决`

# 发压力

rpc_press会动态加载proto文件，无需把proto文件编译为c++源文件。rpc_press会加载json格式的输入文件，转为pb请求，发向server，收到pb回复后如有需要会转为json并写入用户指定的文件。

rpc_press的所有的选项都来自命令行参数，而不是配置文件.

如下的命令向下游0.0.0.0:8002用baidu_std重复发送两个pb请求，分别转自'{"message":"hello"}和'{"message":"world"}，持续压力直到按ctrl-c，qps为100。

json也可以写在文件中，假如./input.json包含了上述两个请求，-input=./input.json也是可以的。

必需参数：

- -proto：指定相关的proto文件名。
- -method：指定方法名，形式必须是package.service.method。
- -server：当-lb_policy为空时，是服务器的ip:port；当-lb_policy不为空时，是集群地址，比如bns://node-name, file://server_list等等。具体见[命名服务](client.md#命名服务)。
- -input: 指定json请求或包含json请求的文件。r32157后json间不需要分隔符，r32157前json间用分号分隔。

可选参数:

- -inc: 包含被import的proto文件的路径。rpc_press默认支持import目录下的其他proto文件，但如果proto文件在其他目录，就要通过这个参数指定，多个路径用分号(;)分隔。
- -lb_policy: 指定负载均衡算法，默认为空，可选项为: rr random la c_murmurhash c_md5，具体见[负载均衡](client.md#负载均衡)。
- -timeout_ms: 设定超时,单位是毫秒(milliseconds),默认是1000(1秒)
- -max_retry: 最大的重试次数,默认是3, 一般无需修改. brpc的重试行为具体请见[这里](client.md#重试).
- -protocol: 连接server使用的协议，可选项见[协议](client.md#协议), 默认是baidu_std
- -connection_type: 连接方式，可选项为: single pooled short，具体见[连接方式](client.md#连接方式)。默认会根据协议自动选择,无需指定.
- -output: 如果不为空，response会转为json并写入这个文件，默认为空。
- -duration：大于0时表示发送这么多秒的压力后退出，否则一直发直到按ctrl-c或进程被杀死。默认是0（一直发送）。
- -qps：大于0时表示以这个压力发送，否则以最大速度(自适应)发送。默认是100。
- -dummy_port：修改dummy_server的端口，默认是8888

常用的参数组合：

- 向下游0.0.0.0:8002、用baidu_std重复发送./input.json中的所有请求，持续压力直到按ctrl-c，qps为100。
  ./rpc_press -proto=echo.proto -method=example.EchoService.Echo -server=0.0.0.0:8002 -input=./input.json -qps=100
- 以round-robin分流算法向bns://node-name代表的所有下游机器、用baidu_std重复发送两个pb请求，持续压力直到按ctrl-c，qps为100。
  ./rpc_press -proto=echo.proto -method=example.EchoService.Echo -server=bns://node-name -lb_policy=rr -input='{"message":"hello"} {"message":"world"}' -qps=100
- 向下游0.0.0.0:8002、用hulu协议重复发送两个pb请求，持续压力直到按ctrl-c，qps为100。
  ./rpc_press -proto=echo.proto -method=example.EchoService.Echo -server=0.0.0.0:8002 -protocol=hulu_pbrpc -input='{"message":"hello"} {"message":"world"}' -qps=100
- 向下游0.0.0.0:8002、用baidu_std重复发送两个pb请求，持续最大压力直到按ctrl-c。
  ./rpc_press -proto=echo.proto -method=example.EchoService.Echo -server=0.0.0.0:8002 -input='{"message":"hello"} {"message":"world"}' -qps=0
- 向下游0.0.0.0:8002、用baidu_std重复发送两个pb请求，持续最大压力10秒钟。
  ./rpc_press -proto=echo.proto -method=example.EchoService.Echo -server=0.0.0.0:8002 -input='{"message":"hello"} {"message":"world"}' -qps=0 -duration=10
- echo.proto中import了另一个目录下的proto文件
  ./rpc_press -proto=echo.proto -inc=<another-dir-with-the-imported-proto> -method=example.EchoService.Echo -server=0.0.0.0:8002 -input='{"message":"hello"} {"message":"world"}' -qps=0 -duration=10

rpc_press启动后会默认在8888端口启动一个dummy server，用于观察rpc_press本身的运行情况：

```
./rpc_press -proto=echo.proto -service=example.EchoService -method=Echo -server=0.0.0.0:8002 -input=./input.json -duration=0 -qps=100
TRACE: 01-30 16:10:04:   * 0 src/brpc/server.cpp:733] Server[dummy_servers] is serving on port=8888.
TRACE: 01-30 16:10:04:   * 0 src/brpc/server.cpp:742] Check out http://db-rpc-dev00.db01.baidu.com:8888 in your web browser.</code>
```

dummy_server启动时会在终端打印日志，一般按住ctrl点击那个链接可以直接打开对应的内置服务页面，就像这样：

![img](../images/rpc_press_1.png)

切换到vars页面，在Search框中输入rpc_press可以看到当前压力的延时分布情况:

![img](../images/rpc_press_2.png)

你可以通过-dummy_port参数修改dummy_server的端口，请确保端口可以在浏览器中访问。

如果你无法打开浏览器，命令行中也会定期打印信息：

```
2016/01/30-16:19:01     sent:101       success:101       error:0         total_error:0         total_sent:28379     
2016/01/30-16:19:02     sent:101       success:101       error:0         total_error:0         total_sent:28480     
2016/01/30-16:19:03     sent:101       success:101       error:0         total_error:0         total_sent:28581     
2016/01/30-16:19:04     sent:101       success:101       error:0         total_error:0         total_sent:28682     
2016/01/30-16:19:05     sent:101       success:101       error:0         total_error:0         total_sent:28783     
2016/01/30-16:19:06     sent:101       success:101       error:0         total_error:0         total_sent:28884     
2016/01/30-16:19:07     sent:101       success:101       error:0         total_error:0         total_sent:28985     
2016/01/30-16:19:08     sent:101       success:101       error:0         total_error:0         total_sent:29086     
2016/01/30-16:19:09     sent:101       success:101       error:0         total_error:0         total_sent:29187     
2016/01/30-16:19:10     sent:101       success:101       error:0         total_error:0         total_sent:29288     
[Latency]
  avg            122 us
  50%            122 us
  70%            135 us
  90%            161 us
  95%            164 us
  97%            166 us
  99%            172 us
  99.9%          199 us
  99.99%         199 us
  max            199 us
```

上方的字段含义应该是自解释的，在此略过。下方是延时信息，第一项"avg"是10秒内的平均延时，最后一项"max"是10秒内的最大延时，其余以百分号结尾的则代表延时分位值，即有左侧这么多比例的请求延时小于右侧的延时（单位微秒）。一般性能测试需要关注99%之后的长尾区域。

# FAQ

**Q: 如果下游是基于j-protobuf框架的服务模块，压力工具该如何配置？**

A：因为协议兼容性问题，启动rpc_press的时候需要带上-baidu_protocol_use_fullname=false


# ========= ./rpc_replay.md ========
r31658后，brpc能随机地把一部分请求写入一些文件中，并通过rpc_replay工具回放。目前支持的协议有：baidu_std, hulu_pbrpc, sofa_pbrpc。

# 获取工具

先按照[Getting Started](getting_started.md)编译好brpc，再去tools/rpc_replay编译。

在CentOS 6.3上如果出现找不到libssl.so.4的错误，可执行`ln -s /usr/lib64/libssl.so.6 libssl.so.4临时解决`

# 采样

brpc通过如下flags打开和控制如何保存请求，包含(R)后缀的flag都可以动态设置。

![img](../images/rpc_replay_1.png)

![img](../images/rpc_replay_2.png)

参数说明：

- -rpc_dump是主开关，关闭时其他以rpc_dump开头的flag都无效。当打开-rpc_dump后，brpc会以一定概率采集请求，如果服务的qps很高，brpc会调节采样比例，使得每秒钟采样的请求个数不超过-bvar_collector_expected_per_second对应的值。这个值在目前同样影响rpcz和contention profiler，一般不用改动，以后会对不同的应用独立开来。
- -rpc_dump_dir：设置存放被dump请求的目录
- -rpc_dump_max_files: 设置目录下的最大文件数，当超过限制时，老文件会被删除以腾出空间。
- -rpc_dump_max_requests_in_one_file：一个文件内的最大请求数，超过后写新文件。

brpc通过一个[bvar::Collector](https://github.com/brpc/brpc/blob/master/src/bvar/collector.h)来汇总来自不同线程的被采样请求，不同线程之间没有竞争，开销很小。

写出的内容依次存放在rpc_dump_dir目录下的多个文件内，这个目录默认在./rpc_dump_<app>，其中<app>是程序名。不同程序在同一个目录下同时采样时会写入不同的目录。如果程序启动时rpc_dump_dir已经存在了，目录将被清空。目录中的每个文件以requests.yyyymmdd_hhmmss_uuuuus命名，以保证按时间有序方便查找，比如：

![img](../images/rpc_replay_3.png)

目录下的文件数不超过rpc_dump_max_files，超过后最老的文件被删除从而给新文件腾出位置。

文件是二进制格式，格式与baidu_std协议的二进制格式类似，每个请求的binary layout如下：

```
"PRPC" (4 bytes magic string)
body_size(4 bytes)
meta_size(4 bytes)
RpcDumpMeta (meta_size bytes)
serialized request (body_size - meta_size bytes, including attachment)
```

请求间紧密排列。一个文件内的请求数不超过rpc_dump_max_requests_in_one_file。

> 一个文件可能包含多种协议的请求，如果server被多种协议访问的话。回放时被请求的server也将收到不同协议的请求。

brpc提供了[SampleIterator](https://github.com/brpc/brpc/blob/master/src/brpc/rpc_dump.h)从一个采样目录下的所有文件中依次读取所有的被采样请求，用户可根据需求把serialized request反序列化为protobuf请求，做一些二次开发。

```c++
#include <brpc/rpc_dump.h>
...
brpc::SampleIterator it("./rpc_data/rpc_dump/echo_server");         
for (SampleRequest* req = it->Next(); req != NULL; req = it->Next()) {
    ...                    
    // req->meta的类型是brpc::RpcDumpMeta，定义在src/brpc/rpc_dump.proto
    // req->request的类型是butil::IOBuf，对应格式说明中的"serialized request"
    // 使用结束后必须delete req。
}
```

# 回放

brpc在[tools/rpc_replay](https://github.com/brpc/brpc/tree/master/tools/rpc_replay/)提供了默认的回放工具。运行方式如下：

![img](../images/rpc_replay_4.png)

主要参数说明：

- -dir指定了存放采样文件的目录
- -times指定循环回放次数。其他参数请加上--help运行查看。
- -connection_type： 连接server的方式
- -dummy_port：修改dummy_server的端口
- -max_retry：最大重试次数，默认3次。
- -qps：大于0时限制qps，默认为0（不限制）
- -server：server的地址
- -thread_num：发送线程数，为0时会根据qps自动调节，默认为0。一般不用设置。
- -timeout_ms：超时
- -use_bthread：使用bthread发送，默认是。

rpc_replay会默认启动一个仅监控用的dummy server。打开后可查看回放的状况。其中rpc_replay_error是回放失败的次数。

![img](../images/rpc_replay_5.png)

如果你无法打开浏览器，命令行中也会定期打印信息：

```
2016/01/30-16:19:01     sent:101       success:101       error:0         total_error:0         total_sent:28379     
2016/01/30-16:19:02     sent:101       success:101       error:0         total_error:0         total_sent:28480     
2016/01/30-16:19:03     sent:101       success:101       error:0         total_error:0         total_sent:28581     
2016/01/30-16:19:04     sent:101       success:101       error:0         total_error:0         total_sent:28682     
2016/01/30-16:19:05     sent:101       success:101       error:0         total_error:0         total_sent:28783     
2016/01/30-16:19:06     sent:101       success:101       error:0         total_error:0         total_sent:28884     
2016/01/30-16:19:07     sent:101       success:101       error:0         total_error:0         total_sent:28985     
2016/01/30-16:19:08     sent:101       success:101       error:0         total_error:0         total_sent:29086     
2016/01/30-16:19:09     sent:101       success:101       error:0         total_error:0         total_sent:29187     
2016/01/30-16:19:10     sent:101       success:101       error:0         total_error:0         total_sent:29288     
[Latency]
  avg            122 us
  50%            122 us
  70%            135 us
  90%            161 us
  95%            164 us
  97%            166 us
  99%            172 us
  99.9%          199 us
  99.99%         199 us
  max            199 us
```

上方的字段含义应该是自解释的，在此略过。下方是延时信息，第一项"avg"是10秒内的平均延时，最后一项"max"是10秒内的最大延时，其余以百分号结尾的则代表延时分位值，即有左侧这么多比例的请求延时小于右侧的延时（单位微秒）。性能测试需要关注99%之后的长尾区域。


# ========= ./rpc_view.md ========
rpc_view可以转发端口被限的server的内置服务。像百度内如果一个服务的端口不在8000-8999，就只能在命令行下使用curl查看它的内置服务，没有历史趋势和动态曲线，也无法点击链接，排查问题不方便。rpc_view是一个特殊的http proxy：把对它的所有访问都转为对目标server的访问。只要把rpc_view的端口能在浏览器中被访问，我们就能通过它看到原本不能直接看到的server了。

# 获取工具

先按照[Getting Started](getting_started.md)编译好brpc，再去tools/rpc_view编译。

在CentOS 6.3上如果出现找不到libssl.so.4的错误，可执行`ln -s /usr/lib64/libssl.so.6 libssl.so.4临时解决`

# 访问目标server

确保你的机器能访问目标server，开发机应该都可以，一些测试机可能不行。运行./rpc_view <server-address>就可以了。

比如：

```
$ ./rpc_view 10.46.130.53:9970
TRACE: 02-14 12:12:20:   * 0 src/brpc/server.cpp:762] Server[rpc_view_server] is serving on port=8888.
TRACE: 02-14 12:12:20:   * 0 src/brpc/server.cpp:771] Check out http://db-rpc-dev00.db01.baidu.com:8888 in web browser.
```

打开rpc_view在8888端口提供的页面（在secureCRT中按住ctrl点url）：

![img](../images/rpc_view_1.png)

这个页面正是目标server的内置服务，右下角的提示告诉我们这是rpc_view提供的。这个页面和真实的内置服务基本是一样的，你可以做任何操作。

# 更换目标server

你可以随时停掉rpc_view并更换目标server，不过你觉得麻烦的话，也可以在浏览器上操作：给url加上?changetarget=<new-server-address>就行了。

假如我们之前停留在原目标server的/connections页面：

![img](../images/rpc_view_2.png)

加上?changetarge后就跳到新目标server的/connections页面了。接下来点击其他tab都会显示新目标server的。

![img](../images/rpc_view_3.png)


# ========= ./rpcz.md ========
用户能通过/rpcz看到最近请求的详细信息，并可以插入注释（annotation），不同于tracing system（如[dapper](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/36356.pdf)）以全局视角看到整体系统的延时分布，rpcz更多是一个调试工具，虽然角色有所不同，但在brpc中rpcz和tracing的数据来源是一样的。当每秒请求数小于1万时，rpcz会记录所有的请求，超过1万时，rpcz会随机忽略一些请求把采样数控制在1万左右。rpcz可以淘汰时间窗口之前的数据，通过-span_keeping_seconds选项设置，默认1小时。[一个长期运行的例子](http://brpc.baidu.com:8765/rpcz)。

关于开销：我们的实现完全规避了线程竞争，开销极小，在qps 30万的测试场景中，观察不到明显的性能变化，对大部分应用而言应该是“free”的。即使采集了几千万条请求，rpcz也不会增加很多内存，一般在50兆以内。rpcz会占用一些磁盘空间（就像日志一样），如果设定为存一个小时的数据，一般在几百兆左右。

## 开关方法

默认不开启，加入[-enable_rpcz](http://brpc.baidu.com:8765/flags/*rpcz*)选项会在启动后开启。

| Name                       | Value                | Description                              | Defined At                             |
| -------------------------- | -------------------- | ---------------------------------------- | -------------------------------------- |
| enable_rpcz (R)            | true (default:false) | Turn on rpcz                             | src/baidu/rpc/builtin/rpcz_service.cpp |
| rpcz_hex_log_id (R)        | false                | Show log_id in hexadecimal               | src/baidu/rpc/builtin/rpcz_service.cpp |
| rpcz_database_dir          | ./rpc_data/rpcz      | For storing requests/contexts collected by rpcz. | src/baidu/rpc/span.cpp                 |
| rpcz_keep_span_db          | false                | Don't remove DB of rpcz at program's exit | src/baidu/rpc/span.cpp                 |
| rpcz_keep_span_seconds (R) | 3600                 | Keep spans for at most so many seconds   | src/baidu/rpc/span.cpp                 |

若启动时未加-enable_rpcz，则可在启动后访问SERVER_URL/rpcz/enable动态开启rpcz，访问SERVER_URL/rpcz/disable则关闭，这两个链接等价于访问SERVER_URL/flags/enable_rpcz?setvalue=true和SERVER_URL/flags/enable_rpcz?setvalue=false。在r31010之后，rpc在html版本中增加了一个按钮可视化地开启和关闭。

![img](../images/rpcz_4.png)

![img](../images/rpcz_5.png)

如果只是brpc client或没有使用brpc，看[这里](dummy_server.md)。 

## 数据展现

/rpcz展现的数据分为两层。

### 第一层

看到最新请求的概况，点击链接进入第二层。

![img](../images/rpcz_6.png)

### 第二层

看到某系列(trace)或某个请求(span)的详细信息。一般通过点击链接进入，也可以把trace=和span=作为query-string拼出链接

![img](../images/rpcz_7.png)

内容说明：

- 时间分为了绝对时间（如2015/01/21-20:20:30.817392，小数点后精确到微秒）和前一个时间的差值（如.    19，代表19微秒)。
- trace=ID有点像“session id”，对应一个系统中完成一次对外服务牵涉到的所有服务，即上下游server都共用一个trace-id。span=ID对应一个server或client中一个请求的处理过程。trace-id和span-id在概率上唯一。
- 第一层页面中的request=和response=后的是数据包的字节数，包括附件但不包括协议meta。第二层中request和response的字节数一般在括号里，比如"Responded(13)"中的13。
- 点击链接可能会访问其他server上的rpcz，点浏览器后退一般会返回到之前的页面位置。
- I'm the last call, I'm about to ...都是用户的annotation。

## Annotation

只要你使用了brpc，就可以使用[TRACEPRINTF](https://github.com/brpc/brpc/blob/master/src/brpc/traceprintf.h)打印内容到事件流中，比如：

```c++
TRACEPRINTF("Hello rpcz %d", 123);
```

这条annotation会按其发生时间插入到对应请求的rpcz中。从这个角度看，rpcz是请求级的日志。如果你用TRACEPRINTF打印了沿路的上下文，便可看到请求在每个阶段停留的时间，牵涉到的数据集和参数。这是个很有用的功能。


# ========= ./server.md ========
[English version](../en/server.md)

# 示例程序

Echo的[server端代码](https://github.com/brpc/brpc/blob/master/example/echo_c++/server.cpp)。

# 填写proto文件

请求、回复、服务的接口均定义在proto文件中。

```C++
# 告诉protoc要生成C++ Service基类，如果是java或python，则应分别修改为java_generic_services和py_generic_services
option cc_generic_services = true;
 
message EchoRequest {
      required string message = 1;
};
message EchoResponse {
      required string message = 1;
};
 
service EchoService {
      rpc Echo(EchoRequest) returns (EchoResponse);
};
```

protobuf的更多用法请阅读[protobuf官方文档](https://developers.google.com/protocol-buffers/docs/proto#options)。

# 实现生成的Service接口

protoc运行后会生成echo.pb.cc和echo.pb.h文件，你得include echo.pb.h，实现其中的EchoService基类：

```c++
#include "echo.pb.h"
...
class MyEchoService : public EchoService  {
public:
    void Echo(::google::protobuf::RpcController* cntl_base,
              const ::example::EchoRequest* request,
              ::example::EchoResponse* response,
              ::google::protobuf::Closure* done) {
        // 这个对象确保在return时自动调用done->Run()
        brpc::ClosureGuard done_guard(done);
         
        brpc::Controller* cntl = static_cast<brpc::Controller*>(cntl_base);
 
        // 填写response
        response->set_message(request->message());
    }
};
```

Service在插入[brpc.Server](https://github.com/brpc/brpc/blob/master/src/brpc/server.h)后才可能提供服务。

当客户端发来请求时，Echo()会被调用。参数的含义分别是：

**controller**

在brpc中可以静态转为brpc::Controller（前提是代码运行brpc.Server中），包含了所有request和response之外的参数集合，具体接口查阅[controller.h](https://github.com/brpc/brpc/blob/master/src/brpc/controller.h)

**request**

请求，只读的，来自client端的数据包。

**response**

回复。需要用户填充，如果存在**required**字段没有被设置，该次调用会失败。

**done**

done由框架创建，递给服务回调，包含了调用服务回调后的后续动作，包括检查response正确性，序列化，打包，发送等逻辑。

**不管成功失败，done->Run()必须在请求处理完成后被用户调用一次。**

为什么框架不自己调用done->Run()？这是为了允许用户把done保存下来，在服务回调之后的某事件发生时再调用，即实现**异步Service**。

强烈建议使用**ClosureGuard**确保done->Run()被调用，即在服务回调开头的那句：

```c++
brpc::ClosureGuard done_guard(done);
```

不管在中间还是末尾脱离服务回调，都会使done_guard析构，其中会调用done->Run()。这个机制称为[RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)。没有这个的话你得在每次return前都加上done->Run()，**极易忘记**。

在异步Service中，退出服务回调时请求未处理完成，done->Run()不应被调用，done应被保存下来供以后调用，乍看起来，这里并不需要用ClosureGuard。但在实践中，异步Service照样会因各种原因跳出回调，如果不使用ClosureGuard，一些分支很可能会在return前忘记done->Run()，所以我们也建议在异步service中使用done_guard，与同步Service不同的是，为了避免正常脱离函数时done->Run()也被调用，你可以调用done_guard.release()来释放其中的done。

一般来说，同步Service和异步Service分别按如下代码处理done：

```c++
class MyFooService: public FooService  {
public:
    // 同步服务
    void SyncFoo(::google::protobuf::RpcController* cntl_base,
                 const ::example::EchoRequest* request,
                 ::example::EchoResponse* response,
                 ::google::protobuf::Closure* done) {
         brpc::ClosureGuard done_guard(done);
         ...
    }
 
    // 异步服务
    void AsyncFoo(::google::protobuf::RpcController* cntl_base,
                  const ::example::EchoRequest* request,
                  ::example::EchoResponse* response,
                  ::google::protobuf::Closure* done) {
         brpc::ClosureGuard done_guard(done);
         ...
         done_guard.release();
    }
};
```

ClosureGuard的接口如下：

```c++
// RAII: Call Run() of the closure on destruction.
class ClosureGuard {
public:
    ClosureGuard();
    // Constructed with a closure which will be Run() inside dtor.
    explicit ClosureGuard(google::protobuf::Closure* done);
    
    // Call Run() of internal closure if it's not NULL.
    ~ClosureGuard();
 
    // Call Run() of internal closure if it's not NULL and set it to `done'.
    void reset(google::protobuf::Closure* done);
 
    // Set internal closure to NULL and return the one before set.
    google::protobuf::Closure* release();
};
```

## 标记当前调用为失败

调用Controller.SetFailed()可以把当前调用设置为失败，当发送过程出现错误时，框架也会调用这个函数。用户一般是在服务的CallMethod里调用这个函数，比如某个处理环节出错，SetFailed()后确认done->Run()被调用了就可以跳出函数了(若使用了ClosureGuard，跳出函数时会自动调用done，不用手动)。Server端的done的逻辑主要是发送response回client，当其发现用户调用了SetFailed()后，会把错误信息送回client。client收到后，它的Controller::Failed()会为true（成功时为false），Controller::ErrorCode()和Controller::ErrorText()则分别是错误码和错误信息。

用户可以为http访问设置[status-code](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)，在server端一般是调用`controller.http_response().set_status_code()`，标准的status-code定义在[http_status_code.h](https://github.com/brpc/brpc/blob/master/src/brpc/http_status_code.h)中。Controller.SetFailed也会设置status-code，值是与错误码含义最接近的status-code，没有相关的则填500错误(brpc::HTTP_STATUS_INTERNAL_SERVER_ERROR)。如果你要覆盖status_code，设置代码一定要放在SetFailed()后，而不是之前。

## 获取Client的地址

`controller->remote_side()`可获得发送该请求的client地址和端口，类型是butil::EndPoint。如果client是nginx，remote_side()是nginx的地址。要获取真实client的地址，可以在nginx里设置`proxy_header ClientIp $remote_addr;`, 在rpc中通过`controller->http_request().GetHeader("ClientIp")`获得对应的值。

打印方式：

```c++
LOG(INFO) << "remote_side=" << cntl->remote_side(); 
printf("remote_side=%s\n", butil::endpoint2str(cntl->remote_side()).c_str());
```

## 获取Server的地址

controller->local_side()获得server端的地址，类型是butil::EndPoint。

打印方式：

```c++
LOG(INFO) << "local_side=" << cntl->local_side(); 
printf("local_side=%s\n", butil::endpoint2str(cntl->local_side()).c_str());
```

## 异步Service

即done->Run()在Service回调之外被调用。

有些server以等待后端服务返回结果为主，且处理时间特别长，为了及时地释放出线程资源，更好的办法是把done注册到被等待事件的回调中，等到事件发生后再调用done->Run()。

异步service的最后一行一般是done_guard.release()以确保正常退出CallMethod时不会调用done->Run()。例子请看[example/session_data_and_thread_local](https://github.com/brpc/brpc/tree/master/example/session_data_and_thread_local/)。

Service和Channel都可以使用done来表达后续的操作，但它们是**完全不同**的，请勿混淆：

* Service的done由框架创建，用户处理请求后调用done把response发回给client。
* Channel的done由用户创建，待RPC结束后被框架调用以执行用户的后续代码。

在一个会访问下游服务的异步服务中会同时接触两者，容易搞混，请注意区分。

# 加入Service

默认构造后的Server不包含任何服务，也不会对外提供服务，仅仅是一个对象。

通过如下方法插入你的Service实例。

```c++
int AddService(google::protobuf::Service* service, ServiceOwnership ownership);
```

若ownership参数为SERVER_OWNS_SERVICE，Server在析构时会一并删除Service，否则应设为SERVER_DOESNT_OWN_SERVICE。

插入MyEchoService代码如下：

```c++
brpc::Server server;
MyEchoService my_echo_service;
if (server.AddService(&my_echo_service, brpc::SERVER_DOESNT_OWN_SERVICE) != 0) {
    LOG(FATAL) << "Fail to add my_echo_service";
    return -1;
}
```

Server启动后你无法再修改其中的Service。

# 启动

调用以下[Server](https://github.com/brpc/brpc/blob/master/src/brpc/server.h)的一下接口启动服务。

```c++
int Start(const char* ip_and_port_str, const ServerOptions* opt);
int Start(EndPoint ip_and_port, const ServerOptions* opt);
int Start(int port, const ServerOptions* opt);
int Start(const char *ip_str, PortRange port_range, const ServerOptions *opt);  // r32009后增加
```

"localhost:9000", "cq01-cos-dev00.cq01:8000", “127.0.0.1:7000"都是合法的`ip_and_port_str`。

`options`为NULL时所有参数取默认值，如果你要使用非默认值，这么做就行了：

```c++
brpc::ServerOptions options;  // 包含了默认值
options.xxx = yyy;
...
server.Start(..., &options);
```

## 监听多个端口

一个server只能监听一个端口（不考虑ServerOptions.internal_port），需要监听N个端口就起N个Server。

# 停止

```c++
server.Stop(closewait_ms); // closewait_ms实际无效，出于历史原因未删
server.Join();
```

Stop()不会阻塞，Join()会。分成两个函数的原因在于当多个Server需要退出时，可以先全部Stop再一起Join，如果一个个Stop/Join，可能得花费Server个数倍的等待时间。

不管closewait_ms是什么值，server在退出时会等待所有正在被处理的请求完成，同时对新请求立刻回复ELOGOFF错误以防止新请求加入。这么做的原因在于只要server退出时仍有处理线程运行，就有访问到已释放内存的风险。如果你的server“退不掉”，很有可能是由于某个检索线程没结束或忘记调用done了。

当client看到ELOGOFF时，会跳过对应的server，并在其他server上重试对应的请求。所以在一般情况下brpc总是“优雅退出”的，重启或上线时几乎不会或只会丢失很少量的流量。

RunUntilAskedToQuit()函数可以在大部分情况下简化server的运转和停止代码。在server.Start后，只需如下代码即会让server运行直到按到Ctrl-C。

```c++
// Wait until Ctrl-C is pressed, then Stop() and Join() the server.
server.RunUntilAskedToQuit();
 
// server已经停止了，这里可以写释放资源的代码。
```

Join()完成后可以修改其中的Service，并重新Start。

# 被http/h2访问

使用Protobuf的服务通常可以通过http/h2+json访问，存于body的json串可与对应protobuf消息相互自动转化。

以[echo server](https://github.com/brpc/brpc/blob/master/example/echo_c%2B%2B/server.cpp)为例，你可以用[curl](https://curl.haxx.se/)访问这个服务。

```shell
# -H 'Content-Type: application/json' is optional
$ curl -d '{"message":"hello"}' http://brpc.baidu.com:8765/EchoService/Echo
{"message":"hello"}
```

注意：也可以指定`Content-Type: application/proto`用http/h2+protobuf二进制串访问服务，序列化性能更好。

## json<=>pb

json字段通过匹配的名字和结构与pb字段一一对应。json中一定要包含pb的required字段，否则转化会失败，对应请求会被拒绝。json中可以包含pb中没有定义的字段，但它们会被丢弃而不会存入pb的unknown字段。转化规则详见[json <=> protobuf](json2pb.md)。

开启选项-pb_enum_as_number后，pb中的enum会转化为它的数值而不是名字，比如在`enum MyEnum { Foo = 1; Bar = 2; };`中不开启此选项时MyEnum类型的字段会转化为"Foo"或"Bar"，开启后为1或2。此选项同时影响client发出的请求和server返回的回复。由于转化为名字相比数值有更好的前后兼容性，此选项只应用于兼容无法处理enum为名字的老代码。

## 兼容早期版本client

早期的brpc允许一个pb service被http协议访问时不填充pb请求，即使里面有required字段。一般来说这种service会自行解析http请求和设置http回复，并不会访问pb请求。但这也是非常危险的行为，毕竟这是pb service，但pb请求却是未定义的。

这种服务在升级到新版本rpc时会遇到障碍，因为brpc已不允许这种行为。为了帮助这种服务升级，brpc允许经过一些设置后不把http body自动转化为pb request(从而可自行处理），方法如下：

```c++
brpc::ServiceOptions svc_opt;
svc_opt.ownership = ...;
svc_opt.restful_mappings = ...;
svc_opt.allow_http_body_to_pb = false; //关闭http/h2 body至pb request的自动转化
server.AddService(service, svc_opt);
```

如此设置后service收到http/h2请求后不会尝试把body转化为pb请求，所以pb请求总是未定义状态，用户得在`cntl->request_protocol() == brpc::PROTOCOL_HTTP || cntl->request_protocol() == brpc::PROTOCOL_H2`成立时自行解析body。

相应地，当cntl->response_attachment()不为空且pb回复不为空时，框架不再报错，而是直接把cntl->response_attachment()作为回复的body。这个功能和设置allow_http_body_to_pb与否无关。如果放开自由度导致过多的用户犯错，可能会有进一步的调整。

# 协议支持

server端会自动尝试其支持的协议，无需用户指定。`cntl->protocol()`可获得当前协议。server能从一个listen端口建立不同协议的连接，不需要为不同的协议使用不同的listen端口，一个连接上也可以传输多种协议的数据包, 但一般不会这么做(也不建议)，支持的协议有：

- [百度标准协议](baidu_std.md)，显示为"baidu_std"，默认启用。

- [流式RPC协议](streaming_rpc.md)，显示为"streaming_rpc", 默认启用。

- http/1.0和http/1.1协议，显示为”http“，默认启用。

- http/2和gRPC协议，显示为"h2c"(未加密)或"h2"(加密)，默认启用。

- RTMP协议，显示为"rtmp", 默认启用。

- hulu-pbrpc的协议，显示为"hulu_pbrpc"，默认启动。

- sofa-pbrpc的协议，显示为”sofa_pbrpc“, 默认启用。

- 百盟的协议，显示为”nova_pbrpc“, 默认不启用，开启方式：

  ```c++
  #include <brpc/policy/nova_pbrpc_protocol.h>
  ...
  ServerOptions options;
  ...
  options.nshead_service = new brpc::policy::NovaServiceAdaptor;
  ```

- public_pbrpc协议，显示为"public_pbrpc"，默认不启用，开启方式：

  ```c++
  #include <brpc/policy/public_pbrpc_protocol.h>
  ...
  ServerOptions options;
  ...
  options.nshead_service = new brpc::policy::PublicPbrpcServiceAdaptor;
  ```

- nshead+mcpack协议，显示为"nshead_mcpack"，默认不启用，开启方式：

  ```c++
  #include <brpc/policy/nshead_mcpack_protocol.h>
  ...
  ServerOptions options;
  ...
  options.nshead_service = new brpc::policy::NsheadMcpackAdaptor;
  ```

  顾名思义，这个协议的数据包由nshead+mcpack构成，mcpack中不包含特殊字段。不同于用户基于NsheadService的实现，这个协议使用了mcpack2pb，使得一份代码可以同时处理mcpack和pb两种格式。由于没有传递ErrorText的字段，当发生错误时server只能关闭连接。

- 和UB相关的协议请阅读[实现NsheadService](nshead_service.md)。

如果你有更多的协议需求，可以联系我们。

# 设置

## 版本

Server.set_version(...)可以为server设置一个名称+版本，可通过/version内置服务访问到。虽然叫做"version“，但设置的值请包含服务名，而不仅仅是一个数字版本。

## 关闭闲置连接

如果一个连接在ServerOptions.idle_timeout_sec对应的时间内没有读取或写出数据，则被视为”闲置”而被server主动关闭。默认值为-1，代表不开启。

打开[-log_idle_connection_close](http://brpc.baidu.com:8765/flags/log_idle_connection_close)后关闭前会打印一条日志。

| Name                      | Value | Description                              | Defined At          |
| ------------------------- | ----- | ---------------------------------------- | ------------------- |
| log_idle_connection_close | false | Print log when an idle connection is closed | src/brpc/socket.cpp |

## pid_file

如果设置了此字段，Server启动时会创建一个同名文件，内容为进程号。默认为空。

## 在每条日志后打印hostname

此功能只对[butil/logging.h](https://github.com/brpc/brpc/blob/master/src/butil/logging.h)中的日志宏有效。

打开[-log_hostname](http://brpc.baidu.com:8765/flags/log_hostname)后每条日志后都会带本机名称，如果所有的日志需要汇总到一起进行分析，这个功能可以帮助你了解某条日志来自哪台机器。

## 打印FATAL日志后退出程序

此功能只对[butil/logging.h](https://github.com/brpc/brpc/blob/master/src/butil/logging.h)中的日志宏有效，glog默认在FATAL日志时crash。

打开[-crash_on_fatal_log](http://brpc.baidu.com:8765/flags/crash_on_fatal_log)后如果程序使用LOG(FATAL)打印了异常日志或违反了CHECK宏中的断言，那么程序会在打印日志后abort，这一般也会产生coredump文件，默认不打开。这个开关可在对程序的压力测试中打开，以确认程序没有进入过严重错误的分支。

> 一般的惯例是，ERROR表示可容忍的错误，FATAL代表不可逆转的错误。

## 最低日志级别

此功能由[butil/logging.h](https://github.com/brpc/brpc/blob/master/src/butil/logging.h)和glog各自实现，为同名选项。

只有**不低于**-minloglevel指定的日志级别的日志才会被打印。这个选项可以动态修改。设置值和日志级别的对应关系：0=INFO 1=NOTICE 2=WARNING 3=ERROR 4=FATAL，默认为0。

未打印日志的开销只是一次if判断，也不会评估参数(比如某个参数调用了函数，日志不打，这个函数就不会被调用）。如果日志最终打印到自定义LogSink，那么还要经过LogSink的过滤。

## 归还空闲内存至系统

选项-free_memory_to_system_interval表示每过这么多秒就尝试向系统归还空闲内存，<= 0表示不开启，默认值为0，若开启建议设为10及以上的值。此功能支持tcmalloc，之前程序中对`MallocExtension::instance()->ReleaseFreeMemory()`的定期调用可改成设置此选项。

## 打印发送给client的错误

server的框架部分一般不针对个别client打印错误日志，因为当大量client出现错误时，可能导致server高频打印日志而严重影响性能。但有时为了调试问题，或就是需要让server打印错误，打开参数[-log_error_text](http://brpc.baidu.com:8765/flags/log_error_text)即可。

## 定制延时的分位值

显示的服务延时分位值**默认**为**80** (曾经为50), 90, 99, 99.9, 99.99，前三项可分别通过-bvar_latency_p1, -bvar_latency_p2, -bvar_latency_p3三个gflags定制。

以下是正确的设置：
```shell
-bvar_latency_p3=97   # p3从默认99修改为97
-bvar_latency_p1=60 -bvar_latency_p2=80 -bvar_latency_p3=95
```
以下是错误的设置：
```shell
-bvar_latency_p3=100   # 设置值必须在[1,99]闭区间内，gflags解析会失败
-bvar_latency_p1=-1    # 同上
```

## 设置栈大小

brpc的Server是运行在bthread之上，默认栈大小为1MB，而pthread默认栈大小为10MB，所以在pthread上正常运行的程序，在bthread上可能遇到栈不足。

可设置如下的gflag以调整栈的大小:
```shell
--stack_size_normal=10000000    # 表示调整栈大小为10M左右
--tc_stack_normal=1             # 默认为8，表示每个worker缓存的栈的个数(以加快分配速度)，size越大，缓存数目可以适当调小(以减少内存占用)
```
注意：不是说程序coredump就意味着”栈不够大“，只是因为这个试起来最容易，所以优先排除掉可能性。事实上百度内如此多的应用也很少碰到栈不够大的情况。

## 限制最大消息

为了保护server和client，当server收到的request或client收到的response过大时，server或client会拒收并关闭连接。此最大尺寸由[-max_body_size](http://brpc.baidu.com:8765/flags/max_body_size)控制，单位为字节。

超过最大消息时会打印如下错误日志：

```
FATAL: 05-10 14:40:05: * 0 src/brpc/input_messenger.cpp:89] A message from 127.0.0.1:35217(protocol=baidu_std) is bigger than 67108864 bytes, the connection will be closed. Set max_body_size to allow bigger messages
```

protobuf中有[类似的限制](https://github.com/google/protobuf/blob/master/src/google/protobuf/io/coded_stream.h#L364)，出错时会打印如下日志：

```
FATAL: 05-10 13:35:02: * 0 google/protobuf/io/coded_stream.cc:156] A protocol message was rejected because it was too big (more than 67108864 bytes). To increase the limit (or to disable these warnings), see CodedInputStream::SetTotalBytesLimit() in google/protobuf/io/coded_stream.h.
```

brpc移除了protobuf中的限制，全交由此选项控制，只要-max_body_size足够大，用户就不会看到错误日志。此功能对protobuf的版本没有要求。

## 压缩

set_response_compress_type()设置response的压缩方式，默认不压缩。

注意附件不会被压缩。HTTP body的压缩方法见[这里](http_service.md#压缩response-body)。

支持的压缩方法有：

- brpc::CompressTypeSnappy : [snanpy压缩](http://google.github.io/snappy/)，压缩和解压显著快于其他压缩方法，但压缩率最低。
- brpc::CompressTypeGzip : [gzip压缩](http://en.wikipedia.org/wiki/Gzip)，显著慢于snappy，但压缩率高
- brpc::CompressTypeZlib : [zlib压缩](http://en.wikipedia.org/wiki/Zlib)，比gzip快10%~20%，压缩率略好于gzip，但速度仍明显慢于snappy。

更具体的性能对比见[Client-压缩](client.md#压缩).

## 附件

baidu_std和hulu_pbrpc协议支持传递附件，这段数据由用户自定义，不经过protobuf的序列化。站在server的角度，设置在Controller.response_attachment()的附件会被client端收到，Controller.request_attachment()则包含了client端送来的附件。

附件不会被框架压缩。

在http协议中，附件对应[message body](http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html)，比如要返回的数据就设置在response_attachment()中。

## 开启SSL

要开启SSL，首先确保代码依赖了最新的openssl库。如果openssl版本很旧，会有严重的安全漏洞，支持的加密算法也少，违背了开启SSL的初衷。然后设置`ServerOptions.ssl_options`，具体见[ssl_options.h](https://github.com/brpc/brpc/blob/master/src/brpc/ssl_options.h)。

```c++
// Certificate structure
struct CertInfo {
    // Certificate in PEM format.
    // Note that CN and alt subjects will be extracted from the certificate,
    // and will be used as hostnames. Requests to this hostname (provided SNI
    // extension supported) will be encrypted using this certifcate.
    // Supported both file path and raw string
    std::string certificate;

    // Private key in PEM format.
    // Supported both file path and raw string based on prefix:
    std::string private_key;

    // Additional hostnames besides those inside the certificate. Wildcards
    // are supported but it can only appear once at the beginning (i.e. *.xxx.com).
    std::vector<std::string> sni_filters;
};

// SSL options at server side
struct ServerSSLOptions {
    // Default certificate which will be loaded into server. Requests
    // without hostname or whose hostname doesn't have a corresponding
    // certificate will use this certificate. MUST be set to enable SSL.
    CertInfo default_cert;
    
    // Additional certificates which will be loaded into server. These
    // provide extra bindings between hostnames and certificates so that
    // we can choose different certificates according to different hostnames.
    // See `CertInfo' for detail.
    std::vector<CertInfo> certs;
    
    // When set, requests without hostname or whose hostname can't be found in
    // any of the cerficates above will be dropped. Otherwise, `default_cert'
    // will be used.
    // Default: false
    bool strict_sni;
 
    // ... Other options
};
```

- Server端开启SSL**必须**要设置一张默认证书`default_cert`（默认SSL连接都用此证书），如果希望server能支持动态选择证书（如根据请求中域名，见[SNI](https://en.wikipedia.org/wiki/Server_Name_Indication)机制），则可以将这些证书加载到`certs`。最后用户还可以在Server运行时，动态增减这些动态证书：

  ```c++
  int AddCertificate(const CertInfo& cert);
  int RemoveCertificate(const CertInfo& cert);
  int ResetCertificates(const std::vector<CertInfo>& certs);
  ```

- 其余选项还包括：密钥套件选择（推荐密钥ECDHE-RSA-AES256-GCM-SHA384，chrome默认第一优先密钥，安全性很高，但比较耗性能）、session复用等。

- SSL层在协议层之下（作用在Socket层），即开启后，所有协议（如HTTP）都支持用SSL加密后传输到Server，Server端会先进行SSL解密后，再把原始数据送到各个协议中去。

- SSL开启后，端口仍然支持非SSL的连接访问，Server会自动判断哪些是SSL，哪些不是。如果要屏蔽非SSL访问，用户可通过`Controller::is_ssl()`判断是否是SSL，同时在[connections](connections.md)内置监控上也可以看到连接的SSL信息。

## 验证client身份

如果server端要开启验证功能，需要实现`Authenticator`中的接口:

```c++
class Authenticator {
public:
    // Implement this method to verify credential information `auth_str' from
    // `client_addr'. You can fill credential context (result) into `*out_ctx'
    // and later fetch this pointer from `Controller'.
    // Returns 0 on success, error code otherwise
    virtual int VerifyCredential(const std::string& auth_str,
                                 const base::EndPoint& client_addr,
                                 AuthContext* out_ctx) const = 0;
    }; 

class AuthContext {
public:
    const std::string& user() const;
    const std::string& group() const;
    const std::string& roles() const;
    const std::string& starter() const;
    bool is_service() const;
};
```

server的验证是基于连接的。当server收到连接上的第一个请求时，会尝试解析出其中的身份信息部分（如baidu_std里的auth字段、HTTP协议里的Authorization头），然后附带client地址信息一起调用`VerifyCredential`。若返回0，表示验证成功，用户可以把验证后的信息填入`AuthContext`，后续可通过`controller->auth_context()`获取，用户不需要关心其分配和释放。否则表示验证失败，连接会被直接关闭，client访问失败。

后续请求默认通过验证么，没有认证开销。

把实现的`Authenticator`实例赋值到`ServerOptions.auth`，即开启验证功能，需要保证该实例在整个server运行周期内都有效，不能被析构。

## worker线程数

设置ServerOptions.num_threads即可，默认是cpu core的个数（包含超线程的）。

注意: ServerOptions.num_threads仅仅是个**提示**。

你不能认为Server就用了这么多线程，因为进程内的所有Server和Channel会共享线程资源，线程总数是所有ServerOptions.num_threads和-bthread_concurrency中的最大值。比如一个程序内有两个Server，num_threads分别为24和36，bthread_concurrency为16。那么worker线程数为max(24, 36, 16) = 36。这不同于其他RPC实现中往往是加起来。

Channel没有相应的选项，但可以通过选项-bthread_concurrency调整。

另外，brpc**不区分IO线程和处理线程**。brpc知道如何编排IO和处理代码，以获得更高的并发度和线程利用率。

## 限制最大并发

“并发”可能有两种含义，一种是连接数，一种是同时在处理的请求数。这里提到的是后者。

在传统的同步server中，最大并发不会超过工作线程数，设定工作线程数量一般也限制了并发。但brpc的请求运行于bthread中，M个bthread会映射至N个worker中（一般M大于N），所以同步server的并发度可能超过worker数量。另一方面，虽然异步server的并发不受线程数控制，但有时也需要根据其他因素控制并发量。

brpc支持设置server级和method级的最大并发，当server或method同时处理的请求数超过并发度限制时，它会立刻给client回复**brpc::ELIMIT**错误，而不会调用服务回调。看到ELIMIT错误的client应重试另一个server。这个选项可以防止server出现过度排队，或用于限制server占用的资源。

默认不开启。

### 为什么超过最大并发要立刻给client返回错误而不是排队？

当前server达到最大并发并不意味着集群中的其他server也达到最大并发了，立刻让client获知错误，并去尝试另一台server在全局角度是更好的策略。

### 为什么不限制QPS?

QPS是一个秒级的指标，无法很好地控制瞬间的流量爆发。而最大并发和当前可用的重要资源紧密相关："工作线程"，“槽位”等，能更好地抑制排队。

另外当server的延时较为稳定时，限制并发的效果和限制QPS是等价的。但前者实现起来容易多了：只需加减一个代表并发度的计数器。这也是大部分流控都限制并发而不是QPS的原因，比如TCP中的“窗口"即是一种并发度。

### 计算最大并发数

最大并发度 = 极限QPS * 低负载延时  ([little's law](https://en.wikipedia.org/wiki/Little%27s_law))

极限QPS指的是server能达到的最大qps，低负载延时指的是server在没有严重积压请求的前提下时的平均延时。一般的服务上线都会有性能压测，把测得的QPS和延时相乘一般就是该服务的最大并发度。

### 限制server级别并发度

设置ServerOptions.max_concurrency，默认值0代表不限制。访问内置服务不受此选项限制。

Server.ResetMaxConcurrency()可在server启动后动态修改server级别的max_concurrency。

### 限制method级别并发度

server.MaxConcurrencyOf("...") = ...可设置method级别的max_concurrency。也可以通过设置ServerOptions.method_max_concurrency一次性为所有的method设置最大并发。
当ServerOptions.method_max_concurrency和server.MaxConcurrencyOf("...")=...同时被设置时，使用server.MaxConcurrencyOf()所设置的值。

```c++
ServerOptions.method_max_concurrency = 20;                   // Set the default maximum concurrency for all methods
server.MaxConcurrencyOf("example.EchoService.Echo") = 10;    // Give priority to the value set by server.MaxConcurrencyOf()
server.MaxConcurrencyOf("example.EchoService", "Echo") = 10;
server.MaxConcurrencyOf(&service, "Echo") = 10;
server.MaxConcurrencyOf("example.EchoService.Echo") = "10";  // You can also assign a string value
```

此设置一般**发生在AddService后，server启动前**。当设置失败时（比如对应的method不存在），server会启动失败同时提示用户修正MaxConcurrencyOf设置错误。

当method级别和server级别的max_concurrency都被设置时，先检查server级别的，再检查method级别的。

注意：没有service级别的max_concurrency。

### 使用自适应限流算法
实际生产环境中,最大并发未必一成不变，在每次上线前逐个压测和设置服务的最大并发也很繁琐。这个时候可以使用自适应限流算法。

自适应限流是method级别的。要使用自适应限流算法，把method的最大并发度设置为"auto"即可:

```c++
// Set auto concurrency limiter for all methods
brpc::ServerOptions options;
options.method_max_concurrency = "auto";

// Set auto concurrency limiter for specific method
server.MaxConcurrencyOf("example.EchoService.Echo") = "auto";
```
关于自适应限流的更多细节可以看[这里](auto_concurrency_limiter.md)

## pthread模式

用户代码（客户端的done，服务器端的CallMethod）默认在栈为1MB的bthread中运行。但有些用户代码无法在bthread中运行，比如：

- JNI会检查stack layout而无法在bthread中运行。
- 代码中广泛地使用pthread local传递session级别全局数据，在RPC前后均使用了相同的pthread local的数据，且数据有前后依赖性。比如在RPC前往pthread-local保存了一个值，RPC后又读出来希望和之前保存的相等，就会有问题。而像tcmalloc虽然也使用了pthread/LWP local，但每次使用之间没有直接的依赖，是安全的。

对于这些情况，brpc提供了pthread模式，开启**-usercode_in_pthread**后，用户代码均会在pthread中运行，原先阻塞bthread的函数转而阻塞pthread。

打开pthread模式后在性能上的注意点：

- 同步RPC都会阻塞worker pthread，server端一般需要设置更多的工作线程(ServerOptions.num_threads)，调度效率会略微降低。
- 运行用户代码的仍然是bthread，只是很特殊，会直接使用pthread worker的栈。这些特殊bthread的调度方式和其他bthread是一致的，这方面性能差异很小。
- bthread支持一个独特的功能：把当前使用的pthread worker 让给另一个新创建的bthread运行，以消除一次上下文切换。brpc client利用了这点，从而使一次RPC过程中3次上下文切换变为了2次。在高QPS系统中，消除上下文切换可以明显改善性能和延时分布。但pthread模式不具备这个能力，在高QPS系统中性能会有一定下降。
- pthread模式中线程资源是硬限，一旦线程被打满，请求就会迅速拥塞而造成大量超时。一个常见的例子是：下游服务大量超时后，上游服务可能由于线程大都在等待下游也被打满从而影响性能。开启pthread模式后请考虑设置ServerOptions.max_concurrency以控制server的最大并发。而在bthread模式中bthread个数是软限，对此类问题的反应会更加平滑。

pthread模式可以让一些老代码快速尝试brpc，但我们仍然建议逐渐地把代码改造为使用bthread local或最好不用TLS，从而最终能关闭这个开关。

## 安全模式

如果你的服务流量来自外部（包括经过nginx等转发），你需要注意一些安全因素：

### 对外隐藏内置服务

内置服务很有用，但包含了大量内部信息，不应对外暴露。有多种方式可以对外隐藏内置服务：

- 设置内部端口。把ServerOptions.internal_port设为一个**仅允许内网访问**的端口。你可通过internal_port访问到内置服务，但通过对外端口(Server.Start时传入的那个)访问内置服务时将看到如下错误：

  ```
  [a27eda84bcdeef529a76f22872b78305] Not allowed to access builtin services, try ServerOptions.internal_port=... instead if you're inside internal network
  ```

- http proxy指定转发路径。nginx等可配置URL的映射关系，比如下面的配置把访问/MyAPI的外部流量映射到`target-server`的`/ServiceName/MethodName`。当外部流量尝试访问内置服务，比如说/status时，将直接被nginx拒绝。
```nginx
  location /MyAPI {
      ...
      proxy_pass http://<target-server>/ServiceName/MethodName$query_string   # $query_string是nginx变量，更多变量请查询http://nginx.org/en/docs/http/ngx_http_core_module.html
      ...
  }
```
**请勿在对外服务上开启**-enable_dir_service和-enable_threads_service两个选项，它们虽然很方便，但会严重泄露服务器上的其他信息。检查对外的rpc服务是否打开了这两个开关：

```shell
curl -s -m 1 <HOSTNAME>:<PORT>/flags/enable_dir_service,enable_threads_service | awk '{if($3=="false"){++falsecnt}else if($3=="Value"){isrpc=1}}END{if(isrpc!=1||falsecnt==2){print "SAFE"}else{print "NOT SAFE"}}'
```
### 转义外部可控的URL

可调用brpc::WebEscape()对url进行转义，防止恶意URI注入攻击。

### 不返回内部server地址

可以考虑对server地址做签名。比如在设置ServerOptions.internal_port后，server返回的错误信息中的IP信息是其MD5签名，而不是明文。

## 定制/health页面

/health页面默认返回"OK"，若需定制/health页面的内容：先继承[HealthReporter](https://github.com/brpc/brpc/blob/master/src/brpc/health_reporter.h)，在其中实现生成页面的逻辑（就像实现其他http service那样），然后把实例赋给ServerOptions.health_reporter，这个实例不被server拥有，必须保证在server运行期间有效。用户在定制逻辑中可以根据业务的运行状态返回更多样的状态信息。

## 线程私有变量

百度内的检索程序大量地使用了[thread-local storage](https://en.wikipedia.org/wiki/Thread-local_storage) (缩写TLS)，有些是为了缓存频繁访问的对象以避免反复创建，有些则是为了在全局函数间隐式地传递状态。你应当尽量避免后者，这样的函数难以测试，不设置thread-local变量甚至无法运行。brpc中有三套机制解决和thread-local相关的问题。

### session-local

session-local data与一次server端RPC绑定: 从进入service回调开始，到调用server端的done结束，不管该service是同步还是异步处理。 session-local data会尽量被重用，在server停止前不会被删除。

设置ServerOptions.session_local_data_factory后访问Controller.session_local_data()即可获得session-local数据。若没有设置，Controller.session_local_data()总是返回NULL。

若ServerOptions.reserved_session_local_data大于0，Server会在提供服务前就创建这么多个数据。

**示例用法**

```c++
struct MySessionLocalData {
    MySessionLocalData() : x(123) {}
    int x;
};
 
class EchoServiceImpl : public example::EchoService {
public:
    ...
    void Echo(google::protobuf::RpcController* cntl_base,
              const example::EchoRequest* request,
              example::EchoResponse* response,
              google::protobuf::Closure* done) {
        ...
        brpc::Controller* cntl = static_cast<brpc::Controller*>(cntl_base);
 
        // Get the session-local data which is created by ServerOptions.session_local_data_factory
        // and reused between different RPC.
        MySessionLocalData* sd = static_cast<MySessionLocalData*>(cntl->session_local_data());
        if (sd == NULL) {
            cntl->SetFailed("Require ServerOptions.session_local_data_factory to be set with a correctly implemented instance");
            return;
        }
        ...
```

```c++
struct ServerOptions {
    ...
    // The factory to create/destroy data attached to each RPC session.
    // If this field is NULL, Controller::session_local_data() is always NULL.
    // NOT owned by Server and must be valid when Server is running.
    // Default: NULL
    const DataFactory* session_local_data_factory;
 
    // Prepare so many session-local data before server starts, so that calls
    // to Controller::session_local_data() get data directly rather than
    // calling session_local_data_factory->Create() at first time. Useful when
    // Create() is slow, otherwise the RPC session may be blocked by the
    // creation of data and not served within timeout.
    // Default: 0
    size_t reserved_session_local_data;
};
```

session_local_data_factory的类型为[DataFactory](https://github.com/brpc/brpc/blob/master/src/brpc/data_factory.h)，你需要实现其中的CreateData和DestroyData。

注意：CreateData和DestroyData会被多个线程同时调用，必须线程安全。

```c++
class MySessionLocalDataFactory : public brpc::DataFactory {
public:
    void* CreateData() const {
        return new MySessionLocalData;
    }  
    void DestroyData(void* d) const {
        delete static_cast<MySessionLocalData*>(d);
    }  
};
 
int main(int argc, char* argv[]) {
    ...
    MySessionLocalDataFactory session_local_data_factory;
 
    brpc::Server server;
    brpc::ServerOptions options;
    ...
    options.session_local_data_factory = &session_local_data_factory;
    ...
```

### server-thread-local

server-thread-local与一次service回调绑定，从进service回调开始，到出service回调结束。所有的server-thread-local data会被尽量重用，在server停止前不会被删除。在实现上server-thread-local是一个特殊的bthread-local。

设置ServerOptions.thread_local_data_factory后访问brpc::thread_local_data()即可获得thread-local数据。若没有设置，brpc::thread_local_data()总是返回NULL。

若ServerOptions.reserved_thread_local_data大于0，Server会在启动前就创建这么多个数据。

**与session-local的区别**

session-local data得从server端的Controller获得， server-thread-local可以在任意函数中获得，只要这个函数直接或间接地运行在server线程中。

当service是同步时，session-local和server-thread-local基本没有差别，除了前者需要Controller创建。当service是异步时，且你需要在done->Run()中访问到数据，这时只能用session-local，因为server-thread-local在service回调外已经失效。

**示例用法**

```c++
struct MyThreadLocalData {
    MyThreadLocalData() : y(0) {}
    int y;
};
 
class EchoServiceImpl : public example::EchoService {
public:
    ...
    void Echo(google::protobuf::RpcController* cntl_base,
              const example::EchoRequest* request,
              example::EchoResponse* response,
              google::protobuf::Closure* done) {
        ...
        brpc::Controller* cntl = static_cast<brpc::Controller*>(cntl_base);
         
        // Get the thread-local data which is created by ServerOptions.thread_local_data_factory
        // and reused between different threads.
        // "tls" is short for "thread local storage".
        MyThreadLocalData* tls = static_cast<MyThreadLocalData*>(brpc::thread_local_data());
        if (tls == NULL) {
            cntl->SetFailed("Require ServerOptions.thread_local_data_factory "
                            "to be set with a correctly implemented instance");
            return;
        }
        ...
```

```c++
struct ServerOptions {
    ...    
    // The factory to create/destroy data attached to each searching thread
    // in server.
    // If this field is NULL, brpc::thread_local_data() is always NULL.
    // NOT owned by Server and must be valid when Server is running.
    // Default: NULL
    const DataFactory* thread_local_data_factory;
 
    // Prepare so many thread-local data before server starts, so that calls
    // to brpc::thread_local_data() get data directly rather than calling
    // thread_local_data_factory->Create() at first time. Useful when Create()
    // is slow, otherwise the RPC session may be blocked by the creation
    // of data and not served within timeout.
    // Default: 0
    size_t reserved_thread_local_data;
};
```

thread_local_data_factory的类型为[DataFactory](https://github.com/brpc/brpc/blob/master/src/brpc/data_factory.h)，你需要实现其中的CreateData和DestroyData。

注意：CreateData和DestroyData会被多个线程同时调用，必须线程安全。

```c++
class MyThreadLocalDataFactory : public brpc::DataFactory {
public:
    void* CreateData() const {
        return new MyThreadLocalData;
    }  
    void DestroyData(void* d) const {
        delete static_cast<MyThreadLocalData*>(d);
    }  
};
 
int main(int argc, char* argv[]) {
    ...
    MyThreadLocalDataFactory thread_local_data_factory;
 
    brpc::Server server;
    brpc::ServerOptions options;
    ...
    options.thread_local_data_factory  = &thread_local_data_factory;
    ...
```

### bthread-local

Session-local和server-thread-local对大部分server已经够用。不过在一些情况下，我们需要更通用的thread-local方案。在这种情况下，你可以使用bthread_key_create, bthread_key_destroy, bthread_getspecific, bthread_setspecific等函数，它们的用法类似[pthread中的函数](http://linux.die.net/man/3/pthread_key_create)。

这些函数同时支持bthread和pthread，当它们在bthread中被调用时，获得的是bthread私有变量; 当它们在pthread中被调用时，获得的是pthread私有变量。但注意，这里的“pthread私有变量”不是通过pthread_key_create创建的，使用pthread_key_create创建的pthread-local是无法被bthread_getspecific访问到的，这是两个独立的体系。由gcc的__thread，c++11的thread_local等声明的私有变量也无法被bthread_getspecific访问到。

由于brpc会为每个请求建立一个bthread，server中的bthread-local行为特殊：一个server创建的bthread在退出时并不删除bthread-local，而是还回server的一个pool中，以被其他bthread复用。这可以避免bthread-local随着bthread的创建和退出而不停地构造和析构。这对于用户是透明的。

**主要接口**

```c++
// Create a key value identifying a slot in a thread-specific data area.
// Each thread maintains a distinct thread-specific data area.
// `destructor', if non-NULL, is called with the value associated to that key
// when the key is destroyed. `destructor' is not called if the value
// associated is NULL when the key is destroyed.
// Returns 0 on success, error code otherwise.
extern int bthread_key_create(bthread_key_t* key, void (*destructor)(void* data));
 
// Delete a key previously returned by bthread_key_create().
// It is the responsibility of the application to free the data related to
// the deleted key in any running thread. No destructor is invoked by
// this function. Any destructor that may have been associated with key
// will no longer be called upon thread exit.
// Returns 0 on success, error code otherwise.
extern int bthread_key_delete(bthread_key_t key);
 
// Store `data' in the thread-specific slot identified by `key'.
// bthread_setspecific() is callable from within destructor. If the application
// does so, destructors will be repeatedly called for at most
// PTHREAD_DESTRUCTOR_ITERATIONS times to clear the slots.
// NOTE: If the thread is not created by brpc server and lifetime is
// very short(doing a little thing and exit), avoid using bthread-local. The
// reason is that bthread-local always allocate keytable on first call to
// bthread_setspecific, the overhead is negligible in long-lived threads,
// but noticeable in shortly-lived threads. Threads in brpc server
// are special since they reuse keytables from a bthread_keytable_pool_t
// in the server.
// Returns 0 on success, error code otherwise.
// If the key is invalid or deleted, return EINVAL.
extern int bthread_setspecific(bthread_key_t key, void* data);
 
// Return current value of the thread-specific slot identified by `key'.
// If bthread_setspecific() had not been called in the thread, return NULL.
// If the key is invalid or deleted, return NULL.
extern void* bthread_getspecific(bthread_key_t key);
```

**使用方法**

用bthread_key_create创建一个bthread_key_t，它代表一种bthread私有变量。

用bthread_[get|set]specific查询和设置bthread私有变量。一个线程中第一次访问某个私有变量返回NULL。

在所有线程都不使用和某个bthread_key_t相关的私有变量后再删除它。如果删除了一个仍在被使用的bthread_key_t，相关的私有变量就泄露了。

```c++
static void my_data_destructor(void* data) {
    ...
}
 
bthread_key_t tls_key;
 
if (bthread_key_create(&tls_key, my_data_destructor) != 0) {
    LOG(ERROR) << "Fail to create tls_key";
    return -1;
}
```
```c++
// in some thread ...
MyThreadLocalData* tls = static_cast<MyThreadLocalData*>(bthread_getspecific(tls_key));
if (tls == NULL) {  // First call to bthread_getspecific (and before any bthread_setspecific) returns NULL
    tls = new MyThreadLocalData;   // Create thread-local data on demand.
    CHECK_EQ(0, bthread_setspecific(tls_key, tls));  // set the data so that next time bthread_getspecific in the thread returns the data.
}
```
**示例代码**

```c++
static void my_thread_local_data_deleter(void* d) {
    delete static_cast<MyThreadLocalData*>(d);
}  
 
class EchoServiceImpl : public example::EchoService {
public:
    EchoServiceImpl() {
        CHECK_EQ(0, bthread_key_create(&_tls2_key, my_thread_local_data_deleter));
    }
    ~EchoServiceImpl() {
        CHECK_EQ(0, bthread_key_delete(_tls2_key));
    };
    ...
private:
    bthread_key_t _tls2_key;
}
 
class EchoServiceImpl : public example::EchoService {
public:
    ...
    void Echo(google::protobuf::RpcController* cntl_base,
              const example::EchoRequest* request,
              example::EchoResponse* response,
              google::protobuf::Closure* done) {
        ...
        // You can create bthread-local data for your own.
        // The interfaces are similar with pthread equivalence:
        //   pthread_key_create  -> bthread_key_create
        //   pthread_key_delete  -> bthread_key_delete
        //   pthread_getspecific -> bthread_getspecific
        //   pthread_setspecific -> bthread_setspecific
        MyThreadLocalData* tls2 = static_cast<MyThreadLocalData*>(bthread_getspecific(_tls2_key));
        if (tls2 == NULL) {
            tls2 = new MyThreadLocalData;
            CHECK_EQ(0, bthread_setspecific(_tls2_key, tls2));
        }
        ...
```

# FAQ

### Q: Fail to write into fd=1865 SocketId=8905@10.208.245.43:54742@8230: Got EOF是什么意思

A: 一般是client端使用了连接池或短连接模式，在RPC超时后会关闭连接，server写回response时发现连接已经关了就报这个错。Got EOF就是指之前已经收到了EOF（对端正常关闭了连接）。client端使用单连接模式server端一般不会报这个。

### Q: Remote side of fd=9 SocketId=2@10.94.66.55:8000 was closed是什么意思

这不是错误，是常见的warning，表示对端关掉连接了（EOF)。这个日志有时对排查问题有帮助。

默认关闭，把参数-log_connection_close设置为true就打开了（支持[动态修改](flags.md#change-gflag-on-the-fly)）。

### Q: 为什么server端线程数设了没用

brpc同一个进程中所有的server[共用线程](#worker线程数)，如果创建了多个server，最终的工作线程数多半是最大的那个ServerOptions.num_threads。

### Q: 为什么client端的延时远大于server端的延时

可能是server端的工作线程不够用了，出现了排队现象。排查方法请查看[高效率排查服务卡顿](server_debugging.md)。

### Q: Fail to open /proc/self/io

有些内核没这个文件，不影响服务正确性，但如下几个bvar会无法更新：
```
process_io_read_bytes_second
process_io_write_bytes_second
process_io_read_second
process_io_write_second
```
### Q: json串"[1,2,3]"没法直接转为protobuf message

这不是标准的json。最外层必须是花括号{}包围的json object。

# 附:Server端基本流程

![img](../images/server_side.png)


# ========= ./server_debugging.md ========
# 1.检查工作线程的数量

查看 /vars/bthread_worker_**count** 和 /vars/bthread_worker_**usage**。分别是工作线程的个数，和正在被使用的工作线程个数。

> 如果usage和count接近，说明线程不够用了。

比如，下图中有24个工作线程，正在使用的是23.93个，说明所有的工作线程都被打满了，不够用了。

![img](../images/full_worker_usage.png)

下图中正在使用的只有2.36个，工作线程明显是足够的。

![img](../images/normal_worker_usage.png)

把 /vars/bthread_worker_count;bthread_worker_usage?expand 拼在服务url后直接看到这两幅图，就像[这样](http://brpc.baidu.com:8765/vars/bthread_worker_count;bthread_worker_usage?expand)。

# 2.检查CPU的使用程度

查看 /vars/system_core_**count** 和 /vars/process_cpu_**usage**。分别是cpu核心的个数，和正在使用的cpu核数。

> 如果usage和count接近，说明CPU不够用了。

下图中cpu核数为24，正在使用的核心数是20.9个，CPU是瓶颈了。

![img](../images/high_cpu_usage.png)

下图中正在使用的核心数是2.06，CPU是够用的。

![img](../images/normal_cpu_usage.png)

# 3.定位问题

如果process_cpu_usage和bthread_worker_usage接近，说明是cpu-bound，工作线程大部分时间在做计算。

如果process_cpu_usage明显小于bthread_worker_usage，说明是io-bound，工作线程大部分时间在阻塞。

1 - process_cpu_usage / bthread_worker_usage就是大约在阻塞上花费的时间比例，比如process_cpu_usage = 2.4，bthread_worker_usage = 18.5，那么工作线程大约花费了87.1% 的时间在阻塞上。

## 3.1 定位cpu-bound问题

原因可能是单机性能不足，或上游分流不均。

### 排除上游分流不均的嫌疑

在不同服务的[vars界面](http://brpc.baidu.com:8765/vars)输入qps，查看不同的qps是否符合预期，就像这样：

![img](../images/bthread_creation_qps.png)

或者在命令行中用curl直接访问，像这样：

```shell
$ curl brpc.baidu.com:8765/vars/*qps*
bthread_creation_qps : 95
rpc_server_8765_example_echo_service_echo_qps : 57
```

如果不同机器的分流确实不均，且难以解决，可以考虑[限制最大并发](server.md#限制最大并发)。

### 优化单机性能

请使用[CPU profiler](cpu_profiler.md)分析程序的热点，用数据驱动优化。一般来说一个卡顿的cpu-bound程序一般能看到显著的热点。

## 3.2 定位io-bound问题

原因可能有：

- 线程确实配少了
- 访问下游服务的client不支持bthread，且延时过长
- 阻塞来自程序内部的锁，IO等等。

如果阻塞无法避免，考虑用异步。

### 排除工作线程数不够的嫌疑

如果线程数不够，你可以尝试动态调大工作线程数，切换到/flags页面，点击bthread_concurrency右边的(R):

![img](../images/bthread_concurrency_1.png)

进入后填入新的线程数确认即可：

![img](../images/bthread_concurrency_2.png)

回到/flags界面可以看到bthread_concurrency已变成了新值。

![img](../images/bthread_concurrency_3.png)

不过，调大线程数未必有用。如果工作线程是由于访问下游而大量阻塞，调大工作线程数是没有用的。因为真正的瓶颈在于后端的，调大线程后只是让每个线程的阻塞时间变得更长。

比如在我们这的例子中，调大线程后新增的工作线程仍然被打满了。

![img](../images/full_worker_usage_2.png)

### 排除锁的嫌疑

如果程序被某把锁挡住了，也可能呈现出“io-bound”的特征。先用[contention profiler](contention_profiler.md)排查锁的竞争状况。

### 使用rpcz

rpcz可以帮助你看到最近的所有请求，和处理它们时在每个阶段花费的时间（单位都是微秒）。

![img](../images/rpcz.png)

点击一个span链接后看到该次RPC何时开始，每个阶段花费的时间，何时结束。

![img](../images/rpcz_2.png)

这是一个典型的server在严重阻塞的例子。从接收到请求到开始运行花费了20ms，说明server已经没有足够的工作线程来及时完成工作了。

现在这个span的信息比较少，我们去程序里加一些。你可以使用TRACEPRINTF向rpcz打印日志。打印内容会嵌入在rpcz的时间流中。

![img](../images/trace_printf.png)

重新运行后，查看一个span，里面的打印内容果然包含了我们增加的TRACEPRINTF。

![img](../images/rpcz_3.png)

在运行到第一条TRACEPRINTF前，用户回调已运行了2051微秒（假设这符合我们的预期），紧接着foobar()却花费了8036微秒，我们本来以为这个函数会很快返回的。范围进一步缩小了。

重复这个过程，直到找到那个造成问题的函数。

### 使用bvar

TRACEPRINTF主要适合若干次的函数调用，如果一个函数调用了很多次，或者函数本身开销很小，每次都往rpcz打印日志是不合适的。这时候你可以使用bvar。

[bvar](bvar.md)是一个多线程下的计数库，可以以极低的开销统计用户递来的数值，相比“打日志大法”几乎不影响程序行为。你不用完全了解bvar的完整用法，只要使用bvar::LatencyRecorder即可。

仿照如下代码对foobar的运行时间进行监控。

```c++
#include <butil/time.h>
#include <bvar/bvar.h>
 
bvar::LatencyRecorder g_foobar_latency("foobar");
 
...
void search() {
    ...
    butil::Timer tm;
    tm.start();
    foobar();
    tm.stop();
    g_foobar_latency << tm.u_elapsed();
    ...
}
```

重新运行程序后，在vars的搜索框中键入foobar，显示如下：

![img](../images/foobar_bvar.png)

点击一个bvar可以看到动态图，比如点击cdf后看到

![img](../images/foobar_latency_cdf.png)

根据延时的分布，你可以推测出这个函数的整体行为，对大多数请求表现如何，对长尾表现如何。

你可以在子函数中继续这个过程，增加更多bvar，并比对不同的分布，最后定位来源。

### 只使用了brpc client

得打开dummy server提供内置服务，方法见[这里](dummy_server.md)。


# ========= ./server_push.md ========
[English version](../en/server_push.md)

# Server push

server push指的是server端发生某事件后立刻向client端发送消息，而不是像通常的RPC访问中那样被动地回复client。brpc中推荐以如下两种方式实现推送。

# 远程事件

和本地事件类似，分为两步：注册和通知。client发送一个代表**事件注册**的异步RPC至server，处理事件的代码写在对应的RPC回调中。此RPC同时也在等待通知，server收到请求后不直接回复，而是等到对应的本地事件触发时才调用done->Run()**通知**client发生了事件。可以看到server也是异步的。这个过程中如果连接断开，client端的RPC一般会很快失败，client可选择重试或结束。server端应通过Controller.NotifyOnFailed()及时获知连接断开的消息，并删除无用的done。

这个模式在原理上类似[long polling](https://en.wikipedia.org/wiki/Push_technology#Long_polling)，听上去挺古老，但可能仍是最有效的推送方式。“server push“乍看是server访问client，与常见的client访问server方向相反，挺特殊的，但server发回client的response不也和“client访问server”方向相反么？为了理解response和push的区别，我们假定“client随时可能收到server推来的消息“，并推敲其中的细节：

* client首先得认识server发来的消息，否则是鸡同鸭讲。
* client还得知道怎么应对server发来的消息，如果client上没有对应的处理代码，仍然没用。

换句话说，client得对server消息“有准备”，这种“准备”还往往依赖client当时的上下文。综合来看，由client告知server“我准备好了”(注册)，之后server再通知client是更普适的模式，**这个模式中的"push"就是"response"**，一个超时很长或无限长RPC的response。

在一些非常明确的场景中，注册可以被简略，如http2中的[push promise](https://tools.ietf.org/html/rfc7540#section-8.2)并不需要浏览器(client)向server注册，因为client和server都知道它们之间的任务就是让client尽快地下载必要的资源。由于每个资源有唯一的URI，server可以直接向client推送资源及其URI，client看到这些资源时会缓存下来避免下次重复访问。类似的，一些协议提供的"双向通信"也是在限定的场景中提高推送效率，而不是实现通用的推送，比如多媒体流，格式固定的key/value对等。client默认能处理server所有可能推送的消息，以至于不需要额外的注册。但推送仍可被视作"response"：client和server早已约定好的请求的response。

# Restful回调

client希望在事件发生时调用一个给定的URL，并附上必要的参数。在这个模式中，server在收到client注册请求时可以直接回复，因为事件不由注册用RPC的结束触发。由于回调只是一个URL，可以存放于数据库或经消息队列流转，这个模式灵活性很高，在业务系统中使用广泛。

URL和参数中必须有足够的信息使回调知道这次调用对应某次注册，因为client未必一次只关心一个事件，即使一个事件也可能由于网络抖动、机器重启等因素注册多次。如果回调是固定路径，client应在注册请求中置入一个唯一ID，在回调时带回。或者client为每次注册生成唯一路径，自然也可以区分。本质上这两种形式是一样的，只是唯一标志符出现的位置不同。

回调应处理[幂等问题](https://en.wikipedia.org/wiki/Idempotence)，server为了确保不漏通知，在网络出现问题时往往会多次重试，如果第一次的通知已经成功了，后续的通知就应该不产生效果。上节“远程事件”模式中的幂等性由RPC代劳，它会确保done只被调用一次。

为了避免重要的通知被漏掉，用户往往还需灵活组合RPC和消息队列。RPC的时效性和开销都明显好于消息队列，但由于内存有限，在重试过一些次数后仍然失败的话，server就得把这部分内存空出来去做其他事情了。这时把通知放到消息队列中，利用其持久化能力做较长时间的重试直至成功，辅以回调的幂等性，就能使绝大部分通知既及时又不会被漏掉。

# ========= ./status.md ========
[English version](../en/status.md)

[/status](http://brpc.baidu.com:8765/status)可以访问服务的主要统计信息。这些信息和/vars是同源的，但按服务重新组织方便查看。

![img](../images/status.png)

上图中字段的含义分别是：

- **non_service_error**: 在service处理过程之外的错误个数。比如client断开连接导致server无法成功写回response算*non_service_error*，此时service处理已结束。作为对比，服务过程中对后端服务的访问错误不是*non_service_error*。即使写出的response代表错误，此error也被记入对应的service，而不是*non_service_error*。
- **connection_count**: 向该server发起请求的连接个数。不包含记录在/vars/rpc_channel_connection_count的对外连接的个数。
- **example.EchoService**: 服务的完整名称，包含proto中的包名。
- **Echo (EchoRequest) returns (EchoResponse)**: 方法签名，一个服务可包含多个方法，点击request/response上的链接可查看对应的protobuf结构体。
- **count**: 成功处理的请求总个数。
- **error**: 失败的请求总个数。
- **latency**: 在html下是*从右到左*分别是过去60秒，60分钟，24小时，30天的平均延时。纯文本下是10秒内([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制)的平均延时。
- **latency_percentiles**: 是延时的50%, 90%, 99%, 99.9%分位值，统计窗口默认10秒([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制)，在html下有曲线。
- **latency_cdf**: 用[CDF](https://en.wikipedia.org/wiki/Cumulative_distribution_function)展示分位值, 只能在html下查看。
- **max_latency**: 在html下*从右到左*分别是过去60秒，60分钟，24小时，30天的最大延时。纯文本下是10秒内([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制)的最大延时。
- **qps**: 在html下从右到左分别是过去60秒，60分钟，24小时，30天的平均qps(Queries Per Second)。纯文本下是10秒内([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制)的平均qps。
- **processing**: (新版改名为concurrency)正在处理的请求个数。在压力归0后若此指标仍持续不为0，server则很有可能bug，比如忘记调用done了或卡在某个处理步骤上了。


用户可通过让对应Service实现[brpc::Describable](https://github.com/brpc/brpc/blob/master/src/brpc/describable.h)自定义在/status页面上的描述.

```c++
class MyService : public XXXService, public brpc::Describable {
public:
    ...
    void DescribeStatus(std::ostream& os, const brpc::DescribeOptions& options) const {
        os << "my_status: blahblah";
    }
};
```

比如:

![img](../images/status_2.png)


# ========= ./streaming_log.md ========
# Name

streaming_log - Print log to std::ostreams

# SYNOPSIS

```c++
#include <butil/logging.h>

LOG(FATAL) << "Fatal error occurred! contexts=" << ...;
LOG(WARNING) << "Unusual thing happened ..." << ...;
LOG(TRACE) << "Something just took place..." << ...;
LOG(TRACE) << "Items:" << noflush;
LOG_IF(NOTICE, n > 10) << "This log will only be printed when n > 10";
PLOG(FATAL) << "Fail to call function setting errno";
VLOG(1) << "verbose log tier 1";
CHECK_GT(1, 2) << "1 can't be greater than 2";
 
LOG_EVERY_SECOND(INFO) << "High-frequent logs";
LOG_EVERY_N(ERROR, 10) << "High-frequent logs";
LOG_FIRST_N(INFO, 20) << "Logs that prints for at most 20 times";
LOG_ONCE(WARNING) << "Logs that only prints once";
```

# DESCRIPTION

流式日志是打印复杂对象或模板对象的不二之选。大部分业务对象都很复杂，如果用printf形式的函数打印，你需要先把对象转成string，才能以%s输出。但string组合起来既不方便（比如没法append数字），还得分配大量的临时内存(string导致的）。C++中解决这个问题的方法便是“把日志流式地送入std::ostream对象”。比如为了打印对象A，那么我们得实现如下的函数：

```c++
std::ostream& operator<<(std::ostream& os, const A& a);
```

这个函数的意思是把对象a打印入os，并返回os。之所以返回os，是因为operator<<对应了二元操作 << （左结合），当我们写下`os << a << b << c;`时，它相当于`operator<<(operator<<(operator<<(os, a), b), c);` 很明显，operator<<需要不断地返回os（的引用），才能完成这个过程。这个过程一般称为chaining。在不支持重载二元运算符的语言中，你可能会看到一些更繁琐的形式，比如`os.print(a).print(b).print(c)`。

我们在operator<<的实现中也使用chaining。事实上，流式打印一个复杂对象就像DFS一棵树一样：逐个调用儿子节点的operator<<，儿子又逐个调用孙子节点的operator<<，以此类推。比如对象A有两个成员变量B和C，打印A的过程就是把其中的B和C对象送入ostream中：

```c++
struct A {
    B b;
    C c;
};
std::ostream& operator<<(std::ostream& os, const A& a) {
    return os << "A{b=" << a.b << ", c=" << a.c << "}";
}
```

B和C的结构及打印函数分别如下：

```c++
struct B {
    int value;
};
std::ostream& operator<<(std::ostream& os, const B& b) {
    return os << "B{value=" << b.value << "}";
}
 
struct C {
    string name;
};
std::ostream& operator<<(std::ostream& os, const C& c) {
    return os << "C{name=" << c.name << "}";
}
```

那么打印某个A对象的结果可能是

```
A{b=B{value=10}, c=C{name=tom}}
```

在打印过程中，我们不需要分配临时内存，因为对象都被直接送入了最终要送入的那个ostream对象。当然ostream对象自身的内存管理是另一会儿事了。

OK，我们通过ostream把对象的打印过程串联起来了，最常见的std::cout和std::cerr都继承了ostream，所以实现了上面函数的对象就可以输出到std::cout和std::cerr了。换句话说如果日志流也继承了ostream，那么那些对象也可以打入日志了。流式日志正是通过继承std::ostream，把对象打入日志的，在目前的实现中，送入日志流的日志被记录在thread-local的缓冲中，在完成一条日志后会被刷入屏幕或logging::LogSink，这个实现是线程安全的。

## LOG

如果你用过glog，应该是不用学习的，因为宏名称和glog是一致的，如下打印一条FATAL。注意不需要加上std::endl。

```c++
LOG(FATAL) << "Fatal error occurred! contexts=" << ...;
LOG(WARNING) << "Unusual thing happened ..." << ...;
LOG(TRACE) << "Something just took place..." << ...;
```

streaming log的日志等级与glog映射关系如下：

| streaming log | glog                 | 使用场景                                     |
| ------------- | -------------------- | ---------------------------------------- |
| FATAL         | FATAL (coredump)     | 致命错误。但由于百度内大部分FATAL实际上非致命，所以streaming log的FATAL默认不像glog那样直接coredump，除非打开了[-crash_on_fatal_log](http://brpc.baidu.com:8765/flags/crash_on_fatal_log) |
| ERROR         | ERROR                | 不致命的错误。                                  |
| WARNING       | WARNING              | 不常见的分支。                                  |
| NOTICE        | -                    | 一般来说你不应该使用NOTICE，它用于打印重要的业务日志，若要使用务必和检索端同学确认。glog没有NOTICE。 |
| INFO, TRACE   | INFO                 | 打印重要的副作用。比如打开关闭了某某资源之类的。                 |
| VLOG(n)       | INFO                 | 打印分层的详细日志。                               |
| DEBUG         | INFOVLOG(1) (NDEBUG) | 仅为代码兼容性，基本没有用。若要使日志仅在未定义NDEBUG时才打印，用DLOG/DPLOG/DVLOG等即可。 |

## PLOG

PLOG和LOG的不同之处在于，它会在日志后加上错误码的信息，类似于printf中的%m。在posix系统中，错误码就是errno。

```c++
int fd = open("foo.conf", O_RDONLY);   // foo.conf does not exist, errno was set to ENOENT
if (fd < 0) { 
    PLOG(FATAL) << "Fail to open foo.conf";    // "Fail to open foo.conf: No such file or directory"
    return -1;
}
```

## noflush

如果你暂时不希望刷到屏幕，加上noflush。这一般会用在打印循环中：

```c++
LOG(TRACE) << "Items:" << noflush;
for (iterator it = items.begin(); it != items.end(); ++it) {
    LOG(TRACE) << ' ' << *it << noflush;
}
LOG(TRACE);
```

前两次TRACE日志都没有刷到屏幕，而是还记录在thread-local缓冲中，第三次TRACE日志则把缓冲都刷入了屏幕。如果items里面有三个元素，不加noflush的打印结果可能是这样的：

```
TRACE: ... Items:
TRACE: ...  item1
TRACE: ...  item2
TRACE: ...  item3
```

加了是这样的：

```
TRACE: ... Items: item1 item2 item3 
```

noflush支持bthread，可以实现类似于UB的pushnotice的效果，即检索线程一路打印都暂不刷出（加上noflush），直到最后检索结束时再一次性刷出。注意，如果检索过程是异步的，就不应该使用noflush，因为异步显然会跨越bthread，使noflush仍然失效。

## LOG_IF

`LOG_IF(log_level, condition)`只有当condition成立时才会打印，相当于if (condition) { LOG() << ...; }，但更加简短。比如：

```c++
LOG_IF(NOTICE, n > 10) << "This log will only be printed when n > 10";
```

## XXX_EVERY_SECOND

XXX可以是LOG，LOG_IF，PLOG，SYSLOG，VLOG，DLOG等。这类日志每秒最多打印一次，可放在频繁运行热点处探查运行状态。第一次必打印，比普通LOG增加调用一次gettimeofday（30ns左右）的开销。

```c++
LOG_EVERY_SECOND(INFO) << "High-frequent logs";
```

## XXX_EVERY_N

XXX可以是LOG，LOG_IF，PLOG，SYSLOG，VLOG，DLOG等。这类日志每触发N次才打印一次，可放在频繁运行热点处探查运行状态。第一次必打印，比普通LOG增加一次relaxed原子加的开销。这个宏是线程安全的，即不同线程同时运行这段代码时对N的限制也是准确的，glog中的不是。

```c++
LOG_EVERY_N(ERROR, 10) << "High-frequent logs";
```

## XXX_FIRST_N

XXX可以是LOG，LOG_IF，PLOG，SYSLOG，VLOG，DLOG等。这类日志最多打印N次。在N次前比普通LOG增加一次relaxed原子加的开销，N次后基本无开销。

```c++
LOG_FIRST_N(ERROR, 20) << "Logs that prints for at most 20 times";
```

## XXX_ONCE

XXX可以是LOG，LOG_IF，PLOG，SYSLOG，VLOG，DLOG等。这类日志最多打印1次。等价于XXX_FIRST_N(..., 1)

```c++
LOG_ONCE(ERROR) << "Logs that only prints once";
```

## VLOG

VLOG(verbose_level)是分层的详细日志，通过两个gflags：*--verbose*和*--verbose_module*控制需要打印的层（注意glog是--v和–vmodule）。只有当–verbose指定的值大于等于verbose_level时，对应的VLOG才会打印。比如

```c++
VLOG(0) << "verbose log tier 0";
VLOG(1) << "verbose log tier 1";
VLOG(2) << "verbose log tier 2";
```

当`--verbose=1`时，前两条会打印，最后一条不会。--`verbose_module`可以覆盖某个模块的级别，模块指**去掉扩展名的文件名或文件路径**。比如:

```bash
--verbose=1 --verbose_module="channel=2,server=3"                # 打印channel.cpp中<=2，server.cpp中<=3，其他文件<=1的VLOG
--verbose=1 --verbose_module="src/brpc/channel=2,server=3"  # 当不同目录下有同名文件时，可以加上路径
```

`--verbose`和`--verbose_module`可以通过`google::SetCommandLineOption`动态设置。

VLOG有一个变种VLOG2让用户指定虚拟文件路径，比如：

```c++
// public/foo/bar.cpp
VLOG2("a/b/c", 2) << "being filtered by a/b/c rather than public/foo/bar";
```

> VLOG和VLOG2也有相应的VLOG_IF和VLOG2_IF。

## DLOG

所有的日志宏都有debug版本，以D开头，比如DLOG，DVLOG，当定义了**NDEBUG**后，这些日志不会打印。

**千万别在D开头的日志流上有重要的副作用。**

“不会打印”指的是连参数都不会评估。如果你的参数是有副作用的，那么当定义了NDEBUG后，这些副作用都不会发生。比如DLOG(FATAL) << foo(); 其中foo是一个函数，它修改一个字典，反正必不可少，但当定义了NDEBUG后，foo就运行不到了。

## CHECK

日志另一个重要变种是CHECK(expression)，当expression为false时，会打印一条FATAL日志。类似gtest中的ASSERT，也有CHECK_EQ, CHECK_GT等变种。当CHECK失败后，其后的日志流会被打印。

```c++
CHECK_LT(1, 2) << "This is definitely true, this log will never be seen";
CHECK_GT(1, 2) << "1 can't be greater than 2";
```

运行后你应该看到一条FATAL日志和调用处的call stack：

```
FATAL: ... Check failed: 1 > 2 (1 vs 2). 1 can't be greater than 2
#0 0x000000afaa23 butil::debug::StackTrace::StackTrace()
#1 0x000000c29fec logging::LogStream::FlushWithoutReset()
#2 0x000000c2b8e6 logging::LogStream::Flush()
#3 0x000000c2bd63 logging::DestroyLogStream()
#4 0x000000c2a52d logging::LogMessage::~LogMessage()
#5 0x000000a716b2 (anonymous namespace)::StreamingLogTest_check_Test::TestBody()
#6 0x000000d16d04 testing::internal::HandleSehExceptionsInMethodIfSupported<>()
#7 0x000000d19e96 testing::internal::HandleExceptionsInMethodIfSupported<>()
#8 0x000000d08cd4 testing::Test::Run()
#9 0x000000d08dfe testing::TestInfo::Run()
#10 0x000000d08ec4 testing::TestCase::Run()
#11 0x000000d123c7 testing::internal::UnitTestImpl::RunAllTests()
#12 0x000000d16d94 testing::internal::HandleSehExceptionsInMethodIfSupported<>()
```

callstack中的第二列是代码地址，你可以使用addr2line查看对应的文件行数：

```
$ addr2line -e ./test_base 0x000000a716b2 
/home/gejun/latest_baidu_rpc/public/common/test/test_streaming_log.cpp:223
```

你**应该**根据比较关系使用具体的CHECK_XX，这样当出现错误时，你可以看到更详细的信息，比如：

```C++
int x = 1;
int y = 2;
CHECK_GT(x, y);  // Check failed: x > y (1 vs 2).
CHECK(x > y);    // Check failed: x > y.
```

和DLOG类似，你不应该在DCHECK的日志流中包含重要的副作用。

## LogSink

streaming log通过logging::SetLogSink修改日志刷入的目标，默认是屏幕。用户可以继承LogSink，实现自己的日志打印逻辑。我们默认提供了个LogSink实现：

### StringSink

同时继承了LogSink和string，把日志内容存放在string中，主要用于单测，这个case说明了StringSink的典型用法：

```C++
TEST_F(StreamingLogTest, log_at) {
    ::logging::StringSink log_str;
    ::logging::LogSink* old_sink = ::logging::SetLogSink(&log_str);
    LOG_AT(FATAL, "specified_file.cc", 12345) << "file/line is specified";
    // the file:line part should be using the argument given by us.
    ASSERT_NE(std::string::npos, log_str.find("specified_file.cc:12345"));
    // restore the old sink.
    ::logging::SetLogSink(old_sink);
}
```


# ========= ./streaming_rpc.md ========
[English version](../en/streaming_rpc.md)

# 概述

在一些应用场景中， client或server需要向对面发送大量数据，这些数据非常大或者持续地在产生以至于无法放在一个RPC的附件中。比如一个分布式系统的不同节点间传递replica或snapshot。client/server之间虽然可以通过多次RPC把数据切分后传输过去，但存在如下问题：

- 如果这些RPC是并行的，无法保证接收端有序地收到数据，拼接数据的逻辑相当复杂。
- 如果这些RPC是串行的，每次传递都得等待一次网络RTT+处理数据的延时，特别是后者的延时可能是难以预估的。

为了让大块数据以流水线的方式在client/server之间传递， 我们提供了Streaming RPC这种交互模型。Streaming RPC让用户能够在client/service之间建立用户态连接，称为Stream,  同一个TCP连接之上能同时存在多个Stream。 Stream的传输数据以消息为基本单位， 输入端可以源源不断的往Stream中写入消息， 接收端会按输入端写入顺序收到消息。

Streaming RPC保证：

- 有消息边界。
- 接收消息的顺序和发送消息的顺序严格一致。
- 全双工。
- 支持流控。
- 提供超时提醒

目前的实现还没有自动切割过大的消息，同一个tcp连接上的多个Stream之间可能有[Head-of-line blocking](https://en.wikipedia.org/wiki/Head-of-line_blocking)问题，请尽量避免过大的单个消息，实现自动切割后我们会告知并更新文档。

例子见[example/streaming_echo_c++](https://github.com/brpc/brpc/tree/master/example/streaming_echo_c++/)。

# 建立Stream

目前Stream都由Client端建立。Client先在本地创建一个Stream，再通过一次RPC（必须使用baidu_std协议）与指定的Service建立一个Stream，如果Service在收到请求之后选择接受这个Stream， 那在response返回Client后Stream就会建立成功。过程中的任何错误都把RPC标记为失败，同时也意味着Stream创建失败。用linux下建立连接的过程打比方，Client先创建[socket](http://linux.die.net/man/7/socket)（创建Stream），再调用[connect](http://linux.die.net/man/2/connect)尝试与远端建立连接（通过RPC建立Stream），远端[accept](http://linux.die.net/man/2/accept)后连接就建立了（service接受后创建成功）。

> 如果Client尝试向不支持Streaming RPC的老Server建立Stream，将总是失败。

程序中我们用StreamId代表一个Stream，对Stream的读写，关闭操作都将作用在这个Id上。

```c++
struct StreamOptions
    // The max size of unconsumed data allowed at remote side.
    // If |max_buf_size| <= 0, there's no limit of buf size
    // default: 2097152 (2M)
    int max_buf_size;
 
    // Notify user when there's no data for at least |idle_timeout_ms|
    // milliseconds since the last time that on_received_messages or on_idle_timeout
    // finished.
    // default: -1
    long idle_timeout_ms;
     
    // How many messages at most passed to handler->on_received_messages
    // default: 1
    size_t max_messages_size;
 
    // Handle input message, if handler is NULL, the remote side is not allowd to
    // write any message, who will get EBADF on writting
    // default: NULL
    StreamInputHandler* handler;
};
 
// [Called at the client side]
// Create a Stream at client-side along with the |cntl|, which will be connected
// when receiving the response with a Stream from server-side. If |options| is
// NULL, the Stream will be created with default options
// Return 0 on success, -1 otherwise
int StreamCreate(StreamId* request_stream, Controller &cntl, const StreamOptions* options);
```

# 接受Stream

如果client在RPC上附带了一个Stream， service在收到RPC后可以通过调用StreamAccept接受。接受后Server端对应产生的Stream存放在response_stream中，Server可通过这个Stream向Client发送数据。

```c++
// [Called at the server side]
// Accept the Stream. If client didn't create a Stream with the request
// (cntl.has_remote_stream() returns false), this method would fail.
// Return 0 on success, -1 otherwise.
int StreamAccept(StreamId* response_stream, Controller &cntl, const StreamOptions* options);
```

# 读取Stream

在建立或者接受一个Stream的时候， 用户可以继承StreamInputHandler并把这个handler填入StreamOptions中. 通过这个handler，用户可以处理对端的写入数据，连接关闭以及idle timeout

```c++
class StreamInputHandler {
public:
    // 当接收到消息后被调用
    virtual int on_received_messages(StreamId id, butil::IOBuf *const messages[], size_t size) = 0;
 
    // 当Stream上长时间没有数据交互后被调用
    virtual void on_idle_timeout(StreamId id) = 0;
 
    // 当Stream被关闭时被调用
    virtual void on_closed(StreamId id) = 0;
};
```

>***第一次收到请求的时间***
>
>在client端，如果建立过程是一次同步RPC， 那在等待的线程被唤醒之后，on_received_message就可能会被调用到。 如果是异步RPC请求， 那等到这次请求的done->Run() 执行完毕之后， on_received_message就可能会被调用。
>
>在server端， 当框架传入的done->Run()被调用完之后， on_received_message就可能会被调用。

# 写入Stream

```c++
// Write |message| into |stream_id|. The remote-side handler will received the
// message by the written order
// Returns 0 on success, errno otherwise
// Errno:
//  - EAGAIN: |stream_id| is created with positive |max_buf_size| and buf size
//            which the remote side hasn't consumed yet excceeds the number.
//  - EINVAL: |stream_id| is invalied or has been closed
int StreamWrite(StreamId stream_id, const butil::IOBuf &message);
```

# 流控

当存在较多已发送但未接收的数据时，发送端的Write操作会立即失败(返回EAGAIN）， 这时候可以通过同步或异步的方式等待对端消费掉数据。

```c++
// Wait util the pending buffer size is less than |max_buf_size| or error occurs
// Returns 0 on success, errno otherwise
// Errno:
//  - ETIMEDOUT: when |due_time| is not NULL and time expired this
//  - EINVAL: the Stream was close during waiting
int StreamWait(StreamId stream_id, const timespec* due_time);
 
// Async wait
void StreamWait(StreamId stream_id, const timespec *due_time,
                void (*on_writable)(StreamId stream_id, void* arg, int error_code),
                void *arg);
```

# 关闭Stream

```c++
// Close |stream_id|, after this function is called:
//  - All the following |StreamWrite| would fail
//  - |StreamWait| wakes up immediately.
//  - Both sides |on_closed| would be notifed after all the pending buffers have
//    been received
// This function could be called multiple times without side-effects
int StreamClose(StreamId stream_id);
```



# ========= ./thread_local.md ========
本页说明bthread下使用pthread-local可能会导致的问题。bthread-local的使用方法见[这里](server.md#bthread-local)。

# thread-local问题

调用阻塞的bthread函数后，所在的pthread很可能改变，这使[pthread_getspecific](http://linux.die.net/man/3/pthread_getspecific)，[gcc __thread](https://gcc.gnu.org/onlinedocs/gcc-4.2.4/gcc/Thread_002dLocal.html)和c++11
thread_local变量，pthread_self()等的值变化了，如下代码的行为是不可预计的：

```
thread_local SomeObject obj;
...
SomeObject* p = &obj;
p->bar();
bthread_usleep(1000);
p->bar();
```

bthread_usleep之后，该bthread很可能身处不同的pthread，这时p指向了之前pthread的thread_local变量，继续访问p的结果无法预计。这种使用模式往往发生在用户使用线程级变量传递业务变量的情况。为了防止这种情况，应该谨记：

- 不使用线程级变量传递业务数据。这是一种槽糕的设计模式，依赖线程级数据的函数也难以单测。判断是否滥用：如果不使用线程级变量，业务逻辑是否还能正常运行？线程级变量应只用作优化手段，使用过程中不应直接或间接调用任何可能阻塞的bthread函数。比如使用线程级变量的tcmalloc就不会和bthread有任何冲突。
- 如果一定要（在业务中）使用线程级变量，使用bthread_key_create和bthread_getspecific。

# gcc4下的errno问题

gcc4会优化[标记为__attribute__((__const__))](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html#index-g_t_0040code_007bconst_007d-function-attribute-2958)的函数，这个标记大致指只要参数不变，输出就不会变。所以当一个函数中以相同参数出现多次时，gcc4会合并为一次。比如在我们的系统中errno是内容为*__errno_location()的宏，这个函数的签名是：

```
/* Function to get address of global `errno' variable.  */
extern int *__errno_location (void) __THROW __attribute__ ((__const__));
```

由于此函数被标记为`__const__`，且没有参数，当你在一个函数中调用多次errno时，可能只有第一次才调用__errno_location()，而之后只是访问其返回的`int*`。在pthread中这没有问题，因为返回的`int*`是thread-local的，一个给定的pthread中是不会变化的。但是在bthread中，这是不成立的，因为一个bthread很可能在调用一些函数后跑到另一个pthread去，如果gcc4做了类似的优化，即一个函数内所有的errno都替换为第一次调用返回的int*，这中间bthread又切换了pthread，那么可能会访问之前pthread的errno，从而造成未定义行为。

比如下文是一种errno的使用场景：

```
Use errno ...   (original pthread)
bthread functions that may switch to another pthread.
Use errno ...   (another pthread) 
```

我们期望看到的行为：

```
Use *__errno_location() ...  -  the thread-local errno of original pthread
bthread may switch another pthread ...
Use *__errno_location() ...  -  the thread-local errno of another pthread
```

使用gcc4时的实际行为：

```
int* p= __errno_location();
Use *p ...                   -  the thread-local errno of original pthread
bthread context switches ...
Use *p ...                   -  still the errno of original pthread, undefined behavior!!
```

严格地说这个问题不是gcc4导致的，而是glibc给__errno_location的签名不够准确，一个返回thread-local指针的函数依赖于段寄存器（TLS的一般实现方式），这怎么能算const呢？由于我们还未找到覆盖__errno_location的方法，所以这个问题目前实际的解决方法是：

**务必在直接或间接使用bthread的项目的gcc编译选项中添加`-D__const__=`，即把`__const__`定义为空，避免gcc4做相关优化。**

把`__const__`定义为空对程序其他部分的影响几乎为0。另外如果你没有**直接**使用errno（即你的项目中没有出现errno），或使用的是gcc
3.4，即使没有定义`-D__const__=`，程序的正确性也不会受影响，但为了防止未来可能的问题，我们强烈建议加上。

需要说明的是，和errno类似，pthread_self也有类似的问题，不过一般pthread_self除了打日志没有其他用途，影响面较小，在`-D__const__=`后pthread_self也会正常。


# ========= ./threading_overview.md ========
[English version](../en/threading_overview.md)

# 常见线程模型

## 连接独占线程或进程

在这个模型中，线程/进程处理来自绑定连接的消息，在连接断开前不退也不做其他事情。当连接数逐渐增多时，线程/进程占用的资源和上下文切换成本会越来越大，性能很差，这就是[C10K问题](http://en.wikipedia.org/wiki/C10k_problem)的来源。这种方法常见于早期的web server，现在很少使用。

## 单线程[reactor](http://en.wikipedia.org/wiki/Reactor_pattern)

以[libevent](http://libevent.org/), [libev](http://software.schmorp.de/pkg/libev.html)等event-loop库为典型。这个模型一般由一个event dispatcher等待各类事件，待事件发生后**原地**调用对应的event handler，全部调用完后等待更多事件，故为"loop"。这个模型的实质是把多段逻辑按事件触发顺序交织在一个系统线程中。一个event-loop只能使用一个核，故此类程序要么是IO-bound，要么是每个handler有确定的较短的运行时间（比如http server)，否则一个耗时漫长的回调就会卡住整个程序，产生高延时。在实践中这类程序不适合多开发者参与，一个人写了阻塞代码可能就会拖慢其他代码的响应。由于event handler不会同时运行，不太会产生复杂的race condition，一些代码不需要锁。此类程序主要靠部署更多进程增加扩展性。

单线程reactor的运行方式及问题如下图所示：

![img](../images/threading_overview_1.png)

## N:1线程库

又称为[Fiber](http://en.wikipedia.org/wiki/Fiber_(computer_science))，以[GNU Pth](http://www.gnu.org/software/pth/pth-manual.html), [StateThreads](http://state-threads.sourceforge.net/index.html)等为典型，一般是把N个用户线程映射入一个系统线程。同时只运行一个用户线程，调用阻塞函数时才会切换至其他用户线程。N:1线程库与单线程reactor在能力上等价，但事件回调被替换为了上下文(栈,寄存器,signals)，运行回调变成了跳转至上下文。和event loop库一样，单个N:1线程库无法充分发挥多核性能，只适合一些特定的程序。只有一个系统线程对CPU cache较为友好，加上舍弃对signal mask的支持的话，用户线程间的上下文切换可以很快(100~200ns)。N:1线程库的性能一般和event loop库差不多，扩展性也主要靠多进程。

## 多线程reactor

以[boost::asio](http://www.boost.org/doc/libs/1_56_0/doc/html/boost_asio.html)为典型。一般由一个或多个线程分别运行event dispatcher，待事件发生后把event handler交给一个worker线程执行。 这个模型是单线程reactor的自然扩展，可以利用多核。由于共用地址空间使得线程间交互变得廉价，worker thread间一般会更及时地均衡负载，而多进程一般依赖更前端的服务来分割流量，一个设计良好的多线程reactor程序往往能比同一台机器上的多个单线程reactor进程更均匀地使用不同核心。不过由于[cache一致性](atomic_instructions.md#cacheline)的限制，多线程reactor并不能获得线性于核心数的性能，在特定的场景中，粗糙的多线程reactor实现跑在24核上甚至没有精致的单线程reactor实现跑在1个核上快。由于多线程reactor包含多个worker线程，单个event handler阻塞未必会延缓其他handler，所以event handler未必得非阻塞，除非所有的worker线程都被阻塞才会影响到整体进展。事实上，大部分RPC框架都使用了这个模型，且回调中常有阻塞部分，比如同步等待访问下游的RPC返回。

多线程reactor的运行方式及问题如下：

![img](../images/threading_overview_2.png)

## M:N线程库

即把M个用户线程映射入N个系统线程。M:N线程库可以决定一段代码何时开始在哪运行，并何时结束，相比多线程reactor在调度上具备更多的灵活度。但实现全功能的M:N线程库是困难的，它一直是个活跃的研究话题。我们这里说的M:N线程库特别针对编写网络服务，在这一前提下一些需求可以简化，比如没有时间片抢占，没有(完备的)优先级等。M:N线程库可以在用户态也可以在内核中实现，用户态的实现以新语言为主，比如GHC threads和goroutine，这些语言可以围绕线程库设计全新的关键字并拦截所有相关的API。而在现有语言中的实现往往得修改内核，比如[Windows UMS](https://msdn.microsoft.com/en-us/library/windows/desktop/dd627187(v=vs.85).aspx)和google SwicthTo(虽然是1:1，但基于它可以实现M:N的效果)。相比N:1线程库，M:N线程库在使用上更类似于系统线程，需要用锁或消息传递保证代码的线程安全。

# 问题

## 多核扩展性

理论上代码都写成事件驱动型能最大化reactor模型的能力，但实际由于编码难度和可维护性，用户的使用方式大都是混合的：回调中往往会发起同步操作，阻塞住worker线程使其无法处理其他请求。一个请求往往要经过几十个服务，线程把大量时间花在了等待下游请求上，用户得开几百个线程以维持足够的吞吐，这造成了高强度的调度开销，并降低了TLS相关代码的效率。任务的分发大都是使用全局mutex + condition保护的队列，当所有线程都在争抢时，效率显然好不到哪去。更好的办法也许是使用更多的任务队列，并调整调度算法以减少全局竞争。比如每个系统线程有独立的runqueue，由一个或多个scheduler把用户线程分发到不同的runqueue，每个系统线程优先运行自己runqueue中的用户线程，然后再考虑其他线程的runqueue。这当然更复杂，但比全局mutex + condition有更好的扩展性。这种结构也更容易支持NUMA。

当event dispatcher把任务递给worker线程时，用户逻辑很可能从一个核心跳到另一个核心，并等待相应的cacheline同步过来，并不很快。如果worker的逻辑能直接运行于event dispatcher所在的核心上就好了，因为大部分时候尽快运行worker的优先级高于获取新事件。类似的是收到response后最好在当前核心唤醒正在同步等待RPC的线程。

## 异步编程

异步编程中的流程控制对于专家也充满了陷阱。任何挂起操作，如sleep一会儿或等待某事完成，都意味着用户需要显式地保存状态，并在回调函数中恢复状态。异步代码往往得写成状态机的形式。当挂起较少时，这有点麻烦，但还是可把握的。问题在于一旦挂起发生在条件判断、循环、子函数中，写出这样的状态机并能被很多人理解和维护，几乎是不可能的，而这在分布式系统中又很常见，因为一个节点往往要与多个节点同时交互。另外如果唤醒可由多种事件触发（比如fd有数据或超时了），挂起和恢复的过程容易出现race condition，对多线程编码能力要求很高。语法糖(比如lambda)可以让编码不那么“麻烦”，但无法降低难度。

共享指针在异步编程中很普遍，这看似方便，但也使内存的ownership变得难以捉摸，如果内存泄漏了，很难定位哪里没有释放；如果segment fault了，也不知道哪里多释放了一下。大量使用引用计数的用户代码很难控制代码质量，容易长期在内存问题上耗费时间。如果引用计数还需要手动维护，保持质量就更难了，维护者也不会愿意改进。没有上下文会使得[RAII](http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)无法充分发挥作用, 有时需要在callback之外lock，callback之内unlock，实践中很容易出错。


# ========= ./thrift.md ========
[English Version](../en/thrift.md)

[thrift](https://thrift.apache.org/)是应用较广的RPC框架，最初由Facebook发布，后交由Apache维护。为了和thrift服务互通，同时解决thrift原生方案在多线程安全、易用性、并发能力等方面的一系列问题，brpc实现并支持thrift在NonBlocking模式下的协议(FramedProtocol), 下文均直接称为thrift协议。

示例程序：[example/thrift_extension_c++](https://github.com/brpc/brpc/tree/master/example/thrift_extension_c++/)

相比使用原生方案的优势有：
- 线程安全。用户不需要为每个线程建立独立的client.
- 支持同步、异步、批量同步、批量异步等访问方式，能使用ParallelChannel等组合访问方式.
- 支持多种连接方式(连接池, 短连接), 支持超时、backup request、取消、tracing、内置服务等一系列RPC基本福利.
- 性能更好.

# 编译
为了复用解析代码，brpc对thrift的支持仍需要依赖thrift库以及thrift生成的代码，thrift格式怎么写，代码怎么生成，怎么编译等问题请参考thrift官方文档。

brpc默认不启用thrift支持也不需要thrift依赖。但如果需用thrift协议, 配置brpc环境的时候需加上--with-thrift或-DWITH_THRIFT=ON.

Linux下安装thrift依赖
先参考[官方wiki](https://thrift.apache.org/docs/install/debian)安装好必备的依赖和工具，然后从[官网](https://thrift.apache.org/download)下载thrift源代码，解压编译。
```bash
wget http://www.us.apache.org/dist/thrift/0.11.0/thrift-0.11.0.tar.gz
tar -xf thrift-0.11.0.tar.gz
cd thrift-0.11.0/
./configure --prefix=/usr --with-ruby=no --with-python=no --with-java=no --with-go=no --with-perl=no --with-php=no --with-csharp=no --with-erlang=no --with-lua=no --with-nodejs=no
make CPPFLAGS=-DFORCE_BOOST_SMART_PTR -j 4 -s
sudo make install
```

配置brpc支持thrift协议后make。编译完成后会生成libbrpc.a, 其中包含了支持thrift协议的扩展代码, 像正常使用brpc的代码一样链接即可。
```bash
# Ubuntu
sh config_brpc.sh --headers=/usr/include --libs=/usr/lib --with-thrift
# Fedora/CentOS
sh config_brpc.sh --headers=/usr/include --libs=/usr/lib64 --with-thrift
# Or use cmake
mkdir build && cd build && cmake ../ -DWITH_THRIFT=1
```
更多编译选项请阅读[Getting Started](../cn/getting_started.md)。

# Client端访问thrift server
基本步骤：
- 创建一个协议设置为brpc::PROTOCOL_THRIFT的Channel
- 创建brpc::ThriftStub
- 使用原生Request和原生Response>发起访问

示例代码如下：
```c++
#include <brpc/channel.h>
#include <brpc/thrift_message.h>         // 定义了ThriftStub
...

DEFINE_string(server, "0.0.0.0:8019", "IP Address of thrift server");
DEFINE_string(load_balancer, "", "The algorithm for load balancing");
...
  
brpc::ChannelOptions options;
options.protocol = brpc::PROTOCOL_THRIFT;
brpc::Channel thrift_channel;
if (thrift_channel.Init(Flags_server.c_str(), FLAGS_load_balancer.c_str(), &options) != 0) {
   LOG(ERROR) << "Fail to initialize thrift channel";
   return -1;
}

brpc::ThriftStub stub(&thrift_channel);
...

// example::[EchoRequest/EchoResponse]是thrift生成的消息
example::EchoRequest req;
example::EchoResponse res;
req.data = "hello";

stub.CallMethod("Echo", &cntl, &req, &res, NULL);

if (cntl.Failed()) {
    LOG(ERROR) << "Fail to send thrift request, " << cntl.ErrorText();
    return -1;
} 
```

# Server端处理thrift请求
用户通过继承brpc::ThriftService实现处理逻辑，既可以调用thrift生成的handler以直接复用原有的函数入口，也可以像protobuf服务那样直接读取request和设置response。
```c++
class EchoServiceImpl : public brpc::ThriftService {
public:
    void ProcessThriftFramedRequest(brpc::Controller* cntl,
                                    brpc::ThriftFramedMessage* req,
                                    brpc::ThriftFramedMessage* res,
                                    google::protobuf::Closure* done) override {
        // Dispatch calls to different methods
        if (cntl->thrift_method_name() == "Echo") {
            return Echo(cntl, req->Cast<example::EchoRequest>(),
                        res->Cast<example::EchoResponse>(), done);
        } else {
            cntl->SetFailed(brpc::ENOMETHOD, "Fail to find method=%s",
                            cntl->thrift_method_name().c_str());
            done->Run();
        }
    }

    void Echo(brpc::Controller* cntl,
              const example::EchoRequest* req,
              example::EchoResponse* res,
              google::protobuf::Closure* done) {
        // This object helps you to call done->Run() in RAII style. If you need
        // to process the request asynchronously, pass done_guard.release().
        brpc::ClosureGuard done_guard(done);

        res->data = req->data + " (processed)";
    }
};
```

实现好thrift service后，设置到ServerOptions.thrift_service并启动服务
```c++
    brpc::Server server;
    brpc::ServerOptions options;
    options.thrift_service = new EchoServiceImpl;
    options.idle_timeout_sec = FLAGS_idle_timeout_s;
    options.max_concurrency = FLAGS_max_concurrency;

    // Start the server.
    if (server.Start(FLAGS_port, &options) != 0) {
        LOG(ERROR) << "Fail to start EchoServer";
        return -1;
    }
```

# 简单的和原生thrift性能对比实验
测试环境: 48核  2.30GHz
## server端返回client发送的"hello"字符串
框架 | 线程数 | QPS | 平响 | cpu利用率
---- | --- | --- | --- | ---
native thrift | 60 | 6.9w | 0.9ms | 2.8%
brpc thrift | 60 | 30w | 0.2ms | 18%

## server端返回"hello" * 1000 字符串
框架 | 线程数 | QPS | 平响 | cpu利用率
---- | --- | --- | --- | ---
native thrift | 60 | 5.2w | 1.1ms | 4.5%
brpc thrift | 60 | 19.5w | 0.3ms | 22%

## server端做比较复杂的数学计算并返回"hello" * 1000 字符串
框架 | 线程数 | QPS | 平响 | cpu利用率
---- | --- | --- | --- | ---
native thrift | 60 | 1.7w | 3.5ms | 76%
brpc thrift | 60 | 2.1w | 2.9ms | 93%


# ========= ./timer_keeping.md ========
在几点几分做某件事是RPC框架的基本需求，这件事比看上去难。

让我们先来看看系统提供了些什么： posix系统能以[signal方式](http://man7.org/linux/man-pages/man2/timer_create.2.html)告知timer触发，不过signal逼迫我们使用全局变量，写[async-signal-safe](https://docs.oracle.com/cd/E19455-01/806-5257/gen-26/index.html)的函数，在面向用户的编程框架中，我们应当尽力避免使用signal。linux自2.6.27后能以[fd方式](http://man7.org/linux/man-pages/man2/timerfd_create.2.html)通知timer触发，这个fd可以放到epoll中和传输数据的fd统一管理。唯一问题是：这是个系统调用，且我们不清楚它在多线程下的表现。

为什么这么关注timer的开销?让我们先来看一下RPC场景下一般是怎么使用timer的：

- 在发起RPC过程中设定一个timer，在超时时间后取消还在等待中的RPC。几乎所有的RPC调用都有超时限制，都会设置这个timer。
- RPC结束前删除timer。大部分RPC都由正常返回的response导致结束，timer很少触发。

你注意到了么，在RPC中timer更像是”保险机制”，在大部分情况下都不会发挥作用，自然地我们希望它的开销越小越好。一个几乎不触发的功能需要两次系统调用似乎并不理想。那在应用框架中一般是如何实现timer的呢？谈论这个问题需要区分“单线程”和“多线程”:

- 在单线程框架中，比如以[libevent](http://libevent.org/)[, ](http://en.wikipedia.org/wiki/Reactor_pattern)[libev](http://software.schmorp.de/pkg/libev.html)为代表的eventloop类库，或以[GNU Pth](http://www.gnu.org/software/pth/pth-manual.html), [StateThreads](http://state-threads.sourceforge.net/index.html)为代表的coroutine / fiber类库中，一般是以[小顶堆](https://en.wikipedia.org/wiki/Heap_(data_structure))记录触发时间。[epoll_wait](http://man7.org/linux/man-pages/man2/epoll_wait.2.html)前以堆顶的时间计算出参数timeout的值，如果在该时间内没有其他事件，epoll_wait也会醒来，从堆中弹出已超时的元素，调用相应的回调函数。整个框架周而复始地这么运转，timer的建立，等待，删除都发生在一个线程中。只要所有的回调都是非阻塞的，且逻辑不复杂，这套机制就能提供基本准确的timer。不过就像[Threading Overview](threading_overview.md)中说的那样，这不是RPC的场景。
- 在多线程框架中，任何线程都可能被用户逻辑阻塞较长的时间，我们需要独立的线程实现timer，这种线程我们叫它TimerThread。一个非常自然的做法，就是使用用锁保护的小顶堆。当一个线程需要创建timer时，它先获得锁，然后把对应的时间插入堆，如果插入的元素成为了最早的，唤醒TimerThread。TimerThread中的逻辑和单线程类似，就是等着堆顶的元素超时，如果在等待过程中有更早的时间插入了，自己会被插入线程唤醒，而不会睡过头。这个方法的问题在于每个timer都需要竞争一把全局锁，操作一个全局小顶堆，就像在其他文章中反复谈到的那样，这会触发cache bouncing。同样数量的timer操作比单线程下的慢10倍是非常正常的，尴尬的是这些timer基本不触发。

我们重点谈怎么解决多线程下的问题。

一个惯例思路是把timer的需求散列到多个TimerThread，但这对TimerThread效果不好。注意我们上面提及到了那个“制约因素”：一旦插入的元素是最早的，要唤醒TimerThread。假设TimerThread足够多，以至于每个timer都散列到独立的TimerThread，那么每次它都要唤醒那个TimerThread。 “唤醒”意味着触发linux的调度函数，触发上下文切换。在非常流畅的系统中，这个开销大约是3-5微秒，这可比抢锁和同步cache还慢。这个因素是提高TimerThread扩展性的一个难点。多个TimerThread减少了对单个小顶堆的竞争压力，但同时也引入了更多唤醒。

另一个难点是删除。一般用id指代一个Timer。通过这个id删除Timer有两种方式：1.抢锁，通过一个map查到对应timer在小顶堆中的位置，定点删除，这个map要和堆同步维护。2.通过id找到Timer的内存结构，做个标记，留待TimerThread自行发现和删除。第一种方法让插入逻辑更复杂了，删除也要抢锁，线程竞争更激烈。第二种方法在小顶堆内留了一大堆已删除的元素，让堆明显变大，插入和删除都变慢。

第三个难点是TimerThread不应该经常醒。一个极端是TimerThread永远醒着或以较高频率醒过来（比如每1ms醒一次），这样插入timer的线程就不用负责唤醒了，然后我们把插入请求散列到多个堆降低竞争，问题看似解决了。但事实上这个方案提供的timer精度较差，一般高于2ms。你得想这个TimerThread怎么写逻辑，它是没法按堆顶元素的时间等待的，由于插入线程不唤醒，一旦有更早的元素插入，TimerThread就会睡过头。它唯一能做的是睡眠固定的时间，但这和现代OS scheduler的假设冲突：频繁sleep的线程的优先级最低。在linux下的结果就是，即使只sleep很短的时间，最终醒过来也可能超过2ms，因为在OS看来，这个线程不重要。一个高精度的TimerThread有唤醒机制，而不是定期醒。

另外，更并发的数据结构也难以奏效，感兴趣的同学可以去搜索"concurrent priority queue"或"concurrent skip list"，这些数据结构一般假设插入的数值较为散开，所以可以同时修改结构内的不同部分。但这在RPC场景中也不成立，相互竞争的线程设定的时间往往聚集在同一个区域，因为程序的超时大都是一个值，加上当前时间后都差不多。

这些因素让TimerThread的设计相当棘手。由于大部分用户的qps较低，不足以明显暴露这个扩展性问题，在r31791前我们一直沿用“用一把锁保护的TimerThread”。TimerThread是brpc在默认配置下唯一的高频竞争点，这个问题是我们一直清楚的技术债。随着brpc在高qps系统中应用越来越多，是时候解决这个问题了。r31791后的TimerThread解决了上述三个难点，timer操作几乎对RPC性能没有影响，我们先看下性能差异。

> 在示例程序example/mutli_threaded_echo_c++中，r31791后TimerThread相比老TimerThread在24核E5-2620上（超线程），以50个bthread同步发送时，节省4%cpu（差不多1个核），qps提升10%左右；在400个bthread同步发送时，qps从30万上升到60万。新TimerThread的表现和完全关闭超时时接近。

那新TimerThread是如何做到的？

- 一个TimerThread而不是多个。
- 创建的timer散列到多个Bucket以降低线程间的竞争，默认12个Bucket。
- Bucket内不使用小顶堆管理时间，而是链表 + nearest_run_time字段，当插入的时间早于nearest_run_time时覆盖这个字段，之后去和全局nearest_run_time（和Bucket的nearest_run_time不同）比较，如果也早于这个时间，修改并唤醒TimerThread。链表节点在锁外使用[ResourcePool](memory_management.md)分配。
- 删除时通过id直接定位到timer内存结构，修改一个标志，timer结构总是由TimerThread释放。
- TimerThread被唤醒后首先把全局nearest_run_time设置为几乎无限大(max of int64)，然后取出所有Bucket内的链表，并把Bucket的nearest_run_time设置为几乎无限大(max of int64)。TimerThread把未删除的timer插入小顶堆中维护，这个堆就它一个线程用。在每次运行回调或准备睡眠前都会检查全局nearest_run_time， 如果全局更早，说明有更早的时间加入了，重复这个过程。

这里勾勒了TimerThread的大致工作原理，工程实现中还有不少细节问题，具体请阅读[timer_thread.h](https://github.com/brpc/brpc/blob/master/src/bthread/timer_thread.h)和[timer_thread.cpp](https://github.com/brpc/brpc/blob/master/src/bthread/timer_thread.cpp)。

这个方法之所以有效：

- Bucket锁内的操作是O(1)的，就是插入一个链表节点，临界区很小。节点本身的内存分配是在锁外的。
- 由于大部分插入的时间是递增的，早于Bucket::nearest_run_time而参与全局竞争的timer很少。
- 参与全局竞争的timer也就是和全局nearest_run_time比一下，临界区很小。
- 和Bucket内类似，极少数Timer会早于全局nearest_run_time并去唤醒TimerThread。唤醒也在全局锁外。
- 删除不参与全局竞争。
- TimerThread自己维护小顶堆，没有任何cache bouncing，效率很高。 
- TimerThread醒来的频率大约是RPC超时的倒数，比如超时=100ms，TimerThread一秒内大约醒10次，已经最优。

至此brpc在默认配置下不再有全局竞争点，在400个线程同时运行时，profiling也显示几乎没有对锁的等待。

下面是一些和linux下时间管理相关的知识：

- epoll_wait的超时精度是毫秒，较差。pthread_cond_timedwait的超时使用timespec，精度到纳秒，一般是60微秒左右的延时。
- 出于性能考虑，TimerThread使用wall-time，而不是单调时间，可能受到系统时间调整的影响。具体来说，如果在测试中把系统时间往前或往后调一个小时，程序行为将完全undefined。未来可能会让用户选择单调时间。
- 在cpu支持nonstop_tsc和constant_tsc的机器上，brpc和bthread会优先使用基于rdtsc的cpuwide_time_us。那两个flag表示rdtsc可作为wall-time使用，不支持的机器上会转而使用较慢的内核时间。我们的机器（Intel Xeon系列）大都有那两个flag。rdtsc作为wall-time使用时是否会受到系统调整时间的影响，未测试不清楚。


# ========= ./ub_client.md ========
brpc可通过多种方式访问用ub搭建的服务。

# ubrpc (by protobuf)

r31687后，brpc支持通过protobuf访问ubrpc，不需要baidu-rpc-ub，也不依赖idl-compiler。（也可以让protobuf服务被ubrpc client访问，方法见[使用ubrpc的服务](nshead_service.md#使用ubrpc的服务)）。

**步骤：**

1. 用[idl2proto](https://github.com/brpc/brpc/blob/master/tools/idl2proto)把idl文件转化为proto文件，老版本idl2proto不会转化idl中的service，需要手动转化。

   ```protobuf
   // Converted from echo.idl by brpc/tools/idl2proto
   import "idl_options.proto";
   option (idl_support) = true;
   option cc_generic_services = true;
   message EchoRequest {
     required string message = 1; 
   }
   message EchoResponse {
     required string message = 1; 
   }
    
   // 对于idl中多个request或response的方法，要建立一个包含所有request或response的消息。
   // 这个例子中就是MultiRequests和MultiResponses。
   message MultiRequests {
     required EchoRequest req1 = 1;
     required EchoRequest req2 = 2;
   }
   message MultiResponses {
     required EchoRequest res1 = 1;
     required EchoRequest res2 = 2;
   }
    
   service EchoService {
     // 对应idl中的void Echo(EchoRequest req, out EchoResponse res);
     rpc Echo(EchoRequest) returns (EchoResponse);
    
     // 对应idl中的uint32_t EchoWithMultiArgs(EchoRequest req1, EchoRequest req2, out EchoResponse res1, out EchoResponse res2);
     rpc EchoWithMultiArgs(MultiRequests) returns (MultiResponses);
   }
   ```

   原先的echo.idl文件：

   ```idl
   struct EchoRequest {
       string message;
   };
    
   struct EchoResponse {
       string message;
   };
    
   service EchoService {
       void Echo(EchoRequest req, out EchoResponse res);
       uint32_t EchoWithMultiArgs(EchoRequest req1, EchoRequest req2, out EchoResponse res1, out EchoResponse res2);
   };
   ```

2. 插入如下片段以使用代码生成插件。

   BRPC_PATH代表brpc产出的路径（包含bin include等目录），PROTOBUF_INCLUDE_PATH代表protobuf的包含路径。注意--mcpack_out要和--cpp_out一致。

   ```shell
   protoc --plugin=protoc-gen-mcpack=$BRPC_PATH/bin/protoc-gen-mcpack --cpp_out=. --mcpack_out=. --proto_path=$BRPC_PATH/include --proto_path=PROTOBUF_INCLUDE_PATH
   ```

3. 用channel发起访问。

   idl不同于pb，允许有多个请求，我们先看只有一个请求的情况，和普通的pb访问基本上是一样的。

   ```c++
   #include <brpc/channel.h>
   #include "echo.pb.h"
   ...
    
   brpc::Channel channel;
   brpc::ChannelOptions opt;
   opt.protocol = brpc::PROTOCOL_UBRPC_COMPACK; // or "ubrpc_compack";
   if (channel.Init(..., &opt) != 0) {
       LOG(ERROR) << "Fail to init channel";
       return -1;
   }
   EchoService_Stub stub(&channel);
   ...
    
   EchoRequest request;
   EchoResponse response;
   brpc::Controller cntl;
    
   request.set_message("hello world");
    
   stub.Echo(&cntl, &request, &response, NULL);
        
   if (cntl.Failed()) {
       LOG(ERROR) << "Fail to send request, " << cntl.ErrorText();
       return;
   }
   // 取response中的字段
   // [idl] void Echo(EchoRequest req, out EchoResponse res);
   //                              ^ 
   //                  response.message();
   ```

   多个请求要设置一下set_idl_names。

   ```c++
   #include <brpc/channel.h>
   #include "echo.pb.h"
   ...
    
   brpc::Channel channel;
   brpc::ChannelOptions opt;
   opt.protocol = brpc::PROTOCOL_UBRPC_COMPACK; // or "ubrpc_compack";
   if (channel.Init(..., &opt) != 0) {
       LOG(ERROR) << "Fail to init channel";
       return -1;
   }
   EchoService_Stub stub(&channel);
   ...
    
   MultiRequests multi_requests;
   MultiResponses multi_responses;
   brpc::Controller cntl;
    
   multi_requests.mutable_req1()->set_message("hello");
   multi_requests.mutable_req2()->set_message("world");
   cntl.set_idl_names(brpc::idl_multi_req_multi_res);
   stub.EchoWithMultiArgs(&cntl, &multi_requests, &multi_responses, NULL);
        
   if (cntl.Failed()) {
       LOG(ERROR) << "Fail to send request, " << cntl.ErrorText();
       return;
   }
   // 取response中的字段
   // [idl] uint32_t EchoWithMultiArgs(EchoRequest req1, EchoRequest req2,
   //        ^                         out EchoResponse res1, out EchoResponse res2);
   //        |                                           ^                      ^
   //        |                          multi_responses.res1().message();       |
   //        |                                                 multi_responses.res2().message();
   // cntl.idl_result();
   ```

   例子详见[example/echo_c++_ubrpc_compack](https://github.com/brpc/brpc/blob/master/example/echo_c++_ubrpc_compack/)。

# ubrpc (by baidu-rpc-ub)

server端由public/ubrpc搭建，request/response使用idl文件描述字段，序列化格式是compack或mcpack_v2。

**步骤：**

1. 依赖public/baidu-rpc-ub模块，这个模块是brpc的扩展，不需要的用户不会依赖idl/mcpack/compack等模块。baidu-rpc-ub只包含扩展代码，brpc中的新特性会自动体现在这个模块中。

2. 编写一个proto文件，其中定义了service，名字和idl中的相同，但请求类型必须是baidu.rpc.UBRequest，回复类型必须是baidu.rpc.UBResponse。这两个类型定义在brpc/ub.proto中，使用时得import。

   ```protobuf
   import "brpc/ub.proto";              // UBRequest, UBResponse
   option cc_generic_services = true;
   // Define UB service. request/response must be UBRequest/UBResponse
   service EchoService {
       rpc Echo(baidu.rpc.UBRequest) returns (baidu.rpc.UBResponse);
   };
   ```

3. 在COMAKE包含baidu-rpc-ub/src路径。

   ```python
   # brpc/ub.proto的包含路径
   PROTOC(ENV.WorkRoot()+"third-64/protobuf/bin/protoc")
   PROTOFLAGS("--proto_path=" + ENV.WorkRoot() + "public/baidu-rpc-ub/src/")
   ```

4. 用法和访问其他协议类似：创建Channel，ChannelOptions.protocol为**brpc::PROTOCOL_NSHEAD_CLIENT**或**"nshead_client"**。request和response对象必须是baidu-rpc-ub提供的类型

   ```c++
   #include <brpc/ub_call.h>
   ...
       
   brpc::Channel channel;
   brpc::ChannelOptions opt;
   opt.protocol = brpc::PROTOCOL_NSHEAD_CLIENT; // or "nshead_client";
   if (channel.Init(..., &opt) != 0) {
       LOG(ERROR) << "Fail to init channel";
       return -1;
   }
   EchoService_Stub stub(&channel);    
   ...
    
   const int BUFSIZE = 1024 * 1024;  // 1M
   char* buf_for_mempool = new char[BUFSIZE];
   bsl::xmempool pool;
   if (pool.create(buf_for_mempool, BUFSIZE) != 0) {
       LOG(FATAL) << "Fail to create bsl::xmempool";
       return -1;
   }
    
   // 构造UBRPC的request/response，idl结构体作为模块参数传入。为了构造idl结构，需要传入一个bsl::mempool
   brpc::UBRPCCompackRequest<example::EchoService_Echo_params> request(&pool);
   brpc::UBRPCCompackResponse<example::EchoService_Echo_response> response(&pool);
    
   // 设置字段
   request.mutable_req()->set_message("hello world");
    
   // 发起RPC
   brpc::Controller cntl;
   stub.Echo(&cntl, &request, &response, NULL);
       
   if (cntl.Failed()) {
       LOG(ERROR) << "Fail to Echo, " << cntl.ErrorText();
       return;
   }
   // 取回复中的字段
   response.result_params().res().message();
   ...
   ```

   具体example代码可以参考[echo_c++_compack_ubrpc](https://github.com/brpc/brpc/tree/master/example/echo_c++_compack_ubrpc/)，类似的还有[echo_c++_mcpack_ubrpc](https://github.com/brpc/brpc/tree/master/example/echo_c++_mcpack_ubrpc/)。

# nshead+idl

server端是由public/ub搭建，通讯包组成为nshead+idl::compack/idl::mcpack(v2)

由于不需要指定service和method，无需编写proto文件，直接使用Channel.CallMethod方法发起RPC即可。请求包中的nshead可以填也可以不填，框架会补上正确的magic_num和body_len字段：

```c++
#include <brpc/ub_call.h>
...
 
brpc::Channel channel;
brpc::ChannelOptions opt;
opt.protocol = brpc::PROTOCOL_NSHEAD_CLIENT; // or "nshead_client";
 
if (channel.Init(..., &opt) != 0) {
    LOG(ERROR) << "Fail to init channel";
    return -1;
}
...
 
// 构造UB的request/response，完全类似构造原先idl结构，传入一个bsl::mempool（变量pool）
// 将类型作为模板传入，之后在使用上可以直接使用对应idl结构的接口
brpc::UBCompackRequest<example::EchoRequest> request(&pool);
brpc::UBCompackResponse<example::EchoResponse> response(&pool);
 
// Set `message' field of `EchoRequest'
request.set_message("hello world");
// Set fields of the request nshead struct if needed
request.mutable_nshead()->version = 99;
 
brpc::Controller cntl;
channel.CallMethod(NULL, &cntl, &request, &response, NULL);    // 假设channel已经通过之前所述方法Init成功
 
// Get `message' field of `EchoResponse'
response.message();
```

具体example代码可以参考[echo_c++_mcpack_ub](https://github.com/brpc/brpc/blob/master/example/echo_c++_mcpack_ub/)，compack情况类似，不再赘述

# nshead+mcpack(非idl产生的)

server端是由public/ub搭建，通讯包组成为nshead+mcpack包，但不是idl编译器生成的，RPC前需要先构造RawBuffer将其传入，然后获取mc_pack_t并按之前手工填写mcpack的方式操作：

```c++
#include <brpc/ub_call.h>
...
 
brpc::Channel channel;
brpc::ChannelOptions opt;
opt.protocol = brpc::PROTOCOL_NSHEAD_CLIENT; // or "nshead_client";
if (channel.Init(..., &opt) != 0) {
    LOG(ERROR) << "Fail to init channel";
    return -1;
}
...
 
// 构造RawBuffer，一次RPC结束后RawBuffer可以复用，类似于bsl::mempool
const int BUFSIZE = 10 * 1024 * 1024;
brpc::RawBuffer req_buf(BUFSIZE);
brpc::RawBuffer res_buf(BUFSIZE);
 
// 传入RawBuffer来构造request和response
brpc::UBRawMcpackRequest request(&req_buf);
brpc::UBRawMcpackResponse response(&res_buf);
         
// Fetch mc_pack_t and fill in variables
mc_pack_t* req_pack = request.McpackHandle();
int ret = mc_pack_put_str(req_pack, "mystr", "hello world");
if (ret != 0) {
    LOG(FATAL) << "Failed to put string into mcpack: "
               << mc_pack_perror((long)req_pack) << (void*)req_pack;
    break;
}  
// Set fields of the request nshead struct if needed
request.mutable_nshead()->version = 99;
 
brpc::Controller cntl;
channel.CallMethod(NULL, &cntl, &request, &response, NULL);    // 假设channel已经通过之前所述方法Init成功
 
// Get response from response buffer
const mc_pack_t* res_pack = response.McpackHandle();
mc_pack_get_str(res_pack, "mystr");
```

具体example代码可以参考[echo_c++_raw_mcpack](https://github.com/brpc/brpc/blob/master/example/echo_c++_raw_mcpack/)。

# nshead+blob

r32897后brpc直接支持用nshead+blob访问老server（而不用依赖baidu-rpc-ub）。example代码可以参考[nshead_extension_c++](https://github.com/brpc/brpc/blob/master/example/nshead_extension_c++/client.cpp)。

```c++
#include <brpc/nshead_message.h>
...
 
brpc::Channel;
brpc::ChannelOptions opt;
opt.protocol = brpc::PROTOCOL_NSHEAD; // or "nshead"
if (channel.Init(..., &opt) != 0) {
    LOG(ERROR) << "Fail to init channel";
    return -1;
} 
...
brpc::NsheadMessage request;
brpc::NsheadMessage response;
       
// Append message to `request'
request.body.append("hello world");
// Set fields of the request nshead struct if needed
request.head.version = 99;
 
 
brpc::Controller cntl;
channel.CallMethod(NULL, &cntl, &request, &response, NULL);
 
if (cntl.Failed()) {
    LOG(ERROR) << "Fail to access the server: " << cntl.ErrorText();
    return -1;
}
// response.head and response.body contains nshead_t and blob respectively.
```

或者用户也可以使用baidu-rpc-ub中的UBRawBufferRequest和UBRawBufferResponse来访问。example代码可以参考[echo_c++_raw_buffer](https://github.com/brpc/brpc/blob/master/example/echo_c++_raw_buffer/)。

```c++
brpc::Channel channel;
brpc::ChannelOptions opt;
opt.protocol = brpc::PROTOCOL_NSHEAD_CLIENT; // or "nshead_client"
if (channel.Init(..., &opt) != 0) {
    LOG(ERROR) << "Fail to init channel";
    return -1;
}
...
 
// 构造RawBuffer，一次RPC结束后RawBuffer可以复用，类似于bsl::mempool
const int BUFSIZE = 10 * 1024 * 1024;
brpc::RawBuffer req_buf(BUFSIZE);
brpc::RawBuffer res_buf(BUFSIZE);
 
// 传入RawBuffer来构造request和response
brpc::UBRawBufferRequest request(&req_buf);
brpc::UBRawBufferResponse response(&res_buf);
         
// Append message to `request'
request.append("hello world");
// Set fields of the request nshead struct if needed
request.mutable_nshead()->version = 99;
 
brpc::Controller cntl;
channel.CallMethod(NULL, &cntl, &request, &response, NULL);    // 假设channel已经通过之前所述方法Init成功
 
// Process response. response.data() is the buffer, response.size() is the length.
```


# ========= ./vars.md ========
[English version](../en/vars.md)

[bvar](https://github.com/brpc/brpc/tree/master/src/bvar/)是多线程环境下的计数器类库，方便记录和查看用户程序中的各类数值，它利用了thread local存储减少了cache bouncing，相比UbMonitor(百度内的老计数器库)几乎不会给程序增加性能开销，也快于竞争频繁的原子操作。brpc集成了bvar，[/vars](http://brpc.baidu.com:8765/vars)可查看所有曝光的bvar，[/vars/VARNAME](http://brpc.baidu.com:8765/vars/rpc_socket_count)可查阅某个bvar，增加计数器的方法请查看[bvar](bvar.md)。brpc大量使用了bvar提供统计数值，当你需要在多线程环境中计数并展现时，应该第一时间想到bvar。但bvar不能代替所有的计数器，它的本质是把写时的竞争转移到了读：读得合并所有写过的线程中的数据，而不可避免地变慢了。当你读写都很频繁或得基于最新值做一些逻辑判断时，你不应该用bvar。

## 查询方法

[/vars](http://brpc.baidu.com:8765/vars) : 列出所有曝光的bvar

[/vars/NAME](http://brpc.baidu.com:8765/vars/rpc_socket_count)：查询名字为NAME的bvar

[/vars/NAME1,NAME2,NAME3](http://brpc.baidu.com:8765/vars/pid;process_cpu_usage;rpc_controller_count)：查询名字为NAME1或NAME2或NAME3的bvar

[/vars/foo*,b$r](http://brpc.baidu.com:8765/vars/rpc_server*_count;iobuf_blo$k_*)：查询名字与某一统配符匹配的bvar，注意用$代替?匹配单个字符，因为?是URL的保留字符。

以下动画演示了如何使用过滤功能。你可以把包含过滤表达式的url复制粘贴给他人，他们点开后将看到相同的计数器条目。(数值可能随运行变化)

![img](../images/vars_1.gif)

/vars左上角有一个搜索框能加快寻找特定bvar的速度，在这个搜索框你只需键入bvar名称的一部分，框架将补上*进行模糊查找。不同的名称间可以逗号、分号或空格分隔。

![img](../images/vars_2.gif)

你也可以在命令行中访问vars：

```
$ curl brpc.baidu.com:8765/vars/bthread*
bthread_creation_count : 125134
bthread_creation_latency : 3
bthread_creation_latency_50 : 3
bthread_creation_latency_90 : 5
bthread_creation_latency_99 : 7
bthread_creation_latency_999 : 12
bthread_creation_latency_9999 : 12
bthread_creation_latency_cdf : "click to view"
bthread_creation_latency_percentiles : "[3,5,7,12]"
bthread_creation_max_latency : 7
bthread_creation_qps : 100
bthread_group_status : "0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 "
bthread_num_workers : 24
bthread_worker_usage : 1.01056
```

## 查看历史趋势

点击大部分数值型的bvar会显示其历史趋势。每个可点击的bvar记录了过去60秒，60分钟，24小时，30天总计174个值。当有1000个可点击bvar时大约会占1M内存。

![img](../images/vars_3.gif)

## 统计和查看分位值

x%分位值（percentile）是指把一段时间内的N个统计值排序，排在第N * x%位的值。比如一段时间内有1000个值，先从小到大排序，排在第500位(1000 * 50%)的值是50%分位值（即中位数），排在第990位的是99%分位值(1000 * 99%)，排在第999位的是99.9%分位值。分位值能比平均值更准确的刻画数值的分布，对理解系统行为有重要意义。工业级应用的SLA一般在99.97%以上(此为百度对二级系统的要求，一级是99.99%以上)，一些系统即使平均值不错，但不佳的长尾区域也会明显拉低和打破SLA。分位值能帮助分析长尾区域。

分位值可以绘制为CDF曲线和按时间变化的曲线。

**下图是分位值的CDF**，横轴是比例(排序位置/总数)，纵轴是对应的分位值。比如横轴=50%处对应的纵轴值便是50%分位值。如果系统要求的性能指标是"99.9%的请求在xx毫秒内完成“，那么你就得看下99.9%那儿的值。

![img](../images/vars_4.png)

为什么叫它[CDF](https://en.wikipedia.org/wiki/Cumulative_distribution_function)? 当选定一个纵轴值y时，对应横轴的含义是"数值 <= y的比例”，由于数值一般来自随机采样，横轴也可以理解为“数值 <= y的概率”，或P(数值 <= y)，这就是CDF的定义。

CDF的导数是[概率密度函数](https://en.wikipedia.org/wiki/Probability_density_function)。如果把CDF的纵轴分为很多小段，对每个小段计算两端对应的横轴值之差，并把这个差作为新的横轴，那么我们便绘制了PDF曲线，就像顺时针旋转了90度的正态分布那样。但中位数的密度往往很高，在PDF中很醒目，这使得边上的长尾很扁而不易查看，所以大部分系统测量结果选择CDF曲线而不是PDF曲线。

可用一些简单规则衡量CDF曲线好坏：

- 越平越好。一条水平线是最理想的，这意味着所有的数值都相等，没有任何等待，拥塞，停顿。当然这是不可能的。
- 99%和100%间的面积越小越好：99%之后是长尾的聚集地，对大部分系统的SLA有重要影响。

一条缓慢上升且长尾区域面积不大的CDF便是不错的曲线。

**下图是按分位值按时间变化的曲线**，包含了4条曲线，横轴是时间，纵轴从上到下分别对应99.9%，99%，90%，50%分位值。颜色从上到下也越来越浅（从橘红到土黄）。

![img](../images/vars_5.png)

滑动鼠标可以阅读对应数据点的值，上图中显示的是”39秒种前的99%分位值是330**微秒**”。这幅图中不包含99.99%的曲线，因为99.99%分位值常明显大于99.9%及以下的分位值，画在一起的话会使得其他曲线变得很”矮“，难以辨认。你可以点击以"\_latency\_9999"结尾的bvar独立查看99.99%曲线。按时间变化曲线可以看到分位值的变化趋势，对分析系统的性能变化很实用。

brpc的服务都会自动统计延时分布，用户不用自己加了。如下图所示：

![img](../images/vars_6.png)

你可以用bvar::LatencyRecorder统计任何代码的延时，这么做(更具体的使用方法请查看[bvar-c++](bvar_c++.md)):

```c++
#include <bvar/bvar.h>

...
bvar::LatencyRecorder g_latency_recorder("client");  // expose this recorder
... 
void foo() {
    ...
    g_latency_recorder << my_latency;
    ...
}
```

如果这个程序使用了brpc server，那么你应该已经可以在/vars看到client_latency, client_latency_cdf等变量，点击便可查看动态曲线。如下图所示：

![img](../images/vars_7.png)

## 非brpc server

如果你的程序只是一个brpc client或根本没有使用brpc，并且你也想看到动态曲线，看[这里](dummy_server.md)。
