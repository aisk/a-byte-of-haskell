HTTP 请求：request 库
================================================================================

调用 HTTP 接口是日常程序最常见的网络操作。Haskell 里底层的 HTTP 客户端是 ``http-client`` 及其 TLS 支持包 ``http-client-tls``\ ，功能完整但接口偏底层。本章使用在它们之上封装的 `request <https://hackage.haskell.org/package/request>`_ 库，它的接口参考了 Python 的 ``requests``\ ：一个 ``Request`` 记录、一个 ``Response`` 记录、一个 ``send`` 函数，加上 ``get`` / ``post`` 等快捷方式。请求体和响应体的编码解码由类型决定，配合上一章的 ``aeson`` 可以直接把 JSON 接口的返回值解析成记录类型。

安装与依赖
--------------------------------------------------------------------------------

.. code:: text

   build-depends:    base >= 4.14 && < 5
                   , request >= 0.4
                   , aeson
                   , text
                   , bytestring

``request`` 的记录类型 ``Request`` 和 ``Response`` 都有 ``headers`` 和 ``body`` 字段，因此推荐开启两个语言扩展：

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}

``OverloadedRecordDot``\ （GHC 9.2 起可用）允许用 ``resp.status`` 这样的点语法访问字段，\ ``DuplicateRecordFields`` 允许两个记录类型共用字段名。这两个扩展不在 ``GHC2021`` 里，需要显式写出。如果不想开启，库也提供了 ``responseStatus``\ 、\ ``responseBody``\ 、\ ``responseHeaders`` 以及 ``requestMethod`` 等前缀式访问函数，后文会给出对照写法。

本章示例使用 `httpbin.org <https://httpbin.org/>`_ 作为测试服务，它会把收到的请求原样描述在响应里，便于观察。

发起 GET 请求
--------------------------------------------------------------------------------

最短的例子：

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}

   import Network.HTTP.Request

   main :: IO ()
   main = do
     resp <- get "https://httpbin.org/get" :: IO (Response String)
     print resp.status
     mapM_ print resp.headers
     putStrLn resp.body

输出（部分响应头省略）：

.. code:: text

   200
   ("Content-Type","application/json")
   ("Content-Length","279")
   ("Server","gunicorn/19.9.0")
   {
     "args": {},
     "headers": {
       "Accept-Encoding": "gzip",
       "Host": "httpbin.org",
       "User-Agent": "haskell-request/0.4.2.0"
     },
     "origin": "203.0.113.10",
     "url": "https://httpbin.org/get"
   }

注意 ``:: IO (Response String)`` 这个类型标注。\ ``get`` 的签名是：

.. code:: haskell

   get :: FromResponseBody a => String -> IO (Response a)

响应体类型 ``a`` 由调用方决定，库根据它选择解码方式。\ ``FromResponseBody`` 的内置实例有 ``String``\ 、\ ``Text``\ 、严格和惰性 ``ByteString``\ ，以及任何实现了 ``FromJSON`` 的类型。如果后面的代码已经能推断出 ``a``\ （比如把 ``resp.body`` 传给了一个接收 ``Text`` 的函数），标注可以省略；否则编译器会报类型不明确的错误，加上标注即可。

Request 与 Response 记录
--------------------------------------------------------------------------------

``get`` 只是 ``send`` 的快捷方式。完整的请求用 ``Request`` 记录描述：

.. code:: haskell

   data Method = DELETE | GET | HEAD | OPTIONS | PATCH | POST | PUT | TRACE | Method String

   type Header  = (ByteString, ByteString)
   type Headers = [Header]

   data Request a = Request
     { method  :: Method
     , url     :: String
     , headers :: Headers
     , body    :: a
     }

   data Response a = Response
     { status  :: Int
     , headers :: Headers
     , body    :: a
     }

   send :: (ToRequestBody a, FromResponseBody b) => Request a -> IO (Response b)

