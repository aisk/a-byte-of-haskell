健壮的错误处理：Maybe、Either 与异常模型
================================================================================

图灵奖得主托尼·霍尔（Tony Hoare）曾将空指针（Null / Nil）称为计算机科学中“价值十亿美元的愚蠢错误”。隐式的空值会彻底破坏类型系统的可靠性，让运行时充满难以追踪的崩溃。

Haskell 从语言层面彻底消灭了 Null 引用，将“缺失的数据”与“失败的原因”提升为显式的、强类型的代数数据类型。

Maybe：优雅表达“数据的可能缺失”
--------------------------------------------------------------------------------

定义在标准 Prelude 中：

.. code:: haskell

   data Maybe a = Nothing | Just a deriving (Show, Eq)

- ``Nothing``\ ：明确表示数据不存在。
- ``Just a``\ ：包含一个具体的有效数据 ``a``\ 。

Data.Maybe 核心工具箱
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

标准库 ``Data.Maybe`` 提供了极其丰富的辅助函数：

1. **fromMaybe（安全默认兜底）**\ ：

   .. code:: haskell

      fromMaybe :: a -> Maybe a -> a

   如果为 ``Just x`` 则解出 ``x``\ ；如果为 ``Nothing`` 则返回指定的默认值：

   .. code:: text

      ghci> import Data.Maybe
      ghci> fromMaybe 0 (Just 42)
      42
      ghci> fromMaybe 0 Nothing
      0

2. **核心解构函数：maybe（折叠原语）**\ ：

   .. code:: haskell

      maybe :: b -> (a -> b) -> Maybe a -> b

   提供默认值与变换函数，一步完成模式匹配与安全映射：

   .. code:: text

      ghci> maybe "未知年龄" (\n -> show n ++ " 岁") (Just 25)
      "25 岁"
      ghci> maybe "未知年龄" (\n -> show n ++ " 岁") Nothing
      "未知年龄"

3. **状态判断与列表转换**\ ：

   - ``isJust :: Maybe a -> Bool``
   - ``isNothing :: Maybe a -> Bool``
   - ``listToMaybe :: [a] -> Maybe a``\ ：安全提取列表首项（全函数，面对空列表返回 ``Nothing``\ ）。
   - ``maybeToList :: Maybe a -> [a]``\ ：将 Maybe 转换为 0 个或 1 个元素的列表。

4. **数据清洗利器：mapMaybe**\ ：

   .. code:: haskell

      mapMaybe :: (a -> Maybe b) -> [a] -> [b]

   遍历列表，同时完成计算过滤与 ``Just`` 值的安全解包（自动丢弃所有 ``Nothing``\ ）：

   .. code:: text

      ghci> import Text.Read (readMaybe)
      ghci> mapMaybe readMaybe ["10", "abc", "20", "xyz", "30"] :: [Int]
      [10,20,30]

.. warning::

   **严禁使用偏函数 fromJust**\ ：标准库中的 ``fromJust :: Maybe a -> a`` 在遇到 ``Nothing`` 时会直接引发异常崩溃。在生产环境中应严格禁止使用 ``fromJust``\ ，始终使用模式匹配、\ ``fromMaybe`` 或 ``maybe``\ 。

Either：携带具体错误原因的计算
--------------------------------------------------------------------------------

当操作失败时，\ ``Maybe`` 只能告诉调用方“失败了”，却无法告知“因何失败”。此时我们需要 ``Either``\ ：

.. code:: haskell

   data Either a b = Left a | Right b deriving (Show, Eq)

- ``Left a``\ ：按照工业界惯例，用于包裹\ **错误详情**\ （Error / Exception）。
- ``Right b``\ ：用于包裹\ **成功的计算结果**\ （“Right”在英文中同时具有“正确”与“右侧”的双重含义）。

