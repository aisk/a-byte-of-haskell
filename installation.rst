安装
====

在正式安装前，我们需要知道，正如 Python 有最早的 Guido 本人亲自的实现（CPython），以及性能更好的 Pypy，以及基于 Java 平台的 Jython 等不同实现，Haskell 本身也有多个实现。

不过放在今天，Haskell 最完善，特性完整，并且使用最广泛的实现就是 Glasgow Haskell Compiler（后简称 GHC）这一实现。本书以此实现进行编写，并且后续如不特指，均默认以 GHC 实现为准。

在 Arch Linux 中安装
--------------------
如果你是 Arch Linux 发行版用户，只需要执行

.. code:: sh

   $ sudo pacman -S ghc ghc-static

即可。

在 macOS 中安装
---------------
如果你使用的是 macOS 操作系统，并且幸运的安装了 Homebrew 作为包管理系统，那你秩序执行

.. code:: sh

   $ brew install ghc

就可以顺利安装。
