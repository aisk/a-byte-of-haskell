.. a-byte-of-haskell documentation master file

================================================================================
简明 Haskell 教程
================================================================================

这是一本面向有编程经验的读者的 Haskell 入门教程。

全书从 Lambda 演算和类型系统讲起，覆盖常用的数据结构（List、Map、Set、Vector、Text 与 ByteString），再进入 Monoid、Functor、Applicative、Monad 等抽象，然后讨论惰性求值、IO、并发和项目组织，最后用一组常用类库（文件与目录、命令行参数、JSON、HTTP、Socket）演示如何写出日常可用的程序。各章末尾都有小结，多数章节还附有面向其他语言使用者的类比说明。

.. toctree::
   :maxdepth: 2
   :caption: 第一部分：函数式基石与环境

   01-lambda
   02-environment
   03-basics

.. toctree::
   :maxdepth: 2
   :caption: 第二部分：类型系统核心

   04-types
   05-typeclasses
   06-patterns
   07-recursion

.. toctree::
   :maxdepth: 2
   :caption: 第三部分：数据结构与数据建模

   08-lists
   09-strings
   10-folds
   11-collections
   12-adts
   13-error-handling

.. toctree::
   :maxdepth: 2
   :caption: 第四部分：抽象代数与函子单子体系

   14-monoid
   15-functor
   16-applicative
   17-monad
   18-foldable-traversable

.. toctree::
   :maxdepth: 2
   :caption: 第五部分：深入计算与工程实践

   19-reader-state
   20-monad-transformers
   21-non-strictness
   22-io
   23-concurrency
   24-modules-cabal

.. toctree::
   :maxdepth: 2
   :caption: 第六部分：常用类库与实战

   25-files-and-directories
   26-command-line
   27-json-aeson
   28-http-request
   29-network
