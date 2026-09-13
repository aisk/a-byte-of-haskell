常用核心容器与文本：Map、Set、Vector 与 Text
================================================================================

在掌握了基础列表（List）之后，许多初学者在进入实际工业级开发时常会感到困惑：单向链表在处理大规模随机访问、键值映射与大段文本时性能不尽如人意。

现代 Haskell 生态在 ``containers``\ 、\ ``vector``\ 、\ ``text`` 与 ``bytestring`` 库中提供了丰富的高性能容器与文本替代类型。本章将系统剖析这些不可或缺的核心数据结构。

为什么需要列表之外的高级容器？
--------------------------------------------------------------------------------

虽然列表具有出色的惰性管道流处理能力，但在以下场景中存在结构性短板：

1. **随机下标访问慢**\ ：列表访问第 $n$ 个元素的时间复杂度为 **O(n)**\ ，无法满足数组类算法。
2. **键值检索慢**\ ：在关联列表（Association List ``[(k, v)]``\ ）中按键查找同样需要 **O(n)** 线性扫描。
3. **String 的内存开销巨大**\ ：在底层，\ ``String`` 等价于 ``[Char]``\ 。因为它是装箱链表，在 64 位机器上保存一个单个字符就需要约 40 字节内存！

--------------------------------------------------------------------------------
键值映射表：Data.Map
--------------------------------------------------------------------------------

``Data.Map`` 定义在标准自带的 ``containers`` 库中，基于高效的\ **自平衡二叉树（Size-Balanced Binary Trees）**\ 实现。无论是插入、删除还是查找，其时间复杂度均稳定在 **O(log n)**\ 。

导入规范与最佳实践
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于 ``Data.Map`` 中的大量函数（如 ``null``\ 、\ ``size``\ 、\ ``lookup``\ 、\ ``map``\ ）与标准 Prelude 冲突，必须使用\ **限定前缀导入（qualified）**\ 。

此外，通常强烈推荐导入\ **严格求值版本（Strict）**\ ，防止未求值的 Thunk 堆积在值节点中引发内存泄漏：

.. code:: haskell

   import qualified Data.Map.Strict as Map
   import Data.Map.Strict (Map)

创建与构建
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.empty``\ ：创建一个空的映射表。
- ``Map.singleton k v``\ ：创建只包含单个键值对的映射表。
- ``Map.fromList :: Ord k => [(k, a)] -> Map k a``\ ：从二元组列表构建。如果存在重复键，后面的值会覆盖前面的值。
- ``Map.fromListWith :: Ord k => (a -> a -> a) -> [(k, a)] -> Map k a``\ ：通过自定义函数合并重复键的值。

.. code:: text

   ghci> m1 = Map.fromList [("Alice", 95), ("Bob", 80)]
   ghci> m1
   fromList [("Alice",95),("Bob",80)]

   -- 遇到相同键时，将数值相加而不是粗暴覆盖：
   ghci> Map.fromListWith (+) [("Apple", 10), ("Banana", 5), ("Apple", 20)]
   fromList [("Apple",30),("Banana",5)]

查询与检索
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.lookup :: Ord k => k -> Map k a -> Maybe a``\ ：安全查找，找到返回 ``Just val``\ ，不存在返回 ``Nothing``\ ，绝不抛出异常。
- ``Map.findWithDefault :: Ord k => a -> k -> Map k a -> a``\ ：带有后备默认值的查找。
- ``Map.member :: Ord k => k -> Map k a -> Bool``\ ：检查键是否存在。
- ``Map.size :: Map k a -> Int``\ ：获取键值对总数。

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
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.insert :: Ord k => k -> a -> Map k a -> Map k a``\ ：插入或覆盖键值。
- ``Map.insertWith :: Ord k => (a -> a -> a) -> k -> a -> Map k a -> Map k a``\ ：如果键已存在，使用函数合并旧值与新值。
- ``Map.delete :: Ord k => k -> Map k a -> Map k a``\ ：删除指定键。
- ``Map.adjust :: Ord k => (a -> a) -> k -> Map k a -> Map k a``\ ：仅在键存在时修改对应的值。

.. code:: text

   ghci> Map.insert "Charlie" 88 m1
   fromList [("Alice",95),("Bob",80),("Charlie",88)]

   ghci> Map.adjust (+ 5) "Bob" m1
   fromList [("Alice",95),("Bob",85)]

