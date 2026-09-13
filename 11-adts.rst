代数数据类型（ADT）与数据建模
================================================================================

在 Haskell 中，所有复杂的数据建模都建立在\ **代数数据类型（Algebraic Datatypes，简称 ADT）**\ 之上。理解 ADT 的数学代数本质、递归数据结构、严格字段注解以及它在抽象语法树（AST）中的应用，是掌握函数式领域建模的关键。

为什么叫“代数”数据类型？
--------------------------------------------------------------------------------

在类型理论中，类型的可能取值集合的大小被称为该类型的\ **基数（Cardinality）**\ ：

- 空类型 ``Void``\ （来自 ``Data.Void``\ ）：基数为 0（没有任何合法值，用于表示在逻辑上不可能发生的事件）。
- 单元类型 ``()``\ ：基数为 1（只有唯一值 ``()``\ ）。
- 布尔类型 ``Bool``\ ：基数为 2（\ ``True`` 或 ``False``\ ）。

ADT 之所以被称为“代数”，是因为任何复杂数据类型的可能状态空间，都是通过其子类型的基数通过\ **相加（Sum）**\ 与\ **相乘（Product）**\ 计算得出的。

和类型（Sum Types）与逻辑“或”
--------------------------------------------------------------------------------

和类型表示一个值可以是若干种形态中的\ **某一种**\ （逻辑 OR）。它的基数是所有分支构造器基数的\ **加法总和**\ ：

.. code:: haskell

   data NetworkStatus
     = Offline       -- 1
     | Connecting    -- 1
     | Online        -- 1

该类型的可能状态数恰好为 1 + 1 + 1 = 3。和类型彻底消灭了命令式语言中使用随意魔数或枚举整数带来的状态混淆。

积类型（Product Types）与逻辑“与”
--------------------------------------------------------------------------------

积类型表示一个值同时包含若干个组成部分（逻辑 AND）。它的基数是所有内部字段基数的\ **笛卡尔积（乘法）**\ ：

.. code:: haskell

   data Point = Point Double Double

一个 ``Point`` 的合法状态数是两个 ``Double`` 集合大小的相乘。

.. note::

   **函数类型的指数规律**\ ：
   一个签名形如 ``a -> b`` 的纯函数，其可能实现的理论数量恰好是 ``|b|^|a|``\ （以返回值基数为底，入参基数为指数）。这也是为什么代数数据类型能够严密映射初等代数运算法则。

.. tip::

   **如果你熟悉其他语言**\ ：

   - **和类型（Sum Types）**\ ：

     - **Rust / Swift**\ ：等价于可携带载荷数据的强类型 ``enum``\ （如 Rust 的 ``enum WebEvent { PageLoad, KeyPress(char) }``\ ）。
     - **TypeScript**\ ：等价于\ **可辨识联合类型（Discriminated Unions）**\ （例如 ``type Shape = { kind: 'circle'; r: number } | { kind: 'rect'; w: number; h: number }``\ ）。

   - **积类型（Product Types）**\ ：

     - **C / Go / Rust**\ ：等价于最基础的结构体 ``struct``\ 。
     - **Python**\ ：等价于 ``@dataclass`` 或 ``NamedTuple``\ 。
     - **Java**\ ：等价于现代 Java 的 ``record``\ 。

   - **核心优势**\ ：传统命令式语言经常需要通过继承、向下转型或随意魔数来模拟和类型，而在 Haskell 中，编译器能在编译期完整分析所有分支可能，确保没有无效状态产生。

递归数据结构：从零建模二叉搜索树（BST）
--------------------------------------------------------------------------------

代数数据类型的定义可以是自引用的（递归的），最经典的应用是树状结构：

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

   -- 检索元素
   containsTree :: Ord a => a -> Tree a -> Bool
   containsTree _ Leaf = False
   containsTree x (Node left val right)
     | x == val  = True
     | x < val   = containsTree x left
     | otherwise = containsTree x right

在 GHCi 中验证树的构建与检索：

.. code:: text

   ghci> t = insertTree 5 (insertTree 3 (insertTree 7 Leaf))
   ghci> containsTree 3 t
   True
   ghci> containsTree 9 t
   False

