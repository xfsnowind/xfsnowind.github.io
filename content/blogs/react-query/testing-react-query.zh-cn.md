---
title: "测试React Query"
date: 2023-06-12T23:56:57+08:00
author: "Feng Xue"
tags: ["React Query", "Translation", "Blogs"]
toc: true
usePageBundles: false
draft: false
---

> 本文翻译自 [TkDodo](https://github.com/tkdodo) 的 [Testing React Query](https://tkdodo.eu/blog/testing-react-query)

谈到测试，经常会和React Query一起出现一些问题，所以我将在这里尝试回答其中的一些问题。我认为其中一个原因是测试“智能”组件（也称为[容器组件](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)）并不是一件容易的事情。随着hooks的兴起，这种分离已经大部分过时。现在倾向于直接在需要它们的地方使用hooks，而不是进行主要是随意地分离并向下传递props。

我认为这通常是一个非常好的改进，有助于集中放置和代码可读性，但现在更多组件会去消耗props之外的依赖项。

它们可能是`useContext`。它们可能是`useSelector`。或者它们可能是`useQuery`。

这些组件在技术上不再是纯净的，因为在不同的环境中调用它们会导致不同的结果。在测试它们时，你需要仔细设置这些周围的环境以使其正常工作。

## 模拟网络请求

由于React Query是一个异步的服务器状态管理库，你的组件很可能会向后端发送请求。在测试时，这个后端无法用以提供真实数据，即使可用，你可能也不希望使测试依赖于它。

已经有海量的文章介绍如何使用jest模拟数据。如果你有api客户端，可以模拟它。你可以直接模拟fetch或axios。我非常认同Kent C. Dodds在他的文章[《Stop mocking fetch》](https://kentcdodds.com/blog/stop-mocking-fetch)中所写的内容：

使用[@ApiMocking](https://twitter.com/ApiMocking)的[mock service worker](https://mswjs.io/)

在模拟API方面，它可以成为你的唯一真实来源：

* 适用于测试的Node环境
* 支持REST和GraphQL
* 具有[storybook插件](https://storybook.js.org/addons/msw-storybook-addon/)，因此你可以为使用`useQuery`的组件编写story
* 在浏览器中用于开发目的，你仍然可以在浏览器开发工具中看到请求的发送情况
* 与Cypress一起使用，类似于fixtures

通过处理我们的网络层，我们可以开始讨论一些需要关注的React Query特定事项：

## QueryClientProvider

每当你使用React Query时，你需要一个QueryClientProvider，并给它一个queryClient——这是一个保存`QueryCache`的容器。而Cache会保存你的请求的数据。

我更喜欢为每个测试提供独立的QueryClientProvider，并为每个测试创建一个新的QueryClient。这样，测试之间完全隔离。另一种方法可能是在每个测试之后清除缓存，但我喜欢尽量将测试之间的共享状态保持最小化。否则，如果你同时运行测试，可能会出现意外和不稳定的结果。

## 对于自定义的Hooks

如果你正在测试自定义hooks，我非常确定你正在使用[react-hooks-testing-library](https://react-hooks-testing-library.com/)。这是最简单的测试hooks的方法。我们可以通过这个库将我们的钩子包装在一个包装器中，该包装器是一个React组件，在渲染时用于包装测试组件。我认为这是创建QueryClient的理想位置，因为它将在每个测试中都执行一次：

```tsx
const createWrapper = () => {
  // ✅ creates a new QueryClient for each test
  const queryClient = new QueryClient()
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  )
}

test("my first test", async () => {
  const { result } = renderHook(() => useCustomHook(), {
    wrapper: createWrapper()
  })
})
```

### 对于组件

如果你想测试一个使用useQuery hooks的组件，你还需要将该组件包在QueryClientProvider中。使用[react-testing-library](https://testing-library.com/docs/react-testing-library/intro/)里的一个小包装器包住render似乎是一个不错的选择。看看React Query在他们的[测试中是如何内部处理](https://github.com/TanStack/query/blob/ead2e5dd5237f3d004b66316b5f36af718286d2d/src/react/tests/utils.tsx#L6-L17)的。

## 关闭重试

这是使用React Query进行测试时最常见的问题之一：该库默认使用指数轮询进行三次重试，这意味着如果你想测试一个错误的查询，测试很可能会超时。最简单的方法是通过`QueryClientProvider`关闭重试。让我们扩展上面的示例：

```tsx
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        // ✅ turns retries off
        retry: false,
      },
    },
  })

  return ({ children }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  )
}

test("my first test", async () => {
  const { result } = renderHook(() => useCustomHook(), {
    wrapper: createWrapper()
  })
}
```

这会将组件树中所有查询默认设置为“无重试”。重要的是，这仅在你的实际的useQuery没有显性设置重试时才起作用。如果你有一个要求5次重试的查询，它仍会优先使用这5次重试设置，因为默认值仅作为备用。

### setQueryDefaults

The best advice I can give you for this problem is: Don't set these options on useQuery directly. Try to use and override the defaults as much as possible, and if you really need to change something for specific queries, use queryClient.setQueryDefaults.

So for example, instead of setting retry on useQuery:

我可以给你的最佳建议是：不要在useQuery上直接设置这些选项。尽可能使用和覆盖默认值，如果你确实需要针对特定查询更改某些东西，请使用[queryClient.setQueryDefaults](https://tanstack.com/query/latest/docs/react/reference/QueryClient?from=reactQueryV3&original=https%3A%2F%2Ftanstack.com%2Fquery%2Fv3%2Fdocs%2Freference%2FQueryClient#queryclientsetquerydefaults)。

例如，避免在useQuery上设置重试参数:

```tsx
const queryClient = new QueryClient()

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Example />
    </QueryClientProvider>
  )
}

function Example() {
  // 🚨 you cannot override this setting for tests!
  const queryInfo = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    retry: 5,
  })
}
```

改成这样：

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 2,
    },
  },
})

// ✅ only todos will retry 5 times
queryClient.setQueryDefaults(['todos'], { retry: 5 })

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Example />
    </QueryClientProvider>
  )
}
```

这样，所有的查询都会尝试两次，只有*todos*会重试五次，而且我也还有选项来在测试中把所有的查询都关掉🙌。

### ReactQueryConfigProvider

当然，这仅适用于已知的查询键。有时，你确实需要在组件树的某个子集上设置一些配置。在v2中，React Query提供了[ReactQueryConfigProvider](https://react-query-v2.tanstack.com/docs/api#reactqueryconfigprovider)来满足这个特定的用例。你可以在v3中使用几行代码实现相同的效果：

```jsx
const ReactQueryConfigProvider = ({ children, defaultOptions }) => {
  const client = useQueryClient()
  const [newClient] = React.useState(
    () =>
      new QueryClient({
        queryCache: client.getQueryCache(),
        muationCache: client.getMutationCache(),
        defaultOptions,
      })
  )

  return (
    <QueryClientProvider client={newClient}>{children}</QueryClientProvider>
  )
}
```

你也可以在这个[codesandbox的示例](https://codesandbox.io/s/react-query-config-provider-v3-lt00f)里看到。

## 始终等待查询的完成

Since React Query is async by nature, when running the hook, you won't immediately get a result. It usually will be in loading state and without data to check. The async utilities from react-hooks-testing-library offer a lot of ways to solve this problem. For the simplest case, we can just wait until the query has transitioned to success state:

由于React Query的本质是异步的，当运行hooks时，你并不会立即获得结果。它通常处于加载状态并且没有数据可供检查。react-hooks-testing-library的[异步工具](https://react-hooks-testing-library.com/reference/api#async-utilities)提供了许多解决此问题的方法。对于最简单的情况，我们可以等待查询过渡到成功状态：

```tsx
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
      },
    },
  })
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  )
}

