# [Golang] 初探之 sync.Once

##  描述

**sync.Once** 是 Golang package 中使方法只执行一次的对象实现，作用与 init 函数类似。但也有所不同。

* `init` 函数是在文件包首次被加载的时候执行，且只执行一次
* `sync.Onc` 是在代码运行中需要的时候执行，且只执行一次

当一个函数不希望程序在一开始的时候就被执行的时候，我们可以使用 sync.Once 。

## Example

```golang
package main

import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once
    onceBody := func() {
        fmt.Println("Only once")
    }
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            once.Do(onceBody)
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}

# Output:
# Only once
```

**sync.Once** 使用变量 done 来记录函数的执行状态，使用 sync.Mutex 和 sync.atomic 来保证线程安全的读取 done 。

## 源码

```golang
package sync

import (
    "sync/atomic"
)

// Once is an object that will perform exactly one action.
type Once struct {
    m    Mutex
    done uint32
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once. In other words, given
//  var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
//  config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return
    }
    // Slow-path.
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```






