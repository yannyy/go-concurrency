# go并发

# 同步原语

+ Mutex
+ RWMutex
+ WaitGroup
+ Cond
+ Once
+ Pool
+ Context

# 应用场景：

+ 共享资源： 并发读写公共资源，会出现竞争（data race）的问题，Mutex, RWMutex提供对共享资源的保护
+ 任务编排： goroutine之间存在相互依赖、相互等待的顺序关系，需要对goroutine进行编排
+ 消息传递： 信息交流以及不同的 goroutine 之间的线程安全的数据交流

|应用场景| 同步原语|
|:--|:--|
|共享资源|Mutex, RWMutex|
|任务编排|WaitGroup，Channel |
|消息传递| Channel |

# 学习方法：

go并发的学习，会对每个原语分为概念、用法、原理、示例几个方面进行：

+ 概念
+ 用法：展示原语基本用法。
+ 原理：通过[Go](https://github.com/golang/go)源码理解各个原语实现的细节。
+ 示例：通过源码展示原语的用法。