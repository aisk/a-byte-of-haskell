JSON 处理：aeson
================================================================================

JSON 是配置文件、HTTP API 和日志里最常见的数据格式。Haskell 社区处理 JSON 的事实标准是 ``aeson`` 库，它把 JSON 文本和 Haskell 记录类型之间的转换交给两个类型类，大多数情况下一行 ``deriving`` 加两行空实例就能完成。

本章介绍 ``aeson`` 的核心类型 ``Value``\ 、自动派生与手写实例、常见的字段名映射，以及在不知道结构时如何取字段。读完可以直接读写配置文件和调用返回 JSON 的接口。

安装与依赖
--------------------------------------------------------------------------------

``aeson`` 不随 GHC 附带，需要写进 ``.cabal`` 文件的 ``build-depends``\ 。它的输入输出类型是惰性 ``ByteString``\ ，字符串字段用 ``Text``\ ，所以这两个包也要列出：

.. code:: text

   build-depends:    base >= 4.14 && < 5
                   , aeson >= 2.0
                   , bytestring
                   , text

本章示例默认开启 ``OverloadedStrings``\ ，这样字符串字面量可以直接当作 ``Text`` 或 ``ByteString`` 使用（见字符串一章）。

Value：JSON 的通用表示
--------------------------------------------------------------------------------

``aeson`` 用一个代数数据类型描述任意 JSON 文档：

.. code:: haskell

   data Value
     = Object Object      -- 键值对，底层是 KeyMap Value
     | Array  Array       -- 有序数组，底层是 Vector Value
     | String Text
     | Number Scientific  -- 任意精度的十进制数
     | Bool   Bool
     | Null

三个最常用的函数：

.. code:: haskell

   decode       :: FromJSON a => BL.ByteString -> Maybe a
   eitherDecode :: FromJSON a => BL.ByteString -> Either String a
   encode       :: ToJSON a   => a -> BL.ByteString

``BL`` 指 ``Data.ByteString.Lazy``\ 。对严格 ``ByteString`` 有对应的 ``decodeStrict`` 和 ``eitherDecodeStrict``\ ；从文件读取时用 ``BL.readFile`` 即可。

在 GHCi 里先把 JSON 解码成 ``Value`` 看一看：

.. code:: haskell

   ghci> :set -XOverloadedStrings
   ghci> import Data.Aeson
   ghci> decode "{\"name\":\"Alice\",\"tags\":[\"a\",\"b\"],\"age\":30,\"active\":true,\"extra\":null}" :: Maybe Value
   Just (Object (fromList [("active",Bool True),("age",Number 30.0),("extra",Null),("name",String "Alice"),("tags",Array [String "a",String "b"])]))
   ghci> decode "{\"name\":" :: Maybe Value
   Nothing
   ghci> eitherDecode "{\"name\":" :: Either String Value
   Left "Unexpected end-of-input, expecting JSON value"
   ghci> import qualified Data.ByteString.Lazy.Char8 as BL
   ghci> BL.putStrLn (encode (object ["x" .= (1 :: Int), "y" .= ("hi" :: String)]))
   {"x":1,"y":"hi"}

``decode`` 失败只给 ``Nothing``\ ，\ ``eitherDecode`` 会附带错误位置和原因，调试时更有用。

不过日常代码里很少直接操作 ``Value``\ 。更常见的做法是定义一个记录类型，让 ``aeson`` 在它和 JSON 之间来回转换：

.. mermaid::

   graph LR
     REC["Haskell 记录<br/>User { name, age }"] -- "toJSON" --> VAL["Value<br/>Object (fromList [...])"]
     VAL -- "parseJSON" --> REC
     VAL -- "序列化" --> TXT["ByteString 文本<br/>{#quot;name#quot;:#quot;Alice#quot;,#quot;age#quot;:30}"]
     TXT -- "解析" --> VAL
     REC -. "encode" .-> TXT
     TXT -. "decode / eitherDecode" .-> REC

``encode`` 和 ``decode`` 是两步的组合：先经过 ``Value``\ ，再和文本互转。实现 ``ToJSON`` 与 ``FromJSON`` 实例时，处理的只是左边这一步。

自动派生实例
--------------------------------------------------------------------------------

