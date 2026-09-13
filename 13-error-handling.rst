错误处理：Maybe、Either 与异常
================================================================================

图灵奖得主托尼·霍尔（Tony Hoare）把空指针（Null / Nil）称为计算机科学中“价值十亿美元的错误”。隐式的空值会破坏类型系统的可靠性，让运行时充满难以追踪的崩溃。

Haskell 没有 Null 引用，而是把“数据缺失”与“失败原因”表达为显式的、强类型的代数数据类型。

Maybe：表达“数据可能缺失”
--------------------------------------------------------------------------------

定义在 Prelude 中：

.. code:: haskell

   data Maybe a = Nothing | Just a deriving (Show, Eq)

- ``Nothing``\ ：数据不存在。
- ``Just a``\ ：包含一个有效的数据 ``a``\ 。

Data.Maybe 常用函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

标准库 ``Data.Maybe`` 提供了一组辅助函数：

1. **fromMaybe（提供默认值）**\ ：

   .. code:: haskell

      fromMaybe :: a -> Maybe a -> a

   如果是 ``Just x`` 则取出 ``x``\ ；如果是 ``Nothing`` 则返回给定的默认值：

   .. code:: text

      ghci> import Data.Maybe
      ghci> fromMaybe 0 (Just 42)
      42
      ghci> fromMaybe 0 Nothing
      0

2. **maybe（解构原语）**\ ：

   .. code:: haskell

      maybe :: b -> (a -> b) -> Maybe a -> b

   同时提供默认值与变换函数，一步完成模式匹配与映射：

   .. code:: text

      ghci> maybe "未知年龄" (\n -> show n ++ " 岁") (Just 25)
      "25 岁"
      ghci> maybe "未知年龄" (\n -> show n ++ " 岁") Nothing
      "未知年龄"

3. **状态判断与列表转换**\ ：

   - ``isJust :: Maybe a -> Bool``
   - ``isNothing :: Maybe a -> Bool``
   - ``listToMaybe :: [a] -> Maybe a``\ ：安全地取列表首项（全函数，空列表返回 ``Nothing``\ ）。
   - ``maybeToList :: Maybe a -> [a]``\ ：把 Maybe 转换为 0 个或 1 个元素的列表。

4. **mapMaybe**\ ：

   .. code:: haskell

      mapMaybe :: (a -> Maybe b) -> [a] -> [b]

   遍历列表，把映射结果为 ``Just`` 的值解包收集，丢弃所有 ``Nothing``\ ：

   .. code:: text

      ghci> import Text.Read (readMaybe)
      ghci> mapMaybe readMaybe ["10", "abc", "20", "xyz", "30"] :: [Int]
      [10,20,30]

.. warning::

   **避免使用 fromJust**\ ：标准库中的 ``fromJust :: Maybe a -> a`` 遇到 ``Nothing`` 时会抛出异常。应当用模式匹配、\ ``fromMaybe`` 或 ``maybe`` 代替。

Either：携带错误原因
--------------------------------------------------------------------------------

操作失败时，\ ``Maybe`` 只能表达“失败了”，无法说明“为什么失败”。这时需要 ``Either``\ ：

.. code:: haskell

   data Either a b = Left a | Right b deriving (Show, Eq)

- ``Left a``\ ：按惯例用于包裹\ **错误信息**\ 。
- ``Right b``\ ：用于包裹\ **成功的结果**\ （“Right”在英文中既有“正确”也有“右侧”的含义）。

