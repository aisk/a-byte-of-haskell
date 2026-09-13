并发编程与异步运行时调度
================================================================================

在很多主流语言中，高并发编程往往伴随着复杂的两难抉择：要么使用操作系统的内核原生线程，承受巨大的内存与上下文切换开销；要么使用基于异步回调（Event Loop）的模型，承受“函数着色（Function Coloring）”与嵌套代码的折磨。

Haskell 的编译器 GHC 采用了一套极为先进的\ **用户态 M:N 调度运行时**\ 。它在底层将类似 Node.js、Python asyncio 的事件循环异步机制，与同步直观的代码风格完美融合，为现代多核高并发开发提供了卓越的工程体验。

GHC 绿色线程：轻量级并发基石
--------------------------------------------------------------------------------

现代操作系统的内核线程（1:1 模型）通常需要分配 1MB ~ 8MB 的栈内存，且线程切换需要穿越用户态与内核态的上下文屏障，单个进程并发数千个线程就可能逼近系统极限。

GHC 运行时的设计哲学完全不同：

- **极小初始开销**\ ：GHC 的线程是完全在用户态调度的 **绿色线程（Green Threads）**\ ，初始堆栈内存仅约 **1KB**；
- **百万级高并发**\ ：单个 64 位进程可以轻松创建几十万甚至上百万个绿色线程，并支持按需自动伸缩栈内存；
- **极速调度**\ ：上下文切换纯粹在用户空间完成，开销仅需几纳秒。

使用 forkIO 创建轻量线程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``Control.Concurrent`` 中，可以使用 ``forkIO`` 启动一个轻量级线程：

.. code:: haskell

   import Control.Concurrent
   import Control.Monad (forever)

   simpleThreadDemo :: IO ()
   simpleThreadDemo = do
     -- 启动一个后台工作线程
     threadId <- forkIO $ forever $ do
       putStrLn "后台守护线程工作中..."
       threadDelay 1000000 -- 暂停 1 秒（单位是微秒）

     putStrLn $ "已成功启动轻量线程，ID: " ++ show threadId
     threadDelay 3000000 -- 主线程等待 3 秒后退出
     putStrLn "主线程退出，全部轻量线程随之终止。"

底层异步秘密：I/O Manager 与 epoll 事件循环
--------------------------------------------------------------------------------

你可能会好奇：如果大量绿色线程都在调用 ``getLine`` 或等待网络 Socket，难道不会把 CPU 和系统线程耗尽吗？

这正是 GHC 运行时最强悍的特性之一：**看似写的是同步代码，底层执行的其实是高性能异步非阻塞 I/O**。

在底层，GHC 运行时内置了一个基于操作系统高性能多路复用 API 的 **I/O Manager（事件管理器）**\ ：

- 在 Linux 上使用 **epoll**\ ；
- 在 macOS / BSD 上使用 **kqueue**\ ；
- 在 Windows 上使用 **IOCP**\ 。

当一个 Haskell 绿色线程执行网络读取时，它并不会让底层的操作系统线程陷入休眠；相反，它会向 GHC 的 I/O Manager 注册一个“监听读就绪”的事件，随后该绿色线程立刻挂起让出控制权，GHC 调度器转而毫秒级切换去执行其他就绪的绿色线程。当内核的 `epoll` 通知数据就绪后，I/O Manager 会自动唤醒挂起的绿色线程恢复执行。

.. tip::

   **如果你熟悉其他语言：消灭“函数着色问题”的高性能异步**\ ：

   - **对比 Python asyncio / C# / JavaScript**\ ：
     在这些语言中，异步依靠事件循环（Event Loop）驱动。但它们普遍面临臭名昭著的\ **“函数着色问题（Function Coloring Problem）”**\ ——一旦底层某个函数被声明为 ``async``\ ，整个调用链上的所有上层函数都必须被迫加上 ``async`` 与 ``await`` 关键字，同步与异步世界被割裂成了无法轻易混用的两种颜色。
   - **Haskell 的降维打击**\ ：
     在 Haskell 中，所有 I/O 统一使用纯粹的 ``IO`` 抽象。你无需考虑函数是同步还是异步，直接以平铺直叙的同步顺序书写代码，GHC 运行时在后台自动通过 `epoll` 完成了全套非阻塞异步转换！

操作系统阻塞 I/O 与 -threaded 运行时
--------------------------------------------------------------------------------

阻塞 I/O 的现实困境
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

