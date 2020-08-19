# 生命周期和订阅

我们现在学过的概念已经能够包含编写 Halogen 组件所需的大部分知。大多数组件有内部状态、渲染 HTML 以及响应用户的点击、鼠标悬停或者其他在 HTML 上的交互行为。

其实行为也可以从其他时间里触发，如下是几个常见的例子：

1. 当组件启动时需要执行某种行为（例如执行某种外部作用来初始化状态）或者当组件从 DOM 中移除时（例如释放申请到的资源）。这些被称为**生命周期事件**。
2. 你需要定期的执行某个行为（例如每隔 10 秒钟执行更新），或者当事件发生在生成的 HTML 之外时（例如，当用户按下按钮时执行某种行为），又或者你需要处理某些第三方组件的事件（例如某些文本编辑器）。这些可以通过**事件源订阅**机制处理，又是我们也叫做**订阅机制**或者**事件源机制**。

在下一章学习父子组件时，我们会学习另外注意红触发行为的方式。这一章我们会主要关注生命周期和订阅机制。

## 生命周期事件

每个 Halogen 组件可以使用两个生命周期事件：

1. 组件初始化时可以执行某些行为（Halogen 创建组件时）
2. 组件销毁时可以执行某些行为（Halogen 移除组件时）

我们在 `eval` 函数中声明组件初始化和销毁时需要执行的行为，和我们提供 `handleAction` 函数是相同的。在下一章，我们会讨论 `eval` 的详细细节，这里我们先看一下生命周期的实际应用。

下面这个例子和前面随机数的例子几乎一样，有几个重要的不同指出：

1. 除了 `Regenerate`，我们还添加了 `Initialize` 和 `Finalize` 行为
2. 我们扩展了 `eval` ，添加了 `initialize` 字段，表明组件初始化时需要执行 `Initialize` 行为，同时添加了另外一个 `finalize` 字段，声明组件销毁时需要执行 `Finalize` 行为。
3. 因为我们添加了两个新的行为，所以我们在 `handleAction` 函数里添加了两个 `case` 语句来处理这个两个行为。

尝试阅读一下代码：

```purs
module Main where

import Prelude

import Data.Maybe (Maybe(..), maybe)
import Effect (Effect)
import Effect.Class (class MonadEffect)
import Effect.Class.Console (log)
import Effect.Random (random)
import Halogen as H
import Halogen.Aff as HA
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.HTML as HH
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  runUI component unit body

type State = Maybe Number

data Action
  = Initialize
  | Regenerate
  | Finalize

component :: forall query input output m. MonadEffect m => H.Component HH.HTML query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , initialize = Just Initialize
        , finalize = Just Finalize
        }
    }

initialState :: forall input. input -> State
initialState _ = Nothing

render :: forall m. State -> H.ComponentHTML Action () m
render state = do
  let value = maybe "No number generated yet" show state
  HH.div_
    [ HH.h1_
        [ HH.text "Random number" ]
    , HH.p_
        [ HH.text ("Current value: " <> value) ]
    , HH.button
        [ HE.onClick \_ -> Just Regenerate ]
        [ HH.text "Generate new number" ]
    ]

handleAction :: forall output m. MonadEffect m => Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  Initialize -> do
    handleAction Regenerate
    newNumber <- H.get
    log ("Initialized: " <> show newNumber)

  Regenerate -> do
    newNumber <- H.liftEffect random
    H.put (Just newNumber)

  Finalize -> do
    number <- H.get
    log ("Finalized! Last number was: " <> show number)
```

当组件初始化时，我们会生成一个随机数然后输出到命令行中。接下来我们会在用户点击按钮之后生成随机数，最后当组件被销毁时，我们会将组件状态中的随机数输出到命令行中。

另一个有意思的改动是：在处理 `Initialize` 时，我们调用了 `handleAction Regenerate` -- 也即是我们递归调用了 `handleAction` 函数。递归调用其他行为处理时很方便的，和我们这里做的一样。当然我们也可以内联 `Regenerate` 的处理函数，像下面的代码这样：

```purs
  Initialize -> do
    newNumber <- H.liftEffect random
    H.put (Just newNumber)
    log ("Initialized: " <> show newNumber)
```

在我们介绍订阅和事件源之前，我们先来看一下 `eval` 函数。

