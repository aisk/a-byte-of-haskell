单子变换子（Monad Transformers）与 MTL 风格
================================================================================

在前面的章节中，我们分别接触了处理不同效果的单子：\ ``Maybe``\ （缺失）、\ ``Either``\ （错误）、\ ``Reader``\ （配置）、\ ``State``\ （状态）以及 ``IO``\ （副作用）。

实际项目里往往\ **需要同时拥有多种能力**\ ：一个 Web 接口既要读取配置、又要访问数据库、还要维护请求上下文，并在校验失败时提前返回错误。

为什么 Monad 不能自动复合？
--------------------------------------------------------------------------------

范畴论里有一个常被引用的事实：

- 两个 ``Functor`` 的嵌套组合仍然是 ``Functor``\ 。
- 两个 ``Applicative`` 的嵌套组合仍然是 ``Applicative``\ 。
- **但两个 Monad 的嵌套，一般不能自动组合成新的 Monad。**

原因在于 ``join``\ 。要让 ``m (n a)`` 成为 Monad，需要定义一个展平操作 ``m (n (m (n a))) -> m (n a)``\ 。在不知道 ``n`` 的具体结构时，没有办法通用地写出这个函数：中间那层 ``m`` 卡在两层 ``n`` 之间，只靠 ``m`` 和 ``n`` 各自的 ``join`` 是拿不掉的。

因此需要为每一种"外层能力"单独写一个知道如何跨过内层单子的版本，这就是\ **单子变换子（Monad Transformers）**\ 。

常见的变换子
--------------------------------------------------------------------------------

变换子通常以 ``T`` 结尾，定义在 ``transformers`` 与 ``mtl`` 库中：

- **MaybeT m a**\ ：给基础单子 ``m`` 附加缺失处理，内部是 ``m (Maybe a)``\ 。
- **ExceptT e m a**\ ：给基础单子 ``m`` 附加错误处理，内部是 ``m (Either e a)``\ 。
- **ReaderT r m a**\ ：给基础单子 ``m`` 附加只读环境，内部是 ``r -> m a``\ 。
- **StateT s m a**\ ：给基础单子 ``m`` 附加状态，内部是 ``s -> m (a, s)``\ 。

MonadTrans 与 lift
--------------------------------------------------------------------------------

把一个 Monad 放进变换子里之后，如何在外层调用底层 Monad 的操作？通过 ``MonadTrans`` 类型类的 ``lift``\ ：

.. code:: haskell

   class MonadTrans t where
     lift :: Monad m => m a -> t m a

``lift`` 把下层单子的动作"抬"一层，让它可以出现在外层的 ``do`` 代码块中。

MonadIO 与 liftIO
--------------------------------------------------------------------------------

大多数应用的单子栈最底层是 ``IO``\ 。为了避免多层嵌套时写 ``lift . lift . lift``\ ，标准库提供了 ``MonadIO``\ ：

.. code:: haskell

   class Monad m => MonadIO m where
     liftIO :: IO a -> m a

无论栈有多少层，一个 ``liftIO`` 就能把 ``IO`` 动作提升到当前层。

以三层栈 ``ReaderT Env (StateT S IO)`` 为例，\ ``lift`` 每次只向上抬一层，\ ``liftIO`` 则直接从底层的 ``IO`` 抬到最外层：

.. mermaid::

   graph BT
     IO["IO a<br/>最底层"]
     ST["StateT S IO a"]
     RT["ReaderT Env (StateT S IO) a<br/>最外层，业务代码所在"]
     IO -- "lift" --> ST
     ST -- "lift" --> RT
     IO -- "liftIO（一步到位）" --> RT

在最外层写 ``lift get`` 得到的是 ``StateT`` 的 ``get``\ ，写 ``lift (lift (putStrLn ...))`` 或 ``liftIO (putStrLn ...)`` 得到的是 ``IO`` 动作。使用 mtl 的类型类（如 ``MonadState``\ ）时，这些 ``lift`` 会由实例自动补上。

堆叠顺序的语义差异
--------------------------------------------------------------------------------

变换子的堆叠顺序不是无关紧要的，\ **不同顺序对应不同的错误处理语义**\ 。

对比以下两种组合：

1. **StateT 在内，ExceptT 在外（ExceptT e (State s) a）**\ ：

   - 展开后等价于 ``s -> (Either e a, s)``\ 。
   - **语义**\ ：即使抛出错误（\ ``throwError``\ ），状态仍然和错误一起返回，\ **错误发生前的状态修改会被保留**\ 。

2. **ExceptT 在内，StateT 在外（StateT s (Except e) a）**\ ：

   - 展开后等价于 ``s -> Either e (a, s)``\ 。
   - **语义**\ ：一旦抛出错误，整个 ``(a, s)`` 元组被丢弃，\ **之前的状态变更全部丢失**\ ，类似数据库事务回滚。

设计时需要根据业务是否要求"出错回滚"来决定嵌套顺序。

两种嵌套展开后的结构如下，区别在于 ``Either`` 包在状态元组的里面还是外面：

.. mermaid::

   graph TD
     subgraph A["ExceptT e (State s) a"]
       A1["s -#gt;"] --> A2["( Either e a , s )"]
       A2 --> A3["Left e：错误<br/>状态 s 仍然返回"]
       A2 --> A4["Right a：结果<br/>状态 s 返回"]
     end
     subgraph B["StateT s (Except e) a"]
       B1["s -#gt;"] --> B2["Either e ( a , s )"]
       B2 --> B3["Left e：错误<br/>没有状态，之前的修改丢失"]
       B2 --> B4["Right ( a , s )：结果与状态一起返回"]
     end