尽管网络通信具备成熟的 `epoll` 等异步机制，但在现代操作系统内核中，**本地文件系统磁盘读写、DNS 域名解析以及大量外部 C 语言动态链接库（FFI）调用，本质上仍然是强制阻塞的**。

如果使用 GHC 的默认单线程运行时（Single-threaded RTS）：
一旦某一个绿色线程触发了内核阻塞调用（例如执行了一个耗时很长的磁盘读盘或 C 语言系统调用），**整个进程唯一的操作系统线程将被内核直接挂起**！这将导致进程内的其余几十万个绿色线程全部陷入卡死停摆。

拯救方案：启用多线程运行时（-threaded）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了彻底解决这一问题，现代 Haskell 生产项目在编译时\ **强烈建议启用 ``-threaded`` 编译选项**\ ：

.. code:: sh

   $ ghc -O2 -threaded -rtsopts -with-rtsopts="-N" MyApp.hs

- ``-threaded``\ ：将运行时切换为多操作系统的 **M:N 混合调度器**。
- ``-rtsopts -with-rtsopts="-N"``\ ：让 GHC 运行时自动感知机器的物理核心数，启动对应数量的操作系统工作线程（OS Capability），实现真正的多核硬件并行。

.. tip::

   **如果你熟悉其他语言：类比 Go 的调度器与 Rust 的 Tokio**\ ：

   - **Go 语言的 GMP 模型**\ ：Go 语言的 Goroutine 在遇到阻塞系统调用时，调度器会自动将阻塞的 M（系统线程）与 P（处理器）解绑，并新建或唤醒另一个系统线程来接管其他等待的 Goroutine。
   - **Rust 的 Tokio 异步运行时**\ ：Tokio 在遇到无法异步的本地文件或计算密集型任务时，要求显式使用 ``tokio::task::spawn_blocking`` 将任务移交到底层阻塞线程池。
   - **Haskell 的多线程运行时**\ ：原理完全同构。GHC 的 M:N 调度器在操作系统原生支持异步（如网络通信）时充分利用 `epoll` 事件循环复用；当遇到必须阻塞的系统调用时，自动交由独立的操作系统线程承担，保证其他 CPU 核心上的 OS 线程继续调度执行其余绿色线程，绝不产生全局停顿。

结构化并发：Control.Concurrent.Async
--------------------------------------------------------------------------------

虽然原生的 ``forkIO`` 极度轻量，但命令式地手动管理线程生命周期容易引发严重弊端：子线程返回值难以获取、未捕获异常导致线程静默消亡、父线程退出时子线程泄漏等。

现代工业界统一推荐使用 **Async** 库提供的\ **结构化并发（Structured Concurrency）**\ 范式。

并发同时执行两个任务：concurrently
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

``concurrently`` 会在两个独立的绿色线程中并发运行任务，等待双方全部完成后将结果组装为二元组返回。**如果其中任何一个任务抛出异常，另一个任务会立即被自动取消，杜绝孤儿协程泄漏**。

竞态竞赛：race
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在分布式高可用或超时兜底场景中，我们常常需要发起“竞态”：

.. code:: haskell

   -- 哪个任务先执行完毕就取哪个的结果，自动取消并杀死另一个落后者：
   fetchWithTimeout :: IO (Either () String)
   fetchWithTimeout =
     race (threadDelay 500000) fetchUserData

- 若 500ms 计时先完成，返回 ``Left ()``\ （代表超时）；
- 若网络请求先完成，返回 ``Right "用户档案数据"``\ ，计时器被自动销毁。

纯函数式无死锁状态：软件事务内存（STM）
--------------------------------------------------------------------------------

传统多线程编程通常通过互斥锁（Mutex / Lock）来保护共享可变内存。然而，基于锁的并发模型有着臭名昭著的缺陷：

- **死锁（Deadlock）**\ ：两个线程以不同的顺序申请锁 A 与锁 B，极易引发死锁；
- **锁无法自由组合**\ ：如果模块 X 与模块 Y 各自是线程安全的，将它们组合成一个复合操作（如从账户 A 转账到账户 B）通常必须暴露内部锁细节或引入更粗粒度的全局大锁，破坏封装性与并发吞吐；
- **竞态与条件变量的噩梦**\ ：传统 ``wait()`` 与 ``notify()``\ / 条件变量不仅难以编写，还容易出现虚假唤醒（Spurious Wakeup）与丢失唤醒。

