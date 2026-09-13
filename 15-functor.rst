函子（Functor）与范畴映射
================================================================================

实际代码中的数据很少是裸露的，通常被包裹在某种\ **上下文（Context）**\ 中：可能为空的 ``Maybe a``\ 、包含多个元素的列表 ``[a]``\ 、携带错误信息的 ``Either e a``\ 、或者依赖输入环境的函数 ``r -> a``\ 。

**函子（Functor）**\ 定义了如何把一个普通函数“提升”（Lift）到这些上下文中，对内部的值做变换，而不改变上下文本身。

Functor 的定义
--------------------------------------------------------------------------------

定义在标准库中：

.. code:: haskell

   class Functor f where
     fmap :: (a -> b) -> f a -> f b

   (<$>) :: Functor f => (a -> b) -> f a -> f b
   (<$>) = fmap

- ``f`` 的 Kind 必须是 ``* -> *``\ ，即它是一个单参数的\ **类型构造器**\ 。
- ``fmap`` 接收一个函数 ``(a -> b)`` 与一个上下文中的值 ``f a``\ ，返回变换后的 ``f b``\ 。
- 中缀操作符 ``<$>`` 是 ``fmap`` 的同义词，写法上与函数应用符 ``$`` 对应。

.. tip::

   **如果你熟悉其他语言：把 Functor 理解为 Mappable**\ ：

   “函子”这个名字来自范畴论。在日常编程中，可以直接把 Functor 理解为 ``Mappable``\ ，也就是“支持 map 的上下文或容器”。

   - **直觉**\ ：只要一种数据结构或计算上下文包装了值，并且允许你\ **传入一个普通函数作用于内部的值，同时保持外层结构不变**\ ，它就是 Functor。
   - **“单个盒子”心智模型**\ ：Functor 只处理“单个盒子”里的值。普通函数进不了盒子，\ ``fmap``\ （或 ``map``\ ）把函数送进盒子对值做变换，但不改变盒子本身。
   - **跨语言映射**\ ：

     - **Elm**\ ：不提 Functor 这个概念，直接提供动词 ``map``\ 。
     - **JavaScript / TypeScript**\ ：数组的 ``[].map(...)``\ ，以及回调只做纯变换时的 ``Promise.then(...)``\ 。
     - **Java**\ ：\ ``Stream.map(...)``\ 、\ ``Optional.map(...)``\ 、\ ``CompletableFuture.thenApply(...)``\ 。
     - **Rust**\ ：\ ``Option::map(...)``\ 、\ ``Result::map(...)``\ 、\ ``Iterator::map(...)``\ 。
     - **Python**\ ：列表推导式 ``[f(x) for x in xs]`` 或内置的 ``map(f, xs)``\ 。

   - **需要注意的地方**\ ：从范畴论的定义看，Functor 是保持对象与态射结构的映射，并不是所有 Functor 都是数据容器。例如函数类型 ``(->) r`` 也是 Functor，它表示的是对函数输出做后处理。不过在入门阶段，把 Functor 理解为“支持 map 的上下文”已经够用。

标准库中的 Functor 实例
--------------------------------------------------------------------------------

1. 列表
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

列表的 ``fmap`` 就是 ``map``\ ：

.. code:: text

   ghci> (* 2) <$> [1, 2, 3]
   [2,4,6]

2. Maybe
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

只在有值时映射；为 ``Nothing`` 时原样传递：

.. code:: text

   ghci> (+ 10) <$> Just 5
   Just 15
   ghci> (+ 10) <$> Nothing
   Nothing

3. Either e：只作用于 Right 分支
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Either`` 的 Kind 是 ``* -> * -> *``\ ，必须先部分应用左侧的错误类型，得到 ``Either e``\ （Kind 为 ``* -> *``\ ）才能作为 Functor：

.. code:: text

   ghci> (+ 1) <$> (Right 41 :: Either String Int)
   Right 42
   ghci> (+ 1) <$> (Left "超时" :: Either String Int)
   Left "超时"

4. 函数也是 Functor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

函数类型构造器 ``(->) r`` 也是 Functor，对函数做 ``fmap`` 就是函数复合：

.. code:: haskell

   instance Functor ((->) r) where
     fmap = (.)

.. code:: text

   -- 先乘以 2，再加 1
   ghci> pipeline = (+ 1) <$> (* 2)
   ghci> pipeline 10
   21

常用辅助操作符：<$、$> 与 void
--------------------------------------------------------------------------------

见过几个实例之后，再看 ``Data.Functor`` 提供的三个辅助操作符，它们都是 ``fmap`` 的特例：

