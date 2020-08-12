# Halogen 指南

Halogen 是一个声明式的基于组件的 PureScript UI 库，强调类型安全。在本指南中，你会学习到 Halogen 的核心概念以及编写实际应用时需要用到的常见模式。

下面是个小例子，支持增加或者减少的计数器：

```purs
module Main where

import Prelude

import Data.Maybe (Maybe(..))
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

data Action = Increment | Decrement

component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }
  where
  initialState _ = 0

  render state =
    HH.div_
      [ HH.button [ HE.onClick \_ -> Just Decrement ] [ HH.text "-" ]
      , HH.div_ [ HH.text $ show state ]
      , HH.button [ HE.onClick \_ -> Just Increment ] [ HH.text "+" ]
      ]

  handleAction = case _ of
    Increment -> H.modify_ \state -> state + 1
    Decrement -> H.modify_ \state -> state - 1
```

你可以把这个例子（或者本指南中的其他例子）复制到 [Try PureScript](https://try.purescript.org)。强烈建议你这样做，可以交互式的分析这个例子。例如，你可以修改按钮的文案为 `"Increment"` 和 `"Decrement"`，而不是使用符号 `"+"` 和 `"-"`。

> 默认情况下，Try PureScript 会在你修改了代码后自动重新编译。你可以禁用这个特性，然后通过点击"Compile"按钮来手动触发编译。

现在不用担心不理解这段代码，在你阅读完本指南之后，你会理解这个组件是如何工作以及如何编写自己的组件。

## 如何阅读本指南

在本指南中，我们会学习 Halogen 应用的基本组成部分：元素和组件。当你理解了这些之后，你就可以用可重用的小组件构建复杂应用了。

这是一个逐步递进的介绍 Halogen 关键概念的指南。每一章都是建立在前一章内容的基础上，我们建议你按照顺序阅读每一章的内容。

Halogen 是一个 PureScript 库，需要读者了解 PureScript 中的基本概念，例如函数、结构体、数组、 `do` 表达式、 `Effect`和`Aff`。了解 HTML 和 DOM 也会对理解相关概念有所帮助。如果你需要回顾一下，我们推荐阅读：

- 关于 PureScript：[PureScript Book](https://book.purescript.org) 和 Jordan Martinez 的 [PureScript Reference](https://github.com/JordanMartinez/purescript-jordans-reference).
- 关于 HTML: MDN 的[HTML 介绍](https://developer.mozilla.org/en-US/docs/Learn/HTML/Introduction_to_HTML) 和 [DOM 事件](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events).

## 目录

1. [渲染 Halogen HTML](./01-Rendering-Halogen-HTML.md)
2. [组件简介](./02-Introducing-Components.md)
3. [执行作用](./03-Performing-Effects.md)
4. [生命周期和订阅](./04-Lifecycles-Subscriptions.md)
5. [父子组件](./05-Parent-Child-Components.md)
6. [运行应用](./06-Running-Application.md)
7. [下一步](./07-Next-Steps.md)
