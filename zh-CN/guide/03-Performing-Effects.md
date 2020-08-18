# 执行外部作用

我们到现在已经介绍了很多内容，如何编写 Halogen HTML，如何定义可以相应用户交互的组件以及需要使用的类型。有了这些基础之后，我们就可以进入下一个编写应用的工具：执行外部副作用。

这一章我们会通过两个例子来讲述如何在组件中执行外部副作用：生成随机数和发送 HTTP 请求。当你知道如何执行副作用之后，你就走上了掌握 Halogen 之路了。

在开始之前，最重要的是知道我们只能在求值（evaluation）的时候才能执行副作用，也就意味着只能在函数使用了类型 `HalogenM` 的 `handleAction` 函数中。在应用初始化内部状态或者渲染过程中是不行的。因为我们只能在 `HalogenM` 中执行副作用，在介绍例子之前，我们首先简单介绍一下这个类型。

## `HalogenM` 类型

在上一章， `handleAction` 函数返回一个类型 `HalogenM`。下面是我们定义的 `handleAction` ：

```purs
handleAction :: forall output m. Action -> HalogenM State Action () output m Unit
```

`HalogenM` 是 Halogen 的关键，通常称为“求值”单子（"eval" monad）。借助这个单子，Halogen 支持了内部状态、线程、订阅等功能。但是其本身功能有限，支持 Halogen 特定的功能。实际上， Halogen 组件并没有内置外部副作用的支持。

相反，Halogen 允许你指定组件想要使用的单子。这样，你既获得了 `HalogenM` 的所有功能，也获得了你所选择的单子支持的功能。这一点是靠类型参数 `m` 指定的，`m` 代表了 "monad"。

如果组件只需要使用 Halogen 特定的功能，可以保持这个类型参数的开放性。例如计数器，只需要更新内部状态。如果一个组件需要执行副作用，则需要使用 `Effect` 或者 `Aff` 单子，或者可以使用自己定制的单子。

下面的 `handleAction` 函数既可以执行 `HalogenM` 提供中的函数 `modify_` ，而也可以执行 `Effect` 中有外部作用的函数:

```purs
handleAction :: forall o. Action -> HalogenM State Action () o Effect Unit
```

下面这个函数则可以使用来自 `HalogenM` 中的函数以及 `Aff` 中有副作用的函数：

```purs
handleAction :: forall output. Action -> HalogenM State Action () output Aff Unit
```

通常，在 Halogen 中会在类型参数 `m` 上使用类型约束来描述这个单子能够做什么，而不是使用特定的单子类型，这样，在应用功能复杂之后，你可以混合使用多个不同的单子。例如，很多 Halogen 应用使用 `Aff` 是都会使用类型：

```purs
handleAction :: forall output m. MonadAff m => Action -> HalogenM State Action () output m Unit
```

这样，你既可以和写死 `Aff` 类型一样，而且你还可以混合使用其他的类型约束。

最后一点：当你为组件选择了单子之后，这个单子类型会出现在 `HalogenM` 类型， `Component` 类型以及 `ComponentHTML` 类型中（如果使用了子组件）：

```purs
component :: forall query input output m. MonadAff m => H.Component query input output m

handleAction :: forall output m. MonadAff m => Action -> HalogenM State Action () output m Unit

-- We aren't using child components, so we don't have to use the constraint here, but
-- we'll learn about when it's required in the parent & child components chapter.
render :: forall m. State -> H.ComponentHTML Action () m
```

## 一个使用 `Effect` 的例子：随机数字

我们来创建一个简单的新组件，这个组件里，每次你点击按钮，都会生成一个新的随机数。阅读这个例子，注意我们是怎么使用和计数器组件相同的类型和函数的。时间长了，你就会习惯看一下状态、行为和 Halogen 组件中的类型就大致了解这个组件做了些什么，而且会逐渐习惯 `initialState`、 `render` 和 `handleAction`这些标准函数。

