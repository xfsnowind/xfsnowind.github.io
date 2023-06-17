---
title: "Leveraging the Query Function Context"
date: 2023-06-17T18:08:37+08:00
author: "Feng Xue"
draft: true
---

我们都努力成为更好的工程师，随着时间的推移，我们希望在这个努力中取得成功。也许我们学到了新的东西，这些新东西让我们之前的想法变得无效或受到挑战。或者我们意识到，我们认为理想的模式在现在的需求水平下无法扩展。

自从我开始使用 React Query 以来已经过了相当长的时间。我认为在这个过程中我学到了很多，也看到了很多东西。我希望我的博客尽可能更新，这样你回来重新阅读时，知道其中的概念仍然有效。现在更重要的是，[Tanner Linsley](https://twitter.com/tannerlinsley)同意将我的博客链接到官方的 [React Query 文档](https://react-query.tanstack.com/community/tkdodos-blog) 中。

这就是为什么我决定在我的 [Effective React Query Keys](effective-react-query-keys) 文章中撰写这个附录的原因。请确保先阅读它，以了解我们所讨论的内容。

## 个人观点

> 不要使用内联函数 - 利用提供给你的查询函数上下文，并使用生成对象键的查询键工厂

内联函数是将参数传递给 `queryFn` 最简单的方法，因为它可以封闭你的自定义钩子中的其他可用变量。让我们看一下永恒的待办事项示例：

```ts:title=inline-query-fn
type State = 'all' | 'open' | 'done'
type Todo = {
  id: number
  state: TodoState
}
type Todos = ReadonlyArray<Todo>

const fetchTodos = async (state: State): Promise<Todos> => {
  const response = await axios.get(`todos/${state}`)
  return response.data
}

export const useTodos = () => {
  // 假设这里从某个地方获取当前用户的选择，比如 URL
  const { state } = useTodoParams()

  // ✅ queryFn 是一个内联函数，封闭了传递的状态
  return useQuery({
    queryKey: ['todos', state],
    queryFn: () => fetchTodos(state),
  })
}
```

也许你认识这个示例 - 它是 [#1: Practical React Query - Treat the query key like a dependency array](practical-react-query#treat-the-query-key-like-a-dependency-array) 的一种变化。这在简单的示例中效果很好，但是在有很多参数的情况下，它存在一个相当大的问题。在较大的应用程序中，有很多过滤和排序选项并不罕见，我个人见过最多传递 10 个参数。

假设我们想要在查询中添加排序功能。我喜欢自底向上地处理这些问题 - 从 `queryFn` 开始，让编译器告诉我接下来需要修改什么：

```ts:title=sorting-todos {1,4,6}
type Sorting = 'dateCreated' | 'name'
const fetchTodos = async (
  state: State,
  sorting: Sorting
): Promise<Todos> => {
  const response = await axios.get(`todos/${state}?sorting=${sorting}`)
  return response.data
}
```

这肯定会在我们调用 `fetchTodos` 的自定义钩子中产生一个错误，所以让我们来修复它：

```ts:title=useTodos-with-sorting {2,8}
export const useTodos = () => {
  const { state, sorting } = useTodoParams()

  // 🚨 你能发现错误吗 ⬇️
  return useQuery({
    queryKey: ['todos', state],
    queryFn: () => fetchTodos(state, sorting),
  })
}
```

也许你已经发现了问题：我们的 `queryKey` 与实际依赖项不同步，而且没有红色的波浪线在警告我们 😔。在上述情况下，你可能会很快发现问题（希望是通过集成测试），因为更改排序不会自动触发重新获取。而且，坦率地说，在这个简单的示例中很明显。然而，在过去几个月中，我已经多次看到 `queryKey` 与实际依赖项不一致，随着复杂性的增加，这些问题可能会导致一些难以追踪的问题。这也是为什么 React 自带了 [react-hooks/exhaustive-deps eslint 规则](https://reactjs.org/docs/hooks-rules.html#eslint-plugin) 来避免这种情况。

那么 React Query 现在会提供自己的 eslint 规则吗？👀

嗯，这是一种选择。还有一个解决这个问题的方法是使用 [babel-plugin-react-query-key-gen](https://github.com/dominictwlee/babel-plugin-react-query-key-gen)，它会为你生成查询键，包括所有的依赖项。然而，React Query 提供了一种不同的内置处理依赖项的方式：`QueryFunctionContext`。

**更新**: 上述的 lint 规则现已存在。请查看[此处的文档](https://tanstack.com/query/v4/docs/react/eslint/eslint-plugin-query)。🚀

## QueryFunctionContext

`QueryFunctionContext` 是作为参数传递给 `queryFn` 的一个对象。在处理无限查询（infinite queries）时，你可能已经使用过它：

```js:title=useInfiniteQuery
// 这是 QueryFunctionContext ⬇️
const fetchProjects = ({ pageParam = 0 }) =>
  fetch('/api/projects?cursor=' + pageParam)

useInfiniteQuery({
  queryKey: ['projects'],
  queryFn: fetchProjects,
  getNextPageParam: (lastPage) => lastPage.nextCursor,
})
```

React Query 使用该对象向 `queryFn` 注入关于查询的信息。在无限查询的情况下，你将得到 `getNextPageParam` 的返回值注入为 `pageParam`。

然而，这个上下文还包含了用于此查询的 `queryKey`（我们将在上下文中添加更多的有趣功能），这意味着实际上你不需要闭包来处理这些内容，因为 React Query 将为你提供它们：

```js:title=query-function-context {1,3,12-15}
const fetchTodos = async ({ queryKey }) => {
  // 🚀 我们可以从 queryKey 中获取所有的参数
  const [, state, sorting] = queryKey
  const response = await axios.get(`todos/${state}?sorting=${sorting}`)
  return response.data
}

export const useTodos = () => {
  const { state, sorting } = useTodoParams()

  // ✅ 无需手动传递参数
  return useQuery({
    queryKey: ['todos', state, sorting],
    queryFn: fetchTodos,
  })
}
```

使用这种方法，你基本上无法在 `queryFn` 中使用任何额外的参数，除非将它们添加到 `queryKey` 中。🎉

## 如何为 QueryFunctionContext 添加类型

其中一个目标是通过使用传递给 `useQuery` 的 `queryKey` 来完全获得类型安全，并推断出 `QueryFunctionContext` 的类型。这并不容易，但自从 [v3.13.3](https://github.com/tannerlinsley/react-query/releases/tag/v3.13.3) 起，React Query 就支持了这一点。如果内联使用 `queryFn`，你会发现类型会被正确推断出来（感谢泛型）：

```ts:title=query-key-type-inference {6,9}
export const useTodos = () => {
  const { state, sorting } = useTodoParams()

  return useQuery({
    queryKey: ['todos', state, sorting] as const,
    queryFn: async ({ queryKey }) => {
      const response = await axios.get(
        // ✅ 这是安全的，因为 queryKey 是一个元组
        `todos/${queryKey[1]}?sorting=${queryKey[2]}`
      )
      return response.data
    },
  })
}
```

这很好，但仍然存在一些缺点：

- 你仍然可以使用闭包中的任何内容来构建你的查询。
- 以上方式中使用 `queryKey` 来构建 URL 仍然是不安全的，因为你可以将所有内容转换为字符串。

### 查询键工厂

这就是查询键工厂再次发挥作用的地方。如果我们有一个类型安全的查询键工厂来构建我们的键，我们可以使用该工厂的返回类型来为我们的 `QueryFunctionContext` 添加类型。以下是可能的实现方式：

```ts:title=typed-query-function-context {11,12,21-24}
const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (state: State, sorting: Sorting) =>
    [...todoKeys.lists(), state, sorting] as const,
}

const fetchTodos = async ({
  queryKey,
}: // 🤯 仅接受来自工厂的键
QueryFunctionContext<ReturnType<typeof todoKeys['list']>>) => {
  const [, , state, sorting] = queryKey
  const response = await axios.get(`todos/${state}?sorting=${sorting}`)
  return response.data
}

export const useTodos = () => {
  const { state, sorting } = useTodoParams()

  // ✅ 通过工厂构建键
  return useQuery({ queryKey: todoKeys.list(state, sorting), queryFn: fetchTodos })
}
```

类型 `QueryFunctionContext` 是由 React Query 导出的。它接受一个泛型，用于定义 `queryKey` 的类型。在上面的示例中，我们将其设置为与键工厂的 _list_ 函数返回的类型相等。由于我们使用了 [const 断言](the-power-of-const-assertions)，所有的键都将是严格类型化的元组。因此，如果我们尝试使用不符合该结构的键，将会得到一个类型错误。

## 对象查询键

在逐渐过渡到上述方法时，我注意到数组键的性能并不是很好。这在我们看如何解构查询键时变得明显：

```ts:title=weird-destruct
const [, , state, sorting] = queryKey
```

我们基本上忽略了前两个部分（我们硬编码的 _todo_ 和 _list_）只使用动态部分。当然，我们很快又在开头添加了另一个范围，这导致错误地构建了 URL：

<img src="./destruct-query-key.png" />

<SmallCentered>来源：我最近提交的 PR</SmallCentered>

事实证明，_对象_ 非常好地解决了这个问题，因为你可以使用命名解构。此外，在查询键中使用对象没有任何缺点，因为模糊匹配用于查询失效的方式对于对象和数组是一样的。如果你对其如何工作感兴趣，可以查看 [partialDeepEqual](https://github.com/tannerlinsley/react-query/blob/9e414e8b4f3118b571cf83121881804c0b58a814/src/core/utils.ts#L321-L338) 函数。

记住这一点，这是我根据我现在所了解的情况如何构建查询键的方式：

```ts:title=object-keys {3-6,11}
const todoKeys = {
  // ✅ 所有键都是包含一个对象的数组
  all: [{ scope: 'todos' }] as const,
  lists: () => [{ ...todoKeys.all[0], entity: 'list' }] as const,
  list: (state: State, sorting: Sorting) =>
    [{ ...todoKeys.lists()[0], state, sorting }] as const,
}

const fetchTodos = async ({
  // ✅ 从 queryKey 中提取命名属性
  queryKey: [{ state, sorting }],
}: QueryFunctionContext<ReturnType<typeof todoKeys['list']>>) => {
  const response = await axios.get(`todos/${state}?sorting=${sorting}`)
  return response.data
}

export const useTodos = () => {
  const { state, sorting } = useTodoParams()

  return useQuery({ queryKey: todoKeys.list(state, sorting), queryFn: fetchTodos })
}
```

对象查询键甚至可以使你的模糊匹配功能更强大，因为它们没有顺序。使用数组方法，你可以处理与待办事项相关的所有内容、所有待办事项列表，或具有特定筛选器的待办事项列表。使用对象键，你也可以做到这一点，但还可以处理所有列表（例如待办事项列表和个人资料列表）：

```js:title=fuzzy-matching-with-object-keys
// 🕺 删除与待办事项相关的所有内容
queryClient.removeQueries({ queryKey: [{ scope: 'todos' }] })

// 🚀 重置所有待办事项列表
queryClient.resetQueries({ queryKey: [{ scope: 'todos', entity: 'list' }] })

// 🙌 使所有范围内的列表失效
queryClient.invalidateQueries({ queryKey: [{ entity: 'list' }] })
```

如果你有多个具有层次结构但仍然希望匹配属于子范围的所有内容的重叠范围，这可能非常方便。

## 这值得吗？

和往常一样：这取决于情况。我最近一直喜欢这种方法（这也是我想与你分享的原因），但在复杂性和类型安全之间肯定存在权衡。在键工厂中组合查询键略微复杂（因为 _queryKeys_ 仍然必须是顶层的数组），根据键工厂的返回类型为上下文添加类型也不是一件简单的事情。如果你的团队规模较小，API 接口较简单和/或使用纯 JavaScript，你可能不想选择这种方法。像往常一样，选择最适合你特定情况的工具和方法。🙌
