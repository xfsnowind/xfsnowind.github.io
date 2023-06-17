---
title: "React Query Effective Query Keys"
date: 2023-06-17T16:02:03+08:00
author: "Feng Xue"
tags: ["React", "React Query", "Typescript", "Translation"]
toc: true
usePageBundles: false
draft: false
---

[Query Keys](https://react-query.tanstack.com/guides/query-keys)是React Query中非常重要的核心概念。它们可以使库能够正确地内部缓存数据，并在查询的依赖发生更改时自动重新获取数据。最后，它将允许你在需要时手动与查询缓存进行交互，例如在执行变更后更新数据或手动使某些查询失效。

在向你展示我个人如何组织Query Keys以更有效地执行这些操作之前，让我们快速了解一下这三个要点的含义。

## 缓存数据

在内部，查询缓存只是一个JavaScript对象，其中键是序列化的Query Keys，值是你的查询数据加上元信息。键以[deterministic way](https://react-query.tanstack.com/guides/query-keys#query-keys-are-hashed-deterministically)进行散列，因此你也可以使用对象（但在顶层，键必须是字符串或数组）。

最重要的部分是，键对于你的查询必须是*唯一的*。如果React Query在缓存中找到一个键的条目，它就会用它。还请注意，你不能将相同的键用于 `useQuery` 和 `useInfiniteQuery`。毕竟，只有一个查询缓存，数据会在这两者之间共享。这会造成问题，因为无限查询与*常规*查询具有根本不同的结构。

```ts
useQuery({ queryKey: ['todos'], queryFn: fetchTodos })

// 🚨 这不可行
useInfiniteQuery({ queryKey: ['todos'], queryFn: fetchInfiniteTodos })

// ✅ 选择其他内容
useInfiniteQuery({
  queryKey: ['infiniteTodos'],
  queryFn: fetchInfiniteTodos,
})
```

## 自动重新获取

> 查询是声明性的。

这是一个非常重要的概念，怎么强调都不过分，而且这也是可能需要一些时间来理解的概念。大多数人以*命令式*的方式思考查询，尤其是重新获取。

我有一个查询，它获取一些数据。现在我点击这个按钮，我想要重新获取，但使用不同的参数。我看到过许多类似下面这样的尝试：

```jsx
function Component() {
  const { data, refetch } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  // ❓ 如何将参数传递给 refetch ❓
  return <Filters onApply={() => refetch(???)} />
}
```

答案是：**你不需要这样做。**

这不是`refetch`的用途 - 它是用于使用*相同的参数*重新获取数据的。

如果你有一些会改变数据的*状态*，你只需将其放入Query Keys中，因为React Query会在键发生更改时自动触发重新获取。因此，当你想要应用筛选器时，只需更改你的*客户端状态*：

```jsx
function Component() {
  const [filters, setFilters] = React.useState()
  const { data } = useQuery({
    queryKey: ['todos', filters],
    queryFn: () => fetchTodos(filters),
  })

  // ✅ 设置本地状态，让其*驱动*查询
  return <Filters onApply={setFilters} />
}
```

通过`setFilters`更新引起的重新渲染将向React Query传递一个不同的Query Keys，这将让它去重新获取数据。在[#1: Practical React Query - Treat the query key like a dependency array](https://tkdodo.eu/blog/practical-react-query#treat-the-query-key-like-a-dependency-array)中有一个更详细的示例。

## 手动交互

与查询缓存的手动交互是你的Query Keys结构最重要的部分。许多交互方法，如[invalidateQueries](https://react-query.tanstack.com/reference/QueryClient#queryclientinvalidatequeries)或[setQueriesData](https://react-query.tanstack.com/reference/QueryClient#queryclientsetqueriesdata)，支持[Query Filters](https://react-query.tanstack.com/guides/filters#query-filters)，允许你模糊匹配Query Keys。

## 有效的React Query Keys

请注意，这些要点反映了我的个人观点（实际上，这个博客上的所有内容都是如此），所以不要将其视为在使用Query Keys时必须遵循的规定。我发现这些策略在应用程序变得更复杂时效果最好，并且它们的扩展性也非常好。在制作一个待办事项应用程序时，你绝对不需要这样做 😁。

### 放在一起

如果你还没有阅读过[Kent C. Dodds](https://twitter.com/kentcdodds)的文章[Maintainability through colocation](https://kentcdodds.com/blog/colocation)，请务必读一下。我不认为将所有Query Keys全局存储在 `/src/utils/queryKeys.ts` 中有什么用。我会把我的Query Keys与相应的Queries放在同一个功能目录下，例如：

```
- src
  - features
    - Profile
      - index.tsx
      - queries.ts
    - Todos
      - index.tsx
      - queries.ts

```

*queries*文件包含了与React Query相关的所有内容。我通常只导出自定义hooks，因此实际的查询函数和Query Keys将保持局部。

### 总是使用数组键

是的，Query Keys也可以是字符串，但为了保持统一，我倾向于数组。React Query会在内部将其转换为数组，所以：

```ts
// 🚨 无论如何都会被转换为 ['todos']
useQuery({ queryKey: 'todos' })
// ✅
useQuery({ queryKey: ['todos'] })
```

**更新**：在 React Query v4 中，所有Key都需要是数组。

### 结构

把Query Keys从*最通用*到*最特定*进行结构化，之间的粒度层级取决于你认为合适的数量。下面是一个我如何为待办事项列表结构化Query Keys的示例，该列表允许使用筛选器和详细视图：

```js
['todos', 'list', { filters: 'all' }]
['todos', 'list', { filters: 'done' }]
['todos', 'detail', 1]
['todos', 'detail', 2]
```

通过这种结构，我可以使用`['todos']`来无效化与待办事项相关的所有内容，无论是列表还是详细视图，也可以针对特定的列表进行定位，如果我知道确切的键。[更新Mutation的返回值](https://react-query.tanstack.com/guides/updates-from-mutation-responses)会让操作变得更加灵活，因为你可以在需要时对所有列表进行操作：

```js
function useUpdateTitle() {
  return useMutation({
    mutationFn: updateTitle,
    onSuccess: (newTodo) => {
      // ✅ 更新待办事项的详细信息
      queryClient.setQueryData(['todos', 'detail', newTodo.id], newTodo)

      // ✅ 更新包含此待办事项的所有列表
      queryClient.setQueriesData(['todos', 'list'], (previous) =>
        previous.map((todo) => (todo.id === newTodo.id ? newtodo : todo))
      )
    },
  })
}
```

但是如果列表和详细视图的结构差异很大，则可能不起作用，不过你还可以无效化所有列表：

```js
function useUpdateTitle() {
  return useMutation({
    mutationFn: updateTitle,
    onSuccess: (newTodo) => {
      queryClient.setQueryData(['todos', 'detail', newTodo.id], newTodo)

      // ✅ 只无效化所有列表
      queryClient.invalidateQueries({ queryKey: ['todos', 'list'] })
    },
  })
}
```

如果你知道当前所在的列表，例如通过从URL读取筛选器，就可以构造出精确的Query Keys，你还可以将这两种方法结合起来，在列表上调用`setQueryData`并无效化其他所有列表：

```js
function useUpdateTitle() {
  // 假设有一个自定义钩子，返回当前筛选器，
  // 存储在 URL 中
  const { filters } = useFilterParams()

  return useMutation({
    mutationFn: updateTitle,
    onSuccess: (newTodo) => {
      queryClient.setQueryData(['todos', 'detail', newTodo.id], newTodo)

      // ✅ 即时更新当前列表
      queryClient.setQueryData(['todos', 'list', { filters }], (previous) =>
        previous.map((todo) => (todo.id === newTodo.id ? newTodo : todo))
      )

      // ✅ 无效化所有其他列表
      queryClient.invalidateQueries({ queryKey: ['todos', 'list'], exact: false })
    },
  })
}
```

**更新**：在v4中，`refetchActive`已被替换为`refetchType`。在上面的示例中，要改成`refetchType: 'none'`，因为我们不想要重新获取任何数据。

### 使用Query Key工厂

在上面的示例中，你可以看到我手动声明了很多Query Keys。这不仅容易出错，而且在将来进行更改时变得更加困难，例如，如果你想要为键添加*另一个*级别。

这就是为什么我建议每个功能使用一个Query Key工厂。它只是一个简单的对象，包含条目和生成Query Key的函数，你可以在自定义hooks中使用这些键。对于上面的示例结构，它可能如下所示：

```ts
const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: string) => [...todoKeys.lists(), { filters }] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: number) => [...todoKeys.details(), id] as const,
}
```

这给我带来了很大的灵活性，因为每个级别都是建立在另一个级别之上，但仍然可以独立访问：

```ts
// 🕺 移除与待办事项相关的所有内容
queryClient.removeQueries({ queryKey: todoKeys.all })

// 🚀 使所有列表无效
queryClient.invalidateQueries({ queryKey: todoKeys.lists() })

// 🙌 预取单个待办事项
queryClient.prefetchQueries({
  queryKey: todoKeys.detail(id),
  queryFn: () => fetchTodo(id),
})
```

今天就到这里。如果你有任何问题，请随时在 Twitter 上与我联系，或者在下面留言。