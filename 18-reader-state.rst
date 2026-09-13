状态与环境管理：Reader、Writer 与 State
================================================================================

在没有全局可变变量的前提下，如何优雅地处理“全局只读配置传递”、“边计算边记录日志”以及“动态状态流转”？

Haskell 通过 **Reader**\ （只读环境注入）、\ **Writer**\ （纯函数式日志审计）与 **State**\ （纯函数式状态机）三剑客，给出了高度严密、类型安全的解答。

Reader：纯函数式只读环境注入
--------------------------------------------------------------------------------

在底层，\ ``Reader`` 本质上就是对普通函数 ``r -> a`` 的 Monad 化封装：

.. code:: haskell

   newtype Reader r a = Reader { runReader :: r -> a }

- ``r``\ ：表示只读的\ **全局配置/环境（Environment）**\ 。
- ``a``\ ：表示该计算最终产生的纯值结果。

核心操作接口
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.Reader`` 中：

- ``ask :: Reader r r``\ ：提取完整的环境配置。
- ``asks :: (r -> a) -> Reader r a``\ ：从环境中通过函数抽取局部字段。
- ``local :: (r -> r) -> Reader r a -> Reader r a``\ ：在指定计算内部，局部临时修改环境配置。
- ``runReader :: Reader r a -> r -> a``\ ：传入具体配置，执行计算并返回结果。

实战示例：配置注入
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
     -- 局部临时覆盖端口进行测试
     testConn <- local (\cfg -> cfg { dbPort = 9999 }) connectDatabase
     realConn <- connectDatabase
     return $ "测试连接: " ++ testConn ++ " | 正式连接: " ++ realConn

.. code:: text

   ghci> runReader mainTask (AppConfig "localhost" 5432)
   "测试连接: 成功连接至 localhost:9999 | 正式连接: 成功连接至 localhost:5432"

Writer：纯函数式日志审计与输出
--------------------------------------------------------------------------------

当计算不仅需要产出结果，还需要在执行过程中逐步累积生成审计日志、追踪信息时，使用 ``Writer``\ ：

.. code:: haskell

   newtype Writer w a = Writer { runWriter :: (a, w) }

- ``w`` 必须是一个 ``Monoid``\ （如列表、字符串或累加数值），这样每一步生成的日志都能通过 ``(<>)`` 自动合并。

核心操作接口
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.Writer`` 中：

- ``tell :: Monoid w => w -> Writer w ()``\ ：追加一段日志。
- ``runWriter :: Writer w a -> (a, w)``\ ：执行计算，同时返回 ``(最终结果, 完整日志)``\ 。

实战示例：带执行日志的算术计算
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

State：纯函数式状态转移机
--------------------------------------------------------------------------------

命令式语言可以直接对内存变量进行多次写覆盖（如 ``x += 1``\ ）。而在纯函数式语言中，有状态的计算被严格建模为\ **状态转移函数**\ ：接收一个旧状态，产出结果和演变后的新状态。

.. code:: haskell

   newtype State s a = State { runState :: s -> (a, s) }

- ``s``\ ：状态类型。
- ``a``\ ：本次计算产生的结果。

核心操作接口
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.State`` 中：

- ``get :: State s s``\ ：提取当前状态。
- ``put :: s -> State s ()``\ ：用新状态完全覆盖当前状态。
- ``modify :: (s -> s) -> State s ()``\ ：使用纯函数就地更新当前状态。
- ``runState :: State s a -> s -> (a, s)``\ ：传入初始状态，返回 ``(结果, 最终状态)``\ 。
- ``evalState :: State s a -> s -> a``\ ：只获取结果。
- ``execState :: State s a -> s -> s``\ ：只获取最终状态。

实战示例：纯函数式计数器与内存堆栈
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Monad.State

   type Stack = [Int]

   pop :: State Stack Int
   pop = do
     (x:xs) <- get
     put xs
     return x

   push :: Int -> State Stack ()
   push x = modify (x :)

   stackDemo :: State Stack Int
   stackDemo = do
     push 10
     push 20
     a <- pop
     b <- pop
     push (a + b)
     pop

.. code:: text

   ghci> runState stackDemo []
   (30,[])