``FromJSON`` 负责从 JSON 解析，\ ``ToJSON`` 负责生成 JSON。对普通记录类型，派生 ``Generic`` 后写两个空实例就够了：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Data.Aeson
   import Data.Text (Text)
   import GHC.Generics (Generic)
   import qualified Data.ByteString.Lazy.Char8 as BL

   data User = User
     { name  :: Text
     , age   :: Int
     , email :: Maybe Text
     } deriving (Show, Generic)

   instance FromJSON User
   instance ToJSON User

   main :: IO ()
   main = do
     -- 编码：Haskell 值 -> JSON 文本
     BL.putStrLn (encode (User "Alice" 30 (Just "alice@example.com")))
     BL.putStrLn (encode (User "Bob" 25 Nothing))

     -- 解码：JSON 文本 -> Haskell 值
     print (decode "{\"name\":\"Carol\",\"age\":41}" :: Maybe User)
     print (decode "{\"name\":\"Carol\"}" :: Maybe User)
     print (eitherDecode "{\"name\":\"Carol\"}" :: Either String User)

     -- 列表和嵌套结构直接可用
     print (decode "[{\"name\":\"A\",\"age\":1},{\"name\":\"B\",\"age\":2}]" :: Maybe [User])

输出：

.. code:: text

   {"age":30,"email":"alice@example.com","name":"Alice"}
   {"age":25,"email":null,"name":"Bob"}
   Just (User {name = "Carol", age = 41, email = Nothing})
   Nothing
   Left "Error in $: parsing Main.User(User) failed, key \"age\" not found"
   Just [User {name = "A", age = 1, email = Nothing},User {name = "B", age = 2, email = Nothing}]

几点说明：

- 记录的字段名就是 JSON 的键名。\ ``GHC2021`` 已经包含 ``DeriveGeneric``\ ，不需要单独开启。
- ``Maybe`` 字段是可选的：JSON 里缺失或为 ``null`` 都解析成 ``Nothing``\ ，编码时 ``Nothing`` 输出为 ``null``\ 。
- 非 ``Maybe`` 字段缺失就是错误，\ ``eitherDecode`` 的信息里会指出缺哪个键。
- ``[a]``\ 、\ ``Maybe a``\ 、嵌套记录、\ ``Map Text a`` 都有现成实例。键不固定的对象（比如 ``{"zh": "...", "en": "..."}``\ ）用 ``Map Text a`` 或 ``HashMap Text a`` 接收。

.. tip::

   **如果你熟悉其他语言**\ ：

   - 对比 Python 的 ``json.loads``\ ：Python 返回 ``dict``\ ，字段存不存在、是什么类型要在使用时才知道。\ ``aeson`` 在解码那一刻就按记录类型检查完毕，之后的代码拿到的是有明确类型的值。
   - 对比 Go 的 ``encoding/json``\ ：思路很接近，都是把 JSON 映射到结构体。Go 用 struct tag 改键名，\ ``aeson`` 用下一节的 ``Options``\ 。

字段名映射
--------------------------------------------------------------------------------

很多接口用 ``snake_case`` 键名，而 Haskell 记录习惯 ``camelCase``\ 。不必手写整个实例，给派生过程传一个 ``Options`` 即可：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Data.Aeson
   import Data.Text (Text)
   import GHC.Generics (Generic)
   import qualified Data.ByteString.Lazy.Char8 as BL

   data Repo = Repo
     { fullName        :: Text
     , stargazersCount :: Int
     , homepage        :: Maybe Text
     } deriving (Show, Generic)

   jsonOptions :: Options
   jsonOptions = defaultOptions
     { fieldLabelModifier = camelTo2 '_'   -- fullName -> full_name
     , omitNothingFields  = True           -- Nothing 字段不输出
     }

   instance FromJSON Repo where
     parseJSON = genericParseJSON jsonOptions

   instance ToJSON Repo where
     toJSON = genericToJSON jsonOptions

   main :: IO ()
   main = do
     print (eitherDecode "{\"full_name\":\"haskell/aeson\",\"stargazers_count\":1300}" :: Either String Repo)
     BL.putStrLn (encode (Repo "haskell/aeson" 1300 Nothing))

输出：

.. code:: text

   Right (Repo {fullName = "haskell/aeson", stargazersCount = 1300, homepage = Nothing})
   {"full_name":"haskell/aeson","stargazers_count":1300}

``fieldLabelModifier`` 是一个 ``String -> String`` 函数，作用在每个字段名上。\ ``camelTo2`` 是 ``aeson`` 自带的驼峰转下划线工具，也可以传 ``drop 4`` 之类的函数去掉前缀。\ ``omitNothingFields`` 让 ``Nothing`` 字段在输出中省略，而不是写成 ``null``\ 。

手写实例
--------------------------------------------------------------------------------

