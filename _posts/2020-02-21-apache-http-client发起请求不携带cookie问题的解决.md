---
layout: post
title: "apache-http-client发起请求不携带cookie问题的解决"
date: 2020-02-21 10:40:34 +0800
categories: 日常工作
---

> 最近公司的项目在使用http接口作为其他项目的服务提供者，调用的时候需要使用`apache`的`http-client`作为发起http请求的基础部件。使用过程中发现请求时无法携带cookie，上网查阅资料并查找源码，最终问题得以解决并记录如下。

<!-- more -->

### 起因

---

- 目前在公司负责的项目需要作为一个服务提供者，为公司的其他项目提供http接口服务；为了方便其他项目组的调用，使用`http-client`封装了一个jar包，外部项目调用的时候直接使用jar包封装好的方法进行调用，降低外部系统调用的难度，并可以保证调用方式的正确性。在封装并调试的过程中，发现了一个问题，就是直接使用`http-client`调用的时候不会携带cookie，即使按照网上教程、官网文档的方式、依旧无法添加cookie。请求时如果不能携带cookie，则被调用方会因为没有cookie，而重新创建session；目前该项目的session是存储在redis里，虽然每个seesion占用的空间并不大，但是如果`http-client`发起的请求过多，那么redis中存储的session依旧有可能占满redis的内存，同时对于系统问题的排除也比较不利，故需要解决。

### 解决过程

---

- 网上能找到最多的代码大概类似于下面这样

```
BasicCookieStore cookieStore = new BasicCookieStore();
BasicClientCookie cookie = new BasicClientCookie("cookieName", "cookieValue");
cookie.setDomain("test.com"); //cookie的域名
cookie.setPath("/");  //cookie的域名的存储路径
cookieStore.addCookie(cookie);
HttpClient client = HttpClientBuilder.create().setDefaultCookieStore(cookieStore).build();
final HttpGet requestGET = new HttpGet("http://test.com");
HttpResponse res = client.execute(requestGET);
```

