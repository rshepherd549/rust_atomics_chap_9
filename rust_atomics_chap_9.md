# Rust Atomics and Locks

## Chapter 9: Building Out Own Locks

### Richard Shepherd

---

This chapter builds a mutex, condition variable and reader-writer lock, without busy looping
- so we can understand what they need to take into account
- to understand the more basic tools better.

We're going to use the futex-like `wake` and `wait` operations available for most platforms:

Platform | `wake`, `wait` equivalents
---      | ---
Linux    | futex syscall
Windows  | WaitOnAddress family
FreeBSD  | _umtx_op syscall
macOS    | libc++ (20) std::atomic<T>:wait()

## `atomic-wait` crate

Wraps platform specifics into `wait`, `wake_one`, `wake_all`

---

## `wait(&AtomicU32, u32)`
Block if the value stored in the atomic variable is the equal to the given value.
This thread wakes up and re-checks when another thread calls `wake_*`

## `wake_one(&AtomicU32)`
Directly after modifying the atomic variable, inform one waiting thread of the change.

## `wake_all(&AtomicU32)`
Directly after modifying the atomic variable, inform all waiting threads of the change.

---

# Mutex

```rust
pub struct Mutex<T> {
  state: AtomicU32, ///0: unlocked, 1: locked
  value: UnsafeCell<T>,
}
```
Promise that the `Mutex` is safe to share between threads:
```rust
unsafe impl<T> Sync for Mutex<T> where T: Send {}
```

<!--
Instead of the `AtomicBool` we used in chapter 4 for the `SpinLock`, use `AtomicU32` 0 & 1.
-->

---

## MutexGuard

A safe locking interface for using the `Mutex`:
```rust
pub struct MutexGuard<'a, T> {
  mutex: &'a Mutex<T>,
}
impl<T> Deref for MutexGuard<'_, T> {
  type Target = T; /// ?
  fn deref(&self) -> &T {
    unsafe { &*self.mutex.value.get() }
  }
}
impl<T> DerefMut for MutexGuard<'_, T> {
  fn deref_mut(&mut self) -> &mut T {
    unsafe { &mut *self.mutex.value.get() }
  }
}
```

---

```rust
impl<T> Mutex<T> {
  pub const fn new(value: T) -> Self {
    state: AtomicU32::new(0), //unlocked state
    value UnsafeCell::new(value),
  }
}
```

---

## Mutex::lock

Instead of using a spin-lock to wait to changed from `unlocked` to `locked`, we use `wait()`:

```rust
pub fn lock(&self) -> MutexGuard<T> {
  // Set the state to 1: locked
  while self.state.swap(1, Acquire) == 1 {
    // If it was already locked..
    // ..wait, unless the state is no longer 1.
    wait(&self.state, 1);
  }
  MutexGuard { mutex: self}
}
```

<!--
For the memory ordering, the same reasoning applies as with our spin lock.

The `wait` only blocks if the state is still 1
-->

---

## MutexGuard::Drop

```rust
impl<T> Drop for MutexGuard<'_, T> {
  fn drop(&mut self) {
    // Set the state back to 0: unlocked.
    self.mutex.state.store(0, Release);
    // Wake up one of the waiting threads, if any.
    wake_one(&self.mutex.state);
  }
}
```

- `wake_one` is sufficient and `wake_all` would not achieve any more but might waste other processes' time.
- Another thread creating a `MutexGuard` might still take the lock first
- Without the `wake_one_` the code would still be safe and correct, but unpredictable.
- Without the `wait` and `wake` we'd be back to a spin-lock.
- In general, `wait` and `wake` do not effect the correctness of the memory-safety correctness.

<!--
Need to prompt re-check on the waiting thread with wake.
-->

---

## Avoiding Syscalls

`wait` and `wake` can be slow calls to the kernels.
`wait` is only called when needed, but `wake` is always called.
`wake` is only needed if there are waiters, which we can track with an extra locked state:
```rust
impl<T> Mutex<T> {
  pub const fn new(value: T) -> Self {
    state: AtomicU32::new(0), //0:unlocked state, 1:no listeners, 2:listeners
    value UnsafeCell::new(value),
  }
}
```

---


```rust
pub fn lock(&self) -> MutexGuard<T> {
  if self.state.compare_exchange(0,1,Acquire,Relaxed).ie_err() { //already locked
    while self.state.swap(2, Acquire) != 0 { //additional lock if still locked
      wait(&self.state, 2);
    }
  }
  MutexGuard { mutex: self}
}
```
```rust
impl<T> Drop for MutexGuard<'_, T> {
  fn drop(&mut self) {
    if self.mutex.state.swap(0, Release) == 2 { //if other listeners
      wake_one(&self.mutex.state);
    }
}
```
In the uncontested case no `wait` or `wake` is used.

---

