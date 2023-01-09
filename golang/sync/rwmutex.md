# RWMutex

## RWMutex的作用

### 1.增加并行度

让多个读者或者一个写者得到锁。

### 2.保证公平性

多个读者持锁时，新的写者到达并堵塞，此后到达的读者也要等待，等持锁的读者都退出后，上面堵塞的写者先被唤醒。

写者持锁时到达的读者堵塞后，在该写者释放锁后，这批等待的读者要被先唤醒，然后才能让后续的写者申请锁。

```go

type RWMutex struct {
    w           Mutex  // 用于写者之间互斥。
    writerSem   uint32 // 有读者持锁时，写者等待该信号量。
    readerSem   uint32 // 有写者持锁时，读者等待该信号量。
    readerCount int32  // 申请了读锁且还没有退出的量，包含没申请到并堵塞了的
    readerWait  int32  // 有写者堵塞后，还没有退出的读者的数量。
}

const rwmutexMaxReaders = 1 << 30

```

## 加解锁分析

### 1.申请只读锁

```go

// 申请读锁首先是将readerCount加一，再看readerCount的结果。如果此时只有读者，那么readerCount肯定大于0，
// 直接就得到锁了；如果此时有写者，那readerCount一定小于0，因为写者会将readerCount减去一个固定的大整数，
// 这样readerCount即包含了读者的数量，也会变成负数。
// 如果有写者，那就等待在readerSem上，等待写者退出后唤醒。
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // 有写者在等待了，后面到达的读者都得等待在信号量上。
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}

```

### 2.释放只读锁

```go
func (rw *RWMutex) RUnlock() {
    // 释放只读锁和其他读者互不影响。如果减一之后readerCount还是大于或者等于零，那证明此时没有写者等待，可以直接释放。
    // 如果小于零，那就进入下面的慢路径，也就是看是否需要由自己唤醒写者。
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        // Outlined slow-path to allow the fast-path to be inlined
        rw.rUnlockSlow(r)
    }
}

func (rw *RWMutex) rUnlockSlow(r int32) {
    if r+1 == 0 || r+1 == -rwmutexMaxReaders {
        fatal("sync: RUnlock of unlocked RWMutex")
    }
    // 每个释放读锁的读者，在发现当前有写者等待时，都要对readerWait减一，减完之后，
    // 如果大于1，那自己不是最后一个释放的读者，不用唤醒写者。
    // 如果等于1，那自己的确是最后一个释放的读者，唤醒写者。
    // 如果小于1，则是写者申请和读者释放几乎同时，且读者先完成readerWait的减一，不用唤醒。
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // The last reader unblocks the writer.
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

### 3.申请写锁

```go
func (rw *RWMutex) Lock() {
    // 先申请互斥锁，和其他写者互斥。
    rw.w.Lock()
    // 将readerCount减去一个rwmutexMaxReaders，这样后续的读者就知道已经有写者在等待了。
    // 注意readerCount此时仍然保留读者数量信息，r就是当前读者的数量。
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 这里判断写者是否需要等待。尤其是写者申请和最后一个读者的释放同时的时候。
    // 如果r == 0，那没有读者，也就不用等待，直接得到锁。
    // 如果r != 0，那将readerWait加上r，看其结果
    // 如果readerWait等于0，证明这r个读者恰好刚刚退出了，所以不用等待。
    // 如果readerWait不等于0，那至少有一个读者还没有执行readerWait的减一，那必定有一个读者会执行到释放信号量那一步，所以这里先等待在信号量上。
    // 这里readerWait会被r个读者分别减一，以及被写者加r，这r+1个事件可以随意排列也不影响正确性。
    // 在读者角度，因为readerWait初始值是0，如果读者发现减完结果为0，那么说明写者加r已经完成，正在等待信号量，自己是最后一个读者，需要执行唤醒。
    // 如果发现减完小于0，那么写者还没有加r，自己也许是最后一个读者，但这时自己先于写者释放了，也就不用堵塞写者了；如果自己不是最后一个读者，那让最后一个读者关心唤醒的事情吧。
    // 如果减完大于0，那还有读者没有到这一步呢，唤醒的事让后面的读者处理吧。
    // 在写者角度，加完r的readerWait是一定大于等于0的，如果等于0那意味着读者都退出了，不用阻塞，否则有一个读者会走到唤醒步骤，自己先堵塞。
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```

### 4.释放写锁

注意此时默认当前goroutine持有了rw.w互斥量的。

```go
func (rw *RWMutex) Unlock() {
    // 把rwmutexMaxReaders这个大整数加回来，readerCount就是写者持锁期间到达并堵塞的读者数量。
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    // rwmutexMaxReaders比可能的goroutine数量大很多个数量级，所以如果r大于rwmutexMaxReaders，
    // 那只可能是（没有持锁的情况下）重复释放，导致多次加法。
    // 注意这里不需要担心溢出，因为rwmutexMaxReaders的值是2^30，至少第一次重复加的时候不会溢出，
    // 而这时候肯定已经被下上面的判断panic了，后面也没有溢出的机会了。
    if r >= rwmutexMaxReaders {
        race.Enable()
        fatal("sync: Unlock of unlocked RWMutex")
    }
    // 如果存在堵塞的读者，逐一唤醒。注意这里的数量是精确的。
    // 当然，此刻达到的读者也能直接得到锁，不用通过唤醒。
    // 此时rw.w互斥量没有释放，所以读者的唤醒是先于下一个写者的。
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    // 释放rw.w，以便下一个写者能申请锁，正常情况它都晚于上面被唤醒的读者，从而保证公平。
    rw.w.Unlock()
}
```

## 性能分析

读写锁在读多写少的情况下增加了读的并发度，比互斥量更好。但这只是在lock contention不严重的情况是真的，如果在cpu核心数很大，多个goroutine同时竞争一个锁的情况下，读写锁也会越来越慢。

原因是加解锁有一个AddInt32原子操作，这是一个Read-Modify-Write操作，每次修改都要先读到最新值，然后再修改。根据MESI协议，每个CPU修改之后都要通知其他CPU将该cache line标记为无效，这样多个CPU同时修改时，就会导致缓存的反复失效。CPU越多这个情况越严重。

如果使用的场景是RWMutex+map，那可以考虑使用sync.Map，当然要符合sync.Map的设计针对的两种场景才行，否则应该重新设计。

### MESI协议

I： Invalid，此状态下要读取或者修改都要在总线上发布。

S：多个cache都有该block，并且是干净的。

E：只有当前cache有这个block，并且是干净的。

M：只有当前cache有这个block，并且已经被修改了。

对E状态的block写，直接变成M状态，不用在总线通知其他cache。

对S状态的block写，需要在总线通知其他cache要invalidate这个block，然后自己的block变成M。

对I状态的block写，实际是write miss。如果此时没有其他cache有这个block，直接读下级内存，并标记block为M。
如果此时有其他多个cache为S，或者其他一个E，读下级内存，并且通知其他cache设置该block为I，当前block为M。
如果此时有其他一个为M。另外那个为M的cache将block写入下级内存，标记为I，然后原来的cache再读下级内存，标记为M。
