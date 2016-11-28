---
title: Retrofit Multipart多文件上传
date: 2016-11-28 09:03:43
thumbnail: http://7xs23g.com1.z0.glb.clouddn.com/retrofit.jpg
tags: [Retrofit]
---


# 背景

新项目的网络库已经由Volley切到了Retrofit，配合``Rxjava + Dagger2 + CleanArchitecture`` ，有效的将项目解耦和，各个层次职责更清晰。依赖注入和注解、动态代理极大的简化了网络请求，开发更高效。

项目中经常会有上传文件的需求，特别是上传图片。上传图片通常有两种方式：

 - bitmap通过Base64转为String，这种方式对于客户端最友好，但是对于服务端要复杂些
 - multipart/form-data 方式

<!--more-->

# multipart/form-data 是什么

http协议将请求分为3个部分：状态行，请求头，请求体。
而RESTFul风格请求更multipart又有些不同，具体的：

 1. ``multipart/form-data``的基础方法是post，也就是说是由post方法来组合实现的。
 2. ``multipart/form-data``与post方法的不同之处：请求头，请求体。
 3. ``multipart/form-data``的请求头必须包含一个特殊的头信息：Content-Type，且其值也必须规定为multipart/form-data，同时还需要规定一个内容分割符用于分割请求体中的多个post的内容，如文件内容和文本内容自然需要分割开来，不然接收方就无法正常解析和还原这个文件。

```
Content-Type: multipart/form-data; boundary=${bound}
```

其中${bound}是定义的分隔符，用于分割各项内容(文件,key-value对)。post格式如下：

```java
--${bound}
Content-Disposition: form-data; name="Filename"
 
HTTP.pdf
--${bound}
Content-Disposition: form-data; name="file000"; filename="HTTP协议详解.pdf"
Content-Type: application/octet-stream
 
%PDF-1.5
file content
%%EOF
 
--${bound}
Content-Disposition: form-data; name="Upload"
 
Submit Query
--${bound}--
```

${bound}是Content-Type里boundary的值



# Volley multipart

在之前的项目中使用Volley，由于Volley的高度可扩展性实现起来比较方便，封装一个[MultipartRequest](https://gist.github.com/w4lle/aecfecc5285c6d8e85eeff80685cadbb)即可。主要的这行代码更改Content-Type的值

```java
    @Override
    public String getBodyContentType() {
        return "multipart/form-data;charset=utf-8;boundary=" + boundary ;
    }
```

# Retrofit multipart

由于Retrofit是一个网络库的封装，具体的网络请求默认是使用OkHttp，Retrofit对于multipart的支持最终也会转换成OkHttp的实现。

在Retrofit中实现一个multipart上传：

```java
    @Multipart
    @POST("upload")
    Call<ResponseBody> uploadFile(
        @Part("description") RequestBody description,
        @Part MultipartBody.Part file);
```

其中：

- ``@retrofit2.http.Multipart``注解: 标记一个请求是multipart/form-data类型,需要和@retrofit2.http.POST一同使用，参数可以是``MultipartBody.Part``或``RequestBody``。
- ``@retrofit2.http.Part``注解: 代表Multipart里的一项数据,即用${bound}分隔的内容块。

可以很方便的上传一个文件和一个参数。但是这样就有一个问题，如果我有一个实体类想要一起上传怎么办，总不能再uploadFile方法里定义很多``@Part("description") RequestBody description``这种参数吧，如果我有多张图片要一起上传呢？

# 文件和参数一起提交

可以使用@PartMap()注解。

```java
    @Multipart
    @POST("upload")
    Call<ResponseBody> uploadFileWithPartMap(
            @PartMap() Map<String, RequestBody> partMap,
            @Part MultipartBody.Part file);
}
```

@PartMap注解代表参数的一个集合。这样使用起来就很方便，对于多个参数：

```java
Map<String, String> partMap = new HashMap<>();
    MultiPartUtil.putRequestBodyMap(partMap, "service_code", certiValue[checkedItem]);
    MultiPartUtil.putRequestBodyMap(partMap, "good_at", goodAt);
    MultiPartUtil.putRequestBodyMap(partMap, "introduction", des);
    MultiPartUtil.putRequestBodyMap(partMap, "integrity", "certificate");
    
    public static final String MULTIPART_FORM_DATA = "multipart/form-data";
    
    public static void putRequestBodyMap(Map map, String key, String value) {
        putRequestBodyMap(map, key, createPartFromString(value));
    }
    
    @NonNull
    public static RequestBody createPartFromString(String descriptionString) {
        if (descriptionString == null) {
            descriptionString = "";
        }
        return RequestBody.create(
                MediaType.parse(MULTIPART_FORM_DATA), descriptionString);
    }

    public static void putRequestBodyMap(Map map, String key, RequestBody body) {
        if (!TextUtils.isEmpty(key) && body != null) {
            map.put(key, body);
        }
    }
```

构造了``Content-Type``为``multipart/form-data``的RequestBody（正常情况下应该为 ``application/json; charset=utf-8``）,同样的构造File part

```java
    public static MultipartBody.Part prepareFilePart(String partName, String fileUri) {
    //压缩
        File file = ViewUtils.compressImageToFile(fileUri, new File(FileUtils.getImageCache(BaseApplication.getContext()), partName + ".png"));
        if (file != null) {
            // 为file建立RequestBody实例
            RequestBody requestFile =
            RequestBody.create(MediaType.parse(MULTIPART_FORM_DATA), file);
            // MultipartBody.Part借助文件名完成最终的上传
            return MultipartBody.Part.createFormData(partName, file.getName(), requestFile);
        }
        return null;
    }
```

这样就基本符合需求，那么如果有多张图片呢？总不能一个参数一个参数写吧。
其实也很简单，由于``@Part MultipartBody.Part file``类型一致，那么我们就可以使用可变参数列表。如下：

```java
    @Multipart
    @POST("upload")
    Observable<JsonObject> authCertification(@PartMap Map<String, RequestBody> partMap, @Part MultipartBody.Part... files);
}
```

```java
api.authCertification(partMap, avatarPart,
                idFrontPart, idBackPart, authOnePart, authTwoPart, authThreePart)
```
                
可以传任意多个文件。

# 参考

 - [Retrofit 2 — Passing Multiple Parts Along a File with @PartMap](https://futurestud.io/tutorials/retrofit-2-passing-multiple-parts-along-a-file-with-partmap)
 - [Retrofit2 multpart多文件上传详解](http://www.chenkaihua.com/2016/04/02/retrofit2-upload-multipart-files/)
 - [HTTP协议之multipart/form-data请求分析](https://my.oschina.net/cnlw/blog/168466)