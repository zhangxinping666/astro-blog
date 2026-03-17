---
title: React当中的this指向
link: this
catalog: true
date: 2025-01-05 13:00:00
description: React Hooks 使用方法， 让你深入了解react hooks。
tags:
  - React
  - this
categories:
  - [前端, React]
---

# JavaScript 与 React 中的 `this` 指向详解

## 一、 JavaScript 当中的 `this` 指向

### 1. 定义

在 JavaScript 中，`this` 是一个关键字，它代表函数执行时的**当前环境（Context）**。

> **核心法则**：`this` 的指向不是在函数定义时确定的，而是在**函数被调用时**确定的。

### 2. `this` 常见的指向

- **全局环境**：在浏览器环境下，全局作用域中的 `this` 指向 `window` 对象。
- **普通函数调用**：在非严格模式下，普通函数直接调用，`this` 指向 `window`；严格模式（`use strict`）下指向 `undefined`。
- **对象方法调用**：如果函数作为对象的一个属性被调用，`this` 指向该**对象**。
- **构造函数调用**：使用 `new` 关键字时，`this` 指向**新创建的实例对象**。
- **箭头函数**：箭头函数没有自己的 `this`，它的 `this` 继承自**外层（父级）作用域**。

---

### 3. 手动绑定 `this`（显式绑定）

当我们需要强制改变函数的 `this` 指向时，可以使用以下三个方法：

#### (1) `call()`

- **用法**：`fn.call(targetObj, arg1, arg2, ...)`
- **特点**：立即执行函数，参数需要**一个一个列出来**。

#### (2) `apply()`

- **用法**：`fn.apply(targetObj, [arg1, arg2, ...])`
- **特点**：立即执行函数，参数需要以**数组（或类数组）**的形式传入。

#### (3) `bind()`

- **用法**：`const newFn = fn.bind(targetObj, arg1, ...)`
- **特点**：**不会立即执行**，而是返回一个新的函数，这个新函数的 `this` 永远指向 `targetObj`。

---

## 二、 React 当中的 `this` 指向绑定

在 React 的类组件（Class Component）中，事件处理函数默认不绑定 `this`。如果直接调用，`this` 会是 `undefined`。为了在方法中访问 `this.state` 或 `this.setState`，需要进行绑定：

### 1. 在构造函数中使用 `bind(this)`

这是 React 官方一度最推荐的做法。

```javascript
constructor(props) {
  super(props);
  this.state = { count: 0 };
  // 提前绑定，确保只绑定一次，性能较好
  this.handleClick = this.handleClick.bind(this);
}

handleClick() {
  this.setState({ count: this.state.count + 1 });
}

```

### 2. 使用箭头函数（类属性语法）

目前最流行、最简洁的写法，利用了箭头函数捕获外层 `this` 的特性。

```javascript
// 直接定义为箭头函数，无需在 constructor 绑定
handleClick = () => {
  this.setState({ count: this.state.count + 1 });
};
```

### 3. 使用内联 `bind`

在渲染时直接绑定。

- **代码**：`<button onClick={this.handleClick.bind(this)}>Click</button>`
- **缺点**：组件每次重新渲染（render）时都会创建一个新的函数实例，**影响性能**。

### 4. 使用内联箭头函数

在渲染时定义一个匿名箭头函数。

- **代码**：`<button onClick={() => this.handleClick()}>Click</button>`
- **缺点**：同内联 `bind`，每次渲染都会创建新函数。但在需要**传递参数**时非常方便。

---

## 三、 总结对比

| 绑定方式               | 推荐程度   | 优点               | 缺点                                           |
| ---------------------- | ---------- | ------------------ | ---------------------------------------------- |
| **Constructor Bind**   | ⭐⭐⭐⭐   | 性能最优，逻辑清晰 | 样板代码多（啰嗦）                             |
| **箭头函数属性**       | ⭐⭐⭐⭐⭐ | 代码简洁，写法现代 | 需要 Babel 插件支持（现在基本都带）            |
| **内联 Bind/箭头函数** | ⭐⭐⭐       | 传参方便           | 每次 render 产生新对象，在大列表场景有性能风险 |

---