## `eval` 函数， `mkEval` 以及 `EvalSpec`

在迄今为止我们介绍的所有组件中都使用了 `eval` 函数，但是我们只使用了 `handleAction` 函数来处理组件内部触发的行为。但是 `eval` 函数可以描述组件在响应事件时求值 `HalogenM` 的所有方式。

在多数情况下，你不需要关注组件规范以及求值规范里的所有类型和函数，但是我们还是会分解涉及到的类型，以便你能了解到底是怎么工作的。

`mkComponent` 函数接收一个 `ComponentSpec` 做参数，这个参数包含三个字段：

```purs
H.mkComponent
  { initialState :: input -> state
  , render :: state -> H.ComponentHTML action slots m
  , eval :: H.HalogenQ query action input ~> H.HalogenM state action slots output m
  }
```

我们之前已经花精力介绍过了 `initialState` 和 `render` 两个函数。但是 `eval` 函数看起来不一样 -- 什么是 `HalogenQ`以及函数 `handleAction` 是怎么配合进来的？现在我们只需要关注这个函数的常见使用方式，可以在概念索引（未完成）中找到详细细节。

`eval` 函数描述了如何处理组件内部触发的事件。通常我们通过使用一个 `EvalSpec` 值当参数调用 `mkEval` 函数来生成这个函数，和我们通过使用 `ComponentSpec` 当参数调用 `mkComponent` 生成 `Component` 一样。

为了方便，Halogen 提供了一个完整的 `EvalSpec` 实现，就是 `defaultEval`，这个默认的实现在组件事件时不执行任何操作。使用这个默认实现，你可以选择性的覆盖掉你需要的值，其余部分则不会做任何事情。

下面是我们如何定义一个只处理行为的 `eval` 函数：

```purs
H.mkComponent
  { initialState
  , render
  , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
  }

-- assuming we've defined a `handleAction` function in scope...
handleAction = ...
```

