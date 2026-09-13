容器抽象：Foldable 与 Traversable
================================================================================

早期 Haskell 中，\ ``map`` 与 ``foldr`` 只对列表有效。现在这些操作被抽象为两个通用的容器类型类：\ **Foldable**\ （泛化折叠）与 **Traversable**\ （带效果的遍历与结构翻转）。

有了这两个类型类，同一套代码可以操作列表、二叉树、集合以及任何自定义容器。

Foldable：泛化的折叠
--------------------------------------------------------------------------------

定义在 ``Data.Foldable`` 中：

.. code:: haskell

   class Foldable t where
     foldr   :: (a -> b -> b) -> b -> t a -> b
     foldMap :: Monoid m => (a -> m) -> t a -> m

只要一个类型构造器 ``t``\ （Kind 为 ``* -> *``\ ）装着元素，并且能按某种顺序遍历，它就可以成为 ``Foldable``\ 。

foldMap 与 Monoid
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``foldMap`` 接收一个把元素映射为某种 Monoid 的函数，然后用该 Monoid 的 ``(<>)`` 与 ``mempty`` 把整个容器合并起来：

.. code:: haskell

   import Data.Foldable
   import Data.Monoid

   -- 为二叉树实现 Foldable
   data Tree a = Leaf | Node (Tree a) a (Tree a) deriving (Show, Eq)

   instance Foldable Tree where
     foldMap _ Leaf = mempty
     foldMap f (Node left val right) =
       foldMap f left <> f val <> foldMap f right

只实现这一个函数，二叉树就可以使用所有标准折叠函数：

.. code:: text

   ghci> myTree = Node (Node Leaf 1 Leaf) 2 (Node Leaf 3 Leaf)
   ghci> sum myTree
   6
   ghci> length myTree
   3
   ghci> toList myTree
   [1,2,3]
   ghci> elem 2 myTree
   True

Data.Foldable 常用函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``toList :: Foldable t => t a -> [a]``\ ：把任意容器转为列表。
- ``find :: Foldable t => (a -> Bool) -> t a -> Maybe a``\ ：查找第一个满足条件的元素。
- ``foldl' :: Foldable t => (b -> a -> b) -> b -> t a -> b``\ ：严格左折叠。

.. tip::

   **如果你熟悉其他语言：把 Foldable 理解为 Reducible**\ ：

   - **直觉**\ ：``Reducible``\ ，即可以规约成单个值的容器。核心操作是 ``foldr``\ （通用折叠）与 ``foldMap``\ （借助 Monoid 合并）。
   - **跨语言映射**\ ：

     - **JavaScript**\ ：数组的 ``[].reduce((acc, x) => ..., initial)``\ 。
     - **Python**\ ：标准库 ``functools.reduce(function, iterable[, initializer])``\ 。
     - **Java**\ ：Stream API 的 ``stream.reduce(...)``\ 。

   - **与命令式循环的差异**\ ：Foldable 不需要显式的游标和局部累加变量。只要元素之间有一个二元合并操作（或者本身是 Monoid），就可以把整个容器归并为一个结果。

Traversable：带效果的遍历与结构翻转
--------------------------------------------------------------------------------

考虑这样一个场景：有一个用户 ID 列表 ``[101, 102, 103]``\ ，需要逐一调用网络接口 ``fetchUser :: Int -> IO User``\ 。

如果只用 ``fmap``\ ：

.. code:: text

   ghci> :t fmap fetchUser [101, 102, 103]
   fmap fetchUser [101, 102, 103] :: [IO User]

得到的是一个\ **由 IO 动作组成的列表（[IO User]）**\ 。但多数时候我们想要的是\ **一个返回用户列表的 IO 动作（IO [User]）**\ ，执行它时依次完成所有查询。

两种结构的差别如下图，\ ``traverse`` 做的就是把左边变成右边：

.. mermaid::

   graph LR
     subgraph before["fmap fetchUser ids :: [IO User]"]
       direction TB
       L["[ ]"] --> IO1["IO User"]
       L --> IO2["IO User"]
       L --> IO3["IO User"]
     end
     subgraph after["traverse fetchUser ids :: IO [User]"]
       direction TB
       IO["IO"] --> L2["[ ]"]
       L2 --> U1["User"]
       L2 --> U2["User"]
       L2 --> U3["User"]
     end
     before ==>|traverse| after

     classDef io fill:#fff3d6,stroke:#c48a1a;
     classDef lst fill:#dbe9ff,stroke:#3b6fb6;
     class IO1,IO2,IO3,IO io;
     class L,L2 lst;

“遍历容器的同时执行 Applicative 效果，并把内外两层结构翻转”，这就是 **Traversable** 做的事。

Traversable 的定义
--------------------------------------------------------------------------------

定义在 ``Data.Traversable`` 中：

.. code:: haskell

   class (Functor t, Foldable t) => Traversable t where
     traverse  :: Applicative f => (a -> f b) -> t a -> f (t b)
     sequenceA :: Applicative f => t (f a) -> f (t a)
     sequenceA = traverse id

- ``sequenceA``\ ：结构翻转。接收一个装着效果的容器（如 ``[Maybe a]``\ ），返回一个装着容器的效果（如 ``Maybe [a]``\ ）。
- ``traverse``\ ：先映射产生效果，再翻转。等价于 ``traverse f = sequenceA . fmap f``\ 。