Haskell 首创了将数据库事务（ACID）思想引入内存并发的顶尖方案——\ **软件事务内存（Software Transactional Memory, STM）**\ 。

核心原语：TVar、atomically 与 retry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``Control.Concurrent.STM`` 中：

.. code:: haskell

   import Control.Concurrent.STM

   -- 1. 事务性变量：只能在 STM 单子内读写
   type BankAccount = TVar Int

   -- 2. 纯事务动作（注意：类型是 STM ()，绝非 IO！）
   transfer :: BankAccount -> BankAccount -> Int -> STM ()
   transfer fromAcc toAcc amount = do
     fromBal <- readTVar fromAcc
     if fromBal < amount
       then retry -- 余额不足：智能挂起当前线程，等待相关 TVar 变动后再试！
       else do
         writeTVar fromAcc (fromBal - amount)
         toBal <- readTVar toAcc
         writeTVar toAcc (toBal + amount)

   -- 3. 在 IO 中原子化提交事务
   executeTransfer :: BankAccount -> BankAccount -> IO ()
   executeTransfer a b = atomically (transfer a b 100)

事务回退与备选分支：orElse
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

STM 另一个惊艳工业界的特性是组合子 ``orElse``\ ：

.. code:: haskell

   -- 优先从主账户取款，若余额不足（触发 retry），自动平滑尝试从备用账户取款：
   smartWithdraw :: BankAccount -> BankAccount -> Int -> BankAccount -> STM ()
   smartWithdraw primary secondary amount target =
     transfer primary target amount `orElse` transfer secondary target amount

基于锁的模型根本无法实现类似 ``orElse`` 的能力，因为在传统锁模型中，若第 1 步失败，很难自动回滚已经获取的锁并无副作用地尝试第 2 步。

.. tip::

   **如果你熟悉其他语言：为什么只有 Haskell 的 STM 真正成功了？**\ ：

   - **命令式语言的惨痛教训**\ ：
     STM 概念早在学术界被提出，并在 Java、C++ 等命令式语言中尝试过工程落地，但无一例外走向了衰落甚至失败。**根本原因在于命令式语言无法阻止在事务内执行不可逆的物理副作用**\ ！
     如果在 Java 事务内部调用了打印、网络请求甚至硬件控制，当事务检测到数据冲突需要“回滚（Rollback）”时，运行时根本无法“撤销”已经发送的网络包或打印出去的日志。
   - **类型系统铸就的函数式护城河**\ ：
     Haskell 的类型系统将副作用隔离到了极致——\ **在 ``STM`` 单子内，编译器严禁包含任何 ``IO`` 动作**\ ！你在 ``STM`` 中所做的一切，仅仅是对 ``TVar`` 读写日志的纯内存记录。因为绝无不可逆的物理副作用，GHC 运行时可以随心所欲、百分之百安全地将事务**透明回滚并重试**（乐观并发控制，Optimistic Concurrency Control）。
   - **对比各大主流语言的并发状态模型**\ ：

     - **Go**\ ：推荐通过 Channel 传递数据（CSP 模型），但在处理复杂共享状态时仍需依赖 ``sync.Mutex``\ ，死锁与锁竞态风险依然由开发者人工肉身防范。
     - **Rust**\ ：通过所有权类型系统（\ ``Arc<Mutex<T>>``\ ）在编译期消灭了数据竞争（Data Race），但在运行期依然可能因为加锁顺序不当发生死锁。
     - **Haskell STM**\ ：将并发状态推升到了数据库级别的\ **可串行化（Serializable）隔离级别**\ ——无死锁、无锁竞态、支持声明式条件重试（\ ``retry``\ ）与任意细粒度原子组合（\ ``orElse``\ ）。

小结
--------------------------------------------------------------------------------

- GHC 的用户态绿色线程栈开销仅约 1KB，支持单机百万级高并发（``forkIO``）。
- 底层依靠基于 epoll / kqueue 的 I/O Manager 实现高性能异步事件循环，无需开发者手动进行“函数着色”。
- 遇到文件与系统级强制阻塞 I/O 时，必须开启 **``-threaded``** 编译选项，交由多操作系统工作线程（M:N）进行并发分担，对标 Go 调度器与 Rust Tokio。
- 生产环境推崇使用 **``Control.Concurrent.Async``** 实施结构化并发，杜绝协程泄漏与孤儿任务。
- **STM（软件事务内存）**\ 将数据库 ACID 事务机制引入多线程，从根本上消灭了传统死锁与锁竞态难题。