.. tip::

   **如果你熟悉其他语言**\ ：

   - **Maybe a**\ ：

     - **Rust**\ ：对应 ``Option<T>``\ （``Some(x)`` 与 ``None``\ ）。
     - **Java**\ ：类似 ``Optional<T>``\ 。
     - **Swift**\ ：对应可选类型 ``Optional<T>``\ （语法糖 ``T?``\ ）。

   - **Either a b**\ ：

     - **Rust**\ ：对应 ``Result<T, E>``\ （``Ok(val)`` 对应 ``Right``\ ，\ ``Err(e)`` 对应 ``Left``\ ）。
     - **Swift**\ ：对应 ``Result<Success, Failure>``\ 。
     - **C++23**\ ：对应 ``std::expected<T, E>``\ 。

   - **共同思路**\ ：这些语言都在转向这种错误处理模型：不再返回 ``-1``\ 、\ ``null`` 或直接抛出运行时异常，而是把“可能失败”写进函数的类型签名里。

Data.Either 常用函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **either（解构原语）**\ ：

   .. code:: haskell

      either :: (a -> c) -> (b -> c) -> Either a b -> c

   分别提供错误处理函数与成功处理函数，把两个分支统一为类型 ``c``\ ：

   .. code:: text

      ghci> handleRes = either (\err -> "失败: " ++ err) (\n -> "成功: " ++ show n)
      ghci> handleRes (Left "网络超时")
      "失败: 网络超时"
      ghci> handleRes (Right 200)
      "成功: 200"

2. **批量拆分**\ ：

   - ``lefts :: [Either a b] -> [a]``\ ：提取列表中所有的错误项。
   - ``rights :: [Either a b] -> [b]``\ ：提取列表中所有的成功项。
   - ``partitionEithers :: [Either a b] -> ([a], [b])``\ ：一次遍历把列表拆成错误列表与成功列表。

.. code:: text

   ghci> import Data.Either
   ghci> results = [Right 10, Left "超时", Right 20, Left "磁盘满"]
   ghci> partitionEithers results
   (["超时","磁盘满"],[10,20])

纯错误处理与运行时异常的分工
--------------------------------------------------------------------------------

Haskell 程序中的错误通常分为两类：

1. **纯代码中的可恢复错误**\ ：

   - **工具**\ ：\ ``Maybe``\ 、\ ``Either``\ 、\ ``ExceptT``\ 。
   - **原则**\ ：业务上预期内的错误（参数非法、用户不存在、权限不足）应当\ **写在函数的类型签名中**\ 。类型系统会要求调用方处理所有错误分支，否则无法通过编译。

2. **IO 中的外部异常**\ ：

   - **工具**\ ：\ ``Control.Exception``\ （\ ``try``\ 、\ ``catch``\ 、\ ``throwIO`` 等）。
   - **原则**\ ：用于真正来自外部环境的错误（磁盘错误、网络中断、内存耗尽）。

.. warning::

   **注意纯代码中的 throw**\ ：
   Haskell 允许在纯代码中调用 ``throw :: Exception e => e -> a``\ 。但由于惰性求值，抛出的异常\ **不会在生成它的地方立即触发**\ ，而是藏在 Thunk 中，直到别处的代码真正求值该值时才会出现，那时已经很难追溯来源。
   建议的做法是：纯函数只返回 ``Either`` 或 ``Maybe``\ ；需要抛异常时，在 ``IO`` 中使用 ``throwIO``\ 。

示例：强类型的业务校验
--------------------------------------------------------------------------------

实际项目中，应尽量避免用 ``String`` 表达错误，而是为业务定义专门的错误类型：

.. code:: haskell

   -- 领域错误类型
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

在 REPL 中验证：

.. code:: text

   ghci> validateUsername "Al"
   Left (UsernameTooShort 2)

   ghci> validateUsername "Alice"
   Right "Alice"

   ghci> validateAge (-5)
   Left (AgeOutOfRange (-5))

小结
--------------------------------------------------------------------------------

- Haskell 用 ``Maybe`` 与 ``Either`` 代替 Null 引用。
- ``maybe`` 与 ``either`` 是这两个类型的标准解构函数。
- 避免在纯代码中使用 ``fromJust`` 或 ``throw``\ 。
- 业务错误用 ``Either`` 写进函数签名；外部环境异常在 IO 层用 ``Control.Exception`` 处理。
