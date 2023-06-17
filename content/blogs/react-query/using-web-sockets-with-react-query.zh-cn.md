---
title: "使用React Query和Web Sockets"
date: 2023-06-17T14:48:58+08:00
author: "Feng Xue"
tags: ["React", "React Query", "Typescript", "Translation", "Blogs"]
toc: true
usePageBundles: false
draft: false
---

> 本文翻译自 [TkDodo](https://github.com/tkdodo) 的 [Using WebSockets with React Query](https://tkdodo.eu/blog/using-web-sockets-with-react-query)

如何使用WebSockets与React Query处理实时数据是最近最常被问到的问题之一，因此我想尝试一下，并报告我的发现。这就是本文的内容 :)

## 什么是WebSockets

简而言之，WebSockets允许从服务器推送消息或"实时数据"到客户端（浏览器）。通常情况下，使用HTTP时，客户端向服务器发出请求，希望获取一些数据，服务器响应该数据或错误，然后连接关闭。

由于客户端是打开连接并发起请求的一方，这就没有机会让服务器在有更新可用时向客户端推送数据。

这就是[WebSockets](https://en.wikipedia.org/wiki/WebSocket)的作用。

就像其他HTTP请求一样，浏览器发起连接，但表示希望将连接升级为WebSocket。如果服务器接受了这个请求，它们将切换协议。这个连接不会终止，而是保持打开状态，直到任一方决定关闭它。现在，我们拥有了一个完全功能的双向连接，双方都可以传输数据。

这主要的优点是服务器现在可以向客户端推送选择性的更新。如果多个用户查看相同的数据，并且其中一个用户进行了更新。通常情况下，其他客户端在主动刷新之前不会看到该更新。而使用WebSockets可以实时推送这些更新。

## React Query 集成

由于React Query主要是一个客户端异步状态管理库，所以我不会讨论如何在服务器上设置WebSockets。老实说，我从没做过，而且还取决于你在后端使用的技术。

React Query没有专门为WebSockets构建的内置功能。这并不意味着不支持WebSockets，或者它们与库的兼容性不好。只是当涉及到数据获取时，React Query在使用方式上非常不偏不倚：它只需要一个已解决或拒绝的`Promise`即可工作 - 其余的由你决定。

## 逐步进行

一般的思路是按照通常的方式设置查询，就像你不使用WebSockets一样。大多数情况下，你将拥有用于查询和更改实体的常规HTTP端点。

```jsx
const usePosts = () =>
  useQuery({ queryKey: ['posts', 'list'], queryFn: fetchPosts })

const usePost = (id) =>
  useQuery({
    queryKey: ['posts', 'detail', id],
    queryFn: () => fetchPost(id),
  })
```

此外，你可以设置一个全局的`useEffect`来连接到WebSocket终端。具体如何操作完全取决于你使用的技术。我看到过有人从[Hasura](https://github.com/TanStack/query/issues/171#issuecomment-649810136)订阅实时数据。有一篇很棒的文章介绍了如何连接到[Firebase](https://aggelosarvanitakis.medium.com/a-real-time-hook-with-firebase-react-query-f7eb537d5145)。在我的示例中，我将简单地使用浏览器原生的[WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)：

```jsx
const useReactQuerySubscription = () => {
  React.useEffect(() => {
    const websocket = new WebSocket('wss://echo.websocket.org/')
    websocket.onopen = () => {
      console.log('connected')
    }

    return () => {
      websocket.close()
    }
  }, [])
}
```

### 处理数据

在设置连接之后，当WebSocket上有数据传入时，我们很可能会有一些回调函数会被调用。再次强调，数据的具体内容完全取决于你的设置方式。受到 [Tanner Linsley](https://github.com/tannerlinsley)发布的[这条信息](https://github.com/TanStack/query/issues/171#issuecomment-649716718)的启发，我喜欢从后端发送事件而不是完整的数据对象：

```jsx
const useReactQuerySubscription = () => {
  const queryClient = useQueryClient()
  React.useEffect(() => {
    const websocket = new WebSocket('wss://echo.websocket.org/')
    websocket.onopen = () => {
      console.log('connected')
    }
    websocket.onmessage = (event) => {
      const data = JSON.parse(event.data)
      const queryKey = [...data.entity, data.id].filter(Boolean)
      queryClient.invalidateQueries({ queryKey })
    }

    return () => {
      websocket.close()
    }
  }, [queryClient])
}
```

这就是在接收到事件时更新列表和详细视图的全部内容了。

* `{ "entity": ["posts", "list"] }` 将使post list无效
* `{ "entity": ["posts", "detail"], id: 5 }` 将使单个post无效
* `{ "entity": ["posts"] }` 将使所有和post有关的无效

[查询无效](https://tanstack.com/query/latest/docs/react/guides/query-invalidation?from=reactQueryV3&original=https%3A%2F%2Ftanstack.com%2Fquery%2Fv3%2Fdocs%2Fguides%2Fquery-invalidation)与WebSockets非常匹配。这种方法避免了过度推送的问题，因为如果我们接收到的事件与我们当前不感兴趣的实体相关，什么都不会发生。例如，如果我们当前在个人资料页面，而我们接收到了post的更新，`invalidateQueries`将确保下次我们进入post页面时会重新获取数据。然而，它不会立即重新获取数据，因为我们没有活动的观察者。如果我们再也不去那个页面，推送的更新将是完全不必要的。

### 更新部分数据

当然，如果你有大型数据集，但是频繁接收小的更新，你可能仍然希望通过WebSocket推送部分数据。

post的标题发生了变化？只需推送标题。点赞数发生变化？也推送下来。

对于这些部分更新，你可以使用[queryClient.setQueryData](https://tanstack.com/query/latest/docs/react/reference/QueryClient?from=reactQueryV3&original=https%3A%2F%2Ftanstack.com%2Fquery%2Fv3%2Fdocs%2Freference%2FQueryClient#queryclientsetquerydata)直接更新查询缓存，而不是使其无效。

如果你的同一数据有多个查询键，例如，如果你的查询键中包含多个过滤条件，或者如果你想使用同一条消息更新列表视图和详细视图，那么这将变得有些麻烦。[queryClient.setQueriesData](https://tanstack.com/query/latest/docs/react/reference/QueryClient?from=reactQueryV3&original=https%3A%2F%2Ftanstack.com%2Fquery%2Fv3%2Fdocs%2Freference%2FQueryClient#queryclientsetquerydata)是该库的一个相对较新的功能，它允许你处理这种情况：

```jsx
const useReactQuerySubscription = () => {
  const queryClient = useQueryClient()
  React.useEffect(() => {
    const websocket = new WebSocket('wss://echo.websocket.org/')
    websocket.onopen = () => {
      console.log('connected')
    }
    websocket.onmessage = (event) => {
      const data = JSON.parse(event.data)
      queryClient.setQueriesData(data.entity, (oldData) => {
        const update = (entity) =>
          entity.id === data.id ? { ...entity, ...data.payload } : entity
        return Array.isArray(oldData) ? oldData.map(update) : update(oldData)
      })
    }

    return () => {
      websocket.close()
    }
  }, [queryClient])
}
```

对于我个人而言，这种方法有点太动态了，无法处理添加或删除，并且 TypeScript 对此也不太友好，所以我个人更倾向于使用查询无效。

尽管如此，在这里有一个[CodeSandbox示例](https://codesandbox.io/s/react-query-websockets-ep1op)，我在其中处理了无效和部分更新两种类型的事件。（_注意：在示例中，自定义hook有点复杂，因为我使用相同的WebSocket模拟了服务器的往返。如果你有一个真实的服务器，就不必担心这个问题_）。

### 增加过期时间

React Query的[默认过期时间](https://tanstack.com/query/latest/docs/react/guides/important-defaults?from=reactQueryV3&original=https%3A%2F%2Ftanstack.com%2Fquery%2Fv3%2Fdocs%2Fguides%2Fimportant-defaults)是0。这意味着每个查询都会立即被视为过期，也就是说，当新的订阅者挂载或用户重新聚焦窗口时，它会重新获取数据。这是为了保持数据的最新状态。

这个目标与WebSockets的目标有很多重叠，因为WebSockets实时更新数据。如果服务器刚刚通过专用消息告诉我手动使其 _无效_，为什么我还需要重新获取数据呢？

因此，如果你无论如何都通过WebSockets更新所有数据，请考虑设置一个较长的`staleTime`。在我的示例中，我只是使用了`Infinity`。这意味着数据将通过`useQuery`进行初始获取，然后始终从缓存中获取。重新获取只会通过显式的查询无效来发生。

在创建`QueryClient`时，通过设置全局查询默认值可以最好地实现这一点。

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: Infinity,
    },
  },
})
```

今天就到这里。如果你有任何问题，请随时在 Twitter 上与我联系，或者在下面留言。
