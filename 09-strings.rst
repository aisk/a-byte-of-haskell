字符串与文本处理
================================================================================

上一章讲的是列表。在 Haskell 中 ``String`` 就是 ``[Char]``\ ，所以列表上的函数都可以直接用在字符串上。但字符串处理还有一些专门的问题：字符分类、大小写、拆分与拼接、和数字互转、以及 ``String`` 在性能上的不足。本章按这些主题介绍标准库里的常用函数，后半部分介绍实际项目中更常用的 ``Text``\ 。

字符串字面量与 Unicode
--------------------------------------------------------------------------------

``Char`` 是一个 Unicode 码点，\ ``String`` 是 ``Char`` 的链表。字面量支持常见的转义序列：

.. code:: text

   ghci> "tab\there"
   "tab\there"
   ghci> "quote: \"hi\""
   "quote: \"hi\""
   ghci> "\x41\66\955"          -- 十六进制、十进制码点，以及 λ
   "AB\955"
   ghci> length "\955"
   1

很长的字面量可以用反斜杠拆成多行，两个反斜杠之间的空白会被忽略：

.. code:: haskell

   banner :: String
   banner = "第一行内容，\
            \直接接在后面"

GHCi 的显示与 putStrLn 的区别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

初学者常被这个现象困扰：在 GHCi 里直接输入中文字符串，显示出来的是一串数字。

.. code:: text

   ghci> "你好"
   "\20320\22909"
   ghci> putStrLn "你好"
   你好

GHCi 显示结果时调用的是 ``show``\ ，它会把非 ASCII 字符转义成十进制码点，目的是让输出可以原样粘贴回源码。字符串本身没有问题，用 ``putStrLn`` 输出就能看到原文。

Data.Char：字符分类与转换
--------------------------------------------------------------------------------

``Data.Char`` 提供单个字符层面的操作：

.. code:: haskell

   import Data.Char

- 分类判断：\ ``isDigit``\ 、\ ``isAlpha``\ 、\ ``isAlphaNum``\ 、\ ``isSpace``\ 、\ ``isUpper``\ 、\ ``isLower``\ 、\ ``isPunctuation``\ 。
- 大小写：\ ``toUpper``\ 、\ ``toLower``\ 。
- 码点互转：\ ``ord :: Char -> Int``\ 、\ ``chr :: Int -> Char``\ 。
- 数字字符互转：\ ``digitToInt``\ （支持十六进制字符）、\ ``intToDigit``\ 。

.. code:: text

   ghci> map toUpper "haskell"
   "HASKELL"
   ghci> filter isDigit "a1b2c3"
   "123"
   ghci> ord 'A'
   65
   ghci> chr 955
   '\955'
   ghci> digitToInt 'f'
   15

一个常见的小函数，首字母大写：

.. code:: haskell

   capitalize :: String -> String
   capitalize []       = []
   capitalize (c : cs) = toUpper c : cs

.. note::

   ``toUpper`` 只做单个字符到单个字符的映射。像德语 ``'ß'`` 这样大写形式是两个字符的情况它无法处理（\ ``toUpper 'ß'`` 仍然是 ``'ß'``\ ）。需要完整的 Unicode 大小写规则时用后面介绍的 ``Data.Text.toUpper``\ 。

把字符串当作列表来处理
--------------------------------------------------------------------------------

列表上的函数对字符串都适用：

.. code:: text

   ghci> length "Haskell"
   7
   ghci> reverse "stressed"
   "desserts"
   ghci> 'k' `elem` "Haskell"
   True
   ghci> replicate 3 "ab"
   ["ab","ab","ab"]
   ghci> concat (replicate 3 "ab")
   "ababab"
   ghci> zip "abc" [1 ..]
   [('a',1),('b',2),('c',3)]

组合这些函数就能写出大部分简单的字符串逻辑。例如判断回文，忽略大小写和非字母字符：

.. code:: haskell

   import Data.Char (isAlpha, toLower)

   isPalindrome :: String -> Bool
   isPalindrome s = cleaned == reverse cleaned
     where
       cleaned = map toLower (filter isAlpha s)

.. code:: text

   ghci> isPalindrome "A man, a plan, a canal: Panama"
   True

注意 ``String`` 是链表，\ ``length``\ 、\ ``!!`` 和 ``last`` 都是 O(n)，在循环里反复调用会明显变慢。

拆分与拼接
--------------------------------------------------------------------------------

按空白与换行拆分
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Prelude 提供了两对互逆的函数：

