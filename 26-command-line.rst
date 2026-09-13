命令行程序：参数、环境变量与 getOpt
================================================================================

命令行工具是 Haskell 最容易上手的实际项目。本章介绍写一个命令行程序需要的几样东西：读取参数和环境变量、用退出码报告结果、把标准输入当作数据来源、调用外部命令，以及用 ``System.Console.GetOpt`` 解析选项。

这些模块全部在 ``base`` 里，只有调用外部命令用到的 ``process`` 是单独的包。它随 GHC 一起发布，但要写进 ``build-depends``\ ：

.. code:: text

   build-depends:    base >= 4.14 && < 5
                   , text >= 2.0
                   , process >= 1.6

参数与环境变量
--------------------------------------------------------------------------------

``System.Environment`` 提供了程序运行环境的基本信息：

.. code:: haskell

   getArgs        :: IO [String]                -- 命令行参数，不含程序名
   getProgName    :: IO String                  -- 程序名
   lookupEnv      :: String -> IO (Maybe String) -- 读环境变量，缺失时返回 Nothing
   getEnv         :: String -> IO String         -- 读环境变量，缺失时抛出异常
   setEnv         :: String -> String -> IO ()   -- 设置环境变量，只影响当前进程及其子进程
   getEnvironment :: IO [(String, String)]       -- 全部环境变量

``lookupEnv`` 比 ``getEnv`` 更常用，因为环境变量缺失通常是正常情况，应该走默认值而不是崩溃。环境变量的值总是字符串，转成数字时用 ``Text.Read.readMaybe`` 而不是 ``read``\ ，这样非法输入不会让程序直接崩溃：

.. code:: haskell

   module Main (main) where

   import System.Environment (getArgs, getProgName, lookupEnv)
   import System.Exit (exitWith, ExitCode (..))
   import System.IO (hPutStrLn, stderr)
   import Text.Read (readMaybe)

   main :: IO ()
   main = do
     prog <- getProgName
     args <- getArgs
     putStrLn ("程序名: " ++ prog)
     putStrLn ("参数: " ++ show args)

     -- 环境变量缺失时使用默认端口，无法解析为数字时报错退出
     portEnv <- lookupEnv "PORT"
     port <- case portEnv of
       Nothing -> return (8080 :: Int)
       Just s -> case readMaybe s of
         Just n -> return n
         Nothing -> do
           hPutStrLn stderr ("PORT 不是合法的端口号: " ++ s)
           exitWith (ExitFailure 2)
     putStrLn ("监听端口: " ++ show port)

.. code:: sh

   $ PORT=3000 runghc Env.hs --verbose x
   程序名: Env.hs
   参数: ["--verbose","x"]
   监听端口: 3000

   $ PORT=abc runghc Env.hs
   程序名: Env.hs
   参数: []
   PORT 不是合法的端口号: abc
   $ echo $?
   2

退出码与标准错误
--------------------------------------------------------------------------------

上面的例子用到了 ``System.Exit``\ ：

.. code:: haskell

   exitSuccess :: IO a               -- 退出码 0
   exitFailure :: IO a               -- 退出码 1
   exitWith    :: ExitCode -> IO a   -- ExitSuccess 或 ExitFailure n

退出码是命令行程序和 shell 之间的约定：0 表示成功，非 0 表示失败。\ ``set -e`` 的脚本、\ ``a && b`` 这样的组合、CI 系统判断一个步骤是否通过，都依赖它。程序正常走完 ``main`` 时退出码是 0，未捕获的异常会得到 1。

与退出码配套的是输出的去向。正常结果写到标准输出，错误和警告写到标准错误：

.. code:: haskell

   import System.IO (hPutStrLn, stderr)

   hPutStrLn stderr "警告：配置文件缺失，使用默认值"

这样 ``myprog > result.txt`` 只会把结果存进文件，警告仍然显示在终端上。

.. note::

   ``exitWith`` 的返回类型是 ``IO a`` 而不是 ``IO ()``\ ，因为它永远不会正常返回，可以放在任何需要 IO 值的位置。它的实现是抛出一个 ``ExitCode`` 异常，由运行时在最外层接住。这意味着 ``catch`` 所有异常的代码会把它一并捕获，所以捕获异常时应当指定具体的类型，而不是 ``SomeException``\ 。

