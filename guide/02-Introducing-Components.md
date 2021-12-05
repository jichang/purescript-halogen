# 组件介绍

Halogen HTML 是 Halogen 应用的基本构成部分。但是只有生成 HTML 的纯函数是不够的，实际应用中还需要其他功能：程序运行时的状态、执行网络请求等副作用以及响应 DOM 事件（例如，用户点击按钮）。

Halogen 组件根据输入生成 HTML，这点和我们之前看到的函数一样。但是和普通函数不同的是，组件可以包含内部状态，也可以在响应事件时更新状态或者执行外部调用，此外，还可以和其它组件进行通信。

Halogen 使用基于组件的架构，你可以把 UI 拆分成独立可重用的组件，每个组件都相对独立。然后可以通过组合不同的组件来构建完整的复杂应用。

例如，每个 Halogen 应用都至少由一个组件构成，通常成为根（root）组件。每个 Halogen 组件还可以包含其他组件，最后生成的组件树构成了完整的 Halogen 应用。

在这一章，我们会学习编写 Halogen 组件时要用到的类型和函数。对于初学者来说，这是本指南里最难的部分，因为多数概念对于初学者都非常陌生。首次阅读时如果感到太复杂也不要担心，在写 Halogen 应用过程中，你会不断的用到这些类型以及函数，很快你就会熟悉这些概念。如果阅读本章过程中感到吃力，要试着反复阅读并且自己构建一些不同的组件来加深理解。

在本章中，我们会看到更多 Halogen 声明式编程的例子。当编写组件，我们需要做的是描述对于任意状态状态其对应的 UI 是什么样的，Halogen 会根据你想要的 UI 去更新 DOM 元素。

## 小例子

我们之前看到了一个组件的例子：一个可以增加或者减少的计数器。

```purs
module Main where

import Prelude

import Data.Maybe (Maybe(..))
import Halogen as H
import Halogen.HTML as HH
import Halogen.HTML.Events as HE

data Action = Increment | Decrement

component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval H.defaultEval { handleAction = handleAction }
    }
  where
  initialState _ = 0

  render state =
    HH.div_
      [ HH.button [ HE.onClick \_ -> Just Decrement ] [ HH.text "-" ]
      , HH.text (show state)
      , HH.button [ HE.onClick \_ -> Just Increment ] [ HH.text "+" ]
      ]

  handleAction = case _ of
    Decrement ->
      H.modify_ \state -> state - 1

    Increment ->
      H.modify_ \state -> state + 1
```

这个组件使用一个整数作为内部状态，当点击按钮时去更新这个状态。

这个组件功能没有问题，但是在实际工作中，我们一般不会忽略掉类型声明。接下来，我们重新构建这个组件，但是这次要声明所有的类型。

## 构建一个简单组件（有类型）

一个典型的 Halogen 组件包含接受输入，维护一个内部状态，根据内部状态生成 HTML，同时在响应事件时，会更新状态或者执行其他外部作用等功能。在这个例子中，我们不会执行外部作用，相关内容后边章节会介绍。

接下来，我们拆分每个部分，然后同时介绍一下相关的类型。

### 输入

Halogen 组件可以接受来自父组件或者根组件的输入。如果你把组件当作一个函数，那么这里的输入就是指这个函数的参数。

如果组件需要输入，需要为其指定一个类型。例如，一个接受整数作为输入的组件，可以使用类型：

```purs
type Input = Int
```

我们的计数器不需要任何输入，这里有两个选择。一是我们声明输入类型为 `Unit`，意味着我们需要一个虚拟的值，并且最终会丢弃这个值：

```purs
type Input = Unit
```

第二个选择更加常见，每当我们需要使用这个输入类型时，我们可以用一个类型参数：`forall i. ...`。 两种方式都没有问题，在接下来的代码里我们都会使用第二种类型参数的方式。

### 状态

Halogen 组件会维护一个内部的状态，组件会使用这个状态来控制组件行为，生成 HTML。我们的计数器组件维护了一个整数值作为内部状态，所以可以用整数类型来作为状态的类型：

```purs
type State = Int
```

组件还需要负责生成初始状态。所有 Halogen 组件都需要实现 `initialState` 函数，这个函数负责根据输入来生成初始状态：

```purs
initialState :: Input -> State
```

我们的计数器组件没有使用输入，所以 `initialState` 函数可以使用类型参数，不需要声明具体的输入类型。运行计数器组件时，初始状态应该设置为 0:

```purs
initialState :: forall input. input -> State
initialState _ = 0
```

### 行为

Halogen 组件在响应内部事件时，可以更新状态，执行外部作用或者和其他组件通信。组件可以用一个类型来描述当处理内部事件时可以执行的不同操作。

我们的计数器组件有两种内部事件：

