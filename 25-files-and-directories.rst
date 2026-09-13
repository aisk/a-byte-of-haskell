读写文件与目录
================================================================================

从这一章开始，我们把注意力从语言本身转到日常编程会用到的库上。本章讨论最基础的一类需求：读写文件、遍历目录、拼接路径，以及处理这些操作可能抛出的异常。

涉及的模块大多来自 ``base``\ 、\ ``text`` 以及随 GHC 一起发布的 ``directory`` 和 ``filepath`` 两个包。后两者虽然随 GHC 安装，但仍然需要写进 ``.cabal`` 文件的 ``build-depends`` 里：

.. code:: text

   build-depends:    base >= 4.14 && < 5
                   , text >= 2.0
                   , bytestring >= 0.11
                   , directory >= 1.3
                   , filepath >= 1.4

IO 一章已经介绍过 ``withFile``\ 、\ ``bracket`` 和惰性 I/O 的问题，字符串一章介绍过 ``Text`` 和 ``ByteString`` 的区别。本章在这两章的基础上，直接给出可以照着写的代码。

三种读写方式
--------------------------------------------------------------------------------

按内容的类型，读写文件的函数分成三组，签名的形状完全一样，只是字符串类型不同：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 模块
     - 类型
     - 适用场景
   * - ``Prelude``
     - ``String``
     - 小脚本、教学示例。惰性读取，句柄关闭时机不可控
   * - ``Data.Text.IO``
     - ``Text``
     - 文本文件的默认选择。严格读取，读完即关闭句柄
   * - ``Data.ByteString``
     - ``ByteString``
     - 二进制文件，或者需要自己控制编码的场合

三组函数的名字都是 ``readFile``\ 、\ ``writeFile``\ 、\ ``appendFile``\ ：

.. code:: haskell

   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO
   import qualified Data.ByteString as BS

   demo :: IO ()
   demo = do
     -- String 版本，来自 Prelude
     s <- readFile "notes.txt"
     putStrLn (take 20 s)

     -- Text 版本，推荐用于文本文件
     t <- TIO.readFile "notes.txt"
     TIO.writeFile "copy.txt" t
     TIO.appendFile "copy.txt" (T.pack "追加的一行\n")

     -- ByteString 版本，按字节读写
     bytes <- BS.readFile "logo.png"
     BS.writeFile "logo-copy.png" bytes

``writeFile`` 会覆盖已有内容，\ ``appendFile`` 在末尾追加。三者都会在文件不存在时自动创建。

Prelude 的 ``readFile`` 返回惰性的 ``String``\ ，文件句柄要等字符串被完全消费后才关闭。IO 一章的警告已经说明了它带来的问题。一个常见的错误是读一个文件再写回同一个文件：

.. code:: haskell

   -- 运行时报错：data.txt: withFile: resource busy (file is locked)
   badRewrite :: IO ()
   badRewrite = do
     content <- readFile "data.txt"
     writeFile "data.txt" (map toUpper content)

``writeFile`` 执行时，\ ``readFile`` 打开的句柄还没有关闭，于是触发文件锁冲突。改用 ``Data.Text.IO`` 就不存在这个问题，因为 ``TIO.readFile`` 返回时已经读完并关闭了文件。

编码
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``String`` 和 ``Text`` 的读写函数都按\ **当前区域设置（locale）**\ 决定编码。在多数 Linux 和 macOS 环境里这就是 UTF-8。在编码不确定的环境（例如某些 Docker 镜像、Windows 终端）里，读中文文件可能报 ``invalid byte sequence`` 错误。有两种办法把编码固定为 UTF-8：

.. code:: haskell

   import System.IO
   import GHC.IO.Encoding (setLocaleEncoding)

   -- 方法一：在 main 开头设置全局默认编码
   main :: IO ()
   main = do
     setLocaleEncoding utf8
     ...

   -- 方法二：只对某个句柄设置
   readUtf8 :: FilePath -> IO String
   readUtf8 path = withFile path ReadMode $ \h -> do
     hSetEncoding h utf8
     contents <- hGetContents h
     length contents `seq` return contents   -- 强制读完再关闭句柄

