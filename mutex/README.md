# Mutex
## 概念
Mutex是使用最广泛的同步原因，我们称之为`互斥锁`或者`排他锁`，通常我们使用Mutex、RWMutex对共享资源进行保护，解决资源竞争的问题。
## 用法
下面的代码中展示了一个计数器计数的例子，例子中创建了10个goroutine，每个goroutine中对计数器进行100000次的+1操作，我们期望最后的结果为10* 100000= 1000000。

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var counter int
    
    var wg sync.WaitGroup
    wg.Add(10)
    
    for i := 0; i < 10; i++ {
        go func(){
			defer wg.Done()
            for j := 0; j < 100000; j++ {
                counter++
            }
        }()
    }
    
    wg.Wait()
    
    fmt.Println(counter)
}
```

在 https://tour.go-zh.org/ 里执行代码，执行结果为516264，并不是我们期望的1000000。原因是因为`counter++`并不是一个原子操作，它包含：

1. 读取counter当前值
2. 当前值+1
3. 结果保存到counter

比如，10个goroutine同时读到counter的值是9，按照相同的逻辑+1，然后保存到变量身上。counter增加的总数应该是10，但是实际上结果增加了1。

修改一下代码，引入`sync.Mutex`对非原子操作的`counter++`进行保护，同时只能有一个goroutine对counter进行`counter++`的操作：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var counter int
    
    var mu sync.Mutex
    var wg sync.WaitGroup
    wg.Add(10)
    
    for i := 0; i < 10; i++ {
        go func(){
			defer wg.Done()
            for j := 0; j < 100000; j++ {
				mu.Lock()
                counter++
				mu.Unlock()
            }
        }()
    }
    
    wg.Wait()
    
    fmt.Println(counter)
}
```
这次的结果是我们期望的1000000。上面的代码就是典型的`sync.Mutex`应用场景，对共享资源进行保护。  
我们还可以把Mutex嵌入到结构体中:

```go
package main

import (
    "fmt"
    "sync"
)

type Counter struct {
	mu sync.Mutex
	val int
}

func (c *Counter) incr() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.val++
}

func (c *Counter) count() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	
	return c.val
}


func main() {
	var counter Counter
    
    var wg sync.WaitGroup
    wg.Add(10)
    
    for i := 0; i < 10; i++ {
        go func(){
			defer wg.Done()
            for j := 0; j < 100000; j++ {
				counter.incr()
            }
        }()
    }
    
    wg.Wait()
    
    fmt.Println(counter.count())
}
```

同时，我们可以修改`Counter`结构体部分的代码，让代码看起来更简洁: 

```go
type Counter struct {
    sync.Mutex
    val int
}

func (c *Counter) incr() {
    c.Lock()
    defer c.Unlock()
    c.val++
}

func (c *Counter) count() int {
	c.Lock()
	defer c.Unlock()
	
	return c.val
}
```

我们采用把Mutex嵌入到结构体中的方式，通常把`sync.Mutex`和被保护的属性放在一起:

```go
type Counter struct {
    AttrA int
    AttrB int
	
    sync.Mutex
    val int
}
```

## 原理
Mutex的源码地址： https://github.com/golang/go/blob/master/src/sync/mutex.go  
`package sync`定义了一个`Locker`的接口，`Mutex`实现了这个接口: 

### Locker接口
```go
type Locker interface {
	Lock()
	Unlock()
}
```
这个接口的方法集很简单，就是请求锁和释放锁。简单来说，Mutex 就提供两个方法 Lock 和 Unlock：进入临界区之前调用 Lock 方法，退出临界区的时候调用 Unlock 方法。

 Mutex给锁提供给了三种状态来控制一个goroutine释放能获取到锁的所有权。

1. mutexLocked，锁定
2. mutexWoken，唤醒
3. mutexStarving，饥饿

代码注释中解释了goroutine索取锁的过程：

锁有两种状态：正常状态和饥饿状态。  

+ 正常状态下，等待锁的goroutine都在FIFO队列中排队，FIFO中的goroutine被唤醒后需要和新加入的goroutine竞争锁的所有权。新加入的goroutine赢得竞争的可能性更大，因为它已在CPU上执行了，被唤醒后但是在锁的所有权的竞争中失败的goroutine会在队列中排在最前面。如果一个等待的goroutine超过1ms没有获取到锁，goroutine会转为饥饿模式。  
+ 饥饿模式下，锁的所有权从unlock的goroutine直接交给等待队列中的第一个。新来的goroutine将不尝试去获取锁，放在队列的尾部。如果一个锁获取到了锁，并且满足下面的任何一个条件，锁的状态转为正常状态：
	- 这个goroutine是队列中的最后一个
	- 等待的时间小于1ms


互斥锁利用三个状态标记锁的状态：  
![](https://raw.githubusercontent.com/yannyy/go-concurrency/master/mutex/stat.png)
  

```go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema  uint32
}

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota
	starvationThresholdNs = 1e6
)

```

### Lock

```go
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```

利用`atomic.CompareAndSwapInt32`比较当前锁的状态，如果锁的状态为初始状态，将锁的状态设置为`mutexLocked `标记。 如果不是初始状态，调用`Lock()`的goroutine进入阻塞状态：

```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```

### Unlock

## 错误示例
## 示例

