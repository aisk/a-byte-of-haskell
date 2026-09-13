常用容器与文本类型：Map、Set、Vector、Seq 与 Text
================================================================================

掌握列表之后，进入实际项目时常会发现，单向链表在随机下标访问、键值查找与大量文本处理上性能不够理想。

Haskell 生态在 ``containers``\ 、\ ``vector``\ 、\ ``text`` 与 ``bytestring`` 这几个库中提供了对应的容器与文本类型。本章依次介绍它们。

为什么需要列表之外的容器？
--------------------------------------------------------------------------------

列表适合惰性的管道式处理，但在以下场景中有结构上的短板：

1. **随机下标访问慢**\ ：访问第 n 个元素需要 **O(n)**\ ，不适合数组类算法与密集数值计算。
2. **键值查找慢**\ ：在关联列表 ``[(k, v)]`` 中按键查找同样需要 **O(n)** 的线性扫描。
3. **两端操作不对称**\ ：头部插入 O(1)，但尾部追加或弹出是 O(n)。
4. **String 的内存开销大**\ ：\ ``String`` 就是 ``[Char]``\ ，是装箱的链表。在 64 位机器上，每保存一个字符大约需要 40 字节。

默认规则很简单：先用列表。它和惰性求值、模式匹配、\ ``Data.List`` 的配合最好，大多数中间数据也只是顺序走一遍。只有当代码里出现“按键找”“按下标取”“两头进出”这些动作时，再换成本章对应的容器。换的成本很低，因为这些容器都提供 ``fromList`` 和 ``toList``\ ，和列表之间来回转换是常规操作。

键值映射：Data.Map
--------------------------------------------------------------------------------

``Data.Map`` 位于 GHC 自带的 ``containers`` 库中，基于\ **平衡二叉树（Size-Balanced Trees）**\ 实现。插入、删除和查找的时间复杂度都是 **O(log n)**\ 。因为是有序树，键的类型必须是 ``Ord`` 的实例，这也是为什么 ``Map`` 的函数签名里都带着 ``Ord k =>``\ ，以及 ``toList`` 出来的结果总是按键排好序的。

键是 ``Int`` 时，同一个库里的 ``Data.IntMap`` 更快。键没有自然顺序、或者需要哈希表那样的平均 O(1) 查找时，用 ``unordered-containers`` 包的 ``Data.HashMap.Strict``\ ，它要求键是 ``Hashable`` 的实例。三者的 API 几乎一样，本章只讲 ``Map``\ 。

导入方式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Data.Map`` 中的许多函数（如 ``null``\ 、\ ``size``\ 、\ ``lookup``\ 、\ ``map``\ ）与 Prelude 同名，所以要使用\ **限定导入（qualified）**\ 。

此外，通常建议导入\ **严格版本（Strict）**\ ，避免未求值的 Thunk 堆积在值节点中：

.. code:: haskell

   import qualified Data.Map.Strict as Map
   import Data.Map.Strict (Map)

用词频统计走一遍 Map
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

下面用一个任务串起 ``Map`` 最常用的操作：统计一段文本里每个词出现的次数，然后回答几个问题。

**构建**\ 。把文本切成词，每个词配上计数 1，交给 ``fromListWith``\ 。它和 ``fromList`` 的区别在于遇到重复键时不是覆盖，而是用给定的函数合并两个值：

.. code:: haskell

   import qualified Data.Map.Strict as Map
   import Data.Map.Strict (Map)
   import Data.List (sortOn)
   import Data.Ord (Down (..))

   wordFrequency :: String -> Map String Int
   wordFrequency text = Map.fromListWith (+) [ (w, 1) | w <- words text ]

.. code:: text

   ghci> freq = wordFrequency "haskell is pure haskell is lazy"
   ghci> freq
   fromList [("haskell",2),("is",2),("lazy",1),("pure",1)]

   ghci> Map.fromList [("a", 1), ("a", 2)]
   fromList [("a",2)]

从空 ``Map.empty`` 出发逐个 ``Map.insert`` 也能建出同样的表，但一次性从列表构建更常见。

**查询**\ 。某个词出现了几次？\ ``lookup`` 返回 ``Maybe``\ ，键不存在时得到 ``Nothing`` 而不是抛异常；确定要一个默认值时用 ``findWithDefault``\ ；只关心有没有时用 ``member``\ ：

.. code:: text

   ghci> Map.lookup "haskell" freq
   Just 2
   ghci> Map.lookup "python" freq
   Nothing
   ghci> Map.findWithDefault 0 "python" freq
   0
   ghci> Map.member "is" freq
   True
   ghci> Map.size freq
   4

