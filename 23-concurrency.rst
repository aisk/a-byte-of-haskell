并发编程与运行时调度
================================================================================

在很多主流语言中，高并发编程需要在两种方案里取舍：使用操作系统内核线程，承担较大的内存与上下文切换开销；或者使用基于事件循环的异步回调模型，承担"函数着色（Function Coloring）"与回调嵌套的代价。

GHC 采用的是\ **用户态 M:N 调度运行时**\ 。它在底层实现了类似 Node.js、Python asyncio 的事件循环，同时让代码保持同步风格。

GHC 绿色线程
--------------------------------------------------------------------------------

操作系统内核线程（1:1 模型）通常要分配 1MB 到 8MB 的栈，线程切换需要进出内核态，单个进程开几千个线程就会接近上限。

GHC 运行时的线程模型不同：

- **初始开销小**\ ：GHC 的线程是完全在用户态调度的 **绿色线程（Green Threads）**\ ，初始栈只有约 **1KB**\ ，按需增长；
- **数量可以很大**\ ：单个进程创建几十万甚至上百万个绿色线程是可行的；
- **切换开销小**\ ：上下文切换在用户空间完成，开销远小于内核线程切换。

使用 forkIO 创建线程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Control.Concurrent`` 提供 ``forkIO`` 启动一个绿色线程：

.. code:: haskell

   import Control.Concurrent
   import Control.Monad (forever)

   simpleThreadDemo :: IO ()
   simpleThreadDemo = do
     -- 启动一个后台线程
     threadId <- forkIO $ forever $ do
       putStrLn "后台线程工作中..."
       threadDelay 1000000 -- 暂停 1 秒（单位是微秒）

     putStrLn $ "已启动线程，ID: " ++ show threadId
     threadDelay 3000000 -- 主线程等待 3 秒后退出
     putStrLn "主线程退出，其余线程随之终止。"

I/O Manager 与事件循环
--------------------------------------------------------------------------------

如果大量绿色线程都在等待网络 Socket，会不会把系统线程耗尽？

答案是不会：\ **代码是同步风格的，底层执行的是非阻塞 I/O**\ 。

GHC 运行时内置了一个 **I/O Manager**\ ，基于操作系统的多路复用接口：

- Linux 上使用 **epoll**\ ；
- macOS / BSD 上使用 **kqueue**\ ；
- Windows 上有独立的实现（较新版本提供基于 IOCP 的 WinIO，需要显式启用）。

当一个绿色线程执行网络读取时，它不会让底层的操作系统线程休眠。它向 I/O Manager 注册一个"读就绪"事件，然后挂起让出控制权，调度器转而执行其他就绪的绿色线程。内核通过 epoll 通知数据就绪后，I/O Manager 唤醒对应的绿色线程继续执行。

.. tip::

   **如果你熟悉其他语言：没有"函数着色问题"的异步**\ ：

   - **对比 Python asyncio / C# / JavaScript**\ ：
     这些语言的异步靠事件循环驱动，但存在\ **"函数着色问题（Function Coloring Problem）"**\ ：一旦底层某个函数被声明为 ``async``\ ，调用链上的所有上层函数都得加上 ``async`` 与 ``await``\ ，同步代码和异步代码变成两个不能随意混用的世界。
   - **Haskell 的做法**\ ：
     所有 I/O 都是 ``IO`` 类型，没有同步与异步之分。代码按顺序书写，运行时在后台通过 epoll 完成非阻塞调度。

阻塞 I/O 与 -threaded 运行时
--------------------------------------------------------------------------------

阻塞调用的问题
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

网络通信有 epoll 这类异步机制，但在现代操作系统内核里，\ **本地文件读写、DNS 解析以及大部分外部 C 库（FFI）调用仍然是阻塞的**\ 。

如果使用 GHC 默认的单线程运行时（non-threaded RTS）：
一旦某个绿色线程触发了阻塞的系统调用（比如一次耗时很长的磁盘读取或 C 函数调用），\ **进程唯一的操作系统线程会被内核挂起**\ ，其余所有绿色线程也跟着停下来。

启用多线程运行时
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了解决这个问题，实际项目通常在编译时开启 ``-threaded``\ ：

.. code:: sh

   $ ghc -O2 -threaded -rtsopts -with-rtsopts="-N" MyApp.hs

- ``-threaded``\ ：让运行时使用多个操作系统线程来调度绿色线程（M:N）。
- ``-rtsopts -with-rtsopts="-N"``\ ：让运行时按机器的核心数启动相应数量的工作线程（Capability），实现多核并行。

开启 ``-threaded`` 之后，绿色线程、Capability、操作系统线程与核心之间的关系如下：

