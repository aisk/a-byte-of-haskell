命令行程序：参数、环境变量与选项解析
================================================================================

命令行工具是 Haskell 最容易上手的实际项目。本章介绍写一个命令行程序需要的几样东西：读取参数和环境变量、用退出码报告结果、把标准输入当作数据来源、调用外部命令，以及解析选项的三种方式：手写模式匹配、\ ``base`` 自带的 ``getOpt``\ 、第三方库 ``optparse-applicative``\ 。

这些模块大部分在 ``base`` 里。调用外部命令用到的 ``process`` 随 GHC 一起发布，最后一节的 ``optparse-applicative`` 需要从 Hackage 安装，两者都要写进 ``build-depends``\ ：

.. code:: text

   build-depends:    base >= 4.14 && < 5
                   , text >= 2.0
                   , process >= 1.6
                   , optparse-applicative >= 0.17

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

解析选项：三个层次
--------------------------------------------------------------------------------

``getArgs`` 只是把命令行按空格切开交给你，\ ``-v``\ 、\ ``--output FILE`` 这类选项要自己识别。按程序的复杂程度，有三种做法：直接对参数列表做模式匹配、用 ``base`` 自带的 ``getOpt``\ 、用第三方库 ``optparse-applicative``\ 。前两种简单介绍，重点放在第三种，因为它很好地展示了 Haskell 做抽象的方式。

第一层：直接匹配参数列表
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

参数只有一两个、位置固定时，对 ``getArgs`` 的结果做模式匹配就够了：

.. code:: haskell

   module Main (main) where

   import System.Environment (getArgs)
   import System.Exit (exitFailure)
   import System.IO (hPutStrLn, stderr)

   main :: IO ()
   main = do
     args <- getArgs
     case args of
       [path] -> run False path
       ["-v", path] -> run True path
       _ -> do
         hPutStrLn stderr "用法: manual [-v] FILE"
         exitFailure
     where
       run verbose path = do
         content <- readFile path
         if verbose
           then putStrLn (path ++ ": " ++ show (length (lines content)) ++ " 行")
           else print (length (lines content))

.. code:: sh

   $ runghc Manual.hs notes.txt
   3
   $ runghc Manual.hs -v notes.txt
   notes.txt: 3 行
   $ runghc Manual.hs notes.txt -v
   用法: manual [-v] FILE

这种写法的问题是每一种参数组合都要写一个分支。\ ``-v`` 能不能放在文件名后面、能不能和别的选项同时出现，取决于你列举了哪些模式。选项一多，分支数量会爆炸。

第二层：getOpt
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``base`` 自带的 ``System.Console.GetOpt`` 实现了 GNU 风格的选项识别：短选项 ``-v``\ 、长选项 ``--verbose``\ 、带参数的 ``-o FILE`` 与 ``--output=FILE``\ 、多个短选项合写 ``-vl``\ ，选项和普通参数可以任意交错。

每个选项用一个 ``OptDescr a`` 值描述，\ ``getOpt`` 把参数列表变成三样东西：

.. code:: haskell

   data OptDescr a = Option
     [Char]        -- 短选项字符，如 ['o']
     [String]      -- 长选项名，如 ["output"]
     (ArgDescr a)  -- 是否带参数，以及如何得到一个 a
     String        -- 帮助文本

   data ArgDescr a
     = NoArg a                            -- 不带参数，如 -v
     | ReqArg (String -> a) String        -- 必须带参数，如 -o FILE；第二项是参数的占位名
     | OptArg (Maybe String -> a) String  -- 参数可选

   getOpt :: ArgOrder a -> [OptDescr a] -> [String] -> ([a], [String], [String])
   --                                                    选项值  普通参数  错误信息

   usageInfo :: String -> [OptDescr a] -> String

类型参数 ``a`` 是“解析出一个选项后得到什么值”。习惯做法是让 ``a`` 为 ``Options -> Options``\ ，也就是一个修改配置记录的函数。这样解析结果是一串修改函数，从默认配置出发用 ``foldl`` 依次应用即可，后出现的选项覆盖先出现的：