如果编码不是 UTF-8，或者需要自己处理解码失败，就用 ``Data.ByteString`` 读入字节，再用字符串一章介绍的 ``decodeUtf8'`` 之类的函数转换。

按行处理
--------------------------------------------------------------------------------

日志、CSV、配置文件都是按行组织的。把整个文件读成 ``Text``\ ，用 ``T.lines`` 拆开，处理完再用 ``T.unlines`` 拼回去，是最直接的写法。

下面是一个简化版的 ``grep``\ ，打印文件中包含关键字的行：

.. code:: haskell

   module Main (main) where

   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO
   import System.Environment (getArgs)

   main :: IO ()
   main = do
     [keyword, path] <- getArgs
     content <- TIO.readFile path
     let matched = filter (T.isInfixOf (T.pack keyword)) (T.lines content)
     TIO.putStr (T.unlines matched)

.. code:: sh

   $ runghc Grep.hs words notes.txt
   line two has more words

按同样的思路，统计行数和单词数只是把 ``filter`` 换成 ``length``\ ：

.. code:: haskell

   stats :: T.Text -> (Int, Int)
   stats content = (length (T.lines content), length (T.words content))

处理结果要写到\ **另一个文件**\ ，或者先全部读完再写回。\ ``TIO.readFile`` 是严格的，所以下面的写法是安全的：

.. code:: haskell

   -- 去掉每行末尾的空白，写回原文件
   stripTrailing :: FilePath -> IO ()
   stripTrailing path = do
     content <- TIO.readFile path
     TIO.writeFile path (T.unlines (map T.stripEnd (T.lines content)))

句柄操作
--------------------------------------------------------------------------------

一次读完整个文件对多数场景足够。文件很大、或者只想读前几行时，就要回到 ``System.IO`` 的句柄（Handle）接口，一行一行地读：

.. code:: haskell

   module Main (main) where

   import Control.Monad (unless)
   import System.IO

   -- 逐行读取，只在内存中保留当前行
   countLongLines :: FilePath -> IO Int
   countLongLines path =
     withFile path ReadMode $ \h -> do
       hSetEncoding h utf8
       let loop n = do
             eof <- hIsEOF h
             if eof
               then return n
               else do
                 line <- hGetLine h
                 loop (if length line > 80 then n + 1 else n)
       loop 0

   main :: IO ()
   main = do
     hSetBuffering stdout NoBuffering
     putStr "请输入文件路径: "
     path <- getLine
     n <- countLongLines path
     unless (n == 0) $
       hPutStrLn stderr ("警告：有 " ++ show n ++ " 行超过 80 个字符")
     putStrLn "检查完成"

几个值得注意的点：

- ``withFile`` 保证句柄在函数返回或抛出异常时关闭，不需要手动 ``hClose``\ 。
- ``hIsEOF`` 与 ``hGetLine`` 配合是逐行读取的标准写法。\ ``hGetLine`` 在文件末尾会抛出异常，所以要先检查。
- ``stdout``\ 、\ ``stderr``\ 、\ ``stdin`` 也是句柄。\ ``hPutStrLn stderr`` 把错误信息写到标准错误，不会混进程序的正常输出里，方便用管道处理。
- 标准输出默认是行缓冲的。\ ``putStr`` 打印一个没有换行的提示符时，文字可能停留在缓冲区里，用户看不到。\ ``hSetBuffering stdout NoBuffering`` 关闭缓冲，或者在 ``putStr`` 后面调用 ``hFlush stdout``\ 。

写文件对应的是 ``hPutStr`` 和 ``hPutStrLn``\ ，用法与读取对称：

.. code:: haskell

   writeReport :: FilePath -> [String] -> IO ()
   writeReport path rows =
     withFile path WriteMode $ \h -> do
       hSetEncoding h utf8
       mapM_ (hPutStrLn h) rows

``Data.Text.IO`` 里也有同名的 ``hGetLine``\ 、\ ``hPutStrLn`` 等函数，接受 ``Text`` 参数。

目录与路径
--------------------------------------------------------------------------------