.. mermaid::

   graph TD
     subgraph hs["Haskell 绿色线程（数量可以很大）"]
       H1["线程 1"]
       H2["线程 2"]
       H3["线程 3"]
       H4["..."]
       Hn["线程 n"]
     end
     subgraph cap["Capability（数量由 -N 决定）"]
       C1["Capability 1<br/>运行队列"]
       C2["Capability 2<br/>运行队列"]
     end
     subgraph os["操作系统线程与核心"]
       O1["OS 线程 1"] --> K1["核心 1"]
       O2["OS 线程 2"] --> K2["核心 2"]
       O3["OS 线程 3<br/>执行阻塞的系统调用或 FFI"]
     end
     IOM["I/O Manager<br/>用 epoll / kqueue 等待网络事件<br/>就绪后把线程放回运行队列"]
     H1 --> C1
     H2 --> C1
     H3 --> C2
     H4 --> C2
     Hn --> C2
     C1 --> O1
     C2 --> O2
     C1 -. "阻塞调用时让出 Capability" .-> O3
     H2 -. "等待 Socket" .-> IOM
     IOM -. "唤醒" .-> C1

一个 Capability 同一时刻只运行一个绿色线程。绿色线程发起阻塞调用时，运行时把 Capability 交给另一个操作系统线程继续调度，阻塞的调用则由原来的线程等待完成。

.. tip::

   **如果你熟悉其他语言：类比 Go 的调度器与 Rust 的 Tokio**\ ：

   - **Go 的 GMP 模型**\ ：Goroutine 遇到阻塞系统调用时，调度器会把阻塞的 M（系统线程）与 P（处理器）解绑，并新建或唤醒另一个系统线程接管其他 Goroutine。
   - **Rust 的 Tokio**\ ：遇到无法异步化的本地文件操作或计算密集任务时，需要显式用 ``tokio::task::spawn_blocking`` 把任务交给阻塞线程池。
   - **Haskell 的多线程运行时**\ ：原理相同。对支持异步的操作（如网络）使用 epoll 复用；遇到必须阻塞的系统调用时，交给单独的操作系统线程，其他核心上的线程继续调度剩余的绿色线程。

结构化并发：Control.Concurrent.Async
--------------------------------------------------------------------------------

``forkIO`` 本身很轻量，但手动管理线程生命周期容易出问题：子线程的返回值不好拿到、未捕获的异常让线程静默退出、父线程退出时子线程泄漏。

通常的做法是使用 **async** 库提供的\ **结构化并发（Structured Concurrency）**\ 接口。

并发执行两个任务：concurrently
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Concurrent.Async
   import Control.Concurrent (threadDelay)

   fetchUserData :: IO String
   fetchUserData = do
     threadDelay 200000
     return "用户档案数据"

   fetchUserOrders :: IO [String]
   fetchUserOrders = do
     threadDelay 300000
     return ["订单 #1001", "订单 #1002"]

   fetchCompleteDashboard :: IO (String, [String])
   fetchCompleteDashboard =
     concurrently fetchUserData fetchUserOrders

``concurrently`` 在两个绿色线程中并发运行任务，等双方都完成后把结果组成二元组返回。\ **如果其中一个抛出异常，另一个会被自动取消**\ ，不会留下孤儿线程。

竞速：race
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

超时兜底之类的场景需要"谁先完成用谁"：

.. code:: haskell

   -- 哪个任务先完成就取哪个的结果，另一个被取消：
   fetchWithTimeout :: IO (Either () String)
   fetchWithTimeout =
     race (threadDelay 500000) fetchUserData

- 若 500ms 计时先完成，返回 ``Left ()``\ （代表超时）；
- 若网络请求先完成，返回 ``Right "用户档案数据"``\ ，计时器被取消。

共享状态的三个层次
--------------------------------------------------------------------------------

前面的例子里线程之间不共享任何数据，\ ``concurrently`` 把结果收回来就完事了。一旦几个线程要读写同一个计数器、同一个句柄、同一个队列，就要选一种装共享状态的容器。\ ``base`` 和 ``stm`` 提供了三层，从简单到强大：

.. list-table::
   :header-rows: 1
   :widths: 16 40 44

   * - 容器
     - 适合的场景
     - 核心操作
   * - ``IORef``
     - 单个变量：计数器、缓存、开关。只做原子更新，不需要等待别的线程
     - ``newIORef``\ 、\ ``readIORef``\ 、\ ``atomicModifyIORef'``
   * - ``MVar``
     - 一把锁或一个单槽信箱：保护一个句柄、两个线程之间交接一个值、等待完成信号
     - ``newMVar``\ 、\ ``takeMVar``\ 、\ ``putMVar``\ 、\ ``withMVar``
   * - ``TVar`` 与 STM
     - 多个变量必须一起改，或者需要按条件等待：余额够了再扣款，队列非空再取
     - ``newTVar``\ 、\ ``readTVar``\ 、\ ``writeTVar``\ 、\ ``atomically``\ 、\ ``retry``

