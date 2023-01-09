# Once

sync.Once的目的是让所有通过once.Do发起的函数只执行一次，并且任何once.Do返回的时候，函数确保执行完了。

```go

type Once struct {
    done uint32 // done表示是否已经执行过，并且已经完成
    m    Mutex  // 让第一批同时调用Do的goroutine除第一个以外的都互斥等待。
}

// Do被多次调用时，只有第一次实际执行f函数
func (o *Once) Do(f func()) {
    // 如果o.done为1，那么表明已经执行完成，不用再尝试执行，直接返回。
    // 如果o.done为0，需要尝试执行
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    // 第一批同时调用Do的goroutine需要互斥
    o.m.Lock()
    defer o.m.Unlock()
    // 这里自己不一定是第一个拿到锁的G，需要再次判断o.done是否为0，如果为1，那表明已经有一个G曾经拿到锁并执行完了。
    // 如果为0，那当前G的确是第一个，开始执行f
    // 这里o.done不需要用原子操作的原因是，第一批同时发起的goroutine的hanppens-before关系已经通过Mutex实现，
    // 这里不需要参与保证happens-before关系。而且也不可能有其他对o.done的写操作。
    if o.done == 0 {
        // f执行完后再设置o.done为1，这以后调用Do的G就不用阻塞了
        // 已经堵塞在o.m的G会因为o.done为1而在得到锁后直接退出
        // 注意这里的store已经要在f()之后执行，这样能才能保证f()的执行hanppens-before后续G的LoadUint32(&o.done) == 1的关系。
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}

```