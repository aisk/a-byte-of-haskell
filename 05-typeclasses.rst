类型类（Typeclasses）与法则约束
================================================================================

在面向对象语言中，多态通常依赖类继承（Inheritance）或接口（Interface）。而在 Haskell 中，实现“对不同类型提供统一操作契约”的机制是\ **类型类（Typeclasses）**\ ，它对应于计算机科学中的\ **特设多态（Ad-hoc Polymorphism）**\ 。

什么是类型类？
--------------------------------------------------------------------------------

类型类定义了一组函数签名，代表某种\ **行为契约**\ 。只要一个具体类型实现了这些函数，该类型就成为了该类型类的一个\ **实例（Instance）**\ 。

与面向对象接口的区别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **非侵入式**\ ：在传统面向对象语言中，一个类必须在声明自身时显式写明 ``implements Interface``\ 。而在 Haskell 中，类型定义与其类型类实例是分离的。你可以为第三方库乃至系统内置类型补充新的类型类实例。
2. **基于字典传递的实现**\ ：类型类在编译后通过字典传递（Dictionary Passing）实现，字典就是一组函数的记录。当调用处的类型已经确定时，编译器通常能把字典特化并内联，不再有运行时派发；只有在真正多态的代码里才会在运行时传递字典。
3. **基于返回值的多态**\ ：Haskell 的类型类不仅能约束入参，还能根据调用的\ **期望返回值类型**\ 进行多态分发（如 ``read`` 与 ``mempty``\ ）。

.. tip::

   **如果你熟悉其他语言**\ ：

   可以这样建立对类型类的直觉映射：

   - **Rust**\ ：几乎等价于 Rust 中的 ``trait``\ （Rust 的 trait 设计参考了 Haskell 的类型类）。
   - **Java / TypeScript / Go**\ ：类似于参数化接口 ``interface``\ （如 Java 的 ``Comparable<T>``\ ）。区别在于 Haskell 是\ **非侵入式**\ 的（无需修改原始类型的定义即可为其扩展实例），且支持根据\ **期望返回值类型**\ 进行静态多态分发。
   - **Python**\ ：类似于 ``typing.Protocol``\ （结构化子类型）或抽象基类。
   - **动态语言视角（Python / JS / Ruby）**\ ：类型类可以看作\ **编译期检查的鸭子类型（Duck Typing）**\ ：“如果它走起来像鸭子、叫起来像鸭子，那它就是鸭子”。与动态语言不同的是，契约在编译期就会被检查，不会出现运行时找不到方法的错误。

标准库常用类型类的层级
--------------------------------------------------------------------------------

Haskell 标准库中的类型类构成了一棵依赖树。下图中实线箭头表示“子类要求先实现父类”，虚线表示常见的实例类型：

.. mermaid::

   graph TD
     Eq["Eq<br/>(==), (/=)"] --> Ord["Ord<br/>compare, (<), (<=)"]
     Show["Show<br/>show"]
     Read["Read<br/>read"]
     Enum["Enum<br/>succ, pred, toEnum"]
     Bounded["Bounded<br/>minBound, maxBound"]

     Num["Num<br/>(+), (-), (*)"] --> Real["Real<br/>toRational"]
     Num --> Fractional["Fractional<br/>(/), fromRational"]
     Real --> Integral["Integral<br/>div, mod, toInteger"]
     Real --> RealFrac["RealFrac<br/>truncate, round, floor"]
     Fractional --> RealFrac
     Fractional --> Floating["Floating<br/>sin, exp, sqrt, log"]
     RealFrac --> RealFloat["RealFloat<br/>isNaN, isInfinite"]
     Floating --> RealFloat

     Int([Int]) -.实例.-> Integral
     Integer([Integer]) -.实例.-> Integral
     Float([Float]) -.实例.-> RealFloat
     Double([Double]) -.实例.-> RealFloat

     classDef inst fill:#f4f4f4,stroke:#999,stroke-dasharray: 4 2;
     class Int,Integer,Float,Double inst;

注意图中的方框都是类型类，\ ``Int``\ 、\ ``Double`` 这样的具体类型是它们的实例，而不是子类。\ ``Ord`` 的实例必须先是 ``Eq`` 的实例，\ ``Integral`` 的实例必须先是 ``Real`` 和 ``Num`` 的实例，依此类推。

内置类型类及其法则（Laws）
--------------------------------------------------------------------------------

在 Haskell 中，实现一个类型类不仅要求代码能够通过编译，还必须在逻辑上满足该类型类的\ **代数法则（Laws）**\ 。不遵守法则的实例会让依赖这些法则的通用算法产生难以排查的错误。

Eq（等价性）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

提供相等比较 ``(==)`` 与不等比较 ``(/=)``\ ：

