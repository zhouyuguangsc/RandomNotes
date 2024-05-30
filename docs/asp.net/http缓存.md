# 一. 什么是Http缓存
### 1. 强缓存
定义：当命中强缓存的时候，客户端不会再请求服务器，直接从缓存中读取内容，并返回HTTP状态码200。  
强制缓存，在响应头由 Expires、Cache-Control 和 Pragma控制，如下：  
- Expires：值为服务器返回的过期时间，浏览器再次加载资源时，如果在这个过期时间内，则命中强缓存。（HTTP1.0的属性，缺点是客户端和服务器时间不一致会导致命中误差）
- Cache-Control：HTTP1.1属性，优先级更高，以下为常用属性
    - no-store： 禁用缓存
    - no-cache：不使用强缓存，每次需向服务器验证缓存是否失效
    - private/public：private指的单个用户，public可以被任何中间人、CDN等缓存
    - max-age=：max-age是距离请求发起的时间的秒数
    - must-revalidate：在缓存过期前可以使用，过期后必须向服务器验证
- Pragma
    - no-cache：效果和cache-control等no-cache一致。优先级Pragma > Cache-Control > Expires

参考-[HTTP缓存的三种方式详解-稀土掘金](https://juejin.cn/post/7063861101041025031)

### 2. 协商缓存
定义：向服务器发送请求，服务器会根据这个请求的请求头的一些参数来判断是否命中协商缓存，如果命中，则返回304状态码并带上新的响应头通知浏览器从缓存中读取资源  
协商缓存，响应头中有两个字段标记规则，如下：  
- Last-Modified / If-Modified-Since
    - Last-Modified是浏览器第一个请求资源，服务器响应头字段，是资源文件最后一次更改时间(精确到秒)。
    - 下一次发送请求时，请求头里的If-Modified-Since就是之前的Last-Modified
    - 服务器更加最后修改时间判断命中，如果命中，http为304且不返回资源、不返回last-modify
- Etag / If-None-Match：Etag 的校验优先级高于 Last-Modified
    - Etag是加载资源时，服务器返回的响应头字段，是对资源的唯一标记，值是hash码。
    - 浏览器在下一次加载资源向服务器发送请求时，会将上一次返回的Etag值放到请求头里的If-None-Match里
    - 服务器接受到If-None-Match的值后，会拿来跟该资源文件的Etag值做比较，如果相同，则表示资源文件没有发生改变，命中协商缓存。


# 二. 粗略实现
```
// 中间件，如果在action中操作响应头或响应状态容易出错
return app.Use((context, next) =>
{
    // 这里借用OutputCache输出缓存的特性，作为识别标志
    var outputCache = context.GetEndpoint()?.Metadata?.GetMetadata<OutputCacheAttribute>();
    if (outputCache != null && !string.IsNullOrEmpty(outputCache.PolicyName))
    {
        var _memoryCache = app.ApplicationServices.GetRequiredService<IMemoryCache>();
        // 服务端为接口生成缓存标识，在数据发生修改时，则将这个缓存删除
        // 后续每次请求都和缓存比较是否一致
        var currETag = _memoryCache.GetOrCreate(outputCache.PolicyName, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(30);
            return $"\"{DateTime.Now.Ticks}\"";
        });
        // 设置响应head
        App.HttpContext.Response.Headers.ETag = currETag;
        // 比较是否修改，没有修改则返回304状态
        var ifNoneMatchHeader = App.HttpContext.Request.Headers.IfNoneMatch;
        if (ifNoneMatchHeader == currETag)
        {
            App.HttpContext.Response.StatusCode = StatusCodes.Status304NotModified;
            return Task.CompletedTask;
        }
    }
    return next(context);
});
```