- ``words`` / ``unwords``\ ：按任意空白拆分成单词，再用单个空格拼回。
- ``lines`` / ``unlines``\ ：按换行拆分，再用换行拼回。

.. code:: text

   ghci> words "  hello   world  "
   ["hello","world"]
   ghci> unwords ["a", "b", "c"]
   "a b c"
   ghci> lines "first\nsecond\n"
   ["first","second"]
   ghci> unlines ["first", "second"]
   "first\nsecond\n"

``unlines`` 会在每一行后面都加换行，所以 ``unlines . lines`` 不一定还原原字符串。

前缀、后缀与子串
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

来自 ``Data.List``\ ：

.. code:: text

   ghci> import Data.List
   ghci> "http" `isPrefixOf` "http://example.com"
   True
   ghci> ".hs" `isSuffixOf` "Main.hs"
   True
   ghci> "ask" `isInfixOf` "Haskell"
   True
   ghci> stripPrefix "http://" "http://example.com"
   Just "example.com"
   ghci> stripPrefix "ftp://" "http://example.com"
   Nothing

``stripPrefix`` 返回 ``Maybe``\ ，前缀不匹配时得到 ``Nothing``\ ，比先判断再 ``drop`` 更安全。

去除首尾空白
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

标准库没有直接的 ``trim``\ ，但用 ``dropWhile`` 和 ``dropWhileEnd`` 可以组合出来：

.. code:: haskell

   import Data.Char (isSpace)
   import Data.List (dropWhileEnd)

   trim :: String -> String
   trim = dropWhileEnd isSpace . dropWhile isSpace

.. code:: text

   ghci> trim "   padded   "
   "padded"

按分隔符拆分与用分隔符拼接
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

拼接有现成的 ``intercalate``\ （相当于其他语言的 ``join``\ ）：

.. code:: text

   ghci> intercalate ", " ["apple", "banana", "cherry"]
   "apple, banana, cherry"

拆分则没有内置函数。按单个字符拆分可以用 ``break`` 自己写：

.. code:: haskell

   splitOn :: Char -> String -> [String]
   splitOn sep s = case break (== sep) s of
     (chunk, [])         -> [chunk]
     (chunk, _ : rest)   -> chunk : splitOn sep rest

.. code:: text

   ghci> splitOn ',' "a,b,,c"
   ["a","b","","c"]

需要按子串拆分或更多变体时，可以使用 ``split`` 包里的 ``Data.List.Split``\ ，或者直接换用 ``Text``\ ，它自带 ``splitOn``\ 。

与数字互转
--------------------------------------------------------------------------------

show 与 read
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``show`` 把值转成字符串，\ ``read`` 反过来解析。\ ``read`` 的目标类型由上下文决定，在 GHCi 里通常需要显式标注：

.. code:: text

   ghci> show 42
   "42"
   ghci> show 3.14
   "3.14"
   ghci> read "42" :: Int
   42
   ghci> read "3.14" :: Double
   3.14
   ghci> read "abc" :: Int
   *** Exception: Prelude.read: no parse

``read`` 解析失败会抛异常，处理用户输入时应使用 ``Text.Read`` 里的 ``readMaybe``\ ：

.. code:: text

   ghci> import Text.Read (readMaybe)
   ghci> readMaybe "42" :: Maybe Int
   Just 42
   ghci> readMaybe "42abc" :: Maybe Int
   Nothing

格式化输出：printf 与 Numeric
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Text.Printf`` 提供了 C 风格的 ``printf``\ 。它的返回类型是多态的，既可以直接在 ``IO`` 中打印，也可以标注为 ``String`` 拿到结果：

.. code:: text

   ghci> import Text.Printf
   ghci> printf "%s 今年 %d 岁\n" "Alice" (30 :: Int)
   Alice 今年 30 岁
   ghci> printf "%.2f" (3.14159 :: Double) :: String
   "3.14"
   ghci> printf "%5d|%-5d|" (42 :: Int) (42 :: Int) :: String
   "   42|42   |"

数字字面量要加类型标注，否则 ``printf`` 无法确定参数类型。

``Numeric`` 模块有一组更具体的函数，比如控制小数位数的 ``showFFloat`` 和输出十六进制的 ``showHex``\ 。它们的最后一个参数是要接在结果后面的字符串，通常传空串：

.. code:: text

   ghci> import Numeric
   ghci> showFFloat (Just 2) 3.14159 ""
   "3.14"
   ghci> showHex (255 :: Int) ""
   "ff"

