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

全局图景：值的类型是一组类型类的实例
--------------------------------------------------------------------------------

先建立一个贯穿全书的视角。在 Haskell 里，几乎每一个值的类型都同时是好几个类型类的实例。\ ``3 :: Int`` 能比较、能打印、能加减，不是因为 ``Int`` 内置了这些能力，而是因为 ``Int`` 是 ``Eq``\ 、\ ``Ord``\ 、\ ``Show``\ 、\ ``Num`` 的实例。一个函数签名里 ``=>`` 左边的约束，就是在说“这里只用到了这几个类型类的方法”：

.. code:: haskell

   sort    :: Ord a => [a] -> [a]              -- 只需要能比较大小
   sum     :: (Foldable t, Num a) => t a -> a  -- 只需要能折叠、能加
   mconcat :: Monoid a => [a] -> a             -- 只需要能合并、有单位元

这带来三个直接后果。

**实现少数几个方法，就能获得一整套函数**\ 。每个类型类只要求实现一个很小的核心（类定义里的 ``MINIMAL`` 标注），其余方法要么在类内部有默认实现，要么是标准库里以该类型类为约束写好的通用函数。比如为自己的类型实现 ``compare``\ ，就自动拥有 ``<``\ 、\ ``max``\ 、\ ``min``\ ，也能直接调用 ``sort``\ 、\ ``maximum``\ ，还能当 ``Map`` 的键。下表列出最常用的类型类各自要求什么、送什么，后面的章节会逐个展开：

.. list-table::
   :header-rows: 1
   :widths: 18 24 58

   * - 类型类
     - 最少要实现
     - 随之可用
   * - ``Eq``
     - ``==``
     - ``/=``\ ；\ ``elem``\ 、\ ``lookup``\ 、\ ``nub``\ 、\ ``group``
   * - ``Ord``
     - ``compare`` 或 ``<=``
     - ``<``\ 、\ ``>``\ 、\ ``max``\ 、\ ``min``\ ；\ ``sort``\ 、\ ``maximum``\ ，作 ``Map`` 的键、\ ``Set`` 的元素
   * - ``Show``
     - ``show``
     - ``print``\ ，GHCi 里直接显示
   * - ``Enum``
     - ``toEnum``\ 、\ ``fromEnum``
     - ``succ``\ 、\ ``pred``\ ，区间写法 ``[a .. b]``
   * - ``Num``
     - ``+``\ 、\ ``*``\ 、\ ``abs``\ 、\ ``signum``\ 、\ ``fromInteger``\ 、\ ``negate``
     - 数字字面量直接当该类型用；\ ``sum``\ 、\ ``product``
   * - ``Semigroup`` / ``Monoid``
     - ``<>``\ ；\ ``mempty``
     - ``mconcat``\ 、\ ``foldMap``\ 、\ ``stimes``
   * - ``Functor``
     - ``fmap``
     - ``<$>``\ 、\ ``<$``\ 、\ ``void``
   * - ``Applicative``
     - ``pure``\ 、\ ``<*>``
     - ``liftA2``\ 、\ ``*>``\ 、\ ``<*``\ 、\ ``when``\ 、\ ``unless``\ ，配合 ``Traversable`` 的 ``traverse``
   * - ``Alternative``
     - ``empty``\ 、\ ``<|>``
     - ``optional``\ 、\ ``many``\ 、\ ``some``\ 、\ ``asum``\ 、\ ``guard``
   * - ``Monad``
     - ``>>=``
     - ``do`` 记号；\ ``join``\ 、\ ``mapM``\ 、\ ``forM``\ 、\ ``foldM``\ 、\ ``replicateM``\ 、\ ``>=>``
   * - ``Foldable``
     - ``foldr`` 或 ``foldMap``
     - ``length``\ 、\ ``sum``\ 、\ ``elem``\ 、\ ``null``\ 、\ ``toList``\ 、\ ``maximum``\ 、\ ``any``\ 、\ ``traverse_``\ 、\ ``mapM_``
   * - ``Traversable``
     - ``traverse`` 或 ``sequenceA``
     - ``mapM``\ 、\ ``sequence``\ 、\ ``for``\ 、\ ``forM``

**很多方法是运算符**\ 。\ ``==``\ 、\ ``<``\ 、\ ``+``\ 、\ ``<>``\ 、\ ``<$>``\ 、\ ``<*>``\ 、\ ``>>=``\ 、\ ``<|>`` 都是某个类型类里的方法，只是名字用符号写。运算符在 Haskell 里就是普通函数，加上括号就能当函数用（\ ``(+) 1 2``\ ）。Haskell 代码看起来符号很多，实际上翻来覆去就是这十几个运算符，换到不同的类型上含义按实例走。在 GHCi 里 ``:i (<>)`` 或 ``:t (<*>)`` 能看到一个运算符属于哪个类型类。

