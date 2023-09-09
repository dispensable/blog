---
layout: post
title: 记一次golang内存泄露的排查
tags: 
- golang
- gobeansproxy
- douban
- memory leak
---

# 起

最近一直在做beansproxy支持cassandra双写的工作。主要目的是为了迁移已经无人看管的beansdb集群。在很小心的测试之后上线了一点新服务，结果一天过去内存飙升，更是出现了一些正确性问题的反馈，于是开始了一场痛苦的排查之旅。

还是简单介绍一下beansdb吧。无主架构，NRW和merkle tree保持最终一致性。proxy无状态作为集群的schduler根据hash算法和路由去查找对应的key，并返回给client。

# 承

beansdb有c的实现，go的实现，不过都用了一个底层的clib，libmc来提供mc协议访问。在go实现中，使用cgo绑定了libmc的实现，所以读写数据流都会通过mc协议最终封装成libmc的结构体暴露给proxy和server使用。所以遇到内存泄露的时候我第一时间想到的就是应该是哪里没有释放c申请的内存。但是纵观最近的修改，因为还没开始测试c\*的部分，所有的逻辑都和之前的逻辑一致，代码大体是一致的。而且从添加的指标来看c\*的逻辑完全没有使用过，不过泄露真实发生了，从代码上看没出什么。

那就用`go pprof`来查看下内存的使用吧，正好gobeansproxy开启了http的pprof，非常方便就可以inspect线上的内存。然而观察到的是alloc很多，inuse的却很少。从显示的申请部分来看，都是标准库的代码，这肯定是我的问题，不可能是go本身的问题。联系到cgo的内存并不会出现在`pprof`上，所以只能帮我们确认不是gc管理的内存出现了泄露。

因为看代码逻辑没找到什么明显的问题，本地的测试环境也无法复现这些问题。所以想看泄露的内存里到底存的是什么东西呢？这时候gdb上场了，使用gdb的`generate-core-file`指令可以方便的dump一个进程的内存。使用方式如下：

```bash
sudo gdb attach $(pidof gobeansproxy)
gdb> generate-core-file

hexdump -C core.$(pidof gobeansproxy) | less
```

非常不幸，dump的内存里就是我们预想到的key/value内容。但是问题就是这东西在整个repo里都是，无法帮我们缩小代码的范围。不过这告诉我们泄露的内存应该高度可能是cgo相关的，因为value的值都出现在cgo管理的内存（大部分）里。

走到这里真的是很痛苦了，因为这个泄露的速度比较慢，而且进程本身在重启之后也会有一个内存上涨的过程，导致的结果就是我们修改之后需要很长的时间才能复现问题。于此同时出现了一些正确性方面的问题反馈，只能暂时把服务下线。

# 转

事情出现了转机。

仔细分析了这几天的监控之后发发现，有几个时间段的内存上涨的非常快，几乎是陡峭的直线。现在就想办法在本地环境里复现这个才能继续追查了。找到那几个时间点的日志，写脚本重放请求（get/getm请求都是幂等的）。但是在本地环境里没有复现，内存很稳定。

只能找个生产环境去试试看了，正好有一个备份的节点没有流量，更新-重放-查看内存，终于复现了。。。进一步的筛选发现主要是getm这个操作，且集中在一些key的值非常大的情况下。

继续查看getm的代码（之前从未想过可能是这里，重点都在看set的逻辑，因为set的value的内存是proxy需要负责释放的），但是还是没找到问题在哪里。但是这一定是逻辑问题了，diff代码之后，一半一半的注释代码，终于找到了问题。（找问题非常困难，找到了就发现是非常愚蠢的错误）

```go
bkeys, ckeys := c.pswitcher.ReadEnableOnKeys(keys)
rs = make(map[string]*mc.Item, len(keys))
	
if len(bkeys) > 0 {
  c.sched = GetScheduler()
  var lock sync.Mutex
  gs := c.sched.DivideKeysByBucket(bkeys)
  reply := make(chan bool, len(gs))
  for _, ks := range gs {
    if len(ks) > 0 {
      go func(keys []string) {
        r, t, e := c.getMulti(bkeys)
        if e != nil {
          err = e
        } else {
          for k, v := range r {
            lock.Lock()
            rs[k] = v
            c.SuccessedTargets = append(c.SuccessedTargets, t...)
            lock.Unlock()
          }
        }
        reply <- true
      }(ks)
    } else {
      reply <- true
    }
  }
```

能看出来哪里有问题么？

.

.

.

是了，`r, t, e := c.getMulti(bkeys)` 这行出现了问题，在这个go func里，arg的名字是`keys`，这里本来该写keys，却写成了bkeys。导致本来应该根据bucket分区的keys的子集变成了keys的全集。由于闭包变量捕获的原因，编译器并不会报错。由于beansdb的集群设计，不同的bucket存的key范围是不一样的。本来应该找到对应的机器去请求，结果之前的bucket分区完全没有用上。返回的结果里（结果会有cgo分配的内存）有重复的值，更新到map中的时候，如果值已经在map内，那替换之前应该释放老的value分配的内存。直接添加进去就无人再引用老value的cgo值了，内存就无法释放了。

bkeys/ckeys是为了支持getm操作能够按照前缀分别从beansdb和c\*中取值，结果在复制bdb逻辑的时候，直接替换keys参数成bkeys酿成了这样的惨剧。由于bucket分区的逻辑完全没用上，在不存在这个key的host上查找，返回未找到（其实存在）进而导致了正确性问题。

# 合

知道原因来看结果，这个也真的是够蠢的。通过这个我也学到了很多：

1. `for` `go func() {}`的写法很爽，闭包变量捕获也让大家写代码的时候很舒服。但是这是有代价的，代价就是本来应该通过参数传递的清晰的逻辑关系被闭包所模糊。相信大家也遇到过变量捕获中同名变量的问题吧，没遇到的可以看看： [here](https://eli.thegreenplace.net/2019/go-internals-capturing-loop-variables-in-closures/)
2. 好的代码习惯很重要，`if` `else`的代码写的爽，觉得自己加一点逻辑没必要拆分方法。其实这里如果拆分了两个方法，那根本不会遇到这个问题了。
3. 内存问题很难调试，在beansdb的实现中，value的值只有比较大的时候（看配置）才会动用c内存分配，小的value会依然使用go的内存分配。这时候就会有很奇怪的结果，你的内存可能因为访问模式的原因并不会泄露（访问小key的时候因为内存是gc负责的，所以根本看不到泄露。只有c内存才会有这种问题），但是某个时间却泄露很多。
4. 复现问题比偷懒想通过pprof/gdb/bpf/valgrind直接寻找答案优先级更高，观察问题中的模式，去掉那些干扰信息，能复现就能有好的解决。
5. 不要想当然的猜问题，你觉得是那里的问题可能根本不是（比如我在set/get方法上绕了很久很久）
6. 本地的环境尽量和线上一致点。本地为什么无法复现这个问题，本地的数据太小了，走的都是go的gc。正确性上看，本地只有三个点，无论怎么分key都会分到这三个点上，不会出现生产集群那种bucket分布的。
7. `There are only two hard things in Computer Science: cache invalidation and naming things. -- Phil Karlton` 好好起名！

唉，就记录到这里，我去写脚本修复那些被我搞坏的数据了 :(
