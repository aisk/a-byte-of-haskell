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

键值映射：Data.Map
--------------------------------------------------------------------------------

``Data.Map`` 位于 GHC 自带的 ``containers`` 库中，基于\ **平衡二叉树（Size-Balanced Trees）**\ 实现。插入、删除和查找的时间复杂度都是 **O(log n)**\ 。

导入方式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Data.Map`` 中的许多函数（如 ``null``\ 、\ ``size``\ 、\ ``lookup``\ 、\ ``map``\ ）与 Prelude 同名，所以要使用\ **限定导入（qualified）**\ 。

此外，通常建议导入\ **严格版本（Strict）**\ ，避免未求值的 Thunk 堆积在值节点中：

.. code:: haskell

   import qualified Data.Map.Strict as Map
   import Data.Map.Strict (Map)

创建与构建
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.empty``\ ：空映射。
- ``Map.singleton k v``\ ：只包含一个键值对的映射。
- ``Map.fromList :: Ord k => [(k, a)] -> Map k a``\ ：从二元组列表构建。存在重复键时，后面的值覆盖前面的值。
- ``Map.fromListWith :: Ord k => (a -> a -> a) -> [(k, a)] -> Map k a``\ ：用自定义函数合并重复键的值。

.. code:: text

   ghci> m1 = Map.fromList [("Alice", 95), ("Bob", 80)]
   ghci> m1
   fromList [("Alice",95),("Bob",80)]

   -- 遇到相同键时把数值相加，而不是覆盖：
   ghci> Map.fromListWith (+) [("Apple", 10), ("Banana", 5), ("Apple", 20)]
   fromList [("Apple",30),("Banana",5)]

查询
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.lookup :: Ord k => k -> Map k a -> Maybe a``\ ：找到返回 ``Just val``\ ，不存在返回 ``Nothing``\ ，不会抛异常。
- ``Map.findWithDefault :: Ord k => a -> k -> Map k a -> a``\ ：带默认值的查找。
- ``Map.member :: Ord k => k -> Map k a -> Bool``\ ：检查键是否存在。
- ``Map.size :: Map k a -> Int``\ ：键值对总数。

.. code:: text

   ghci> Map.lookup "Alice" m1
   Just 95
   ghci> Map.lookup "David" m1
   Nothing
   ghci> Map.findWithDefault 0 "David" m1
   0
   ghci> Map.member "Bob" m1
   True

增删与修改
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.insert :: Ord k => k -> a -> Map k a -> Map k a``\ ：插入或覆盖。
- ``Map.insertWith :: Ord k => (a -> a -> a) -> k -> a -> Map k a -> Map k a``\ ：键已存在时，用函数合并旧值与新值。
- ``Map.delete :: Ord k => k -> Map k a -> Map k a``\ ：删除指定键。
- ``Map.adjust :: Ord k => (a -> a) -> k -> Map k a -> Map k a``\ ：仅在键存在时修改对应的值。

集合运算与导出
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.union :: Ord k => Map k a -> Map k a -> Map k a``\ ：合并两个映射（键冲突时取左侧）。
- ``Map.intersection :: Ord k => Map k a -> Map k b -> Map k a``\ ：取两者共有的键。
- ``Map.difference :: Ord k => Map k a -> Map k b -> Map k a``\ ：差集。
- ``Map.keys :: Map k a -> [k]``\ ：所有键。
- ``Map.elems :: Map k a -> [a]``\ ：所有值。
- ``Map.toList :: Map k a -> [(k, a)]``\ ：转换回二元组列表。

示例：统计词频
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import qualified Data.Map.Strict as Map

   wordFrequency :: String -> Map.Map String Int
   wordFrequency text =
     let wordList = words text
     in  Map.fromListWith (+) [ (w, 1) | w <- wordList ]

.. code:: text

   ghci> wordFrequency "haskell is pure haskell is awesome"
   fromList [("awesome",1),("haskell",2),("is",2),("pure",1)]

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

如前所述，原生 ``String``\ （\ ``[Char]``\ ）处理大量字符时内存开销较大。实际项目中通常使用两个库：

- ``Data.Text``\ （\ ``text`` 包）：以 UTF-8 编码的连续内存块存储文本，用于所有面向人的字符串。它的 API 和使用方式在字符串一章已经详细介绍。
- ``Data.ByteString``\ （\ ``bytestring`` 包）：8 位无符号字节（\ ``Word8``\ ）数组，用于网络数据包、二进制文件和编码未知的数据。它有两个版本：

  - **严格版本（Data.ByteString）**\ ：单一连续内存块，适合固定长度的报文头与数据帧。
  - **惰性版本（Data.ByteString.Lazy）**\ ：由一系列 32KB 内存块构成的惰性链表，可以在常数内存下流式读写很大的文件。

两者之间的转换需要明确编码，见字符串一章的 ``encodeUtf8`` 与 ``decodeUtf8``\ 。

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
   * - 用户可见的文本
     - ``Text`` （Data.Text）
     - UTF-8 编码的连续内存块，无装箱开销
   * - 文件 IO、网络协议
     - ``ByteString`` （Data.ByteString）
     - 原始字节块，有严格与惰性两种版本
