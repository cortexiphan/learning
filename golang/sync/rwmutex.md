# RWMutex

## RWMutex的作用

### 增加并行度

让多个读者或者一个写者得到锁。

### 保证公平性

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

## 申请只读锁

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

## 释放只读锁

## 申请写锁

## 释放写锁

注意此时默认当前goroutine持有了rw.w互斥量的。

```go
func (rw *RWMutex) Unlock() {
	// 把rwmutexMaxReaders这个大整数加回来，readerCount就是写者持锁期间到达并堵塞的读者数量。
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    // rwmutexMaxReaders比可能的goroutine数量大很多个数量级，所以如果r大于rwmutexMaxReaders，
    // 那只可能是（没有持锁的情况下）重复释放，导致多次加法。
    // 注意这里不需要担心溢出，因为rwmutexMaxReaders的值是2^30，至少第一次加的时候不会溢出，
    // 而这时候肯定已经panic了，后面也没有溢出的机会了。
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
