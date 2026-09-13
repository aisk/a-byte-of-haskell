半群与幺半群（Semigroup 与 Monoid）
================================================================================

在函数式编程中，“代数”（Algebra）指的是：\ **一个类型集合、定义在该集合上的一组操作符、以及这些操作符必须遵守的数学法则（Laws）**\ 。

在所有代数结构中，最简洁且实用的是\ **半群（Semigroup）**\ 与\ **幺半群（Monoid）**\ 。

半群（Semigroup）
--------------------------------------------------------------------------------

定义在 ``Data.Semigroup`` 中：

.. code:: haskell

   class Semigroup a where
     (<>) :: a -> a -> a

半群只需满足唯一的一条数学定律——\ **结合律（Associativity）**\ ：

.. code:: text

   (x <> y) <> z == x <> (y <> z)

只要一个类型支持某种满足结合律的二元“拼接/合并”操作，它就是一个半群。例如列表的拼接：

.. code:: text

   ghci> [1, 2] <> [3, 4] <> [5]
   [1,2,3,4,5]

幺半群（Monoid）
--------------------------------------------------------------------------------

幺半群在半群的基础上增加了一个\ **单位元（Identity / Neutral Element）**\ ：

.. code:: haskell

   class Semigroup a => Monoid a where
     mempty  :: a
     mappend :: a -> a -> a
     mappend = (<>)

幺半群必须遵守三大定律：

1. **左单位元**\ ：\ ``mempty <> x == x``
2. **右单位元**\ ：\ ``x <> mempty == x``
3. **结合律**\ ：\ ``(x <> y) <> z == x <> (y <> z)``

同一类型的多种 Monoid 语义
--------------------------------------------------------------------------------

思考一个问题：\ **整数（Integer）是一个 Monoid 吗？**

对于加法运算，单位元是 ``0``\ （因为 ``0 + x == x``\ ）；但对于乘法运算，单位元是 ``1``\ （因为 ``1 * x == x``\ ）。为了避免歧义，标准库使用 ``newtype`` 为不同运算赋予了独立的类型标记：

Sum 与 Product
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Data.Monoid

   -- 加法 Monoid：单位元为 0
   ghci> getSum (Sum 3 <> Sum 4 <> mempty)
   7

   -- 乘法 Monoid：单位元为 1
   ghci> getProduct (Product 3 <> Product 4 <> mempty)
   12

Any 与 All
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

布尔类型也有两种 Monoid 视角：

.. code:: text

   -- 逻辑或（只要有一个为真即真），单位元为 False
   ghci> getAny (Any False <> Any True <> mempty)
   True

   -- 逻辑与（全部为真才为真），单位元为 True
   ghci> getAll (All True <> All False <> mempty)
   False

批量折叠：mconcat
--------------------------------------------------------------------------------

因为 Monoid 具备通用的结合律与单位元，标准库提供了 ``mconcat`` 函数，能够将容器内的所有元素一次性合并为一个值：

.. code:: haskell

   mconcat :: Monoid a => [a] -> a
   mconcat = foldr (<>) mempty

.. code:: text

   ghci> mconcat ["Haskell", " ", "is", " ", "awesome!"]
   "Haskell is awesome!"
   ghci> getSum $ mconcat [Sum 10, Sum 20, Sum 30]
   60

工程意义
--------------------------------------------------------------------------------

Monoid 是现代分布式计算（如 MapReduce、流式日志聚合）的理论基石。只要数据的合并逻辑满足结合律与单位元，我们就可以将巨量数据任意切片、派发到成千上万台机器上并行处理，最后按任意顺序归并，结果必定一致。
