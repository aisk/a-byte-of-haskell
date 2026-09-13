开发环境与 GHCi 交互
================================================================================

要开始 Haskell 编程旅程，我们需要一套高效可靠的编译器与交互式工具链。现代 Haskell 事实上的工业与学术标准是 **GHC** （Glasgow Haskell Compiler）。

安装 Haskell 工具链
--------------------------------------------------------------------------------

现代官方标准安装工具是 **GHCup**\ ，它支持在 Linux、macOS 以及 Windows (WSL / PowerShell) 上一键管理 GHC、构建工具 Cabal 和语言服务器 HLS。

使用 GHCup 安装（推荐）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在类 Unix（Linux / macOS）终端中执行：

.. code:: sh

   curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh

按照屏幕指引即可完成 GHC、Cabal 与 Stack 的安装与环境变量配置。

使用 Linux 发行版包管理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

安装完成后，在终端验证：

.. code:: sh

   $ ghc --version
   The Glorious Glasgow Haskell Compilation System, version 9.x.x

使用 GHCi 交互式解释器
--------------------------------------------------------------------------------

GHC 自带了一个功能强大的交互式环境（REPL），名为 **GHCi**\ 。在终端中输入 ``ghci`` 即可启动：

.. code:: text

   $ ghci
   GHCi, version 9.6.3: https://www.haskell.org/ghc/  :? for help
   ghci>

你可以直接把 GHCi 当作高精度计算器或试验场：

.. code:: text

   ghci> 2 + 3 * 4
   14
   ghci> putStrLn "Hello, Haskell!"
   Hello, Haskell!

核心常用命令
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

GHCi 提供了以冒号 ``:`` 开头的一系列内置管理命令：

- **退出**\ ：\ ``:quit`` 或简写 ``:q``\ （在 Linux/macOS 下也可以按 ``Ctrl + D``\ ）。
- **查询类型**\ ：\ ``:type`` 或简写 ``:t``\ ，打印表达式或函数的类型签名。
  
  .. code:: text

     ghci> :t "Hello"
     "Hello" :: [Char]
     ghci> :t (+)
     (+) :: Num a => a -> a -> a

- **查询详细信息**\ ：\ ``:info`` 或简写 ``:i``\ ，查看某个类型、函数或类型类的详细定义、所在模块以及实现的实例列表。
- **加载文件**\ ：\ ``:load <文件名>`` 或简写 ``:l <文件名>``\ 。
- **重新加载**\ ：\ ``:reload`` 或简写 ``:r``\ ，快速重载最后一次修改的文件。
- **查看帮助**\ ：\ ``:?`` 列出所有内置命令。

编写与编译独立程序
--------------------------------------------------------------------------------

除了在 GHCi 中交互试运行，我们还可以将代码编译为操作系统原生的二进制可执行文件。

1. 新建源文件 ``Main.hs``\ ：

.. code:: haskell

   module Main where

   main :: IO ()
   main = putStrLn "Hello, World from compiled Haskell!"

2. 使用 ``ghc`` 进行编译：

.. code:: sh

   $ ghc Main.hs -o myprogram

编译完成后会生成目标文件 ``.o``\ 、接口文件 ``.hi`` 以及可执行二进制文件 ``myprogram``\ 。

3. 执行二进制程序：

.. code:: sh

   $ ./myprogram
   Hello, World from compiled Haskell!

小结
--------------------------------------------------------------------------------

- **GHC** 是 Haskell 的核心编译器，\ **GHCi** 是用于快速实验与探究类型的交互式环境。
- 在日常开发中，善用 ``:t`` 和 ``:i`` 查验类型与定义是掌握 Haskell 的关键直觉。
- 独立程序的入口约定为 ``Main`` 模块下的 ``main`` 函数，其类型通常为 ``IO ()``\ 。
