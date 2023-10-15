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
