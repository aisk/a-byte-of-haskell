I/O 交互、异常处理与工程组织
================================================================================

如果一门语言只能在纯内存中完成计算，却无法与屏幕、磁盘、网络交互，那它将失去实用价值。Haskell 通过独创的 **IO 类型**\ ，在坚守“纯函数引用透明”的同时，构建了一套极为健全、安全的现实世界交互模型。

本章将系统解析 IO 的本质、流式文件操作、基于 ``Control.Exception`` 的异常安全体系以及现代大型 Cabal 工程组织规范。

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

流式文件句柄与 withFile
--------------------------------------------------------------------------------

虽然标准库提供了简便的 ``readFile`` 与 ``writeFile``\ ，但在处理大文件或需要精确控制资源生命周期时，应当使用基于句柄（Handle）的流式操作：

.. code:: haskell

   import System.IO

   -- 低级句柄操作（需手动管理关闭）：
   manualFileDemo :: IO ()
   manualFileDemo = do
     handle <- openFile "sample.txt" ReadMode
     line <- hGetLine handle
     putStrLn line
     hClose handle

RAII 资源安全释放：withFile
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果上面的代码在读取文件时抛出异常，``hClose`` 将永远不会被执行，导致句柄泄漏。使用 ``withFile`` 可以保证句柄在使用后\ **必定被自动安全关闭**\ ：

.. code:: haskell

   safeFileDemo :: IO ()
   safeFileDemo = withFile "sample.txt" ReadMode $ \handle -> do
     contents <- hGetContents handle
     putStr contents

运行时异常安全体系：Control.Exception
--------------------------------------------------------------------------------

在纯代码内部，我们使用 ``Maybe`` 或 ``Either`` 进行显式错误处理；但在与文件、网络等物理环境交互时，不可避免地会遇到外部硬件异常（如文件不存在、网络中断）。

捕获异常：try 与 catch
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用 ``Control.Exception`` 模块提供的异常处理原语：

.. code:: haskell

   import Control.Exception
   import System.IO

   safeRead :: FilePath -> IO (Either IOException String)
   safeRead path = try (readFile path)

   demoCatch :: IO ()
   demoCatch = do
     result <- safeRead "not_exist.txt"
     case result of
       Left ex  -> putStrLn $ "捕获到底层 IO 异常: " ++ show ex
       Right txt -> putStrLn txt

终极资源安全保障：bracket
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``bracket`` 是 Haskell 中最著名的资源管理模式，等价于许多语言中的 ``try-finally``\ ，但更为可靠：

.. code:: haskell

   bracket :: IO a          -- 1. 资源分配（Allocate）
           -> (a -> IO b)   -- 2. 资源清理（Cleanup，无论是否发生异常保证百分之百执行）
           -> (a -> IO c)   -- 3. 业务使用（Use）
           -> IO c

实战示例：数据库连接管理
~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   data Connection = Connection

   openConn :: IO Connection
   openConn = putStrLn "打开数据库连接" >> return Connection

   closeConn :: Connection -> IO ()
   closeConn _ = putStrLn "安全关闭数据库连接"

   useConn :: Connection -> IO ()
   useConn _ = do
     putStrLn "执行核心查询业务..."
     -- 即使此处中途崩溃抛错，closeConn 也必定会被调用！

   mainConnectionTask :: IO ()
   mainConnectionTask = bracket openConn closeConn useConn

现代 Cabal 工程实践指南
--------------------------------------------------------------------------------

现代大型 Haskell 项目统一采用 **Cabal** 进行依赖管理与自动化构建。

初始化项目
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在一个空目录下执行：

.. code:: sh

   $ cabal init --interactive

根据命令行向导选择项目类型（Executable 可执行程序，或 Library 库），Cabal 会自动生成规范的目录骨架与 ``.cabal`` 项目配置文件。

.cabal 配置文件核心结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: text

   cabal-version:      3.0
   name:               my-awesome-app
   version:            0.1.0.0
   license:            BSD-3-Clause
   author:             Developer
   build-type:         Simple

   common common-settings
       default-language: Haskell2010
       ghc-options:      -Wall -O2
       build-depends:    base >= 4.14 && < 5
                       , text
                       , containers

   library
       import:           common-settings
       hs-source-dirs:   src
       exposed-modules:  MyLib.Calculator
                       , MyLib.Network
       other-modules:    MyLib.Internal

   executable my-awesome-app
       import:           common-settings
       hs-source-dirs:   app
       main-is:          Main.hs
       build-depends:    my-awesome-app

核心构建与运行命令
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: sh

   $ cabal build      -- 自动下载依赖并编译项目
   $ cabal run        -- 运行 app/Main.hs 可执行程序
   $ cabal test       -- 自动化运行测试套件
   $ cabal repl       -- 在当前工程完整依赖环境中加载 GHCi 交互式解释器