System.Directory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``directory`` 包的 ``System.Directory`` 模块封装了操作系统的文件系统调用。常用的函数：

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - 函数
     - 作用
   * - ``doesFileExist :: FilePath -> IO Bool``
     - 文件是否存在
   * - ``doesDirectoryExist :: FilePath -> IO Bool``
     - 目录是否存在
   * - ``listDirectory :: FilePath -> IO [FilePath]``
     - 列出目录下的条目名，不含 ``.`` 和 ``..``\ ，也不含路径前缀
   * - ``createDirectoryIfMissing :: Bool -> FilePath -> IO ()``
     - 创建目录，第一个参数为 ``True`` 时连同父目录一起创建
   * - ``removeFile``\ 、\ ``renameFile``\ 、\ ``copyFile``
     - 删除、重命名、复制文件
   * - ``removeDirectoryRecursive``
     - 递归删除目录
   * - ``getCurrentDirectory``\ 、\ ``getHomeDirectory``
     - 当前工作目录、用户主目录
   * - ``getFileSize :: FilePath -> IO Integer``
     - 文件大小（字节）
   * - ``getModificationTime :: FilePath -> IO UTCTime``
     - 最后修改时间

一个把这些函数串起来的例子：

.. code:: haskell

   module Main (main) where

   import System.Directory
   import System.FilePath

   main :: IO ()
   main = do
     home <- getHomeDirectory
     let workDir = home </> ".myapp" </> "cache"
     createDirectoryIfMissing True workDir
     writeFile (workDir </> "hello.txt") "hello\n"
     copyFile (workDir </> "hello.txt") (workDir </> "hello" <.> "bak")
     entries <- listDirectory workDir
     print entries
     size <- getFileSize (workDir </> "hello.txt")
     mtime <- getModificationTime (workDir </> "hello.txt")
     putStrLn (show size ++ " 字节，修改于 " ++ show mtime)
     removeFile (workDir </> "hello.bak")
     removeDirectoryRecursive (home </> ".myapp")

.. code:: text

   ["hello.txt","hello.bak"]
   6 字节，修改于 2026-09-13 08:06:24.461704378 UTC

System.FilePath
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

上面的例子用 ``</>`` 拼接路径，而不是 ``++ "/" ++``\ 。\ ``FilePath`` 只是 ``String`` 的别名，直接拼接当然可行，但 ``filepath`` 包的 ``System.FilePath`` 模块处理了几件容易出错的事：

- 分隔符按平台选择，Windows 上是 ``\``\ ，其他系统是 ``/``\ ；
- ``"dir/" </> "file"`` 和 ``"dir" </> "file"`` 得到同样的结果，不会出现双斜杠；
- 拆分文件名、扩展名、目录名的函数都是纯函数，不需要访问文件系统。

.. code:: haskell

   ghci> import System.FilePath
   ghci> "src" </> "Main.hs"
   "src/Main.hs"
   ghci> takeExtension "src/Main.hs"
   ".hs"
   ghci> takeFileName "src/Main.hs"
   "Main.hs"
   ghci> takeDirectory "src/Main.hs"
   "src"
   ghci> dropExtension "src/Main.hs"
   "src/Main"
   ghci> "report" <.> "pdf"
   "report.pdf"
   ghci> splitDirectories "src/App/Main.hs"
   ["src","App","Main.hs"]

.. tip::

   **如果你熟悉其他语言**\ ：\ ``System.Directory`` 对应 Python 的 ``os`` 和 ``shutil``\ ，\ ``System.FilePath`` 对应 ``os.path``\ 。Go 里则分别是 ``os`` 和 ``path/filepath``\ 。区别在于 Haskell 把纯的路径运算和有副作用的文件系统访问分在两个包里，前者的函数没有 ``IO`` 类型。

异常处理
--------------------------------------------------------------------------------

文件不存在、没有权限、磁盘已满，这些情况都以 ``IOException`` 的形式抛出。IO 一章介绍过 ``try``\ ，这里补充 ``System.IO.Error`` 提供的判断函数，用来区分具体原因：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}
   module Main (main) where

   import Control.Exception (IOException, try)
   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO
   import System.IO.Error (isDoesNotExistError, isPermissionError)

   readConfig :: FilePath -> IO T.Text
   readConfig path = do
     result <- try (TIO.readFile path)
     case result of
       Right content -> return content
       Left e
         | isDoesNotExistError e -> do
             putStrLn (path ++ " 不存在，使用默认配置")
             return "port = 8080"
         | isPermissionError e -> fail ("没有权限读取 " ++ path)
         | otherwise -> fail ("读取失败: " ++ show (e :: IOException))

   main :: IO ()
   main = readConfig "app.conf" >>= TIO.putStrLn

