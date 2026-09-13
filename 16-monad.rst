单子（Monad）与 do 记号深度解析
================================================================================

在函数式编程的世界里，“Monad”曾被赋予了过多神秘的隐喻。然而撕开抽象的外衣，Monad 的数学与工程本质极其明确：\ **它定义了如何将产生上下文的计算串联起来，并在每一步计算中自动扁平化嵌套的上下文层级**\ 。

为什么需要 Monad？破除层层嵌套
--------------------------------------------------------------------------------

假设我们在构建一个电商系统的订单查询管道。每一阶段都可能因数据不存在而返回 ``Maybe``\ ：

.. code:: haskell

   getUser   :: String -> Maybe User
   getOrder  :: User -> Maybe Order
   getPay    :: Order -> Maybe Payment

如果我们仅仅使用 ``fmap``\ ：

.. code:: text

   fmap getOrder (getUser "Alice")
   -- 返回类型变成了 Maybe (Maybe Order)！

如果继续链式调用，返回值将被层层嵌套为令人绝望的 ``Maybe (Maybe (Maybe Payment))``\ 。普通函数式机制无法把产生新上下文的函数与已有上下文拍平。为了解决这一痛点，我们引入了 **Monad**\ 。

Monad 的形式化定义
--------------------------------------------------------------------------------

定义在标准 Prelude 中：

.. code:: haskell

   class Applicative m => Monad m where
     -- 将普通值放入最小上下文（等价于 Applicative 的 pure）
     return :: a -> m a
     return = pure

     -- 核心绑定操作符（Bind）
     (>>=) :: m a -> (a -> m b) -> m b

     -- 忽略前项返回值的顺序执行
     (>>) :: m a -> m b -> m b
     m >> k = m >>= \_ -> k

核心操作符 ``>>=``\ （读作 **Bind**\ ）的核心职责是：
**从上下文 m a 中提取出纯值 a，将其传给产生新上下文的函数 (a -> m b)，并将结果在同一层级内自然返回，绝不产生嵌套。**

join 的等价代数视角
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Monad 还可以完全通过 ``join`` 函数来理解：

.. code:: haskell

   join :: Monad m => m (m a) -> m a

``join`` 的唯一职责就是将两层相同的上下文拍平成单层（例如将 ``[[a]]`` 拍平为 ``[a]``\ ，将 ``Just (Just x)`` 拍平为 ``Just x``\ ）。在范畴论中，\ ``m >>= f`` 完全等价于 ``join (fmap f m)``\ 。

do 记号与脱糖规则
--------------------------------------------------------------------------------

连续书写 ``>>=`` 与匿名函数会使代码向右倾斜。Haskell 提供了语法糖 **do 记号**\ ，让具有时序依赖的纯函数计算能够以命令式的清晰外观呈现：

.. code:: haskell

   getFinalPayment :: String -> Maybe Payment
   getFinalPayment name = do
     user    <- getUser name
     order   <- getOrder user
     payment <- getPay order
     return payment

编译器对 ``do`` 代码块的脱糖规则极其机械透明：

1. **带箭头提取**\ ：\ ``x <- m; rest`` 展开为 ``m >>= \x -> rest``
2. **不带提取的顺序执行**\ ：\ ``m1; m2`` 展开为 ``m1 >> m2``
3. **局部纯计算**\ ：\ ``let x = val`` 展开为常规局部绑定。

因此，上面清爽的 ``do`` 代码在底层编译后就是一条完全纯净的数学管道：

.. code:: haskell

   getFinalPayment name =
     getUser name >>= \user ->
       getOrder user >>= \order ->
         getPay order

.. tip::

   **他山之石：多语言心智模型对照**\ ：

   - **直觉通俗化**\ ：``FlatMappable`` / ``Chainable``\ （支持自动拍平的动态链式调用）。
   - **跨语言映射**\ ：
     - **JavaScript / TypeScript**\ ：``Promise.prototype.then(...)``\ 。注意：在 JS 中，若你在 ``then`` 的回调中返回一个新 Promise，运行时会自动将其展平，绝不会产生 ``Promise<Promise<T>>``——这正是 Monad 拍平嵌套上下文的核心能力！
     - **Java**\ ：``Optional.flatMap(...)``\ 、\ ``Stream.flatMap(...)``\ 、\ ``CompletableFuture.thenCompose(...)``\ 。
     - **Rust**\ ：``Option::and_then(...)``\ 、\ ``Result::and_then(...)``\ ，以及广受好评的 ``?`` 错误传播操作符（``?`` 本质上就是在 ``Result`` 单子中执行带提前短路返回的 ``>>=``\ ）。
   - **“可编程的分号”**\ ：
     - 在 C、Java 或 Go 等传统命令式语言中，语句间的分号 ``;`` 只代表机械的顺序流转。
     - 而在 Haskell 的 ``do`` 记号中，\ **换行与分号是可编程的上下文拦截切面**\ ——在迈入下一行之前，单子会自动完成环境预检：在 ``Maybe`` 中自动判定是否为空并短路，在 ``Either`` 中自动拦截错误，在 ``State`` 中自动传递演化状态，在 ``IO`` 中安全编排副作用。

现代 Haskell 的模式匹配失败机制：MonadFail
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``do`` 记号中，如果箭头左侧书写了可失败的模式匹配（例如 ``Just x <- action``\ ），若匹配失败，编译器会将其脱糖为调用 ``fail`` 函数：

