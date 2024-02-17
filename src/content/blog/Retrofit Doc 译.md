---
pubDatetime: 2016-04-05
title: Retrofit Doc 译
featured: false
draft: false
tags:
  - Android
description: 为加深对Retrofit网络请求框架的理解，遂将官网Doc通读一遍并翻译成中文。
---

为加深对Retrofit网络请求框架的理解，遂将官网Doc通读一遍并翻译成中文。

<!-- more -->

## 引言

Retrofit将你的HTTP API换成Java接口。

```Java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

**Retrofit** 类生成一个 **GitHubService** 接口的实例。

```Java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

**GitHubService** 中的每一个 **Call** 都可以用来创建同步或者异步的HTTP请求。

```Java
Call<List<Repo>> repos = service.listRepos("octocat");
```

使用注解(annotations)的方式来来描述一个HTTP请求：

- 支持URL参数替换和查询参数
- 对象转换为请求body (例如 JSON，protocol buffers)
- 多请求体和文件上传
  _Note：_ 后续新版2.0的API扩展仍在进行中。

## API 说明

接口中方法以及其参数的注解用于表明这个请求会被处理的方式。

#### 请求方法

每个方法都必须有一个声明了请求类型和相对URL地址的HTTP注解，内置支持五种类型注解：**GET，POST，PUT，DELETE，HEAD**，相对URL地址在注解中进行指定。

```Java
@GET("users/list")
```

也可以直接在URL中写请求参数。

```Java
@GET("users/list?sort=desc")
```

#### URL操作

通过在方法中使用替换块和参数，可以实现请求URL地址的动态更新。一个替换块(replacement block)是一个由花括号“{...}”包裹的字符串，与之通信的参数必须使用 **@Path** 注解相同的字符串。

```Java
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId);
```

同时也可以添加查询参数

```Java
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @Query("sort") String sort);
```

更复杂的参数组合可以使用 **Map**

```Java
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);
```

#### 请求体（Request Body)

使用 **@Body** 注解来指定一个对象，用作HTTP请求体。

```Java
@POST("users/new")
Call<User> createUser(@Body User user);
```

该对象也会被 **Retrofit** 实例中指定的转换器(Converter)进行转换。如果没有指定转换器，则只能使用 **RequestBody** 。

#### 表单数据及操作

可以声明方法来发送表单以及多请求体数据。
当方法上出现 **@FormUrlEncoded** 时会发送表单形式的数据，键值对通过 **Field** 注解来声明。

```Java
@FormUrlEncoded
@POST("user/edit")
Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);
```

当方法上出现 **@Multipart** 时会以多请求体的形式发送数据。每一部分使用 **@Part** 注解。

```Java
@Multipart
@PUT("user/photo")
Call<User> updateUser(@Part("photo") RequestBody photo, @Part("description") RequestBody description);
```

此部分需要使用 **Retrofit**

#### 请求头部操作

可以使用 **@Header** 注解为每一个方法设置静态请求头部。

```Java
@Headers("Cache-Control: max-age=640000")
@GET("widget/list")
Call<List<Widget>> widgetList();
```

```Java
@Headers({
    "Accept: application/vnd.github.v3.full+json",
    "User-Agent: Retrofit-Sample-App"
})
@GET("users/{username}")
Call<User> getUser(@Path("username") String username);
```

注意：请求头部之间不会被重写，所有相同名字的Header都会被包含到请求中。
请求头部可以通过 **Header** 注解来进行动态更新，如果参数的对象值为null，该头部将会被忽略；否则会调用 **toString()** 方法来取得值。

```Java
@GET("user")
Call<User> getUser(@Header("Authorization") String authorization)
```

**每个请求都使用的头部Header可以使用[OkHttp interceptor](https://github.com/square/okhttp/wiki/Interceptors)来指定**

#### 同步 VS 异步

**Call** 的实例既可以同步执行，也可以异步执行；每个实例只能使用一次，可以调用 **clone()** 方法来创建一个新的实例使用。
在Android上，回调(callbacks)会在主线程中执行；在JVM中，回调会在执行HTTP请求的线程中执行。

## Retrofit 配置

**Retrofit** 是一个可以把你的API接口转换成可被调用的对象的类。默认情况下，Retrofit会为你的平台提供合理的默认配置，同时它也支持自定义配置。

#### 转换器(Converters)

默认情况下，Retrofit只能将HTTP请求体反序列化成OkHttp的 **RetrofitBody** 类型，而且 **@Body** 也只接受 **RetrofitBody** 类型。
添加转化器可以支持其他的类型。为了方便使用，以下六个流行的序列化库已被支持(retrofit 2)。

- **[Gson](https://github.com/google/gson)** : com.squareup.retrofit2:converter-gson
- **[Jackson](http://wiki.fasterxml.com/JacksonHome)** : com.squareup.retrofit2:converter-jackson
- **[Moshi](https://github.com/square/moshi/)** : com.squareup.retrofit2:converter-moshi
- **[ProtocolBuf](https://developers.google.com/protocol-buffers/)** : com.squareup.retrofit2:converter-protobuf
- **[Wire](https://github.com/square/wire)** : com.squareup.retrofit2:converter-wire
- **[Simple XML](http://simple.sourceforge.net/)** : com.squareup.retrofit2:converter-simplexml

一个使用 **GsonConverterFactory** 类来生成 **GitHubService** 实例的例子。

```Java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

#### 自定义转化器

如果与API的通信使用了Retrofit不支持的格式的内容，或者希望使用一个不同序列化库，那么就需要创建一个自定义的Converter。集成 **Converter.Factory** 类创建一个新类，并且在build Retrofit的时候传入该类的实例。

## 参考资料

- 译自[Retrofit官网](http://square.github.io/retrofit/) (翻译官方Doc时的Retrofit版本为2.0.1，一切更新以官网为准)
