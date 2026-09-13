代数数据类型（ADT）与数据建模
================================================================================

Haskell 中的数据建模都建立在\ **代数数据类型（Algebraic Datatypes，简称 ADT）**\ 之上。本章讨论 ADT 的代数含义、递归数据结构、严格字段注解，以及它在抽象语法树（AST）中的应用。

为什么叫“代数”数据类型？
--------------------------------------------------------------------------------

在类型理论中，一个类型的可能取值的数量称为该类型的\ **基数（Cardinality）**\ ：

- 空类型 ``Void``\ （来自 ``Data.Void``\ ）：基数为 0（没有任何值，用于表示逻辑上不可能发生的情况）。
- 单元类型 ``()``\ ：基数为 1（只有唯一值 ``()``\ ）。
- 布尔类型 ``Bool``\ ：基数为 2（\ ``True`` 或 ``False``\ ）。

ADT 之所以叫“代数”，是因为复合类型的状态空间可以由其组成部分的基数通过\ **相加（Sum）**\ 与\ **相乘（Product）**\ 计算得出。

和类型（Sum Types）与逻辑“或”
--------------------------------------------------------------------------------

和类型表示一个值是若干种形态中的\ **某一种**\ （逻辑 OR）。它的基数是各分支基数之\ **和**\ ：

.. code:: haskell

   data NetworkStatus
     = Offline       -- 1
     | Connecting    -- 1
     | Online        -- 1

该类型的可能状态数为 1 + 1 + 1 = 3。和类型避免了命令式语言中用魔数或整数枚举表示状态带来的混淆。

积类型（Product Types）与逻辑“与”
--------------------------------------------------------------------------------

积类型表示一个值同时包含若干组成部分（逻辑 AND）。它的基数是各字段基数的\ **乘积**\ ：

.. code:: haskell

   data Point = Point Double Double

一个 ``Point`` 的可能状态数是两个 ``Double`` 集合大小的乘积。

.. note::

   **函数类型与指数**\ ：
   签名为 ``a -> b`` 的纯函数，可能的实现数量是 ``|b|^|a|``\ （以返回值基数为底，参数基数为指数）。这也是代数数据类型能与初等代数运算对应起来的原因。

.. tip::

   **如果你熟悉其他语言**\ ：

   - **和类型（Sum Types）**\ ：

     - **Rust / Swift**\ ：对应可携带数据的 ``enum``\ （如 Rust 的 ``enum WebEvent { PageLoad, KeyPress(char) }``\ ）。
     - **TypeScript**\ ：对应\ **可辨识联合类型（Discriminated Unions）**\ （例如 ``type Shape = { kind: 'circle'; r: number } | { kind: 'rect'; w: number; h: number }``\ ）。

   - **积类型（Product Types）**\ ：

     - **C / Go / Rust**\ ：对应结构体 ``struct``\ 。
     - **Python**\ ：对应 ``@dataclass`` 或 ``NamedTuple``\ 。
     - **Java**\ ：对应 ``record``\ 。

   - **区别**\ ：传统面向对象语言常需要用继承、向下转型或魔数来模拟和类型，而在 Haskell 中，编译器能在编译期检查所有分支是否被处理，非法状态在类型上就无法表示。

递归数据结构：二叉搜索树（BST）
--------------------------------------------------------------------------------

代数数据类型的定义可以自引用（递归），最典型的应用是树：

.. code:: haskell

   -- 递归代数数据类型：二叉树
   data Tree a
     = Leaf
     | Node (Tree a) a (Tree a)
     deriving (Show, Eq)

   -- 插入元素
   insertTree :: Ord a => a -> Tree a -> Tree a
   insertTree x Leaf = Node Leaf x Leaf
   insertTree x (Node left val right)
     | x < val   = Node (insertTree x left) val right
     | x > val   = Node left val (insertTree x right)
     | otherwise = Node left val right  -- 元素已存在，保持原样

   -- 查找元素
   containsTree :: Ord a => a -> Tree a -> Bool
   containsTree _ Leaf = False
   containsTree x (Node left val right)
     | x == val  = True
     | x < val   = containsTree x left
     | otherwise = containsTree x right

在 GHCi 中验证：

.. code:: text

   ghci> t = insertTree 5 (insertTree 3 (insertTree 7 Leaf))
   ghci> containsTree 3 t
   True
   ghci> containsTree 9 t
   False

插入顺序是 7、3、5，所以 7 成为根节点，3 在它的左子树，5 又在 3 的右子树。上面的 ``t`` 结构如下（``Leaf`` 是空节点）：