比较与排序
--------------------------------------------------------------------------------

``String`` 的 ``Ord`` 实例是按字符码点的字典序：

.. code:: text

   ghci> "apple" < "banana"
   True
   ghci> "Zebra" < "apple"      -- 大写字母的码点小于小写字母
   True
   ghci> "10" < "9"             -- 按字符比较，不是按数值
   True

排序字符串列表直接用 ``sort``\ 。要按其他标准排序，用 ``Data.List`` 的 ``sortOn`` 和 ``Data.Ord`` 的 ``comparing``\ 、\ ``Down``\ ：

.. code:: text

   ghci> import Data.List (sort, sortOn)
   ghci> import Data.Ord (Down (..))
   ghci> sort ["pear", "fig", "apple"]
   ["apple","fig","pear"]
   ghci> sortOn length ["pear", "fig", "apple"]
   ["fig","pear","apple"]
   ghci> sortOn Down ["pear", "fig", "apple"]
   ["pear","fig","apple"]

Text：实际项目中的字符串类型
--------------------------------------------------------------------------------

``String`` 每个字符占一个链表节点，在 64 位机器上大约 40 字节，而且所有操作都是链表遍历。处理大段文本时应改用 ``text`` 包里的 ``Data.Text``\ ，它把文本存成 UTF-8 编码的连续内存块。

导入约定与 OverloadedStrings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Data.Text`` 里的函数名大量与 Prelude 重复（\ ``length``\ 、\ ``map``\ 、\ ``filter`` 等），惯例是限定导入：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Data.Text (Text)
   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO

``T.pack`` 和 ``T.unpack`` 负责与 ``String`` 互转。开启 ``OverloadedStrings`` 扩展后，双引号字面量可以根据类型推断直接成为 ``Text``\ ，省去到处写 ``T.pack``\ ：

.. code:: haskell

   greeting :: Text
   greeting = "你好"          -- 有了 OverloadedStrings 才能这样写

在 GHCi 里用 ``:set -XOverloadedStrings`` 开启。

常用操作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Data.Text`` 覆盖了本章前面提到的所有操作，而且补上了 ``String`` 缺少的部分：

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - 函数
     - 说明
   * - ``T.length``\ 、\ ``T.null``\ 、\ ``T.reverse``
     - 基本查询。注意 ``T.length`` 是 O(n)，因为 UTF-8 是变长编码
   * - ``T.toUpper``\ 、\ ``T.toLower``
     - 完整的 Unicode 大小写转换，结果长度可能变化
   * - ``T.strip``\ 、\ ``T.stripStart``\ 、\ ``T.stripEnd``
     - 去除首尾空白
   * - ``T.words``\ 、\ ``T.lines``\ 、\ ``T.unwords``\ 、\ ``T.unlines``
     - 与 Prelude 同名函数行为一致
   * - ``T.splitOn``\ 、\ ``T.intercalate``
     - 按子串拆分与拼接，分隔符不能为空
   * - ``T.replace``
     - 替换所有出现的子串
   * - ``T.isPrefixOf``\ 、\ ``T.isSuffixOf``\ 、\ ``T.isInfixOf``
     - 前缀、后缀、子串判断
   * - ``T.stripPrefix``\ 、\ ``T.stripSuffix``
     - 去掉前缀或后缀，返回 ``Maybe``
   * - ``T.breakOn``
     - 在第一次出现某子串的位置切成两半
   * - ``T.take``\ 、\ ``T.drop``\ 、\ ``T.filter``\ 、\ ``T.map``
     - 与列表同名函数对应

.. code:: text

   ghci> :set -XOverloadedStrings
   ghci> import qualified Data.Text as T
   ghci> T.splitOn "," "a,b,c"
   ["a","b","c"]
   ghci> T.intercalate " | " ["x", "y", "z"]
   "x | y | z"
   ghci> T.strip "   padded   "
   "padded"
   ghci> T.replace "l" "L" "hello"
   "heLLo"
   ghci> T.breakOn "=" "key=value"
   ("key","=value")
   ghci> T.toUpper "straße"
   "STRASSE"

拼接大量片段时，优先使用 ``T.concat``\ 、\ ``T.intercalate`` 或 ``T.unlines`` 这类一次性完成的函数，而不是在循环里反复 ``<>``\ ，后者每次都会复制已有内容。需要逐步构建很长文本时，可以用 ``Data.Text.Lazy.Builder``\ ，它把所有片段收集起来最后只分配一次。

