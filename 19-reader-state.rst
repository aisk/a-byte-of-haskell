状态与环境管理：Reader、Writer 与 State
================================================================================

在没有全局可变变量的前提下，如何处理"只读配置的传递"、"边计算边记录日志"以及"状态的逐步演化"？

Haskell 用三个单子分别回答这三个问题：\ **Reader**\ （只读环境）、\ **Writer**\ （累积输出）与 **State**\ （状态转移）。它们都是纯函数上的封装，没有任何魔法。

Reader：只读环境
--------------------------------------------------------------------------------

``Reader`` 本质上是对普通函数 ``r -> a`` 的 Monad 封装。概念上它等价于：

.. code:: haskell

   newtype Reader r a = Reader { runReader :: r -> a }

（在 ``mtl`` 里，\ ``Reader r`` 实际是 ``ReaderT r Identity`` 的别名，行为与上面的定义一致。）

- ``r``\ ：只读的\ **环境（Environment）**\ ，通常是配置。
- ``a``\ ：该计算最终产生的结果。

核心操作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.Reader`` 中：

- ``ask :: Reader r r``\ ：取出完整的环境。
- ``asks :: (r -> a) -> Reader r a``\ ：通过投影函数从环境中取出某个字段。
- ``local :: (r -> r) -> Reader r a -> Reader r a``\ ：在某个局部计算内部临时修改环境。
- ``runReader :: Reader r a -> r -> a``\ ：传入具体环境，执行计算并返回结果。

示例：配置注入
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
     -- 局部临时覆盖端口
     testConn <- local (\cfg -> cfg { dbPort = 9999 }) connectDatabase
     realConn <- connectDatabase
     return $ "测试连接: " ++ testConn ++ " | 正式连接: " ++ realConn

.. code:: text

   ghci> runReader mainTask (AppConfig "localhost" 5432)
   "测试连接: 成功连接至 localhost:9999 | 正式连接: 成功连接至 localhost:5432"

.. tip::

   **如果你熟悉其他语言：Reader 就是依赖注入**\ ：

   - **依赖注入（Dependency Injection）**\ ：Spring、Dagger 之类的框架通过反射或注解容器把依赖注入到对象里。\ ``Reader`` 做的是同一件事，只是完全靠函数和类型完成，没有运行期反射，注入了什么在类型签名里就能看到。
   - **类似机制**\ ：Go 里逐层显式传递的 ``context.Context``\ ，或 React 的 ``Context API``\ （在组件树顶层提供配置，子节点按需读取，避免一层层手动传 props）。

Writer：累积日志
--------------------------------------------------------------------------------

当计算除了产出结果，还需要在执行过程中逐步累积一些附加输出（日志、追踪信息）时，可以使用 ``Writer``\ ：

.. code:: haskell

   newtype Writer w a = Writer { runWriter :: (a, w) }

- ``w`` 必须是一个 ``Monoid``\ （如列表、字符串或数值和），这样每一步产生的输出都能用 ``(<>)`` 合并。

核心操作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.Writer`` 中：

- ``tell :: Monoid w => w -> Writer w ()``\ ：追加一段输出。
- ``runWriter :: Writer w a -> (a, w)``\ ：执行计算，返回 ``(结果, 累积的输出)``\ 。

示例：带执行日志的欧几里得算法
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

   **Writer 的空间泄漏问题**\ ：
   默认的 ``Control.Monad.Writer``\ （即 ``Control.Monad.Writer.Lazy``\ ）在每次绑定时以惰性二元组的形式累积输出。在几十万次的高频循环里，未求值的 Thunk 链会持续堆积，内存占用随之增长。
   如果需要在长循环中收集日志，通常的做法是改用基于 CPS 的 ``Control.Monad.Writer.CPS``\ ，或者用 ``State`` 严格累加，或者直接在 ``IO`` 中写入文件或通道。

State：状态转移
--------------------------------------------------------------------------------

命令式语言可以直接覆写内存变量（如 ``x += 1``\ ）。在纯函数式语言中，有状态的计算被建模为\ **状态转移函数**\ ：接收旧状态，产出结果和新状态：

.. code:: haskell

   newtype State s a = State { runState :: s -> (a, s) }

- ``s``\ ：状态类型。
- ``a``\ ：本次计算产生的结果。

核心操作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义在 ``Control.Monad.State`` 中：

- ``get :: State s s``\ ：读取当前状态。
- ``gets :: (s -> a) -> State s a``\ ：从当前状态中取出某个投影。
- ``put :: s -> State s ()``\ ：用新状态替换当前状态。
- ``modify :: (s -> s) -> State s ()``\ ：用函数更新当前状态。
- ``state :: (s -> (a, s)) -> State s a``\ ：从状态转移函数构造 State。
- ``runState :: State s a -> s -> (a, s)``\ ：传入初始状态，返回 ``(结果, 最终状态)``\ 。
- ``evalState :: State s a -> s -> a``\ ：只取结果。
- ``execState :: State s a -> s -> s``\ ：只取最终状态。

示例：栈
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Monad.State

   type Stack = [Int]

   -- 用 case 显式处理空栈，而不是在 do 块里写 (x:xs) <- get
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

下图按执行顺序列出 ``stackDemo`` 每一步之后的栈内容，以及这一步在 ``do`` 块中绑定到的返回值：

.. mermaid::

   graph LR
     S0["初始栈<br/>[]"] -- "push 10<br/>返回 ()" --> S1["[10]"]
     S1 -- "push 20<br/>返回 ()" --> S2["[20, 10]"]
     S2 -- "pop<br/>mA = Just 20" --> S3["[10]"]
     S3 -- "pop<br/>mB = Just 10" --> S4["[]"]
     S4 -- "push 30<br/>返回 ()" --> S5["[30]"]
     S5 -- "pop<br/>结果 Just 30" --> S6["最终栈<br/>[]"]

每个箭头就是一次状态转移函数 ``s -> (a, s)`` 的应用，\ ``>>=`` 负责把上一步产出的新状态传给下一步。

.. note::

   为什么 ``pop`` 不直接写 ``(x:xs) <- get``\ ？在 ``do`` 块中使用可失败的模式匹配，编译器会要求该单子有 ``MonadFail`` 实例，而 ``State`` 没有。用 ``case`` 显式处理每个分支既能通过编译，也把空栈的情况明确写在了类型里。

.. tip::

   **如果你熟悉其他语言：State 就是状态机**\ ：

   - **Redux Reducer / Elm 架构**\ ：前端常见的状态转移函数 ``(state, action) -> newState`` 与 ``State`` 单子是同一种建模方式。
   - **和可变变量的区别**\ ：\ ``State`` 并不是并发工具，它是单线程的纯计算。它的价值在于状态的每一次变化都显式地经过 ``get``\ 、\ ``put`` 或 ``modify``\ ，可以在类型和代码里追踪，而不是散落在各处的赋值语句。真正的并发共享状态需要 ``IORef``\ 、\ ``MVar`` 或 ``STM``\ ，见后面的并发章节。

小结
--------------------------------------------------------------------------------

- ``Reader`` 把只读配置隐式传递给一组计算，避免逐层传参。
- ``Writer`` 沿途累积一个 Monoid 输出。惰性版本在长循环中会堆积 Thunk，需要时改用 CPS 版本。
- ``State`` 把状态转移封装为 ``s -> (a, s)``\ ，用 ``get`` / ``put`` / ``modify`` 读写状态。
- ``State`` 没有 ``MonadFail`` 实例，do 块里不要用可失败模式，用 ``case`` 显式分支。
