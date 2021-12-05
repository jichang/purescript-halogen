# 父子组件

Halogen 是一个没有特定倾向的 UI 库：你可以在没有特定架构要求下创建用户界面。

目前为止，我们的应用都只是包含单个 Halogen 组件。你可以像 Elm 架构那样，构建一个单组件应用，然后逐步把 `handleAction` 和 `render` 函数拆分到独立的模块中。

不过， Halogen 也支持任意深度的组件树。这意味任意组件都可以包含其他更多子组件，每个子组件都可以有自己的状态和行为。大多数 Halogen 组件都是用这种架构，包括 [实战 Halogen](https://github.com/thomashoneyman/purescript-halogen-realworld) 中的应用。

当涉及到多个组件时，我们就需要一种组件间进行通信的机制。Halogen 提供了三种父子组件通信的方式：

1. 父组件可以向子组件发送 _查询（queries）_ ，通知子组件执行某种操作或者获取子组件内部信息。
2. 父组件可以向子组件传递 _输入（input）_ ，每当父组件重新渲染时都会重新发送。
3. 子组件可以像父组件发送 _输出消息_ ，通知父组件某种事件发生。

这些类型参数都在 `Component` 类型中，有些也出现在 `ComponentHTML` 和 `HalogenM` 类型里。例如，一个支持查询、输入和输出消息的组件类型：

```purs
component :: forall m. H.Component Query Input Output m
```

可以把组件和其他组件交流的方式当作一种 _公开接口_，公共接口会定义在 `Component` 类型

在这一章我们会学习：

1. 如何在 Halogen HTML 中渲染其他组件
2. 组件间通信的三种方式：查询、输入和输出消息
3. 组件插槽， `slot` 函数以及 `Slot` 类型，借助这些类型能够让通信做到类型安全

我们首先渲染一个没有查询和输出消息的子组件。然后，我们会用这三种方式进行通信，构建一个涉及到所有通信方式的父子组件例子。

建议将这些例子加载到 [Try Purescript](https://try.purescript.org)，探索一下本章介绍到的每种通信方式。

## 渲染组件

在本指南开始的章节里，我们首先学习的是编写渲染 Halogen HTML 元素的函数。这些函数可以被用来构建更大的 HTML 元素树。

编写组件时，我们首先要写的是 `render` 函数。虽然组件可以维持内部状态以及执行外部作用等等，但从概念上说，所谓组件就是通过这个 render 函数生成 Halogen HTML 。

虽然到现在为止，我们在 `render` 函数中只有使用 HTML 元素，但其实也可以使用 _components_ 来生成 HTML。虽然这个说法不是特别的准确，但是这有助于理解在实现组件 render 函数时如何使用 components。

当一个组件渲染其他组件时，我们称它为父组件，被渲染的组件则称为子组件。

让我们看一下除了 HTML 元素之外，如何在 `render` 函数中渲染组件。首先，我们写一个渲染按钮的辅助函数。然后，我们会把这个辅助函数转化成一个组件，我们也要修改一下父组件去渲染这个子组件。

首先，我们编写一个使用辅助函数渲染 HTML 的组件：

```purs
module Main where

import Prelude

import Halogen as H
import Halogen.HTML as HH

parent :: forall query input output m. H.Component query input output m
parent =
  H.mkComponent
    { initialState: identity
    , render
    , eval: H.mkEval H.defaultEval
    }
  where
  render :: forall state action. state -> H.ComponentHTML action () m
  render _ = HH.div_ [ button { label: "Click Me" } ]

button :: forall w i. { label :: String } -> HH.HTML w i
button { label } = HH.button [ ] [ HH.text label ]
```

这段代码包含一个渲染 `div` 的简单组件以及一个辅助函数 `button`，这个辅助函数通过输入参数中的标签渲染一个按钮。注意，组件 `parent` 使用类型参数声明了内部状态和行为类型，因为组件自身并没有内部状态，也没有任何行为。

现在，让我们把 `button` 函数转化成一个组件来做演示（在实际应用中，通常不会针对这么简单的组件这么做）：

```purs
type Input = { label :: String }

type State = { label :: String }

button :: forall query output m. H.Component query Input output m
button =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval H.defaultEval
    }
  where
  initialState :: Input -> State
  initialState input = input

  render :: forall action. State -> H.ComponentHTML action () m
  render { label } = HH.button [ ] [ HH.text label ]
```

我们用以下几步把 button 函数转换成了一个组件：

1. 把辅助函数的参数转换成组件的 `Input` 类型。父组件负责提供输入，在后边的小节中会进行介绍输入。
2. 我们把 HTML 代码移动到组件的 `render` 函数。 因为 `render` 函数只能访问组件`State` 类型的内部状态，所以在 `initialState` 函数中，我们要把输入值复制到内部状态中，然后才能在渲染时使用。把输入拷贝到内部状态中是 Halogen 中一个常见的模式，同时要注意，`render` 没有声明具体的行为类型（因为组件没有任何行为）而且因为没有子组件，我们使用了 `()`。
3. 我们使用了 `defaultEval` 作为 `EvalSpec`，因为这个组件并不需要响应任何内部事件 -- 组件没有任何行为，也没有使用任何生命周期函数。

但是父组件现在其实是有问题的！如果你照着做，会看到如下错误：

```purs
[1/1 TypesDoNotUnify]

  16    render _ = HH.div_ [ button { label: "Click Me" } ]
                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  Could not match type

    Component HTML t2 { label :: String }

  with type

    Function
```

组件不能和函数一样，给定输入之后直接调用进行渲染。因为组件除了生成正常的 HTML 之外，还可以和父组件进行通信。因此，组件需要额外的信息才能像正常的元素一样被渲染。

组件都会在 HTML 树中占据一个插槽（位置）。插槽是一个组件在被移除前生成 Halogen HTML 的位置。插槽处的组件可以被当作是一个有状态的动态 HTML 元素。我们可以混合使用动态组件和普通的 Halogen HTML 元素，但是动态组件的运行需要更多的信息。

这里更多信息则是来自 `slot` 函数和 `ComponentHTML`中的插槽类型，至今为止，我们一直使用的是 `()`。我们很快就会讨论如何利用插槽渲染组件，不过首先要让代码能编译通过。

我们可以通过使用 `slot` 函数，在指定插槽内渲染组件来解决 `render` 函数的编译问题。当然，我们需要更新 `ComponentHTML` 中的插槽类型来包含我们支持的组件。下面的更改显示了渲染 HTML 元素和渲染组件的区别。

```diff
+ import Type.Proxy (Proxy(..))
+
+ type Slots = ( button :: forall query. H.Slot query Void Int )
+
+ _button = Proxy :: Proxy "button"

  parent :: forall query input output m. H.Component query input output m
  parent =
    H.mkComponent
      { initialState: identity
      , render
      , eval: H.mkEval H.defaultEval
      }
    where
-   render :: forall state action. state -> H.ComponentHTML action () m
+   render :: forall state action. state -> H.ComponentHTML action Slots m
    render _ =
-     HH.div_ [ button { label: "Click Me" } ]
+     HH.div_ [ HH.slot_ _button 0 button { label: "Click Me" } ]
```

现在父组件可以渲染子组件 -- 按钮组件。渲染一个子组件有两个大的改动：

1. 使用 `slot` 函数渲染组件，这个函数接受一些参数，有些我们还没有接触到。其中两个参数，一个是 `button` 组件，一个是需要的输入标签。
2. 添加了新类型 `Slots`，这个新类型包含了一个标签 `button`，其值为类型 `H.Slot`。我们在 `ComponentHTML` 使用了这个新类型，而不是我们之前一直看到的 `()`。

`slot` 函数和 `Slot` 类型可以用来渲染一个有状态、可执行外部副作用的子组件，就和渲染其他 HTML 元素一样。但是，为什么需要这么多参数和类型呢？为什么我们不能直接用对应的输入调用 `button` 函数呢？

原因是 Halogen 提供了两种父子组件间通信的方式，我们需要保证通信方式的类型安全。`slot` 函数让我们可以：

1. 确定如何通过标签（类型层面的字符串 "button"，在值层面，我们使用了 `SProxy :: SProxy "button"` 来表示）和一个唯一的标识符（数字 0）来标识特定的组件，我们就可以向组件发送 _查询_ 。
2. 渲染组件（`button`），提供组件所需的 _输入_ （`{ label: "Click Me" }`），每次父组件重新渲染时，输入会被重新发送到子组件以防输入随着时间产生改变。
3. 决定如何处理子组件的 _输出消息_ , slot 函数可以指定一个处理子组件输出的函数， slot\_ 函数用在子组件没有任何输出或者想要忽略掉子组件输出的场景。这就是子组件主动和父组件通信的方式。

借助`slot`、`_slot` 函数和 `H.Slot` 类型，可以用类型安全的方式管理三种通信机制。在本章剩余部分，我们会专注在父子组件如何通信，同时我们会学习插槽以及相关的类型。

## 组件间通信

当从使用单个组件转到使用多个组件时，你很快就需要某种方式来实现组件间通信。在 Halogen 中，有三种父子组件间通信的方式：

1. 父组件可以为子组件提供输入。每当父组件渲染时，父组件会向子组件发送输入，由子组件决定如何处理新输入。
2. 子组件可以向父组件发送消息，类似于我们介绍过的事件源。在重要事件发生时，子组件可以通过发送消息来改变父组件状态，例如一个弹出框被关闭或者表单提交之后，父组件可以决定如何处理。
3. 父组件可以查询子组件，可以告诉子组件执行某些操作，也可以查询子组件的某些内部信息。父组件决定何时发送何种查询请求，而子组件决定如何处理这些查询请求。

这三种机制提供了组件间通信的多种方式。我们首先来看一下这三种机制，然后我们看一下 `slot` 函数和插槽类型，看一下这些是如何做到类型安全的。

### 输入

父组件可以给子组件提供输入，而且每次渲染时都会重新发送输入。我们已经看过了很多次 -- `input` 类型用于生成子组件内部状态。在本章的那个例子里，按钮组件就是从父组件获得了文本标签。

到现在为止，我们只使用输入生成初始状态。但是输入并不是一次性的，在每次渲染的时候都会收到新的输入，子组件可以用通过 `receive` 函数来决定如何处理新的收入。

```purs
receive :: input -> Maybe action
```

`receive` 函数应该会让你联想到 `initialize` 和 `finalize` 函数，这两个函数可以用来在组件创建和销毁时执行某个行为。同样的， `receive` 函数可以指定在父组件发送输入时应该执行何种行为。

默认状态下，Halogen 的 `defaultSpec` 并没有提供收到新输入时应该执行的行为。如果你的子组件不希望在收到初始输入后做任何事，你可以不用做任何改动。例如，按钮收到文本标签并复制到状态后，没有任何必要监听输入的变化。

父组件渲染时，子组件能收到新的输入是一个很有用的特性。这意味着我们可以声明式给子组件提供输入。虽然父组件可以通过其他方式和子组件通信，但是声明式输入在多数情况下都是最优的选择。

我们重新看一下我们的例子，更具体的看一下这个问题。在这个版本，按钮组件没有任何变化 -- 它接受一个文本作为输入然后用这个文本初始化内部状态 -- 但是父组件需要做些改动。父组件在初始化时启动一个定时器，每秒增加一个计数值，然后使用这个计数值作为按钮的文本标签。

简单说，按钮的输入每秒钟都会被重新发送。复制这个例子到 [Try PureScript](https://try.purescript.org) ，看一下发生了什么 -- 按钮的文本标签按时不是会每秒钟更新一次？

```purs
module Main where

import Prelude

import Control.Monad.Rec.Class (forever)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Aff (Milliseconds(..))
import Effect.Aff as Aff
import Effect.Aff.Class (class MonadAff)
import Halogen as H
import Halogen.Aff (awaitBody, runHalogenAff)
import Halogen.HTML as HH
import Halogen.Subscription as HS
import Halogen.VDom.Driver (runUI)
import Type.Proxy (Proxy(..))

main :: Effect Unit
main = runHalogenAff do
  body <- awaitBody
  runUI parent unit body

type Slots = ( button :: forall q. H.Slot q Void Unit )

_button = Proxy :: Proxy "button"

type ParentState = { count :: Int }

data ParentAction = Initialize | Increment

parent :: forall query input output m. MonadAff m => H.Component query input output m
parent =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , initialize = Just Initialize
        }
    }
  where
  initialState :: input -> ParentState
  initialState _ = { count: 0 }

  render :: ParentState -> H.ComponentHTML ParentAction Slots m
  render { count } =
    HH.div_ [ HH.slot_ _button unit button { label: show count } ]

  handleAction :: ParentAction -> H.HalogenM ParentState ParentAction Slots output m Unit
  handleAction = case _ of
    Initialize -> do
      { emitter, listener } <- H.liftEffect HS.create
      void $ H.subscribe emitter
      void
        $ H.liftAff
        $ Aff.forkAff
        $ forever do
            Aff.delay $ Milliseconds 1000.0
            H.liftEffect $ HS.notify listener Increment
    Increment -> H.modify_ \st -> st { count = st.count + 1 }

-- Now we turn to our child component, the button.

type ButtonInput = { label :: String }

type ButtonState = { label :: String }

button :: forall query output m. H.Component query ButtonInput output m
button =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval H.defaultEval
    }
  where
  initialState :: ButtonInput -> ButtonState
  initialState { label } = { label }

  render :: forall action. ButtonState -> H.ComponentHTML action () m
  render { label } = HH.button_ [ HH.text label ]
```

如果把这段代码加载到 Try PureScript，你会发现我们的按钮。。。没有任何变化！虽然父组件每次都重新发送了输入（每次父组件渲染时），但是子组件从来没有收到过。只是收到输入是不够的，我们还要指定每次收到新输入是应该执行何种操作。

将按钮组件的代码替换为如下内容，看看差别是什么：

```purs
data ButtonAction = Receive ButtonInput

type ButtonInput = { label :: String }

type ButtonState = { label :: String }

button :: forall query output m. H.Component query ButtonInput output m
button =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , receive = Just <<< Receive
        }
    }
  where
  initialState :: ButtonInput -> ButtonState
  initialState { label } = { label }

  render :: ButtonState -> H.ComponentHTML ButtonAction () m
  render { label } = HH.button_ [ HH.text label ]

  handleAction :: ButtonAction -> H.HalogenM ButtonState ButtonAction () output m Unit
  handleAction = case _ of
    -- When we receive new input we update our `label` field in state.
    Receive input ->
      H.modify_ _ { label = input.label }
```

为了保证能和父组件的最新输入同步，我们做了一些操作：

1. 添加了一个新的行为 `Receive`，接受 `Input` 类型值作为参数。 我们在 `handleAction` 函数中处理这个行为时会更新内部状态。
2. 我们在求值规范里添加了一个新的字段 `receive`，这个字段是一个函数，每次有新输入时都会调用这个函数。这个函数返回了 `Receive` 行为，我们会在 `handleAction` 中处理。

这些改动能让子组件订阅到父组件的新输入。现在，按钮应该就会每秒钟更新一次了。作为一个练习，你可以把 `receive` 函数替换为 `const Nothing`，这是按钮组件又会忽略新的输入了。（因为没有返回任何行为，handleAction 也就不会被调用更新内部状态）

### 输出消息

有时，有些事件发生后不应该由子组件处理。

例如，我们写了一个弹窗组件，当用户点击关闭按钮时，我们需要执行一些行为。为了更加灵活，我们希望让父组件决定窗口关闭时应该做什么。

在 Halogen 里，为了处理这种情况，我们选择让弹窗组件（子组件）向父组件触发一个 **输出消息** 。父组件可以像处理其他行为一样，在 `handleAction` 函数里处理这些消息。概念上，子组件就像是一个事件源，而且父组件还会自动订阅。

具体点说，弹窗组件可以向父组件触发 `Closed` 事件。父组件可以更新内部状态，标识弹窗不应该被展示了，下次渲染周期时，弹窗就会从 DOM 中被移除。

作为简单例子，我们考虑如何设计一个按钮，这个按钮在点击时让父组件决定要执行的操作。

```purs
module Button where

-- This component can notify parent components of one event, `Clicked`
data Output = Clicked

-- This component can handle one internal event, `Click`
data Action = Click

-- Our output type shows up in our `Component` type
button :: forall query input m. H.Component query input Output m
button =
  H.mkComponent
    { initialState: identity
    , render
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }
  where
  render _ =
    HH.button
      [ HE.onClick \_ -> Click ]
      [ HH.text "Click me" ]

  -- Our output type also shows up in our `HalogenM` type, because this is
  -- where we can emit these output messages.
  handleAction :: forall state. Action -> H.HalogenM state Action () Output m Unit
  handleAction = case _ of
    -- When the button is clicked we notify the parent component that the
    -- `Clicked` event has happened by emitting it with `H.raise`.
    Click ->
      H.raise Clicked
```

我们用了几步实现输出消息。

1. 添加了 `Output` 类型，描述组件可能触发的消息。这个类型算是组件的公共接口的一部分，同时，因为输出消息是在 `HalogenM` 类型中触发的，所以也用到了这个类型。
2. 添加了 `Action` 类型，它包含一个 `Click` 构造函数，用于处理 Halogen HTML 中的点击事件。
3. 在 `handleAction` 中处理 `Click` 行为时，我们 _触发_ 了一个输出消息。触发消息可以使用 `H.raise` 函数。

我们已经知道了组件如何触发消息，现在，让我们看一下如何处理子组件触发的消息。有三件事需要记住：

1. 渲染子组件时，需要把组件相关类型添加到插槽类型中，`ComponentHTML` 和 `HalogenM` 类型会用到这个插槽类型。添加的类型里需要包含组件的输出消息类型，这样编译器就能利用类型信息来校验消息处理函数。
2. 当用 `slot` 函数渲染子组件时，要提供一个输出消息对应的需要执行的行为。这和 `initialize` 函数类似，可以在组件初始化时执行一个行为。
3. 最后，在 `handleAction` 需要添加一个新的分支，用来处理对应于子组件输出的新行为。

我们首先编写父组件的插槽类型：

```purs
module Parent where

import Button as Button

type Slots = ( button :: forall query. H.Slot query Button.Output Int )

-- We can refer to the `button` label using a symbol proxy, which is a
-- way to refer to a type-level string like `button` at the value level.
-- We define this for convenience, so we can use _button to refer to its
-- label in the slot type rather than write `Proxy` over and over.
_button = Proxy :: Proxy "button"
```

插槽类型是一个行类型，每一个标签代表了我们支持的某个特定类型的子组件，使用类型 `H.Slot` 定义：

```purs
H.Slot query output id
```

这个类型（Slots）包含了此组件可以接受的查询类型、父组件需要处理的输出消息以及可以唯一标识单个组件的类型。

假设我们需要渲染 10 个按钮组件 -- 如何知道应该向哪个组件发送查询请求？这是插槽的 id 就发挥作用了。讨论查询时我们会讲解这个。

父组件的插槽行类型说明只支持一种子组件类型，我们可以用符号 `button` 和 `Int` 型变量来唯一索引子组件。我们不能向子组件发送查询请求，因为类型参数（query）是开放的，但是子组件可以触发类型为 `Button.Output` 的输出消息。

接下来，我们要定义一个行为来处理这种输出：

```purs
data Action = HandleButton Button.Output
```

处理这种行为时，我们可以从其中获得 `Button.Output` 类型的值，根据这个值来确定来确定操作。我们已经定义好了插槽以及行为类型，现在开始编写父组件：

```purs
parent :: forall query input output m. H.Component query input output m
parent =
  H.mkComponent
    { initialState: identity
    , render
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }
  where
  render :: forall state. state -> H.ComponentHTML Action Slots m
  render _ =
    HH.div_
      [ HH.slot _button 0 button unit HandleButton ]

  handleAction :: forall state. Action -> H.HalogenM state Action Slots output m Unit
  handleAction = case _ of
    HandleButton output ->
      case output of
        Button.Clicked -> do
          ...
```

可以注意到，在 `ComponentHTML` 和 `HalogenM` 类型中都使用了 `Slots` 这个类型。每当子组件中 `Button.Clicked` 事件触发时，父组件都会得到通知，并决定如何响应这个事件。

现在我们已经知道如何从子组件向父组件发送输出消息以及父组件如何处理这些消息。这是子组件向父组件通信的最主要方式。接下来，我们来看一下父组件如何向子组件发送消息。

### 查询

查询是父组件向子组件发送的命令或者请求。查询和行为类似，行为可以用函数 `handleAction` 函数处理，查询则是使用函数 `handleQuery` 函数处理。区别在于，查询是由组件外部出发的，而行为是组件内部触发的，所以查询会包含在组建的公共接口里。

查询多应用在父组件想要控制事件触发而不是子组件来控制的场景。例如：

- 父组件想要 _控制_ 表单的提交，而不是等待用户点击提交按钮。
- 父组件想要 _查询_ 一个自动填充表单的选定内容，而不是等待子组件在用户选定内容时触发输出消息。

查询是父组件用命令式控制子组件的方式。我们之前介绍过，查询分为两种：告知方式，当父组件要求子组件执行某种操作时，另外一种是请求方式，用于父组件想要从子组件查询信息时。

父组件发送查询，但是子组件负责定义和处理查询。这一点和行为类似：实现行为时，我们首先定义了 `Action` 类型以及处理函数 `handleAction`，对于查询，我们需要定义 `Query` 类型和查询处理函数 `handleQuery` 。

下面是一个查询类型的例子，包含了告知方式和请求方式。

```purs
data Query a
  = Tell a
  | Request (Boolean -> a)
```

我们可以这样理解查询：父组件可以通过 `Tell` 告知子组件执行某种操作，也可以通过发送 `Reqeust` 来请求一个 `Boolean` 型的值。首先查询类型时，需要注意的是类型参数 `a` 需要出现在每一个构造函数中。对于告知式查询，这个类型参数应该是最后一个类型参数，对于请求方式查询，这个类型参数应该是函数返回值类型。

查询是通过 `handleQuery` 函数处理的，和行为需要通过 `handleAction` 函数处理一样。我们假设状态、行为、输出消息类型已经定义好了，让我们来实现 `handleQuery` 函数：

```purs
handleQuery :: forall a m. Query a -> H.HalogenM State Action () Output m (Maybe a)
handleQuery = case _ of
  Tell a ->
    -- ... do something, then return the `a` we received
    pure (Just a)

  Request reply ->
    -- ... do something, then provide the requested `Boolean` to the `reply`
    -- function to produce the `a` we need to return
    pure (Just (reply true))
```

`handleQuery` 的参数类型是 `Query a` ，返回值是 `HalogenM` 类型值，针对这个值进行求值返回结果为 `Maybe a`。这就是为什么我们的每个构造函数都需要包含类型参数 `a`：我们需要在 `handleQuery` 中返回这个值。

当我们收到告知类型的查询时，我们可以直接用 `Just` 包裹 `a` ，然后返回即可，就是我们在上面 `handleQuery` 代码里如何处理 `Tell a` 的。

不过如果是请求方式的查询，我们需要多做一点工作。这种情况下我们不会收到一个 `a` 类型的值，而是会收到一个函数，调用这个函数时能得到一个 `a` 类型的值。例如，在处理 `Request (Boolean -> a)` 时，我们收到一个返回类型为 `a`，参数值为 `Boolean` 的函数。通常来说，当作模式匹配时，这个函数会命名为 `reply` 。在 `handleQuery` 函数，我们用 `true` 做参数调用这个函数获得一个 `a` 类型的值，然后使用 `Just` 包裹后返回。

请求方式的查询看起来很奇怪，但是这种方式，可以允许我们返回多种类型的值。下面是一些不同的查询，返回的是不同的值：

```purs
data Requests a
  = GetInt (Int -> a)
  | GetRecord ({ a :: Int, b :: String } -> a)
  | GetString (String -> a)
  | ...
```

父组件可以使用 `GetInt` 获得一个 `Int` 类型的值，可以使用 `GetString` 获得一个 `String` 类型的值，等等。可以把类型 `a` 当作查询的返回类型，而请求类型的插叙能提供了一种方式，可以让 `a` 支持多种不同的类型。稍后，我们将在父组件中讲解如何做到这一点。

让我们来看另一个例子，看一下在组件中如何定义和处理查询。

```purs
-- This component can be told to increment or can answer requests for
-- the current count
data Query a
  = Increment a
  | GetCount (Int -> a)

type State = { count :: Int }

-- Our query type shows up in our `Component` type
counter :: forall input output m. H.Component Query input output m
counter =
  H.mkComponent
    { initialState: \_ -> { count: 0 }
    , render
    , eval: H.mkEval $ H.defaultEval { handleQuery = handleQuery }
    }
  where
  render { count } =
    HH.div_
      [ HH.text $ show count ]

  -- We write a function to handle queries when they arise.
  handleQuery :: forall action a. Query a -> H.HalogenM State action () output m (Maybe a)
  handleQuery = case _ of
    -- When we receive the `Increment` query we'll increment our state.
    Increment a -> do
      H.modify_ \state -> state { count = state.count + 1 }
      pure (Just a)

    -- When we receive the `GetCount` query we'll respond with the state.
    GetCount reply -> do
      { count } <- H.get
      pure (Just (reply count))
```

在这个例子里，我们定义了一个计数器，父组件可以 _告知_ 组件加 1，也可以 _请求_ 当前的计数值。为支持这一点，我们需要：

1. 实现一个查询类型，类型包含一个告知行查询 `Increment a` 以及一个请求式查询 `GetCount (Int -> a)`。我们把这个类型添加到组件类型的公共接口中。
2. 实现查询处理函数 `handleQuery` ，当接收到查询时，该函数会被执行，这个函数下会被添加到 `eval` 中。

现在我们知道如何在子组件里定义和处理查询。现在，让我看一下如何向子组件 _发送_ 查询。首先，我们要先定义插槽类型：

```purs
module Parent where

type Slots = ( counter :: H.Slot Counter.Query Void Int )

_counter = SProxy :: SProxy "counter"
```

插槽类型包含了计数器组件的查询类型，输出消息类型定义为 `Void`，表明没有输出。

当父组件初始化时，我们从子组件获取计数值，然后执行加一操作，然后会再次获取计数值，确定数值确实已经加一了。为了这点，我们需要定义一个初始化时执行的行为：

```purs
data Action = Initialize
```

接着，我们可以转到组件的定义。

```purs
parent :: forall query input output m. H.Component query input output m
parent =
  H.mkComponent
    { initialState: identity
    , render
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , initialize = Just Initialize
        }
    }
  where
  render :: forall state. state -> H.ComponentHTML Action Slots m
  render _ =
    HH.div_
      [ HH.slot_ _counter unit counter unit ]

  handleAction :: forall state. Action -> H.HalogenM state Action Slots output m Unit
  handleAction = case _ of
    Initialize ->
      -- startCount :: Maybe Int
      startCount <- H.request _counter unit Counter.GetCount
      -- _ :: Maybe Unit
      H.tell _counter unit Counter.Increment
      -- endCount :: Maybe Int
      endCount <- H.request _counter unit Counter.GetCount

      when (startCount /= endCount) do
        -- ... do something
```

有几件事需要注意。

1. 在插槽类型里，对于计数器组件的标签，我们使用了类型代理 `_counter` 和标识符 `unit`，这个标签会用在 `slot` 函数渲染组件以及使用 `tell` 和 `request` 函数向子组件发送查询的时候。和特定组件交互，会一直使用标签和标识符。
2. 使用 `H.tell` 函数发送告知式查询 `Increment`，使用 `H.request` 函数发送请求类查询 `GetCount`。`GetCount` 有一个应答函数，类型是 `(Int -> a)`，注意，我们会得到一个类型为 `Maybe Int` 的返回值。

`tell` 和 `request` 函数的参数是标签、插槽标识符以及一个要发送的查询。tell 函数没有返回值，但是 request 有返回值，返回值使用 `Maybe` 包裹，`Nothing` 表明查询失败（要么是因为子组件返回了 `Nothing`，要么是根据提供的标签和标识符找不到对应的子组件）。同时 Halogen 还有函数 `tellAll` 和 `requestAll` ，用来通过标签、插槽标识符向某一类子组件发送同一个查询。

很多人都会觉得查询令人困惑，幸好查询在实践中应用并不多，如果还有困惑，可以反复阅读这一章。

## 组件插槽

我们已经学习了很多有关组件通信的知识，在我们进入最终例子之前，我们来回顾一下有关插槽的知识。

一个组件需要知道自己支持的子组件以便能够与之进行通信。它需要知道子组件支持的查询以及可能触发的输出消息。当然，它就要知道如何能够标识发送查询到哪一个组件。

`H.Slot` 类型包含了子组件支持的查询、输出以及唯一性标识。我们可以把多个插槽组合到一个 _行_ 类型，其中行类型的每一个标签都唯一的标识了一种类型的组件。下面的内容演示了如何越多不同类型的插槽：

```purs
type Slots = ()
```

这个类型表明组件不支持子组件。

```purs
type Slots = ( button :: forall query. H.Slot query Void Unit )
```

这个类型表明组件只支持一种类型的子组件，组件靠符号 `button` 标识。而且，不能向子组件发送查询（因为 `q` 是个开放的类型参数），子组件不会触发输出消息（通常使用 `Void` 表示，处理函数可以使用 `absurd` ）。而且，只能有一个这种类型的子组件，因为只有值 `unit` 属于类型 `Unit` 。

```purs
type Slots = ( button :: forall query. H.Slot query Button.Output Int )
```

这个类型和上个类型相同，区别是子组件可以触发输出消息 `Button.Output`，而且可以有多个这种类型的子组件，因为标识符式 `Int` 。

```purs
type Slots =
  ( button :: H.Slot Button.Query Void Int
  , modal :: H.Slot Modal.Query Modal.Output Unit
  )
```

这个插槽类型表明，组件支持两种类型的子组件，分别使用标签 `button` 和 `modal` 标识。父组件可以向按钮子组件发送 `Button.Query` 类型的查询，并且不会收到此种组件发送的输出消息。对于弹窗组件，可发送的查询类型是 `Modal.Query` ，同时会收到的消息类型是 `Modal.Output` 。子组件可以包含多个按钮子组件，但是只能最多有一个弹窗子组件。

常见的模式是组件导出自己的插槽类型，因为组件自身是知道自己支持的查询和消息类型，但是并不导出表示本组件的标识符类型，因为这点应该是父组件来决定的。

例如，按钮和弹窗组件可以像这样导出他们的插槽类型：

```purs
module Button where

type Slot id = H.Slot Query Void id

module Modal where

type Slot id = H.Slot Query Output id
```

我们最后一个插槽类型就会变为：

```purs
type Slots =
  ( button :: Button.Slot Int
  , modal :: Modal.Slot Unit
  )
```

这样写，代码会更简介，而且更容易在之后的修改中能保持一致性，因为如果插槽类型改变，我们只需修改插槽类型的来源模块，不需要修改使用的地方。

## 完整示例

总结一下，我们编写了这个父子组件通信的例子，涉及了本章讨论的三种通信方式。这个例子里重要的代码片段都包含了注释。

同样，建议你把代码复制到 [Try PureScript](https://try.purescript.org) ，可以交互式的学些这个例子。

```purs
module Main where

import Prelude

import Data.Maybe (Maybe(..))
import Data.Symbol (SProxy(..))
import Effect (Effect)
import Effect.Class (class MonadEffect)
import Effect.Class.Console (logShow)
import Halogen as H
import Halogen.Aff as HA
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  runUI parent unit body

-- The parent component supports one type of child component, which uses the
-- `ButtonSlot` slot type. You can have as many of this type of child component
-- as there are integers.
type Slots = ( button :: ButtonSlot Int )

-- The parent component can only evaluate one action: handling output messages
-- from the button component, of type `ButtonOutput`.
data ParentAction = HandleButton ButtonOutput

-- The parent component maintains in local state the number of times all its
-- child component buttons have been clicked.
type ParentState = { clicked :: Int }

-- The parent component uses no query, input, or output types of its own. It can
-- use any monad so long as that monad can run `Effect` functions.
parent :: forall query input output m. MonadEffect m => H.Component HH.HTML query input output m
parent =
  H.mkComponent
    { initialState
    , render
      -- The only internal event this component can handle are actions as
      -- defined in the `ParentAction` type.
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }
  where
  initialState :: input -> ParentState
  initialState _ = { clicked: 0 }

  -- We render three buttons, handling their output messages with the `HandleButton`
  -- action. When our state changes this render function will run again, each time
  -- sending new input (which contains a new label for the child button component
  -- to use.)
  render :: ParentState -> H.ComponentHTML ParentAction Slots m
  render { clicked } = do
    let clicks = show clicked
    HH.div_
      [ -- We render our first button with the slot id 0
        HH.slot _button 0 button { label: clicks <> " Enabled" } (Just <<< HandleButton)
        -- We render our second button with the slot id 1
      , HH.slot _button 1 button { label: clicks <> " Power" } (Just <<< HandleButton)
        -- We render our third button with the slot id 2
      , HH.slot _button 2 button { label: clicks <> " Switch" } (Just <<< HandleButton)
      ]

  handleAction :: ParentAction -> H.HalogenM ParentState ParentAction Slots output m Unit
  handleAction = case _ of
    -- We handle one action, `HandleButton`, which itself handles the output messages
    -- of our button component.
    HandleButton output -> case output of
      -- There is only one output message, `Clicked`.
      Clicked -> do
        -- When the `Clicked` message arises we will increment our clicked count
        -- in state, then send a query to the first button to tell it to be `true`,
        -- then send a query to all the child components requesting their current
        -- enabled state, which we log to the console.
        H.modify_ \state -> state { clicked = state.clicked + 1 }
        _ <- H.query _button 0 $ H.tell (SetEnabled true)
        on <- H.queryAll _button $ H.request GetEnabled
        logShow on

-- We now move on to the child component, a component called `button`.

-- This component can accept queries of type `ButtonQuery` and send output
-- messages of type `ButtonOutput`. This slot type is exported so that other
-- components can use it when constructing their row of slots.
type ButtonSlot = H.Slot ButtonQuery ButtonOutput

-- We think our button will have the label "button" in the row where it's used,
-- so we're exporting a symbol proxy for convenience.
_button = SProxy :: SProxy "button"

-- This component accepts two queries. The first is a request-style query that
-- lets a parent component request a `Boolean` value from us. The second is a
-- tell-style query that lets a parent component send a `Boolean` value to us.
data ButtonQuery a
  = GetEnabled (Boolean -> a)
  | SetEnabled Boolean a

-- This component can notify parent components of one event, `Clicked`
data ButtonOutput
  = Clicked

-- This component can handle two internal actions. It can evaluate a `Click`
-- action and it can receive new input when its parent re-renders.
data ButtonAction
  = Click
  | Receive ButtonInput

-- This component accepts a label as input
type ButtonInput = { label :: String }

-- This component stores a label and an enabled flag in state
type ButtonState = { label :: String, enabled :: Boolean }

-- This component supports queries of type `ButtonQuery`, requires input of
-- type `ButtonInput`, and can send outputs of type `ButtonOutput`. It doesn't
-- perform any effects, which we can tell because the `m` type parameter has
-- no constraints.
button :: forall m. H.Component HH.HTML ButtonQuery ButtonInput ButtonOutput m
button =
  H.mkComponent
    { initialState
    , render
      -- This component can handle internal actions, handle queries sent by a
      -- parent component, and update when it receives new input.
    , eval: H.mkEval $ H.defaultEval
        { handleAction = handleAction
        , handleQuery = handleQuery
        , receive = Just <<< Receive
        }
    }
  where
  initialState :: ButtonInput -> ButtonState
  initialState { label } = { label, enabled: false }

  -- This component has no child components. When the rendered button is clicked
  -- we will evaluate the `Click` action.
  render :: ButtonState -> H.ComponentHTML ButtonAction () m
  render { label, enabled } =
    HH.button
      [ HE.onClick \_ -> Just Click ]
      [ HH.text $ label <> " (" <> (if enabled then "on" else "off") <> ")" ]

  handleAction
    :: ButtonAction
    -> H.HalogenM ButtonState ButtonAction () ButtonOutput m Unit
  handleAction = case _ of
    -- When we receive new input we update our `label` field in state.
    Receive input ->
      H.modify_ _ { label = input.label }

    -- When the button is clicked we update our `enabled` field in state, and
    -- we notify our parent component that the `Clicked` event happened.
    Click -> do
      H.modify_ \state -> state { enabled = not state.enabled }
      H.raise Clicked

  handleQuery
    :: forall a
     . ButtonQuery a
    -> H.HalogenM ButtonState ButtonAction () ButtonOutput m (Maybe a)
  handleQuery = case _ of
    -- When we receive a the tell-style `SetEnabled` query with a boolean, we
    -- set that value in state.
    SetEnabled value next -> do
      H.modify_ _ { enabled = value }
      pure (Just next)

    -- When we receive a the request-style `GetEnabled` query, which requires
    -- a boolean result, we get a boolean from our state and reply with it.
    GetEnabled reply -> do
      enabled <- H.gets _.enabled
      pure (Just (reply enabled))
```

在下一章我们会学习如何运行 Halogen 应用。
