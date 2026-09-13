状态与环境管理：Reader、Writer 与 State
================================================================================

在没有全局可变变量的前提下，如何优雅地处理“全局只读配置传递”、“边计算边记录日志”以及“动态状态机流转”？

Haskell 通过 **Reader**\ （只读环境注入）、\ **Writer**\ （纯函数式日志审计）与 **State**\ （纯函数式状态转移）三剑客，给出了高度严密、类型安全的解答。

Reader：纯函数式只读环境注入
--------------------------------------------------------------------------------

在底层，\ ``Reader`` 本质上就是对普通函数 ``r -> a`` 的 Monad 化封装：

.. code:: haskell

   newtype Reader r a = Reader { runReader :: r -> a }

- ``r``\ ：表示只读的\ **全局配置/环境（Environment）**\ 。
- ``a``\ ：表示该计算最终产生的纯值结果。

核心操作接口
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.Reader`` 中：

- ``ask :: Reader r r``\ ：提取完整的环境配置。
- ``asks :: (r -> a) -> Reader r a``\ ：从环境中通过投影函数抽取局部字段。
- ``local :: (r -> r) -> Reader r a -> Reader r a``\ ：在指定局部计算内部，临时修改环境配置。
- ``runReader :: Reader r a -> r -> a``\ ：传入具体配置，执行计算并返回结果。

实战示例：配置注入
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Monad.Reader

   data AppConfig = AppConfig
     { dbHost :: String
     , dbPort :: Int
     } deriving (Show)

   connectDatabase :: Reader AppConfig String
   connectDatabase = do
     host <- asks dbHost
     port <- asks dbPort
     return $ "成功连接至 " ++ host ++ ":" ++ show port

   mainTask :: Reader AppConfig String
   mainTask = do
     -- 局部临时覆盖端口进行隔离测试
     testConn <- local (\cfg -> cfg { dbPort = 9999 }) connectDatabase
     realConn <- connectDatabase
     return $ "测试连接: " ++ testConn ++ " | 正式连接: " ++ realConn

.. code:: text

   ghci> runReader mainTask (AppConfig "localhost" 5432)
   "测试连接: 成功连接至 localhost:9999 | 正式连接: 成功连接至 localhost:5432"

.. tip::

   **如果你熟悉其他语言：Reader 的本质是依赖注入**\ ：

   - **纯函数式依赖注入（Dependency Injection / DI 容器）**\ ：在 Spring 或 Dagger 等传统面向对象框架中，依赖注入通常依赖运行期反射或魔法般的注解容器。而在 Haskell 中，\ ``Reader`` 是\ **纯函数式、编译期完全类型安全且零反射开销**\ 的环境注入方案。
   - **类比机制**\ ：类似于 Go 语言中层层显式透传的 ``context.Context``\ ，或是 React 前端框架中的 ``Context API``\ （在组件树顶层注入配置，子树节点按需自取，彻底消灭逐层手动传递的 Props Drilling）。

Writer：纯函数式日志审计与工程警示
--------------------------------------------------------------------------------

当计算不仅需要产出结果，还需要在执行过程中逐步累积生成审计日志、追踪信息时，使用 ``Writer``\ ：

.. code:: haskell

   newtype Writer w a = Writer { runWriter :: (a, w) }

- ``w`` 必须是一个 ``Monoid``\ （如列表、字符串或累加数值），这样每一步生成的日志都能通过 ``(<>)`` 自动合并。

核心操作接口
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.Writer`` 中：

- ``tell :: Monoid w => w -> Writer w ()``\ ：追加一段日志。
- ``runWriter :: Writer w a -> (a, w)``\ ：执行计算，同时返回 ``(最终结果, 完整日志)``\ 。

实战示例：带执行日志的欧几里得算法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Monad.Writer

   gcdWithLog :: Int -> Int -> Writer [String] Int
   gcdWithLog a b
     | b == 0 = do
         tell ["到达终点，最大公约数为: " ++ show a]
         return a
     | otherwise = do
         tell [show a ++ " 对 " ++ show b ++ " 取模得到 " ++ show (a `mod` b)]
         gcdWithLog b (a `mod` b)