请求体类型 ``a`` 决定发送时怎么编码，响应体类型 ``b`` 决定收到后怎么解码：

.. mermaid::

   graph LR
     subgraph REQ["Request a（ToRequestBody）"]
       direction TB
       R1["() 无请求体"]
       R2["String / Text / ByteString<br/>text/plain"]
       R3["任何 ToJSON 类型<br/>application/json"]
       R4["Form a<br/>x-www-form-urlencoded"]
     end
     REQ -- "send" --> RESP
     subgraph RESP["Response b（FromResponseBody）"]
       direction TB
       B1["String / Text / ByteString<br/>原文"]
       B2["任何 FromJSON 类型<br/>自动解码"]
       B3["Value<br/>结构未知时"]
       B4["StreamBody ByteString<br/>分块读取"]
     end

需要自定义请求头时，直接构造 ``Request``\ ：

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}
   {-# LANGUAGE OverloadedStrings #-}

   import Network.HTTP.Request
   import qualified Data.Text as T
   import qualified Data.Text.IO as TIO

   main :: IO ()
   main = do
     let req = Request
           { method  = GET
           , url     = "https://httpbin.org/headers"
           , headers = [("User-Agent", "a-byte-of-haskell/1.0"), ("Accept", "application/json")]
           , body    = ()
           }
     resp <- send req :: IO (Response T.Text)
     print resp.status
     TIO.putStrLn resp.body

.. code:: text

   200
   {
     "headers": {
       "Accept": "application/json",
       "Accept-Encoding": "gzip",
       "Host": "httpbin.org",
       "User-Agent": "a-byte-of-haskell/1.0"
     }
   }

请求头是 ``ByteString`` 二元组的列表，所以这里需要 ``OverloadedStrings``\ 。没有请求体时 ``body`` 填 ``()``\ 。

不开启记录扩展时，同样的程序写成：

.. code:: haskell

   import Network.HTTP.Request

   main :: IO ()
   main = do
     resp <- send (Request GET "https://httpbin.org/get" [] ()) :: IO (Response String)
     print (responseStatus resp)
     putStrLn (responseBody resp)

``Request`` 的四个参数按 ``method``\ 、\ ``url``\ 、\ ``headers``\ 、\ ``body`` 的顺序位置传入。

JSON 请求与响应
--------------------------------------------------------------------------------

这是 ``request`` 与 ``aeson`` 配合最方便的地方。响应体类型写成一个 ``FromJSON`` 记录，库就会自动解码；请求体传一个 ``ToJSON`` 值，库自动编码并设置 ``Content-Type: application/json``\ ：

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}
   {-# LANGUAGE OverloadedStrings #-}

   import Network.HTTP.Request
   import Data.Aeson (FromJSON, ToJSON, Value, encode)
   import Data.Text (Text)
   import GHC.Generics (Generic)
   import qualified Data.ByteString.Lazy.Char8 as BL

   -- httpbin.org/uuid 返回 {"uuid": "..."}
   newtype UUID = UUID { uuid :: Text }
     deriving (Show, Generic)

   instance FromJSON UUID

   -- 要发送的数据
   data NewUser = NewUser
     { name :: Text
     , age  :: Int
     } deriving (Show, Generic)

   instance ToJSON NewUser
   instance FromJSON NewUser

   -- httpbin.org/post 会把收到的 JSON 原样放在 "json" 字段里返回
   newtype Echo = Echo { json :: NewUser }
     deriving (Show, Generic)

   instance FromJSON Echo

   main :: IO ()
   main = do
     -- 响应体直接解码成记录
     r1 <- get "https://httpbin.org/uuid" :: IO (Response UUID)
     print r1.status
     print r1.body.uuid

     -- 请求体是 ToJSON 类型，自动编码并设置 Content-Type: application/json
     r2 <- post "https://httpbin.org/post" (NewUser "Alice" 30) :: IO (Response Echo)
     print r2.status
     print r2.body.json

     -- 不清楚结构时，先解码成 Value 看一看
     r3 <- get "https://httpbin.org/json" :: IO (Response Value)
     BL.putStrLn (encode r3.body)

.. code:: text

   200
   "cb6f8d3c-6f06-4477-b03f-f805b9f4eda8"
   200
   NewUser {name = "Alice", age = 30}
   {"slideshow":{"author":"Yours Truly","date":"date of publication","slides":[...],"title":"Sample Slide Show"}}

``post``\ 、\ ``put``\ 、\ ``patch`` 的签名都是 ``String -> a -> IO (Response b)``\ ，第二个参数是请求体。传 ``String`` 或 ``Text`` 时按 ``text/plain`` 发送，传 ``ToJSON`` 类型时按 JSON 发送。响应体不确定长什么样时，先用 ``Value`` 接收并打印出来，再据此定义记录类型。

.. tip::

   **如果你熟悉 Python requests**\ ：

   - ``requests.get(url).json()`` 对应 ``get url :: IO (Response Value)`` 或直接指定记录类型。区别是 Python 在调用 ``.json()`` 时才解析且不检查结构，Haskell 在 ``get`` 返回时就已按类型解析完成。
   - ``requests.post(url, json=data)`` 对应 ``post url data``\ ，只要 ``data`` 的类型有 ``ToJSON`` 实例。
   - ``requests.post(url, data={...})`` 对应下一节的 ``Form``\ 。

表单与认证
--------------------------------------------------------------------------------

传统的表单提交（登录接口、OAuth 的 token 接口）用 ``application/x-www-form-urlencoded`` 编码。把键值对列表包在 ``Form`` 里即可，库会负责百分号编码：

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}
   {-# LANGUAGE OverloadedStrings #-}

   import Network.HTTP.Request

   main :: IO ()
   main = do
     -- 表单提交，Content-Type 为 application/x-www-form-urlencoded
     r1 <- post "https://httpbin.org/post"
                (Form [("username", "alice"), ("password", "s3cret")])
             :: IO (Response String)
     print r1.status

     -- HTTP Basic 认证
     let req = basicAuth "alice" "s3cret"
                 (Request GET "https://httpbin.org/basic-auth/alice/s3cret" [] ())
     r2 <- send req :: IO (Response String)
     print r2.status

     -- Bearer Token
     let req' = bearerAuth "my-token" (Request GET "https://httpbin.org/bearer" [] ())
     r3 <- send req' :: IO (Response String)
     print r3.status
     putStrLn r3.body

.. code:: text

   200
   200
   200
   {
     "authenticated": true,
     "token": "my-token"
   }

``Form`` 之所以是一个单独的包装类型，是为了区分两种编码：\ ``post url x`` 走 JSON，\ ``post url (Form x)`` 走表单。自定义记录类型也可以实现 ``ToForm`` 类，返回 ``[(ByteString, ByteString)]``\ 。

``basicAuth`` 和 ``bearerAuth`` 都是 ``Request a -> Request a`` 的纯函数，只是往请求头里加一条 ``Authorization``\ ，可以和其他修改请求的函数随意组合。

错误处理
--------------------------------------------------------------------------------

三种情况需要分开对待：

1. 服务器返回了非 2xx 状态码。\ **这不是异常**\ ，\ ``send`` 正常返回，由调用方检查 ``status``\ 。
2. 网络层面失败，比如域名不存在、连接被拒绝、TLS 握手失败。这会抛出 ``http-client`` 的 ``HttpException``\ 。
3. 响应体解码失败，比如期望 JSON 但服务器返回了 HTML。这会抛出 ``aeson`` 的 ``AesonException``\ 。

后两种都用 IO 一章介绍的 ``try`` 捕获。\ ``HttpException`` 类型没有被 ``request`` 重新导出，不想额外依赖 ``http-client`` 的话，直接捕获 ``SomeException``\ ：

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}

   import Network.HTTP.Request
   import Control.Exception (try, SomeException)
   import Data.Aeson (Value)

   main :: IO ()
   main = do
     -- 1. 非 2xx 状态码不会抛异常，需要自己检查
     r1 <- get "https://httpbin.org/status/404" :: IO (Response String)
     if r1.status == 200
       then putStrLn "成功"
       else putStrLn ("服务器返回 " ++ show r1.status)

     -- 2. 连接失败、域名不存在等网络错误会抛异常
     r2 <- try (get "https://no-such-host.invalid/" :: IO (Response String))
     case r2 of
       Left (e :: SomeException) -> putStrLn ("请求失败: " ++ takeWhile (/= '\n') (show e))
       Right resp -> print resp.status

     -- 3. 响应不是合法 JSON 时，解码失败也会抛异常
     r3 <- try (get "https://httpbin.org/html" :: IO (Response Value))
     case r3 of
       Left (e :: SomeException) -> putStrLn ("解码失败: " ++ take 60 (show e))
       Right resp -> print resp.status

