# Mutex
## 概念

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

在https://tour.go-zh.org/里执行代码，执行结果为516264，并不是我们期望的1000000。原因是因为`counter++`并不是一个原子操作，它包含：

1. 读取counter当前值
2. 当前值+1
3. 结果保存到counter

比如，10个goroutine同时读到counter的值是9，按照相同的逻辑+1，然后保存到变量身上。但是实际上，counter增加的总数应该是10，但是结果增加了1。

修改一下代码，引入`sync.Mutex`对非原子操作的`counter++`进行保护，同时只能有一个goroutine对counter进行+1的操作：

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
	Val int
}

func (c *Counter) incr() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.Val++
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
    
   fmt.Println(counter.Val)
}
```

同时，我们可以修改`Counter`结构体部分的代码，让代码看起来更简洁: 

```go
type Counter struct {
	sync.Mutex
	Val int
}

func (c *Counter) incr() {
	c.Lock()
	defer c.Unlock()
	c.Val++
}
```

我们采用把Mutex嵌入到结构体中的方式，通常把`sync.Mutex`和被保护的属性放在一起:

```go
type Counter struct {
	AttrA int
	AttrB int
	
	sync.Mutex
	Val int
}
```

## 原理

## 示例

