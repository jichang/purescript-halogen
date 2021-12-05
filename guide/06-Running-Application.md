# 运行应用

在这份指南中，已经出现了很多次运行 Halogen 应用的标准方式。在本章中，我们会学习当我们运行 Halogen 应用时到底发生了什么以及如何从外部控制应用。

## 使用 `runUI` 和 `awaitBody`

PureScript 应用使用`Main`中的`main`函数作为程序入口。下面是 Halogen 应用的标准`main`函数。

```purs
module Main where

import Prelude

import Effect (Effect)
import Halogen.Aff as HA
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  runUI component unit body

-- Assuming you have defined a root component for your application
component :: forall query input output m. H.Component query input output m
component = ...
```

`main`函数中最重要的就是 `runUI` 这个函数。调用 `runUI` 时，提供根组件、根组件输入以及一个 DOM 元素的引用，这函数会将组件交给 Halogen 虚拟 DOM。虚拟 DOM 会负责将应用渲染到根元素，并且会一直维护应用，直到结束。

```purs
runUI
  :: forall query input output
   . Component query input output Aff
  -> input
  -> DOM.HTMLElement
  -> Aff (HalogenIO query output Aff)
```

可以看到， `runUI` 函数要求应用可以运行在 `Aff` 单子中。在本指南中，我们一直使用的是 `MonadEffect` 和 `MonadAff` 这些约束，这些约束都是满足 `Aff` 的。

> 如果你选择使用其他的单子来运行应用，需要在调用 `runUI` 之前把这个单子提升到 `Aff` 。[Halogen 实战](https://github.com/thomashoneyman/purescript-halogen-realworld) 中使用了单子 `AppM`， 是一个很好的例子。

除了 `runUI`，我们还用了其他两个辅助函数。第一个是我们用 `awaitBody` 来等待页面加载完成然后获取 `<body>` 标签的索引，然后将其作为应用运行的根元素。第二个，我们用 `runHalogenAff` 在 `Effect` 中启动异步作用（指的是包含`awaitBody` 和 `runUI`的代码片段）。需要这样做的原因是 `awaitBody`、`runUI` 还有我们的应用代码都是运行在 `Aff` 单子中，但是 PureScript `main` 函数是运行在 `Effect` 单子中。

`main` 函数是我们运行一个 Halogen 应用的标准方式，一般页面上只有一个这一个应用在运行。但是，有时候，我们只想要 Halogen 接管页面的一部分，或者我们像运行多个 Halogen 应用。这种情况下，我们需要使用另一组辅助函数：

1. `awaitLoad` 会阻塞执行，直到文档已经加载完毕，就可以安全的获取页面上 HTML 元素的索引。
2. `selectElement` 可以用于在页面上选择特定的 HTML 元素。

## 使用 `HalogenIO`

当调用 `runUI` 运行 Halogen 应用时，会收到一个类型为 `DriverIO` 的结构体。这些函数可以用来从外部控制应用程序。概念上，类似于是整个应用的一个虚拟父组件。

```purs
type HalogenIO query output m =
  { query :: forall a. query a -> m (Maybe a)
  , messages :: Event output
  , dispose :: m Unit
  }
```

1. `query` 函数应该很熟悉，和父组件用来向子组件发送查询的 `H.query` 相似，用来通知组件执行某些操作或者请求某些信息。
2. `messages` 函数用于订阅组件的输出流，类似于我们提供给 `slot` 函数的回调函数，只不过这里我们是用来执行某种外部副作用。
3. `dispose` 函数用于挂起以及清理 Halogen 应用。调用此函数，会杀死所有创建的进程，关闭所有订阅等。

一个常见的模式是 Halogen 应用使用 `Route` 组件作为根组件，然后在 URL 改变时使用 `query` 函数触发应用内路由切换。你可以在这里看到一个完整的例子 [Real World Halogen `Main.purs` file](https://github.com/thomashoneyman/purescript-halogen-realworld/blob/master/src/Main.purs).

## 完整示例：使用 `HalogenIO` 控制按钮

你可以把下面这个例子复制到 [Try PureScript](https://try.purescript.org) ，查看如何使用 `HalogenIO` 控制应用的根组件。

```purs
module Example.Driver.IO.Main where

import Prelude

import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Console (log)
import Halogen (liftEffect)
import Halogen as H
import Halogen.HTML as HH
import Halogen.Aff as HA
import Halogen.HTML.Events as HE
import Halogen.HTML.Properties as HP
import Halogen.Subscription as HS
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  io <- runUI component unit body

  _ <- liftEffect $ HS.subscribe io.messages \(Toggled newState) -> do
    liftEffect $ log $ "Button was internally toggled to: " <> show newState
    pure Nothing

  state0 <- io.query $ H.mkRequest IsOn
  liftEffect $ log $ "The button state is currently: " <> show state0

  void $ io.query $ H.mkTell (SetState true)

  state1 <- io.query $ H.mkRequest IsOn
  liftEffect $ log $ "The button state is now: " <> show state1

-- Child component implementation

type Slot = H.Slot Query Message

data Query a
  = IsOn (Boolean -> a)
  | SetState Boolean a

data Message = Toggled Boolean

data Action = Toggle

type State = { enabled :: Boolean }

component :: forall i m. H.Component Query i Message m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , handleQuery = handleQuery
        }
    }

initialState :: forall i. i -> State
initialState _ = { enabled: false }

render :: forall m. State -> H.ComponentHTML Action () m
render state =
  let
    label = if state.enabled then "On" else "Off"
  in
    HH.button
      [ HP.title label
      , HE.onClick \_ -> Toggle
      ]
      [ HH.text label ]

handleAction :: forall m. Action -> H.HalogenM State Action () Message m Unit
handleAction = case _ of
  Toggle -> do
    newState <- H.modify \st -> st { enabled = not st.enabled }
    H.raise (Toggled newState.enabled)

handleQuery :: forall m a. Query a -> H.HalogenM State Action () Message m (Maybe a)
handleQuery = case _ of
  IsOn k -> do
    enabled <- H.gets _.enabled
    pure (Just (k enabled))
  SetState enabled a -> do
    H.modify_ (_ { enabled = enabled })
    pure (Just a)
```
