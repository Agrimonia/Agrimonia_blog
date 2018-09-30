+++
title = "Vue-Apollo 官方文档拾遗"
date = 2018-09-10T03:13:55+08:00
tags = ["Vue", "GraphQL"]
categories = ["前端"]
draft = false
+++

# Vue-Apollo 官方文档拾遗

实习期间做的 Vue 项目因为用到了 GraphQL，自然接触了 Vue-Apollo。Vue-Apollo 的文档用起来没有很大问题，但它属于 Apollo Client 的实现，如果你没有对照 Apollo Client 的文档看，可能会觉得它有点语焉不详，下面是我使用时总结的一些经验。由于 Vue-Apollo 没有中文文档，下文尽量采用原词来保证指代明确。

## 数据更新

原则上我们应该尽可能减少发起 Query 的数量，而 Apollo Query 会缓存并订阅结果，这能够应付通常情况下对数据新鲜度的需求。而如果需要实时更新数据，可以采用轮询的方式定期更新，如果需要响应用户操作来更新数据，可以在 Click 绑定的 handler 里使用 `refetch()`。参考 Apollo Client 文档的 [Queries](https://www.apollographql.com/docs/react/essentials/queries.html#refetching) 部分。

## 更改 Apollo 返回的数据

Apollo 返回的数据是只读的，但是经常需要对其进行更改。这种情况可以通过做一次深拷贝来解决，比如 loadsh 的 cloneDeep() 或者 `JSON.parse(JSON.stringfy())`。

## 避免 query 结果的相互依赖

写在 apollo 对象里的 queries 有先后顺序，但 query 请求们几乎是同时发出的，所以如果 query 结果之间有依赖关系，比如你在 `result()` 函数里调用了另一个 query 的结果，就会因为返回的时序出问题。尽量避免这种写法。时序问题可以通过分拆出子组件的方式解决，相当于把其中一个依赖项换成了 props。实在不行也可以用拿到数据再 `addSmartQuery()`的方式暴力解决（不推荐）。

## update

> to customize the value that is set in the vue property, for example if the field names don't match.

也许我视力不行，update 这个选项的一句话描述我第一次看没 get 到点上，后来出现了 **field names don‘t match** 的情况才发现它很好用。

有时候 GraphQL document 里的 field name 和我们需要的变量名不吻合，比如：

```Json
allUser: {
    id
    // name
}
```

但我们需要一个叫 userId 的数组，就可以用 `update: data => data.allUser.id`解决。

如果 Schema 里我们需要的数据在内层 ，比如 `data.config.userConfig.avatar`，这样写起来更方便了。

如果我们只需要部分数据，比如我不需要 `name = 'robot'` 的 user，这时候`update: data => data.allUser.filter(user => user.name != 'robot')` 是很方便的过滤方式（用 `result() `也行）。

## 临时 query

如果我只需要临时发起一个 query，只需要像 mutation 那样结合 async/await 来用：

```
this.$apollo.query({
    query: xxx
})
```

这样做的好处是它不会被保存到 `this.$apollo.quries` 里，而不像 `addSmartQuery()`。

## 错误处理

可以在 apolloProvider 的 Constructor 里添加 Global error handler，来为所有的 smart queries 和 subscriptions 加上报错。

## 额外的 options

Smart Query 支持全部的 [graphql-query-options](https://www.apollographql.com/docs/react/api/react-apollo.html#graphql-query-options)，而API 文档上略去了一些。补充几个比较常用的：

`fetchPolicy`: allows you to specify how you want your component to interact with the Apollo data cache. See [options.fetchPolicy](https://www.apollographql.com/docs/react/api/react-apollo.html#graphql-config-options-fetchPolicy).

`pollInterval`: The interval in milliseconds at which you want to start polling. See [options.pollInterval](https://www.apollographql.com/docs/react/api/react-apollo.html#graphql-config-options-pollInterval).

`context`: Everything under the `context` object gets passed directly to your network chain. This can be used to do things like **set `headers` on HTTP requests from props**, **control which endpoint you send a query to**, and so much more depending on what links your app is using. See [ApolloLink-context](https://www.apollographql.com/docs/link/overview.html#context).