.. mermaid::

   graph TD
     N7["Node 7"] --> N3["Node 3"]
     N7 --> L1["Leaf"]
     N3 --> L2["Leaf"]
     N3 --> N5["Node 5"]
     N5 --> L3["Leaf"]
     N5 --> L4["Leaf"]

     classDef leaf fill:#f4f4f4,stroke:#999,stroke-dasharray: 4 2;
     class L1,L2,L3,L4 leaf;

记录语法（Record Syntax）与严格字段
--------------------------------------------------------------------------------

当积类型包含多个字段时，记录语法可以给字段命名：

.. code:: haskell

   data User = User
     { userId       :: !Integer    -- 感叹号 '!' 标记为严格字段
     , userName     :: !String
     , userEmail    :: String      -- 默认为惰性字段
     , userIsAdmin  :: !Bool
     } deriving (Show, Eq)

记录语法会在当前模块自动生成同名的\ **取值函数（Field Selectors）**\ ：

.. code:: text

   ghci> u = User { userId = 101, userName = "Alice", userEmail = "alice@example.com", userIsAdmin = True }
   ghci> userName u
   "Alice"

.. tip::

   **用严格字段（Bang Annotation）避免空间泄漏**\ ：
   默认情况下，记录的字段是惰性的（以 Thunk 形式存储）。如果一个对象的某个字段在循环中被频繁更新，Thunk 链会在堆上不断增长。
   在字段类型前加上感叹号（如 ``!Integer``\ ），GHC 会在构造记录时把该字段求值到弱头范式（WHNF）。对于简单的数值和布尔字段，加严格注解是常见做法。

记录更新语法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   updateEmail :: String -> User -> User
   updateEmail newMail user = user { userEmail = newMail }

部分记录字段的问题
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果和类型的某些分支没有定义同名字段，GHC 仍然允许编译，但在运行时访问缺失的字段会崩溃：

.. code:: haskell

   -- 不推荐：在和类型的不同分支中使用不同的字段
   data BadConfig
     = ServerCfg { port :: Int, host :: String }
     | FileCfg   { filePath :: String }

.. code:: text

   ghci> cfg = FileCfg "app.log"
   ghci> port cfg
   *** Exception: No match in record selector port

通常的做法是：如果不同分支有不同的字段，为每个分支单独声明一个积类型记录，再用一个纯粹的和类型把它们组合起来。

newtype：零开销的类型封装
--------------------------------------------------------------------------------

如果只是想给一个已有类型加一层区分（例如避免把“用户 ID”误传给需要“订单 ID”的函数），应当使用 ``newtype``\ ：

.. code:: haskell

   newtype UserId = UserId Integer deriving (Show, Eq)
   newtype OrderId = OrderId Integer deriving (Show, Eq)

``newtype`` 在编译后会去掉外层包装，运行期与 ``Integer`` 完全相同，因此既有类型上的区分，又没有额外开销。

.. list-table:: data、newtype 与 type 三者对比
   :widths: 20 25 25 30
   :header-rows: 1

   * - 关键字
     - 运行期开销
     - 构造器数量限制
     - 适用场景
   * - ``type``
     - 无（编译期别名替换）
     - 无数据构造器
     - 提高可读性、简写长类型
   * - ``newtype``
     - 无（编译后剥除包装）
     - 恰好一个构造器且只有一个字段
     - 类型隔离、为已有类型提供独立的类型类实例
   * - ``data``
     - 有装箱与构造器开销
     - 任意多构造器、任意多字段
     - 定义新的和类型与积类型

示例：抽象语法树（AST）求值器
--------------------------------------------------------------------------------

代数数据类型的经典应用是表示语法树。下面定义一个算术表达式的求值器：

.. code:: haskell

   -- 用递归代数数据类型表示表达式语法树
   data Expr
     = Lit Int            -- 常数字面量
     | Add Expr Expr      -- 加法节点
     | Mul Expr Expr      -- 乘法节点
     deriving (Show, Eq)

   -- 递归求值
   eval :: Expr -> Int
   eval (Lit n)     = n
   eval (Add e1 e2) = eval e1 + eval e2
   eval (Mul e1 e2) = eval e1 * eval e2

计算嵌套表达式 (2 × 3) + 4：

.. code:: text

   ghci> expr = Add (Mul (Lit 2) (Lit 3)) (Lit 4)
   ghci> eval expr
   10

小结
--------------------------------------------------------------------------------

- ADT 的状态空间由和类型的加法与积类型的乘法决定。
- 递归 ADT 可以自然地表示树、链表与语法树。
- 记录字段可以用感叹号标记为严格，避免 Thunk 堆积。
- 避免在和类型的不同分支中使用部分记录字段；用 ``newtype`` 做零开销的类型区分。
