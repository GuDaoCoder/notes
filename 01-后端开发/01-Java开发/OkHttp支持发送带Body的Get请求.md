---
tags:
- OkHttp
- Java

---

## 前言

http协议不推荐Get请求带Body，可能会带来一些未知问题，但是也没有明确禁止这种行为。<!-- more -->

在[http协议文档](https://datatracker.ietf.org/doc/html/rfc7231#section-4.3.1)中4.3.1一节中有对于Get请求带Body的说明。

```
A payload within a GET request message has no defined semantics;sending a payload body on a GET request might cause some existing mplementations to reject the request.
```

## OkHttp本身禁止发送带Body的Get请求

OkHttp严格遵守http协议，禁止发送带Body的Get请求。在OkHttp中构建Request的Builder类并没有提供带参数的get方法。

![image-20240621181742957](http://cdn.road4code.com/image-bed/20240621181743.png)

而当我们试图通过method方法去构建带Body的Get请求时则是会直接抛出异常。

![image-20240621181918449](http://cdn.road4code.com/image-bed/20240621181918.png)

在Github上有人提了相关的[Issue](https://github.com/square/okhttp/issues/3154)，官方对此作出了回应表示不会修改，顺便还嘲讽了下Elasticsearch(*^▽^*)。

![image-20240624104437452](http://cdn.road4code.com/image-bed/20240624104437.png)

## OkHttp支持发送带Body的Get请求

如果只能使用OkHttp客户端且接口提供方无法修改，可以参考以下方式。

翻看OkHttp源码，OkHttp没有直接提供任何修改的方式。所以我们只能先通过Post将Body传入到Request中，再通过反射强行将Method修改为Get以此绕过OkHttp的限制。

``` java
public class FixGetWithBody {

    public static class GetRequestBody extends okhttp3.Request.Builder {
        /**
         * 实现get请求带body的方法，先通过原有post方法传递body参数，然后通过反射修改method字段绕过校验
         *
         * @param body
         * @return Builder
         **/
        public okhttp3.Request.Builder get(RequestBody body) {
            this.post(body);
            try {
                Field field = okhttp3.Request.Builder.class.getDeclaredField("method");
                field.setAccessible(true);
                field.set(this, HttpMethod.GET.name());
            } catch (Exception e) {
            }
            return this;
        }
    }
}
```

在构建Request对象时使用我们自己的Builder。

```java
		/**
     * 修复后的发送方法，如果是GET请求带body，则使用自定义的FixGetWithBody，并添加header标记
     *
     * @param url
     * @param method
     * @param body
     * @return Response
     **/
    private Response sendFix(String url, HttpMethod method, RequestBody body) throws Exception {
        Headers.Builder headerBuilder = new Headers.Builder();
        Request.Builder requestBuilder;
        if (StringUtils.equalsAnyIgnoreCase("GET", method.name()) && body != null) {
            requestBuilder = new FixGetWithBody.GetRequestBody().get(body);
            // 这里在header中添加了个标记，具体作用下面解释
            headerBuilder.add(FixGetWithBody.HEAD_KEY, FixGetWithBody.HEAD_VALUE);
        } else {
            requestBuilder = new Request.Builder().method(method.name(), body);
        }
        Request request = requestBuilder.url(url).headers(headerBuilder.build()).build();
        return client.newCall(request).execute();
    }
```

这样就可以直接发送带Body的Get请求了。

当然，到这里还没有结束。虽然我们绕过了发送的校验，但是OkHttp在处理请求返回时也会根据请求方式和是否有Body作出不同的处理。OkHttp源码的CallServerInterceptor类47行可以看到相关的代码。

![image-20240624105414247](http://cdn.road4code.com/image-bed/20240624105414.png)

万幸的是这次OkHttp在第43行提供了个可以修改的入口。

![image-20240624105644486](http://cdn.road4code.com/image-bed/20240624105644.png)

这里提供了requestHeadersStart和requestHeadersEnd两个事件，我们需要在requestHeadersEnd事件中根据之前添加的Header标记选择将Request对象中的method再次通过反射修改回Post。

```java
    public static class CustomEventListenerImpl extends EventListener {
        /**
         * 发送完成后需要将method修改回post，否则后续请求会校验失败
         *
         * @param call
         * @param request
         * @return void
         **/
        public void requestHeadersEnd(Call call, Request request) {
            if (HEAD_VALUE.equals(request.header(HEAD_KEY))) {
                try {
                    Field field = Request.class.getDeclaredField("method");
                    field.setAccessible(true);
                    field.set(request, HttpMethod.POST.name());
                } catch (Exception ignored) {
                }
            }
        }
    }
```

## 总结

以上的修改方式不算完美，只能算是不得已而为之的方法了。完整的源码请查看https://github.com/GuDaoCoder/code-demo/tree/main/okhttp-get-with-body中的单元测试类。