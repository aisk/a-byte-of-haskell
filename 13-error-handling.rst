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

它就是上一章的和类型，基数是 ``|a| + 1``\ ；下面的 ``Either`` 同样是和类型。上一章说“非法状态在类型上无法表示”，用到“失败”这件事上就是这两个类型：函数可能失败，就让它的返回类型多出一个分支，而不是返回一个碰巧不能用的值。

.. warning::

   **避免使用 fromJust**\ ：标准库中的 ``fromJust :: Maybe a -> a`` 遇到 ``Nothing`` 时会抛出异常，等于把 Null 引用又请了回来。应当用模式匹配、\ ``fromMaybe`` 或 ``maybe`` 代替。

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

用一个任务串起常用函数
--------------------------------------------------------------------------------

``Data.Maybe`` 和 ``Data.Either`` 里的辅助函数不多，与其逐个记，不如看它们在一段真实代码里各自出场的位置。任务是：从一份 ``[(String, String)]`` 形式的配置里取出端口号，转成数字，再检查范围。三步都可能失败，缺键、不是数字、超出范围。

最直接的写法是逐层 ``case``\ ：

.. code:: haskell

   import Text.Read (readMaybe)

   config :: [(String, String)]
   config = [("port", "8080"), ("workers", "abc"), ("timeout", "30")]

   portV1 :: Int
   portV1 =
     case lookup "port" config of
       Nothing -> 80
       Just s -> case readMaybe s of
         Nothing -> 80
         Just n -> n

能工作，但默认值 ``80`` 写了两遍，每多一步就多一层缩进。\ ``Data.Maybe`` 的两个函数专门收拾这种代码：

.. code:: haskell

   maybe     :: b -> (a -> b) -> Maybe a -> b   -- Nothing 给默认值，Just 交给函数
   fromMaybe :: a -> Maybe a -> a               -- 只补默认值

.. code:: haskell

   import Data.Maybe (fromMaybe)

   -- maybe 把“缺键”和“不是数字”折叠成同一个 Nothing
   parsePort :: Maybe Int
   parsePort = maybe Nothing readMaybe (lookup "port" config)

   portV2 :: Int
   portV2 = fromMaybe 80 parsePort

范围检查需要说明失败原因，换成 ``Either``\ 。\ ``either`` 是它的解构函数，两个分支各给一个处理函数，把结果统一成同一种类型：

.. code:: haskell

   either :: (a -> c) -> (b -> c) -> Either a b -> c

.. code:: haskell

   checkPort :: Int -> Either String Int
   checkPort n
     | n > 0 && n < 65536 = Right n
     | otherwise = Left ("端口超出范围: " ++ show n)

   report :: Either String Int -> String
   report = either ("配置错误: " ++) (\n -> "监听端口 " ++ show n)

.. code:: text

   ghci> report (checkPort portV2)
   "监听端口 8080"
   ghci> report (checkPort 70000)
   "配置错误: 端口超出范围: 70000"

批量处理时有两种态度。\ ``mapMaybe`` 跳过失败的项，只收集成功的；\ ``traverse`` 要求全部成功，否则整体失败。后者来自 Traversable 一章，这里先看效果：

.. code:: text

   ghci> import Data.Maybe (mapMaybe)
   ghci> mapMaybe readMaybe ["10", "abc", "20"] :: [Int]
   [10,20]
   ghci> traverse readMaybe ["10", "abc", "20"] :: Maybe [Int]
   Nothing
   ghci> traverse readMaybe ["10", "20"] :: Maybe [Int]
   Just [10,20]

还有几个一看名字就知道用途的函数：\ ``isJust``\ 、\ ``isNothing``\ 、\ ``catMaybes``\ （丢掉列表里的 ``Nothing``\ ）、\ ``listToMaybe``\ （安全地取列表首项）、\ ``maybeToList``\ ；\ ``Either`` 这边有 ``lefts``\ 、\ ``rights`` 和 ``partitionEithers``\ （一次遍历把成功和失败分成两个列表）。

怎么选
--------------------------------------------------------------------------------

面对一个可能失败的函数，先问调用方需要知道什么，再决定返回类型：