1. 点击"-"按钮，减少计数值
2. 点击"+"按钮，增加计数值

我们可以定义一个 `Action` 类型来描述组件可以处理的内部事件：

```purs
data Action = Increment | Decrement
```

这个类型描述了计数器组件可以处理增加或者减少两种事件。稍后，我们将看到如何在 HTML 中使用这个类型。

类似于 State 类型需要定义一个 `initialState` 生成初始状态，`Action` 类型需要定义一个函数 `handleAction` 来描述如何处理这些事件。

```purs
handleAction :: forall output m. Action -> H.HalogenM State Action () output m Unit
```

和输入类型一样，对于那些我们不需要使用的参数，可以使用类型参数来描述。

- `()` 类型意味着我们的组件没有子组件。我们也可以使用类型参数（通常使用 slots），而不是`()`。不过因为这个类型非常简短，我们会经常使用这个类型。
- `output` 类型参数是描述和父组件通信时需要用到的。
- `m` 类型只有在组件需要执行外部作用时才会用到。

我们的计数器组件不需要子节点，所以我们使用`()`。同时因为组件也不需要和父节点通信，不需要执行外部作用，所以我们会使用类型参数 `output` 和 `m`。

下面是计数器组件的 `handleAction` 函数：

```purs
handleAction :: forall output m. Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  Decrement ->
    H.modify_ \state -> state - 1

  Increment ->
    H.modify_ \state -> state + 1
```

`handleAction` 函数在响应 `Decrement` 时，会对内部状态做减 1 操作，响应 `Increment` 是会做加 1 操作。Halogen 提供了很多可以在`handleAction` 函数中使用的更新状态的函数，以下几个是最常用的：

- `modify` 用来更新状态，提供当前的状态，返回最新的状态
- `modify_` 和 `modify`相同，区别是不需要返回最新状态。（所以不需要像`modify`那样丢弃结果）
- `get` 用来获取状态
- `gets` 用来获取状态然后用状态作为参数调用一个函数（通常使用 `_.fieldName` 去获取结构体中的某个字段）

我们会在之后讲解执行外部作用是介绍 `HalogenM` 。计数器组件不需要执行外部作用，我们需要的只是状态更新函数。

### 渲染

Halogen 组件使用 `render` 函数来根据内部状态生成 HTML。`render` 函数会在每次内部状态发生改变时执行。这就是为什么 Halogen 是声明式的：我们只需要指定内部状态对应的 UI，Halogen 会负责处理更新，生成你需要的 UI。

渲染过程在 Halogen 是纯函数，意味着你不能获取当前事件、执行网络请求或者类似的操作，你能做的就是根据内部状态生成对应的 HTML。

我们看一下`render`函数的类型，里面有上一章介绍过的 `ComponentHTML`。这个类型是一个特定的 `HTML` 类型，是针对组件生成的 HTML。我们这里还是会使用 `()` 和 `m`，这些是涉及到子组件的时候才会用到，我们之后的章节会做介绍。

```purs
render :: forall m. State -> H.ComponentHTML Action () m
```

现在看一下渲染函数，这和上一章介绍的 Halogen HTML 内容一样，对于类型 `ComponentHTML` ，我们可以编写正常的 HTML。

```purs
import Halogen.HTML.Events

render :: forall m. State -> H.ComponentHTML Action () m
render state =
  HH.div_
    [ HH.button [ HE.onClick \_ -> Just Decrement ] [ HH.text "-" ]
    , HH.text (show state)
    , HH.button [ HE.onClick \_ -> Just Increment ] [ HH.text "+" ]
    ]
```

#### 处理事件

接下来我们看一下 Halogen 中如何处理事件。首先，我们需要声明事件处理函数，事件处理函数和其他属性一样，都是写在属性数组参数中。在事件处理函数中，我们要返回一个当前组件能够处理的 `Action` 类型值。当事件发生时，会调用 `handleAction` 函数来处理。

你可能会好奇我们在使用 `onClick` 时为什么会提供匿名函数做参数，让我们看一下它的类型：

```purs
onClick
  :: forall row action
   . (MouseEvent -> Maybe action)
  -> IProp (onClick :: MouseEvent | row) action

-- Specialized to our component
onClick
  :: forall row
   . (MouseEvent -> Maybe Action)
  -> IProp (onClick :: MouseEvent | row) Action
```

Halogen 中，事件处理函数的第一个参数是一个回调函数。回调函数的参数是当前发生的 DOM 事件（在这个例子里，是一个鼠标点击事件 `MouseEvent` ），这个事件里包含了你可能会使用的信息，回调函数需要返回一个 Halogen 需要运行的行为（action）。在我们的例子里，我们不需要使用这个具体的事件，所以我们丢弃这个值，直接返回一个我们想要运行的行为（`Increment` 或者 `Decrement`）。