**读代码有固定套路**\ 。看到一个运算符，问它属于哪个类型类；看到一个签名，看 ``=>`` 左边有哪些约束，就知道函数体里能对这个值做什么、不能做什么。反过来写代码时，也先问“我需要它有什么能力”，再写约束，而不是先定死具体类型。

标准库类型类的层级
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

标准库中的类型类构成两棵依赖树。实线箭头表示“子类要求先实现父类”，虚线表示常见的实例类型。第一棵描述单个值的能力，比较、枚举、算术：

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

第二棵描述“容器”和“带效果的计算”的能力：合并、映射、组合、遍历。这些是第四部分的主题，现在只需要知道它们之间的关系：

.. mermaid::

   graph TD
     Semigroup["Semigroup<br/>(<>)"] --> Monoid["Monoid<br/>mempty, mconcat"]

     Functor["Functor<br/>fmap, (<$>)"] --> Applicative["Applicative<br/>pure, (<*>)"]
     Applicative --> Monad["Monad<br/>(>>=), return"]
     Applicative --> Alternative["Alternative<br/>empty, (<|>)"]
     Monad --> MonadFail["MonadFail<br/>fail"]
     Monad --> MonadIO["MonadIO<br/>liftIO"]

     Foldable["Foldable<br/>foldr, foldMap, toList"] --> Traversable["Traversable<br/>traverse, sequenceA"]
     Functor --> Traversable

     List(["[a]"]) -.实例.-> Monoid
     List -.实例.-> Alternative
     List -.实例.-> Traversable
     Maybe(["Maybe a"]) -.实例.-> Alternative
     Maybe -.实例.-> Traversable
     IO(["IO a"]) -.实例.-> MonadIO
     Text(["Text / ByteString"]) -.实例.-> Monoid
     Map(["Map k v"]) -.实例.-> Monoid
     Map -.实例.-> Traversable

     classDef inst fill:#f4f4f4,stroke:#999,stroke-dasharray: 4 2;
     class List,Maybe,IO,Text,Map inst;

图中的方框都是类型类，\ ``Int``\ 、\ ``Maybe a`` 这样的具体类型是它们的实例，而不是子类。\ ``Ord`` 的实例必须先是 ``Eq`` 的实例，\ ``Monad`` 的实例必须先是 ``Applicative`` 和 ``Functor`` 的实例，依此类推。注意第二棵树里的类型类约束的是\ **类型构造器**\ ：\ ``Functor`` 的实例是 ``Maybe`` 而不是 ``Maybe Int``\ ，是 ``[]`` 而不是 ``[Int]``\ 。这是 ``Text`` 能是 ``Monoid`` 却不能是 ``Functor`` 的原因：它不装别的类型的值。

下表是常见类型各自实现了哪些类型类。空格表示没有实例，括号内是附加条件：

.. list-table::
   :header-rows: 1
   :widths: 22 14 20 26 18

   * - 类型
     - Eq / Ord
     - Semigroup / Monoid
     - Functor / Applicative / Monad
     - Foldable / Traversable
   * - ``Int``\ 、\ ``Double``\ 、\ ``Char``\ 、\ ``Bool``
     - ✓
     -
     -
     -
   * - ``String``\ 、\ ``Text``\ 、\ ``ByteString``
     - ✓
     - ✓
     -
     -
   * - ``[a]``
     - ✓（a 也是）
     - ✓
     - ✓，还有 Alternative
     - ✓
   * - ``Maybe a``
     - ✓（a 也是）
     - ✓（a 是 Semigroup）
     - ✓，还有 Alternative
     - ✓
   * - ``Either e a``
     - ✓（e、a 也是）
     - 只有 Semigroup
     - ✓
     - ✓
   * - ``(a, b)``
     - ✓（a、b 也是）
     - ✓（a、b 也是）
     - ✓（作用在 b 上，a 是 Monoid）
     - ✓（作用在 b 上）
   * - ``Map k v``
     - ✓
     - ✓（左优先合并）
     - 只有 Functor
     - ✓
   * - ``Set a``
     - ✓
     - ✓（并集）
     -
     - 只有 Foldable
   * - ``IO a``
     -
     - ✓（a 也是）
     - ✓，还有 Alternative、MonadIO
     -
   * - ``r -> a``
     -
     - ✓（a 也是）
     - ✓
     -

