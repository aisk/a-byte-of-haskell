单子变换子（Monad Transformers）与 MTL 风格
================================================================================

在前面的章节中，我们分别掌握了处理各种不同效果的单子：\ ``Maybe``\ （处理缺失）、\ ``Either``\ （错误处理）、\ ``Reader``\ （配置注入）、\ ``State``\ （状态演化）以及 ``IO``\ （物理副作用）。

但在真实工业级项目中，我们往往\ **需要同时拥有多种能力**\ ：例如一个 Web 接口既要读取全局配置、又要访问数据库、还要维护请求局部状态，并在出现校验失败时优雅抛错短路。

为什么 Monad 无法自由复合？
--------------------------------------------------------------------------------

在范畴论中存在一个经典事实：
- 任意两个 ``Functor`` 的嵌套组合天然也是一个合法的 ``Functor``\ 。
- 任意两个 ``Applicative`` 的嵌套组合天然也是一个合法的 ``Applicative``\ 。
- **但任意两个 Monad 的嵌套，却无法通用地自动复合为一个新的 Monad！**

因为 Monad 的核心是拍平操作 ``join :: m (n (m (n a))) -> m (n a)``\ 。如果内外两个单子彼此不知道对方的内部具体数据构造细节，就无法推导出通用的展平规则。

为此，Haskell 社区提出了成熟的工程解决方案——\ **单子变换子（Monad Transformers）**\ 。

变换子家族：给现有单子附加超能力
--------------------------------------------------------------------------------

通常，变换子以大写字母 ``T`` 结尾，定义在 ``transformers`` 库中：

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

实战构建：多层应用服务栈
--------------------------------------------------------------------------------

构建一个兼备配置注入、错误短路与 IO 能力的应用栈：

.. code:: haskell

   import Control.Monad.Reader
   import Control.Monad.Except

   data Env = Env { serverPort :: Int }
   type App = ReaderT Env (ExceptT String IO)

   runApp :: Env -> App a -> IO (Either String a)
   runApp env app = runExceptT (runReaderT app env)

   serviceHandler :: App ()
   serviceHandler = do
     port <- asks serverPort
     liftIO $ putStrLn $ "服务启动中，监听端口: " ++ show port
     if port < 1024
       then throwError "非特权用户无法监听特权端口！"
       else liftIO $ putStrLn "服务启动成功。"

现代工程标杆：MTL 风格（Monad Transformer Library）
--------------------------------------------------------------------------------

传统具象变换子栈的痛点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果我们在所有业务代码中都写死具体的单子栈类型 ``App``\ ，并到处充斥着 ``lift``\ ，一旦某天架构重构（例如需要额外增加一层 ``StateT``\ ），原先所有业务代码的 ``lift`` 层级都会全部失效！

MTL 风格：基于类型类约束面向接口编程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MTL（Monad Transformer Library）是现代 Haskell 大型软件的绝对主流范式。它的核心思想是：\ **业务逻辑不声明具体的变换子栈，而是通过类型类对环境能力提出需求约束**\ ：

- 需要读配置？声明约束：\ ``MonadReader Env m``
- 需要抛异常？声明约束：\ ``MonadError String m``
- 需要调用 IO？声明约束：\ ``MonadIO m``

.. code:: haskell

   -- 业务函数完全不绑定具体的单子栈！
   pureBusinessLogic :: (MonadReader Env m, MonadError String m, MonadIO m) => m ()
   pureBusinessLogic = do
     port <- asks serverPort
     liftIO $ putStrLn $ "当前端口: " ++ show port
     when (port < 1024) $ throwError "非法端口"

**MTL 风格的巨大工程价值**\ ：
1. **解耦调用层级**\ ：不再需要手写任何繁琐的 ``lift``\ ，MTL 的实例机制会自动替你跨层寻址。
2. **极速单元测试**\ ：在生产运行中使用真实的 ``ReaderT Env (ExceptT String IO)``\ ；在自动化单元测试中，可以直接用纯内存的 Dummy 纯函数类型运行，完全无需模拟真实的外部 IO 依赖！
