模块系统与项目组织
================================================================================

项目超过几千行之后，单靠零散的源文件就不够用了。需要清晰的\ **模块边界**\ 、\ **信息隐藏与封装**\ ，以及标准化的\ **包构建与依赖管理**\ 。

本章介绍 Haskell 的模块（Module）系统、封装的常见做法，以及用 **Cabal** 组织项目的方式。

模块声明与导出控制
--------------------------------------------------------------------------------

每个源文件定义一个模块，模块名必须与文件路径对应（例如模块 ``Data.Graph.Tree`` 保存在 ``Data/Graph/Tree.hs``\ ）。

显式导出列表
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``src/MyLib/Calculator.hs`` 中：

.. code:: haskell

   module MyLib.Calculator
     ( -- 1. 导出具体函数
       addNumbers
     , calculateTotal
       -- 2. 导出数据类型及其全部构造器
     , CalculationMode(..)
       -- 3. 只导出类型名，隐藏构造器
       --    （这实现的是抽象数据类型 Abstract Data Type，
       --      注意与代数数据类型 Algebraic Data Type 不是一回事）
     , SecretToken
     , createToken
     , readToken
     ) where

   -- 数据类型：外部可以直接模式匹配
   data CalculationMode = Fast | Precise deriving (Show, Eq)

   -- 隐藏构造器，外部只能通过智能构造函数创建
   newtype SecretToken = SecretToken String deriving (Show, Eq)

   createToken :: String -> Maybe SecretToken
   createToken str
     | length str >= 8 = Just (SecretToken str)
     | otherwise       = Nothing

   readToken :: SecretToken -> String
   readToken (SecretToken s) = s

   addNumbers :: Int -> Int -> Int
   addNumbers x y = x + y

   calculateTotal :: [Int] -> Int
   calculateTotal = sum

智能构造函数（Smart Constructor）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

注意 ``SecretToken`` 的导出方式：导出列表里写的是 ``SecretToken``\ ，而不是 ``SecretToken(..)``\ 。
这意味着：

- 外部可以在类型签名里使用 ``SecretToken``\ ；
- 但外部\ **不能直接调用构造器创建值，也不能通过模式匹配查看内部实现**\ ；
- 外部必须通过 ``createToken`` 这个带校验的函数（智能构造函数）来创建值。这样只要拿到一个 ``SecretToken``\ ，就可以确定它满足长度要求。

模块导入语法
--------------------------------------------------------------------------------

在使用方，Haskell 提供了几种导入方式：

1. **全量导入**\ ：

   .. code:: haskell

      import MyLib.Calculator

   把该模块导出的全部名字导入当前命名空间。项目变大后容易出现名字冲突。

2. **显式选择导入（推荐）**\ ：

   .. code:: haskell

      import MyLib.Calculator (addNumbers, CalculationMode(..))

   只导入列出的名字，读代码时一眼能看出依赖了外部的哪些符号。

3. **限定导入（Qualified Import）**\ ：

   .. code:: haskell

      import qualified Data.Map.Strict as Map
      import qualified Data.Text as T

   调用时必须带前缀（如 ``Map.lookup``\ 、\ ``T.pack``\ ），避免与 Prelude 的 ``filter``\ 、\ ``lookup`` 等同名函数冲突。

4. **排除特定名字（hiding）**\ ：

   .. code:: haskell

      import Prelude hiding (head, id)

   导入 Prelude 的全部内容，但屏蔽指定的名字。

用 Cabal 组织项目
--------------------------------------------------------------------------------

**Cabal** 是 Haskell 官方推荐的构建工具与包管理器。另一个常见选择是 Stack，它在 Cabal 的包格式之上加了版本快照管理，两者的 ``.cabal`` 文件是通用的。

目录结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一个典型的 Cabal 项目目录：

