---
title: "Learning Note - React Hooks"
date: 2022-08-09T20:27:47+02:00
author: "Feng Xue"
tags: ["Javascript", "React Hooks", "Learning Notes"]
toc: true
draft: false
usePageBundles: false
---

## Basic chapter 

### Reason to Hooks

React的本质就是从Model到View的映射。这里的Model就是Component的props和state。

当Model的数据变化时，React不管它是怎么变化的，只关心它变化之后和之前的区别。这也就是所谓的声明式declarative，而这通过React的diff函数来实现。

所以UI的展现更像是一个函数的执行。Model是输入参数，View是函数，执行结果就是Dom的改变。而React只要保证通过最优的办法执行变化的过程就行了。

Class作为React Component不合适的两点:

1. React Component之间很少使用继承.
2. React是由state驱动的，并不需要调用外部生成的class的instance的method。

Function作为React Component的局限:

1. Function无法提供内部状态，必须是纯函数
2. Function也无法提供完整的lifecycle

所以React的中的Hooks就是把target钩到某个可能会变化的data source或者event source中，当被钩到的data或者event变化的时候，这个target会被重新执行，产生新的结果。

Hooks中被钩的对象不仅仅可以是数据，也可以是另一个Hook执行的结果，这样可以带来逻辑的复用。


Hooks是在High order Component的使用背景下被创造出来的，所以Hook解决了HOC留下的wrapper hell和code难以理解的问题


### Hooks basic usage

* `useState` -- 使用useState保存的值的原则是，不要使用需要计算得到的值
* `useEffect` -- 应该用来执行并不影响当前结果的代码，是不影响当前渲染出来的UI的
  * Deps使用reference来比较值是否有变化，所以数组和object要小心
  * ***NB***: useEffect是在render执行之后调用的
* `useCallback` -- 这个hook的使用目的是，在需要把函数作为值来传递到UI的时候，为了避免多余的render，而保证在某些条件传递的函数本身是不变的
* `useMemo` -- useMemo其实可以理解为useEffect和useState的结合，即在deps变化的时候来执行useEffect的函数来计算值的变化，同时将值通过useState来赋予state
* `useRef`
  * 在多次渲染之间共享数据
  * 存某个 DOM 节点的引用
* `useContext` -- 定义全局状态

useEffect基本上等价于componentDidMount, componentDidUpdate 和componentWillUnmount。但是并不完全等价。区别在于：
useEffect的callback函数只有在deps变化的时候才会被调用，而componentDidUpdate则一定会被调用
useEffect里callback函数的返回函数(用来做清理工作)，是在依赖项变化或者组件销毁前被调用

如果需要有一个constructor的功能，可以使用以下代码：

```javascript
// 创建一个自定义 Hook 用于执行一次性代码
function useSingleton(callback) {
  // 用一个 called ref 标记 callback 是否执行过
  const called = useRef(false);
  // 如果已经执行过，则直接返回
  if (called.current) return;
  // 第一次调用时直接执行
  callBack();
  // 设置标记为已执行过
  called.current = true;
}
```

Hook可以实现大部分的lifecycle的功能，但是对于 getSnapshotBeforeUpdate, componentDidCatch, getDerivedStateFromError，这些周期还是只能使用class来实现

## Practice

### Data consistence

在保证 State 完整性的同时，也要保证它的最小化。
某些数据如果能从已有的 State 中计算得到，那么我们就应该始终在用的时候去计算，而不要把计算的结果存到某个 State 中。

#### 避免中间状态，确保唯一数据源

在任何时候想要定义新状态的时候，都要问自己一下：这个状态有必要吗？是否能通过计算得到？是否只是一个中间状态？

### Render props模式

通过把render函数传递给某个组件，让这个render函数来决定渲染的效果，从而实现组件的复用

### Self defined event

#### Synthetic Event

当我们绑定一个时间到节点上的时候，因为virtual dom的存在，react会把时间绑定到APP根节点上，React 17之前是在document上，17以后是在react的根节点上
一是因为virtual dom在render的时候，可能节点还没有render到页面上，所以无法绑定
二是可以屏蔽底层细节，避免浏览器兼容性问题

所以react component自定义的事件本质上是回调函数

### Organize project structure via business 
为了降低项目的复杂度，通过业务特征和逻辑来组织项目结构，这样可以让各个feature能相对独立，便于管理和维护

为了实现松耦合，可以针对dynamic的部分单独设计组件，并传递动态的参数以达到动态内容的修改不影响其他部分

### Form
React是状态驱动，Form是事件驱动
React的onChange函数会在用户任何输入的时候都调用函数，但是html的原生onchange函数则只会在输入框失去focus的时候触发

#### Controlled vs uncontrolled
对于uncontrolled component，并不会传递value给component，而是通过获取这个component的值来得到其状态，例如useRef。好处在于，其值的变化并不会触发render。坏处则是无法检测值的变化

而对于controlled component，则接受value as props，同时一个callback函数来update这个值

![Form elements](/images/react-hooks/form_element.png "Form elements")

使用Controlled component来维护表单主要核心三个部分：
1. 字段的名称
2. 绑定value
3. 处理onChange事件

所以Hook对于Form的贡献在于，可以把Form里面的value都放到hook里面，并且通过useState来提供处理的函数。例如：

```javascript
import { useState, useCallback } from "react";

const useForm = (initialValues = {}) => {
  // 设置整个 form 的状态：values
  const [values, setValues] = useState(initialValues);
  
  // 提供一个方法用于设置 form 上的某个字段的值
  const setFieldValue = useCallback((name, value) => {
    setValues((values) => ({
      ...values,
      [name]: value,
    }));
  }, []);

  // 返回整个 form 的值以及设置值的方法
  return { values, setFieldValue };
};
```

