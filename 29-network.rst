网络编程：Socket
================================================================================

这一章介绍如何用 Haskell 写 TCP 客户端和服务器。使用的是 ``network`` 包，它是 Haskell 生态里 Socket 编程的标准库，绝大多数网络库（HTTP 服务器、数据库驱动等）都建立在它之上。本章的例子会用到前面章节讲过的 ``bracket``\ （IO 一章）、\ ``forkFinally``\ （并发一章）以及 ``Text`` 与 ``ByteString`` 的编码转换（字符串一章）。

需要在 ``.cabal`` 文件中加入这些依赖：

.. code:: text

   build-depends:    base >= 4.14 && < 5
                   , network >= 3.1.2
                   , bytestring
                   , text
                   , containers
                   , stm

Socket 基础
--------------------------------------------------------------------------------

``Network.Socket`` 模块暴露的接口和 POSIX 的 Socket API 基本一一对应，几个核心类型：

- ``Socket``\ ：一个套接字句柄，对应操作系统的文件描述符；
- ``SockAddr``\ ：套接字地址，包含 IP 和端口；
- ``AddrInfo``\ ：地址解析的结果，除了 ``SockAddr`` 还带有协议族、套接字类型等信息。

地址解析
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

不管是连接远端还是监听本机端口，第一步都是把主机名和端口解析成 ``AddrInfo``\ ：

.. code:: haskell

   import Network.Socket

   -- 客户端：解析要连接的目标
   let hints = defaultHints { addrSocketType = Stream }
   addrs <- getAddrInfo (Just hints) (Just "example.com") (Just "80")

   -- 服务器：解析本机监听地址，AI_PASSIVE 表示接受任意来源的连接
   let serverHints = defaultHints { addrSocketType = Stream, addrFlags = [AI_PASSIVE] }
   serverAddrs <- getAddrInfo (Just serverHints) Nothing (Just "4000")

``getAddrInfo`` 返回一个列表，因为一个主机名可能同时对应 IPv4 和 IPv6 地址。日常代码通常取第一个结果。解析失败时它会抛出 ``IOException``\ ，所以列表不会为空，本章的例子用一个小函数 ``firstAddr`` 取第一个元素。

``addrSocketType = Stream`` 表示 TCP，\ ``Datagram`` 则表示 UDP。本章只讨论 TCP。

创建与关闭 Socket
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

拿到 ``AddrInfo`` 之后，用 ``openSocket`` 创建对应的 Socket，它会自动从 ``AddrInfo`` 里取出协议族、类型和协议号：

.. code:: haskell

   openSocket :: AddrInfo -> IO Socket
   close      :: Socket -> IO ()

Socket 是系统资源，用完必须关闭。和文件句柄一样，推荐用 ``bracket`` 包起来，保证异常发生时也会释放：

.. code:: haskell

   bracket (openSocket addr) close $ \sock -> do
     -- 在这里使用 sock
     ...

.. note::

   ``withSocketsDo`` 是为 Windows 准备的初始化函数，在其他平台上什么也不做。把 ``main`` 写成 ``main = withSocketsDo $ do ...`` 可以让程序跨平台运行，本章的例子都这样写。

发送与接收
--------------------------------------------------------------------------------

``Network.Socket`` 本身只提供基于 ``String`` 的收发函数，而且已经不推荐使用。实际的数据收发用 ``Network.Socket.ByteString`` 模块：

.. code:: haskell

   import Network.Socket.ByteString (recv, sendAll)

   sendAll :: Socket -> ByteString -> IO ()
   recv    :: Socket -> Int -> IO ByteString

- ``sendAll`` 会一直写，直到整个 ``ByteString`` 都发送出去；
- ``recv sock n`` 最多读取 ``n`` 个字节，有多少读多少。\ **返回空的 ByteString 表示对方已经关闭连接**\ 。

发送文本时先用 ``Data.Text.Encoding`` 转成 UTF-8 字节，收到的字节再解码回 ``Text``\ ：

.. code:: haskell

   import Data.Text.Encoding (decodeUtf8, encodeUtf8)

   sendAll sock (encodeUtf8 "你好\n")
   bytes <- recv sock 4096
   let text = decodeUtf8 bytes

.. warning::

   **TCP 是字节流，不是消息流**\ 。一次 ``sendAll`` 发出的数据，对方可能需要多次 ``recv`` 才能收齐，也可能一次 ``recv`` 收到两条消息的拼接。协议必须自己规定消息边界，常见做法有两种：每条消息以换行符结尾，或者在消息前面加一个固定长度的字节数。本章的例子都采用按行分隔。

基于行的处理
--------------------------------------------------------------------------------