当 JSON 结构和记录不是一一对应，比如需要默认值、要从嵌套对象里取字段，就手写实例。解析这边的写法是固定套路：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Data.Aeson
   import Data.Text (Text)
   import qualified Data.ByteString.Lazy.Char8 as BL

   data User = User
     { userName  :: Text
     , userAge   :: Int
     , userEmail :: Maybe Text
     } deriving (Show)

   instance FromJSON User where
     parseJSON = withObject "User" $ \o ->
       User <$> o .:  "name"
            <*> o .:? "age" .!= 0
            <*> o .:? "email"

   instance ToJSON User where
     toJSON u = object
       [ "name"  .= userName u
       , "age"   .= userAge u
       , "email" .= userEmail u
       ]

   main :: IO ()
   main = do
     print (eitherDecode "{\"name\":\"Alice\"}" :: Either String User)
     print (eitherDecode "{\"age\":3}" :: Either String User)
     print (eitherDecode "{\"name\":\"Alice\",\"age\":\"thirty\"}" :: Either String User)
     BL.putStrLn (encode (User "Alice" 30 Nothing))

输出：

.. code:: text

   Right (User {userName = "Alice", userAge = 0, userEmail = Nothing})
   Left "Error in $: key \"name\" not found"
   Left "Error in $.age: parsing Int failed, expected Number, but encountered String"
   {"age":30,"email":null,"name":"Alice"}

涉及的几个操作符：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 名称
     - 作用
   * - ``withObject "User" f``
     - 确认输入是对象，是则把 ``Object`` 交给 ``f``\ ，否则报错并带上 ``"User"`` 作为提示
   * - ``o .: "key"``
     - 取必需字段，缺失或类型不符则失败
   * - ``o .:? "key"``
     - 取可选字段，结果是 ``Maybe``\ ，缺失或 ``null`` 得到 ``Nothing``
   * - ``p .!= 默认值``
     - 把 ``Maybe`` 的结果换成默认值
   * - ``"key" .= 值``
     - 生成一个键值对，交给 ``object`` 组装

``parseJSON`` 的结果类型是 ``Parser a``\ 。\ ``Parser`` 是 Applicative，所以 ``User <$> ... <*> ... <*> ...`` 就是 Applicative 一章里的写法：每个 ``o .: "..."`` 是一个可能失败的小解析，用 ``<*>`` 把它们按构造器的参数顺序拼起来，任何一步失败整个解析就失败，并且错误信息里带着路径（如上面的 ``$.age``\ ）。它也是 Monad，需要先取一个字段再决定后面怎么解析时可以用 ``do`` 记号。

枚举与和类型
--------------------------------------------------------------------------------

只有无参构造器的类型通常映射成字符串。\ ``withText`` 和 ``withObject`` 类似，确认输入是字符串后交给处理函数：

.. code:: haskell

   data Status = Active | Disabled
     deriving (Show, Eq)

   instance ToJSON Status where
     toJSON Active   = String "active"
     toJSON Disabled = String "disabled"

   instance FromJSON Status where
     parseJSON = withText "Status" $ \t -> case t of
       "active"   -> pure Active
       "disabled" -> pure Disabled
       other      -> fail ("unknown status: " ++ show other)

.. code:: text

   ghci> BL.putStrLn (encode [Active, Disabled])
   ["active","disabled"]
   ghci> eitherDecode "[\"active\",\"paused\"]" :: Either String [Status]
   Left "Error in $[1]: unknown status: \"paused\""

``fail`` 在 ``Parser`` 里表示解析失败，信息会原样进入 ``eitherDecode`` 的 ``Left``\ 。带参数的和类型也可以派生 ``Generic`` 实例，默认编码格式是 ``{"tag": "构造器名", "contents": ...}``\ ，可以通过 ``Options`` 的 ``sumEncoding`` 调整。

不定义类型的取值方式
--------------------------------------------------------------------------------

有时只想从一大段 JSON 里取一两个深层字段，为此定义一串记录类型并不划算。\ ``Data.Aeson.Types`` 提供的 ``parseMaybe`` 和 ``parseEither`` 可以直接对 ``Value`` 运行一个 ``Parser``\ ：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Data.Aeson
   import Data.Aeson.Types (parseMaybe, parseEither)
   import qualified Data.Aeson.KeyMap as KM
   import Data.Text (Text)

   raw :: Value
   raw = object
     [ "data" .= object
         [ "items" .= [ object ["id" .= (1 :: Int), "title" .= ("first" :: Text)]
                      , object ["id" .= (2 :: Int), "title" .= ("second" :: Text)]
                      ]
         , "total" .= (2 :: Int)
         ]
     ]

   titles :: Value -> Maybe [Text]
   titles = parseMaybe $ withObject "root" $ \o -> do
     d     <- o .: "data"
     items <- d .: "items"
     mapM (.: "title") items

   main :: IO ()
   main = do
     print (titles raw)
     print (parseEither (withObject "root" (.: "missing")) raw :: Either String Int)
     -- 直接在 Object 上查找
     case raw of
       Object o -> print (KM.lookup "data" o >>= \d -> case d of
                            Object d' -> KM.lookup "total" d'
                            _ -> Nothing)
       _ -> putStrLn "not an object"

输出：