.. code:: text

   服务器返回 404
   请求失败: HttpExceptionRequest Request {
   解码失败: AesonException "Unexpected \"<!DOCTYPE html>\\n<html>\\n  <h

``Left (e :: SomeException)`` 这种在模式里写类型的语法来自 ``ScopedTypeVariables``\ ，\ ``GHC2021`` 已包含。

自动解码有一个实际问题：接口出错时返回的 JSON 结构往往和成功时不同，比如 GitHub 的 404 响应是 ``{"message": "Not Found", ...}``\ 。如果响应体类型写成成功时的记录，遇到错误就会因为解码失败而抛异常，状态码反而看不到了。需要按状态码区分处理时，先用 ``ByteString`` 接收原文，检查状态码后再手动调用 ``eitherDecode``\ 。最后的完整示例采用这种写法。

流式下载
--------------------------------------------------------------------------------

下载大文件时不希望把整个响应体读进内存。把响应体类型写成 ``StreamBody ByteString``\ ，\ ``send`` 会在收到响应头后立刻返回，之后通过 ``readNext`` 逐块读取：

.. code:: haskell

   data StreamBody a = StreamBody
     { readNext    :: IO (Maybe a)   -- 下一块数据，流结束时返回 Nothing
     , closeStream :: IO ()          -- 关闭连接
     }

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}

   import Network.HTTP.Request
   import qualified Data.ByteString as BS
   import System.IO

   main :: IO ()
   main = do
     resp <- get "https://httpbin.org/bytes/100000" :: IO (Response (StreamBody BS.ByteString))
     print resp.status
     withFile "download.bin" WriteMode $ \h -> do
       let loop total = do
             chunk <- resp.body.readNext
             case chunk of
               Nothing -> return total
               Just bs -> do
                 BS.hPut h bs
                 loop (total + BS.length bs)
       n <- loop (0 :: Int)
       putStrLn ("已写入 " ++ show n ++ " 字节")
     resp.body.closeStream

