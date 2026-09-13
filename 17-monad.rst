单子（Monad）与 do 记号
================================================================================

Monad 这个概念被赋予过很多比喻，但它的定义并不复杂：\ **它描述了如何把一系列会产生上下文的计算串联起来，并在每一步自动把嵌套的上下文拍平**\ 。

为什么需要 Monad：嵌套问题
--------------------------------------------------------------------------------

假设我们在写一个订单查询流程。每一步都可能因为数据不存在而返回 ``Maybe``\ ：

.. code:: haskell

   getUser   :: String -> Maybe User
   getOrder  :: User -> Maybe Order
   getPay    :: Order -> Maybe Payment

如果只用 ``fmap``\ ：

.. code:: text

   fmap getOrder (getUser "Alice")
   -- 返回类型变成了 Maybe (Maybe Order)

继续链式调用，返回值就会变成 ``Maybe (Maybe (Maybe Payment))``\ 。\ ``Functor`` 和 ``Applicative`` 都没有办法把“产生新上下文的函数”与“已有的上下文”拍平。解决这个问题的就是 **Monad**\ 。

Monad 的定义
--------------------------------------------------------------------------------

``Monad`` 是 ``Functor`` 和 ``Applicative`` 之上的第三层抽象，每一层在前一层的基础上增加一个核心操作：

.. mermaid::

   graph BT
     F["Functor<br/>fmap :: (a -> b) -> f a -> f b"]
     A["Applicative<br/>pure :: a -> f a<br/>(<*>) :: f (a -> b) -> f a -> f b"]
     M["Monad<br/>(>>=) :: m a -> (a -> m b) -> m b"]
     A --> F
     M --> A

箭头方向表示“要成为 Monad 必须先是 Applicative，要成为 Applicative 必须先是 Functor”。

定义在标准 Prelude 中：

.. code:: haskell

   class Applicative m => Monad m where
     -- 将普通值放入最小上下文（等价于 Applicative 的 pure）
     return :: a -> m a
     return = pure

     -- 绑定操作符（Bind）
     (>>=) :: m a -> (a -> m b) -> m b

     -- 忽略前项返回值的顺序执行
     (>>) :: m a -> m b -> m b
     m >> k = m >>= \_ -> k

``>>=``\ （读作 **Bind**\ ）的职责是：\ **从上下文 m a 中取出值 a，传给会产生新上下文的函数 (a -> m b)，并把结果保持在同一层，不产生嵌套。**

join 的视角
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Monad 也可以通过 ``join`` 函数来理解：

.. code:: haskell

   join :: Monad m => m (m a) -> m a

``join`` 把两层相同的上下文拍平成一层（例如把 ``[[a]]`` 拍平为 ``[a]``\ ，把 ``Just (Just x)`` 拍平为 ``Just x``\ ）。\ ``m >>= f`` 等价于 ``join (fmap f m)``\ 。

do 记号与脱糖规则
--------------------------------------------------------------------------------

连续书写 ``>>=`` 与匿名函数会让代码不断向右缩进。Haskell 提供了语法糖 **do 记号**\ ，让有先后依赖的计算可以按命令式的顺序书写：

.. code:: haskell

   getFinalPayment :: String -> Maybe Payment
   getFinalPayment name = do
     user    <- getUser name
     order   <- getOrder user
     payment <- getPay order
     return payment

编译器对 ``do`` 块的脱糖规则是机械的：

1. **带箭头提取**\ ：\ ``x <- m; rest`` 展开为 ``m >>= \x -> rest``
2. **不带提取的顺序执行**\ ：\ ``m1; m2`` 展开为 ``m1 >> m2``
3. **局部纯计算**\ ：\ ``let x = val`` 展开为普通的局部绑定。

所以上面的 ``do`` 代码在底层就是一条普通的 ``>>=`` 链：

.. code:: haskell

   getFinalPayment name =
     getUser name >>= \user ->
       getOrder user >>= \order ->
         getPay order