.. code:: text

   Just ["first","second"]
   Left "Error in $: key \"missing\" not found"
   Just (Number 2.0)

``titles`` 里每个 ``.:`` 的结果类型由后面的用法推断：\ ``d`` 被继续当作对象取字段，所以它是 ``Object``\ ；\ ``items`` 传给了 ``mapM (.: "title")``\ ，所以它是 ``[Object]``\ 。

第二种方式是直接模式匹配 ``Value``\ 。\ ``aeson`` 2.x 中 ``Object`` 里装的是 ``KeyMap``\ ，键类型是 ``Key`` 而不是 ``Text``\ ，开了 ``OverloadedStrings`` 可以直接写字面量，从 ``Text`` 转换用 ``Data.Aeson.Key.fromText``\ 。这种写法更啰嗦，通常用于检查一层结构，深层取值还是 ``Parser`` 更顺手。

完整示例：读写配置文件
--------------------------------------------------------------------------------

把上面的内容串起来：读取一个配置文件，打印其中的字段，修改后写到另一个文件。配置文件 ``config.json``\ ：

.. code:: json

   {
     "appName": "demo",
     "listenPort": 8080,
     "debug": false,
     "database": {
       "host": "localhost",
       "port": 5432,
       "user": "app"
     },
     "allowedHosts": ["localhost", "example.com"]
   }

程序：

.. code:: haskell

   {-# LANGUAGE OverloadedStrings #-}

   import Data.Aeson
   import Data.Text (Text)
   import qualified Data.Text as T
   import GHC.Generics (Generic)
   import qualified Data.ByteString.Lazy as BL
   import System.Exit (exitFailure)

   data Database = Database
     { host :: Text
     , port :: Int
     , user :: Text
     } deriving (Show, Generic)

   instance FromJSON Database
   instance ToJSON Database

   data Config = Config
     { appName      :: Text
     , listenPort   :: Int
     , debug        :: Bool
     , database     :: Database
     , allowedHosts :: [Text]
     } deriving (Show, Generic)

   instance FromJSON Config
   instance ToJSON Config

   main :: IO ()
   main = do
     raw <- BL.readFile "config.json"
     case eitherDecode raw of
       Left err -> do
         putStrLn ("config.json 解析失败: " ++ err)
         exitFailure
       Right cfg -> do
         putStrLn ("应用名: " ++ T.unpack (appName cfg))
         putStrLn ("端口: " ++ show (listenPort cfg))
         putStrLn ("数据库: " ++ T.unpack (host (database cfg)) ++ ":" ++ show (port (database cfg)))
         putStrLn ("允许的主机: " ++ show (allowedHosts cfg))
         -- 修改一个字段后写回另一个文件
         let cfg' = cfg { debug = True, allowedHosts = allowedHosts cfg ++ ["127.0.0.1"] }
         BL.writeFile "config.debug.json" (encode cfg')
         putStrLn "已写入 config.debug.json"

运行：

.. code:: text

   $ cabal run
   应用名: demo
   端口: 8080
   数据库: localhost:5432
   允许的主机: ["localhost","example.com"]
   已写入 config.debug.json

   $ cat config.debug.json
   {"allowedHosts":["localhost","example.com","127.0.0.1"],"appName":"demo","database":{"host":"localhost","port":5432,"user":"app"},"debug":true,"listenPort":8080}

两点补充：

- ``eitherDecode raw`` 的目标类型由后面 ``appName cfg`` 等用法推断出来是 ``Config``\ ，不需要写类型标注。
- ``encode`` 的输出是紧凑的单行。需要缩进格式时用 ``aeson-pretty`` 包的 ``encodePretty``\ ，用法相同。

小结
--------------------------------------------------------------------------------

- **核心函数**\ ：\ ``decode`` / ``eitherDecode`` 把惰性 ``ByteString`` 解析成任何 ``FromJSON`` 类型，\ ``encode`` 反向。调试时优先用 ``eitherDecode``\ ，错误信息带路径。
- **自动派生**\ ：记录类型派生 ``Generic``\ ，加上空的 ``FromJSON`` 与 ``ToJSON`` 实例即可。\ ``Maybe`` 字段自动成为可选。
- **字段名映射**\ ：用 ``genericParseJSON`` 与 ``genericToJSON`` 传入 ``Options``\ ，\ ``fieldLabelModifier`` 改键名，\ ``omitNothingFields`` 省略空字段。
- **手写实例**\ ：\ ``withObject`` 加 ``.:``\ 、\ ``.:?``\ 、\ ``.!=`` 组合出解析器，写法就是 Applicative 风格；生成端用 ``object`` 与 ``.=``\ 。
- **临时取值**\ ：不想定义类型时，用 ``parseMaybe`` 或 ``parseEither`` 对 ``Value`` 跑一个 ``Parser``\ 。
