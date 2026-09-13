容器抽象：Foldable 与 Traversable
================================================================================

在早期 Haskell 中，列表拥有专属的 ``map`` 与 ``foldr``\ 。然而在现代 Haskell 中，这些操作被高度抽象为两个更为普适的通用容器类型类：\ **Foldable**\ （泛化折叠）与 **Traversable**\ （效果遍历与结构翻转）。

掌握这两个类型类，能让你用同一套精简的代码操作列表、树、集合乃至自定义的任何容器。

Foldable：泛化容器折叠
--------------------------------------------------------------------------------

定义在标准库 ``Data.Foldable`` 中：

.. code:: haskell

   class Foldable t where
     foldr   :: (a -> b -> b) -> b -> t a -> b
     foldMap :: Monoid m => (a -> m) -> t a -> m

只要一个类型构造器 ``t``\ （Kind 为 ``* -> *``\ ）承载着元素，并能按某种次序遍历，它就可以成为 ``Foldable``\ 。

foldMap：Monoid 聚合的神奇杠杆
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``foldMap`` 接收一个将每个元素映射为某种 Monoid 的函数，然后利用该 Monoid 的 ``(<>)`` 与 ``mempty`` 自动完成整个容器的批量归并：

.. code:: haskell

   import Data.Foldable
   import Data.Monoid

   -- 自定义二叉树实现 Foldable
   data Tree a = Leaf | Node (Tree a) a (Tree a) deriving (Show)

   instance Foldable Tree where
     foldMap _ Leaf = mempty
     foldMap f (Node left val right) =
       foldMap f left <> f val <> foldMap f right

只需实现这一个函数，我们的自定义二叉树就\ **免费获得了全部标准折叠函数**\ ：

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

Data.Foldable 常用工具集
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``toList :: Foldable t => t a -> [a]``\ ：将任意容器转为扁平列表。
- ``find :: Foldable t => (a -> Bool) -> t a -> Maybe a``\ ：查找首个满足条件的元素。
- ``foldl' :: Foldable t => (b -> a -> b) -> b -> t a -> b``\ ：严格左折叠。

Traversable：效果交织与结构翻转
--------------------------------------------------------------------------------

在日常业务开发中，我们常常遇到以下痛点：
我们有一个待处理的用户 ID 列表：\ ``[101, 102, 103]``\ 。我们希望逐一调用网络查询接口 ``fetchUser :: Int -> IO User``\ 。

如果仅仅使用 ``fmap``\ ：

.. code:: text

   ghci> :t fmap fetchUser [101, 102, 103]
   fmap fetchUser [101, 102, 103] :: [IO User]

我们得到了一个\ **由 IO 动作组成的列表（[IO User]）**\ 。但在绝大多数场景下，我们想要的是一个\ **包含整个用户列表的单一 IO 动作（IO [User]）**\ ——在执行该动作时，自动依次完成所有网络查询。

这种“在遍历容器的同时交织 Applicative 效果，并将内外层结构彻底翻转”的绝活，正是 **Traversable**\ 。

Traversable 的形式化定义
--------------------------------------------------------------------------------

定义在 ``Data.Traversable`` 中：

.. code:: haskell

   class (Functor t, Foldable t) => Traversable t where
     traverse  :: Applicative f => (a -> f b) -> t a -> f (t b)
     sequenceA :: Applicative f => t (f a) -> f (t a)

- ``sequenceA``\ ：\ **内外层结构翻转原语**\ 。它接收一个包裹在容器内的效果结构（如 ``[Maybe a]``\ ），直接翻转为包裹在效果之内的容器（如 ``Maybe [a]``\ ）。
- ``traverse``\ ：先映射产生效果，再翻转结构。数学上等价于 ``traverse f = sequenceA . fmap f``\ 。

经典实战场景
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **批量安全解析（短路保障）**\ ：

   .. code:: text

      ghci> import Text.Read (readMaybe)
      -- 全部成功，翻转为包含列表的 Just：
      ghci> traverse readMaybe ["10", "20", "30"] :: Maybe [Int]
      Just [10,20,30]

      -- 只要任意一项解析失败，整体立刻优雅失败：
      ghci> traverse readMaybe ["10", "abc", "30"] :: Maybe [Int]
      Nothing

2. **批量并发或顺序 IO 动作执行**\ ：

   .. code:: haskell

      fetchAllUsers :: [Int] -> IO [User]
      fetchAllUsers ids = traverse fetchUser ids

只重效果、忽略返回值的遍历：traverse\_ 与 for\_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果在遍历时我们只关心其副作用（如向磁盘写日志、发送通知），并不需要把所有子项的结果再封装回一个新列表，使用 ``traverse_`` 与 ``for_`` 可以完全省去内存中分配新列表的开销：

.. code:: haskell

   import Data.Foldable (for_)

   notifyUsers :: [User] -> IO ()
   notifyUsers users = do
     for_ users $ \u -> do
       putStrLn $ "正在通知: " ++ userName u
