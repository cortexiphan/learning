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
		// A writer is pending, wait for it.
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}

```

## 释放只读锁

## 申请写锁

## 释放写锁
