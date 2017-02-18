> 摘录自[OkHttp wiki: Recipes](https://github.com/square/okhttp/wiki/Recipes)

# 基本使用
## 同步Get

```
private final OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

Response response = client.newCall(request).execute();
if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

Headers responseHeaders = response.headers();
for (int i = 0; i < responseHeaders.size(); i++) {
  System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
}

System.out.println(response.body().string());

```
## 异步Get
```
private final OkHttpClient client = new OkHttpClient();

Request request = new Request.Builder()
    .url("http://publicobject.com/helloworld.txt")
    .build();

client.newCall(request).enqueue(new Callback() {
  @Override 
  public void onFailure(Call call, IOException e) {
    e.printStackTrace();
  }

  @Override 
  public void onResponse(Call call, Response response) throws IOException {
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    Headers responseHeaders = response.headers();
    for (int i = 0, size = responseHeaders.size(); i < size; i++) {
      System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
    }

    System.out.println(response.body().string());
  }
});

```

## Header处理
```
    Request request = new Request.Builder()
        .url("https://api.github.com/repos/square/okhttp/issues")
        .header("User-Agent", "OkHttp Headers.java")
        .addHeader("Accept", "application/json; q=0.5")
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println("Server: " + response.header("Server"));
    System.out.println("Vary: " + response.headers("Vary"));
```

## Post String
使用RequestBody.create方法，post一个body
```
public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");
Request request = new Request.Builder()
    .url("https://api.github.com/markdown/raw")
    .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, postBody))
    .build();
```

## Post流
继承RequestBody，复写writeTo(BufferedSink)方法，可以Post流。
```
RequestBody requestBody = new RequestBody() {
      @Override public MediaType contentType() {
        return MediaType.parse("text/x-markdown; charset=utf-8");
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        sink.writeUtf8("Numbers\n");
        // ...
      }
    };

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(requestBody)
        .build();
```

## Post文件
```
File file = new File("README.md");

Request request = new Request.Builder()
    .url("https://api.github.com/markdown/raw")
    .post(RequestBody.create(MediaType.parse("text/x-markdown; charset=utf-8"), file))
    .build();
```

## Post表单

## Post MultiPart

## 使用Gson解析JSON响应
```
Gist gist = new Gson().fromJson(response.body().charStream(), Gist.class);
```

## response caching
```
int cacheSize = 10 * 1024 * 1024; // 10 MiB
Cache cache = new Cache(cacheDirectory, cacheSize);

client = new OkHttpClient.Builder()
    .cache(cache)
    .build();
```

## canceling call

## Timeouts设置

## 单个Call的重配置
如果我们需要在已有的配置上，修改OkHttpClient的部分参数，再进行请求的话，使用client.newBuilder().build()，构建一份拷贝再请求，注意新的client会和原来的client共用connectionPool和dispatcher。

## 处理Authentication

# 拦截器
## 最基本的拦截器示例：
```
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```
## 应用拦截器和网络拦截器

## Rewriting Request和Response
