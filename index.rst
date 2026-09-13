.. a-byte-of-haskell documentation master file

================================================================================
简明 Haskell 教程
================================================================================

欢迎阅读《简明 Haskell 教程》。这是一本专为希望深入理解函数式编程范式的开发者编写的系统指南。

本书立足于严格的函数式思维与现代 Haskell 语言特性，不仅深入剖析数学原理与类型系统，更详尽覆盖了实战开发中必不可少的常用数据容器（List、Map、Set、Vector、Text 与 ByteString）以及进阶单子体系。

.. toctree::
   :maxdepth: 2
   :caption: 模块一：函数式基石与环境

   01-lambda
   02-environment
   03-basics

.. toctree::
   :maxdepth: 2
   :caption: 模块二：类型系统核心

   04-types
   05-typeclasses
   06-patterns
   07-recursion

.. toctree::
   :maxdepth: 2
   :caption: 模块三：常用数据结构与集合容器

   08-lists
   09-folds
   10-collections
   11-adts
   12-error-handling

.. toctree::
   :maxdepth: 2
   :caption: 模块四：抽象代数与函子单子体系

   13-monoid
   14-functor
   15-applicative
   16-monad
   17-foldable-traversable

.. toctree::
   :maxdepth: 2
   :caption: 模块五：深入计算与工程实践

   18-reader-state
   19-monad-transformers
   20-non-strictness
   21-io
   22-concurrency
   23-modules-cabal
