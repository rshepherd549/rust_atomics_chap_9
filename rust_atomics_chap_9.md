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

