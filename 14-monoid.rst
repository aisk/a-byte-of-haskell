半群与幺半群（Semigroup 与 Monoid）
================================================================================

在函数式编程中，“代数”（Algebra）指的是：\ **一个类型、定义在该类型上的一组操作，以及这些操作必须满足的法则（Laws）**\ 。

在各种代数结构中，最简单也最常用的是\ **半群（Semigroup）**\ 与\ **幺半群（Monoid）**\ 。它们描述的是“把两个同类值合并成一个”这件事，是批量合并、并行规约这类操作的基础。

半群（Semigroup）
--------------------------------------------------------------------------------

定义在 ``Data.Semigroup`` 中：

.. code:: haskell

   class Semigroup a where
     (<>) :: a -> a -> a

半群只需满足一条法则，即\ **结合律（Associativity）**\ ：

.. code:: text

   (x <> y) <> z == x <> (y <> z)

只要一个类型上存在某种满足结合律的二元合并操作，它就是一个半群。例如列表与文本的拼接：

.. code:: text

   ghci> [1, 2] <> [3, 4] <> [5]
   [1,2,3,4,5]
   ghci> "Hello, " <> "Haskell"
   "Hello, Haskell"

常用半群操作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``sconcat :: Semigroup a => NonEmpty a -> a``\ ：将非空列表中的所有元素用 ``(<>)`` 依次合并。
- ``stimes :: (Integral b, Semigroup a) => b -> a -> a``\ ：将某个元素与自身合并 ``n`` 次。

.. code:: text

   ghci> import Data.Semigroup
   ghci> stimes 3 "abc"
   "abcabcabc"

幺半群（Monoid）
--------------------------------------------------------------------------------

幺半群在半群的基础上增加了一个\ **单位元（Identity）**\ ：

.. code:: haskell

   class Semigroup a => Monoid a where
     mempty  :: a
     mappend :: a -> a -> a
     mappend = (<>)
     mconcat :: [a] -> a
     mconcat = foldr (<>) mempty

幺半群需要满足三条法则：

1. **左单位元（Left Identity）**\ ：\ ``mempty <> x == x``
2. **右单位元（Right Identity）**\ ：\ ``x <> mempty == x``
3. **结合律（Associativity）**\ ：\ ``(x <> y) <> z == x <> (y <> z)``

.. tip::

   **如果你熟悉其他语言：Semigroup 与 Monoid 是 Appendable 与 Concatenable**\ ：

   半群、幺半群这两个名字来自抽象代数，听起来比它们实际表达的东西复杂。在软件里，它们就是最常见的数据合并能力：

   - **Semigroup（可两两合并）**\ ：可以叫 ``Appendable`` 或 ``Mergeable``\ 。只要某种类型支持把两个同类值合并成一个（操作是 ``<>``\ ），且满足结合律，它就是 Semigroup。例如字符串拼接、列表连接、Python 的字典合并（\ ``dict | dict``\ ）。
   - **Monoid（可合并，且有一个空值）**\ ：可以叫 ``Appendable + 空值``\ ，或者 ``Concatenable``\ 。如果类型在支持合并的基础上，还提供一个“什么都不做的空值”（\ ``mempty``\ ），它就是 Monoid。
   - **为什么空值重要**\ ：有了空值，才能把一个包含任意多元素（包括零个元素）的集合折叠成一个结果（\ ``mconcat``\ ）。例如求和时空列表对应 ``0``\ ，拼接文本时空列表对应 ``""``\ 。只有 Semigroup 没有空值的话，空列表合并时就没有值可以返回。
   - 很多语言或开发者在日常交流中并不严格区分两者，直接统称为 ``Appendable``\ 。

同一类型的多种 Monoid 语义
--------------------------------------------------------------------------------

考虑一个问题：\ **整数（Integer）是一个 Monoid 吗？**

对于加法，单位元是 ``0``\ （因为 ``0 + x == x``\ ）；对于乘法，单位元是 ``1``\ （因为 ``1 * x == x``\ ）。为了避免歧义，标准库用 ``newtype`` 为不同运算分别定义了类型：

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

布尔类型也有两种 Monoid：