**更新**\ 。又读到一个词，或者又来了一整段文本。\ ``insertWith`` 在键已存在时用函数合并旧值与新值，\ ``unionWith`` 对两张表做同样的事；不带 ``With`` 的 ``insert`` 和 ``union`` 则是直接覆盖，\ ``union`` 冲突时取左边。\ ``adjust`` 只在键存在时修改值：

.. code:: text

   ghci> Map.insertWith (+) "haskell" 1 freq
   fromList [("haskell",3),("is",2),("lazy",1),("pure",1)]

   ghci> freq2 = wordFrequency "haskell is fun"
   ghci> Map.unionWith (+) freq freq2
   fromList [("fun",1),("haskell",3),("is",3),("lazy",1),("pure",1)]

   ghci> Map.adjust (* 10) "lazy" freq
   fromList [("haskell",2),("is",2),("lazy",10),("pure",1)]

做完这些之后再看 ``freq``\ ，它一点没变：

.. code:: text

   ghci> freq
   fromList [("haskell",2),("is",2),("lazy",1),("pure",1)]

这是和其他语言的 ``HashMap.put`` 最大的不同。\ ``Map`` 是不可变的，每个更新操作都返回一张新表，旧表仍然有效，可以继续用。这并不意味着每次都复制整棵树：新表和旧表共享所有没有改动的子树，只有从根到被修改节点的一条路径是新分配的，所以代价仍是 **O(log n)**\ 。多个版本同时存在也是安全的，比如保留修改前的快照用于对比或回滚。

**删除**\ 。去掉停用词。删一个键用 ``delete``\ ，按条件批量删用 ``filterWithKey``\ ：

.. code:: text

   ghci> Map.delete "is" freq
   fromList [("haskell",2),("lazy",1),("pure",1)]
   ghci> stopWords = ["is", "the", "a"]
   ghci> Map.filterWithKey (\w _ -> w `notElem` stopWords) freq
   fromList [("haskell",2),("lazy",1),("pure",1)]

**导出**\ 。出现最多的前 N 个词。\ ``toList`` 把表变回按键排序的二元组列表，然后就回到了上一章的列表处理：

.. code:: haskell

   topN :: Int -> Map String Int -> [(String, Int)]
   topN n = take n . sortOn (Down . snd) . Map.toList

.. code:: text

   ghci> topN 2 freq
   [("haskell",2),("is",2)]
   ghci> Map.keys freq
   ["haskell","is","lazy","pure"]
   ghci> Map.elems freq
   [2,2,1,1]

两张表之间的集合运算按键进行：\ ``Map.intersection freq freq2`` 保留两段文本共有的词，\ ``Map.difference freq freq2`` 保留只在第一段出现的词，值取自左边的表。

不用导出也能直接统计。\ ``Map k v`` 是 Foldable 的实例（见 Foldable 与 Traversable 一章），所以 ``sum freq`` 得到总词数 6，\ ``length freq`` 得到 4，\ ``elem 2 freq`` 检查有没有出现两次的词，这些函数遍历的是值。它也是 Monoid 的实例（见 Monoid 一章），\ ``freq <> freq2`` 就是 ``Map.union``\ 。

有序集合：Data.Set
--------------------------------------------------------------------------------

``Data.Set`` 保存不重复的元素，同样基于平衡二叉树实现，插入、查找和删除都是 **O(log n)**\ 。

.. code:: haskell

   import qualified Data.Set as Set
   import Data.Set (Set)

常用操作：

.. code:: text

   ghci> s1 = Set.fromList [1, 2, 3, 3, 2, 1]
   ghci> s1
   fromList [1,2,3]

   ghci> Set.member 2 s1
   True
   ghci> Set.insert 4 s1
   fromList [1,2,3,4]

   -- 集合运算：并集、交集与差集
   ghci> s2 = Set.fromList [3, 4, 5]
   ghci> Set.union s1 s2
   fromList [1,2,3,4,5]
   ghci> Set.intersection s1 s2
   fromList [3]
   ghci> Set.difference s1 s2
   fromList [1,2]

``Set`` 和 ``Map`` 一样是 Foldable 和 Monoid 的实例：\ ``length``\ 、\ ``sum``\ 、\ ``elem`` 直接可用，\ ``s1 <> s2`` 就是并集。列表去重最省事的写法就是 ``Set.toList (Set.fromList xs)``\ ，O(n log n)，比上一章的 ``nub`` 快得多，代价是结果按大小排序而不是保留原顺序。

