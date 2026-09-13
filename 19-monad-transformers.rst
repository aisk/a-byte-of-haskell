单子变换子（Monad Transformers）与 MTL 风格
================================================================================

在前面的章节中，我们分别掌握了处理各种不同效果的单子：\ ``Maybe``\ （处理缺失）、\ ``Either``\ （错误处理）、\ ``Reader``\ （配置注入）、\ ``State``\ （状态演化）以及 ``IO``\ （物理副作用）。

但在真实的工业级项目中，我们往往\ **需要同时拥有多种能力**\ ：例如一个 Web 接口既要读取全局配置、又要访问数据库 IO、还要维护请求上下文状态，并在出现校验失败时优雅抛错短路。

为什么 Monad 无法自动自由复合？
--------------------------------------------------------------------------------

在范畴论中存在一个经典事实：

- 任意两个 ``Functor`` 的嵌套组合天然也是一个合法的 ``Functor``\ 。
- 任意两个 ``Applicative`` 的嵌套组合天然也是一个合法的 ``Applicative``\ 。
- **但任意两个 Monad 的嵌套，却无法通用地自动复合为一个新的 Monad！**

因为 Monad 的核心是拍平操作 ``join :: m (n (m (n a))) -> m (n a)``\ 。如果内外两个单子彼此不知道对方的内部具体数据构造细节，就无法推导出通用的展平规则。

为此，Haskell 社区提出了成熟的工程解决方案——\ **单子变换子（Monad Transformers）**\ 。

变换子家族：给现有单子附加超能力
--------------------------------------------------------------------------------

通常，变换子以大写字母 ``T`` 结尾，定义在 ``transformers`` 与 ``mtl`` 库中：

- **MaybeT m a**\ ：给基础单子 ``m`` 附加缺失处理能力，内部包裹 ``m (Maybe a)``\ 。
- **ExceptT e m a**\ ：给基础单子 ``m`` 附加异常错误处理能力，内部包裹 ``m (Either e a)``\ 。
- **ReaderT r m a**\ ：给基础单子 ``m`` 附加只读环境，内部包裹 ``r -> m a``\ 。
- **StateT s m a**\ ：给基础单子 ``m`` 附加状态机，内部包裹 ``s -> m (a, s)``\ 。

核心操作：MonadTrans 与 lift
--------------------------------------------------------------------------------

当我们把一个 Monad 嵌套在变换子内部时，如何从外层调用底层 Monad 的操作？通过 ``MonadTrans`` 类型类的 ``lift`` 函数：

.. code:: haskell

   class MonadTrans t where
     lift :: Monad m => m a -> t m a

``lift`` 将下层 Monad 的动作“提升”一层，使其可以无缝融入当前的外层 ``do`` 代码块。

穿透一切的 MonadIO 与 liftIO
--------------------------------------------------------------------------------

在绝大多数工程应用中，Monad 栈的最底层通常是 ``IO``\ 。为了避免多层嵌套时写出令人头晕的 ``lift . lift . lift``\ ，标准库提供了 ``MonadIO``\ ：

.. code:: haskell

   class Monad m => MonadIO m where
     liftIO :: IO a -> m a

无论你的栈堆叠了多少层变换子，只需一个 ``liftIO``\ ，就能立刻穿透整个栈直达底层执行 IO 操作。

变换子堆叠顺序的语义差异
--------------------------------------------------------------------------------

单子变换子的堆叠顺序绝非无关紧要，**不同的堆叠顺序代表完全不同的业务容错语义**\ 。

对比以下两种经典组合：

1. **StateT 在内，ExceptT 在外（\ ``ExceptT e (State s) a``\ ）**\ ：
   - 底层结构等价于：\ ``s -> (Either e a, s)``\ 。
   - **语义**\ ：即使计算抛出错误（\ ``throwError``\ ），错误分支依然与状态并列，\ **发生错误前的状态修改会被完整保留**\ ！
2. **ExceptT 在内，StateT 在外（\ ``StateT s (Except e) a``\ ）**\ ：
   - 底层结构等价于：\ ``s -> Either e (a, s)``\ 。
   - **语义**\ ：一旦抛出错误，整个状态元组被抛弃，\ **所有之前的状态变更全部回滚丢失**\ （如同数据库事务回滚）。

在架构设计时，必须根据业务是否需要“错误回滚”严格决定变换子的嵌套层级。

现代工业标准：ReaderT 架构设计模式
--------------------------------------------------------------------------------

在过去，开发者常喜欢堆叠 5 到 6 层复杂的变换子。然而现代 Haskell 工业界普遍推崇更为务实轻量的 **ReaderT 设计模式（The ReaderT Pattern）**\ ：

**核心思想**\ ：将应用的核心单子统一固化为 ``ReaderT Env IO``\ ，把数据库连接池、日志句柄、可变内存变量（``IORef`` / ``TVar``）全部装入环境结构体 ``Env`` 中：

.. code:: haskell

   {-# LANGUAGE GeneralizedNewtypeDeriving #-}

   import Control.Monad.Reader

   data Env = Env
     { appPort  :: !Int
     , appDbUrl :: !String
     }

   -- 使用 newtype 封装单一层级的 ReaderT Env IO
   newtype App a = App
     { unApp :: ReaderT Env IO a
     } deriving (Functor, Applicative, Monad, MonadReader Env, MonadIO)

   runApp :: Env -> App a -> IO a
   runApp env action = runReaderT (unApp action) env

   -- 业务函数清爽优雅，天然拥有环境注入与 IO 穿透能力：
   startServer :: App ()
   startServer = do
     port <- asks appPort
     liftIO $ putStrLn $ "服务在端口 " ++ show port ++ " 上平稳启动..."

这种模式结构简单、性能极致，且彻底规避了复杂的变换子性能开销。

MTL 风格：基于类型类约束面向接口编程
--------------------------------------------------------------------------------

如果要在大型系统中实现极致的组件解耦，可以使用 MTL 风格。业务函数完全不绑定具体的单子栈，而是通过类型类约束声明其能力需求：

- 需要读配置？声明约束：\ ``MonadReader Env m``
- 需要抛异常？声明约束：\ ``MonadError String m``
- 需要调用 IO？声明约束：\ ``MonadIO m``

.. code:: haskell

   import Control.Monad.Reader
   import Control.Monad.Except

   pureBusinessLogic :: (MonadReader Env m, MonadError String m, MonadIO m) => m ()
   pureBusinessLogic = do
     port <- asks appPort
     liftIO $ putStrLn $ "当前端口: " ++ show port
     if port < 1024
       then throwError "非特权端口错误"
       else liftIO $ putStrLn "校验通过。"

**工程价值**\ ：
1. **解耦调用层级**\ ：不再需要手写任何繁琐的 ``lift``\ ，MTL 的实例机制会自动跨层寻址。
2. **极速单元测试**\ ：在自动化测试中，可以直接传入纯内存的 Mock 单子运行，完全无需拉起真实的外部网络与数据库环境。

小结
--------------------------------------------------------------------------------

- Monad 无法自动复合，需借助单子变换子（Monad Transformers）按需叠加能力。
- ``lift`` 跨层提升动作，\ ``liftIO`` 穿透整个单子栈直达 IO。
- 变换子堆叠顺序直接决定错误发生时状态是否持久化或回滚。
- 现代工程推崇以 ``ReaderT Env IO`` 为核心的轻量架构，兼具简洁性与高性能。
- MTL 风格基于类型类抽象能力契约，为大型应用提供卓越的模块解耦与可测试性。