A common contested case is a parallel thread locking briefly. We can use a short spin lock to wait for such contention
before resorting to the slower `wait`.
```rust
pub fn lock(&self) -> MutexGuard<T> {
  if self.state.compare_exchange(0,1,Acquire,Relaxed).ie_err() { //already locked
    lock_contended(&self.state);
  }
  MutexGuard { mutex: self}
}
fn lock_contended(state: &AtomicU32) {
  let mut spin_count = 0;
  while state.load(Relaxed) == 1 && spin_count < 100 {
    spin_count += 1;
    std::hint::spin_loop();
  }
  if stated.compare_exchange(0,1,Acquire,Relaxed).is_ok() {
    return;
  }
  while state.swap(2,Acquire) != 0 {
    wait(state,2);
  }
}
```

---

### Further optimization hints

`std::hint::spin_loop();`` //https://doc.rust-lang.org/stable/std/hint/fn.spin_loop.html

Add `#[cold]` to `lock_contended`

Add `#[inline]` to `Mutex.lock` and `MutexGuard.Drop`

---

## Benchmarking

### Uncontended

Platform | 2-state (ms) | 3-state (ms)
--- | --- | ---
Linux                 |  400 | 40
Linux (old processor) | 1800 | 60
macOS                 |   50 | 50

<!--
macOS performs own bookkeeping to avoid unnecessary syscalls
-->

### Contended (4 threads)

Platform | `wait`` (ms) | spin + `wait` (ms)
--- | --- | ---
Linux                 | 650 | 800
Linux (old processor) | 900 | 750

---

# Condition Variable

## What's it for?

Waiting on a condition of mutex protected data

*Threads waiting vector containing elements*.

e.g. the example from chapter 2:

We could do this manually:

---

```rust
let queue = Mutex::new(VecDeque::new());

thread::scope(|s|) {
  s.spawn(|| {
    loop {
      {
        let mut q = queue.lock().unwrap(); //blocking Aquire
        if let Some(item) = q.pop_front() {
          drop(q);
          dbg!(item);
        }
        //unblocking Release
      }
      //'wait' a bit before looking again
      thread::sleep(Duration::from_secs(1));
    }
  });
  for i in 0.. {
    queue.lock().unwrap().push_back(i);
    thread::sleep(Duration::from_secs(1));
  }
}
```

---

Rather than waking regularly and blocking the Mutex while checking the condition,
it would save time if the thread only re-Acquired the Mutex when code that creates the condition
signals that it was worth checking.

`Condvar::notify_one()` and `Condvar::notify_all()` send this signal:

---

```rust
let queue = Mutex::new(VecDeque::new());
let not_empty = Condvar::new();

