应用函子（Applicative）与效果组合
================================================================================

上一章的 ``Functor`` 可以把单参数函数映射到上下文中的值上。现在考虑下面的情况：

有一个两个参数的函数 ``add :: Int -> Int -> Int``\ ，以及两个包在 ``Maybe`` 里的值 ``Just 2`` 与 ``Just 3``\ 。如何把它们相加？

先试试 ``fmap``\ ：

.. code:: text

   ghci> :t (+) <$> Just 2
   (+) <$> Just 2 :: Num a => Maybe (a -> a)

结果是一个\ **包在 Maybe 里的函数**\ 。\ ``Functor`` 没有办法把上下文中的函数应用到另一个上下文中的值上。解决这个问题的是\ **应用函子（Applicative Functor）**\ 。

Applicative 的定义
--------------------------------------------------------------------------------

定义在 ``Control.Applicative`` 中：

.. code:: haskell

   class Functor f => Applicative f where
     pure  :: a -> f a
     (<*>) :: f (a -> b) -> f a -> f b

- ``pure``\ ：把一个普通值放进该上下文的最小环境中。
- ``<*>``\ （有时被叫作 tie-fighter）：接收一个“上下文中的函数”，把它应用到“上下文中的值”上，得到新的上下文结果。

习惯写法
--------------------------------------------------------------------------------

把 ``<$>``\ （即 ``fmap``\ ）与 ``<*>`` 连起来用，就能把任意多参数的普通函数提升到上下文里：

.. code:: text

   ghci> (+) <$> Just 2 <*> Just 3
   Just 5

   ghci> (+) <$> Just 2 <*> Nothing
   Nothing

无论目标函数有三个、四个还是更多参数，依次用 ``<*>`` 串联即可：

.. code:: haskell

   data Profile = Profile String Int String deriving (Show)

   buildProfile :: Maybe String -> Maybe Int -> Maybe String -> Maybe Profile
   buildProfile mName mAge mEmail =
     Profile <$> mName <*> mAge <*> mEmail

.. tip::

   **如果你熟悉其他语言：把 Applicative 理解为 Applyable / Zippable / Combinable**\ ：

   - **为什么 Applicative 不好起名**\ ：Functor 对应 ``map``\ ，Monad 对应 ``flatMap``\ ，而 Applicative 的核心操作是“把一个装在盒子里的函数应用到装在盒子里的值上”（\ ``f (a -> b) -> f a -> f b``\ ）。日常语言里没有现成的动词描述这个动作，所以社区会用 ``Applyable``\ 、\ ``Zippable`` 或 ``Combinable`` 来称呼它。
   - **Elm 的动词化方案：andMap 与 map2**\ ：Elm 不使用类型类名称，而是把 Applicative 的用法提炼成动词：

     - **核心心智**\ ：\ **把两个或多个“装在独立盒子里”的值凑到一起**\ 。
     - Functor 只能处理单个盒子的映射（\ ``map``\ ）；如果有多个包在上下文里的参数（例如用户的名字、年龄、邮箱分别被 ``Maybe`` 包装），就需要把它们一起装配到一个多参数构造函数中。
     - Elm 用 ``andMap`` 把多个盒子串起来：\ ``Ok Profile |> andMap maybeName |> andMap maybeAge``\ ，这与 Haskell 的 ``Profile <$> maybeName <*> maybeAge`` 结构相同。

   - **跨语言映射**\ ：

     - **JavaScript**\ ：类似 ``Promise.all([p1, p2])``\ 。各个异步任务彼此独立，没有先后依赖，可以并发发起，最后把各自的返回值聚合起来。
     - **Java**\ ：类似 ``CompletableFuture.allOf()`` 或二元合并操作 ``thenCombine()``\ 。

   - **与 Monad 的区别**\ ：

     - **Applicative（多个独立盒子）**\ ：类似表单批量校验或并行依赖图，各个计算彼此独立，结构在运行前就确定。
     - **Monad（前后依赖的盒子）**\ ：类似 ``async/await``\ ，第 2 步的计算依赖第 1 步的返回值，整个流程无法在执行前确定。

组合子：liftA2、liftA3、*> 与 <*
--------------------------------------------------------------------------------

liftA2 与 liftA3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

把普通的二元或三元函数直接提升为在 Applicative 上下文中运算的函数：

.. code:: haskell

   liftA2 :: Applicative f => (a -> b -> c) -> f a -> f b -> f c
   liftA2 f x y = f <$> x <*> y

忽略单侧结果：*> 与 <*
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在有副作用的操作中，经常需要依次执行两个动作，但只关心其中一个的返回值：