.. code:: haskell

   class Eq a where
     (==), (/=) :: a -> a -> Bool
     x == y = not (x /= y)
     x /= y = not (x == y)
     {-# MINIMAL (==) | (/=) #-}

**Eq 法则**\ ：

1. **自反性（Reflexivity）**\ ：\ ``x == x = True``
2. **对称性（Symmetry）**\ ：\ ``x == y`` 与 ``y == x`` 结果必须相同。
3. **传递性（Transitivity）**\ ：若 ``x == y && y == z`` 为真，则 ``x == z`` 必为真。
4. **代换性（Substitutability）**\ ：若 ``x == y``\ ，则对任意纯函数 ``f``\ ，必有 ``f x == f y``\ 。

Ord（全序比较）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

继承自 ``Eq``\ ，提供大小比较。核心方法是三态比较函数 ``compare :: Ord a => a -> a -> Ordering``\ （返回 ``LT``\ 、\ ``EQ`` 或 ``GT``\ ）：

.. code:: haskell

   class Eq a => Ord a where
     compare :: a -> a -> Ordering
     (<), (<=), (>), (>=) :: a -> a -> Bool
     max, min :: a -> a -> a

**Ord 法则**\ ：

1. **自反性**\ ：\ ``x <= x = True``
2. **反对称性（Antisymmetry）**\ ：若 ``x <= y && y <= x``\ ，则 ``x == y``\ 。
3. **传递性**\ ：若 ``x <= y && y <= z``\ ，则 ``x <= z``\ 。
4. **完全可比性（Totality）**\ ：对任意 ``x`` 与 ``y``\ ，必有 ``x <= y || y <= x = True``\ 。

Bounded 与 Enum
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **Bounded**\ ：定义具有明确上限与下限的类型：

  .. code:: haskell

     class Bounded a where
       minBound :: a
       maxBound :: a

- **Enum**\ ：定义元素可与连续整数一一对应的顺序枚举类型：

  - ``succ :: Enum a => a -> a``\ ：后继元素。
  - ``pred :: Enum a => a -> a``\ ：前驱元素。
  - ``toEnum :: Enum a => Int -> a`` 与 ``fromEnum :: Enum a => a -> Int``\ 。

Show 与 Read（文本序列化往返）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``Show``\ ：提供 ``show :: Show a => a -> String``\ ，生成文本表示。
- ``Read``\ ：提供 ``read :: Read a => String -> a``\ ，从文本解析值。

**往返法则（Roundtrip Law）**\ ：

.. code:: text

   read (show x) == x

定义自己的类型类
--------------------------------------------------------------------------------

我们可以使用 ``class`` 关键字定义自己的类型类，并提供方法的默认实现：

.. code:: haskell

   -- 声明可描述抽象契约
   class Describable a where
     -- 核心方法
     describe :: a -> String

     -- 带有默认实现的方法
     describeBrief :: a -> String
     describeBrief = take 15 . describe

支持继承/先决约束（Subclassing）：

.. code:: haskell

   -- 要求 a 必须先实现 Describable，才能实现 Detailed
   class Describable a => Detailed a where
     detailedReport :: a -> String

手动编写类型类实例
--------------------------------------------------------------------------------

下面用 ``data`` 定义一个温标类型，并手动实现 ``Eq`` 与 ``Describable`` 实例（\ ``data`` 声明会在代数数据类型一章详细介绍，这里只需知道它定义了一个有两种形态的新类型）：

.. code:: haskell

   data Temperature
     = Celsius Double
     | Fahrenheit Double
     deriving (Show)

   instance Eq Temperature where
     (Celsius c1)    == (Celsius c2)    = c1 == c2
     (Fahrenheit f1) == (Fahrenheit f2) = f1 == f2
     (Celsius c)     == (Fahrenheit f)  = c == (f - 32) * 5 / 9
     (Fahrenheit f)  == (Celsius c)     = (Celsius c) == (Fahrenheit f)

   instance Describable Temperature where
     describe (Celsius c)    = "摄氏温度: " ++ show c ++ " °C"
     describe (Fahrenheit f) = "华氏温度: " ++ show f ++ " °F"

在 GHCi 中验证：

.. code:: text

   ghci> Celsius 0 == Fahrenheit 32
   True
   ghci> describe (Celsius 100)
   "摄氏温度: 100.0 °C"

派生机制（Deriving）与扩展
--------------------------------------------------------------------------------

基本自动派生
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于标准类型类，GHC 能够根据数据结构自动生成实例代码：

.. code:: haskell

   data Priority = Low | Medium | High
     deriving (Eq, Ord, Show, Read, Enum, Bounded)

只需一行，\ ``Priority`` 即可直接排序、打印和遍历：

.. code:: text

   ghci> Low < High
   True
   ghci> minBound :: Priority
   Low
   ghci> succ Low
   Medium
   ghci> [Low .. High]
   [Low,Medium,High]

泛化 newtype 派生（GeneralizedNewtypeDeriving）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果我们用 ``newtype`` 包装了一个底层类型，开启这个语言扩展后，编译器可以直接复用底层类型的实例：

.. code:: haskell

   {-# LANGUAGE GeneralizedNewtypeDeriving #-}

   newtype Score = Score Int
     deriving (Eq, Ord, Show, Num)

.. code:: text

   ghci> Score 80 + Score 15
   Score 95

孤儿实例（Orphan Instances）
--------------------------------------------------------------------------------

如果一个实例的类型类 ``C`` 与目标数据类型 ``T`` 均不在当前模块中定义，那么在该模块中编写 ``instance C T where`` 就构成了\ **孤儿实例**\ 。

- **问题**\ ：实例是全局的。如果两个模块各自为同一对类型类和类型写了实例，同时导入它们的代码会因为实例重叠而无法编译；而且实例是否可见取决于导入了哪些模块，容易造成难以理解的行为。
- **通常做法**\ ：把实例写在\ **定义该类型类的模块**\ 或\ **定义该数据类型的模块**\ 中。如果确实需要为外部类型补充外部类型类的实例，可以先用本地 ``newtype`` 包装一层，再为包装类型写实例。

小结
--------------------------------------------------------------------------------

- 类型类是 Haskell 实现特设多态的机制，实例与类型定义分离，可以为已有类型补充实例。
- 实现实例时要遵守该类型类的法则（如 Eq 的自反、对称、传递）。
- ``class`` 可以定义自己的类型类，并通过超类约束表达依赖关系。
- 标准类型类优先用 ``deriving`` 自动派生；避免写孤儿实例。