``String`` 就是 ``[Char]``\ ，所以列表那一行的能力它全都有，第二行是把它和 ``Text`` 放在一起看作“文本”时的常用视角。

社区里的知名类型类
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

第三方库沿用同一套机制：定义一个类型类，说明“你的类型只要能做这件事，就能用我的全部功能”。下面是最常遇到的几个：

.. list-table::
   :header-rows: 1
   :widths: 20 26 54

   * - 库
     - 类型类
     - 用途
   * - ``aeson``
     - ``ToJSON``\ 、\ ``FromJSON``
     - 类型与 JSON 互转。实现了它，\ ``encode``\ 、\ ``decode`` 以及所有接受 JSON 的 HTTP 库函数就都能用。通常靠 ``Generic`` 派生，一行 ``instance ToJSON User`` 即可。见 JSON 一章。
   * - ``hashable``
     - ``Hashable``
     - 提供哈希值，是 ``HashMap``\ 、\ ``HashSet`` 对键的要求，地位相当于 ``Map`` 之于 ``Ord``\ 。
   * - ``QuickCheck``
     - ``Arbitrary``
     - 随机生成该类型的测试数据，是属性测试的基础。实现 ``arbitrary`` 后，任何以该类型为参数的性质都能自动测试。
   * - ``deepseq``
     - ``NFData``
     - 说明如何把值完整求值，供 ``deepseq``\ 、\ ``force`` 使用，见惰性一章。
   * - ``mtl``
     - ``MonadReader``\ 、\ ``MonadState``\ 、\ ``MonadError``
     - 用约束描述“这段代码需要读配置、改状态、能报错”，而不绑定具体的单子栈，见单子变换器一章。
   * - ``unliftio-core``
     - ``MonadUnliftIO``
     - 让 ``bracket``\ 、\ ``forkIO`` 这类接受 IO 回调的函数能在单子栈里使用。
   * - ``base``
     - ``IsString``\ 、\ ``Generic``\ 、\ ``Exception``
     - ``IsString`` 让字符串字面量可以当 ``Text`` 用（\ ``OverloadedStrings``\ ）；\ ``Generic`` 把类型的结构暴露给其他库，是自动派生 ``ToJSON`` 这类实例的基础；\ ``Exception`` 让一个类型可以被 ``throwIO`` 和 ``catch``\ 。

还有两个著名的库值得单独说，因为它们的核心\ **不是**\ 新的类型类，而是直接建立在标准类型类之上：

- ``lens``\ ：一个 ``Lens' s a`` 的定义是 ``forall f. Functor f => (a -> f a) -> s -> f s``\ ，也就是一个对任意 ``Functor`` 都成立的函数。读取时把 ``f`` 选成 ``Const``\ ，修改时选成 ``Identity``\ ，遍历多个位置的 ``Traversal`` 则把 ``Functor`` 换成 ``Applicative``\ 。正因为 lens 只是函数，它们才能用普通的 ``.`` 组合。库里另外有 ``Ixed``\ （\ ``ix``\ ）、\ ``At``\ （\ ``at``\ ）、\ ``Each``\ （\ ``each``\ ）这样的类型类，为列表、\ ``Map``\ 、元组统一提供“按下标访问”和“遍历每个元素”。
- ``conduit``\ ：流处理的每一段 ``ConduitT i o m r`` 都是 ``Functor``\ 、\ ``Applicative``\ 、\ ``Monad``\ 、\ ``MonadIO`` 的实例。所以写一个处理阶段就是写 ``do`` 块，里面可以用 ``liftIO``\ 、\ ``when``\ 、\ ``forever``\ 、\ ``mapM_``\ 。库自己只新增了把阶段串起来的 ``.|`` 和 ``await``\ 、\ ``yield`` 几个原语。

命令行一章介绍的 ``optparse-applicative`` 也是同样的思路：库只定义 ``Parser a`` 并为它实现 ``Applicative`` 和 ``Alternative``\ ，组合选项用的是标准的 ``<$>``\ 、\ ``<*>`` 和 ``<|>``\ 。学会标准类型类，就等于学会了这些库的大半用法。

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
- 一个类型通常同时是多个类型类的实例。实现最小的核心方法就能得到整套函数，其中很多是运算符；读代码时先看运算符属于哪个类型类、签名里有哪些约束。
- 第三方库沿用同一机制：\ ``aeson`` 的 ``ToJSON``\ 、\ ``hashable`` 的 ``Hashable``\ 、\ ``QuickCheck`` 的 ``Arbitrary`` 等；\ ``lens`` 和 ``conduit`` 则直接建立在 ``Functor``\ 、\ ``Applicative``\ 、\ ``Monad`` 之上。
