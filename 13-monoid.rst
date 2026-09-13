半群与幺半群（Semigroup 与 Monoid）
================================================================================

在函数式编程中，“代数”（Algebra）指的是：\ **一个类型集合、定义在该集合上的一组操作符、以及这些操作符必须遵守的数学法则（Laws）**\ 。

在所有代数结构中，最简洁且在工业开发中应用最广泛的是\ **半群（Semigroup）**\ 与\ **幺半群（Monoid）**\ 。它们为数据的批量合并、并行规约与分布式计算提供了坚不可摧的数学基础。

半群（Semigroup）
--------------------------------------------------------------------------------

定义在 ``Data.Semigroup`` 中：

.. code:: haskell

   class Semigroup a where
     (<>) :: a -> a -> a

半群只需满足唯一的一条数学定律——\ **结合律（Associativity）**\ ：

.. code:: text

   (x <> y) <> z == x <> (y <> z)

只要一个类型支持某种满足结合律的二元“拼接/合并”操作，它就是一个半群。例如列表与文本的拼接：

.. code:: text

   ghci> [1, 2] <> [3, 4] <> [5]
   [1,2,3,4,5]
   ghci> "Hello, " <> "Haskell!"
   "Hello, Haskell!"

常用半群操作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``sconcat :: Semigroup a => NonEmpty a -> a``\ ：将非空列表中的所有元素使用 ``(<>)`` 连续合并。
- ``stimes :: (Integral b, Semigroup a) => b -> a -> a``\ ：将某个元素自身连续合并 ``n`` 次。

.. code:: text

   ghci> import Data.Semigroup
   ghci> stimes 3 "abc"
   "abcabcabc"

幺半群（Monoid）
--------------------------------------------------------------------------------

幺半群在半群的基础上增加了一个\ **单位元（Identity / Neutral Element）**\ ：

.. code:: haskell

   class Semigroup a => Monoid a where
     mempty  :: a
     mappend :: a -> a -> a
     mappend = (<>)
     mconcat :: [a] -> a
     mconcat = foldr (<>) mempty

幺半群必须严格遵守三大定律：

1. **左单位元（Left Identity）**\ ：\ ``mempty <> x == x``
2. **右单位元（Right Identity）**\ ：\ ``x <> mempty == x``
3. **结合律（Associativity）**\ ：\ ``(x <> y) <> z == x <> (y <> z)``

.. tip::

   **如果你熟悉其他语言：Semigroup 与 Monoid 是 Appendable 与 Concatenable**\ ：

   初学者常被半群（Semigroup）和幺半群（Monoid）抽象的抽象代数名词劝退，但在软件工程中，它们就是最常见的数据合并能力：

   - **Semigroup（可两两合并）**\ ：直白叫法是 ``Appendable`` / ``Mergeable``\ 。只要某种类型支持将两个同类值无损合并为一个（核心操作是 ``<>``\ ），且满足结合律，它就是 Semigroup。例如字符串拼接、列表连接、Python 的字典合并（\ ``dict | dict``\ ）。
   - **Monoid（可拼接 + 有空值兜底）**\ ：直白叫法是 ``Appendable + 初始空值（Concatenable）``\ 。如果类型在支持合并的基础上，还提供一个“什么都不做的中立空值”（\ ``mempty``\ ），它就成为了 Monoid。
   - **为什么“空值”如此关键？**\ ：正因为有了空值，我们才能安全地把一个包含任意多元素（甚至\ **空列表**\ ）的集合批量折叠为一个单一结果（\ ``mconcat``\ ）。例如求和时空列表为 ``0``\ ，拼接文本时空列表为 ``""``\ 。如果只有 Semigroup 没有空值，空列表在合并时就会陷入无值可出的未定义状态。
   - **工程直觉**\ ：很多现代语言或开发者在实际交流中，并不严格区分两者，而是直接合称为广义的 ``Appendable``\ 。

同一类型的多种 Monoid 语义
--------------------------------------------------------------------------------

思考一个问题：\ **整数（Integer）是一个 Monoid 吗？**

对于加法运算，单位元是 ``0``\ （因为 ``0 + x == x``\ ）；但对于乘法运算，单位元是 ``1``\ （因为 ``1 * x == x``\ ）。为了避免歧义，标准库使用 ``newtype`` 为不同运算赋予了独立的类型标记：