.. code:: haskell

   module Main (main) where

   import System.Console.GetOpt
   import System.Environment (getArgs)
   import System.Exit (exitFailure)
   import System.IO (hPutStrLn, stderr)

   data Options = Options
     { optVerbose :: Bool
     , optOutput :: Maybe FilePath
     } deriving (Show)

   defaultOptions :: Options
   defaultOptions = Options { optVerbose = False, optOutput = Nothing }

   -- 每个选项解析出来的值是一个修改 Options 的函数
   options :: [OptDescr (Options -> Options)]
   options =
     [ Option ['v'] ["verbose"] (NoArg (\o -> o { optVerbose = True })) "输出详细信息"
     , Option ['o'] ["output"] (ReqArg (\f o -> o { optOutput = Just f }) "FILE") "把结果写入 FILE"
     ]

   main :: IO ()
   main = do
     args <- getArgs
     -- Permute 表示选项和普通参数可以任意交错
     case getOpt Permute options args of
       (actions, files, []) -> do
         let opts = foldl (flip id) defaultOptions actions
         print opts
         print files
       (_, _, errs) -> do
         hPutStrLn stderr (concat errs ++ usageInfo "用法: getopt [选项] [文件...]" options)
         exitFailure

.. code:: text

   $ runghc GetOpt.hs -v -o out.txt a.txt b.txt
   Options {optVerbose = True, optOutput = Just "out.txt"}
   ["a.txt","b.txt"]

   $ runghc GetOpt.hs --bogus
   unrecognized option `--bogus'
   用法: getopt [选项] [文件...]
     -v       --verbose      输出详细信息
     -o FILE  --output=FILE  把结果写入 FILE

``flip id`` 的类型是 ``b -> (b -> c) -> c``\ ，也就是“拿一个值和一个函数，把函数应用到值上”。\ ``ArgOrder`` 还可以取 ``RequireOrder``\ ，遇到第一个非选项参数后不再识别选项，适合 ``prog [选项] 子命令 [子命令的参数]`` 这种结构。

``getOpt`` 只做选项识别，帮助文本和错误信息由它自动生成。但参数值一律是 ``String``\ ，要转成数字得自己调用 ``readMaybe``\ ；哪个选项必填、哪些互斥，也得自己检查。它适合选项不多、又不想引入依赖的小工具。

第三层：optparse-applicative
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``optparse-applicative`` 是 Haskell 社区最常用的选项解析库，需要写进 ``build-depends``\ ，模块名是 ``Options.Applicative``\ 。它的核心类型是 ``Parser a``\ ，表示“从命令行解析出一个 ``a``\ ”。库提供的基本解析器只有几个：

.. code:: haskell

   switch    :: Mod FlagFields Bool -> Parser Bool           -- 开关，如 -v
   strOption :: Mod OptionFields String -> Parser String     -- 带字符串参数的选项，如 -o FILE
   option    :: ReadM a -> Mod OptionFields a -> Parser a    -- 带参数并转换类型，option auto 按 Read 解析
   argument  :: ReadM a -> Mod ArgumentFields a -> Parser a  -- 位置参数，argument str 取原始字符串

这个库的重点不在这几个函数，而在于\ **它没有发明任何自己的组合方式**\ 。描述一个选项、把多个选项合并成一份配置、在两种写法之间二选一，用的全是本书前面已经介绍过的类型类。

**描述一个选项：Monoid 的 <>**\ 。每个基本解析器接收一个 ``Mod`` 值，说明选项的名字、参数占位、帮助文本、默认值。\ ``Mod`` 是 Monoid 的实例，\ ``long "output"``\ 、\ ``short 'o'``\ 、\ ``metavar "FILE"``\ 、\ ``help "..."``\ 、\ ``value "out.txt"`` 每一个都是一个 ``Mod``\ ，用 Monoid 一章的 ``<>`` 拼起来就是完整的描述。不需要的项直接不写：

.. code:: haskell

   strOption (long "output" <> short 'o' <> metavar "FILE" <> help "把结果写入 FILE")

**合并多个选项：Applicative 的 <$> 和 <*>**\ 。\ ``Parser`` 是 Applicative 的实例。回忆 Applicative 一章的 ``Profile <$> mName <*> mAge <*> mEmail``\ ：用 ``<$>`` 和 ``<*>`` 把构造函数依次应用到几个上下文中的值上。把 ``Maybe`` 换成 ``Parser``\ ，就是把几个选项解析器合并成整份配置的解析器：

.. code:: haskell

   Options <$> switch (long "lines" <> short 'l')
           <*> switch (long "words" <> short 'w')
           <*> strOption (long "output" <> short 'o' <> metavar "FILE")

各选项在命令行上出现的顺序无关紧要，\ ``<*>`` 只表示“这几项都要解析，然后把结果交给构造函数”。这里用的是 ``Maybe``\ 、\ ``IO`` 和列表上的同一对运算符，没有任何这个库独有的语法。

**二选一：Alternative 的 <|>**\ 。\ ``Control.Applicative`` 里还有一个 Applicative 的子类 ``Alternative``\ ，本书之前没有用到：

.. code:: haskell

   class Applicative f => Alternative f where
     empty :: f a
     (<|>) :: f a -> f a -> f a   -- 先试左边，失败了再试右边

.. code:: text

   ghci> Nothing <|> Just 3
   Just 3
   ghci> Just 1 <|> Just 3
   Just 1

``Parser`` 也是它的实例。\ ``ToFile <$> strOption (...) <|> pure ToStdout`` 表示“给了 ``-o`` 就写文件，否则写标准输出”，其中 ``pure x`` 是“不消耗任何参数、直接给出 ``x``\ ”的解析器，正好用来兜底。\ ``Alternative`` 还附带了 ``optional``\ （零或一次，结果是 ``Maybe``\ ）、\ ``many``\ （零或多次，结果是列表）和 ``some``\ （一或多次），它们是根据 ``<|>`` 和 ``pure`` 定义的通用函数，所以 ``Parser`` 一实现 ``Alternative`` 就自动拥有了。\ ``many (argument str (metavar "FILE..."))`` 就是“任意多个文件名”。

下面用它重写本章的 ``wc`` 简化版 ``hwc``\ 。它支持 ``-l``\ 、\ ``-w``\ 、\ ``-c`` 三个统计项，可以把结果写到 ``-o`` 指定的文件，没有给出文件名时从标准输入读取：

.. code:: haskell

   module Main (main) where

   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO
   import Options.Applicative

   data Output = ToStdout | ToFile FilePath
     deriving (Show)

   data Options = Options
     { optLines :: Bool
     , optWords :: Bool
     , optChars :: Bool
     , optOutput :: Output
     , optFiles :: [FilePath]
     } deriving (Show)

   -- 描述单个选项：用 <> 把各项修饰拼在一起；二选一：用 <|>
   outputParser :: Parser Output
   outputParser =
     ToFile <$> strOption (long "output" <> short 'o' <> metavar "FILE" <> help "把结果写入 FILE")
       <|> pure ToStdout

   -- 合并多个选项：用 <$> 和 <*> 把它们组成一个 Options
   optionsParser :: Parser Options
   optionsParser =
     Options
       <$> switch (long "lines" <> short 'l' <> help "统计行数")
       <*> switch (long "words" <> short 'w' <> help "统计单词数")
       <*> switch (long "chars" <> short 'c' <> help "统计字符数")
       <*> outputParser
       <*> many (argument str (metavar "FILE..."))

   count :: Options -> T.Text -> String
   count opts content = unwords (map show selected)
     where
       -- 没有指定任何统计项时，三项全部输出
       showAll = not (optLines opts || optWords opts || optChars opts)
       selected =
         [length (T.lines content) | optLines opts || showAll]
           ++ [length (T.words content) | optWords opts || showAll]
           ++ [T.length content | optChars opts || showAll]

   main :: IO ()
   main = do
     opts <- execParser (info (helper <*> optionsParser) (progDesc "统计行数、单词数和字符数"))
     results <- case optFiles opts of
       [] -> do
         content <- TIO.getContents
         return [count opts content]
       files -> mapM (\path -> do
                         content <- TIO.readFile path
                         return (count opts content ++ "\t" ++ path)) files
     let output = unlines results
     case optOutput opts of
       ToStdout -> putStr output
       ToFile path -> writeFile path output

``main`` 里的 ``info`` 给解析器加上程序描述，\ ``execParser`` 读取 ``getArgs``\ 、执行解析，出错时打印用法并以退出码 1 结束。\ ``helper`` 的类型是 ``Parser (a -> a)``\ ：看到 ``--help`` 就打印帮助并退出，否则给出 ``id``\ 。连 ``--help`` 的处理也只是又一个用 ``<*>`` 接进来的解析器。\ ``count`` 里的列表推导式 ``[ x | 条件 ]`` 在条件为假时得到空列表，为真时得到单元素列表，是按条件拼装列表的一种简洁写法。

因为有第三方依赖，这个程序要放进 cabal 项目里编译。几种用法的实际输出：

.. code:: text

   $ hwc --help
   Usage: hwc [-l|--lines] [-w|--words] [-c|--chars] [-o|--output FILE] [FILE...]

     统计行数、单词数和字符数

   Available options:
     -h,--help                Show this help text
     -l,--lines               统计行数
     -w,--words               统计单词数
     -c,--chars               统计字符数
     -o,--output FILE         把结果写入 FILE

   $ hwc notes.txt app/Main.hs
   3 8 16	notes.txt
   57 258 1755	app/Main.hs

   $ hwc -lw notes.txt
   3 8	notes.txt

   $ echo hello world | hwc -w
   2

   $ hwc --bogus x
   Invalid option `--bogus'

   Usage: hwc [-l|--lines] [-w|--words] [-c|--chars] [-o|--output FILE] [FILE...]
   ...
   $ echo $?
   1

   $ hwc -o
   The option `-o` expects an argument.
   ...