对于按行分隔的文本协议，不需要自己在 ``recv`` 的结果里找换行符。\ ``socketToHandle`` 可以把 Socket 转换成一个普通的 ``Handle``\ ，然后就能用 IO 一章介绍的 ``hGetLine`` 和 ``hPutStrLn`` 读写：

.. code:: haskell

   import System.IO

   h <- socketToHandle sock ReadWriteMode
   hSetBuffering h LineBuffering   -- 每写完一行就发送，不用手动 hFlush
   line <- hGetLine h              -- 读到换行符为止
   hPutStrLn h "OK"
   hClose h                        -- 关闭 Handle 时 Socket 也随之关闭

转换之后原来的 ``Socket`` 值就不应再直接使用，收发和关闭都通过 ``Handle`` 进行。\ ``hIsEOF`` 可以判断对方是否已经关闭连接。

TCP 客户端
--------------------------------------------------------------------------------

下面是一个完整的客户端。它从命令行接收主机、端口和一条消息（\ ``getArgs`` 的用法见命令行一章），连接后把消息发送过去，再打印服务器的回复：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Control.Exception (bracket)
   import qualified Data.Text as T
   import Data.Text.Encoding (decodeUtf8, encodeUtf8)
   import qualified Data.Text.IO as TIO
   import Network.Socket
   import Network.Socket.ByteString (recv, sendAll)
   import System.Environment (getArgs)
   import System.Exit (exitFailure)
   import System.IO (hPutStrLn, stderr)

   main :: IO ()
   main = withSocketsDo $ do
     args <- getArgs
     case args of
       [host, port, message] -> runClient host port message
       _ -> do
         hPutStrLn stderr "用法: client HOST PORT MESSAGE"
         exitFailure

   runClient :: String -> String -> String -> IO ()
   runClient host port message = do
     -- 1. 解析地址，取第一个结果
     let hints = defaultHints { addrSocketType = Stream }
     addr <- firstAddr <$> getAddrInfo (Just hints) (Just host) (Just port)
     -- 2. 打开 Socket 并连接，bracket 保证连接一定会被关闭
     bracket (openSocket addr) close $ \sock -> do
       connect sock (addrAddress addr)
       -- 3. 发送一行文本，然后等待回复
       sendAll sock (encodeUtf8 (T.pack message <> "\n"))
       reply <- recv sock 4096
       TIO.putStr ("服务器回复: " <> decodeUtf8 reply)

   -- getAddrInfo 解析失败时会直接抛出异常，返回的列表不会为空
   firstAddr :: [AddrInfo] -> AddrInfo
   firstAddr (addr : _) = addr
   firstAddr [] = error "getAddrInfo 没有返回任何地址"

流程就是 Socket 编程的固定套路：解析地址、创建 Socket、\ ``connect``\ 、收发数据、关闭。下一节先写一个服务器，再回来用这个客户端测试。

TCP 服务器
--------------------------------------------------------------------------------

服务器端多了三步：\ ``bind`` 把 Socket 绑定到本机端口，\ ``listen`` 开始监听，\ ``accept`` 阻塞等待新的连接。每次 ``accept`` 返回一个新的 Socket 代表这条连接，原来的监听 Socket 继续接受下一个连接。

.. mermaid::

   graph LR
     subgraph server["服务器"]
       S1["openSocket"] --> S2["setSocketOption ReuseAddr"]
       S2 --> S3["bind"] --> S4["listen"] --> S5["accept 循环"]
       S5 -- "每个新连接" --> S6["forkFinally handleConn"]
       S6 --> S7["recv / sendAll"] --> S8["gracefulClose"]
       S5 -. "继续等待下一个连接" .-> S5
     end
     subgraph client["客户端"]
       C1["openSocket"] --> C2["connect"] --> C3["sendAll / recv"] --> C4["close"]
     end
     C2 -. "TCP 三次握手" .-> S5
     C3 <-. "字节流" .-> S7

下面是一个回显（echo）服务器，把每个客户端发来的数据原样发回去：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Control.Concurrent (forkFinally)
   import Control.Exception (bracket)
   import Control.Monad (forever, unless)
   import qualified Data.ByteString as BS
   import Network.Socket
   import Network.Socket.ByteString (recv, sendAll)
   import System.IO (BufferMode (..), hSetBuffering, stdout)

   main :: IO ()
   main = withSocketsDo $ do
     -- 日志输出到管道或文件时也按行刷新
     hSetBuffering stdout LineBuffering
     let hints = defaultHints { addrSocketType = Stream, addrFlags = [AI_PASSIVE] }
     addr <- firstAddr <$> getAddrInfo (Just hints) Nothing (Just "4000")
     bracket (openSocket addr) close $ \sock -> do
       -- 允许程序重启后立即复用端口
       setSocketOption sock ReuseAddr 1
       bind sock (addrAddress addr)
       listen sock 1024
       putStrLn "回显服务器监听 4000 端口"
       forever $ do
         (conn, peer) <- accept sock
         putStrLn $ "新连接: " ++ show peer
         -- 每个连接一个绿色线程，结束时（无论正常还是异常）关闭连接
         _ <- forkFinally (handleConn conn) (const $ gracefulClose conn 5000)
         return ()

   handleConn :: Socket -> IO ()
   handleConn conn = do
     chunk <- recv conn 4096
     -- recv 返回空串表示对方已经关闭连接
     unless (BS.null chunk) $ do
       sendAll conn chunk
       handleConn conn

   firstAddr :: [AddrInfo] -> AddrInfo
   firstAddr (addr : _) = addr
   firstAddr [] = error "getAddrInfo 没有返回任何地址"

