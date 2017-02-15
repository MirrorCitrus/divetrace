> 共3篇：
* Part1: 使用篇
* [Part2: RxJava和Retrofit实践经验](网络库：Retrofit（二）：RxJava和Retrofit实践经验.md)
* [Part3: 原理篇](网络库：Retrofit（三）：原理篇.md)

# 概述
> **Retrofit:** A type-safe HTTP client for Android and Java

Retrofit是一个RESTful的HTTP网络请求框架的封装。注意这里并没有说它是网络请求框架，主要原因在于网络请求的工作并不是 Retrofit 来完成的。Retrofit 2.0 开始内置 OkHttp，前者专注于接口的封装，后者专注于网络请求的高效，二者分工协作。所以说，如果我们的网络接口是符合Restful规范的，使用Retrofit会非常方便。另一方面，Retrofit是支持RxJava的，这一点来说也是非常方便的。  
> 注： restful架构其实是一种架构风格，或者说，更类似于服务端提供API接口时应该遵循的规范，对于客户端来说，可能直接可以理解为：服务端提供的url是否符合规范。这几个规范主要包括：
> - 每一个URI代表一种资源；
> - 客户端和服务器之间，传递这种资源的某种表现层；
> - 客户端通过四个HTTP动词，对服务器端资源进行操作，实现"表现层状态转化"。

我们简单总结一下相关的特点：
- 注解特性：
    - 使用注解方式，轻松替换url中的路径和参数，拼接实际请求的url
    - 请求结果自动转换成responseBody（支持json, protobuf, 设置converter后直接转换成pojo)
    - 支持multipart, 支持文件上传
- 同步/异步请求支持；线程调度
- 灵活的配置

# 使用概述
## ‘Http-Client’的构造
采用Builder构造一个retrofit的实例，然后使用该实例去创建一个Service的动态代理。
```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```
## 多种配置的示例
- 最简单的GET请求
```
@GET("/list")
Call<ResponseBody> list(@Query("page") int page);
```
- Query & QueryMap
```
@GET("/list")
Call<ResponseBody> list(@Query("page") int page);
```
- Field & FieldMap
```
@FormUrlEncoded
@POST("/")   
Call<ResponseBody> example(@Field("name") String name, 
        @Field("occupation") String occupation);
```
- Part & PartMap
    
```
@Multipart
@OST("upload")    
Call<ResponseBody> upload(@Part("description") RequestBody description,
                              @Part MultipartBody.Part file);
```

## Converter: 入参和响应的转换
Converter<F, T>是一个泛型接口，定义了一个方法convert()，负责把F类型转换为T类型。retrofit的请求体必须是RequestBody类型，请求的响应最初是ResponseBody类型，入参和响应的转换都通过Converter。不过，我们倒不是随意在这两处添加转换器，我们需要通过实现Converter.Factory，来更加具体地指定如何转换请求体和响应内容。
例如一个将文件转换为二进制附件的转换器：
 
```
static class FileRequestBodyConverterFactory extends Converter.Factory {    
    @Override
    public Converter<File, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {      
       return new FileRequestBodyConverter();
    }
  }  
       
 static class FileRequestBodyConverter implements Converter<File, RequestBody> {    
    @Override
    public RequestBody convert(File file) throws IOException {      
      return RequestBody.create(MediaType.parse("application/otcet-stream"), file);
    }
  }
```
然后是一个自定义的响应的转换器：
```
static class MyResponseConverterFactory extends Converter.Factory {    
    @Override
    public Converter<ResponseBodu, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {      
       return new MyResponseConverter();
    }
  }  
       
 static class MyResponseConverter implements Converter<ResponseBodu, Result> {    
    @Override
    public Result convert(ResponseBody value) throws IOException {
      RawResult rawResult = new Gson().fromJson(value.string(), RawResult.class);
      Result result = new Result;
      result.body = rawResult.content;
      result.code = rawResult.err;
      return result;
    }
  }
```

# 进阶使用

# 参考
- [x] [理解RESTful架构](http://www.ruanyifeng.com/blog/2011/09/restful)
- [x] [深入浅出 Retrofit](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653577186&idx=1&sn=1a5f6369faeb22b4b68ea39f25020d28&scene=0&key=f5c31ae61525f82eff1e94d54c4abd1c47958e9f7fe5751f2d201958841a354f0423e4ce98cb895058808f330921bde6&ascene=0&uin=MTYzMjY2MTE1&devicetype=iMac+MacBookPro10%2C1+OSX+OSX+10.11.4+build(15E65)&version=11020201&pass_ticket=rsJxex9Bn1eN7iJgW6FUI2KG3V9O6e3TH42j1U%2Fy5SU%3D)
- [x] [Github官网](https://github.com/square/retrofit)
- [x] [Website](http://square.github.io/retrofit/)
- [ ] [Retrofit教程系列](https://futurestud.io/tutorials/retrofit-getting-started-and-android-client)