.. tip::

   **如果你熟悉其他语言：把 Monad 理解为 Chainable / FlatMappable / AndThen-able**\ ：

   - **直觉**\ ：``FlatMappable`` 或 ``Chainable``\ ，即支持自动拍平的、有前后依赖的流水线。核心操作是 ``>>=``\ （即 ``flatMap``\ ）。
   - **各语言中的动词**\ ：

     - **Elm**\ ：直接叫 ``andThen``\ 。前一步计算产生值后，“然后（and then）”把该值交给下一步，生成新的包装上下文。
     - **Rust**\ ：标准库的 ``Option::and_then(...)`` 与 ``Result::and_then(...)``\ 。常用的 ``?`` 错误传播操作符本质上也是在 ``Result`` 单子中做带短路的 ``>>=``\ 。
     - **JavaScript / TypeScript**\ ：``Promise.prototype.then(...)``\ 。在 ``then`` 的回调中返回一个新 Promise，运行时会自动把它展平，不会产生 ``Promise<Promise<T>>``\ 。这正是 Monad 拍平嵌套上下文（\ ``join``\ ）的能力。
     - **Java / Scala**\ ：``Optional.flatMap(...)``\ 、\ ``Stream.flatMap(...)``\ 、\ ``CompletableFuture.thenCompose(...)``\ 。

   - **盒子的三个层次（Functor、Applicative、Monad）**\ ：

     - **Functor**\ （\ ``map``\ ）：单个盒子。函数只作用于盒子内部的值，盒子结构原样保留。
     - **Applicative**\ （\ ``andMap`` / ``map2``\ ）：多个\ **相互独立**\ 的盒子。把各个盒子里的值凑到一起，盒子之间没有先后依赖。
     - **Monad**\ （\ ``andThen`` / ``flatMap``\ ）：\ **有前后依赖**\ 的盒子。后一个盒子的创建依赖前一个盒子解开后的值，因此支持动态决策与短路。

   - **“可编程的分号”**\ ：

     - 在 C、Java 或 Go 中，语句间的分号 ``;`` 只表示顺序执行。
     - 在 Haskell 的 ``do`` 记号中，\ **换行与分号之间可以插入上下文相关的逻辑**\ 。进入下一行之前，单子会先做自己的处理：在 ``Maybe`` 中判断是否为空并短路，在 ``Either`` 中传递错误，在 ``State`` 中传递状态，在 ``IO`` 中安排副作用的顺序。

模式匹配失败：MonadFail
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``do`` 记号中，如果箭头左侧写的是可失败的模式（例如 ``Just x <- action``\ ），匹配失败时编译器会脱糖为调用 ``fail``\ ：

.. code:: haskell

   -- 脱糖逻辑：
   action >>= \x -> case x of
     Just v -> rest
     _      -> fail "Pattern match failure in do expression"

早期 ``fail`` 是 ``Monad`` 类型类的方法，但很多单子并没有合理的失败语义，只能用 ``error`` 实现。从 GHC 8.8 起，\ ``fail`` 被移到独立的 ``Control.Monad.Fail.MonadFail`` 类型类中。这样一来，只有实现了 ``MonadFail`` 的单子（如 ``Maybe``\ 、\ ``IO``\ 、列表）才允许在 ``do`` 块中写可失败的模式，其他单子里这样写会直接编译报错。

Monad 的三条法则
--------------------------------------------------------------------------------

合法的 Monad 实例需要满足以下三条法则：

1. **左单位元（Left Identity）**\ ：

   .. code:: text

      return x >>= f  ==  f x

2. **右单位元（Right Identity）**\ ：

   .. code:: text

      m >>= return  ==  m

3. **结合律（Associativity）**\ ：

   .. code:: text

      (m >>= f) >>= g  ==  m >>= (\x -> f x >>= g)

这些法则保证了在重构、拆分辅助函数时，计算的行为不会改变。

Control.Monad 中的常用函数
--------------------------------------------------------------------------------