Text 与数字
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``read`` 只接受 ``String``\ 。对 ``Text`` 可以先 ``T.unpack`` 再用 ``readMaybe``\ ，或者使用 ``Data.Text.Read`` 里专门的解析函数。后者返回剩余未解析的部分，适合做手写解析：

.. code:: text

   ghci> import Data.Text.Read (decimal, double)
   ghci> decimal "42abc" :: Either String (Int, T.Text)
   Right (42,"abc")
   ghci> decimal "abc" :: Either String (Int, T.Text)
   Left "input does not start with a digit"
   ghci> double "3.14"
   Right (3.14,"")

反方向用 ``T.pack (show n)``\ 。

Text 的输入输出
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``Data.Text.IO`` 提供与 Prelude 对应的 ``putStrLn``\ 、\ ``getLine``\ 、\ ``readFile``\ 、\ ``writeFile``\ ，直接操作 ``Text``\ ，而且 ``readFile`` 是严格读取，没有 Prelude 版本的惰性 I/O 问题（见 IO 一章）：

.. code:: haskell

   countLines :: FilePath -> IO Int
   countLines path = do
     contents <- TIO.readFile path
     return (length (T.lines contents))

.. note::

   ``Data.Text.IO`` 读写文件时按当前系统的 locale 编码转换。在 locale 不是 UTF-8 的环境（某些容器或 Windows 终端）中读 UTF-8 文件会出错。稳妥的做法是用 ``Data.ByteString`` 读原始字节，再用下面的 ``decodeUtf8`` 解码。

ByteString 与编码转换
--------------------------------------------------------------------------------

``Text`` 表示的是字符序列，\ ``ByteString`` 表示的是字节序列。两者之间必须经过明确的编码转换，函数在 ``Data.Text.Encoding`` 中：

.. code:: haskell

   import qualified Data.ByteString as BS
   import Data.Text.Encoding (encodeUtf8, decodeUtf8')

   -- Text -> ByteString，总是成功
   bytes :: BS.ByteString
   bytes = encodeUtf8 "你好"

   -- ByteString -> Text，输入可能不是合法 UTF-8，所以返回 Either
   readUtf8File :: FilePath -> IO (Either String Text)
   readUtf8File path = do
     raw <- BS.readFile path
     return $ case decodeUtf8' raw of
       Left err  -> Left (show err)
       Right txt -> Right txt

``decodeUtf8`` 是遇到非法字节直接抛异常的版本，\ ``decodeUtf8Lenient`` 会把非法字节替换成 U+FFFD。

.. warning::

   ``Data.ByteString.Char8`` 里的 ``pack`` 和 ``unpack`` 只保留每个字符码点的低 8 位。对 ASCII 文本没有问题，但 ``BC.pack "你好"`` 得到的是无意义的两个字节。处理非 ASCII 文本时不要用这个模块做转换，走 ``encodeUtf8`` 和 ``decodeUtf8``\ 。

三种类型的选择
--------------------------------------------------------------------------------

.. list-table::
   :widths: 20 40 40
   :header-rows: 1

   * - 类型
     - 适用场景
     - 说明
   * - ``String``
     - 小段文本、教学示例、与只接受 ``String`` 的旧接口交互
     - 无需额外依赖，可直接模式匹配，但内存和速度都差
   * - ``Text``
     - 所有面向人的文本：用户输入、配置、日志、模板
     - 需要 ``text`` 包，配合 ``OverloadedStrings`` 使用
   * - ``ByteString``
     - 网络协议、文件字节、二进制格式、编码未知的数据
     - 需要 ``bytestring`` 包，与 ``Text`` 之间要显式编解码

小结
--------------------------------------------------------------------------------

- ``String`` 是 ``[Char]``\ ，列表函数全部适用；GHCi 用 ``show`` 显示字符串，非 ASCII 字符会转义，用 ``putStrLn`` 才能看到原文。
- ``Data.Char`` 负责单个字符的分类和转换，\ ``Data.List`` 提供前缀、子串判断和 ``intercalate``\ 。
- 字符串与数字之间用 ``show`` / ``readMaybe`` 互转，格式化输出用 ``printf`` 或 ``Numeric``\ 。
- 实际项目中面向人的文本用 ``Text``\ ，字节数据用 ``ByteString``\ ，两者之间通过 ``encodeUtf8`` / ``decodeUtf8`` 转换。
