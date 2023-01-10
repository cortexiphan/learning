# Map

一般情况下大家应该使用普通的map，如果有读写并发访问，应该使用Mutex或者RWMutex来控制。

但是在锁争用严重的情况下，比如CPU核心数比较多的时候，会出现缓存true sharing的情况，从而使性能大幅下降。

当然也有一些并发map通过分区加锁的方式来降低锁争用，但如果有热点数据，还是会落到同一个分区。

sync.Map能解决读多写少的情况下，多核争用的问题。sync.Map适合以下两种情况：

1. 某个key只会写一次然后多次读取。
2. 各个goroutine读、写、修改的key互不交叉，也就是每个key只被同一个goroutine读写修改。

其他情况下使用sync.Map性能可能反而更低。

## 设计理念

通过读写分离来提高效率。一个只读的read map，一个读写的dirty map，访问read不需要加mutex锁，访问dirty全都要加mutex锁。

注意read的只读只是在map层级不能增加或者删除key，而如果key已经在read里面找到，那是可以读、更新的。

read可以接受key的删除请求，但实际操作是将value的指针设置为nil。key的增加必须通过dirty来修改。

map的value是一个仅包含一个指针的结构体entry，这样同一个entry同时在read和dirty上存在时，他们的value内的指针是指向同一个值的，也就是在一个map更新，就可以在另一个map看到。

查询的时候，先无锁地在read中查找，如果查不到，而dirty不为nil，那证明dirty有一些read中没有的key，需要继续到dirty中查找。当查找dirty的次数到达一定程度，超过map复制的代价，就会用dirty替换掉read，并且把dirty设置为nil

dirty可能是nil，也可能不为nil，当dirty不为nil时，它一定要包含read中所有有效key，这样可以保证dirty升级替换read的时候不会丢失key。

expunged状态的key是上一次新增其他key导致read复制到dirty的时候，将已经设置成nil，也就是刚删除的的value设置成expunged，下一次dirty替换read的时候就会被删除掉。增加expunged状态的目的就是为了方便的处理已删除的key。

### 读取key

```go
func (m *Map) Load(key any) (value any, ok bool) {
    // 首先从read map查找key，如果查找到，无论是读取、存储、删除都可以直接操作，不用加mutex锁。
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    // 没找到并且amended为true。amended为true表明从上一次dirty替换read之后，又有新的key插入了
    // 这时候dirty肯定不为nil，并且有了read中不存在的key，必须去dirty中继续查询
    if !ok && read.amended {
        m.mu.Lock()
        // 拿到mutex锁之后，还需要再查一次read，因为自己不一定是最先拿到mutex的，而前一个拿到mutex的goroutine
        // 可能已经把dirty替换过read了，并且又有其他key插入导致dirty又不为nil了。这时候直接查询read就可以了
        // 否则如果继续查询dirty，可能导致不必要的dirty覆盖read操作。
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // 无论是否在dirty找到key，都增加miss count，必要时dirty覆盖read。
            // 在没有dirty替换read前这个key的查询都得走加锁查询dirty。
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}
```

读取entry

```go
func (e *entry) load() (value any, ok bool) {
    // entry的指针也必须通过原子操作读取，因为有可能其他goroutine也在通过read查到entry，并且正在修改。
    p := atomic.LoadPointer(&e.p)
    // nil表示read复制到dirty之后删除的key，expunged是read复制到dirty之前删除的key
    if p == nil || p == expunged {
        return nil, false
    }
    return *(*any)(p), true
}
```

### 更新key

```go
func (m *Map) Store(key, value any) {
    // 更新key也是直接在read查询，如果能查到就直接尝试更新
    // 如果更新失败，比如这个key的值是expunged，不能直接覆盖，需要走加锁处理路径
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        // 如果能在read中找到，那就准备直接覆盖了，但如果改key已经是expunged了，那意味着
        // dirty不为空，而且dirty中肯定不存在这个key，所以要在dirty中设置这个key，保证
        // read和dirty的一致性，如果read有dirty没有的key，下次替换时就会丢失key
        if e.unexpungeLocked() {
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        // dirty中找到，直接更新，只不过还没有替换到read，暂时每次都得通过加锁访问。
        e.storeLocked(&value)
    } else {
        // read和dirty都没有，那就得插入key了
        if !read.amended {
            // 如果amended为false，那意味着dirty为nil，需要从read把现有key复制到dirty，然后将
            // read的amended属性设置为true，表示已经有新的key插入了，后面read查询miss的时候就得
            // 继续查询dirty了。
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}
```

```go
func (e *entry) tryStore(i *any) bool {
    // 多个goroutine可以同时走到这一步，因为read不加锁。在load之后有可能其他goroutine将器更新为其他值或者nil，
    // 也可能出现dirty替换read的情况，从而使值变成expunged，所以这里需要循环尝试
    for {
        p := atomic.LoadPointer(&e.p)
        // expunged不能直接覆盖，必须在持有锁的的情况下处理。
        if p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
    }
}
```

```go
// 这个函数只在上次dirty替换read之后，第一次插入新key的时候执行
func (m *Map) dirtyLocked() {
    // 这里dirty肯定是nil，这个判断有点多余
    if m.dirty != nil {
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[any]*entry, len(read.m))
    for k, e := range read.m {
        // read中已被删除的key，也就是nil的key，需要设置为expunged，并且不会复制到dirty中。
        // 这样下次dirty替换read的时候，这些expunged的key就真正被删除了。
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    // 这里p被load之后，下面swap之前，有可能被其他goroutine设置为有效值了，所以swap会失败，
    // 但swap之后又被其他goroutine设置nil，所以需要循环尝试，要么设置为expunged成功，
    // 要么其他goroutine设置成有效值成功。
    for p == nil {
        // 只有e.p仍然是nil的情况下，才会设置为expunged。
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    // 这里似乎p不可能是expunged，因为设置为expunged的时机只有上面的swap，而本goroutine设置成功就直接返回了，
    // 而这里是在mutex里面的，不可能有其他goroutine到达执行swap，read里面这时候不会有expunged的key，因为
    // 此前dirty是nil，也就是已经替换了read，read里面不可能有expunged。
    return p == expunged
}

```
