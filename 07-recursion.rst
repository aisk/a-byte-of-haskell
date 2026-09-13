递归与计算模型
================================================================================

Haskell 中的变量与数据结构不可变，所以没有 ``while (i < n) { i++; }`` 这种依赖可变计数器的循环。纯函数式语言中的重复计算与状态演化都通过\ **递归（Recursion）**\ 来表达。

递归的两个组成部分
--------------------------------------------------------------------------------

设计递归函数时，需要明确两个部分：

1. **基准条件（Base Case）**\ ：计算的终止点，直接返回结果，不再递归。
2. **递归步骤（Recursive Step）**\ ：把问题拆成规模更小的同类子问题，逐步向基准条件逼近。

基础示例
--------------------------------------------------------------------------------

阶乘函数（Factorial）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义非负整数的阶乘：

.. code:: haskell

   factorial :: Integer -> Integer
   factorial 0 = 1                       -- 基准条件
   factorial n
     | n > 0     = n * factorial (n - 1) -- 递归步骤
     | otherwise = error "输入必须为非负整数"

展开过程：

.. code:: text

   factorial 3
   => 3 * factorial 2
   => 3 * (2 * factorial 1)
   => 3 * (2 * (1 * factorial 0))
   => 3 * (2 * (1 * 1))
   => 6

斐波那契数列（Fibonacci）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: haskell

   fib :: Int -> Integer
   fib 0 = 0
   fib 1 = 1
   fib n
     | n > 1     = fib (n - 1) + fib (n - 2)
     | otherwise = error "输入必须为非负整数"

整除与余数：在递归中传递状态
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

用重复减法实现带余除法，可以看到状态是如何通过参数在递归中传递的。这里对负数的处理与标准库 ``divMod`` 一致，即余数的符号与除数相同：

.. code:: haskell

   dividedBy :: Integral a => a -> a -> (a, a)
   dividedBy _ 0 = error "除数不能为零"
   dividedBy num denom
     | denom < 0 = let (q, r) = dividedBy (-num) (-denom) in (q, -r)
     | num < 0   = let (q, r) = dividedBy (-num) denom
                   in if r == 0 then (-q, 0) else (-q - 1, denom - r)
     | otherwise = go num denom 0
     where
       go n d count
         | n < d     = (count, n)            -- 余数已小于除数，到达基准条件
         | otherwise = go (n - d) d (count + 1)

在 GHCi 中验证，结果与 ``divMod`` 相同：

.. code:: text

   ghci> dividedBy 7 2
   (3,1)
   ghci> dividedBy (-7) 2
   (-4,1)
   ghci> dividedBy (-6) 3
   (-2,0)
   ghci> dividedBy 7 (-2)
   (-4,-1)

尾递归与累加器模式（Accumulator Pattern）
--------------------------------------------------------------------------------

回看前面的 ``factorial``\ ：

.. code:: haskell

   factorial n = n * factorial (n - 1)

在 ``factorial (n - 1)`` 返回之前，外层的乘法无法完成，所以会积累深度为 **O(n)** 的待计算表达式。

**尾递归（Tail Recursion）**\ 指函数的最后一步操作就是对自身的调用，不再有外层运算等待结果，因此不需要保留当前的调用上下文。

累加器模式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

引入一个局部辅助函数 ``go``\ ，用一个额外参数（\ **累加器**\ ）保存中间结果：

.. code:: haskell

   {-# LANGUAGE BangPatterns #-}

   factorialTail :: Integer -> Integer
   factorialTail n
     | n < 0     = error "输入必须为非负整数"
     | otherwise = go n 1
     where
       go 0 !acc = acc
       go k !acc = go (k - 1) (k * acc)

.. note::

   由于惰性求值，仅写成尾递归形式并不够，累加器里仍可能堆积未求值的 Thunk。用 ``!acc``\ （来自 ``BangPatterns`` 扩展，GHC2021 语言标准默认开启）或 ``seq`` 强制累加器在每一步求值，才能把空间占用降到常数 **O(1)**\ 。

相互递归（Mutual Recursion）
--------------------------------------------------------------------------------

多个函数可以互相调用，形成递归环。例如判断奇偶：

.. code:: haskell

   isEven :: Integral a => a -> Bool
   isEven 0 = True
   isEven n = isOdd (abs n - 1)

   isOdd :: Integral a => a -> Bool
   isOdd 0 = False
   isOdd n = isEven (abs n - 1)

同一模块内的顶层定义可以互相引用，不需要像 C 那样提前声明。

递归与余递归（Corecursion）
--------------------------------------------------------------------------------

除了从大问题逐步收敛到基准条件的“递归”，还有一种反方向的计算模型：\ **余递归（Corecursion）**\ 。

- **递归（Recursion）**\ ：面向\ **消耗**\ 与\ **归纳（Induction）**\ ，输入逐步变小，最终到达基准条件。
- **余递归（Corecursion）**\ ：面向\ **生产**\ 与\ **余归纳（Coinduction）**\ ，从初始种子出发不断生成数据，可以是无限的，只要求每一步都能产出新的部分（Productivity）。

结合惰性求值，余递归可以直接定义无限流：

.. code:: haskell

   -- 无限重复流（余递归）：
   myRepeat :: a -> [a]
   myRepeat x = x : myRepeat x

   -- 自引用的斐波那契无限流：
   fibs :: [Integer]
   fibs = 0 : 1 : zipWith (+) fibs (tail fibs)

消费者调用 ``take 10 fibs`` 时，惰性求值只会驱动余递归生成前 10 项。

发散与底（Bottom / ⊥）
--------------------------------------------------------------------------------

在类型理论中，\ **底**\ （写作 ⊥ 或 Bottom）指\ **无法计算出有效结果的计算**\ 。主要有两种情况：

1. **非终止计算**\ ：无限循环（发散）。
2. **运行时异常**\ ：例如调用 ``error "msg"`` 或 ``undefined``\ 。

.. code:: text

   ghci> let loop = loop in loop
   -- 挂起，永不返回，对应底

偏函数（Partial Functions）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果一个函数对某些类型上合法的输入没有定义返回值（会抛异常或返回底），它就是\ **偏函数**\ 。

标准库中的 ``head`` 就是典型例子：

.. code:: text

   ghci> head []
   *** Exception: Prelude.head: empty list

偏函数是运行期崩溃的常见来源。Haskell 提倡编写\ **全函数（Total Functions）**\ ，即对所有输入都有明确的输出，把“可能失败”用 ``Maybe`` 与 ``Either`` 这样的类型表达出来。

小结
--------------------------------------------------------------------------------

- 递归是纯函数式语言中表达循环的基本方式。
- 普通递归会积累待计算表达式，尾递归配合严格的累加器可以做到常数空间。
- 余递归从种子生成可能无限的数据流，由惰性的消费者决定取多少。
- 避免偏函数与底，用全函数和 ``Maybe``\ /\ ``Either`` 表达失败。
