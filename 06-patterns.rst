函数式控制流与模式匹配
================================================================================

在命令式语言中，流程控制主要依赖条件跳转与可变状态循环（如 ``for``\ 、\ ``while``\ ）。而在纯函数式语言中，控制流通过\ **模式匹配（Pattern Matching）**\ 、\ **守卫（Guards）**\ 与\ **高阶函数组合**\ 以声明式的方式表达。

模式匹配
--------------------------------------------------------------------------------

模式匹配不仅能检查数据是否符合特定形状，还能在匹配成功的同时完成\ **数据解构**\ 并将内部字段绑定到局部变量。

字面量与通配符模式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   greetByCode :: Int -> String
   greetByCode 200 = "成功"
   greetByCode 404 = "资源未找到"
   greetByCode 500 = "服务器内部错误"
   greetByCode _   = "其他状态码"   -- 通配符 '_' 匹配任何其他值

复合结构解构：元组与列表
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   -- 解构元组
   distanceFromOrigin :: (Double, Double) -> Double
   distanceFromOrigin (x, y) = sqrt (x^2 + y^2)

   -- 解构列表前两项并保留尾部
   sumHeadTwo :: Num a => [a] -> a
   sumHeadTwo (x1:x2:_) = x1 + x2
   sumHeadTwo [x]       = x
   sumHeadTwo []        = 0

As-模式（@）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当既需要将数据结构解构为其组成部分，又需要保留对原始整体数据的引用时，使用 ``@`` 语法：

.. code:: haskell

   inspectHeadAndAll :: Show a => [a] -> String
   inspectHeadAndAll all@(x:_) = "首元素是 " ++ show x ++ "，全列表为: " ++ show all
   inspectHeadAndAll []        = "空列表"

Record 记录模式解构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当数据类型使用记录语法声明时（记录语法会在代数数据类型一章详细介绍），我们可以在模式匹配中通过字段名显式解构，不受字段定义顺序的影响：

.. code:: haskell

   data Employee = Employee
     { empId     :: Int
     , empName   :: String
     , empSalary :: Double
     } deriving (Show)

   isHighEarner :: Employee -> Bool
   isHighEarner Employee { empSalary = s } = s > 100000

惰性模式（Lazy / Irrefutable Patterns）
--------------------------------------------------------------------------------

默认情况下，Haskell 的模式匹配是\ **可失败的**\ （Refutable Pattern）：它会检查数据构造器是否匹配。如果匹配失败，则跳转到下一个分支。

但在某些场景中，过早检查构造器可能会引发不必要的底（⊥ / Bottom）求值或死循环。此时可以在模式前添加波浪号 ``~``\ ，将其声明为\ **不可失败的惰性模式（Irrefutable Pattern）**\ ：

.. code:: haskell

   -- 普通模式：入参在匹配时会被求值到构造器层级
   strictPair :: (a, b) -> String
   strictPair (x, y) = "Pair matched"

   -- 惰性模式：波浪号修饰模式，延迟匹配，不立即求值外层构造器
   lazyPair :: (a, b) -> String
   lazyPair ~(x, y) = "Pair matched safely"

在 GHCi 中对比其求值行为：

.. code:: text

   ghci> strictPair undefined
   *** Exception: Prelude.undefined
   ghci> lazyPair undefined
   "Pair matched safely"  -- 模式没有被求值，所以没有崩溃

.. note::

   波浪号 ``~`` 是表达式/方程中的\ **模式修饰符**\ ，不能写在类型签名中。

编译期穷尽性检查（Exhaustiveness Checking）
--------------------------------------------------------------------------------

编写模式匹配时，如果漏掉了某些分支，程序在遇到未覆盖的输入时会在运行期崩溃。

GHC 会做穷尽性检查。建议始终开启 ``-Wincomplete-patterns`` 编译选项（或使用 ``-Wall``\ ）：

.. code:: haskell

   {-# OPTIONS_GHC -Wincomplete-patterns #-}

   -- 如果遗漏了空列表分支，GHC 在编译期就会给出警告：
   headUnsafe :: [a] -> a
   headUnsafe (x:_) = x

编译器会提示：

.. code:: text

   Pattern match(es) are non-exhaustive
   In an equation for 'headUnsafe': Patterns of type '[a]' not matched: []

实际项目中通常会把这类警告当作错误处理，以保证函数是全函数（Total Function）。

条件分支：if-then-else 与 case-of
--------------------------------------------------------------------------------

if 表达式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Haskell 的 ``if-then-else`` 是一个\ **必须返回值的表达式**\ ，而不是命令式语句：

1. 必须始终包含完整的 ``else`` 分支。
2. ``then`` 分支与 ``else`` 分支的表达式返回值类型必须相同。

.. code:: haskell

   absVal :: (Num a, Ord a) => a -> a
   absVal n = if n >= 0 then n else -n

case-of 表达式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

函数参数的模式匹配在底层最终都会被编译器脱糖为 ``case ... of`` 表达式。它可以在任何需要表达式的位置嵌套：

.. code:: haskell

   describeNumber :: Int -> String
   describeNumber n = "该数字是: " ++ case n of
     0 -> "零"
     1 -> "壹"
     _ -> "其他数字"

守卫（Guards）
--------------------------------------------------------------------------------

模式匹配擅长检查数据的“结构”，而\ **守卫**\ 擅长检查“布尔条件”：

.. code:: haskell

   rateInterest :: Double -> Double
   rateInterest balance
     | balance <= 0     = 0.0
     | balance < 10000  = 0.015
     | balance < 100000 = 0.025
     | otherwise        = 0.035

``otherwise`` 在标准库中就是布尔常量 ``otherwise = True``\ ，作为所有未命中条件的默认分支。

函数操作符：$ 与 .
--------------------------------------------------------------------------------

函数应用符：$
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

常规函数调用的优先级最高（优先级 10），并且为左结合。而操作符 ``$`` 的优先级最低（优先级 0），为\ **右结合**\ ：

.. code:: haskell

   ($) :: (a -> b) -> a -> b
   f $ x = f x

它的主要用途是\ **减少括号**\ ：

.. code:: haskell

   -- 括号嵌套：
   print (sum (filter even (map (* 2) [1..10])))

   -- 使用 $ 简化：
   print $ sum $ filter even $ map (* 2) [1..10]

函数组合符：. 与无点风格（Pointfree）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

数学中的函数复合 (f ∘ g)(x) = f(g(x)) 在 Haskell 中用点号 ``.`` 表达：

.. code:: haskell

   (.) :: (b -> c) -> (a -> b) -> a -> c
   (f . g) x = f (g x)

我们可以将多个处理步骤拼接为单一管道，无需显式声明入参变量名，这就是\ **无点风格（Pointfree Style）**\ ：

.. code:: haskell

   -- 普通风格：
   countEvenSquares :: [Int] -> Int
   countEvenSquares xs = length (filter even (map (^2) xs))

   -- 无点风格：
   countEvenSquares' :: [Int] -> Int
   countEvenSquares' = length . filter even . map (^2)

无点风格使代码聚焦于“数据流经的变换管道”，而不是临时变量的逐层传递。

小结
--------------------------------------------------------------------------------

- 模式匹配同时完成结构检查、解构与分支分发。
- 惰性模式 ``~`` 延迟对构造器的求值，只能写在参数模式里。
- ``-Wincomplete-patterns`` 让编译器检查模式是否穷尽。
- ``$`` 用于减少括号，\ ``.`` 用于组合函数并写出无点风格。