.. code:: text

   my-awesome-app/
   ├── cabal.project         # 多包工作区配置
   ├── my-awesome-app.cabal  # 包描述文件
   ├── LICENSE
   ├── README.md
   ├── app/                  # 可执行程序入口
   │   └── Main.hs
   ├── src/                  # 库源码
   │   └── MyLib/
   │       └── Calculator.hs
   └── test/                 # 测试
       └── Spec.hs

这三个目录对应 ``.cabal`` 文件中的三个组件，它们之间的依赖关系如下：

.. mermaid::

   graph TD
     EXE["executable my-awesome-app<br/>app/Main.hs"] --> LIB["library<br/>src/MyLib/Calculator.hs"]
     TEST["test-suite my-awesome-app-test<br/>test/Spec.hs"] --> LIB
     LIB --> BASE["base"]
     LIB --> TEXT["text"]
     LIB --> CONT["containers"]
     LIB --> OTHER["async、stm 等外部包"]

业务逻辑放在 ``library`` 里，可执行程序和测试都只是它的使用方。这样测试可以直接导入库模块，不需要经过 ``main``\ 。

.cabal 文件结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``my-awesome-app.cabal`` 中：

.. code:: text

   cabal-version:      3.0
   name:               my-awesome-app
   version:            0.1.0.0
   synopsis:           Haskell 应用示例
   license:            BSD-3-Clause
   build-type:         Simple

   -- 提取公共编译配置，避免在各个组件中重复书写
   common common-settings
       -- GHC2021 需要 GHC 9.2 以上，默认开启 BangPatterns 等常用扩展
       default-language: GHC2021
       ghc-options:      -Wall
                         -Wcompat
                         -Wincomplete-record-updates
                         -Wincomplete-uni-patterns
                         -Wredundant-constraints
                         -O2
       build-depends:    base >= 4.14 && < 5
                       , text >= 2.0
                       , containers >= 0.6
                       , async >= 2.2
                       , stm >= 2.5

   -- 1. 库组件
   library
       import:           common-settings
       hs-source-dirs:   src
       exposed-modules:  MyLib.Calculator
       -- 内部模块写在 other-modules 里，不对外暴露

   -- 2. 可执行程序组件
   executable my-awesome-app
       import:           common-settings
       hs-source-dirs:   app
       main-is:          Main.hs
       -- 开启多线程运行时
       ghc-options:      -threaded -rtsopts -with-rtsopts="-N"
       build-depends:    my-awesome-app

   -- 3. 测试套件
   test-suite my-awesome-app-test
       import:           common-settings
       type:             exitcode-stdio-1.0
       hs-source-dirs:   test
       main-is:          Spec.hs
       build-depends:    my-awesome-app

常用命令
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **下载依赖并编译**\ ：

  .. code:: sh

     $ cabal build

- **运行可执行程序**\ ：

  .. code:: sh

     $ cabal run my-awesome-app

- **运行测试**\ ：

  .. code:: sh

     $ cabal test --test-show-details=always

- **启动带项目依赖的 REPL**\ ：

  .. code:: sh

     $ cabal repl my-awesome-app

- **清理构建产物**\ ：

  .. code:: sh

     $ cabal clean

小结与全书总结
--------------------------------------------------------------------------------

- **导出列表与信息隐藏**\ ：隐藏构造器并提供智能构造函数，可以保证值在创建时就满足约束。
- **导入方式**\ ：优先使用显式列表导入或 ``qualified as`` 限定导入。
- **Cabal**\ ：用 ``common`` 块复用配置，用 ``library`` 与 ``executable`` 分离库和入口，按需开启 ``-threaded``\ 。

到这里全书结束。我们从 Lambda 演算和类型系统开始，经过常用数据结构、Monoid / Functor / Applicative / Monad 这一组抽象，最后讨论了惰性求值、IO、并发和项目组织。这些内容覆盖了日常使用 Haskell 需要的大部分基础，更深入的主题（类型级编程、依赖注入的其他方案、各种效果系统）可以在此之上继续阅读。