实际代码里很少直接写 ``>>=``\ ，更多是使用 ``Control.Monad`` 模块提供的组合子。下面列出的签名都是列表特化后的版本，在现代 ``base`` 中它们实际上定义在 ``Traversable`` 或 ``Foldable`` 上。

1. mapM 与 mapM\_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

遍历列表，对每个元素执行一个单子动作，并收集所有结果：

.. code:: haskell

   mapM  :: Monad m => (a -> m b) -> [a] -> m [b]
   mapM_ :: Monad m => (a -> m b) -> [a] -> m ()   -- 忽略返回值，仅保留效果

.. code:: haskell

   -- 批量打印输出：
   printAll :: [String] -> IO ()
   printAll = mapM_ putStrLn

2. forM 与 forM\_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``forM`` 是 ``mapM`` 参数翻转后的版本（\ ``forM = flip mapM``\ ）。循环体较长时，把列表写在前面读起来更像其他语言的 ``for`` 循环：

.. code:: haskell

   import Control.Monad (forM_)

   processUsers :: [String] -> IO ()
   processUsers users = do
     forM_ users $ \user -> do
       putStrLn $ "正在初始化用户: " ++ user

3. sequence 与 sequence\_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

把一个由单子动作组成的列表，变成一个返回结果列表的单子动作：

.. code:: haskell

   sequence :: Monad m => [m a] -> m [a]

.. code:: text

   ghci> sequence [Just 1, Just 2, Just 3]
   Just [1,2,3]
   ghci> sequence [Just 1, Nothing, Just 3]
   Nothing

4. when 与 unless
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``do`` 块中，如果只想在满足条件时执行某个动作，写 ``if cond then action else return ()`` 比较啰嗦。\ ``when`` 与 ``unless`` 是这个写法的简写：

.. code:: haskell

   import Control.Monad (when)

   logWarning :: Bool -> String -> IO ()
   logWarning isSevere msg = do
     when isSevere $ do
       putStrLn $ "【警告】: " ++ msg

5. filterM
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在单子环境中过滤列表：

.. code:: haskell

   filterM :: Monad m => (a -> m Bool) -> [a] -> m [a]

利用列表单子表示“多种可能”的特性，一行代码就能求出集合的\ **幂集（Powerset）**\ ：

.. code:: haskell

   powerset :: [a] -> [[a]]
   powerset = filterM (\_ -> [True, False])

.. code:: text

   ghci> powerset [1, 2]
   [[1,2],[1],[2],[]]

6. replicateM 与 forever
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Monad (replicateM)

   -- 执行 N 次动作并收集结果列表（例如生成 3 个随机数或读取 3 行输入）：
   -- replicateM :: Monad m => Int -> m a -> m [a]

``forever`` 则是无限重复一个动作，常用于服务器主循环或后台线程。

7. Kleisli 组合子 (>=>)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果两个函数都返回单子上下文（\ ``f :: a -> m b`` 与 ``g :: b -> m c``\ ），普通的 ``.`` 无法直接复合它们。\ ``Control.Monad`` 提供的 ``>=>``\ （有时叫鱼骨操作符）可以把它们接起来：

.. code:: haskell

   (>=>) :: Monad m => (a -> m b) -> (b -> m c) -> (a -> m c)
   (f >=> g) x = f x >>= g

.. code:: haskell

   -- 拼接两个可能失败的函数：
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

- ``Monad`` 通过 ``(>>=)`` 或等价的 ``join`` 解决上下文嵌套的问题。
- ``do`` 记号是 ``>>=`` 链的语法糖，脱糖规则是机械的。
- ``MonadFail`` 把可失败模式的支持从 ``Monad`` 中拆了出来，不支持失败的单子里写这种模式会编译报错。
- ``Control.Monad`` 中的 ``mapM``\ 、\ ``forM``\ 、\ ``when``\ 、\ ``filterM``\ 、\ ``>=>`` 等函数覆盖了大部分日常用法。
