代数数据类型（ADT）与数据建模
================================================================================

在 Haskell 中，所有复杂的数据建模都建立在\ **代数数据类型（Algebraic Datatypes，简称 ADT）**\ 之上。理解 ADT 的数学代数本质以及它在泛型、记录语法和抽象语法树（AST）中的应用，是掌握函数式领域建模的关键。

为什么叫“代数”数据类型？
--------------------------------------------------------------------------------

在类型理论中，类型的可能取值集合的大小被称为该类型的\ **基数（Cardinality）**\ ：

- 空类型 ``Void``\ ：基数为 0（没有任何合法值）。
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

该类型的可能状态数恰好为 $1 + 1 + 1 = 3$\ 。和类型彻底消灭了命令式语言中使用随意魔数或枚举整数带来的状态混淆。

积类型（Product Types）与逻辑“与”
--------------------------------------------------------------------------------

积类型表示一个值同时包含若干个组成部分（逻辑 AND）。它的基数是所有内部字段基数的\ **笛卡尔积（乘法）**\ ：

.. code:: haskell

   data Point = Point Double Double

一个 ``Point`` 的合法组合数是两个 ``Double`` 集合大小的相乘。

.. note::

   **函数类型的指数规律**\ ：
   一个签名形如 ``a -> b`` 的纯函数，其可能实现的理论数量恰好是 ``|b|^|a|``\ （以返回值基数为底，入参基数为指数）。这也是为什么代数数据类型能够严密映射初等代数运算法则。

参数化数据类型（泛型）
--------------------------------------------------------------------------------

正如函数可以接收值参数，类型也可以接收\ **类型参数**\ ，从而实现通用泛型容器：

.. code:: haskell

   -- 泛型安全包裹盒
   data Box a = EmptyBox | HasItem a deriving (Show, Eq)

- ``Box`` 是一个\ **类型构造器**\ （Type Constructor），其 Kind 为 ``* -> *``\ ，必须接收一个具体类型（如 ``Box Int``\ ）才能构成真正承载值的数据类型。
- ``EmptyBox`` 与 ``HasItem`` 是\ **数据构造器**\ （Data Constructors），用于在运行期构建具体的值对象。

.. code:: text

   ghci> intBox = HasItem (42 :: Int)
   ghci> strBox = HasItem "Haskell"
   ghci> :t intBox
   intBox :: Box Int

记录语法（Record Syntax）与部分字段陷阱
--------------------------------------------------------------------------------

当积类型包含多个字段时，使用传统的匿名位置构造器极易混淆：

.. code:: haskell

   -- 匿名位置构造器：容易弄错字段顺序
   data User = User Integer String String Bool

   -- 记录语法定义：清晰、安全
   data User = User
     { userId       :: Integer
     , userName     :: String
     , userEmail    :: String
     , userIsAdmin  :: Bool
     } deriving (Show, Eq)

记录语法会自动在当前模块生成同名的\ **取值函数（Field Selectors）**\ ：

.. code:: text

   ghci> u = User { userId = 101, userName = "Alice", userEmail = "alice@example.com", userIsAdmin = True }
   ghci> userName u
   "Alice"
   ghci> userIsAdmin u
   True

记录更新语法
~~~~~~~~~~~~

以非侵入式的方式基于现有对象创建修改部分字段后的新副本：

.. code:: haskell

   updateEmail :: String -> User -> User
   updateEmail newMail user = user { userEmail = newMail }

严重反模式警示：部分记录字段（Partial Record Fields）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果在包含多个分支的和类型中，某些分支没有定义同名字段，GHC 依然会允许编译，但在运行时访问缺失字段会导致崩溃：

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

如果仅需要对单个现有类型做业务维度的类型隔离（例如防止将“用户ID”传给“订单ID”），应当使用 ``newtype``\ ：

.. code:: haskell

   newtype UserId = UserId Integer deriving (Show, Eq)
   newtype OrderId = OrderId Integer deriving (Show, Eq)

``newtype`` 在编译后会完全剥离外层包装，运行期底层等同于裸 ``Integer``\ ，实现\ **类型安全与零抽象性能损耗**\ 的兼得。

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

运行测试复杂的嵌套表达式运算（即计算 $(2 \times 3) + 4$\ ）：

.. code:: text

   ghci> expr = Add (Mul (Lit 2) (Lit 3)) (Lit 4)
   ghci> eval expr
   10