标准输入作为管道
--------------------------------------------------------------------------------

命令行工具经常放在管道里使用：\ ``cat log.txt | myfilter | sort``\ 。这时程序从标准输入读数据，写到标准输出。

最简单的写法是 ``interact :: (String -> String) -> IO ()``\ 。它把整个标准输入交给一个纯函数，把返回值写到标准输出：

.. code:: haskell

   module Main (main) where

   import Data.Char (toUpper)

   main :: IO ()
   main = interact (map toUpper)

.. code:: sh

   $ echo hello world | runghc Upper.hs
   HELLO WORLD

``interact`` 和 ``getContents`` 是惰性的，输入一边到达一边处理，对于按行过滤这类任务正合适，也不会把整个输入都放进内存。如果处理逻辑需要按行做 IO（例如给每行加上行号并立即输出），可以用 ``isEOF`` 和 ``getLine`` 写一个循环：

.. code:: haskell

   module Main (main) where

   import System.IO (isEOF)

   main :: IO ()
   main = loop 1
     where
       loop :: Int -> IO ()
       loop n = do
         eof <- isEOF
         if eof
           then return ()
           else do
             line <- getLine
             putStrLn (show n ++ ": " ++ line)
             loop (n + 1)

.. code:: sh

   $ printf 'a\nb\n' | runghc Number.hs
   1: a
   2: b

要读取 ``Text`` 而不是 ``String``\ ，换成 ``Data.Text.IO`` 的 ``getContents`` 和 ``getLine`` 即可。

调用外部命令
--------------------------------------------------------------------------------

``process`` 包的 ``System.Process`` 模块用来启动子进程。三个函数覆盖了大部分需求：

.. code:: haskell

   module Main (main) where

   import System.Exit (ExitCode (..))
   import System.Process (callProcess, readProcess, readProcessWithExitCode)

   main :: IO ()
   main = do
     -- 只关心执行成功与否，输出直接透传到终端；失败时抛出异常
     callProcess "ls" ["-l", "/tmp"]

     -- 捕获标准输出，最后一个参数是喂给标准输入的内容
     version <- readProcess "ghc" ["--numeric-version"] ""
     putStrLn ("GHC 版本: " ++ version)

     -- 同时拿到退出码、标准输出和标准错误
     (code, out, err) <- readProcessWithExitCode "git" ["status", "--short"] ""
     case code of
       ExitSuccess -> putStr out
       ExitFailure n -> putStrLn ("git 退出码 " ++ show n ++ ": " ++ err)

参数以列表传入，不经过 shell，所以不需要考虑空格和引号的转义。确实需要 shell 特性（管道、通配符）时，用 ``callCommand "ls *.hs | wc -l"``\ 。

用 getOpt 解析选项
--------------------------------------------------------------------------------

参数少的时候，直接对 ``getArgs`` 的结果做模式匹配就够了。一旦出现 ``-v``\ 、\ ``--output FILE`` 这类选项，手写解析很快会变得繁琐。\ ``base`` 自带的 ``System.Console.GetOpt`` 实现了 GNU 风格的选项解析，不需要额外依赖。

描述选项
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

每个选项用一个 ``OptDescr a`` 值描述：

.. code:: haskell

   data OptDescr a = Option
     [Char]        -- 短选项字符，如 ['o']
     [String]      -- 长选项名，如 ["output"]
     (ArgDescr a)  -- 是否带参数，以及如何得到一个 a
     String        -- 帮助文本

   data ArgDescr a
     = NoArg a                       -- 不带参数，如 -v
     | ReqArg (String -> a) String   -- 必须带参数，如 -o FILE；第二项是参数的占位名
     | OptArg (Maybe String -> a) String  -- 参数可选

类型参数 ``a`` 是"解析出一个选项后得到什么值"。最常见的做法是让 ``a`` 为 ``Options -> Options``\ ，也就是一个修改配置记录的函数。这么做的原因下面会解释。