用法行、选项对齐、错误信息都是根据同一个 ``Parser Options`` 生成的。因为它是一个普通的值，还可以拆成几段在多个子命令之间复用，或者用 ``execParserPure`` 在测试里喂一个参数列表而不经过真正的命令行。子命令由 ``subparser`` 和 ``command`` 提供，本书不展开。

.. tip::

   **如果你熟悉其他语言**\ ：Python 的 ``argparse`` 靠反复调用 ``parser.add_argument`` 往一个可变对象里登记选项，Go 的 ``flag`` 包把选项绑定到指针上，Rust 的 ``clap`` 用派生宏读取结构体上的属性标注。每个库都要自己定义“怎么把多个选项组合成一份配置”，使用者也得为每个库单独学一套组合方式。

   ``optparse-applicative`` 只定义了“一个选项是什么”（\ ``Parser a``\ ）和“一条修饰是什么”（\ ``Mod``\ ），然后为它们实现 ``Monoid``\ 、\ ``Applicative`` 和 ``Alternative`` 实例。“怎么组合”不是这个库的事：拼修饰用 ``<>``\ ，合并选项用 ``<$>`` 和 ``<*>``\ ，二选一用 ``<|>``\ ，重复和可选用 ``many`` 和 ``optional``\ 。这些运算符和函数来自标准库，读者在 ``Maybe``\ 、列表和 ``IO`` 上已经用过，换到命令行解析上含义不变。

   这是 Haskell 做抽象的典型方式。“合并”“二选一”这类通用形状定义在类型类里，只定义一次；新的库通过写实例接入，使用者带着已有的知识直接上手。相比之下，其他语言的库通常各自发明一套组合 API，学过 ``argparse`` 对学 ``cobra`` 并没有多少帮助。

小结
--------------------------------------------------------------------------------

- **参数与环境变量**\ ：\ ``getArgs`` 读参数，\ ``lookupEnv`` 读环境变量并用 ``readMaybe`` 转换，缺失时走默认值。
- **退出码**\ ：\ ``exitFailure`` 和 ``exitWith (ExitFailure n)`` 报告失败，错误信息写到 ``stderr``\ 。
- **标准输入**\ ：纯文本过滤用 ``interact``\ ，需要逐行 IO 时用 ``isEOF`` 加 ``getLine`` 循环。
- **外部命令**\ ：\ ``callProcess`` 只看成败，\ ``readProcess`` 取输出，\ ``readProcessWithExitCode`` 三项都要。
- **解析选项的三个层次**\ ：参数固定时直接模式匹配；选项不多又不想加依赖时用 ``getOpt``\ ，值类型取 ``Options -> Options`` 再 ``foldl`` 合并；正式的工具用 ``optparse-applicative``\ ，修饰用 ``<>`` 拼，选项用 ``<$>`` 和 ``<*>`` 合并，二选一用 ``<|>``\ 。它没有自己的组合语法，靠的全是标准类型类的实例。