.. code:: text

   -- 逻辑或（有一个为真即为真），单位元为 False
   ghci> getAny (Any False <> Any True <> mempty)
   True

   -- 逻辑与（全部为真才为真），单位元为 True
   ghci> getAll (All True <> All False <> mempty)
   False

其他常用 Monoid
--------------------------------------------------------------------------------

1. Ordering：多字段链式比较
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Ordering`` 的 Monoid 实例的合并规则是“取第一个不是 ``EQ`` 的结果”，正好可以用来做多字段排序：

.. code:: haskell

   data User = User { userAge :: Int, userName :: String } deriving (Show)

   -- 先按年龄升序，年龄相同再按姓名升序
   compareUsers :: User -> User -> Ordering
   compareUsers u1 u2 =
     compare (userAge u1) (userAge u2) <> compare (userName u1) (userName u2)

2. Endo：函数复合
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Endo a`` 包装了一个类型为 ``a -> a`` 的函数，其 ``(<>)`` 是函数复合 ``(.)``\ ，单位元 ``mempty`` 是恒等函数 ``id``\ ：

.. code:: text

   ghci> import Data.Monoid
   ghci> pipeline = appEndo $ mconcat [Endo (+ 1), Endo (* 2), Endo (+ 10)]
   ghci> pipeline 5
   31   -- 等价于 (+ 1) . (* 2) . (+ 10) $ 5

3. 元组与函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **元组实例**\ ：若 ``a`` 和 ``b`` 都是 Monoid，则二元组 ``(a, b)`` 也是 Monoid，合并时两个分量各自合并。
- **函数实例**\ ：若返回值类型 ``b`` 是 Monoid，则函数 ``a -> b`` 也是 Monoid，合并时先分别调用再合并返回值：

.. code:: text

   ghci> f1 = \x -> [x]
   ghci> f2 = \x -> [x * 10]
   ghci> (f1 <> f2) 5
   [5,50]

编写自己的 Monoid 实例：指标聚合
--------------------------------------------------------------------------------

后端统计中经常需要把多个节点上报的指标汇总起来：

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

用 ``mconcat`` 批量合并：

.. code:: text

   ghci> node1 = Metrics (Sum 100) (Sum 2) (Sum 450.0)
   ghci> node2 = Metrics (Sum 150) (Sum 0) (Sum 600.0)
   ghci> mconcat [node1, node2]
   Metrics {requestCount = Sum {getSum = 250}, errorCount = Sum {getSum = 2}, totalLatency = Sum {getSum = 1050.0}}

工程意义
--------------------------------------------------------------------------------

结合律意味着合并的分组方式不影响结果，单位元意味着空输入也有确定的结果。这两点合起来，就允许把一批数据任意切分、在多个线程或机器上分别合并、再把部分结果按任意顺序汇总，最终结果一致。MapReduce 一类的框架依赖的正是这个性质。

下图展示 ``mconcat [a, b, c, d, e, f]`` 的一种分组方式。因为结合律成立，三个分组可以在不同的线程或节点上各自计算，最后再合并，结果与从左到右依次合并相同：

.. mermaid::

   graph BT
     a["a"] --> ab["a <> b"]
     b["b"] --> ab
     c["c"] --> cd["c <> d"]
     d["d"] --> cd
     e["e"] --> ef["e <> f"]
     f["f"] --> ef
     ab --> abcd["(a <> b) <> (c <> d)"]
     cd --> abcd
     abcd --> all["((a <> b) <> (c <> d)) <> (e <> f)"]
     ef --> all

     classDef part fill:#dbe9ff,stroke:#3b6fb6;
     class ab,cd,ef part;

小结
--------------------------------------------------------------------------------

- ``Semigroup`` 描述满足结合律的二元合并操作 ``(<>)``\ 。
- ``Monoid`` 在此基础上增加单位元 ``mempty``\ ，使 ``mconcat`` 可以处理空列表。
- 同一个类型可以有多种 Monoid，标准库用 ``Sum``\ 、\ ``Product``\ 、\ ``Any``\ 、\ ``All`` 等 newtype 区分。
- ``Ordering``\ 、\ ``Endo``\ 、元组与函数的实例在实际代码中都很常用。
- 结合律与单位元使得数据可以任意切分后并行合并。