解析
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   getOpt :: ArgOrder a -> [OptDescr a] -> [String] -> ([a], [String], [String])

   usageInfo :: String -> [OptDescr a] -> String

``getOpt`` 接收参数顺序策略、选项描述列表和 ``getArgs`` 的结果，返回一个三元组：

1. 解析出的选项值列表；
2. 非选项参数，例如文件名；
3. 错误信息，例如未知选项或缺少参数。列表为空表示解析成功。

``ArgOrder`` 决定选项和普通参数混在一起时怎么处理：

- ``Permute``\ ：选项和普通参数可以任意交错，\ ``prog a.txt -l b.txt`` 与 ``prog -l a.txt b.txt`` 等价。这是最常用的选择。
- ``RequireOrder``\ ：遇到第一个非选项参数后，后面的内容全部当作普通参数。适合 ``prog [选项] 子命令 [子命令的参数]`` 这种结构。
- ``ReturnInOrder f``\ ：把每个非选项参数也通过 ``f`` 转成选项值，保持它们在命令行中的相对顺序。

``usageInfo`` 根据同一份描述列表生成帮助文本，第一个参数是放在开头的说明行。

``getOpt`` 支持的写法与 GNU 工具一致：短选项 ``-l``\ 、长选项 ``--lines``\ 、带参数的 ``-o out.txt``\ 、\ ``-oout.txt``\ 、\ ``--output out.txt``\ 、\ ``--output=out.txt``\ ，以及多个短选项合写 ``-lw``\ 。长选项写前缀也能识别，只要不产生歧义。

为什么选项值是函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``getOpt`` 返回的是一个列表 ``[a]``\ ，它本身不知道怎么把多个选项合并成一份配置。如果 ``a`` 是一个专门的和类型（\ ``Verbose | Output FilePath | ...``\ ），后续还要再写一遍模式匹配去填充记录。

让 ``a = Options -> Options`` 就省掉了这一步。每个选项自带"如何修改配置"的知识，解析结果是一串修改函数，从默认配置出发依次应用即可：

.. code:: haskell

   foldl (flip id) defaultOptions actions

``flip id`` 的类型是 ``b -> (b -> c) -> c``\ ，也就是"拿一个值和一个函数，把函数应用到值上"。\ ``foldl`` 从左到右，用累积的配置去调用列表里的每个函数。后出现的选项覆盖先出现的，与用户的直觉一致。

完整示例：一个 wc
--------------------------------------------------------------------------------

下面实现一个 ``wc`` 的简化版 ``hwc``\ 。它支持 ``-l``\ 、\ ``-w``\ 、\ ``-c`` 三个统计项，可以把结果写到 ``-o`` 指定的文件，没有给出文件名时从标准输入读取。

.. code:: haskell

   module Main (main) where

   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO
   import System.Console.GetOpt
   import System.Environment (getArgs, getProgName)
   import System.Exit (exitFailure, exitSuccess)
   import System.IO (hPutStrLn, stderr)

   data Options = Options
     { optLines :: Bool
     , optWords :: Bool
     , optChars :: Bool
     , optOutput :: Maybe FilePath
     , optHelp :: Bool
     } deriving (Show)

   defaultOptions :: Options
   defaultOptions = Options
     { optLines = False
     , optWords = False
     , optChars = False
     , optOutput = Nothing
     , optHelp = False
     }

   -- 每个选项被解析后，得到一个修改 Options 的函数
   options :: [OptDescr (Options -> Options)]
   options =
     [ Option ['l'] ["lines"]  (NoArg (\o -> o { optLines = True }))  "统计行数"
     , Option ['w'] ["words"]  (NoArg (\o -> o { optWords = True }))  "统计单词数"
     , Option ['c'] ["chars"]  (NoArg (\o -> o { optChars = True }))  "统计字符数"
     , Option ['o'] ["output"] (ReqArg (\f o -> o { optOutput = Just f }) "FILE")
                                                                       "把结果写入 FILE"
     , Option ['h'] ["help"]   (NoArg (\o -> o { optHelp = True }))   "显示帮助"
     ]

   parseArgs :: [String] -> IO (Options, [FilePath])
   parseArgs args = do
     prog <- getProgName
     let header = "用法: " ++ prog ++ " [选项] [文件...]"
     case getOpt Permute options args of
       (actions, files, []) -> do
         -- 依次把每个修改函数应用到默认值上
         let opts = foldl (flip id) defaultOptions actions
         if optHelp opts
           then putStr (usageInfo header options) >> exitSuccess
           else return (opts, files)
       (_, _, errs) -> do
         hPutStrLn stderr (concat errs ++ usageInfo header options)
         exitFailure

   count :: Options -> T.Text -> String
   count opts content = unwords (map show selected)
     where
       -- 没有指定任何统计项时，三项全部输出
       showAll = not (optLines opts || optWords opts || optChars opts)
       selected =
         [ length (T.lines content) | optLines opts || showAll ]
           ++ [ length (T.words content) | optWords opts || showAll ]
           ++ [ T.length content | optChars opts || showAll ]

   main :: IO ()
   main = do
     (opts, files) <- parseArgs =<< getArgs
     results <- case files of
       [] -> do
         content <- TIO.getContents
         return [count opts content]
       _ -> mapM (\path -> do
                     content <- TIO.readFile path
                     return (count opts content ++ "\t" ++ path)) files
     let output = unlines results
     case optOutput opts of
       Nothing -> putStr output
       Just path -> writeFile path output

