事发经过：
       生产环境数据异常，经过排查发现一个纯后端任务在短时间内执行了多次。该任务本为xxl-job触发，且当日只应该执行一次。

异常原因：
       根据业务场景分析，此任务原来是单独的定时任务。在业务改造中被改成了通过其他业务模块调用触发执行。跨模块调用使用了openFeign组件。根据日志分析和代码分析，怀疑是因为该任务执行时间较长，超过了等待返回时常，触发了重试机制。而此任务本身作为定时任务为月度触发一次，没有做幂等性处理。导致业务逻辑重复执行

修改方式：
      本业务跨模块调用不需要使用返回结果，使用异步调用的方式进行触发，不等待任务执行结果


openFeign配置的相关参数
```
    @Bean
    public Retryer feignRetry() {
        return new Retryer.Default();
    }

```

使用Default策略的默认执行原则：
Retryer.Default 是 Feign 提供的一个简单且常用的重试策略。它的核心逻辑可以概括为以下几点：

最大重试次数 (Max Attempts): 默认情况下，Retryer.Default 会尝试 5 次，包括初始请求和随后的 4 次重试。

重试间隔 (Interval):

Retryer.Default 采用一种**指数退避（Exponential Backoff）**策略来计算每次重试之间等待的时间。

初始间隔： 默认从 100 毫秒 (ms) 开始。

最大间隔： 默认最大间隔是 1000 毫秒 (1 秒)。

计算方式： 每次重试时，等待时间会增加，但不会超过最大间隔。具体来说，它会在 initialInterval 和 maxInterval 之间生成一个随机值，然后这个值会随着重试次数的增加而指数增长（但同样不会超过 maxInterval）。

伪代码逻辑： currentInterval = min(maxInterval, initialInterval * (1.5 ^ attempt))

实际上，为了避免所有客户端同时重试导致的“惊群效应”，Retryer.Default 会在每次计算出的间隔上增加一个小的随机抖动。

重试条件：
Retryer.Default 只对以下类型的异常进行重试：

RetryableException： 这是 Feign 特有的一个异常，表示请求可以被重试。通常当网络连接问题、读取超时等情况发生时，Feign 内部会抛出或包装为 RetryableException。

IOException (例如连接超时、Socket 异常)： 这些通常是临时的网络问题。

注意： 它不会对 HTTP 状态码（如 5xx 服务器错误，4xx 客户端错误）进行重试，除非 Feign 的 ErrorDecoder 将这些状态码解析并抛出了 RetryableException。默认的 ErrorDecoder 行为是：

如果响应状态码是 503 Service Unavailable，并且响应头中包含 Retry-After，它会抛出 RetryableException 并使用 Retry-After 指定的延迟。

否则，对于其他 5xx 错误，它默认不会抛出 RetryableException，而是抛出 FeignException。这意味着如果你想对所有 5xx 错误进行重试，你需要自定义 ErrorDecoder。

不重试的异常：

FeignException (非 RetryableException 的 Feign 异常)： 通常是由于 HTTP 状态码非 2xx 引起的，比如 4xx 客户端错误，或 5xx 服务器错误但没有匹配 RetryableException 条件。

任何 RuntimeException 的子类 (非 IOException 且非 RetryableException)： 这通常表示代码逻辑错误或不应该重试的业务异常。

总结 Retryer.Default 的默认行为：
重试次数： 最多 5 次 (1 次初始请求 + 4 次重试)。

首次延迟： 大约 100ms。

最大延迟： 1000ms。

延迟策略： 指数退避，带随机抖动。

重试触发条件： RetryableException 和 IOException。