.. code:: text

   200
   已写入 100000 字节

同样的机制还支持 ``StreamBody SseEvent``\ ，用于读取 Server-Sent Events 流，每次 ``readNext`` 返回一个完整事件。

默认情况下所有请求共用一个全局连接池。需要独立的连接池或自定义代理、证书等设置时，用 ``newManager`` 创建一个 ``Manager``\ ，再用 ``sendWith manager req`` 代替 ``send req``\ 。

完整示例：查询 GitHub 仓库信息
--------------------------------------------------------------------------------

从命令行接收一个 ``owner/repo`` 形式的参数（\ ``getArgs`` 的用法见上一章），调用 GitHub API 取回仓库信息并打印。GitHub 要求请求带 ``User-Agent`` 头，否则返回 403。

.. code:: haskell

   {-# LANGUAGE OverloadedRecordDot #-}
   {-# LANGUAGE DuplicateRecordFields #-}
   {-# LANGUAGE OverloadedStrings #-}

   import Network.HTTP.Request
   import Data.Aeson (FromJSON, eitherDecode)
   import qualified Data.ByteString.Lazy as BL
   import qualified Data.ByteString.Lazy.Char8 as BLC
   import Data.Text (Text)
   import qualified Data.Text as T
   import GHC.Generics (Generic)
   import System.Environment (getArgs, getProgName)
   import System.Exit (exitFailure)
   import System.IO (hPutStrLn, stderr)

   -- 字段名与 GitHub API 返回的 JSON 键一致，不需要额外配置
   data Repo = Repo
     { full_name        :: Text
     , description      :: Maybe Text
     , stargazers_count :: Int
     , language         :: Maybe Text
     } deriving (Show, Generic)

   instance FromJSON Repo

   -- 先取回原始字节，根据状态码决定是否解码
   fetchRepo :: String -> IO (Either String Repo)
   fetchRepo name = do
     resp <- send Request
       { method  = GET
       , url     = "https://api.github.com/repos/" ++ name
       , headers = [("User-Agent", "a-byte-of-haskell")]   -- GitHub 要求带 User-Agent
       , body    = ()
       } :: IO (Response BL.ByteString)
     return $ if resp.status == 200
       then eitherDecode resp.body
       else Left ("GitHub 返回 " ++ show resp.status ++ ": " ++ BLC.unpack resp.body)

   main :: IO ()
   main = do
     args <- getArgs
     case args of
       [name] -> do
         result <- fetchRepo name
         case result of
           Left err -> do
             hPutStrLn stderr err
             exitFailure
           Right repo -> do
             putStrLn ("仓库: " ++ T.unpack repo.full_name)
             putStrLn ("简介: " ++ maybe "(无)" T.unpack repo.description)
             putStrLn ("语言: " ++ maybe "(未知)" T.unpack repo.language)
             putStrLn ("Star: " ++ show repo.stargazers_count)
       _ -> do
         prog <- getProgName
         hPutStrLn stderr ("用法: " ++ prog ++ " owner/repo")
         exitFailure

运行：

.. code:: text

   $ cabal run repo-info -- haskell/aeson
   仓库: haskell/aeson
   简介: A fast Haskell JSON library
   语言: Haskell
   Star: 1307

   $ cabal run repo-info -- nobody/no-such-repo
   GitHub 返回 404: {"message":"Not Found","documentation_url":"https://docs.github.com/rest/repos/repos#get-a-repository","status":"404"}

   $ echo $?
   1

``fetchRepo`` 把网络请求和结果判断收在一起，返回 ``Either String Repo``\ ，这样 ``main`` 只需要处理两个分支。JSON 的键名直接用作记录字段名，省去了 ``fieldLabelModifier``\ ；如果更在意 Haskell 侧的命名风格，改用上一章的 ``Options`` 映射即可。

小结
--------------------------------------------------------------------------------

- **三个核心概念**\ ：\ ``Request a`` 描述请求，\ ``Response b`` 描述响应，\ ``send`` 发送。\ ``get`` / ``post`` / ``put`` / ``patch`` / ``delete`` 是快捷方式。
- **类型决定编解码**\ ：请求体是 ``String`` / ``Text`` / ``ByteString`` 时按纯文本发送，是 ``ToJSON`` 类型时按 JSON 发送，包在 ``Form`` 里时按表单发送。响应体类型由 ``:: IO (Response 类型)`` 标注指定，\ ``FromJSON`` 类型会自动解码。
- **错误处理**\ ：非 2xx 状态码不抛异常，检查 ``status``\ ；网络错误和解码错误抛异常，用 ``try`` 捕获。需要按状态码区分时先用 ``ByteString`` 接收再手动解码。
- **认证与流**\ ：\ ``basicAuth`` 与 ``bearerAuth`` 修改请求头；\ ``StreamBody`` 逐块读取大响应。