- ``(*>) :: Applicative f => f a -> f b -> f b``\ ：执行双方的效果，只保留右侧的值。
- ``(<*) :: Applicative f => f a -> f b -> f a``\ ：执行双方的效果，只保留左侧的值。

.. code:: text

   ghci> Just "Hello" *> Just "World"
   Just "World"
   ghci> Nothing *> Just "World"
   Nothing

列表的两种 Applicative：笛卡尔积与 ZipList
--------------------------------------------------------------------------------

默认列表实例：笛卡尔积
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

普通列表的 ``Applicative`` 实例表示所有可能组合的\ **笛卡尔积**\ ：

.. code:: text

   ghci> [(+ 1), (* 2)] <*> [10, 20]
   [11,21,20,40]

   ghci> (,) <$> ["A", "B"] <*> [1, 2]
   [("A",1),("A",2),("B",1),("B",2)]

ZipList：按位置配对
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果希望像 ``zipWith`` 一样把第 i 个函数作用于第 i 个元素，可以使用 ``Control.Applicative`` 提供的 ``ZipList``\ ：

.. code:: text

   ghci> import Control.Applicative
   ghci> getZipList $ ZipList [(+ 1), (* 2)] <*> ZipList [10, 20]
   [11,40]

通过 ``newtype`` 包装，同一个列表类型可以拥有两种不同的 Applicative 行为。

Applicative 的四条法则（Laws）
--------------------------------------------------------------------------------

合法的 Applicative 实例需要满足以下四条法则：

1. **同一律（Identity）**\ ：

   .. code:: text

      pure id <*> v == v

2. **同态律（Homomorphism）**\ ：先在纯值上应用函数再放入上下文，等价于分别放入上下文后再应用：

   .. code:: text

      pure f <*> pure x == pure (f x)

3. **交换律（Interchange）**\ ：把上下文中的函数应用于纯值，等价于把“应用该值”这个操作放入上下文后作用于函数：

   .. code:: text

      u <*> pure y == pure ($ y) <*> u

4. **复合律（Composition）**\ ：函数复合在 Applicative 上下文中保持不变：

   .. code:: text

      pure (.) <*> u <*> v <*> w == u <*> (v <*> w)

这些法则保证效果的组合方式有确定的语义，使用者可以据此做重构和推理。

Applicative 与错误累积：Validation
--------------------------------------------------------------------------------

一个常见的问题是：既然下一章的 ``Monad`` 更通用，为什么还需要 ``Applicative``\ ？

区别在于：\ **Applicative 的各个子计算彼此独立，效果在运行前就已知；而 Monad 的后续计算依赖前一步的返回值**\ 。

Monad 的短路行为
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在表单验证中，如果用 ``Monad`` 或普通的 ``Either`` 做链式验证，用户名校验一失败，计算就会\ **短路退出**\ ，用户每次提交只能看到一个错误。

Applicative 的错误累积
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

利用 Applicative 各子计算相互独立的特点，可以一次性验证所有字段，并用 ``Semigroup``\ （例如列表拼接）把所有错误收集起来：

.. code:: haskell

   data Validation err a
     = Failure err
     | Success a
     deriving (Show, Eq)

   instance Functor (Validation err) where
     fmap _ (Failure err) = Failure err
     fmap f (Success a)   = Success (f a)

   instance Semigroup err => Applicative (Validation err) where
     pure = Success
     -- 两边都失败时，用 (<>) 合并两个错误
     Failure e1 <*> Failure e2 = Failure (e1 <> e2)
     Failure e1 <*> _          = Failure e1
     _          <*> Failure e2 = Failure e2
     Success f  <*> Success a  = Success (f a)

验证一下：

.. code:: haskell

   data User = User String Int deriving (Show)

   checkName :: String -> Validation [String] String
   checkName name = if null name then Failure ["用户名不能为空"] else Success name

   checkAge :: Int -> Validation [String] Int
   checkAge age = if age < 18 then Failure ["年龄必须满 18 岁"] else Success age

.. code:: text

   -- 同时输入非法名字与非法年龄：
   ghci> User <$> checkName "" <*> checkAge 15
   Failure ["用户名不能为空","年龄必须满 18 岁"]

两个字段的错误都被收集到了结果里。这依赖于 Applicative 各效果之间的独立性，用 Monad 是做不到的。

小结
--------------------------------------------------------------------------------

- ``Applicative`` 解决了 ``Functor`` 无法应用上下文中的函数的问题。
- 习惯写法 ``f <$> x <*> y`` 把多参数函数提升到上下文中。
- 列表有笛卡尔积和 ``ZipList`` 两种 Applicative 实例。
- 四条法则（同一、同态、交换、复合）规定了效果组合的语义。
- Applicative 的各子计算彼此独立，适合批量验证与并发这类场景。
