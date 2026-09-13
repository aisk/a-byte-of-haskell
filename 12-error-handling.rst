健壮的错误处理：Maybe 与 Either
================================================================================

图灵奖得主托尼·霍尔（Tony Hoare）曾将空指针（Null / Nil）称为计算机科学中“价值十亿美元的愚蠢错误”。隐式的空值会彻底破坏类型系统的可靠性，让运行时充满难以追踪的 NullPointerException。

Haskell 从语言层面彻底消灭了 Null 引用，将“缺失的数据”与“失败的原因”提升为显式的、强类型的代数数据类型。

Maybe：优雅表达“数据的可能缺失”
--------------------------------------------------------------------------------

定义在标准 Prelude 中：

.. code:: haskell

   data Maybe a = Nothing | Just a deriving (Show, Eq)

- ``Nothing``\ ：明确表示数据不存在。
- ``Just a``\ ：包含一个具体的有效数据 ``a``\ 。

Data.Maybe 核心工具箱
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

标准库 ``Data.Maybe`` 提供了极其丰富的辅助函数：

1. **fromMaybe（安全兜底）**\ ：

   .. code:: haskell

      fromMaybe :: a -> Maybe a -> a

   如果为 ``Just x`` 则解出 ``x``\ ；如果为 ``Nothing`` 则返回指定的默认值：

   .. code:: text

      ghci> import Data.Maybe
      ghci> fromMaybe 0 (Just 42)
      42
      ghci> fromMaybe 0 Nothing
      0

2. **状态判断与列表转换**\ ：

   - ``isJust :: Maybe a -> Bool``
   - ``isNothing :: Maybe a -> Bool``
   - ``listToMaybe :: [a] -> Maybe a``\ ：安全提取列表首项（全函数，不会在空列表时抛出异常！）。
   - ``maybeToList :: Maybe a -> [a]``\ ：将 Maybe 转换为 0 个或 1 个元素的列表。

3. **数据清洗利器：mapMaybe**\ ：

   .. code:: haskell

      mapMaybe :: (a -> Maybe b) -> [a] -> [b]

   遍历列表，同时完成计算过滤与 ``Just`` 值的安全解开（自动丢弃所有 ``Nothing``\ ）：

   .. code:: text

      ghci> import Text.Read (readMaybe)
      ghci> mapMaybe readMaybe ["10", "abc", "20", "xyz", "30"] :: [Int]
      [10,20,30]

.. warning::

   **警惕偏函数 fromJust**\ ：标准库中的 ``fromJust :: Maybe a -> a`` 在遇到 ``Nothing`` 时会直接引发异常崩溃。在生产环境中应严格禁止使用 ``fromJust``\ ，始终使用模式匹配或 ``fromMaybe``\ 。

Either：携带具体错误原因的计算
--------------------------------------------------------------------------------

当操作失败时，\ ``Maybe`` 只能告诉调用方“失败了”，却无法给出“为什么失败”。此时我们需要 ``Either``\ ：

.. code:: haskell

   data Either a b = Left a | Right b deriving (Show, Eq)

- ``Left a``\ ：按照工业界惯例，用于包裹\ **错误详情**\ （Error / Exception）。
- ``Right b``\ ：用于包裹\ **成功的计算结果**\ （“Right”在英文中同时具有“正确”与“右侧”的双重含义）。

Data.Either 核心工具箱
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

除了 ``isLeft`` 与 ``isRight``\ ，最强大的是批量归并函数：

- ``lefts :: [Either a b] -> [a]``\ ：提取列表中所有的错误项。
- ``rights :: [Either a b] -> [b]``\ ：提取列表中所有的成功项。
- ``partitionEithers :: [Either a b] -> ([a], [b])``\ ：一次性将列表拆解为“全部错误组成的列表”与“全部成功结果组成的列表”。

.. code:: text

   ghci> import Data.Either
   ghci> results = [Right 10, Left "网络超时", Right 20, Left "磁盘已满"]
   ghci> partitionEithers results
   (["网络超时","磁盘已满"],[10,20])

实战案例：构建强类型领域业务校验管道
--------------------------------------------------------------------------------

在真实的工程开发中，我们应当避免使用模糊的 ``String`` 表达错误，而是为业务定义强类型的领域错误枚举：

.. code:: haskell

   -- 强类型领域错误定义
   data ValidationError
     = UsernameTooShort Int
     | AgeOutOfRange Int
     | InvalidEmailFormat String
     deriving (Show, Eq)

   type ValidationResult a = Either ValidationError a

   validateUsername :: String -> ValidationResult String
   validateUsername name
     | length name < 3 = Left (UsernameTooShort (length name))
     | otherwise       = Right name

   validateAge :: Int -> ValidationResult Int
   validateAge age
     | age < 0 || age > 150 = Left (AgeOutOfRange age)
     | otherwise            = Right age

在 REPL 中验证校验行为：

.. code:: text

   ghci> validateUsername "Al"
   Left (UsernameTooShort 2)

   ghci> validateUsername "Alice"
   Right "Alice"

   ghci> validateAge (-5)
   Left (AgeOutOfRange (-5))

通过 ``Either`` 与强类型错误，调用方在编译期就被编译器强制要求处理所有可能的失败分支，系统健壮性从根本上得到保证。