``count`` 里的列表推导式 ``[ x | 条件 ]`` 在条件为假时得到空列表，为真时得到单元素列表，是按条件拼装列表的一种简洁写法。

几种用法的实际输出：

.. code:: sh

   $ runghc hwc.hs --help
   用法: hwc.hs [选项] [文件...]
     -l       --lines        统计行数
     -w       --words        统计单词数
     -c       --chars        统计字符数
     -o FILE  --output=FILE  把结果写入 FILE
     -h       --help         显示帮助

   $ runghc hwc.hs notes.txt hwc.hs
   3 8 39	notes.txt
   76 372 2473	hwc.hs

   $ runghc hwc.hs -lw notes.txt
   3 8	notes.txt

   $ echo hello world | runghc hwc.hs -w
   2

   $ runghc hwc.hs --bogus x
   unrecognized option `--bogus'
   用法: hwc.hs [选项] [文件...]
     -l       --lines        统计行数
     ...
   $ echo $?
   1

   $ runghc hwc.hs -o
   option `-o' requires an argument FILE
   ...

帮助文本的对齐、未知选项和缺少参数的错误信息，都是 ``getOpt`` 和 ``usageInfo`` 自动生成的。

.. tip::

   **如果你熟悉其他语言**\ ：\ ``getOpt`` 的定位相当于 Python 的 ``getopt`` 模块或 C 的 ``getopt_long``\ ，只做选项识别，不做类型转换和校验。Python 的 ``argparse``\ 、Go 的 ``cobra`` 那种带子命令、自动类型转换的功能，在 Haskell 里由 ``optparse-applicative`` 提供。它把每个选项写成一个 ``Parser a``\ ，再用 Applicative 一章介绍的 ``<$>`` 和 ``<*>`` 组合成整个配置的解析器。程序规模变大以后可以考虑迁移，本书不展开。

小结
--------------------------------------------------------------------------------

- **参数与环境变量**\ ：\ ``getArgs`` 读参数，\ ``lookupEnv`` 读环境变量并用 ``readMaybe`` 转换，缺失时走默认值。
- **退出码**\ ：\ ``exitFailure`` 和 ``exitWith (ExitFailure n)`` 报告失败，错误信息写到 ``stderr``\ 。
- **标准输入**\ ：纯文本过滤用 ``interact``\ ，需要逐行 IO 时用 ``isEOF`` 加 ``getLine`` 循环。
- **外部命令**\ ：\ ``callProcess`` 只看成败，\ ``readProcess`` 取输出，\ ``readProcessWithExitCode`` 三项都要。
- **getOpt**\ ：用 ``OptDescr`` 列表描述选项，值类型取 ``Options -> Options``\ ，解析后用 ``foldl (flip id) defaultOptions`` 合并，\ ``usageInfo`` 生成帮助。
