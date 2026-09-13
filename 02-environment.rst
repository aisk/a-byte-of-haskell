开发环境与 GHCi 交互
================================================================================

要开始 Haskell 编程旅程，我们需要一套高效可靠的编译器与交互式工具链。现代 Haskell 事实上的工业与学术标准是 **GHC** （Glasgow Haskell Compiler）。

安装 Haskell 工具链
--------------------------------------------------------------------------------

现代官方标准安装工具是 **GHCup**\ ，它支持在 Linux、macOS 以及 Windows (WSL / PowerShell) 上统一管理 GHC、构建工具 Cabal、Stack 和语言服务器 HLS（Haskell Language Server）。

使用 GHCup 安装（推荐）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在类 Unix（Linux / macOS）终端中执行：

.. code:: sh

   curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh

按照屏幕交互指引即可完成 GHC、Cabal、Stack 与 HLS 的安装及环境变量注入。

使用 Linux 发行版包管理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

若使用 Arch Linux：

.. code:: sh

   sudo pacman -S ghc cabal-install

若使用 Ubuntu / Debian：

.. code:: sh

   sudo apt update
   sudo apt install ghc cabal-install

若使用 macOS（Homebrew）：

.. code:: sh

   brew install ghc cabal-install

安装完成后，在终端验证工具链版本：

.. code:: sh

   $ ghc --version
   The Glorious Glasgow Haskell Compilation System, version 9.10.x
   $ cabal --version
   cabal-install version 3.12.x

使用 GHCi 交互式解释器
--------------------------------------------------------------------------------

GHC 自带了一个功能强大的交互式环境（REPL），名为 **GHCi**\ 。在终端中输入 ``ghci`` 即可启动：

.. code:: text

   $ ghci
   GHCi, version 9.10.3: https://www.haskell.org/ghc/  :? for help
   ghci>

你可以直接把 GHCi 当作高精度计算器或试验场：

.. code:: text

   ghci> 2 + 3 * 4
   14
   ghci> putStrLn "Hello, Haskell!"
   Hello, Haskell!

核心常用管理命令
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

GHCi 提供了以冒号 ``:`` 开头的一系列内置管理命令：

- **退出**\ ：\ ``:quit`` 或简写 ``:q``\ （在 Linux/macOS 下也可以直接按 ``Ctrl + D``\ ）。
- **查询类型**\ ：\ ``:type`` 或简写 ``:t``\ ，打印表达式或函数的类型签名。

  .. code:: text

     ghci> :t "Hello"
     "Hello" :: [Char]
     ghci> :t (+)
     (+) :: Num a => a -> a -> a

- **查询详细定义**\ ：\ ``:info`` 或简写 ``:i``\ ，查看某个类型、函数或类型类的声明定义、定义所在源码位置以及实现的实例列表。
- **查询类型阶数（Kind）**\ ：\ ``:kind`` 或简写 ``:k``\ ，查看类型构造器的形态（例如 ``:k Maybe`` 返回 ``* -> *``\ ）。
- **加载与重载文件**\ ：
  - ``:load <文件名>`` 或简写 ``:l <文件名>``\ ：加载当前目录下的源文件。
  - ``:reload`` 或简写 ``:r``\ ：重新编译并载入最后一次修改的文件。
- **模块浏览**\ ：\ ``:browse <模块名>`` 查看指定模块导出的所有函数与类型列表。
- **交互式文档检索**\ ：\ ``:doc <标识符>`` （GHC 9.x+ 特性），在终端直接阅读指定函数或类型的 Haddock 文档。
- **查看内置帮助**\ ：\ ``:?`` 列出所有内置命令速查表。

进阶 REPL 实用技巧
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. 自动打印类型与资源消耗（:set +t 与 :set +s）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 GHCi 中开启这两项配置，能极大提升学习与调试效率：

.. code:: text

   ghci> :set +t
   ghci> 10 + 20
   30
   it :: Num a => a

   ghci> :set +s
   ghci> sum [1..1000000]
   500000500000
   it :: (Num a, Enum a) => a
   (0.04 secs, 80,064,288 bytes)

``:set +t`` 会在每次求值后自动输出结果的类型；\ ``:set +s`` 会输出执行耗时与内存分配量。

2. 开启语言扩展（:set -X...）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

无需修改文件，可在 GHCi 中动态启用 GHC 语言扩展：

.. code:: text

   ghci> :set -XOverloadedStrings
   ghci> :set -XBinaryLiterals
   ghci> 0b101010
   42

3. 多行代码输入块（:{ 与 :}）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当在 REPL 中定义包含较多分支的函数或复杂代数数据类型时，单行输入极不方便。使用 ``:{`` 与 ``:}`` 可进入多行编辑模式：

.. code:: text

   ghci> :{
   ghci| data Shape
   ghci|   = Circle Double
   ghci|   | Rectangle Double Double
   ghci|   deriving (Show)
   ghci| :}
   ghci> Circle 3.14
   Circle 3.14

4. 自定义 .ghci 配置文件
^^^^^^^^^^^^^^^^^^^^^^^^

在用户主目录下的 ``~/.ghci`` 文件中可以写入常用的启动默认配置，避免每次手动设置：

.. code:: text

   :set prompt "\ESC[34mλ> \ESC[m"
   :set +t
   :set -XOverloadedStrings

编写与运行独立程序
--------------------------------------------------------------------------------

除了在 GHCi 中交互试运行，日常开发还常使用脚本化执行与原生编译：

免编译直接执行脚本：runghc / runhaskell
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于轻量级脚本或一次性任务，无需生成编译产物：

1. 新建 ``Script.hs``\ ：

.. code:: haskell

   module Main where

   main :: IO ()
   main = putStrLn "Running Haskell script directly!"

2. 使用 ``runghc`` 快速运行：

.. code:: sh

   $ runghc Script.hs
   Running Haskell script directly!

使用 GHC 编译原生二进制可执行文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. 新建源文件 ``Main.hs``\ ：

.. code:: haskell

   module Main where

   main :: IO ()
   main = putStrLn "Hello, World from compiled Haskell!"

2. 使用 ``ghc`` 进行优化编译：

.. code:: sh

   $ ghc -O2 Main.hs -o myprogram

编译完成后会生成目标文件 ``.o``\ 、接口文件 ``.hi`` 以及可执行二进制文件 ``myprogram``\ 。参数 ``-O2`` 用于开启 GHC 深度激进优化。

3. 执行二进制程序：

.. code:: sh

   $ ./myprogram
   Hello, World from compiled Haskell!

小结
--------------------------------------------------------------------------------

- **GHC** 是现代 Haskell 的核心编译器，\ **GHCi** 是探索类型系统与算法验证的最佳试验场。
- 善用 ``:t``\ 、\ ``:i``\ 、\ ``:k`` 与 ``:doc``\ ，能在编写代码时迅速洞察任何抽象概念。
- 使用 ``:set +t`` 与 ``:set +s`` 可辅助观察类型与计算性能表现。
- 独立程序的标准入口为 ``Main`` 模块下的 ``main :: IO ()``\ ，可通过 ``runghc`` 脚本化运行或 ``ghc -O2`` 编译为高吞吐的原生机器码。