几个值得说明的地方：

- ``setSocketOption sock ReuseAddr 1``\ ：服务器退出后，端口会在操作系统里保留一小段时间（TIME_WAIT 状态）。不设置这个选项，马上重启程序会报 "address already in use"。
- ``listen sock 1024``\ ：第二个参数是等待 ``accept`` 的连接队列长度。
- ``forkFinally``\ ：并发一章介绍的 ``forkIO`` 的变体，第二个参数是线程结束时执行的清理动作，无论线程正常返回还是抛出异常都会执行。这里用它保证每条连接最终都被关闭。
- ``gracefulClose conn 5000``\ ：先关闭写方向并等待对方确认（最多 5000 毫秒），再释放 Socket，比直接 ``close`` 更少丢数据。

**每个连接一个线程**\ 在 Haskell 里是合理的默认写法。并发一章讲过，GHC 的绿色线程初始只占约 1KB，等待网络数据时由 I/O Manager 挂起而不占用操作系统线程，几万个连接对应几万个绿色线程没有问题。不需要像 C 或 Java 那样为了省线程而手写事件循环或线程池。

编译时加上 ``-threaded`` 开启多线程运行时，让多个核心参与处理：

.. code:: sh

   $ ghc -O2 -threaded -rtsopts -with-rtsopts="-N" EchoServer.hs
   $ ./EchoServer
   回显服务器监听 4000 端口

在另一个终端用上一节的客户端测试：

.. code:: sh

   $ ./Client localhost 4000 "hello haskell"
   服务器回复: hello haskell
   $ ./Client localhost 4000 "你好"
   服务器回复: 你好

服务器这边会打印出每个连接的来源地址：

.. code:: text

   新连接: 127.0.0.1:45306
   新连接: 127.0.0.1:45320

如果机器上装了 ``nc``\ （netcat），也可以用 ``nc localhost 4000`` 交互式地测试，输入一行回车后会看到同样的内容被发回来。

.. tip::

   **如果你熟悉其他语言**\ ：

   - **Python 的 socket 模块**\ ：\ ``getAddrInfo`` / ``openSocket`` / ``bind`` / ``listen`` / ``accept`` 与 ``socket.getaddrinfo`` / ``socket.socket`` / ``bind`` / ``listen`` / ``accept`` 一一对应，\ ``socketToHandle`` 相当于 ``sock.makefile("rw")``\ 。区别在于 Python 里为每个连接开一个线程代价较高，通常要改用 ``asyncio``\ ；Haskell 里 ``forkFinally`` 就是绿色线程，直接开即可。
   - **Go 的 net 包**\ ：\ ``net.Listen`` 加上 ``for { conn := l.Accept(); go handle(conn) }`` 的写法与本节的 ``forever`` 加 ``forkFinally`` 几乎逐行对应。Goroutine 和 GHC 绿色线程是同一类设计。

共享状态：键值服务器
--------------------------------------------------------------------------------

