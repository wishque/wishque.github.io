---
title: "rust atomics and locks 阅读笔记（翻译每章总结）"
date: 2023-05-23T16:17:12+08:00
categories: [rust]
draft: false
---

## 第一章.rust并发基础（Basic of Rust Concurrency）

- 一个程序可以有多个线程同步运行，并且可以在任意时间生成新线程
- 主线程结束时，整个程序结束
- 数据竞争是safe rust通过类型系统完全避免的未定义行为
- 实现Send Trait的数据能够被送往其他线程，实现Sync Trait的数据能够在线程间共享
- 普通的线程可能和程序运行时间一样，因此只能借用static生命周期的数据，例如静态数据或者泄漏的堆内存分配
- Arc引用计数能够共享所用权，使得只要有一个线程在使用数据，数据就不会被释放
- scope thread能够限制一个线程的生命周期，这使得线程能够借用non-static生命周期的数据，例如本地变量
- &T叫做共享引用，&mut T叫做独占引用。普通类型不允许通过共享引用改变数据
- 基于UnsafeCell,一些类型拥有内部可变性，允许通过共享引用改变数据
- Cell和RefCell是单线程内部可变性的标准类型。Atomics,Mutex和RwLock是多线程的
- Cell和Atomics只允许整个替换内部数据的值，而RefCell,Mutex和RwLock通过运行时访问规则检查，允许你直接改变内部值
- 线程parking在等待某些条件时可能是一个方便的方式
- 当某些条件是关于被Mutex保护的数据是，使用Condvar是一个更方便，高效的方式

- Multiple threads can run concurrently within the same program and can be spawned at any time. 
- When the main thread ends, the entire program ends. 
- Data races are undefined behavior, which is fully prevented (in safe code) by Rust’s type system. 
- Data that is Send can be sent to other threads, and data that is Sync can be shared between threads.
- Regular threads might run as long as the program does, and thus can only borrow 'static data such as statics and leaked allocations. 
- Reference counting (Arc) can be used to share ownership to make sure data lives as long as at least one thread is using it. 
- Scoped threads are useful to limit the lifetime of a thread to alllow it to borrow non-'static data, such as local variables. 
- &T is a shared reference. &mut T is an exclusive reference. Regular types do not allow mutation through a shared reference. 
- Some types have interior mutability, thanks to UnsafeCell, which allows for mutation through shared references. 
- Cell and RefCell are the standard types for single-threaded interior mutability. Atomics, Mutex, and RwLock are their multi-threaded equivalents. 
- Cell and atomics only allow replacing the value as a whole, while RefCell, Mutex, and RwLock allow you to mutate the value directly by dynamically enforcing access rules. 
- Thread parking can be a convenient way to wait for some condition. 
- When a condition is about data protected by a Mutex, using a Condvar is more convenient, and can be more efficient, than thread parking.

## 第二章.原子操作（Atomics)

- 原子操作是不可分割的；他们要么是全部做完，或者完全没做。
- rust通过标准库的原子类型std::sync::atomic进行原子操作，例如AtomicI32
- 某些原子操作不被所有平台支持
- 当设置许多原子变量时，原子操作的相对顺序是不完全相同的
- 简单的loads和stores对于简单的线程间通讯是很有用的，像暂停flag和状态检查
- 懒初始化有时会出现竞争，但不会引起数据竞争
- fetch_and_modify操作允许一系列小的原子操作同时进行，这在多个线程同时修改一个原子变量时特别有用
- 原子加和原子减在溢出时默认wrap
- compare_and_exchange操作是最灵活和最通用的，是许多其他原子操作的基石
- weak_compare_and_exchange可能是更有效的（当compare相等时，可能不会exchange）

- Atomic operations are indivisable; they have either fully completed, or they haven’t happened yet. 
- Atomic operations in Rust are done through the atomic types in std::sync::atomic, such as AtomicI32. 
- Not all atomic types are available on all platforms. 
- The relative ordering of atomic operations is tricky when multiple variables are involved. More in Chapter 3. 
- Simple loads and stores are nice for very basic inter-thread communication, like stop flags and status reporting. 
- Lazy initialization can be done as a race, without causing a data race. 
- Fetch-and-modify operations allow for a small set of basic atomic modifications that are especially useful when multiple threads are modifying the same atomic variable. 
- • Atomic addition and subtraction silently wrap around on overflow. 
- Compare-and-exchange operations are the most flexible and general, and a building block for making any other atomic operation.
- A weak compare-and-exchange operation can be slightly more efficient.

## 第三章.内存顺序（Memory Ordering）