Sum 与 Product
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Data.Monoid

   -- 加法 Monoid：单位元为 0
   ghci> getSum (Sum 3 <> Sum 4 <> mempty)
   7

   -- 乘法 Monoid：单位元为 1
   ghci> getProduct (Product 3 <> Product 4 <> mempty)
   12

Any 与 All
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

布尔类型也有两种 Monoid 视角：

.. code:: text

   -- 逻辑或（只要有一个为真即真），单位元为 False
   ghci> getAny (Any False <> Any True <> mempty)
   True

   -- 逻辑与（全部为真才为真），单位元为 True
   ghci> getAll (All True <> All False <> mempty)
   False

工业高频实用 Monoid
--------------------------------------------------------------------------------

1. Ordering 字典序多字段链式比较
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Ordering`` 类型的 Monoid 实例极其精妙：它的合并规则是“优先取第一个非 ``EQ`` 的结果”：

.. code:: haskell

   data User = User { userAge :: Int, userName :: String } deriving (Show)

   -- 多字段联合排序：先按年龄升序，年龄相同再按姓名升序
   compareUsers :: User -> User -> Ordering
   compareUsers u1 u2 =
     compare (userAge u1) (userAge u2) <> compare (userName u1) (userName u2)

2. 函数自同态：Endo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Endo a`` 包装了一个类型为 ``a -> a`` 的变换函数，其 ``(<>)`` 是函数复合 ``(.)``\ ，单位元 ``mempty`` 是恒等函数 ``id``\ ：

.. code:: text

   ghci> import Data.Monoid
   ghci> pipeline = appEndo $ mconcat [Endo (+ 1), Endo (* 2), Endo (+ 10)]
   ghci> pipeline 5
   31   -- 等价于 (+ 1) . (* 2) . (+ 10) $ 5

3. 元组与函数的高阶派生
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **元组实例**\ ：若 ``a`` 和 ``b`` 均为 Monoid，则二元组 ``(a, b)`` 自动成为 Monoid，合并时各自分别合并。
- **函数实例**\ ：若返回值 ``b`` 是 Monoid，则函数 ``a -> b`` 自动成为 Monoid（按点对点合并）：

.. code:: text

   ghci> f1 = \x -> [x]
   ghci> f2 = \x -> [x * 10]
   ghci> (f1 <> f2) 5
   [5,50]

从零编写业务指标聚合 Monoid 实例
--------------------------------------------------------------------------------

在分布式系统或后端统计中，我们经常需要聚合来自多个节点的性能指标：

.. code:: haskell

   import Data.Monoid

   data Metrics = Metrics
     { requestCount :: !(Sum Int)
     , errorCount   :: !(Sum Int)
     , totalLatency :: !(Sum Double)
     } deriving (Show, Eq)

   instance Semigroup Metrics where
     m1 <> m2 = Metrics
       { requestCount = requestCount m1 <> requestCount m2
       , errorCount   = errorCount m1   <> errorCount m2
       , totalLatency = totalLatency m1 <> totalLatency m2
       }

   instance Monoid Metrics where
     mempty = Metrics mempty mempty mempty

测试批量归并（mconcat）：

.. code:: text

   ghci> node1 = Metrics (Sum 100) (Sum 2) (Sum 450.0)
   ghci> node2 = Metrics (Sum 150) (Sum 0) (Sum 600.0)
   ghci> mconcat [node1, node2]
   Metrics {requestCount = Sum {getSum = 250}, errorCount = Sum {getSum = 2}, totalLatency = Sum {getSum = 1050.0}}

工程意义
--------------------------------------------------------------------------------

Monoid 是现代分布式计算（如 MapReduce、流式日志聚合）的理论基石。只要数据的合并逻辑满足结合律与单位元，我们就可以将巨量数据任意切片、派发到成千上万台计算节点上并发处理，最后按任意拓扑结构归并，结果必定一致。

小结
--------------------------------------------------------------------------------

- ``Semigroup`` 抽象了满足结合律的二元拼接 ``(<>)``\ 。
- ``Monoid`` 补充了单位元 ``mempty``\ ，使集合能够安全执行 ``mconcat`` 批量归并。
- 善用 ``Sum``\ 、\ ``Product``\ 、\ ``Endo`` 与 ``Ordering`` 等内置 Monoid 简化业务代码。
- 只要业务结构满足结合律，即可天然获得无死锁并发切分与零成本合并能力。