.. tip::

   **他山之石：多语言心智模型对照**\ ：

   - **Maybe a**\ ：
     - **Rust**\ ：等价于 **``Option<T>``**\ （``Some(x)`` 与 ``None``\ ）。
     - **Java**\ ：类似于 **``Optional<T>``**\ 。
     - **Swift**\ ：等价于可选类型 **``Optional<T>``**\ （语法糖 ``T?``\ ）。
   - **Either a b**\ ：
     - **Rust**\ ：等价于 **``Result<T, E>``**\ （``Ok(val)`` 对应 ``Right``\ ，\ ``Err(e)`` 对应 ``Left``\ ）。
     - **Swift**\ ：等价于 **``Result<Success, Failure>``**\ 。
     - **C++23**\ ：等价于 **``std::expected<T, E>``**\ 。
   - **核心设计哲学**\ ：主流现代系统语言正在全面转向这种函数式错误处理模型——告别返回 ``-1``\ 、\ ``null`` 或粗暴抛出运行时异常的不透明做法，将一切可能失败的操作作为确定性的类型显式暴露在接口签名中。

Data.Either 核心工具箱
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **核心解构函数：either（折叠原语）**\ ：

   .. code:: haskell

      either :: (a -> c) -> (b -> c) -> Either a b -> c

   分别提供错误处理函数与成功处理函数，将双分支统一归一化为类型 ``c``\ ：

   .. code:: text

      ghci> handleRes = either (\err -> "失败: " ++ err) (\n -> "成功: " ++ show n)
      ghci> handleRes (Left "网络超时")
      "失败: 网络超时"
      ghci> handleRes (Right 200)
      "成功: 200"

2. **批量归并函数**\ ：

   - ``lefts :: [Either a b] -> [a]``\ ：提取列表中所有的错误项。
   - ``rights :: [Either a b] -> [b]``\ ：提取列表中所有的成功项。
   - ``partitionEithers :: [Either a b] -> ([a], [b])``\ ：一次性将列表拆解为“全部错误组成的列表”与“全部成功结果组成的列表”。

.. code:: text

   ghci> import Data.Either
   ghci> results = [Right 10, Left "超时", Right 20, Left "磁盘满"]
   ghci> partitionEithers results
   (["超时","磁盘满"],[10,20])

纯错误处理 vs 运行时异常：选型与避坑哲学
--------------------------------------------------------------------------------

在 Haskell 软件架构中，错误被明确划分为两个世界：

1. **纯代码领域（代数可恢复错误）**\ ：
   - **适用工具**\ ：\ ``Maybe``\ 、\ ``Either``\ 、\ ``ExceptT``\ 。
   - **哲学**\ ：业务预期内的错误（如参数非法、用户未找到、权限不足）必须\ **显式编码在函数类型签名中**\ 。类型系统强制调用方必须处理所有错误分支，否则编译无法通过。
2. **IO 领域（环境外部不可控异常）**\ ：
   - **适用工具**\ ：\ ``Control.Exception`` （如 ``try``\ 、\ ``catch``\ 、\ ``throwIO``\ ）。
   - **哲学**\ ：针对真正的灾难性硬件/系统错误（如磁盘断开、网络断开、内存耗尽）。

.. warning::

   **绝对警惕在纯代码中使用纯 throw**\ ：
   Haskell 允许在任何纯代码中调用 ``throw :: Exception e => e -> a``\ 。但是，因为 Haskell 是惰性求值的，被抛出的纯异常**不会在生成它的地方立即崩溃**，而是像一颗“未引爆的地雷”悬挂在 Thunk 中，直到数个模块之外的代码某天真正求值该值时才不可预期地爆发！
   **工程铁律**\ ：纯函数只返回 ``Either`` 或 ``Maybe``\ ；如需抛出异常，必须在 ``IO`` 上下文中使用确定性的 ``throwIO``\ 。

实战案例：构建强类型领域业务校验管道
--------------------------------------------------------------------------------

在工程开发中，我们应当避免使用模糊的 ``String`` 表达错误，而是为业务定义强类型的领域错误枚举：

.. code:: haskell

   -- 强类型领域错误定义
   data ValidationError
     = UsernameTooShort !Int
     | AgeOutOfRange !Int
     | InvalidEmailFormat !String
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

小结
--------------------------------------------------------------------------------

- Haskell 用静态代数类型彻底消灭 Null 引用。
- ``maybe`` 与 ``either`` 提供了标准而完备的高阶函数解构范式。
- 严禁在纯代码中使用 ``fromJust`` 偏函数或纯 ``throw``\ 。
- 业务错误使用 ``Either`` 显式标注在函数签名中，系统物理异常在 IO 层由 ``Control.Exception`` 统一防御。