集合论运算与遍历
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Map.union :: Ord k => Map k a -> Map k a -> Map k a``\ ：合并两个映射（左偏优先）。
- ``Map.intersection :: Ord k => Map k a -> Map k b -> Map k a``\ ：取两者的共有键。
- ``Map.difference :: Ord k => Map k a -> Map k b -> Map k a``\ ：求差集。
- ``Map.keys :: Map k a -> [k]``\ ：导出所有键组成的列表。
- ``Map.elems :: Map k a -> [a]``\ ：导出所有值组成的列表。
- ``Map.toList :: Map k a -> [(k, a)]``\ ：转换回二元组列表。

实战案例：统计文本词频
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import qualified Data.Map.Strict as Map

   wordFrequency :: String -> Map.Map String Int
   wordFrequency text =
     let wordList = words text
     in  Map.fromListWith (+) [ (w, 1) | w <- wordList ]

.. code:: text

   ghci> wordFrequency "haskell is pure haskell is awesome"
   fromList [("awesome",1),("haskell",2),("is",2),("pure",1)]

--------------------------------------------------------------------------------
有序唯一集合：Data.Set
--------------------------------------------------------------------------------

``Data.Set`` 用于保存不重复的元素集合，同样基于平衡二叉搜索树实现，所有操作均为 **O(log n)**\ 。

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

--------------------------------------------------------------------------------
高性能连续数组：Data.Vector
--------------------------------------------------------------------------------

当算法需要频繁进行下标随机索引（例如科学计算、图像矩阵或动态规划）时，来自第三方库 ``vector`` 的 ``Data.Vector`` 是标准方案。

与基于指针链接的单向链表不同，\ ``Vector`` 在内存中以\ **连续内存块**\ 存储，具有极致的 CPU 缓存亲和力，其下标索引时间复杂度为 **O(1)**\ 。

.. code:: haskell

   import qualified Data.Vector as V

   -- 构建与快速下标访问：
   myVec = V.fromList [10, 20, 30, 40, 50]
   thirdElement = myVec V.! 2  -- O(1) 取出 30

--------------------------------------------------------------------------------
现代工业级文本替代：Text 与 ByteString
--------------------------------------------------------------------------------

如前所述，原生 ``String`` （\ ``[Char]``\ ）在处理海量字符时开销过大。Haskell 社区在现代工程中普遍使用两个高效库：

Data.Text（Unicode 自然语言文本）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Data.Text`` 使用紧凑的 UTF-16 编码连续内存块存储文本。所有涉及人类可读语言、国际化字符串处理的模块都应当优先使用它。

.. code:: haskell

   import qualified Data.Text as T
   import Data.Text (Text)

   -- String 与 Text 互转：
   t1 = T.pack "你好，Haskell！"   -- String -> Text
   s1 = T.unpack t1                -- Text -> String

Data.ByteString（字节流与二进制 IO）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Data.ByteString`` 是紧凑的 8 位无符号字节（\ ``Word8``\ ）数组，专为网络协议数据包、图片文件二进制编解码及高性能底层 I/O 设计。

启用 OverloadedStrings 语言扩展
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了避免在代码中频繁手写 ``T.pack "hello"``\ ，可以开启 GHC 语言扩展：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import qualified Data.Text as T
   import Data.Text (Text)

   -- 开启扩展后，双引号字面量自动多态匹配为 Text 类型：
   myTitle :: Text
   myTitle = "直接使用双引号字面量！"

总结与数据结构选型指南
--------------------------------------------------------------------------------

.. list-table:: Haskell 常用容器选型矩阵
   :widths: 20 20 60
   :header-rows: 1

   * - 业务需求
     - 推荐类型
     - 核心优势与适用场景
   * - 顺序处理、管道流、无限流
     - ``[a]`` （List）
     - 极度契合惰性求值，轻量灵活，支持 ``map``\ /\ ``filter``\ /\ ``foldr``
   * - 键值查找与关联存储
     - ``Map k v`` （Data.Map.Strict）
     - 查找与插入平衡 **O(log n)**\ ，纯函数式不可变自平衡树
   * - 去重与集合关系判断
     - ``Set a`` （Data.Set）
     - 自动有序、快速集合交并差运算
   * - 密集数值计算、随机下标查找
     - ``Vector a`` （Data.Vector）
     - 连续内存存储，常数级 **O(1)** 访问，缓存友好
   * - 用户可见的自然语言文本
     - ``Text`` （Data.Text）
     - UTF-16 内存紧凑，杜绝 ``String`` 的 40 字节节点损耗
   * - 文件磁盘 IO、网络协议通信
     - ``ByteString`` （Data.ByteString）
     - 原生裸字节块，支持极速直接内存映射与网络传输
