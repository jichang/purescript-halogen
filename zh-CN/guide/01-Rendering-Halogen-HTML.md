# 渲染 Halogen HTML

Halogen HTML 元素是 Halogen 应用中的最小组成部分，使用这些元素可以描述你想在界面上看到的内容。

Halogen HTML 元素并不是组件（我们会在下一张中讨论组件），而且元素不能在组件之外进行渲染。但是，通常我们会编写一些通用的帮助函数来生成 Halogen 元素，然后就可以在组件中使用这些函数来构建界面。

在本章中，我们将介绍如何编写单纯的 HTML，不涉及组件和事件。

## Halogen HTML

如示例所示，你可以使用模块`Halogen.HTML`或者`Halogen.HTML.Keyed`中的函数来编写 HTML：

```purs
import Halogen.HTML as HH

element = HH.h1 [ ] [ HH.text "Hello, world" ]
```

Halogen HTML 元素类似于浏览器中的 DOM 元素，但是他们是有 Halogen 控制的虚拟元素，不是实体的 DOM 元素。实际上，Halogen 会负责根据你所编写的代码来更新实际的 DOM 元素。

Halogn 中的元素可接受两个参数：

1. 第一个参数是数组，可包含属性、事件处理回调函数或者作用在本元素上的索引，对应于在 HTML 中的属性，例如`placeholder`，以及事件处理回调函数，例如`onClick`。我们将在下一张中介绍如何处理事件，本章我们主要关注在属性上。
2. 如果支持子元素的话，还可以传递第二个参数。参数值就是当前元素的子元素。

来看一个简单示例，我们把下面这段 HTML 翻译成 Halogen HTML 中的元素：

```html
<div id="root">
  <input placeholder="Name" />
  <button class="btn-primary" type="submit">
    Submit
  </button>
</div>
```

让我们来分解一下对应的 Halogen HTML：

1. Halogen HTML 代码和普通的 HTML 有相同的结构：一个 `div` 节点，包含了一个 `input` 和一个 `button`，而这个 button 又包含了一个纯文本。
2. 属性也不再是定义在 tag 上的键值对，而是一个相同属性的数组。
3. 子元素也不再是定义在闭合标签内，而是和属性一样，是一个数组。

定义 HTML 元素相关属性的相关函数来自于模块`Halogen.HTML.Properties`。

```purs
import Halogen.HTML as HH
import Halogen.HTML.Properties as HP

html =
  HH.div
    [ HP.id_ "root" ]
    [ HH.input
        [ HP.placeholder "Name" ]
    , HH.button
        [ HP.classes [ HH.ClassName "btn-primary" ]
        , HP.type_ HP.ButtonSubmit
        ]
        [ HH.text "Submit" ]
    ]
```

在这里，我们可以注意到 Halogen 如何强调类型安全

1. input 元素不能有自元素，所以 Halogen 在定义函数时，不允许 HH.input 接受子节点数组作为参数。
2. button 的`type`只能是特定的几个值，所以 Halogen 定义了一个 Sum 类型的值来限制取值范围。
3. CSS 中的 class 使用了一个特殊的 newtype `ClassName` 来定义，可以做一些特殊的操作，例如 `classes` 函数会保证定义在合并定义的 class 之后，class 名称之间都是空格隔开的格式。

有些 HTML 元素或者属性的名字和 PureScript 中的关键字或者函数是冲突的，所以 Halogen 会在这些名称之前添加一个下划线*。这就是为什么上面的例子里我们使用了 `type*`而不是`type`。（你可能会注意到我们用了`id\_`而不是`id`，但是在PureScript 0.12之后，`id`函数被重命名为`identity`，之后版本的Halogen会直接使用`id`）

如果你不需要制定任何属性，可以直接使用带有下划线前缀的函数，例如下面的例子中， `div` 和 `button` 元素都没有任何属性。

```purs
html = HH.div [ ] [ HH.button [ ] [ HH.text "Click me!"] ]
```

所以我们可以重写为如下版本，可以是代码看起来更简短，整洁：

```purs
html = HH.div_ [ HH.button_ [ HH.text "Click me!" ] ]
```

## 在 Halogen HTML 中编写函数

通常，我们需要写一些辅助函数，因为 Halogen HTML 是基于普通的 PureScript 函数构建的，所以你可以在其中使用自由的使用其他函数。

如下例，我们编写了一个函数，接受一个整数，将其渲染为一个文本节点。

```purs
header :: forall w i. Int -> HH.HTML w i
header visits =
  HH.h1_
    [ HH.text $ "You've had " <> show visits <> " visitors" ]
```

我们还可以渲染一个字符串列表：

```purs
lakes = [ "Lake Norman", "Lake Wylie" ]

html :: forall w i. HH.HTML w i
html = HH.div_ (map HH.text lakes)
-- same as: HH.div_ [ HH.text "Lake Norman", HH.text "Lake Wylie" ]
```

上面的函数使用了一个新的类型 `HH.HTML`，这个我们还没有遇到过，但是不用担心，在下一节中，我们会介绍这个类型，我们还是继续学习在 HTML 中使用函数。