`onClick` 函数的返回值类型是 `IProp`。在前面的章节我们提到过 `IProp` ，Halogen HTML 元素会定义自身支持的属性和事件。设置的属性或者事件也都会声明自己的类型，Halogen 通过类型检测确保你没有设置此元素不支持的属性和事件。在这个例子中，按钮支持 `onClick` 事件，所以代码不会报错。

在这个例子里，参数 MouseEvent 在 onClick 的处理函数里是被直接忽略掉了，因为要执行的行为可以根据点击的按钮直接确定。在第三章里我们会介绍如何使用事件信息。

### 整合

接下来，我们通过整合介绍过的类型和函数来构建我们计数器组件。这一次我们会带上类型。

```purs
-- This can be specified if your component takes input, or you can leave
-- the type variable open if your component doesn't.
type Input = Unit

type State = Int

initialState :: forall input. input -> State
initialState = ...

data Action = Increment | Decrement

handleAction :: forall slots output m. Action -> H.HalogenM State Action () output m Unit
handleAction = ...

render :: forall m. State -> H.ComponentHTML Action () m
render = ...
```

这些类型和函数是 Halogen 组件的核心部分。但是单独每一部分都是不够的，我们需要把他们组合起来。

我们需要使用 `H.mkComponent` 来做这一点。这个函数参数类型是 `ComponentSpec`，这个类型是一个包含了 `initialState`, `render` 和 `eval` 的结构体，函数返回一个 `Component` 类型值：

```purs
component =
  H.mkComponent
    { -- First, we provide our function that describes how to produce the first state
      initialState
      -- Then, we provide our function that describes how to produce HTML from the state
    , render
      -- Finally, we provide our function that describes how to handle actions that
      -- occur while the component is running, which updates the state.
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }
```

之后的章节我们会详细介绍 `eval` 函数。现在你只需要知道 `eval` 是定义组件如何响应事件的。我们现在需要响应的事件只有行为，所以我们只需要使用 `handleAction` 函数。

我们的组件基本完整了，但是我们还漏掉了最后一个概念：组件类型。

### `H.Component` 类型

`mkComponent` 函数根据 `ComponentSpec` 生成一个组件，组件其实是由函数构成的结构体。在下一章我们会探讨其细节。

```purs
mkComponent :: H.ComponentSpec ... -> H.Component query input output m
```

得到的组件类型是 `H.Component`，这个类型有 4 个类型参数，每个类型参数描述了组建的一些公共接口。我们构建的组件不需要和父子组件通信，所以用不到这些。不过，我们还是会介绍这些类型参数，以便更好的学习接下来的章节内容。

1. 第一个参数 `query` 描述父组件如何与本组件通信。我们会在后面介绍父子组件时介绍这个概念。
2. 第二个参数 `input` 代表了本组件接受的输入。在本章的例子里，组件不接受任何输入，我们会使用类型参数。
3. 第三个参数 `output` 描述了本组件任何与父组件通信。和 `query` 一样，我们会在后面介绍父子组件时介绍这个概念。
4. 最后一个参数 `m`，描述了组件执行外部作用时所用到的 Monad 类型。在本章的例子里，组件不需要执行任何外部作用，我们会使用类型参数。

## 最终结果

介绍了很多内容之后，我们终于为计数器组件声明了所有的类型信息。如果你能自如的按照这种方式构建组件，那你已经对构建 Halogen 组件有了大体的认识。本指南剩余的内容都是建立在对内部状态、行为和渲染 HTML 的基础之上的。

下面的代码里，我们添加了 `main` 函数，这样就可以把代码复制到 [Try PureScript](https://try.purescript.org) 中运行。我们在之后的章节会介绍如何运行 Halogen 应用。现在你可以先忽略 `main` 函数，重点关注我们定义的组件。

```purs
module Main where

import Prelude

import Effect (Effect)
import Halogen as H
import Halogen.Aff as HA
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = HA.runHalogenAff do
  body <- HA.awaitBody
  runUI component unit body

type State = Int

data Action = Increment | Decrement

component :: forall query input output m. H.Component query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval H.defaultEval { handleAction = handleAction }
    }

initialState :: forall input. input -> State
initialState _ = 0

render :: forall m. State -> H.ComponentHTML Action () m
render state =
  HH.div_
    [ HH.button [ HE.onClick \_ -> Decrement ] [ HH.text "-" ]
    , HH.text (show state)
    , HH.button [ HE.onClick \_ -> Increment ] [ HH.text "+" ]
    ]

handleAction :: forall output m. Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  Decrement ->
    H.modify_ \state -> state - 1

  Increment ->
    H.modify_ \state -> state + 1
```