保持结构替换值：<$ 与 $>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``(<$) :: Functor f => a -> f b -> f a``\ ：将上下文中的所有值替换为左侧的常量，结构保持不变。
- ``($>) :: Functor f => f a -> b -> f b``\ ：同上，只是参数顺序相反。

.. code:: text

   ghci> import Data.Functor
   ghci> 0 <$ [1, 2, 3, 4]
   [0,0,0,0]

   ghci> "OK" <$ Just 123
   Just "OK"

   ghci> "OK" <$ Nothing
   Nothing

它的典型场景是“只关心上下文的形状，不关心里面的值”。比如查一个配置项是否存在，存在就当作开关打开，值本身是什么无所谓：

.. code:: text

   ghci> import qualified Data.Map as M
   ghci> settings = M.fromList [("debug", "1")]
   ghci> True <$ M.lookup "debug" settings
   Just True
   ghci> True <$ M.lookup "trace" settings
   Nothing

解析器库里的开关选项也是这个思路：匹配 ``--verbose`` 这个动作本身没有值，用 ``True <$ 匹配动作`` 就得到一个产出 ``Bool`` 的解析器。

丢弃值只保留效果：void
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   void :: Functor f => f a -> f ()
   void x = () <$ x

``void`` 把上下文内部的值丢掉，替换为 ``()``\ 。在 IO 和并发代码中，如果只关心动作是否执行、不关心返回值，通常会用 ``void``\ ：

.. code:: haskell

   -- 读取一行文本并丢弃内容
   clearOneLine :: IO ()
   clearOneLine = void getLine

为自定义类型实现与自动派生 Functor
--------------------------------------------------------------------------------

手动为一个树结构实现 Functor：

.. code:: haskell

   data Tree a
     = Leaf a
     | Node (Tree a) (Tree a)
     deriving (Show, Eq)

   instance Functor Tree where
     fmap f (Leaf x)   = Leaf (f x)
     fmap f (Node l r) = Node (fmap f l) (fmap f r)

用 GHC 扩展自动派生
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

开启 ``DeriveFunctor`` 扩展后，编译器可以自动生成这段遍历代码：

.. code:: haskell

   {-# LANGUAGE DeriveFunctor #-}

   data Tree a
     = Leaf a
     | Node (Tree a) (Tree a)
     deriving (Show, Eq, Functor)  -- 编译器自动生成 fmap

双函子：Data.Bifunctor
--------------------------------------------------------------------------------

对于有两个类型参数的类型（如二元组 ``(,)`` 与 ``Either``\ ），\ ``Data.Bifunctor`` 提供了同时或分别变换两侧的 ``Bifunctor``\ ：

.. code:: haskell

   import Data.Bifunctor

   -- bimap :: (a -> b) -> (c -> d) -> p a c -> p b d
   -- first :: (a -> b) -> p a c -> p b c
   -- second :: (b -> c) -> p a b -> p a c

.. code:: text

   ghci> bimap (* 2) reverse (10, "hello")
   (20,"olleh")

   ghci> first (++ "错误: ") (Left "404" :: Either String Int)
   Left "404错误: "

Functor 的两条法则（Laws）
--------------------------------------------------------------------------------

合法的 Functor 实例需要满足以下两条法则：

1. 同一律（Identity）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

映射恒等函数 ``id`` 之后，结果与原上下文相同：

.. code:: text

   fmap id == id

2. 复合律（Composition）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

先映射 ``g`` 再映射 ``f``\ ，等价于一次映射复合函数 ``(f . g)``\ ：

.. code:: text

   fmap (f . g) == fmap f . fmap g

这两条法则合起来的含义是：\ ``fmap`` 只变换上下文内部的值，不改变外层结构。例如列表的 ``fmap`` 不能丢弃元素或改变顺序，\ ``Maybe`` 的 ``fmap`` 不能把 ``Just`` 变成 ``Nothing``\ 。有了这个保证，使用者才能放心地对任何 Functor 做等式推理和重构。

小结
--------------------------------------------------------------------------------

- ``Functor`` 定义了在上下文内部映射函数的能力，核心方法是 ``fmap``\ （即 ``<$>``\ ）。
- ``<$``\ 、\ ``$>`` 与 ``void`` 用于替换或丢弃上下文中的值。
- 列表、\ ``Maybe``\ 、\ ``Either e`` 和函数 ``(->) r`` 都是 Functor，函数的 ``fmap`` 就是复合。
- ``DeriveFunctor`` 可以自动生成实例，\ ``Bifunctor`` 处理有两个类型参数的类型。
- 同一律与复合律保证 ``fmap`` 不改变外层结构。
