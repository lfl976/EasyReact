# EasyReact

<https://pomb.us/build-your-own-react/>

## Step 0:React 渲染到页面的步骤

1. React 代码

```javascript
const element = <h1 title="foo">Hello</h1>;
const container = document.getElementById("root");
ReactDOM.render(element, container);
```

2. Babel 会把 JSX 转换 React.creatElement()的形式

```javascript
const element = React.createElement("h1", { title: "foo" }, "Hello");
const container = document.getElementById("root");
ReactDOM.render(element, container);
```

3. React.createElement()函数返回的是个对象，所以可以把上面的代码下面这样，是等效的

```javascript
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
};
const container = document.getElementById("root");
ReactDOM.render(element, container);
```

4. 根据 element 对象创建 HTML 元素，添加到页面上

```javascript
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
};
const container = document.getElementById("root");
// 创建 h1 标签并加上 title 属性
const node = document.createElement(element.type);
node["title"] = element.props.title;
// 创建 h1 的子元素“Hello”字符串
const text = document.createTextNode("");
text["nodeValue"] = element.props.children;

node.appendChild(text);
container.appendChild(node);
```

查看页面：[step-0.html](step-0.html)

## Step 1: createElement 函数

1. 一段 React 代码如下

```javascript
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
);
const container = document.getElementById("root");
ReactDOM.render(element, container);
```

2. 通过 [Babel.js](https://babeljs.io/repl) 编译后如下

```javascript
const element = React.createElement(
  "div",
  {
    id: "foo",
  },
  React.createElement("a", null, "bar"),
  React.createElement("b", null)
);
const container = document.getElementById("root");
ReactDOM.render(element, container);
```

3. 可以看到 `React.createElement()` 接收参数，返回值是一个包含 type 和 props 属性的对象，实现如下

```javascript
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children,
    },
  };
}
```

子元素中除了 HTML 元素，还可能包含字符串，我们把字符串转换为文本节点，修改上面代码如下

```javascript
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map((child) =>
        typeof child === "object" ? child : createTextElement(child)
      ),
    },
  };
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  };
}
```

4. 把自己写的 `createElement` 放到一个叫 `Didact` 的对象里

```javascript
const Didact = {
  createElement,
};
```

5. 怎么使用新的 `createElement` 呢，Babeljs 编译 JSX 时加上 _/\*\* @jsx Didact.createElement \*/_ 这段注释就可以

```javascript
/** @jsx Didact.createElement */
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
);
const container = document.getElementById("root");
ReactDOM.render(element, container);

// 编译后的结果
/** @jsx Didact.createElement */
const element = Didact.createElement(
  "div",
  {
    id: "foo",
  },
  Didact.createElement("a", null, "bar"),
  Didact.createElement("b", null)
);
const container = document.getElementById("root");
ReactDOM.render(element, container);
```

## Step 2: Render 函数

`render` 函数根据 `createElement` 返回的对象创建 DOM 并挂载到根节点

```javascript
function render(element, container) {
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type);

  const isProperty = (key) => key !== "children";
  Object.keys(element.props)
    .filter(isProperty)
    .forEach((name) => {
      dom[name] = element.props[name];
    });

  element.props.children.forEach((child) => render(child, dom));

  container.appendChild(dom);
}
```

## Step-3: Concurrent Mode

递归执行 render 渲染一棵树有个问题，如果这个 DOM 树很大，render 函数执行完成需要花费很多时间，会影响浏览器渲染页面的线程，造成页面卡顿

所以把渲染一棵树的任务，拆分成很多小单元执行

```javascript
let nextUnitOfWork = null;

function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}

requestIdleCallback(workLoop);

function performUnitOfWork(nextUnitOfWork) {
  // TODO
}
```

上述代码中可以把 `requestIdleCallback` 看作是 `setTimeout` , 区别是浏览器线程空闲时会执行 `requestIdleCallback`, `requestIdleCallback` 函数会给一个 deadline 参数，告诉我们现在这个空闲期间还剩多少时间可以使用，空闲期间我们可以用来执行 javascript，空闲期间结束后浏览器会重新控制主线程。

## Step-4: Fibers

HTML 的结构是树状结构，渲染时需要递归执行。

fiber 也是一种数据结构，它通过关联 根节点 → 子节点 → 兄弟节点 的方式可以访问整棵 DOM 树

把每个节点元素变成一个 fiber 结构的对象，每个 fiber 就是一个单元任务

1. 假设渲染如下的树

```javascript
Didact.render(
  <div>
    <h1>
      <p />
      <a />
    </h1>
    <h2 />
  </div>,
  container
);
```

```html
root
↓↑
<div>
↓↑   ↖︎
<h1>  <h2>
↓↑   ↖︎
<p> → <a>
```

2. createDom

```javascript
function createDom(fiber) {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type)

  const isProperty = key => key !== "children"
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name]
    })

  return dom
}
```

3. render

```javascript
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  }
}
```

4. workLoop

```javascript
let nextUnitOfWork = null
function on workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}
requestIdleCallback(workLoop)
```

5. performUnitOfWork

```javascript
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }

  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }

  const elements = fiber.props.children
  let index = 0
  let prevSibling = null

  while (index < elements.length) {
    const element = elements[index]

    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }

    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }

    prevSibling = newFiber
    index++
  }

  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}

```

## Step-5: Render and Commit 阶段

现在执行每一个元素节点时都会执行把DOM添加到页面的操作，这个过程可能被打断

```javascript
function commitRoot() {
  // TODO add nodes to dom
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  }
  nextUnitOfWork = wipRoot
}

let nextUnitOfWork = null
let wipRoot = null

function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }

  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
  }

  requestIdleCallback(workLoop)
}

requestIdleCallback(workLoop)
```

## Step-6: Reconciliation

现在只执行了添加操作，添加更新和删除操作

```javascript
function commitRoot() {
  commitWork(wipRoot.child)
  currentRoot = wipRoot
  wipRoot = null
}

function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  domParent.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot,
  }
  nextUnitOfWork = wipRoot
}

let nextUnitOfWork = null
let currentRoot = null
let wipRoot = null
```

重写 `performUnitOfWork` 

```javascript
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }

  const elements = fiber.props.children
  reconcileChildren(fiber, elements)

  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}

function reconcileChildren(wipFiber, elements) {
  let index = 0
  let prevSibling = null

  while (index < elements.length) {
    const element = elements[index]

    const newFiber = {
      type: element.type,
      props: element.props,
      parent: wipFiber,
      dom: null,
    }

    if (index === 0) {
      wipFiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }

    prevSibling = newFiber
    index++
  }
}
```