- 由于事情可能在不同的线程一不同的顺序发生，可能没有对于所有原子操作一致的顺序
- 然而，所有单独的原子变量有他自己的修改顺序，无论怎样的内存顺序，这是所有线程都遵循的
- 操作间的顺序是通过happens-before关系来正式定义的
- 在单个线程中，每一步操作间都是遵循happens-before关系的
- spawn一个线程一定发生在join该线程之前
- unlock一个mutex锁一定发生在再次lock该mutex之前
- Acquire-load一个Release-store后的值建立了一个happens-before关系。这个值可能被任意数量的fetch-and-modify和compare-and-exchange操作修改
- consume-load如果存在的话会是一个轻量级的acquire-load
- 线性一致顺序（SeqCst)能够做到全局一致的操作顺序，但他几乎从来都不是必要的，甚至会让你的代码审核变复杂
- fence能够组合多个操作的内存顺序，或者条件性的添加内存顺序

- There might not be a global consistent order of all atomic operations, as things can appear to happen in a different order from different threads. 
- However, each individual atomic variable has its own total modification order, regardless of memory ordering, which all threads agree on.
- The order of operations is formally defined through happens-before relationships. 
- Within a single thread, there is a happens-before relationship between every single operation. 
- Spawning a thread happens-before everything the spawned thread does. 
- Everything a thread does happens-before joining that thread. 
- Unlocking a mutex happens-before locking that mutex again. 
- Acquire-loading the value from a release-store establishes a happens-before rela‐ tionship. This value may be modified by any number of fetch-and-modify and compare-and-exchange operations. 
- A consume-load would be a lightweight version of an acquire-load, if it existed. 
- Sequentially consistent ordering results in a globally consistent order of operations, but is almost never necessary and can make code review more complicated. 
- Fences allow you to combine the memory ordering of multiple operations or apply a memory ordering conditionally

## 第四章.构建自己的自旋锁（Building Our Own Spin Lock）

- 自旋锁是一种在等待时不断循环或者说自旋的mutex
- 自旋可以降低延迟，但是可能会浪费时钟周期，降低性能
- 自旋循环提示，std::hint::spin_loop()用来提示cpu自旋循环，能够提高性能
- 自旋锁能够仅通过AtomicBool和UnsafeCell实现，UnsafeCell对于实现内部可变性是必要的
- unlock和lock操作的happens-before关系对于放置数据竞争是必要的，数据竞争会导致未定义行为
- Acquire和release内存顺序非常符合这个使用场景
- 当为了避免未定义行为做出推测时，将函数定义为unsafe能够将该责任转给调用者
- Deref和DerefMut trait能够将类型用作引用，提供对其他（内部）变量的透明性
- Drop trait能够在变量drop时做一些事情，例如当他离开作用域，或者对其调用了drop
- lock guard是一个有用的设计模式，用来实现一种特别的类型，代表了对于被lock的锁安全访问。这种类型的行为通常和引用相似，这归功于Deref trait,并且通过Drop trait实现了自动unlock

- A spin lock is a mutex that busy-loops, or spins, while waiting. 
- Spinning can reduce latency, but can also be a waste of clockcycles and reduce performance. 
- A spin loop hint, std::hint::spin_loop(), can be used to inform the processor of a spin loop, which might increase its efficiency. 
- A SpinLock can be implement with just an AtomicBool and an Unsafe Cell, the latter of which is necessary for interior mutability (see “Interior Mutability” on page 13). 
- A happens-before relationship between unlock and lock operations is necessary to prevent a data race, which would result in undefined behavior. 
- Acquire and release memory ordering are a perfect fit for this use case. 
- When making unchecked assumptions necessary to avoid undefined behavior, the responsibility can be shifted to the caller by making the function unsafe. 
- The Deref and DerefMut traits can be used to make a type behave like a reference, transparently providing access to another object. 
- The Drop trait can be used to do something when an object is dropped, such as when it goes out of scope, or when it is passed to drop(). 
- A lock guard is a useful design pattern of a special type that’s used to represent (safe) access to a locked lock. Such a type usually behaves similarly to a reference, thanks to the Deref traits, and implements automatic unlocking through the Drop trait

## 第五章.构建自己的Channel（Building Our Own Channel）

- channel被用来在线程间传递数据
- 很容易使用Mutex和Condvar来实现一个简单灵活但是不高效的的Channel
- 一个一次性channel被用来发送单次数据
- MaybeUninit类型被用来表示可能还没有初始化的T。他的大多数接口都是unsafe的，用户对跟踪是否初始化，不多次复制数据，必要时丢弃内容负责
- 不drop数据是安全的 （被叫做leaking或forgetting），但是没有合适的理由的话不要这么做
- panic对创造一个安全的接口是重要的工具
- 获取一个非Copy对象的值（所有权）能够被用来放置重复做一件事
- 独占借用和共享借用对于保证正确性是一个好用的工具
- 我们可以保证一个对象不离开他的线程通过不实现Send trait,这可以同过PhantomData 标记类型来实现
- 每个设计和实现都包含了对于脑中某个用例的权衡
- 在没有用例时设计一些东西会是有教育意义和有趣的，但是可能永远做不完