.. code:: text

   ghci> runWriter (gcdWithLog 21 14)
   (7,["21 对 14 取模得到 7","14 对 7 取模得到 0","到达终点，最大公约数为: 7"])

.. warning::

   **工程警示：Writer 的空间泄漏隐患**\ ：
   标准库中默认的 ``Control.Monad.Writer.Lazy`` 在每次绑定时以惰性二元组的形式累积日志。在处理数十万次的高频日志循环中，未求值的日志 Thunk 链会在堆内存中迅速爆炸，引发严重的内存泄漏。
   **工业最佳实践**\ ：在高性能生产系统中，若需收集日志，推荐使用带连续传递风格的 ``Control.Monad.Writer.CPS``\ ，或直接使用严格累加的 ``State``\ ，或者在底层的 ``IO`` 中借助并发通道写入磁盘。

State：纯函数式状态转移机
--------------------------------------------------------------------------------

命令式语言可以直接对内存变量进行多次覆写（如 ``x += 1``\ ）。而在纯函数式语言中，有状态的计算被严格建模为\ **状态转移函数**\ ：接收一个旧状态，产出结果和演变后的新状态：

.. code:: haskell

   newtype State s a = State { runState :: s -> (a, s) }

- ``s``\ ：状态类型。
- ``a``\ ：本次计算产生的结果。

核心操作接口
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.State`` 中：

- ``get :: State s s``\ ：提取当前状态。
- ``gets :: (s -> a) -> State s a``\ ：从当前状态中抽取特定属性投影。
- ``put :: s -> State s ()``\ ：用新状态完全覆盖当前状态。
- ``modify :: (s -> s) -> State s ()``\ ：使用纯函数就地更新当前状态。
- ``state :: (s -> (a, s)) -> State s a``\ ：从原子转移函数构造 State。
- ``runState :: State s a -> s -> (a, s)``\ ：传入初始状态，返回 ``(结果, 最终状态)``\ 。
- ``evalState :: State s a -> s -> a``\ ：只获取结果。
- ``execState :: State s a -> s -> s``\ ：只获取最终状态。

实战示例：纯函数式内存堆栈
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Monad.State

   type Stack = [Int]

   -- 使用安全的模式匹配，杜绝 MonadFail 编译错误与运行期崩溃
   pop :: State Stack (Maybe Int)
   pop = do
     s <- get
     case s of
       []     -> return Nothing
       (x:xs) -> do
         put xs
         return (Just x)

   push :: Int -> State Stack ()
   push x = modify (x :)

   stackDemo :: State Stack (Maybe Int)
   stackDemo = do
     push 10
     push 20
     mA <- pop
     mB <- pop
     case (mA, mB) of
       (Just a, Just b) -> do
         push (a + b)
         pop
       _ -> return Nothing

在 GHCi 中验证：

.. code:: text

   ghci> runState stackDemo []
   (Just 30,[])
   ghci> evalState stackDemo []
   Just 30

.. tip::

   **如果你熟悉其他语言：State 的本质是纯状态机**\ ：

   - **纯状态机与 Redux Reducer**\ ：前端或微服务架构中广泛流行的 **Redux / Elm 架构**\ （状态转移函数 ``(state, action) -> (newState, result)``\ ），本质上正是 ``State`` 单子的具象化工程落地。
   - **天然杜绝并发数据竞争（Data Race）**\ ：在 Java、Go 等多线程语言中，共享可变状态必须依赖重量级的互斥锁（Mutex）或原子变量；而在 Haskell 中，\ ``State`` 本质上是数学函数的纯净组合，没有物理内存的脏写覆写，在单子内天然具备抗并发数据污染的纯函数特性。

小结
--------------------------------------------------------------------------------

- ``Reader`` 优雅实现全局只读配置的隐式向下传递，解耦函数参数。
- ``Writer`` 收集审计跟踪日志，但需警惕惰性版本的空间泄漏，生产推荐 CPS 版本。
- ``State`` 将状态流转封装为纯代数管道，完全消灭全局可变变量与并发数据竞态。
- 在 ``State`` 中进行模式匹配时，使用显式的安全全函数分支（如 ``Maybe``\ ）可确保现代编译器的严格通过。