左边的组合出错时仍能拿到状态，右边的组合出错时状态一并丢弃。

先问一句：真的需要变换子吗
--------------------------------------------------------------------------------

变换子解决的是“多种能力叠在一起”的问题，但很多程序根本没有这个问题。按程序规模，常见的结构只有三种：

.. list-table::
   :header-rows: 1
   :widths: 24 40 36

   * - 程序规模
     - 推荐结构
     - 例子
   * - 脚本、小工具、单文件命令行程序
     - 裸 ``IO``\ ，配置放进一个记录类型，当普通参数传给需要它的函数
     - 第六部分的文件、命令行、JSON、HTTP、Socket 各章全是这么写的，没有一处用到变换子
   * - 纯算法内部：解释器、解析器、模拟器、多步校验
     - ``State``\ 、\ ``StateT``\ 、\ ``ExceptT`` 在算法边界内使用，对外仍然是普通函数
     - 一个 ``runParser :: String -> Either Err AST`` 内部可以是 ``StateT String (Either Err)``\ ，调用方看不到
   * - 长期运行、有共享资源的应用：服务、守护进程
     - 下一节的 ``ReaderT Env IO``\ ，一层变换子到底
     - Web 服务、数据库连接池、带日志和指标的后台任务

判断标准很简单：先用裸 ``IO`` 写，直到发现同一个配置或句柄要穿过三四层函数、每层都只是转交，再换成 ``ReaderT``\ 。\ ``StateT`` 和 ``ExceptT`` 属于算法内部的工具，不要拿它们搭应用骨架。

两点常见的误区：

- **MTL 约束风格有代价**\ 。后面会介绍用 ``(MonadReader Env m, MonadIO m) => m ()`` 描述能力的写法，它的好处是解耦和可测试，代价是签名变长、类型错误信息变得难读、编译变慢。业务函数直接写 ``App ()`` 往往更省事，只在确实需要换一个实现来测试时才升级成约束。
- **ExceptT e IO 是有争议的写法**\ 。\ ``IO`` 本身已经有异常机制，再叠一层 ``Either`` 意味着调用方要同时处理两套错误。业务上可预期的失败用 ``Either`` 返回，环境故障交给异常，这是错误处理一章给出的分工，IO 一章的 ``Control.Exception`` 部分讲的是后者。

ReaderT 模式
--------------------------------------------------------------------------------

过去不少代码会堆叠五六层变换子。现在更常见的做法是 **ReaderT 模式（The ReaderT Pattern）**\ ：

**思路**\ ：把应用的核心单子固定为 ``ReaderT Env IO``\ ，将数据库连接池、日志句柄、可变引用（\ ``IORef`` / ``TVar``\ ）都放进环境结构体 ``Env`` 里：

.. code:: haskell

   {-# LANGUAGE GeneralizedNewtypeDeriving #-}

   import Control.Monad.Reader

   data Env = Env
     { appPort  :: !Int
     , appDbUrl :: !String
     }

   -- 用 newtype 封装单层的 ReaderT Env IO
   newtype App a = App
     { unApp :: ReaderT Env IO a
     } deriving (Functor, Applicative, Monad, MonadReader Env, MonadIO)

   runApp :: Env -> App a -> IO a
   runApp env action = runReaderT (unApp action) env

   -- 业务函数直接拥有读环境和执行 IO 的能力：
   startServer :: App ()
   startServer = do
     port <- asks appPort
     liftIO $ putStrLn $ "服务在端口 " ++ show port ++ " 上启动"

这种模式只有一层变换子，结构简单，也避免了多层变换子带来的额外开销。可变状态通过 ``Env`` 里的 ``IORef`` 或 ``TVar`` 处理，错误则用 ``IO`` 异常或返回 ``Either``\ 。

MTL 风格：用类型类约束描述能力
--------------------------------------------------------------------------------

如果希望业务函数不绑定具体的单子栈，可以使用 MTL 风格：函数只通过类型类约束声明自己需要的能力：

- 需要读配置：\ ``MonadReader Env m``
- 需要抛错误：\ ``MonadError String m``
- 需要执行 IO：\ ``MonadIO m``

.. code:: haskell

   import Control.Monad.Reader
   import Control.Monad.Except

   pureBusinessLogic :: (MonadReader Env m, MonadError String m, MonadIO m) => m ()
   pureBusinessLogic = do
     port <- asks appPort
     liftIO $ putStrLn $ "当前端口: " ++ show port
     if port < 1024
       then throwError "非特权端口错误"
       else liftIO $ putStrLn "校验通过。"

这样写有两个好处：

1. **不用手写 lift**\ ：\ ``MonadReader``\ 、\ ``MonadError`` 等类型类的实例会自动穿过变换子层。
2. **便于测试**\ ：测试时可以换一个纯内存的单子来运行同一段业务逻辑，不需要真实的网络或数据库。

小结
--------------------------------------------------------------------------------

- Monad 一般不能自动复合，需要用变换子逐层叠加能力。
- ``lift`` 把下层动作提升一层，\ ``liftIO`` 直接提升到底层的 ``IO``\ 。
- 变换子的堆叠顺序决定出错时状态是保留还是丢弃。
- 小程序用裸 ``IO`` 加配置记录就够，\ ``StateT``\ 、\ ``ExceptT`` 留在算法内部。
- ``ReaderT Env IO`` 是当前常见的应用结构，只用一层变换子。
- MTL 风格用类型类约束描述函数需要的能力，便于解耦和测试。