- A channel is used to send messages between threads. 
- A simple and flexible, but potentially inefficient, channel is relatively easy to implement with just a Mutex and a Condvar. 
- A one-shot channel is a channel designed to send only one message. 
- The MaybeUninit type can be used to represent a potentially not-yet initialized T. Its interface is mostly unsafe, making its user responsible for tracking whether it has been initialized, not duplicating Copy data, and dropping its contents if necessary. 
- Not dropping objects (also called leaking or forgetting) is safe, but frowned upon when done without good reason. 
- Panicking is an important tool for creating a safe interface. 
- Taking a non-Copy object by value can be used to prevent something to be done more than once. 
- Exclusively borrowing and splitting borrows can be a powerful tool for forcing correctness.
- We can make sure an object stays on the same thread by making sure its type does not implement Send, which can be achieved with the PhantomData marker type.
- Every design and implementation decision involves a trade-off and can best be made with a specific use case in mind. 
- Designing something without a use case can be fun and educational, but can turn out to be an endless task.

## 第六章.构建自己的Arc（Building Our Own Arc）

- Arc 对于引用计数后的内存分配提供共享所有权
- 通过检查引用计数是否唯一，Arc能够提供独占访问（&mut T)
- 增加原子引用计数器时可以使用relaxed内存顺序，但是最后在减少的时候必须和所有之前的减少同步
- Weak可以被用来避免循环引用
- NonNull类型被用来表示一个不为Null的指针
- ManuallyDrop类型被用来手动决定合适drop T通过使用unsafe代码
- 只要使用超过一个原子变量，事情就会变得更复杂
- 在同时处理多个原子变量时实现一个一次性（自旋）锁是一个有效的策略

- Arc provides shared ownership of a reference-counted allocation. 
- By checking if the reference counter is exactly one, an Arc can conditionally provide exclusive access (&mut T). 
- Incrementing the atomic reference counter can be done using a relaxed operation, but the final decrement must synchronize with all previous decrements. 
- A weak pointer (Weak) can be used to avoid cycles. 
- The NonNull type represents a pointer to T that is never null. 
- The ManuallyDrop type can be used to manually decide, using unsafe code, when to drop a T. 
- As soon as more than one atomic variable is involved, things get more complicated. 
- Implementing an ad hoc (spin) lock can sometimes be a valid strategy for operating on multiple atomic variables at once.

## 第七章.理解处理器（Understanding the Processor）

- 在x86-64和arm64架构下，relaxed load和store操作和非原子操作的操作一样
- 常见的原子操作fetch-and-modify和compare-and-exchange操作在x86-64架构和ARMv8.1后的ARM64架构都有相应的指令
- 在x86-64架构，没有相应指令的原子操作都会使用compare-and-exchange循环
- 在ARM64架构，所有的原子操作都能用load-linked/store-conditional循环表示：如果相关的内存操作被干扰，循环重启
- 缓存操作以cache lines为单位，常常为64字节
- 缓存通过缓存一致性协议保持一直，例如writethrough和MESI（modify-exclusive-shared-invalid) 协议
- 填充有时能够提升性能（放置错误的共享缓存），例如通过\#[repr(align(64))]
- load操作比一个失败的compare-and-exchange操作便宜很多，部分是因为compare-and-exchange操作常常要求对于cache line的独占访问
- 指令重排对于单线程程序是不可见的
- 在x86-64架构下，每一个内存操作都具有acquire和release语义，这让relaxed操作和acquire,release操作性能一样。除了store和fences之外的一切都是线性一致语义的
- 在ARM64架构下，acquire和release语义与relaxed操作不同，但是线性一致语义是没有额外的开支的。

- On x86-64 and ARM64, relaxed load and store operations are identical to their non-atomic equivalents. 
- The common atomic fetch-and-modify and compare-and-exchange operations on x86-64 (and ARM64 since ARMv8.1) have their own instructions. 
- On x86-64, an atomic operation for which there is no equivalent instruction compiles down to a compare-and-exchange loop. 
- On ARM64, any atomic operation can be represented by a load-linked/store-conditional loop: a loop that automatically restarts if the attempted memory operation was disrupted. 
- Caches operate on cache lines, which are often 64 bytes in size. 
- Caches are kept consistent with a cache coherence protocol, such as write-through or MESI. 
- Padding, for example through \#[repr(align(64)], can be useful for improving performance by preventing false sharing. 
- A load operation can be significantly cheaper than a failed compare-and-exchange operation, in part because the latter often demands exclusive access to a cache line. 
- Instruction reordering is invisible within a single threaded program. 
- On most architectures, including x86-64 and ARM64, memory ordering is about preventing certain types of instruction reordering. 
- On x86-64, every memory operation has acquire and release semantics, making it exactly as cheap or expensive as a relaxed operation. Everything other than stores and fences also has sequentially consistent semantics at no extra cost. 
- On ARM64, acquire and release semantics are not as cheap as relaxed operations, but do include sequentially consistent semantics at no extra cost.