**IORef** 是最轻的一层，就是一个可变的引用单元。多线程同时修改时不能先 ``readIORef`` 再 ``writeIORef``\ ，两步之间可能被别的线程插进来。要用 ``atomicModifyIORef'``\ ，它把“读、算、写”作为一个原子操作完成；末尾的撇号表示对新值严格求值，避免累积 Thunk：

.. code:: haskell

   module Main (main) where

   import Control.Concurrent
   import Control.Monad (forM_, replicateM_)
   import Data.IORef

   main :: IO ()
   main = do
     counter <- newIORef (0 :: Int)
     done <- newEmptyMVar
     forM_ [1 .. 10 :: Int] $ \_ -> forkIO $ do
       replicateM_ 1000 (atomicModifyIORef' counter (\n -> (n + 1, ())))
       putMVar done ()                 -- 完成后往信箱里放一个信号
     replicateM_ 10 (takeMVar done)    -- 主线程收满 10 个信号再继续
     readIORef counter >>= print       -- 10000

**MVar** 是一个要么空要么满的格子。\ ``takeMVar`` 在格子空时阻塞，取走后格子变空；\ ``putMVar`` 在格子满时阻塞。上面的 ``done`` 就是把它当信箱用：子线程放，主线程取，取不到就等。另一种常见用法是当锁。\ ``newMVar ()`` 创建一个装着 ``()`` 的格子，\ ``withMVar`` 取出、执行、放回，同一时刻只有一个线程能进入临界区：

.. code:: haskell

   lock <- newMVar ()
   forM_ ["甲", "乙", "丙"] $ \name -> forkIO $
     forM_ [1 .. 3 :: Int] $ \i ->
       withMVar lock $ \_ -> putStrLn (name ++ " 第 " ++ show i ++ " 行")

没有锁时三个线程的输出会交错，有了锁每一行都是完整的。\ ``withMVar`` 内部用了 IO 一章的 ``bracket``\ ，临界区抛出异常时锁也会被放回。

**TVar 与 STM** 是第三层，下一节展开。它解决的是 ``MVar`` 做不好的两件事：同时修改多个变量而不暴露中间状态，以及“条件不满足就等，满足了自动醒来”。

怎么选：单个变量、只做原子更新，用 ``IORef``\ ；需要互斥或者两个线程交接，用 ``MVar``\ ；拿不准就用 ``TVar``\ ，它的能力覆盖前两者，代价只是每次访问多一点开销。线程本身的管理也有一条默认规则：上面两个例子用 ``forkIO`` 加 ``MVar`` 计数是为了展示原语，实际代码里除了“发出去就不管”的守护线程，一律优先用 ``async`` 的 ``concurrently``\ 、\ ``race`` 和 ``mapConcurrently``\ ，它们会自动等待子线程、传播异常、取消兄弟线程。

.. tip::

   **如果你熟悉其他语言**\ ：\ ``IORef`` 加 ``atomicModifyIORef'`` 相当于 Java 的 ``AtomicReference`` 或 Go 的 ``atomic`` 包；\ ``MVar`` 当锁用时相当于 ``Mutex``\ ，当信箱用时相当于 Go 里容量为 1 的 channel 或 Java 的 ``SynchronousQueue``\ ；\ ``TVar`` 在主流语言里没有直接对应物，最接近的是数据库事务。

软件事务内存（STM）
--------------------------------------------------------------------------------

传统多线程编程用互斥锁（Mutex / Lock）保护共享可变内存。基于锁的模型有几个众所周知的问题：

- **死锁（Deadlock）**\ ：两个线程以不同顺序申请锁 A 与锁 B，就可能互相等待；
- **锁不能组合**\ ：模块 X 与模块 Y 各自线程安全，把它们组合成一个复合操作（如从账户 A 转账到账户 B）时，通常要暴露内部锁细节或引入更粗粒度的全局锁；
- **条件变量难写对**\ ：\ ``wait()`` / ``notify()`` 容易出现虚假唤醒（Spurious Wakeup）与丢失唤醒。

**软件事务内存（Software Transactional Memory，STM）** 把数据库事务的思路引入内存并发。这个概念由 Shavit 和 Touitou 在 1995 年提出，Haskell 的实现（Harris 等人，2005）是其中应用最成功的之一，现在是 GHC 标准库的一部分。

核心原语：TVar、atomically 与 retry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``Control.Concurrent.STM`` 中：

.. code:: haskell

   import Control.Concurrent.STM

   -- 1. 事务变量：只能在 STM 单子内读写
   type BankAccount = TVar Int

   -- 2. 事务动作（注意：类型是 STM ()，不是 IO）
   transfer :: BankAccount -> BankAccount -> Int -> STM ()
   transfer fromAcc toAcc amount = do
     fromBal <- readTVar fromAcc
     if fromBal < amount
       then retry -- 余额不足：挂起当前线程，等相关 TVar 变化后重试
       else do
         writeTVar fromAcc (fromBal - amount)
         toBal <- readTVar toAcc
         writeTVar toAcc (toBal + amount)

   -- 3. 在 IO 中原子提交事务
   executeTransfer :: BankAccount -> BankAccount -> IO ()
   executeTransfer a b = atomically (transfer a b 100)

``atomically`` 执行一个事务的流程如下。事务中对 ``TVar`` 的读写先记录在线程本地的日志里，提交时才与共享内存比对：

.. mermaid::

   flowchart TD
     S["atomically 开始事务"] --> L["在本地日志中读写 TVar<br/>不改动共享内存"]
     L --> R{"事务中调用了 retry？"}
     R -- "是" --> W["丢弃日志，挂起线程<br/>等待读过的 TVar 被其他线程修改"]
     W --> S
     R -- "否" --> V{"提交校验：<br/>读过的 TVar 是否被别的线程改过？"}
     V -- "没有冲突" --> C["把日志写入共享内存<br/>事务完成"]
     V -- "有冲突" --> D["丢弃日志"]
     D --> S

因为事务内只有对 ``TVar`` 的读写，没有其他副作用，所以丢弃日志重跑是安全的。

备选分支：orElse
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

STM 的另一个组合子是 ``orElse``\ ：

.. code:: haskell

   -- 优先从主账户取款，若余额不足（触发 retry），改为从备用账户取款：
   smartWithdraw :: BankAccount -> BankAccount -> Int -> BankAccount -> STM ()
   smartWithdraw primary secondary amount target =
     transfer primary target amount `orElse` transfer secondary target amount

基于锁的模型很难提供 ``orElse`` 这样的能力：第一步失败后，需要撤销已经获得的锁并且不留下任何副作用，再去尝试第二步。

.. tip::

   **如果你熟悉其他语言：为什么 STM 在 Haskell 里比较成功**\ ：

   - **命令式语言的尝试**\ ：
     STM 在 Java、C++ 等语言里都有过实现，但落地都遇到了困难。一个主要原因是这些语言无法阻止在事务内执行不可回滚的副作用：如果事务里调用了打印、网络请求，当检测到冲突需要回滚时，已经发出的数据包和已经打印的日志是收不回来的。Clojure 的 STM 是一个相对成功的例子，它依赖的正是不可变数据结构。
   - **Haskell 的类型系统**\ ：
     在 ``STM`` 单子内不能执行 ``IO`` 动作，编译器会拒绝这样的代码。事务里能做的只有读写 ``TVar``\ ，这些操作全部记录在事务日志里。因为没有不可逆的副作用，运行时可以安全地回滚并重试事务（乐观并发控制，Optimistic Concurrency Control）。
   - **对比其他语言的并发状态模型**\ ：

     - **Go**\ ：推荐通过 Channel 传递数据（CSP 模型），处理复杂共享状态时仍然需要 ``sync.Mutex``\ ，死锁和锁顺序由开发者自己保证。
     - **Rust**\ ：通过所有权系统（\ ``Arc<Mutex<T>>``\ ）在编译期排除了数据竞争（Data Race），但运行期仍可能因为加锁顺序不当发生死锁。
     - **Haskell STM**\ ：事务之间是可串行化（Serializable）的，没有死锁，支持条件等待（\ ``retry``\ ）与事务组合（\ ``orElse``\ ）。

小结
--------------------------------------------------------------------------------

- GHC 的绿色线程初始栈约 1KB，用 ``forkIO`` 创建，单机可以开到百万级。
- I/O Manager 基于 epoll / kqueue 实现非阻塞调度，代码保持同步风格。
- 涉及文件 I/O 或 FFI 等阻塞调用时，需要开启 ``-threaded``\ ，让多个操作系统线程参与调度。
- 并发任务用 ``Control.Concurrent.Async`` 的 ``concurrently`` 和 ``race``\ ，避免手动管理线程。
- 共享状态分三层：单个变量用 ``IORef`` 加 ``atomicModifyIORef'``\ ，锁和信箱用 ``MVar``\ ，多变量和条件等待用 ``TVar``\ ；拿不准就用 ``TVar``\ 。
- STM 用事务代替锁来处理共享状态，没有死锁，事务可以用 ``retry`` 与 ``orElse`` 组合。