test("my first test", async () => {
  const { result, waitFor } = renderHook(() => useCustomHook(), {
    wrapper: createWrapper()
  })

  // ✅ wait until the query has transitioned to success state
  await waitFor(() => result.current.isSuccess)

  expect(result.current.data).toBeDefined()
}
```

更新：

[@testing-library/react v13.1.0](https://github.com/testing-library/react-testing-library/releases/tag/v13.1.0)也有一个新的[渲染hook](https://github.com/TanStack/query/blob/ead2e5dd5237f3d004b66316b5f36af718286d2d/src/react/tests/utils.tsx#L6-L17)。然而，它并不返回自己的`waitFor`，因此你需要从[@testing-library/react导入](https://testing-library.com/docs/dom-testing-library/api-async/#waitfor)。其API有些不同，它不允许返回布尔值，而是期望返回一个`Promise`。这意味着我们必须稍微调整我们的代码：

```tsx
import { waitFor, renderHook } from '@testing-library/react'

test("my first test", async () => {
  const { result } = renderHook(() => useCustomHook(), {
    wrapper: createWrapper()
  })

  // ✅ return a Promise via expect to waitFor
  await waitFor(() => expect(result.current.isSuccess).toBe(true))

  expect(result.current.data).toBeDefined()
}
```

## 静默错误控制台

默认情况下，React Query将错误打印到控制台。我认为这在测试过程中相当令人困扰，因为即使所有测试都是绿色的，你也会在控制台上看到🔴。React Query允许通过[设置日志记录器](https://tanstack.com/query/latest)来覆盖默认行为，这通常是我所做的：

```tsx
import { setLogger } from 'react-query'

setLogger({
  log: console.log,
  warn: console.warn,
  // ✅ no more errors on the console
  error: () => {},
})
```

更新：

`setLogger`已经从v4中移除了。取而代之的，你可以把你自定义的logger作为参数传递给你所创建的`QueryClient`:

```tsx
const queryClient = new QueryClient({
  logger: {
    log: console.log,
    warn: console.warn,
    // ✅ no more errors on the console
    error: () => {},
  }
})
```

此外，在生产模式下不再记录错误，以避免混淆。

## 将它们整合在一起

我创建了一个快速的存储库，将所有这些内容完美地结合在一起：mock-service-worker、react-testing-library和上述的包装器。它包含四个测试 - 用于自定义hook和组件的失败和成功示例。请在此处查看：https://github.com/TkDodo/testing-react-query