## 第八章.操作系统原语（Operating System Primatives）

- atomic-wait crate提供了基本的类似于futex的功能，这些功能能够在所有最近版本的主流操作系统上使用
- 最小的mutex实现只需要两个state,就像第四章我们实现的SpinLock
- 更高效的mutex记录了是否有等待的线程，来避免不必要的wake操作
- 在很多情况下，在sleep之前先自旋等待是有用的，这很大程度上取决于操作系统，硬件和当时的情况
- 最小的条件变量CondVar只需要一个通知计数器就可以实现，CondVar::wait会在解锁之前和之后检查
- CondVar会记录等待线程的数量来避免不必要的wake操作
- 避免从wait中奇怪的wake是需要技巧的，需呀记录其他信息
- 最简单的读写锁(RwLock)只要一个原子计数器作为state就能实现
- 一个额外的原子变量可以用来独立的wake writers
- 为了避免写饥饿，需要额外的state来提供等待中的writer的优先级

- The atomic-wait crate provides basic futex-like functionality that works on (recent versions of) all major operating systems. 
- A minimal mutex implementation only needs two states, like our SpinLock from Chapter 4. 
- A more efficient mutex tracks whether there are any waiting threads, so it can avoid an unnecessary wake operation. 
- Spinning before going to sleep might in some cases be beneficial, but it depends heavily on the situation, operating system, and hardware. 
- A minimal condition variable only needs a notification counter, which Condvar::wait will have to check both before and after unlocking the mutex. 
- A condition variable could track the number of waiting threads to avoid unnecessary wake operations. 
- Avoiding spuriously waking up from Condvar::wait can be tricky, requiring extra bookkeeping. 
- A minimal reader-writer lock requires only an atomic counter as its state. 
- An additional atomic variable can be used to wake writers independently from readers. 
- To avoid writer starvation, extra state is required to prioritize a waiting writer over new readers.

## 第九章.构建自己的锁（Building Our Own Lock）

- 系统调用是对操作系统内核的调用，比普通的函数调用慢很多
- 通常，程序不会直接调用系统调用，而是使用操作系统库(libc)来与内核沟通。在许多操作系统中，这是唯一支持的与操作系统沟通的方式。
- libc crate让rust能够用使用libc
- 在POSIX系统中，libc为了支持POSIX标准，包含比c标准更多的东西
- POSIX标准包含pthreads,一个包含同步原语的函数库，例如pthread_mutex_t
- Pthread类型是为c设计的，不适合rust。例如，这些类型是不能move的，这对于rust是个问题
- linux有futex系统调用，通过AtomicU32支持很多wait和wake操作。wait操作验证对于atomic的期待值，这被用来避免不必要的通知
- 除了pthread,macos还提供了os_unfair_lock作为轻量的lock原语
- windows重量级的同步原语总是要和操作系统沟通，但是能够在进程间传递，能够被windows标准的wait函数操作
- windows轻量级的同步原语包含轻的读写锁(SRW lock)和条件变量。这些很容易被rust包装，因为他们是可以move的
- windows也提供了基本的与futex相似的操作，通过WaitOnAddress和WakeByAddress

- A syscall is a call into the operating system’s kernel and is relatively slow compared to a regular function call. 
- Usually, programs don’t make syscalls directly, but instead go through the operating system’s libraries (e.g., libc) to interface with the kernel. On many operating systems, this is the only supported way of interfacing with the kernel. 
- The libc crate gives Rust code access to libc. 
- On POSIX systems, libc includes more than what’s required by the C standard to comply with the POSIX standard. 
- The POSIX standard includes pthreads, a library with concurrency primitives such as pthread_mutex_t. 
- Pthread types are designed for C, not for Rust. For example, they are not movable, which can be a problem. 
- Linux has a futex syscall supporting several waiting and waking operations on an AtomicU32. The wait operation verifies the expected value of the atomic, which is used to avoid missing notifications. 
- In addition to pthread, macOS also provides os_unfair_lock as a lightweight locking primitive. 
- Windows heavyweight concurrency primitives always require interacting with the kernel, but can be passed between processes and used with the standard Windows waiting functions. 
- Windows lightweight concurrency primitives include a “slim” reader-writer lock (SRW lock) and a condition variable. These are easily wrapped in Rust, as they are movable. 
- Windows also provides basic futex-like functionality, through WaitOnAddress and WakeByAddress.
