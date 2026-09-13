模块系统与工程组织实践
================================================================================

当编写超过数千行的大型真实项目时，单靠零散的代码文件将难以为继。现代软件工程要求代码具备清晰的**模块边界**、严密的**信息隐藏与抽象封装**，以及标准化的**包构建与依赖管理体系**。

本章将系统解析 Haskell 的模块（Module）系统、封装设计范式，以及现代工业界唯一的官方构建标准——\ **Cabal 3.x** 的工程化最佳实践。

模块声明与显式封装控制
--------------------------------------------------------------------------------

在 Haskell 中，每个源文件通常定义一个独立的模块，模块名必须与文件系统的路径严格保持一致（例如模块 ``Data.Graph.Tree`` 必须保存在路径 ``Data/Graph/Tree.hs`` 下）。

显式导出白名单（Export List）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``src/MyLib/Calculator.hs`` 中：

.. code:: haskell

   module MyLib.Calculator
     ( -- 1. 显式导出具体函数
       addNumbers
     , calculateTotal
       -- 2. 导出数据类型及其全部数据构造器
     , CalculationMode(..)
       -- 3. 仅导出类型名称，隐藏内部构造器（实现抽象数据类型 ADT）
     , SecretToken
     , createToken
     , readToken
     ) where

   -- 数据类型：外部可以直接模式匹配
   data CalculationMode = Fast | Precise deriving (Show, Eq)

   -- 抽象封装：隐藏内部构造器，强制外部通过智能构造函数创建
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

封装核心原则：智能构造函数（Smart Constructor）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

注意上述的 ``SecretToken`` 导出规范：在导出列表中仅写明了 ``SecretToken``\ ，而没有写 ``SecretToken(..)``\ 。
这意味着：

- 外部调用者可以使用 ``SecretToken`` 这个类型标记；
- 但外部代码\ **无法直接使用数据构造器创建无效对象，也无法通过模式匹配直接窥探其内部实现**\ ；
- 外部必须通过 ``createToken`` 这个验证函数（智能构造函数）来实例化。通过类型系统的导出控制，我们在编译期就彻底消灭了非法状态的存在可能！

模块导入语法规范
--------------------------------------------------------------------------------

在消费者模块中，Haskell 提供了严谨多样的导入控制语法：

1. **全量导入（谨慎在生产中使用）**\ ：

   .. code:: haskell

      import MyLib.Calculator

   将该模块导出的全部公开函数与类型导入当前命名空间。在大型工程中极易引发名称冲突。

2. **显式选择导入（推荐规范）**\ ：

   .. code:: haskell

      import MyLib.Calculator (addNumbers, CalculationMode(..))

   白名单式导入，极大提升代码可读性，一眼就能看清当前模块依赖了外部的哪些符号。

3. **带前缀限定导入（Qualified Import，高频最佳实践）**\ ：

   .. code:: haskell

      import qualified Data.Map.Strict as Map
      import qualified Data.Text as T

   在调用时必须附带模块前缀（如 ``Map.lookup``\ 、\ ``T.pack``\ ），彻底解决不同库中函数同名（如与 Prelude 的 ``filter``\ 、\ ``lookup``\ ）冲突的痛点。

4. **排除特定冲突项导入（Hiding）**\ ：

   .. code:: haskell

      import Prelude hiding (head, id)

   导入 Prelude 的全部函数，但显式屏蔽可能引发歧义或不安全的符号。

现代 Cabal 3.x 工业级工程实践
--------------------------------------------------------------------------------

在现代 Haskell 生态中，**Cabal** 是官方且最主流的构建工具与包管理器。

标准工程目录结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一个标准、规范的现代 Cabal 项目目录如下：

.. code:: text

   my-awesome-app/
   ├── cabal.project         # 多包协同工作区配置
   ├── my-awesome-app.cabal  # 核心包描述文件
   ├── LICENSE
   ├── README.md
   ├── app/                  # 可执行程序入口
   │   └── Main.hs
   ├── src/                  # 核心业务库源码
   │   └── MyLib/
   │       └── Calculator.hs
   └── test/                 # 自动化测试套件
       └── Spec.hs

.cabal 配置文件规范结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 ``my-awesome-app.cabal`` 中：

.. code:: text

   cabal-version:      3.0
   name:               my-awesome-app
   version:            0.1.0.0
   synopsis:           现代 Haskell 企业级应用范例
   license:            BSD-3-Clause
   build-type:         Simple

   -- 提取公共编译配置，避免在各个 target 中重复书写
   common common-settings
       default-language: Haskell2010
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

   -- 1. 核心库组件
   library
       import:           common-settings
       hs-source-dirs:   src
       exposed-modules:  MyLib.Calculator
       other-modules:    -- 内部私有模块（不暴露给外部使用者）

   -- 2. 可执行程序组件
   executable my-awesome-app
       import:           common-settings
       hs-source-dirs:   app
       main-is:          Main.hs
       -- 开启多线程异步运行时支持
       ghc-options:      -threaded -rtsopts -with-rtsopts="-N"
       build-depends:    my-awesome-app

   -- 3. 自动化测试套件
   test-suite my-awesome-app-test
       import:           common-settings
       type:             exitcode-stdio-1.0
       hs-source-dirs:   test
       main-is:          Spec.hs
       build-depends:    my-awesome-app

核心构建与开发命令速查
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **下载依赖并编译**\ ：

  .. code:: sh

     $ cabal build

- **直接运行可执行程序**\ ：

  .. code:: sh

     $ cabal run my-awesome-app

- **运行全量测试套件**\ ：

  .. code:: sh

     $ cabal test --test-show-details=always

- **启动绑定当前工程完整依赖的交互式 REPL**\ ：

  .. code:: sh

     $ cabal repl my-awesome-app

- **清理编译构建缓存**\ ：

  .. code:: sh

     $ cabal clean

小结与全书总结
--------------------------------------------------------------------------------

- **模块导出白名单与信息隐藏**\ ：通过隐藏数据构造器并提供智能构造函数，确保领域对象在创建之初便满足所有业务不变量。
- **严谨的限定导入**\ ：始终推荐使用显式列表导入或 ``qualified as`` 前缀限定导入，保障工程长期可维护性。
- **现代化 Cabal 3.x 规范**\ ：通过 ``common`` 块复用配置，通过 ``library`` 与 ``executable`` 隔离核心业务与应用入口，并通过 ``-threaded`` 充分释放多核并发算力。
- **全书脉络串联**\ ：
  回顾这趟 Haskell 探索之旅，我们从最纯粹的 **Lambda 演算** 与 **代数类型系统** 起步，遍历了高效的 **函数式数据结构**（List、Map、Set、Vector、Text 与 ByteString）；随后跨越了优雅的抽象代数之桥——**Semigroup（可合并）**、**Monoid（可拼接有空值）**、**Functor（单盒子纯变换）**、**Applicative（多独立盒子装配）** 与 **Monad（因果链式流水线）**；最后深入到 **惰性求值控制**、**纯函数式 IO 副作用模型** 以及 **现代轻量级 M:N 异步并发与工程架构**。
  
  Haskell 不仅是一门功能强大的工业级编程语言，更是一套重塑思维的强大心智武器。愿你在未来的函数式编程实践中，尽情体会这种严谨、优雅与高效的艺术！
