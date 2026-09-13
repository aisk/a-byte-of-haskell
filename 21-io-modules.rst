I/O 交互、异常处理、并发与工程组织
================================================================================

如果一门语言只能在纯内存中完成计算，却无法与屏幕、磁盘、网络交互，那它将失去实用价值。Haskell 通过独创的 **IO 类型**\ ，在坚守“纯函数引用透明”的同时，构建了一套健全、安全的现实世界交互模型。

本章将系统解析 IO 的本质、流式文件操作、惰性 IO 陷阱防范、运行时异常安全体系、轻量级并发以及现代大型 Cabal 工程组织规范。

IO 的本质：副作用蓝图与食谱
--------------------------------------------------------------------------------

在很多命令式语言中，任何函数随时随地都能暗中修改外部文件或发起网络请求。而在 Haskell 中：

1. **类型即边界**\ ：任何包含与外界物理交互的操作，其返回值类型\ **必须**\ 被标记为 ``IO a``\ 。
2. **动作即食谱（Action as Recipe）**\ ：类型为 ``IO String`` 的值，并不是“一段已经被读取的文本”，而是\ **一份由操作系统执行、读取字符串的“执行食谱”或“待办动作”**\ 。
3. **在 main 中交由运行环境执行**\ ：纯代码负责编排组装这些动作，只有当动作最终被挂载在程序入口——``main :: IO ()`` 时，GHC 运行时环境才会真正将其执行。

控制台与基础文本读写
--------------------------------------------------------------------------------

.. code:: haskell

   putStrLn :: String -> IO ()        -- 输出字符串并换行
   putStr   :: String -> IO ()        -- 输出字符串不换行
   getLine  :: IO String              -- 从控制台读取一行文本
   print    :: Show a => a -> IO ()   -- 打印实现了 Show 的任何数据（等价于 putStrLn . show）

交互示例：

.. code:: haskell

   interactiveGreeting :: IO ()
   interactiveGreeting = do
     putStr "请输入您的姓名: "
     name <- getLine
     putStrLn $ "您好，" ++ name ++ "！欢迎来到 Haskell 的世界。"

流式文件句柄与 RAII 资源安全
--------------------------------------------------------------------------------

低级句柄与 withFile
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在处理大文件或需要精确控制资源生命周期时，应当使用基于句柄（Handle）的流式操作：

.. code:: haskell

   import System.IO

   -- 自动安全关闭句柄（等价于 RAII / with 语句）：
   safeFileDemo :: IO ()
   safeFileDemo = withFile "sample.txt" ReadMode $ \handle -> do
     contents <- hGetContents handle
     putStr contents

工程警示：致命的惰性 I/O 陷阱（Lazy I/O Pitfall）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

标准 Prelude 中的 ``readFile`` 采用了“惰性 I/O”设计：它不会一次性把整个文件读入内存，而是返回一个惰性字符流。

.. warning::

   **惰性 I/O 的死锁与句柄耗尽隐患**\ ：
   当使用 ``readFile`` 读取文件，若后续业务代码没有完全消费该字符串、或者试图在同一进程中重新打开写入该文件时，底层文件句柄将一直保持被操作系统锁定的挂起状态！
   **工程铁律**\ ：在生产级工程中，坚决避免使用 Prelude 的 ``readFile``\ ；始终使用来自 ``Data.Text.IO`` 或 ``Data.ByteString`` 的严格读取函数（如 ``T.IO.readFile``\ ），或者在超大文件场景下使用流式库（如 Streaming / Conduit）。

终极资源安全保障：bracket
--------------------------------------------------------------------------------

``bracket`` 是 Haskell 中最著名的资源管理模式，等价于许多语言中的 ``try-finally``\ ，但更为可靠：

.. code:: haskell

   import Control.Exception (bracket)

   bracket :: IO a          -- 1. 资源分配（Allocate）
           -> (a -> IO b)   -- 2. 资源清理（Cleanup，无论是否发生异常保证百分之百执行）
           -> (a -> IO c)   -- 3. 业务使用（Use）
           -> IO c

实战示例：安全管理数据库连接
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   data Connection = Connection

   openConn :: IO Connection
   openConn = putStrLn "打开数据库连接" >> return Connection

   closeConn :: Connection -> IO ()
   closeConn _ = putStrLn "安全关闭数据库连接"

   useConn :: Connection -> IO ()
   useConn _ = do
     putStrLn "执行核心查询业务..."
     -- 即使此处中途发生错误，closeConn 也百分之百会被调用！

   mainConnectionTask :: IO ()
   mainConnectionTask = bracket openConn closeConn useConn

