---
title: "useSignal() 是 Web 框架的未来"
publishDate: "2023-02-28"
description: "Signals 是一种存储应用程序状态的方式，类似于 React 的 `useState()`。但是，有一些关键的差异使 Signals 更具优势。"
coverImage:
  src: "./cover.png"
  alt: "cover"
tags: ["signal", "翻译", "技术"]
---

> 本文为一篇翻译，你可以点此访问原文：[useSignal() is the Future of Web Frameworks](https://www.builder.io/blog/usesignal-is-the-future-of-web-frameworks)

Signals 是一种存储应用程序状态的方式，类似于 React 的 `useState()`。但是，有一些关键的差异使 Signals 更具优势。

## 什么是 Signals（信号）

`useState() => value + setter`

`useSignals() => getter + setter`

Signals 和 State 之间的主要区别在于 Signals 返回一个 _getter_ 和一个 _setter_ ，而**非响应式系统**返回其值（和一个 _setter_ ）。

> 注意: 有些响应式系统一起返回 getter/setter，有些则作为两个独立的引用，但其思想是相同的。

### State vs. State

本质问题：是 _State_ 这个单词混淆了两个不同的概念。

- **StateReference**： _reference_ 是指状态的引用（译者注：我理解是 _state_ 的指针）。
- **StateValue**：这是存放在状态引用 或 存储中的实际的 _value_。

为什么返回 _getter_ 比返回其值更有优势？因为通过返回 _getter_ ，你可以将**状态引用**的传递与**状态值**的读取分离开来。

让我们以一段 SolidJS 代码为例：

```tsx
export function Counter() {
  const [getCount, setCount] = createSignal(0)

  return <button onClick={() => setCount(getCount() + 1)}>{getCount()}</button>
}
```

- `createSignal()`：分配 `StateStorage` 并将其初始化为 `0`。
- `getCount`：store 的引用，你可以传递它。
- `getCount()`：是你通过引用获取到的状态中的值。

## 我没懂！我看起来觉得一样啊

上面解释了 _Signals_ 与旧的 _State_ 的不同，但没有解释为什么我们应该关注这一差异。

Signals 是可响应的！就意味着它们会追踪对状态感兴趣的（订阅）人，如果状态发生了变化，就通知订阅者状态发生了变化。

要具有响应性，Signals 必须收集谁对 Signals 的值感兴趣（订阅）。他们通过观察在哪些上下文中调用 _state-getter_ 来获得这些信息。通过从 getter 获取值的行为，可以告诉 Signals 这个地方对其值感兴趣。如果值发生更改，则需要使该位置发生重新计算。换句话说，调用 _getter_ 将创建订阅。

这就是为什么传递 _state-getter_ 而不是 _state-value_ 非常重要的原因。状态值的传递不会向 Signals 提供关于实际使用该值的地方的任何信息。这就是为什么区分状态 **reference** 和状态 **value** 在 Signals 中如此重要。

相比之下，[Qwik](https://qwik.builder.io/) 也有同样的例子。注意！（getter/setter）已被替换为单个对象的 `.value` 属性（其实就是 getter/setter）。虽然语法不同，但内部工作原理是相同的。

```tsx
export function Counter() {
  const count = useSignal(0)

  return <button onClick$={() => count.value++}>{count.value}</button>
}
```

重要的是，当单击按钮并累加值时，框架只需要将文本节点从`0`更新到`1`。之所以可以这样做，是因为在模板的初始化渲染期间，Signal 已经了解到只有文本节点才能访问 `count.value` 。因此，它知道如果 `count` 的值发生变化，它只需要更新文本节点，而不需要更新其他内容。

## `useState()` 的缺点

现在让我们看看 React 是如何使用 `useState()` 的，并看看它的缺点：

```tsx
export function Counter() {
  const [count, setCount] = useState(0)

  return <button onClick={() => setCount(count + 1)}>{count}</button>
}
```

React 的 `useState()` 返回的是 _state-value_ 。这意味着 `useState()` 不知道 _state-value_ 在组件或应用程序中被如何使用。这也就意味着：一旦你通过 `setState()` 通知 React 进行状态更改，React 并不知道页面的哪个部分已经更改，因此必须重新渲染整个组件。这是一种昂贵的计算消耗。

## `useRef()` 不进行渲染

`useRef()` 的使用方式和 `useSignal()` 很像，用于传递对状态的引用（_reference_），而非状态本身。可 `useRef()` 缺少的是订阅追踪与通知。

幸好，在基于 Signals 的框架中，`usessignal()` 和 `useRef()` 是相同的东西。`useSignal()` 不仅可以完成 `useRef()` 所做的工作，还可以进行订阅追踪。这进一步简化了框架的 API 设计。

## `useMemo()` 是内置的

Signals 很少有需要使用缓存的情况，因为它开箱即用，对此没有什么工作量！

请思考这两个计数器和其两个子组件的示例：

```tsx
export function Counter() {
  console.log('<Counter />')
  const countA = useSignal(0)
  const countB = useSignal(0)

  return (
    <div>
      <button onClick$={() => countA.value++}>A</button>
      <button onClick$={() => countB.value++}>B</button>
      <Display count={countA.value} />
      <Display count={countB.value} />
    </div>
  )
}

export const Display = component$(({ count }: { count: number }) => {
  console.log(`<Display count={${count}} />`)
  return <div>{count}!</div>
})
```

在上面的示例中，只有两个 `Display` 组件中的一个文本节点会被更新。没有得到更新的文本节点在首次渲染后将永远不会打印。

```shell
# Initial render output
<Counter/>
<Display count={0}/>
<Display count={0}/>

# Subsequent render on click
(blank...)
```

而你完全无法通过 React 实现同样的效果，因为至少会有一个组件需要重新渲染。所以，让我们看看如何在 React 中缓存组件以做到最少次的重渲染。

```tsx
export default function Counter() {
  console.log('<Counter />')
  const [countA, setCountA] = useState(0)
  const [countB, setCountB] = useState(0)

  return (
    <div>
      <button onClick={() => setCountA(countA + 1)}>A</button>
      <button onClick={() => setCountB(countB + 1)}>B</button>
      <Display count={countA} />
      <Display count={countB} />
    </div>
  )
}

export const MemoDisplay = memo(Display)

export function Display({ count }: { count: number }) {
  console.log(`<Display count={${count}} />`)
  return <div>{count}!</div>
}
```

但就算是使用了缓存的方式，React 还是会执行多次重渲染：

```shell
# Initial render output
<Counter/>
<Display count={0}/>
<Display count={0}/>

# Subsequent render on click
<Counter/>
<Display count={1}/>
```

如果我们没有用 `memo` 来处理，那我们可以看到 log：

```shell
# Initial render output
<Counter/>
<Display count={0}/>
<Display count={0}/>

# Subsequent render on click
<Counter/>
<Display count={1}/>
<Display count={0}/>
```

这就比使用 Signals 要做的工作多得多。所以，这就是为什么 Signals 的作用就好像你记住了所有东西，而实际上你自己不需要记住任何东西。

## Props 透传

让我们看一个 React 的购物车示例：

```tsx
export default function App() {
  console.log('<App />')
  const [cart, setCart] = useState([])

  return (
    <div>
      <Main setCart={setCar} />
      <NavBar cart={cart} />
    </div>
  )
}
```

```tsx
export function Main({ setCart }: any) {
  console.log(`<Main />`)

  return (
    <div>
      <Product setCart={setCart} />
    </div>
  )
}

export function Product({ setCart }: any) {
  console.log(`<Product />`)

  return (
    <div>
      <button onClick={() => setCart((cart: any) => [...cart, 'product'])}>Add to cart</button>
    </div>
  )
}
```

```tsx
export function NavBar({ cart }: any) {
  console.log(`<NavBar />`)

  return (
    <div>
      <Cart cart={cart} />
    </div>
  )
}

export function Cart({ cart }: any) {
  console.log(`<Cart />`)

  return <div>Cart: {JSON.stringify(cart)}</div>
}
```

购物车的状态通常被拉高放置在购买按钮和渲染购物车的位置之上的最顶级的公共父级。它通常非常接近组件渲染树的顶部。所以在我们的例子中，我们称之为公共祖先组件。

共同祖先组件有两个分支：

1. `setCart prop-drilling` 分支：透传 `setCart` ，穿过了许多层组件后，达到了购买按钮组件。
2. `cart prop-drilling` 分支：另一个 Props `cart` 状态 ，也穿过了许多层组件，直到它到达购物车组件。

遇到的问题是：每次点击 _**购买**_ 按钮时，组件树中大多数组件都必须重渲染。这导致如下的结果：

```shell
# "buy" button clicked
<App/>
<Main/>
<Product/>
<NavBar/>
<Cart/>
```

如果你使用了缓存的手段，那么你可以避免 `setCart prop-drilling` 分支重渲染，而不是 `cart prop-drilling` 分支，所以输出仍然是这样的：

```shell
# "buy" button clicked
<App/>
<NavBar/>
<Cart/>
```

而这种情况在使用 Signals 时，会打印如下结果：

```shell
# "buy" button clicked
<Cart/>
```

这大大减少了需要执行的代码量。

## 哪些框架支持 Signals ？

![image.png](https://s2.loli.net/2023/03/01/685d2Z7CXMnUTGW.png)

目前支持 Signals 的且较为流行的 Web 框架有：[Vue](https://vuejs.org/)、 [Preact](https://preactjs.com/) 、 [Solid](https://www.solidjs.com/) 以及 [Qwik](https://qwik.builder.io/) 。

在当前，Signals 并不是新鲜的东西；在此之前，它们已经存在于 [Knokout](https://knockoutjs.com/) 或其他框架中了。不同的是，近年来，通过巧妙的编译技巧和与 JSX 的深度集成，Signals 已经极大地改进了它的 DX（开发者体验），这使得它非常简洁，使用起来非常愉快 -- 而这部分正是较新的部分。

## 结论

Signals 是在应用程序中存储状态的一种方式，类似于 React 中的 `useState()`。然而，最关键的区别在于，Signals 返回的是 _getter_ 和 _setter_ ，而**非响应式系统**只返回其 _value_ 和 _setter_ 。

这点恰恰很重要，因为 Signals 是响应式的，这意味着它们需要追踪对状态感兴趣的用户，并将状态的更改通知订阅者。这是通过观察调用 _state-getter_ 的上下文来实现的，它创建了一个订阅。

相反，React 中的 `useState()` 只返回 _state-value_ ，这意味着它不知道组件会在哪儿以及如何使用它的 _state-value_ ，所以必须重新呈现整个组件树以响应状态的更改。

近年来，Signals 已经拥有了良好的 DX，这使得它们变得易于使用。出于这个原因，我认为你将使用的下一个框架会是基于 Signals 的响应式框架。