_注意_: `initialState` 和 `render` 函数使用 _record pun_ 定义的，但是 `handleAction` 却不能使用这种方式，原因它时 _结构体更新_ 的一部分。更多信息，请参考 [结构体语言索引](https://github.com/purescript/documentation/blob/master/language/Records.md#record-update).

如果需要，你可以重写其他的字段。例如，如果你需要支持初始化操作，你可以重写 `initialize` 字段：

```purs
H.mkComponent
  { initialState
  , render
  , eval: H.mkEval $ H.defaultEval
      { handleAction = handleAction
      , initialize = Just Initialize
      }
  }
```

让我们看一下 `EvalSpec` 的完整类型：

```purs
type EvalSpec state query action slots input output m =
  { handleAction :: action -> HalogenM state action slots output m Unit
  , handleQuery :: forall a. query a -> HalogenM state action slots output m (Maybe a)
  , initialize :: Maybe action
  , receive :: input -> Maybe action
  , finalize :: Maybe action
  }
```

`EvalSpec` 包含了组件中涉及到的所有类型。不过好在你不需要声明这个类型 -- 你可以提供给 `mkEval` 一个结构体。我们会在后面的章节里介绍 `handleQuery` 和 `receive` 函数，当然还有 `query` 和 `output` 类型，因为这些类型和子组件有关。

正常情况下，我们都只需要重写 `defaultEval` 中的某个字段，而不是重新实现一个完整的 `EvalSpec`，让我们来看一下 `defaultEval` 是如何实现每个函数的：

```purs
defaultEval =
  { handleAction: const (pure unit)
  , handleQuery: const (pure Nothing) -- we'll learn about this when we cover child components
  , initialize: Nothing
  , receive: const Nothing -- we'll learn about this when we cover child components
  , finalize: Nothing
  }
```

现在，让我们看一下另外一个内部时间的来源：订阅事件源。

## 订阅

有时，我们需要处理一些并不是在我们渲染的 Halogen HTML 中触发的事件或者某种定期触发的事件。两个常见的例子，一是时间相关的事件，另外一个则是发生在另外一个 Halogen DOM 树之外节点的事件（例如 window）。

在 Halogen 中，这些事件都来自 **事件源** 。组件可以订阅这些事件源，指定当这些事件发生时需要触发的行为。

事件源通常使用以下方法来创建：

1. `effectEventSource` 和 `affEventSource` 这两个函数可以从 `Effect` 或则 `Aff` 函数中创建事件源。
2. `eventListenerEventSource` 可以通过在指定 DOM 节点添加事件监听器来创建一个事件源，例如监听浏览器窗口的 resize 事件。

事件源可以被当成一个行为的流：事件流随时可以生成行为，组件只要订阅了这个事件源，就需要执行这些行为。通常情况下，都是在组件初始化时创建并且订阅事件，虽然我们可以在任何订阅或者取消订阅事件源。

让我们看两个事件源的实例：一个是基于 `Aff` 的定时器，会对组件初始化后的秒数进行计数，另外一个则是基于事件监听的事件流，他会报告文档中的键盘事件。

### 实现一个定时器

我们的第一个例子是基于 `Aff` 的定时器，每一秒实现自加一。

```purs
module Main where

import Prelude

import Control.Monad.Rec.Class (forever)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Aff (Milliseconds(..))
import Effect.Aff as Aff
import Effect.Aff.Class (class MonadAff)
import Effect.Exception (error)
import Halogen as H
import Halogen.Aff as HA
import Halogen.HTML as HH
import Halogen.Query.EventSource (EventSource)
import Halogen.Query.EventSource as EventSource
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  runUI component unit body

data Action = Initialize | Tick

type State = Int

component :: forall query input output m. MonadAff m => H.Component HH.HTML query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , initialize = Just Initialize
        }
    }

initialState :: forall input. input -> State
initialState _ = 0

render :: forall m. State -> H.ComponentHTML Action () m
render seconds = HH.text ("You have been here for " <> show seconds <> " seconds")

handleAction :: forall output m. MonadAff m => Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  Initialize -> do
    _ <- H.subscribe timer
    pure unit

  Tick ->
    H.modify_ \state -> state + 1

timer :: forall m. MonadAff m => EventSource m Action
timer = EventSource.affEventSource \emitter -> do
  fiber <- Aff.forkAff $ forever do
    Aff.delay $ Milliseconds 1000.0
    EventSource.emit emitter Tick

  pure $ EventSource.Finalizer do
    Aff.killFiber (error "Event source finalized") fiber
```

大部分代码应该都很熟悉，但是还是有些新知识。

首先，我们定义了一个事件源，这个事件源每秒钟都会触发一个 `Tick` 行为，直到事件源被关闭。

```purs
timer :: forall m. MonadAff m => EventSource m Action
timer = EventSource.affEventSource \emitter -> do
  fiber <- Aff.forkAff $ forever do
    Aff.delay $ Milliseconds 1000.0
    EventSource.emit emitter Tick

  pure $ EventSource.Finalizer do
    Aff.killFiber (error "Event source finalized") fiber
```

`affEventSource` 和 `effectEventSource` 函数接受一个回调函数，这个回调函数的参数是一个 `Emitter` 值。以这个值作为参数，可以调用 `EventSource.emit` 函数广播一个行为，也可以调用 `EventSource.close` 函数关闭这个事件流（这样我们可以直接在内部关闭这个事件源，而不是等到订阅者关闭或者组件销毁时自动关闭）。 我们可以返回要给清理函数，这个清理函数在事件源关闭时会被调用，被称作终结器。在这个例子里，我们会杀死我们用来计数的 Fiber。

第二部，我们使用了 Halogen 中的 `subscribe` 函数订阅这个事件源：

```purs
  Initialize -> do
    _ <- H.subscribe timer
    pure unit
```

`subscribe` 函数接受一个事件源作为参数，会返回一个 `SubscriptionId`。你可以将 `SubscriptionId` 传递到 `unsubscribe` 函数来关闭这个事件源，执行它的清理函数，这样你就不会监听到这事件源的输出。组件在被销毁时会自动取消所有事件源的订阅，不需要手动执行取消订阅的操作。

刚兴趣的话可以看一下[Ace 编辑器的例子](https://github.com/purescript-halogen/purescript-halogen/tree/master/examples/ace)，这个例子里，我们订阅了第三方组件里的事件，用这些事件触发 Halogen 组件的行为。

### 使用事件监听器做事件源

另外一个需要使用事件源的场景时当需要响应某些 DOM 事件，但是这些 DOM 元素并不是 Halogen 可以直接控制的。例如我们需要 HTML 文档元素自身的一些事件，例如滚动、键盘等。

在下面的例子里，我们订阅的文档中的键盘事件，然后保存了所有在长按 `Shift` 的同事点击的按键，当点击回车键时取消监听事件。这个例子演示了如何使用 `eventListenerEventSource` 函数来绑定事件监听器以及使用 `H.unsubscribe` 函数取消订阅。

在示例代码仓库里也有一个响应的[键盘输入例子](https://github.com/purescript-halogen/purescript-halogen/tree/master/examples/keyboard-input)。

```purs
module Main where

import Prelude

import Data.Maybe (Maybe(..))
import Data.String as String
import Effect (Effect)
import Effect.Aff.Class (class MonadAff)
import Halogen as H
import Halogen.Aff as HA
import Halogen.HTML as HH
import Halogen.Query.EventSource (eventListenerEventSource)
import Halogen.VDom.Driver (runUI)
import Web.Event.Event as E
import Web.HTML (window)
import Web.HTML.HTMLDocument as HTMLDocument
import Web.HTML.Window (document)
import Web.UIEvent.KeyboardEvent as KE
import Web.UIEvent.KeyboardEvent.EventTypes as KET

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  runUI component unit body

type State = { chars :: String }

data Action
  = Initialize
  | HandleKey H.SubscriptionId KE.KeyboardEvent

component :: forall query input output m. MonadAff m => H.Component HH.HTML query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , initialize = Just Initialize
        }
    }

initialState :: forall input. input -> State
initialState _ = { chars: "" }

render :: forall m. State -> H.ComponentHTML Action () m
render state =
  HH.div_
    [ HH.p_ [ HH.text "Hold down the shift key and type some characters!" ]
    , HH.p_ [ HH.text "Press ENTER or RETURN to clear and remove the event listener." ]
    , HH.p_ [ HH.text state.chars ]
    ]

handleAction :: forall output m. MonadAff m => Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  Initialize -> do
    document <- H.liftEffect $ document =<< window
    H.subscribe' \sid ->
      eventListenerEventSource
        KET.keyup
        (HTMLDocument.toEventTarget document)
        (map (HandleKey sid) <<< KE.fromEvent)

  HandleKey sid ev
    | KE.shiftKey ev -> do
        H.liftEffect $ E.preventDefault $ KE.toEvent ev
        let char = KE.key ev
        when (String.length char == 1) do
          H.modify_ \st -> st { chars = st.chars <> char }

    | KE.key ev == "Enter" -> do
        H.liftEffect $ E.preventDefault (KE.toEvent ev)
        H.modify_ _ { chars = "" }
        H.unsubscribe sid

    | otherwise ->
        pure unit
```

在这个例子中，我们使用了 `H.subscribe'` 函数，这个函数会传递 `SubscriptionId` 到事件源，而不是返回它。这样我们可以把 ID 保存在行为，而不是内部状态中，会更加方便。

我们把事件源相关逻辑实现在处理 `Initialize` 行为的代码中，包括在 HTML 文档注册事件监听器以及在处理键盘事件时触发 `HandleKey` 行为。

`eventListenerEventSource` 和其他事件源有一些不同。它使用了来自 `purescript-web` 库中的类型和函数，手动在 DOM 中注册事件监听器。

```
eventListenerEventSource
  :: forall action m
   . MonadAff m
  => EventType
  -> EventTarget
  -> (Event -> Maybe action)
  -> EventSource m action
```

这个函数的参数包含：监听的事件类型（在这个例子里时 `keyup`）、监听的目标（在这个例子里是 `HTMLDocument` 元素）、回调函数，这个回调函数负责将事件转化为一个需要触发的类型（在这个例子里，我们触发了 `Action` 类型的值，这个包含了捕获的时间）。

## 总结

Halogen 组件使用 `Action` 类型来处理组件内的各种事件。这些事件可能发生的方式包括：

1. 用户在我们渲染的 HTML 元素上执行了交互操作
2. 声明周期事件
3. 事件源，依赖 `Aff` 和 `Effect` 函数或者 DOM 上的事件监听器。

现在，你已经了解了单个 Halogen 组件所必须的知识。在下一章，我们将会学习如何组合多个 Halogen 组件，构建一个父子组件树。
