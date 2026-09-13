应用函子（Applicative）与效果组合
================================================================================

在上一章中，我们掌握了能够对上下文中的值执行单参数函数映射的 ``Functor``\ 。但考虑以下现实业务场景：

我们有一个需要两个入参的普通函数 ``add :: Int -> Int -> Int``\ ，以及两个包裹在 ``Maybe`` 上下文中的数据：\ ``Just 2`` 与 ``Just 3``\ 。如何将它们安全地相加？

尝试使用 ``fmap``\ ：

.. code:: text

   ghci> :t (+) <$> Just 2
   (+) <$> Just 2 :: Maybe (Integer -> Integer)

结果是一个\ **包裹在 Maybe 上下文中的函数**\ ！普通 ``Functor`` 根本无法将上下文中的函数应用到另一个上下文中的值。为了破除这一局限，\ **应用函子（Applicative Functor）**\ 应运而生。

Applicative 的形式化定义
--------------------------------------------------------------------------------

定义在 ``Control.Applicative`` 中：

.. code:: haskell

   class Functor f => Applicative f where
     pure  :: a -> f a
     (<*>) :: f (a -> b) -> f a -> f b

- ``pure``\ ：将一个普通裸值放入该上下文的最小默认环境之中。
- ``<*>``\ （常被称为 tie-fighter 操作符）：接收一个“包裹在上下文中的函数”，并将其应用到“包裹在上下文中的值”，输出新的上下文结果。

Idiom 语法风格
--------------------------------------------------------------------------------

结合 ``<$>``\ （即 ``fmap``\ ）与 ``<*>``\ ，我们可以以一种极其优雅、直观的风格将任意多参数普通函数无缝提升至上下文：

.. code:: text

   ghci> (+) <$> Just 2 <*> Just 3
   Just 5

   ghci> (+) <$> Just 2 <*> Nothing
   Nothing

无论目标构造函数包含三个、四个乃至更多参数，只需依次用 ``<*>`` 串联：

.. code:: haskell

   data Profile = Profile String Int String deriving (Show)

   buildProfile :: Maybe String -> Maybe Int -> Maybe String -> Maybe Profile
   buildProfile mName mAge mEmail =
     Profile <$> mName <*> mAge <*> mEmail

组合子家族：liftA2、*> 与 <*
--------------------------------------------------------------------------------

liftA2 与 liftA3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

将普通的二元或三元函数直接提升为在 Applicative 上下文中运算的函数：

.. code:: haskell

   liftA2 :: Applicative f => (a -> b -> c) -> f a -> f b -> f c
   liftA2 f x y = f <$> x <*> y

忽略单侧结果：*> 与 <*
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在进行有副作用的操作时，我们常常需要按顺序依次触发两个动作，但只关心其中一侧的返回结果：

- ``(*>) :: Applicative f => f a -> f b -> f b``\ ：依次执行双方的效果，但\ **只保留右侧的值**\ 。
- ``(<*) :: Applicative f => f a -> f b -> f a``\ ：依次执行双方的效果，但\ **只保留左侧的值**\ 。

.. code:: text

   ghci> Just "Hello" *> Just "World"
   Just "World"
   ghci> Nothing *> Just "World"
   Nothing

列表的 Applicative 语义：笛卡尔全组合
--------------------------------------------------------------------------------

对于列表，\ ``Applicative`` 的行为代表所有可能状态的\ **笛卡尔积组合**\ ：

.. code:: text

   ghci> [(+ 1), (* 2)] <*> [10, 20]
   [11,21,20,40]

   ghci> (,) <$> ["A", "B"] <*> [1, 2]
   [("A",1),("A",2),("B",1),("B",2)]

深度对比：Applicative 表单校验与错误累积（Validation 模式）
--------------------------------------------------------------------------------

很多人常问：既然下一章的 ``Monad`` 更加强大，为什么我们还需要 ``Applicative``\ ？

核心分水岭在于：\ **Applicative 的各个子计算在结构上是相互独立的，而 Monad 的后续计算强依赖于前一步的返回值**\ 。

Monad 的短路局限
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在表单验证中，如果我们用 ``Monad`` 或常规 ``Either`` 进行链式验证，一旦用户名校验失败，计算就会\ **立刻短路退出**\ ，用户每次提交只能看到一个错误。

Applicative 的独立错误累积
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

利用 Applicative 的独立性，我们可以一次性并行验证表单的所有字段，并将产生的所有错误通过 ``Semigroup`` （如列表拼接）全部累积收集起来！

.. code:: haskell

   -- 自定义累积校验类型（与 Either 类似，但在 Applicative 行为上不短路）
   data Validation err a
     = Failure err
     | Success a
     deriving (Show, Eq)

   instance Functor (Validation err) where
     fmap _ (Failure err) = Failure err
     fmap f (Success a)   = Success (f a)

   instance Semigroup err => Applicative (Validation err) where
     pure = Success
     -- 核心：当两边都失败时，利用 (<>) 累积合并两个错误！
     Failure e1 <*> Failure e2 = Failure (e1 <> e2)
     Failure e1 <*> _          = Failure e1
     _          <*> Failure e2 = Failure e2
     Success f  <*> Success a  = Success (f a)

实战验证：

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

在上面的代码中，两个字段的错误被\ **同时捕获并聚合**\ ！这种能力的理论支撑正是 Applicative 效果的独立性。
