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

实际代码里很少直接写 ``>>=``\ ，更多是使用 ``Control.Monad`` 模块提供的组合子。下面按“要解决什么问题”分组介绍，签名都写成列表特化后的版本。贯穿的例子是一个小任务：从输入读三行分数，每行都要是 0 到 100 的整数，全部合法且都及格时才写入文件。

先准备一个校验单行的函数：

.. code:: haskell

   import Text.Read (readMaybe)

   parseScore :: String -> Either String Int
   parseScore s = case readMaybe s of
     Just n | n >= 0 && n <= 100 -> Right n
     _ -> Left ("非法分数: " ++ s)

对每个元素做带效果的事：mapM、forM 与 sequence
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

有了 ``parseScore``\ ，校验一整个列表就是 ``mapM``\ ：对每个元素执行一个返回单子的函数，把结果收集成列表。任何一个元素失败，整体就失败：

.. code:: haskell

   mapM  :: Monad m => (a -> m b) -> [a] -> m [b]
   mapM_ :: Monad m => (a -> m b) -> [a] -> m ()   -- 忽略返回值，仅保留效果

.. code:: text

   ghci> mapM parseScore ["90", "75", "60"]
   Right [90,75,60]
   ghci> mapM parseScore ["90", "x", "60"]
   Left "非法分数: x"

``forM`` 是 ``mapM`` 参数翻转后的版本（\ ``forM = flip mapM``\ ）。循环体较长时，把列表写在前面读起来更像其他语言的 ``for`` 循环，所以 IO 代码里更常见的是 ``forM_``\ ：

.. code:: haskell

   forM_ scores $ \s -> putStrLn ("分数: " ++ show s)

``sequence`` 处理的是已经拿在手里的一列单子动作，把 ``[m a]`` 变成 ``m [a]``\ ，等价于 ``mapM id``\ ：

.. code:: text

   ghci> sequence [Just 1, Just 2, Just 3]
   Just [1,2,3]
   ghci> sequence [Just 1, Nothing, Just 3]
   Nothing

这三个函数在现代 ``base`` 里只是 ``traverse``\ 、\ ``for_``\ 、\ ``sequenceA`` 限定在 ``Monad`` 上的别名，下一章会介绍更通用的版本。

条件执行：when 与 unless
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``do`` 块中，如果只想在满足条件时执行某个动作，写 ``if cond then action else return ()`` 比较啰嗦。\ ``when`` 与 ``unless`` 是这个写法的简写：

.. code:: haskell

   when   :: Applicative f => Bool -> f () -> f ()
   unless :: Applicative f => Bool -> f () -> f ()

重复执行：replicateM 与 forever
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``replicateM n act`` 把一个动作执行 ``n`` 次并收集结果，读固定行数的输入、生成若干个随机数都用它。\ ``forever`` 则无限重复一个动作，常用于服务器主循环或后台线程：

.. code:: haskell

   replicateM :: Applicative m => Int -> m a -> m [a]
   forever    :: Applicative f => f a -> f b

把上面几个函数放到一起，就是完整的任务：

.. code:: haskell

   import Control.Monad (replicateM, forM_, when, unless)

   main :: IO ()
   main = do
     ls <- replicateM 3 getLine                      -- 重复执行：读三行
     case mapM parseScore ls of                       -- 对每个元素校验，一个失败全失败
       Left err -> putStrLn err
       Right scores -> do
         forM_ scores $ \s -> putStrLn ("分数: " ++ show s)
         when (all (>= 60) scores) $                  -- 条件执行
           writeFile "pass.txt" (unlines (map show scores))
         unless (all (>= 60) scores) $
           putStrLn "有人不及格，不写文件"

.. code:: text

   $ printf '90\n40\n60\n' | runghc Scores.hs
   分数: 90
   分数: 40
   分数: 60
   有人不及格，不写文件

   $ printf '90\nx\n60\n' | runghc Scores.hs
   非法分数: x

带累积的遍历：foldM
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``foldM`` 是 ``foldl`` 的单子版本，累加函数可以失败或带效果。比如累加总分，超过上限就中止：

.. code:: haskell

   foldM :: Monad m => (b -> a -> m b) -> b -> [a] -> m b

   addBounded :: Int -> Int -> Either String Int
   addBounded acc s
     | acc + s > 250 = Left "总分超出上限"
     | otherwise = Right (acc + s)

.. code:: text

   ghci> foldM addBounded 0 [90, 75, 60]
   Right 225
   ghci> foldM addBounded 0 [90, 95, 99]
   Left "总分超出上限"

同类的还有 ``filterM``\ ，谓词返回 ``m Bool``\ 。它有一个有名的玩法：利用列表单子表示“多种可能”，\ ``filterM (\_ -> [True, False]) [1, 2]`` 得到 ``[[1,2],[1],[2],[]]``\ ，也就是幂集。日常代码里更常见的用法是 ``filterM doesFileExist paths`` 这种带 IO 的过滤。

复合返回单子的函数：>=>
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
- ``Control.Monad`` 的函数按问题分组记：对每个元素做带效果的事用 ``mapM``\ /\ ``forM_``\ ，条件执行用 ``when``\ /\ ``unless``\ ，重复用 ``replicateM``\ /\ ``forever``\ ，带累积用 ``foldM``\ ，复合用 ``>=>``\ 。