根据条件选择性的渲染 HTML 是一个非常常见的需求。我们可以直接使用普通的 `if` 和 `case` 语句，但是为一些常见的模式编写辅助函数是非常有用的。让我们看一下两个辅助函数，你很有可能在编写的应用的时候需要用到这两个函数，这能进一步帮助我们了解如何在 Halogen HTMLl 中编写函数。

首先，你很有可能需要处理某种元素有可能不存在的情况。如下所示的函数会在值存在时渲染对应的元素，如果不存在，则会渲染一个空元素。

```purs
maybeElem :: forall w i a. Maybe a -> (a -> HH.HTML w i) -> HH.HTML w i
maybeElem val f =
  case val of
    Just x -> f x
    _ -> HH.text ""

-- Render the name, if there is one
renderName :: forall w i. Maybe String -> HH.HTML w i
renderName mbName = maybeElem mbName \name -> HH.text name
```

另外，有可能你想要在某个条件为 true 时渲染 HTML，如果条件为 false，则不做任何计算和渲染。这种情况，你可以单独编写一个函数，这样，只有当条件为 true 是才会执行函数，做相应的计算。

```purs
whenElem :: forall w i. Boolean -> (Unit -> HH.HTML w i) -> HH.HTML w i
whenElem cond f = if cond then f unit else HH.text ""

-- Render the old number, but only if it is different from the new number
renderOld :: forall w i. { old :: Number, new :: Number } -> HH.HTML w i
renderOld { old, new } =
  whenElem (old /= new) \_ ->
    HH.div_ [ HH.text $ show old ]
```

至此，我们已经学习了操作 HTMLL 的几种方式，下面让我们来学习一下相关的类型。

## HTML 类型

到现在为止，我们在写 HTML 时都没有指定类型，但是在实际编写 Halogen 应用的时候，是需要包含对应的类型声明的。

### `HTML w i`

`HTML` 是 Halogen 中核心类型，是用来编写通用的 HTML 元素，和具体的组件无关。例如，我们在之前看到的 `h1`、`text`和`button`元素都是使用的这个类型。我们也可以用这个类型自定义 HTML 元素。

`HTML` 类型接受两个类型参数：`w`，代表“控件”(widget)，描述了相关的组件类型，另外一个类型参数`i`，代表“输入”(input),表示处理 DOM 事件的类型。

当你编写 Halogen HTML 的辅助函数，但是不需要响应 DOM 事件，你可以在使用 `HTML` 类型时不声明具体的`w` 和 `i` 类型。例如，下面这个根据一个文本标签创建一个 button 的辅助函数：

```purs
primaryButton :: forall w i. String -> HH.HTML w i
primaryButton label =
  HH.button
    [ HP.classes [ HH.ClassName "primary" ] ]
    [ HH.text label ]
```

除了接受字符串之外，你也可以接受其它的 HTML 作为标签：

```purs
primaryButton :: forall w i. HH.HTML w i -> HH.HTML w i
primaryButton label =
  HH.button
    [ HP.classes [ HH.ClassName "primary" ] ]
    [ label ]
```

当然，对于一个 button，通常在按钮被点击后，我们需要执行某些操作。这些我们会在下一章中介绍。

### `ComponentHTML` 和 `PlainHTML`

在 Halogen 应用中常见的有两种 HTML 相关的类型。

`ComponentHTML` 用来编写针对特定类型组件的 HTML，但是这个类型也可以非组件代码中使用，但是并不常见，在下一章中我们会详细介绍这个类型。

`PlainHTML` 是一个有特殊限制的`HTML`类型，通常用于编写不包含任何组件而且不响应任何 DOM 事件的 HTML 元素。使用这个类型，不需要指定 `HTML` 的两个类型参数，这个需要按值传递 HTML 时特别方便。但是，如果和其他包含组件的 HTML 在一块儿组合使用时，需要使用函数 `fromPlainHTML` 进行转换。

### `IProp`

如果查看 `Halogen.HTML.Properties` 和 `Halogen.HTML.Events` 模块中的函数，你会看到大量使用了 `IProp` 类型。例如，函数 `placeholder` 用来设置 text 输入框的占位符内容：

```purs
placeholder :: forall r i. String -> IProp (placeholder :: String | r) i
placeholder = prop (PropName "placeholder")
```

`IProp` 用来指定事件和属性，使用一个行类型（row type）来唯一的指定特定的事件和属性。这样做，当你在 Halogen HTML element 元素指定属性或者事件时，Halogen 能够判断此元素是否支持这个属性或者事件。

这是因为 Halogen HTML 元素同样有一个行类型列出了支持的属性和事件。当你为元素赋值属性或者事件时，Halogen 会查找当前元素对应的行类型，看是否支持这个属性或者事件。

这有助于保证 HTML 的合理。例如，按照 DOM 标准， `<div>` 元素不支持 `placeholder` 属性。当你试图为 `div` 赋值 `placeholder` 属性时，会触发编译期错误：

```purs
-- ERROR: Could not match type ( placeholder :: String | r )
-- with type ( accessKey :: String, class :: String, ... )
html = HH.div [ HP.placeholder "blah" ] [ ]
```

这个错误提示你试图使用一个此元素不支持的属性。错误信息会首先列出你想要使用的属性，然后列出了当前元素 _实际_ 支持的属性。这也算是 Halogen 类型安全的例证。