回显服务器的每条连接彼此独立。更常见的情况是多条连接需要访问同一份数据，这正是并发一章 STM 的用武之地。下面的服务器实现一个极简的键值存储，支持 ``SET key value``\ 、\ ``GET key`` 和 ``DEL key`` 三条命令，所有连接共享一个 ``TVar (Map Text Text)``\ ：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Control.Concurrent (forkFinally)
   import Control.Concurrent.STM
   import Control.Exception (bracket)
   import Control.Monad (forever)
   import qualified Data.Map.Strict as Map
   import Data.Maybe (fromMaybe)
   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO
   import Network.Socket
   import System.IO

   type Store = TVar (Map.Map T.Text T.Text)

   main :: IO ()
   main = withSocketsDo $ do
     -- 日志输出到管道或文件时也按行刷新
     hSetBuffering stdout LineBuffering
     store <- newTVarIO Map.empty
     let hints = defaultHints { addrSocketType = Stream, addrFlags = [AI_PASSIVE] }
     addr <- firstAddr <$> getAddrInfo (Just hints) Nothing (Just "4001")
     bracket (openSocket addr) close $ \sock -> do
       setSocketOption sock ReuseAddr 1
       bind sock (addrAddress addr)
       listen sock 1024
       putStrLn "KV 服务器监听 4001 端口"
       forever $ do
         (conn, _) <- accept sock
         -- 把 Socket 转成 Handle，之后按行读写
         h <- socketToHandle conn ReadWriteMode
         hSetBuffering h LineBuffering
         _ <- forkFinally (serve store h) (const $ hClose h)
         return ()

   serve :: Store -> Handle -> IO ()
   serve store h = do
     eof <- hIsEOF h
     if eof
       then return ()
       else do
         line <- TIO.hGetLine h
         reply <- handleCommand store (T.words line)
         TIO.hPutStrLn h reply
         serve store h

   handleCommand :: Store -> [T.Text] -> IO T.Text
   handleCommand store cmd = case cmd of
     ["SET", key, value] -> do
       atomically $ modifyTVar' store (Map.insert key value)
       return "OK"
     ["GET", key] -> do
       m <- readTVarIO store
       return $ fromMaybe "(nil)" (Map.lookup key m)
     ["DEL", key] -> do
       atomically $ modifyTVar' store (Map.delete key)
       return "OK"
     _ -> return "ERR unknown command"

   firstAddr :: [AddrInfo] -> AddrInfo
   firstAddr (addr : _) = addr
   firstAddr [] = error "getAddrInfo 没有返回任何地址"

和回显服务器相比，变化有三处：

- 连接建立后立刻 ``socketToHandle``\ ，之后的读写和关闭都走 ``Handle``\ ，按行处理不再需要自己找换行符；
- ``serve`` 用 ``hIsEOF`` 判断客户端是否断开，否则读一行、处理、写一行回复，递归继续；
- 命令解析只是对 ``T.words`` 的结果做模式匹配（模式匹配一章），状态修改通过 ``atomically`` 完成，多个连接同时 ``SET`` 也不会互相破坏。

先后用两个客户端进程连接，可以看到状态在连接之间共享：

.. code:: sh

   $ ./Client localhost 4001 "SET lang haskell"
   服务器回复: OK
   $ ./Client localhost 4001 "GET lang"
   服务器回复: haskell
   $ ./Client localhost 4001 "GET nothing"
   服务器回复: (nil)

用 ``nc localhost 4001`` 保持一条连接，则可以连续输入多条命令：

.. code:: text

   SET lang haskell
   OK
   GET lang
   haskell
   DEL lang
   OK
   GET lang
   (nil)
   PING
   ERR unknown command

更高层的库
--------------------------------------------------------------------------------

直接操作 Socket 适合写自定义协议或理解底层机制，日常项目里更多是使用建立在 ``network`` 之上的库。写 HTTP 服务时，\ ``warp`` 是最常用的服务器实现，可以直接搭配 ``wai`` 接口使用，也可以通过 ``scotty``\ （轻量路由）或 ``servant``\ （用类型描述 API）这类框架来写。发起 HTTP 请求见上一章的 ``request`` 库。WebSocket 有 ``websockets`` 包。这些库的连接处理模型和本章一样，都是每个连接一个绿色线程。

小结
--------------------------------------------------------------------------------

- **地址解析**\ ：用 ``getAddrInfo`` 把主机名和端口解析成 ``AddrInfo``\ ，再用 ``openSocket`` 创建 Socket；服务器端加 ``AI_PASSIVE`` 标志。
- **资源管理**\ ：Socket 用 ``bracket`` 包起来，每个连接用 ``forkFinally`` 处理并在结束时 ``gracefulClose``\ 。
- **收发数据**\ ：用 ``Network.Socket.ByteString`` 的 ``sendAll`` 与 ``recv``\ ，\ ``recv`` 返回空串表示连接关闭；文本通过 ``encodeUtf8`` / ``decodeUtf8`` 转换。
- **消息边界**\ ：TCP 是字节流，按行分隔的协议用 ``socketToHandle`` 转成 ``Handle`` 后按行读写最方便。
- **并发模型**\ ：每个连接一个绿色线程，共享状态用 STM 保护，编译时开启 ``-threaded``\ 。

全书总结
--------------------------------------------------------------------------------

到这里全书结束。我们从值、函数与类型类的整体思路和类型系统开始，经过常用数据结构、Monoid / Functor / Applicative / Monad 这一组抽象，讨论了惰性求值、IO、并发和项目组织，最后用一组常用类库（文件与目录、命令行、JSON、HTTP、Socket）把这些内容落到日常程序里。这些内容覆盖了日常使用 Haskell 需要的大部分基础，更深入的主题（类型级编程、依赖注入的其他方案、各种效果系统）可以在此之上继续阅读。