记录语法（Record Syntax）与严格字段注解
--------------------------------------------------------------------------------

当积类型包含多个字段时，记录语法赋予字段明确的业务命名：

.. code:: haskell

   data User = User
     { userId       :: !Integer    -- 感叹号 '!' 标记为严格字段
     , userName     :: !String
     , userEmail    :: String      -- 默认惰性字段
     , userIsAdmin  :: !Bool
     } deriving (Show, Eq)

记录语法会自动在当前模块生成同名的\ **取值函数（Field Selectors）**\ ：

.. code:: text

   ghci> u = User { userId = 101, userName = "Alice", userEmail = "alice@example.com", userIsAdmin = True }
   ghci> userName u
   "Alice"

.. tip::

   **使用严格字段（Bang Annotation ``!``\ ）防范内存泄漏**\ ：
   默认情况下，记录的字段是惰性的（存储为 Thunk）。如果一个对象的某个字段在循环中频繁更新，Thunk 闭包链会在堆内存中持续膨胀引发空间泄漏。
   在字段类型前加上感叹号 ``!`` （如 ``!Integer``\ ），指示 GHC 在构造该记录时立即将该字段求值到弱顶层范式（WHNF），这是生产级领域数据建模的最佳工程实践。

记录更新语法
~~~~~~~~~~~~

.. code:: haskell

   updateEmail :: String -> User -> User
   updateEmail newMail user = user { userEmail = newMail }

严重反模式警示：部分记录字段（Partial Record Fields）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果在包含多个分支的和类型中，某些分支没有定义同名字段，GHC 依然会允许编译，但在运行时访问缺失字段会导致致命崩溃：

.. code:: haskell

   -- 危险反模式！严禁在和类型不同分支中混用部分字段：
   data BadConfig
     = ServerCfg { port :: Int, host :: String }
     | FileCfg   { filePath :: String }

.. code:: text

   ghci> cfg = FileCfg "app.log"
   ghci> port cfg
   *** Exception: No match in record selector port

**最佳工程实践**\ ：如果不同分支具有不同字段，应当为每个分支单独声明独立的积类型记录，再通过纯和类型加以组合。

newtype 零成本强类型封装
--------------------------------------------------------------------------------

如果仅需要对单个现有类型做业务维度的强类型隔离（例如防止将“用户ID”传给“订单ID”），应当使用 ``newtype``\ ：

.. code:: haskell

   newtype UserId = UserId Integer deriving (Show, Eq)
   newtype OrderId = OrderId Integer deriving (Show, Eq)

``newtype`` 在编译后会完全剥离外层包装，运行期底层直接等同于裸 ``Integer``\ ，实现\ **类型安全与零抽象性能损耗**\ 的兼得。

实战案例：构建抽象语法树（AST）求值器
--------------------------------------------------------------------------------

代数数据类型最经典的工业应用是构建语言解析器与抽象语法树（AST）。我们声明一个纯函数式算术表达式求解器：

.. code:: haskell

   -- 递归代数数据类型定义表达式语法树
   data Expr
     = Lit Int            -- 常数字面量
     | Add Expr Expr      -- 加法运算节点
     | Mul Expr Expr      -- 乘法运算节点
     deriving (Show, Eq)

   -- 递归解释执行求值函数
   eval :: Expr -> Int
   eval (Lit n)     = n
   eval (Add e1 e2) = eval e1 + eval e2
   eval (Mul e1 e2) = eval e1 * eval e2

运行测试嵌套表达式运算（计算 (2 × 3) + 4）：

.. code:: text

   ghci> expr = Add (Mul (Lit 2) (Lit 3)) (Lit 4)
   ghci> eval expr
   10

小结
--------------------------------------------------------------------------------

- ADT 严格基于集合基数的加法（和类型）与乘法（积类型）映射，建模严密确定。
- 递归 ADT 赋予语言表达树状多叉结构、链表与抽象语法树的自然能力。
- 关键记录字段使用感叹号 ``!`` 严格注解，杜绝堆积 Thunk 造成的空间泄漏。
- 坚决杜绝部分记录字段，使用 ``newtype`` 实现零开销的领域强类型安全。
