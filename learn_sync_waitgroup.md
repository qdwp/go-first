# [Golang初探] 之 sync.WaitGroup

## 描述

**sync.WaitGroup**会阻塞并等待多个 `goroutine` 执行结束。主 `goroutine` 调用 `Add` 设置需要等到的 `goroutine` 数量，当每一个 `goroutine` 结束的时候会调用 `Done` 方法。 `Wait` 方法会阻塞程序直到所有的 `goroutine` 都执行结束。

## Example

```golang
var wg sync.WaitGroup
var urls = []string{
        "http://www.golang.org/",
        "http://www.google.com/",
        "http://www.somestupidname.com/",
}
for _, url := range urls {
        // Increment the WaitGroup counter.
        wg.Add(1)
        // Launch a goroutine to fetch the URL.
        go func(url string) {
                // Decrement the counter when the goroutine completes.
                defer wg.Done()
                // Fetch the URL.
                http.Get(url)
        }(url)
}
// Wait for all HTTP fetches to complete.
wg.Wait()
```

## 源码解读

### 结构体定义

```golang
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
	noCopy noCopy

	// 64位数据高位表示计数器，低位表示等待计数。
	// 使用12位(byte)计数(uint32 * 3 => 8 * 3)
	// 8位表示对齐计数，另外4位表示信号量
	state1 [3]uint32
}
```

### `Add` 方法

```golang
func (wg *WaitGroup) Add(delta int) {
	// 取得计数信息和信号量
	statep, semap := wg.state()
	
	... // 忽略 race 相关代码
	
	// 计数器(64位高位) + delta
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)		// 计数器
	w := uint32(state)			// 等待计数
	
	...

	// 计数器为负
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}

	// Add 方法与 Wait 不应该并发执行
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

	// 计数器正常增加
	if v > 0 || w == 0 {
		return
	}

	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// Reset waiters count to 0.
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false)
	}

}

// state 方法
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}

```

### `Done` 方法

```golang
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

### `Wait` 方法

```golang
// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	...

	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)		// 计数器
		w := uint32(state)			// 等待计数
		if v == 0 {
			// 计数器为 0 ，不需要等待
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
		// 增加等待计数
		// 原子级替换操作
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
				race.Write(unsafe.Pointer(semap))
			}
			runtime_Semacquire(semap)
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
	}
}
```