.. tip::

   **如果你熟悉其他语言：把 Traversable 理解为“带效果的 map”（Effectful Mapping）**\ ：

   - **直觉**\ ：普通 ``map``\ （Functor）只能对内部的值做纯计算。如果映射函数本身会产生效果（网络请求、校验、可能失败），用普通 ``map`` 会得到一个“装满效果的容器”（例如 ``[IO User]`` 或 ``[Maybe a]``\ ）。
   - **核心操作**\ ：``traverse``\ （Elm 中叫 ``traverse`` 或 ``combine``\ ）。它一边对每个元素做带效果的变换，一边把所有效果按顺序串联起来，最后把外层容器与内层效果翻转。
   - **跨语言映射**\ ：

     - **JavaScript / TypeScript**\ ：有一个用户 ID 列表 ``users = [1, 2, 3]``\ ，对每个 ID 调用异步查询 ``fetchUser(id)``\ 。直接 ``users.map(fetchUser)`` 得到的是 Promise 数组 ``[Promise<User>]``\ ；要得到 ``Promise<User[]>``\ ，需要写 ``Promise.all(users.map(fetchUser))``\ 。在 Haskell 中这就是 ``traverse fetchUser users``\ 。
     - **Elm**\ ：提供 ``combine`` 函数，把 ``List (Result e a)`` 翻转为 ``Result e (List a)``\ 。
     - **Rust**\ ：把产生 ``Result<T, E>`` 的迭代器收集为一个 ``Result<Vec<T>, E>``\ （即 ``iter.map(fetch).collect::<Result<Vec<_>, _>>()``\ ）。
     - **Java**\ ：把 ``List<CompletableFuture<User>>`` 汇总为一个 ``CompletableFuture<List<User>>``\ 。

   - **判断标准**\ ：手里拿着“容器里装满效果”（如 ``[IO a]`` 或 ``[Maybe a]``\ ），而需要的是“效果里包着容器”（如 ``IO [a]`` 或 ``Maybe [a]``\ ）时，用 ``sequenceA`` 或 ``traverse``\ 。

为自定义 Tree 实现 Traversable
--------------------------------------------------------------------------------

``Tree`` 要支持 ``traverse``\ ，必须先实现 ``Functor``\ ：

.. code:: haskell

   instance Functor Tree where
     fmap _ Leaf = Leaf
     fmap f (Node l x r) = Node (fmap f l) (f x) (fmap f r)

   instance Traversable Tree where
     traverse _ Leaf = pure Leaf
     traverse f (Node l x r) =
       Node <$> traverse f l <*> f x <*> traverse f r

通过 ``<*>`` 串联，遍历既保留了树的形状，又在每个节点上按顺序执行并组合了 Applicative 效果。

用 GHC 扩展自动派生
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

实际项目中通常直接开启扩展让编译器生成这些实例：

.. code:: haskell

   {-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}

   data Tree a = Leaf | Node (Tree a) a (Tree a)
     deriving (Show, Eq, Functor, Foldable, Traversable)

Traversable 的三条法则（Laws）
--------------------------------------------------------------------------------

合法的 Traversable 实例需要满足以下三条法则：

1. **同一律（Identity）**\ ：用 ``Identity`` 函子遍历不改变原结构：

   .. code:: text

      traverse Identity == Identity

2. **复合律（Composition）**\ ：两个 Applicative 效果的嵌套遍历，等价于用复合函子做一次遍历：

   .. code:: text

      traverse (Compose . fmap g . f) == Compose . fmap (traverse g) . traverse f

3. **自然性（Naturality）**\ ：对于任何保持 Applicative 结构的自然变换 ``t``\ ，有：

   .. code:: text

      t . traverse f == traverse (t . f)

常见用法
--------------------------------------------------------------------------------

1. 批量解析（带短路）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: text

   ghci> import Text.Read (readMaybe)
   -- 全部成功，得到包含列表的 Just：
   ghci> traverse readMaybe ["10", "20", "30"] :: Maybe [Int]
   Just [10,20,30]

   -- 任意一项解析失败，整体返回 Nothing：
   ghci> traverse readMaybe ["10", "abc", "30"] :: Maybe [Int]
   Nothing

2. 批量执行 IO 动作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   fetchAllUsers :: [Int] -> IO [User]
   fetchAllUsers ids = traverse fetchUser ids

3. 只要效果、不要返回值：traverse\_ 与 for\_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果遍历时只关心副作用（写日志、发通知），不需要把结果收集成新列表，可以用 ``traverse_`` 与 ``for_``\ ，省去分配新列表的开销：

.. code:: haskell

   import Data.Foldable (for_)

   notifyUsers :: [User] -> IO ()
   notifyUsers users = do
     for_ users $ \u -> do
       putStrLn $ "正在通知: " ++ userName u

小结
--------------------------------------------------------------------------------

- ``Foldable`` 统一了容器的折叠操作，实现 ``foldMap`` 即可获得 ``sum``\ 、\ ``length``\ 、\ ``toList`` 等全部函数。
- ``Traversable`` 在遍历容器的同时执行 Applicative 效果，并把内外结构翻转。
- ``mapM`` 和 ``sequence`` 现在只是 ``traverse`` 和 ``sequenceA`` 限定在 Monad 上的别名。
- ``DeriveTraversable`` 等扩展可以自动为自定义类型生成这些实例。
