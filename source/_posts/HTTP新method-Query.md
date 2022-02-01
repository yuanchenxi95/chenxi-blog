---
title: 'HTTP新method: Query'
date: 2022-01-31 23:50:36
tags: HTTP
---

About: IETF最近的一个关于HTTP新method的proposal

## GET为什么不应该有body
根据 [HTTP Spec][5]
```
A payload within a GET request message has no defined semantics
```
GET request的payload是没有定义的，可以理解为[C语言中的undefined behavior][1]，文章中还提到了，
GET bodies从不被 [Fetch][4] 支持，变成了虽然支持，但是会警告开发者尽量不要用。

因为是 undefined behavior，所以 GET 请求中的 Body，理论上来讲可以被随意处理，像HackerNews
上这个 [Comment][6] 所说的:
```
... 
GET with a body is iffy. If you owned the entire stack it might be doable. 
...
Proxies or firewalls can just drop the message and be completely HTTP compliant.
...
```
GET的body是具有不确定性的，Load Balancer理论上是可以直接丢掉Body内容的。

## 新Method "QUERY"有什么好处

### Safe and Idempotent
[Proposal Section 2][7]
里提到的最重要的好处是safe和Idempotent。

对比POST，QUERY不应改变服务器上的内容，也就是只读请求，而且QUERY同样也应该是幂等的，
"以相同的请求调用这个接口一次和调用这个接口多次，对系统产生的影响是相同的"。

### Caching
[Proposal Section 2.1][2] 所提到的，QUERY支持[HTTP caching的标准][3]。

综合来看，QUERY相当于是一个支持了Body的GET。对比传统通过POST来传Body实现复杂查询，有一些优势。

## Thoughts
HTTP定义的这些方法在当前真的符合各个不同应用的需求吗，HTTP是否应该在它这一层去定义这些方法，
是否应该由一层只处理单向或双向的消息通讯协议， 其余的逻辑，由更上层的协议去做。
像GraphQL在HTTP POST之上定义自己的Query和Mutation逻辑。


# References:
1. [Request bodies in GET requests](https://evertpot.com/get-request-bodies/)
2. [The HTTP QUERY Method](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-safe-method-w-body-02)
3. [Hackernews Thread: Request bodies in GET requests](https://news.ycombinator.com/item?id=30129631)

[1]: <https://evertpot.com/get-request-bodies/>
[2]: <https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-safe-method-w-body-02#section-2.1>
[3]: <https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-cache-19#section-4>
[4]: <https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API>
[5]: <https://datatracker.ietf.org/doc/html/rfc7231#section-4.3.1>
[6]: <https://news.ycombinator.com/item?id=30155696>
[7]: <https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-safe-method-w-body-02#section-2>