.. code:: text

   app.conf 不存在，使用默认配置
   port = 8080

``try`` 的返回类型 ``Either IOException a`` 需要能被推断出来。这里通过 ``show (e :: IOException)`` 标注了一次，也可以直接给 ``result`` 加类型签名。

对于"文件不存在"这一种情况，先用 ``doesFileExist`` 检查也可以。但检查和读取之间文件可能被别的进程删除，所以更可靠的做法还是直接读取并捕获异常。

完整示例：统计目录下的 Haskell 源码行数
--------------------------------------------------------------------------------

把本章的内容组合起来：递归遍历一个目录，找出所有 ``.hs`` 文件，打印每个文件的行数和总行数。

.. code:: haskell

   module Main (main) where

   import Control.Monad (forM)
   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO
   import System.Directory (doesDirectoryExist, listDirectory)
   import System.Environment (getArgs)
   import System.FilePath (takeExtension, (</>))

   -- 递归收集目录下所有文件的路径
   walk :: FilePath -> IO [FilePath]
   walk dir = do
     names <- listDirectory dir
     paths <- forM names $ \name -> do
       let path = dir </> name
       isDir <- doesDirectoryExist path
       if isDir then walk path else return [path]
     return (concat paths)

   main :: IO ()
   main = do
     args <- getArgs
     let root = case args of
           (dir : _) -> dir
           [] -> "."
     files <- walk root
     let hsFiles = filter ((== ".hs") . takeExtension) files
     counts <- forM hsFiles $ \path -> do
       content <- TIO.readFile path
       let n = length (T.lines content)
       putStrLn (show n ++ "\t" ++ path)
       return n
     putStrLn ("总计 " ++ show (sum counts) ++ " 行，" ++ show (length hsFiles) ++ " 个文件")

``walk`` 是本章最值得看的函数。\ ``listDirectory`` 只返回条目名，所以每一项都要用 ``</>`` 拼上父目录。\ ``forM`` 对每个条目执行一个 IO 动作并收集结果，遇到子目录就递归调用自己，最后用 ``concat`` 把嵌套的列表摊平。

放在一个 Cabal 项目里时，把 ``directory``\ 、\ ``filepath``\ 、\ ``text`` 加进 ``build-depends``\ ，然后用 ``cabal run`` 执行；单文件脚本用 ``runghc`` 即可，前提是这几个包已经在全局环境中可用。

.. code:: sh

   $ runghc CountLines.hs sample
   4	sample/src/Sub/Util.hs
   3	sample/src/Main.hs
   总计 7 行，2 个文件

小结
--------------------------------------------------------------------------------

- **三组读写函数**\ ：\ ``Prelude``\ 、\ ``Data.Text.IO``\ 、\ ``Data.ByteString`` 各有一套 ``readFile`` / ``writeFile`` / ``appendFile``\ 。文本文件默认用 ``Data.Text.IO``\ ，它是严格读取，可以安全地读后写回同一文件。
- **编码**\ ：文本读写跟随 locale，需要固定 UTF-8 时用 ``setLocaleEncoding utf8`` 或 ``hSetEncoding``\ 。
- **按行处理**\ ：\ ``T.lines`` 拆分，处理后 ``T.unlines`` 拼回。
- **句柄**\ ：大文件用 ``withFile`` 配合 ``hIsEOF`` / ``hGetLine`` 逐行读取。提示符后记得 ``hFlush stdout``\ 。
- **目录与路径**\ ：文件系统操作在 ``System.Directory``\ ，路径运算在 ``System.FilePath``\ ，拼接路径用 ``</>``\ 。
- **异常**\ ：用 ``try`` 捕获 ``IOException``\ ，用 ``isDoesNotExistError`` 等函数区分原因。
