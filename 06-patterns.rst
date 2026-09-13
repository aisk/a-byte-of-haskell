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

.. note::

   **惰性模式**\ ：模式匹配会把入参求值到能看出构造器为止，\ ``strictPair (x, y) = ...`` 遇到 ``undefined`` 会立刻崩溃。在模式前加波浪号写成 ``lazyPair ~(x, y) = ...``\ ，匹配就被推迟到真正用到 ``x`` 或 ``y`` 的时候。日常代码几乎用不到它，典型场景只有两个：函数必须惰性地返回一个元组（标准库的 ``unzip``\ 、\ ``splitAt``\ ），以及用自引用定义构造循环结构。求值到哪一层、什么叫弱头范式，见惰性求值一章。

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

实际项目中通常会把这类警告当作错误处理，以保证函数是全函数（Total Function）。这正是基础语法一章警告 ``head``\ 、\ ``tail`` 危险的根源：它们对空列表没有分支。有了模式匹配，安全版本可以直接写成 ``safeHead (x : _) = Just x`` 加 ``safeHead [] = Nothing``\ ，把“可能没有结果”放进返回类型里，这是错误处理一章的主题。

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

四种写法的分工
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

到这里已经有了方程式模式匹配、\ ``case``\ 、\ ``if`` 和守卫四种分支手段，它们不是互相替代的关系，各有最自然的位置。用同一个任务来看：把 HTTP 状态码变成一句提示。

.. code:: haskell

   -- 模式匹配处理少数具体值，守卫处理范围，两者可以叠在同一个函数里
   describe :: Int -> String
   describe 200 = "成功"
   describe 404 = "资源未找到"
   describe code
     | code >= 500 && code < 600 = "服务器错误 " ++ show code
     | code >= 400               = "客户端错误 " ++ show code
     | otherwise                 = "其他状态码 " ++ show code

   -- case 匹配的是中间结果，而不是函数的入参
   firstFailure :: [Int] -> String
   firstFailure codes = case filter (>= 400) codes of
     []      -> "全部成功"
     (c : _) -> "首个失败: " ++ describe c

   -- if 只在表达式内部做一次布尔取舍
   retryHint :: Int -> String
   retryHint code = describe code ++ (if code >= 500 then "，可以重试" else "，不要重试")

.. code:: text

   ghci> describe 503
   "服务器错误 503"
   ghci> firstFailure [200, 200, 404, 500]
   "首个失败: 资源未找到"
   ghci> retryHint 403
   "客户端错误 403，不要重试"

- **方程式模式匹配**\ ：按构造器或少数几个字面量分派，同时把字段解构出来。它只能判断“长什么样”，表达不了范围。
- **守卫**\ ：按布尔条件或数值范围分派。
- **模式加守卫**\ ：先按结构拆开，再按条件细分。这是最常见的组合，下一章的 ``factorial`` 就是先匹配 ``0``\ ，再用守卫检查 ``n > 0``\ 。
- **case**\ ：要匹配的不是入参而是某个中间结果，或者匹配出现在一个更大表达式的中间。方程式写法只能匹配参数，\ ``case`` 可以匹配任何表达式。
- **if**\ ：只有一个布尔条件，而且嵌在表达式内部。一旦条件多于一个，或者需要 ``else if`` 链，换成守卫。

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

无点风格使代码聚焦于“数据流经的变换管道”，而不是临时变量的逐层传递。从 ``countEvenSquares xs = ...`` 到 ``countEvenSquares' = ...``\ ，两边同时去掉的那个 ``xs`` 就是第一章的 η 约简，\ ``.`` 只是让约简后的右侧仍然可读。

小结
--------------------------------------------------------------------------------

- 模式匹配同时完成结构检查、解构与分支分发。
- ``-Wincomplete-patterns`` 让编译器检查模式是否穷尽，穷尽的模式匹配就是全函数。
- 模式匹配管结构，守卫管条件，两者常叠用；\ ``case`` 匹配中间结果，\ ``if`` 只做单个布尔取舍。
- ``$`` 用于减少括号，\ ``.`` 用于组合函数并写出无点风格。