> 你可以把这个例子粘贴到 [Try Purescript](https://try.purescript.org) 中运行。你也可以在目录 `examples` 目录中查看运行这个例子的[完整代码](https://github.com/purescript-halogen/purescript-halogen/tree/master/examples/effects-effect-random)。

注意，我们并没有在 `initialState` 和 `render` 函数中执行副作用 -- 例如，我们初始化状态为 `Nothing` ，而不是生成一个随机数，但是我们可以在函数 `handleAction` 中执行副作用（使用了 `HalogenM` 类型）。

```purs
module Main where

import Prelude

import Data.Maybe (Maybe(..), maybe)
import Effect (Effect)
import Effect.Class (class MonadEffect)
import Effect.Random (random)
import Halogen as H
import Halogen.Aff (awaitBody, runHalogenAff)
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = runHalogenAff do
  body <- awaitBody
  runUI component unit body

type State = Maybe Number

data Action = Regenerate

component :: forall query input output m. MonadEffect m => H.Component HH.HTML query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
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
  Regenerate -> do
    newNumber <- H.liftEffect random
    H.modify_ \_ -> Just newNumber
```

可以看到，执行副作用的组件和普通逐渐没有太大车陂！基本上我们只需做两件事：

1. 为类型参数 `m` 和 `handleAction` 函数添加 `MonadEffect`。因为我们没有使用子组件，所以我们不需要为 `render` 函数添加类型约束。
2. 第一次使用了执行副作用的函数：来自 `Effect.Random` 模块的 `random` 函数。

让我们分解一下如何使用这个副作用的。

```purs
--                          [1]
handleAction :: forall output m. MonadEffect m => Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  Regenerate -> do
    newNumber <- H.liftEffect random -- [2]
    H.modify_ \_ -> Just newNumber   -- [3]
```

1. 为类型参数 `m` 添加了约束，表明我们支持任何单子，只要这个单子类型满足了 `MonadEffect` 约束，这是我们需要执行 `Effect` 函数的另外一种说法。
2. `random` 函数的类型是 `Effect Number`，但是我们不能直接使用：组件在声明时使用的是 _任何_ 可以 `Effect` 中执行副作用的单子。有些细微的差别，因为我们需要的是类型 `MonadEffect m => m Number` 而不是 `Effect` 。幸运的是，我们可以使用 `liftEffect` 函数把任意 `Effect` 类型转换成 `MonadEffect m => m` 。这是 Halogen 中的常见模式，所以在使用 `MonadEffect` 时需要时刻记住 `liftEffect` 。
3. `modify_` 函数用来更新内部状态，和其他状态更新函数一样，都是来自 `HalogenM` 。这里，我们用来把随机数更新到内部状态中。

这个例子很好的展示了如何交替使用 `Effect` 中函数和 Halogen 中的特定函数 `modify_`。接下来，我们看一下如何用相同的方式使用 `Aff` 中异步外部作用。

## 一个 `Aff` 例子：HTTP 请求

从网络上获取信息是常见的操作。例如，我们想要用 Github 的 API 获取用户信息。我们需要使用 [`affjax`](https://pursuit.purescript.org/packages/purescript-affjax) 来发送请求，而这个库也是依赖 `Aff` 来实现异步作用的。

这个例子会更加有趣，因为：我们会使用 `preventDefault` 函数来避免表单提交时页面刷新，这一步是在 `Effect` 单子中。因此，这个例子展示了我们如何混合使用不同的外部作用（ `Effect` 和 `Aff`）以及 Halogen 函数（`HalogenM`）。

> 和 Random 例子一样，你可以把这个例子复制粘贴到 [Try Purescript](https://try.purescript.org) 中查看。你也可以在代码仓库中运行这个[完整例子](https://github.com/purescript-halogen/purescript-halogen/tree/master/examples/effects-aff-ajax) 。

这个组件定义现在看起来应该很熟悉了。我们首先定义了 `State` 和 `Action` 类型，然后实现了 `initialState`, `render`, 和 `handleAction` 函数。最后我们通过 `H.mkComponent` 他们组合为组件定义然后转化为一个合法的组件。

再次注意，我们只是在 `handleAction` 函数中执行外部作用，在初始化状态和渲染 Halogen HTML 时没有执行任何副作用。

```purs
module Main where

import Prelude

import Affjax as AX
import Affjax.ResponseFormat as AXRF
import Data.Either (hush)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Aff.Class (class MonadAff)
import Halogen as H
import Halogen.Aff (awaitBody, runHalogenAff)
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.HTML.Properties as HP
import Halogen.VDom.Driver (runUI)
import Web.Event.Event (Event)
import Web.Event.Event as Event

main :: Effect Unit
main = runHalogenAff do
  body <- awaitBody
  runUI component unit body

type State =
  { loading :: Boolean
  , username :: String
  , result :: Maybe String
  }

data Action
  = SetUsername String
  | MakeRequest Event

component :: forall query input output m. MonadAff m => H.Component HH.HTML query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }

initialState :: forall i. i -> State
initialState _ = { loading: false, username: "", result: Nothing }

render :: forall m. State -> H.ComponentHTML Action () m
render st =
  HH.form
    [ HE.onSubmit \ev -> Just (MakeRequest ev) ]
    [ HH.h1_ [ HH.text "Look up GitHub user" ]
    , HH.label_
        [ HH.div_ [ HH.text "Enter username:" ]
        , HH.input
            [ HP.value st.username
            , HE.onValueInput \str -> Just (SetUsername str)
            ]
        ]
    , HH.button
        [ HP.disabled st.loading
        , HP.type_ HP.ButtonSubmit
        ]
        [ HH.text "Fetch info" ]
    , HH.p_
        [ HH.text (if st.loading then "Working..." else "") ]
    , HH.div_
        case st.result of
          Nothing -> []
          Just res ->
            [ HH.h2_
                [ HH.text "Response:" ]
            , HH.pre_
                [ HH.code_ [ HH.text res ] ]
            ]
    ]

handleAction :: forall output m. MonadAff m => Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  SetUsername username -> do
    H.modify_ _ { username = username, result = Nothing }

  MakeRequest event -> do
    H.liftEffect $ Event.preventDefault event
    username <- H.gets _.username
    H.modify_ _ { loading = true }
    response <- H.liftAff $ AX.get AXRF.string ("https://api.github.com/users/" <> username)
    H.modify_ _ { loading = false, result = map _.body (hush response) }
```

这里例子非常有意思，因为：

1. 混合使用了多个单子（ `preventDefault` 是 `Effect`， `AX.get` 是 `Aff`，而 `gets` 和 `modify_` 是 `HalogenM`）。我们可以在组件里使用 `liftEffect` 和 `liftAff` 来配合使用它们。
2. We only have one constraint, `MonadAff`. That's because anything that can be run in `Effect` can also be run in `Aff`, so `MonadAff` implies `MonadEffect`.
3. 我们在同一次求值过程中更改了多次内部状态。

最后一点特别重要：更新内部状态会触发组件渲染。所以在单次求值过程中：

1. 将 `loading` 设为 `true`，触发组件重新渲染，显示 "Working..."
2. 将 `loading` 设为 `false`， 更新 `result`，触发组件渲染，显示 result （如果 API 返回了的话）

值得注意的是，我们使用 `MonadAff` 时，发送请求并不会阻塞组建，卫门也不需要用回调函数来处理异步行为。我们在 `MakeRequest` 执行的计算会被挂起，知道我们得到了响应结果，然后会被重启执行第二次状态更新。

尽量只在必要的时候更新状态或者选择批量更新状态是个明智的选择（像我们调用一次 `modify_` 去同时更新 `loading` 和 `result` 字段），这样可以避免重复渲染组件。
