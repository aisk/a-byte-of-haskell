类型类（Typeclasses）与法则约束
================================================================================

在面向对象语言中，多态通常依赖类继承（Inheritance）或接口（Interface）。而在 Haskell 中，实现“对不同类型提供统一操作契约”的机制是\ **类型类（Typeclasses）**\ ，它对应于计算机科学中的\ **特设多态（Ad-hoc Polymorphism）**\ 。

什么是类型类？
--------------------------------------------------------------------------------

类型类定义了一组函数签名，代表某种\ **行为契约**\ 。只要一个具体类型实现了这些函数，该类型就成为了该类型类的一个\ **实例（Instance）**\ 。

与面向对象接口的核心区别
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **非侵入式解耦**\ ：在传统面向对象语言中，一个类必须在声明自身时显式写明 ``implements Interface``\ 。而在 Haskell 中，类型定义与其类型类实例是完全分离的。你可以为第三方库乃至系统内置类型随时补充实现新的类型类实例。
2. **静态单态化与零开销派发**\ ：Haskell 的类型类在编译期直接由编译器解析特化（Dictionary Passing 机制被广泛优化内联），没有面向对象虚函数表（vtable）的运行时指针跳转开销。
3. **基于返回值的多态**\ ：Haskell 的类型类不仅能约束入参，还能根据调用的\ **期望返回值类型**\ 进行多态分发（如 ``read`` 与 ``mempty``\ ）。

.. tip::

   **如果你熟悉其他语言**\ ：

   可以这样建立对类型类的直觉映射：

   - **Rust**\ ：几乎等价于 Rust 中的 ``trait``\ （Rust 的 Trait 系统正是深度借鉴自 Haskell 的 Typeclass）。
   - **Java / TypeScript / Go**\ ：类似于参数化接口 ``interface``\ （如 Java 的 ``Comparable<T>``\ ）。但关键区别在于 Haskell 是\ **非侵入式**\ 的（无需修改原始类型的定义即可为其扩展实例），且支持根据\ **期望返回值类型**\ 进行静态多态分发。
   - **Python**\ ：类似于 ``typing.Protocol``\ （结构化子类型）或抽象基类。
   - **动态语言视角（Python / JS / Ruby）**\ ：类型类本质上是\ **“编译期零开销的静态鸭子类型（Duck Typing）”**\ ——“如果它走起来像鸭子、叫起来像鸭子，那它就是鸭子”。与动态语言不同的是，Haskell 的编译器会在编译期严格求证该契约，并在生成机器码时直接内联特化，既没有动态查找的运行时损耗，也从根本上消除了运行时属性不存在的报错隐患。

标准库核心类型类继承图谱
--------------------------------------------------------------------------------

Haskell 标准库中的类型类构成了层级清晰的依赖继承树：

.. code:: text

   Eq (等价比较)
    └── Ord (全序大小比较)

   Num (基础算术: +, -, *)
    ├── Real (可转为有理数)
    │    └── Integral (整除与余数: div, mod, toInteger)
    │         ├── Int (机器字长定长整型)
    │         └── Integer (大数任意精度整型)
    └── Fractional (实数除法: /, fromRational)
         ├── Float (单精度浮点)
         ├── Double (双精度浮点)
         └── Floating (初等超越函数: sin, cos, exp, sqrt, log)

核心内置类型类及其数学法则（Laws）
--------------------------------------------------------------------------------

在 Haskell 中，实现一个类型类不仅要求代码能够通过编译，还必须在逻辑上严格满足该类型类的\ **代数法则（Laws）**\ 。不遵守法则的实例会导致依靠等式推理的泛型算法出现难以排查的逻辑诡异行为。

Eq（等价性）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

提供相等比较 ``(==)`` 与不等比较 ``(/=)``\ ：

.. code:: haskell

   class Eq a where
     (==), (/=) :: a -> a -> Bool
     x == y = not (x /= y)
     x /= y = not (x == y)
     {-# MINIMAL (==) | (/=) #-}

**Eq 核心定律**\ ：

1. **自反性（Reflexivity）**\ ：\ ``x == x = True``
2. **对称性（Symmetry）**\ ：\ ``x == y`` 与 ``y == x`` 结果必须相同。
3. **传递性（Transitivity）**\ ：若 ``x == y && y == z`` 为真，则 ``x == z`` 必为真。
4. **代换性（Substitutability）**\ ：若 ``x == y``\ ，则对任意纯函数 ``f``\ ，必有 ``f x == f y``\ 。

Ord（全序比较）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

继承自 ``Eq``\ ，提供大小比较。最核心的原语是三态比较函数 ``compare :: Ord a => a -> a -> Ordering``\ （返回 ``LT``\ 、\ ``EQ`` 或 ``GT``\ ）：

.. code:: haskell

   class Eq a => Ord a where
     compare :: a -> a -> Ordering
     (<), (<=), (>), (>=) :: a -> a -> Bool
     max, min :: a -> a -> a

**Ord 核心定律**\ ：

1. **自反性**\ ：\ ``x <= x = True``
2. **反对称性（Antisymmetry）**\ ：若 ``x <= y && y <= x``\ ，则 ``x == y``\ 。
3. **传递性**\ ：若 ``x <= y && y <= z``\ ，则 ``x <= z``\ 。
4. **完全可比性（Totality）**\ ：对任意 ``x`` 与 ``y``\ ，必有 ``x <= y || y <= x = True``\ 。

Bounded 与 Enum
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **Bounded**\ ：定义具有明确上限与下限边界的类型：

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

定义全新的自定义类型类
--------------------------------------------------------------------------------

我们可以使用 ``class`` 关键字从零定义属于自己的类型类，并提供方法的默认实现：

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

定义温标代数数据类型，并手动实现 ``Eq`` 与 ``Describable`` 实例：

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

对于标准类型类，GHC 能够在编译期自动分析数据结构的代数特征并生成规范的实例代码：

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

在工程中，如果我们用 ``newtype`` 包装了一个底层类型，开启语言扩展后，编译器能将底层类型的能力\ **零开销自动穿透赋予**\ 新包装类型：

.. code:: haskell

   {-# LANGUAGE GeneralizedNewtypeDeriving #-}

   newtype Score = Score Int
     deriving (Eq, Ord, Show, Num)

.. code:: text

   ghci> Score 80 + Score 15
   Score 95

孤儿实例（Orphan Instances）防范准则
--------------------------------------------------------------------------------

如果一个实例的类型类 ``C`` 与目标数据类型 ``T`` 均不在当前模块中定义，那么在该模块中编写 ``instance C T where`` 就构成了\ **孤儿实例**\ 。

- **危害**\ ：当第三方模块导入你的代码时，会暗中引入潜在的全局实例冲突，破坏编译推导的确定性与一致性。
- **最佳实践**\ ：始终保证类型类实例编写在\ **定义该类型类的模块**\ 或\ **定义该数据类型的模块**\ 中。如果实在需要为外部类型补充外部实例，建议先用本地 ``newtype`` 进行封装后再写实例。

小结
--------------------------------------------------------------------------------

- 类型类是 Haskell 特设多态的核心，静态单态化确保了零运行时抽象成本。
- 实现类型类不仅是代码适配，更要严格遵循数学法则（自反、对称、传递、往返）。
- 掌握 ``class`` 自定义与子类约束，能构建高内聚低耦合的领域规范。
- 善用自动派生，严格避免孤儿实例以维护模块生态的整洁稳定。