运行时异常安全体系：Control.Exception
--------------------------------------------------------------------------------

捕获异常：try 与 catch
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   import Control.Exception
   import qualified Data.Text.IO as TIO

   safeRead :: FilePath -> IO (Either IOException String)
   safeRead path = try (readFile path)

   demoCatch :: IO ()
   demoCatch = do
     result <- safeRead "not_exist.txt"
     case result of
       Left ex   -> putStrLn $ "捕获到底层 IO 异常: " ++ show ex
       Right txt -> putStrLn txt

轻量级并发与多核并行
--------------------------------------------------------------------------------

GHC 运行时内置了极高吞吐的 M:N 绿色线程调度器。使用 ``forkIO`` 可以创建开销仅约 1KB 的轻量级线程，单个进程可轻松并发数十万个线程。

Async 现代并发库
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

工业界广泛使用 ``async`` 库进行类型安全的结构化并发编排：

.. code:: haskell

   import Control.Concurrent.Async

   fetchDataFromA :: IO String
   fetchDataFromA = return "来自节点 A 的响应"

   fetchDataFromB :: IO String
   fetchDataFromB = return "来自节点 B 的响应"

   -- 并发同时执行两个任务，等待双方全部完成：
   fetchBoth :: IO (String, String)
   fetchBoth = concurrently fetchDataFromA fetchDataFromB

   -- 竞态执行：哪个任务先返回就取哪个，自动取消另一个：
   fetchFastest :: IO (Either String String)
   fetchFastest = race fetchDataFromA fetchDataFromB

在编译时追加 ``-threaded -rtsopts -with-rtsopts="-N"``\ ，即可自动开启多核 CPU 硬件并行调度。

模块化系统与工程组织
--------------------------------------------------------------------------------

模块声明与显式导出控制
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``src/MyLib/Calculator.hs`` 中规范组织模块：

.. code:: haskell

   module MyLib.Calculator
     ( -- 1. 显式导出具体函数
       addNumbers
     , calculateTotal
       -- 2. 导出类型及其所有数据构造器
     , CalculationMode(..)
       -- 3. 仅导出类型名称（隐藏内部构造器，实现抽象数据类型 ADT）
     , SecretToken
     ) where

   data CalculationMode = Fast | Precise deriving (Show)
   newtype SecretToken = SecretToken String

   addNumbers :: Int -> Int -> Int
   addNumbers x y = x + y

   calculateTotal :: [Int] -> Int
   calculateTotal = sum

模块导入语法规范
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **全量导入**\ ：\ ``import MyLib.Calculator``
- **显式选择导入**\ ：\ ``import MyLib.Calculator (addNumbers)``
- **带前缀限定导入**\ ：\ ``import qualified Data.Map.Strict as Map``
- **排除特定冲突项导入**\ ：\ ``import Prelude hiding (head, id)``

现代 Cabal 工程实践
--------------------------------------------------------------------------------

现代大型 Haskell 项目统一采用 **Cabal** 进行依赖管理与自动化构建。

.cabal 配置文件核心结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: text

   cabal-version:      3.0
   name:               my-awesome-app
   version:            0.1.0.0
   license:            BSD-3-Clause
   build-type:         Simple

   common common-settings
       default-language: Haskell2010
       ghc-options:      -Wall -O2
       build-depends:    base >= 4.14 && < 5
                       , text
                       , containers
                       , async

   library
       import:           common-settings
       hs-source-dirs:   src
       exposed-modules:  MyLib.Calculator
       other-modules:    MyLib.Internal

   executable my-awesome-app
       import:           common-settings
       hs-source-dirs:   app
       main-is:          Main.hs
       build-depends:    my-awesome-app

核心构建与运行命令
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: sh

   $ cabal build      -- 自动下载依赖并编译项目
   $ cabal run        -- 运行 app/Main.hs 可执行程序
   $ cabal test       -- 自动化运行测试套件
   $ cabal repl       -- 在当前工程完整依赖环境中加载 GHCi 交互式解释器

小结
--------------------------------------------------------------------------------

- ``IO`` 将物理副作用约束在类型系统安全边界之内，保持纯逻辑的引用透明。
- 坚决规避 Prelude 的惰性 ``readFile``\ ，优先采用严格文本读取或 ``bracket``/``withFile`` 模式。
- GHC 的轻量级绿色线程与 ``Async`` 库提供了高并发、低心智负担的并发编程模型。
- 通过严谨的模块导出控制与规范的 Cabal 配置，构建高内聚、易维护的现代大型工程架构。