.. code:: haskell

   -- 脱糖逻辑：
   action >>= \case
     Just x -> rest
     _      -> fail "Pattern match failure in do expression"

在现代 Haskell（GHC 8.8+ 及 MonadFail 提案）中，\ ``fail`` 函数被从基础 ``Monad`` 中彻底剥离，归属于独立的 ``Control.Monad.Fail.MonadFail`` 类型类。这从类型系统层面保证了：\ **只有真正支持故障回退的单子（如 Maybe、IO、列表），才允许在 do 块中使用可失败模式**\ ，彻底消灭了以往在不支持失败的单子中调用 ``fail`` 导致未定义崩溃的隐患。

Monad 三大法则
--------------------------------------------------------------------------------

任何合法的 Monad 实例必须遵守以下三大定律：

1. **左单位元（Left Identity）**\ ：

   .. code:: text

      return x >>= f  ==  f x

2. **右单位元（Right Identity）**\ ：

   .. code:: text

      m >>= return  ==  m

3. **结合律（Associativity）**\ ：

   .. code:: text

      (m >>= f) >>= g  ==  m >>= (\x -> f x >>= g)

这保证了我们在代码重构、拆分辅助函数时，计算管道的行为严格确定一致。

Control.Monad 核心高阶控制流工具大全
--------------------------------------------------------------------------------

在真实的工程实践中，我们很少直接写原始的 ``>>=``\ ，而是大量使用 ``Control.Monad`` 模块提供的高阶控制流组合子：

1. mapM 与 mapM\_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

遍历列表，对每个元素执行产生单子效果的动作，并收集所有计算结果：

.. code:: haskell

   mapM  :: Monad m => (a -> m b) -> [a] -> m [b]
   mapM_ :: Monad m => (a -> m b) -> [a] -> m ()   -- 忽略返回值，仅保留效果

.. code:: haskell

   -- 批量打印输出：
   printAll :: [String] -> IO ()
   printAll = mapM_ putStrLn

2. forM 与 forM\_：命令式循环体验
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``forM`` 是 ``mapM`` 的参数翻转版本（\ ``forM = flip mapM``\ ）。当循环体内部逻辑较长时，它能提供如同主流语言中 ``for`` 循环一样的自然阅读体验：

.. code:: haskell

   import Control.Monad (forM_)

   processUsers :: [String] -> IO ()
   processUsers users = do
     forM_ users $ \user -> do
       putStrLn $ "正在初始化用户: " ++ user

3. sequence 与 sequence\_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

将包含多个单子动作的列表，执行并汇总为一个产生结果列表的单子动作：

.. code:: haskell

   sequence :: Monad m => [m a] -> m [a]

.. code:: text

   ghci> sequence [Just 1, Just 2, Just 3]
   Just [1,2,3]
   ghci> sequence [Just 1, Nothing, Just 3]
   Nothing

4. when 与 unless：单子条件分支
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``do`` 块中，如果只想在满足条件时执行某个操作，写 ``if cond then action else return ()`` 极其冗长。使用 ``when`` 与 ``unless`` 可以一气呵成：

.. code:: haskell

   import Control.Monad (when)

   logWarning :: Bool -> String -> IO ()
   logWarning isSevere msg = do
     when isSevere $ do
       putStrLn $ "【警告】: " ++ msg

5. filterM：单子过滤与生成幂集
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在单子环境中过滤列表：

.. code:: haskell

   filterM :: Monad m => (a -> m Bool) -> [a] -> m [a]

利用列表的非确定性单子特性，只需一行代码即可求出集合的\ **全子集（幂集，Powerset）**\ ：

.. code:: haskell

   powerset :: [a] -> [[a]]
   powerset = filterM (\_ -> [True, False])

.. code:: text

   ghci> powerset [1, 2]
   [[1,2],[1],[2],[]]

6. replicateM 与 forever：单子重复执行
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Monad (replicateM)

   -- 执行 N 次动作并收集结果列表（例如生成 3 个随机数或读取 3 行输入）：
   -- replicateM :: Monad m => Int -> m a -> m [a]

7. 鱼骨操作符 (>=>)：Kleisli 组合子
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果两个函数都产生单子上下文（\ ``f :: a -> m b`` 与 ``g :: b -> m c``\ ），普通的点号 ``.`` 无法直接复合它们。来自 ``Control.Monad`` 的 **Kleisli 组合子（>=>）**\ 能直接将它们拼接起来：

.. code:: haskell

   (>=>) :: Monad m => (a -> m b) -> (b -> m c) -> (a -> m c)
   (f >=> g) x = f x >>= g

.. code:: haskell

   -- 管道拼接两个可能失败的安全函数：
   half :: Int -> Maybe Int
   half x = if even x then Just (x `div` 2) else Nothing

   quarter :: Int -> Maybe Int
   quarter = half >=> half

.. code:: text

   ghci> quarter 8
   Just 2
   ghci> quarter 6
   Nothing

小结
--------------------------------------------------------------------------------

- ``Monad`` 通过 ``(>>=)`` 与 ``join`` 解决了上下文多层嵌套的问题。
- ``do`` 记号是纯代数单子绑定的语法糖，使异步与上下文流程清晰如同命令式。
- 现代 GHC 的 ``MonadFail`` 确保了模式匹配失败在类型层面具备安全语义。
- ``Control.Monad`` 组合子家族（``mapM``\ 、\ ``forM``\ 、\ ``when``\ 、\ ``>=>``\ ）构成了 Haskell 高阶控制流的坚实工具箱。