.. list-table::
   :header-rows: 1
   :widths: 30 26 44

   * - 调用方需要什么
     - 用什么
     - 说明
   * - 只需要知道有没有
     - ``Maybe a``
     - 缺失是正常情况而不是错误，如 ``lookup``\ 、\ ``listToMaybe``\ 、\ ``readMaybe``\ 。
   * - 需要一句原因，只用来打日志或显示
     - ``Either String a``
     - 适合原型和小工具。第九章的 ``decodeUtf8'`` 就是这种签名。
   * - 需要根据原因分别处理
     - ``Either MyError a``\ ，\ ``MyError`` 是自定义和类型
     - 调用方可以模式匹配，编译器检查是否漏了分支。见下一节。
   * - 多个可能失败的步骤混在 IO 里
     - ``ExceptT`` 或 ``MonadError``
     - 避免每一步都手写 ``case``\ ，见单子变换子一章。
   * - 来自外部环境的故障
     - ``Control.Exception`` 的异常
     - 文件不存在、网络中断、磁盘满。这些错误不属于业务逻辑，在 IO 层用 ``try``\ 、\ ``catch`` 处理，见 IO 一章。
   * - 程序本身的 bug，逻辑上不可能到达的分支
     - ``error``\ 、\ ``undefined``
     - 只用于“到了这里说明代码写错了”。预期内的失败（用户输入非法、记录不存在）不能用它们，否则调用方无从处理，类型签名也在撒谎。

``String`` 之所以只适合原型，是因为调用方拿到 ``Left "端口超出范围: 70000"`` 后只能原样打印，无法可靠地判断是哪一种错误，更不能让编译器检查“每种错误都处理了”。换成自定义的和类型，这两件事都有了。

.. warning::

   **注意纯代码中的 throw**\ ：
   Haskell 允许在纯代码中调用 ``throw :: Exception e => e -> a``\ 。但由于惰性求值，抛出的异常\ **不会在生成它的地方立即触发**\ ，而是藏在 Thunk 中，直到别处的代码真正求值该值时才会出现，那时已经很难追溯来源。
   建议的做法是：纯函数只返回 ``Either`` 或 ``Maybe``\ ；需要抛异常时，在 ``IO`` 中使用 ``throwIO``\ 。

示例：强类型的业务校验
--------------------------------------------------------------------------------

按上表的第三行，为业务定义专门的错误类型：

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

两个校验单独看都很清楚，真正的问题出在把它们组合起来的时候：

.. code:: haskell

   data User = User { userName :: String, userAge :: Int } deriving (Show)

   validateUser :: String -> Int -> ValidationResult User
   validateUser name age =
     case validateUsername name of
       Left err -> Left err
       Right validName -> case validateAge age of
         Left err -> Left err
         Right validAge -> Right (User validName validAge)

.. code:: text

   ghci> validateUser "Alice" 30
   Right (User {userName = "Alice", userAge = 30})
   ghci> validateUser "Al" 200
   Left (UsernameTooShort 2)

``Left err -> Left err`` 这样的分支每加一个字段就要再写一遍，内容完全一样：失败就原样往外传。这正是单子一章要解决的问题，用 ``do`` 记号写出来只剩三行。另外注意上面的组合在第一个错误处就停下了，\ ``"Al"`` 和 ``200`` 两个问题只报了一个；如果想把所有错误一次收齐，要用应用函子一章的 ``Validation``\ 。

小结
--------------------------------------------------------------------------------

- ``Maybe`` 与 ``Either`` 只是普通的和类型，Haskell 用它们代替 Null 引用，把“可能失败”写进签名。
- ``maybe`` 与 ``either`` 是标准解构函数，\ ``fromMaybe`` 补默认值，\ ``mapMaybe`` 跳过失败项，\ ``traverse`` 要求全部成功。
- 选类型先问调用方需要知道什么：只要有无用 ``Maybe``\ ，要原因用 ``Either``\ ，要分别处理就自定义错误类型，混在 IO 里的多步失败用 ``ExceptT``\ ，环境故障用异常，\ ``error`` 只留给 bug。
- 避免在纯代码中使用 ``fromJust`` 或 ``throw``\ 。
- 逐层 ``case`` 组合多个 ``Either`` 会产生重复的分支，单子一章消除它。