连续内存数组：Data.Vector
--------------------------------------------------------------------------------

当算法需要频繁按下标随机访问（数值计算、矩阵运算、动态规划等）时，``vector`` 库的 ``Data.Vector`` 是常用选择。它在内存中是\ **连续存储**\ 的，下标访问为 **O(1)**\ ，对 CPU 缓存也更友好。

装箱（Boxed）与非装箱（Unboxed）Vector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **装箱 Vector（Data.Vector）**\ ：通用容器，内部存储指向堆对象的指针，可以容纳任何 Haskell 类型（包括未求值的 Thunk）。
- **非装箱 Vector（Data.Vector.Unboxed）**\ ：要求元素实现 ``Unbox`` 类型类（如 ``Int``\ 、\ ``Double``\ 、\ ``Word8`` 等基本类型）。数据直接平铺在连续内存中，没有指针间接寻址和堆对象头，内存布局与 C 数组类似。

.. code:: haskell

   import qualified Data.Vector.Unboxed as VU

   -- 构建非装箱数组：
   denseArray :: VU.Vector Double
   denseArray = VU.fromList [1.0, 2.5, 3.8, 4.2]

   -- O(1) 下标访问：
   firstElem :: Double
   firstElem = denseArray VU.! 0

双端序列：Data.Sequence
--------------------------------------------------------------------------------

``containers`` 库中的 ``Data.Sequence``\ （简称 ``Seq``\ ）基于 2-3 指状树（Finger Trees）实现。它在两端操作上比列表均衡：

- 头部推入与弹出（\ ``<|``\ ）：**O(1)**
- 尾部推入与弹出（\ ``|>``\ ）：**O(1)**
- 两个 Sequence 拼接（\ ``><``\ ）：**O(log(min(n₁, n₂)))**

需要在两端频繁生产与消费（先进先出队列、广度优先搜索等）时，\ ``Seq`` 是合适的数据结构。

Text 与 ByteString
--------------------------------------------------------------------------------

原生 ``String`` 处理大量字符时内存开销大。实际项目里面向人的文本用 ``text`` 包的 ``Text``\ ，网络数据包、二进制文件和编码未知的数据用 ``bytestring`` 包的 ``ByteString``\ ，后者有严格和惰性两个版本，惰性版本可以在常数内存下流式处理大文件。它们的 API、\ ``OverloadedStrings`` 的用法、三者之间怎么选以及编解码转换，字符串一章已经完整介绍，这里不再重复。

数据结构选型
--------------------------------------------------------------------------------

.. list-table:: Haskell 常用容器选型
   :widths: 20 25 55
   :header-rows: 1

   * - 需求
     - 推荐类型
     - 特点与适用场景
   * - 顺序处理、管道流、无限流
     - ``[a]`` （List）
     - 与惰性求值配合好，轻量灵活，支持融合优化
   * - 键值查找与关联存储
     - ``Map k v`` （Data.Map.Strict）
     - 查找与插入 **O(log n)**\ ，不可变的平衡树
   * - 去重与集合运算
     - ``Set a`` （Data.Set）
     - 自动有序，支持交并差运算
   * - 密集数值计算、随机下标访问
     - ``VU.Vector a`` （Vector.Unboxed）
     - 连续内存、无指针，**O(1)** 访问，缓存友好
   * - 双端入队出队、拼接
     - ``Seq a`` （Data.Sequence）
     - 2-3 指状树，两端均为 **O(1)**\ ，适合队列与图遍历

文本类型的选择见字符串一章的表格。

小结
--------------------------------------------------------------------------------

- 默认用列表，出现按键查找、按下标访问、两端进出时再换容器，\ ``fromList`` 和 ``toList`` 负责来回转换。
- ``Map`` 是有序平衡树，键要求 ``Ord``\ ，查找与更新 O(log n)；每个更新操作返回新表，旧表不变，共享子树使得这样做并不昂贵。
- ``fromListWith`` 与 ``insertWith``\ 、\ ``unionWith`` 用合并函数处理重复键，是统计类任务的核心。
- ``Set`` 保存不重复的元素，\ ``Vector`` 提供 O(1) 下标访问，\ ``Seq`` 两端操作都是 O(1)。
- ``Map``\ 、\ ``Set``\ 、\ ``Seq`` 都是 Foldable 和 Monoid 的实例，\ ``length``\ 、\ ``sum``\ 、\ ``elem`` 和 ``<>`` 直接可用。