thread::scope(|s|) {
  s.spawn(|| {
    loop {
      let mut q = queue.lock().unwrap(); //blocking Aquire
      let item = loop {
        if let Some(item) = q.pop_front() {
          break item;
        }
        else { //If no elements then Release, 'wait' until not_empty notification, then Acquire
          q = not_empty.wait(q).unwrap();
        }
      };
      drop(q);
      dbg!(item);
      //Unblocking Release
    }
  });
  for i in 0.. {
    queue.lock().unwrap().push_back(i); // Acquire,push_back,Release..
    not_empty.notify_one();             // ..before notifying
    thread::sleep(Duration::from_secs(1));
  }
}
```

---

A minimal implementation could use a counter:

```rust
pub struct Condvar {
  counter: AtomicU32,
}
impl Condvar {
  pub const fn new() -> Self {
    Self {counter: AtomicU32::new(0)}
  }
}
pub fn notify_one(&self) {
  self.counter.fetch_add(1, Relaxed);
  wake_one(&self.counter);
}
pub fn notify_all(&self) {
  self.counter.fetch_add(1, Relaxed);
  wake_all(&self.counter);
}
```

---

```rust
pub fn wait<'a,T>(&self, guard: MutexGuard<'a,T>) -> MutexGuard<'a,T> {
  let counter_value = self.counter.load(Relaxed);

  // Unlock the mutex by dropping the guard,
  // but remember the mutex so we can lock it again later.
  let mutex = guard.mutex;
  drop(guard);

  // Wait, but only if the counter hasn't changed since unlocking.
  wait(&self.counter, counter_value); //acceptable risk of 4,294,967,296 overflow

  mutex.lock();
}
```

<!--
The counter-intuitive aspect for me is that the majority of the thread's code occurs while holding the mutex locked.
It is released while waiting - which will hopefully be the majority of the time.

Testing requires care, to prove that the condition variable waits or sleeps, not just spins
-->

---

## Avoiding Syscalls

The `wait` wouldn't be called unless our check had already confirmed that the condition hadn't been met.

We can try avoiding the sys calls in the `wake` using the same count of waiting threads.

```rust
pub struct Condvar {
  counter: AtomicU32,
  num_waiters: AtomicUsize, //New! plenty for possible number of threads
}
impl Condvar {
  pub const fn new() -> Self {
    Self {
      counter: AtomicU32::new(0),
      num_waiters: AtomicUsize::new(0), //New!
    }
  }
}
```

--- 

Notifications can avoid the `wait` if there are no waiters..

```rust
pub fn notify_one(&self) {
  if self.num_waiters.load(Relaxed) > 0 { //New!
    self.counter.fetch_add(1, Relaxed);
    wake_one(&self.counter);
  }
}
pub fn notify_all(&self) {
  if self.num_waiters.load(Relaxed) > 0 { //New!
    self.counter.fetch_add(1, Relaxed);
    wake_all(&self.counter);
  }
}
```

---

..and increment it for a waiter, and decrement when woken.

```rust
pub fn wait<'a,T>(&self, guard: MutexGuard<'a,T>) -> MutexGuard<'a,T> {
  self.num_waiters.fetch_add(1, Relaxed); //New!

  let counter_value = self.counter.load(Relaxed);

  let mutex = guard.mutex;
  drop(guard);

  wait(&self.counter, counter_value);

  self.num_waiters.fetch_sum(1, Relaxed); //New!

  mutex.lock();
}
```

<!--
TBD Consider the Relaxed loading throughout - and whether the happens-before relationships are appropriately maintained.
They so seem to rely on human reasoning, rather than the type/static-style checking we've come to expect from Rust. 
-->

---

## Avoiding Spurious Wake-ups

---

## Thundering Herd Problem

---

# Reader-Writer Lock

```rust
pub struct RwLock<T> {
  state: AtomicU32, // Number of readers, or u32::MAX if write-locked
  value: UnSafeCell<T>,
}
```
```rust
unsafe impl<T> Sync for RwLock<T> where T: Send + Sync {}
```

```rust
impl<T> RwLock<T> {
  pub const fn new(value: T) -> Self {
    Self {
      state: AtomicU32::new(0), //Unlocked
      value: UnsafeCell::new(value),
    }
  }
  pub fn read(&self) -> ReadGuard<T> {
    ...
  }
  pub fn write(&self) -> WriteGuard<T> {
    ...
  }
  pub struct ReadGuard<'a,T> {
    rwlock: &'a RwLock<T>,
  }
  pub struct WriteGuard<'a,T> {
    rwlock: &'a RwLock<T>,
  }
}
```

```rust
impl<T> Deref for WriteGuard<'_,T> {
  type Target = T;
  fn deref(&self) -> &T {
    unsafe { &*self.rwlock.value.get()}
  }
}
impl<T> DerefMut for WriteGuard<'_,T> {
  type Target = T;
  fn deref_mut(&self) -> &T {
    unsafe { &*self.rwlock.value.get()}
  }
}
```

```rust
impl<T> Deref for ReadGuard<'_,T> {
  type Target = T;
  fn deref(&self) -> &T {
    unsafe { &*self.rwlock.value.get()}
  }
}
```

```rust
pub fn read(&self) -> ReadGuard<T> {
  let mut s = self.state.load(Relaxed);
  loop {
    if s < u32::MAX {
      assert!(s != u32::MAX-1, "too many readers");
      match self.state.compare_exchange_weak(
        s, s+1, Acquire, Relaxed
      ) {
        Ok(_) => return ReadGuard { rwlock: self },
        Err(e) => s = e,
      }
    }
    if s == u32::MAX {
      wait(&self.state, u32::MAX);
      s = self.state.load(Relaxed);
    }
  }
}
```

```rust
pub fn write(&self) -> WriteGuard<T> {
  while let Err(s) = self.state.compare_exchange(
    0, u32::MAX, Acquire, Relaxed
  ) {
    wait(&self.state, s); //Wait while already locked
  }
  WriteGuard { rwlock: self }
}
```

```rust
impl<T> Drop for ReadGuard<'_,T> {
  fn drop(&mut self) {
    if self.rwlock.state.fetch_sub(1,Release) == 1 {
      //Wake up a waiting writer, if any.
      wake_one(&self.rwlock.state);
    }
  }
}
```
```rust
impl<T> Drop for WriteGuard<'_,T> {
  fn drop(&mut self) {
    self.rwlock.state.store(0,Release) == 1 {
    //Wake up all waiting readers and writers.
    wake_all(&self.rwlock.state);
  }
}
```

---

## Avoiding Busy-Looping Writers

```rust
pub struct RwLock<T> {
  state: AtomicU32, // The number of readers, or u32::MAX if write-locked
  writer_wake_counterL AtomicU32, //New! Incremented to wake up writers
  value: UnsafeCell<T>,
}
impl<T> RwLock<T> {
  pub const fn new(value: T) -> Self {
    Self {
      state: AtomicU32::new(0),
      writer_wake_counter: AtomicU32::new(0), //New!
      value: UnsafeCell::new(value),
    }
  }
}
```

```rust
pub fn write(&self) -> WriteGuard<T> {
  while self.state.compare_exchange(
    0, u32::MAX, Acquire, Relaxed
  ).is_err() {
    let w = self.writer_wake_coutner.load(Acquire);
    if self.state.load(Relaxed) != 0 {
      // Wait if the RwLock is still locked, but only if
      // there have been no wake signals since we checked.
      wait(&self.writer_wake_counter, w);
    }
  }
  WriteGuard { rwlock: self }
}
```