- 但是经过实际的测试，发现并不能添加cookie，遂继续google。直到找到了[这篇blog](https://www.jelliclecat.cn/articles/Java/HttpClientBug)，虽然这篇blog没有解决我的问题，但是阅读大致能够缕清`http-client`中对于cookie的处理方式，再结合[官网doc](https://hc.apache.org/httpcomponents-client-4.5.x/tutorial/html/statemgmt.html#d5e566)上对于cookie处理部分的解释，大致有了思路：

  - `http-client`对于cookie的添加由所谓的`cookie policy`决定，当没有设置`cookie policy`是，会默认使用`DefaultCookieSpecProvider`的`create()`方法，创建一个cookie处理策略`cookieSpec`。

  - 默认处理策略`cookieSpec`创建后，会调用其中的`match()`方法判断cookie是否符合规则，符合规则后，再调用`cookieSpec.formatCookies()`方法，格式化cookie，并最终添加到请求的header里，至此完成cookie的添加。

  - 源码如下

    ```
    //从上下文获取请求设置
    final RequestConfig config = clientContext.getRequestConfig();
    //获取cookie处理策略
    String policy = config.getCookieSpec();
    //cookie处理策略为空，则使用默认的处理策略
    if (policy == null) {
        policy = CookieSpecs.DEFAULT;
    }
    //此处省略无关代码
    ...
    // Get an instance of the selected cookie policy
    // 上面一行是源码带的注释，就是从 registry 获取 CookieSpecProvider
    final CookieSpecProvider provider = registry.lookup(policy);
    if (provider == null) {
        if (this.log.isDebugEnabled()) {
            this.log.debug("Unsupported cookie policy: " + policy);
        }
        return;
    }
    // 调用 CookieSpecProvider 获取 cookie 处理策略实例
    final CookieSpec cookieSpec = provider.create(clientContext);
    // Get all cookies available in the HTTP state
    // 获取cookie
    final List<Cookie> cookies = cookieStore.getCookies();
    // Find cookies matching the given origin
    final List<Cookie> matchedCookies = new ArrayList<Cookie>();
    final Date now = new Date();
    boolean expired = false;
    for (final Cookie cookie : cookies) {
        if (!cookie.isExpired(now)) {
        // 逐一比较cookie是否符合规则，符合规则的才能添加，就是这里出了问题才导致cookie没有成功添加到请求
            if (cookieSpec.match(cookie, cookieOrigin)) {
                if (this.log.isDebugEnabled()) {
                    this.log.debug("Cookie " + cookie + " match " + cookieOrigin);
                }
                matchedCookies.add(cookie);
            }
        } else {
            if (this.log.isDebugEnabled()) {
                this.log.debug("Cookie " + cookie + " expired");
            }
            expired = true;
        }
    }
    // Per RFC 6265, 5.3
    // The user agent must evict all expired cookies if, at any time, an expired cookie
    // exists in the cookie store
    if (expired) {
        cookieStore.clearExpired(now);
    }
    // Generate Cookie request headers
    if (!matchedCookies.isEmpty()) {
    // 使用cookie策略实例将cookie处理为headers，至此添加cookie完成
        final List<Header> headers = cokieSpec.formatCookies(matchedCookies);
        for (final Header header : headers) {
            request.addHeader(header);
        }
    }
    ```

    

- 有了上面的处理过程就好说了，结合官网doc的说明，首先创建一个类实现`CookieSpecProvider`接口

```
public class SingleCookieSpecProvider implements CookieSpecProvider {
    @Override
    public CookieSpec create(HttpContext context) {
        return new SingleCookieSpec();
    }
}
```

​	然后创建一个类实现`CookieSpec`接口：

```
public class SingleCookieSpec implements CookieSpec {

    @Override
    public int getVersion() {
        return 0;
    }

    @Override
    public List<Cookie> parse(Header header, CookieOrigin origin) throws MalformedCookieException {
        return null;
    }

    @Override
    public void validate(Cookie cookie, CookieOrigin origin) throws MalformedCookieException {

    }

    @Override
    public boolean match(Cookie cookie, CookieOrigin origin) {
    	// 直接返回true，保证cookie能直接添加
        return true;
    }

    @Override
    public List<Header> formatCookies(List<Cookie> cookies) {
    	// 将cookie转为headers
        Args.notNull(cookies, "List of cookies");
        return doFormatOneHeader(cookies);
    }

    private List<Header> doFormatOneHeader(final List<Cookie> cookies) {
        final CharArrayBuffer buffer = new CharArrayBuffer(40 * cookies.size());
        buffer.append(SM.COOKIE);
        buffer.append(": ");
        for (final Cookie cooky : cookies) {
            final Cookie cookie = cooky;
            formatCookieAsVer(buffer, cookie);
            buffer.append("; ");
        }
        final List<Header> headers = new ArrayList<Header>(1);
        headers.add(new BufferedHeader(buffer));
        return headers;
    }

    protected void formatCookieAsVer(final CharArrayBuffer buffer,
                                     final Cookie cookie) {
        formatParamAsVer(buffer, cookie.getName(), cookie.getValue());
        if (cookie.getPath() != null) {
            if (cookie instanceof ClientCookie
                    && ((ClientCookie) cookie).containsAttribute(ClientCookie.PATH_ATTR)) {
                buffer.append("; ");
                formatParamAsVer(buffer, "$Path", cookie.getPath());
            }
        }
        if (cookie.getDomain() != null) {
            if (cookie instanceof ClientCookie
                    && ((ClientCookie) cookie).containsAttribute(ClientCookie.DOMAIN_ATTR)) {
                buffer.append("; ");
                formatParamAsVer(buffer, "$Domain", cookie.getDomain());
            }
        }
    }

    protected void formatParamAsVer(final CharArrayBuffer buffer,
                                    final String name, final String value) {
        buffer.append(name);
        buffer.append("=");
        if (value != null) {
           buffer.append(value);
        }
    }

    @Override
    public Header getVersionHeader() {
        return null;
    }
}
```

最后，`http-client`调用方法如下

```
// 设置下cookie
BasicCookieStore cookieStore = new BasicCookieStore();
BasicClientCookie cookie = new BasicClientCookie(cookie.getName(),cookie.getValue());
cookie.setDomain(cookie.getDomain());
cookie.setPath(cookie.getPath());
cookie.setAttribute(ClientCookie.PATH_ATTR, cookie.getPath());
cookie.setAttribute(ClientCookie.DOMAIN_ATTR, cookie.getDomain());
// cookie添加到cookieStore中
cookieStore.addCookie(cookie);
// cookieStore 添加到 httpclient 实例中
CloseableHttpClient httpclient = HttpClients.custom()
        .setDefaultCookieStore(cookieStore).build();

// request context 设置
// 将cookie设置到上下文中
HttpClientContext context = HttpClientContext.create();
context.setCookieStore(cookieStore);
// 注册cookie处理策略
Registry<CookieSpecProvider> registry = RegistryBuilder.<CookieSpecProvider>create()
        .register("singleCookieSpec", new SingleCookieSpecProvider())
        .build();
// 设置使用的cookie策略
RequestConfig requestConfig = RequestConfig.custom()
        .setCookieSpec("singleCookieSpec")
        .build();
context.setRequestConfig(requestConfig);
// 设置注册器
context.setCookieSpecRegistry(registry);
HttpGet httpget = new HttpGet(url);
// 设置超时毫秒
int timeout = 10000;
RequestConfig requestConfig = RequestConfig.custom()
                .setSocketTimeout(timeout)
                .setConnectTimeout(timeout)
                .setConnectionRequestTimeout(timeout)
                .setCookieSpec(SINGLE_COOKIE_SPECIFICATION_NAME)
                .build();
httpget.setConfig(requestConfig);
// 执行get请求.
CloseableHttpResponse response = httpclient.execute(httpget, context);
```

### 总结

---

遇到问题的时候，如果网上的资料不能解决问题，还是得看源码，程序的工作原理都在代码里。