---
title: retrofit+rxjava封装
date: 2017-09-30 12:06:58
categories: android
tags: 
    android
    java
---
很多时候我们在使用开源的网络框架时都需要根据后台返回的数据进行相应的封装,从而使开源框架的使用更简便。下面我就讲讲我是如何封装retrofit。
# 一.分析后台返回数据格式
 1. 请求成功,返回数据类型一
```json
   {
      "status":200,
      "data":{"aa":"bb"}
   }
```
 2. 请求成功,返回数据类型二
```json
   {
      "status":200
      "data":[{"aa":"bb"},{"cc":"dd"}]
   }
```
 3. 请求失败,返回错误数据
 ```json
   {
       "status":400
       "msg":"请求参数错误"
   }
```

# 二.构建实体
从后台返回的数据我们可以知道请求成功时,只有data的类型是不唯一的,它既有可能是json对象也有可能是json数组.所以我们不妨采用泛型来定义data的数据类型.而请求失败时没有data字段而是多了msg字段,既实体应该包含msg属性.
```java
public class WrapperRspEntity<T> {
    private int status;
    private T data;
    private String msg; //errorMSG;


    public int getStatus() {
        return status;
    }

    public void setStatus(int status) {
        this.status = status;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```
这样我们就把响应实体构建好了,接下来就是如何配合retrofit一起使用。
# 三.引入依赖,开始封装retrofit
## 引入相关的依赖包
```gradle
compile 'com.squareup.retrofit2:retrofit:2.1.0'
compile 'com.google.code.gson:gson:2.6.2'
compile 'com.squareup.retrofit2:converter-gson:2.0.0-beta3'
compile 'com.squareup.okhttp3:logging-interceptor:3.3.1'
```
## 采用单例模式封装retrofit
```java
public class RetrofitManager {
    private static RetrofitManager mRetrofitManager;
    private Retrofit mRetrofit;

    private RetrofitManager(){
        initRetrofit();
    }

    public static synchronized RetrofitManager getInstance(){
        if (mRetrofitManager == null){
            mRetrofitManager = new RetrofitManager();
        }
        return mRetrofitManager;
    }

    
    private void initRetrofit() {
        HttpLoggingInterceptor LoginInterceptor = new HttpLoggingInterceptor();
        LoginInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
        OkHttpClient.Builder builder = new OkHttpClient.Builder();

        
        if (AppConfig.DEBUG){
            builder.addInterceptor(LoginInterceptor); //添加retrofit日志打印
        }

        
        builder.connectTimeout(15, TimeUnit.SECONDS);
        builder.readTimeout(20, TimeUnit.SECONDS);
        builder.writeTimeout(20, TimeUnit.SECONDS);
        builder.retryOnConnectionFailure(true);
        OkHttpClient client = builder.build();

        mRetrofit = new Retrofit.Builder()
                .baseUrl(AppConfig.BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .client(client)
                .build();
    }

    public <T> T createReq(Class<T> reqServer){
        return mRetrofit.create(reqServer);
    }
}
```

## 添加数据检测拦截器
   请求成功后,我们需要检测数据格式是否正确和通过status判断请求是否成功,因此我们可以添加一个检查拦截器去对所有请求的响应结果进行检测。

   ```java
    private void initRetrofit() {
       ...
       OkHttpClient.Builder builder = new OkHttpClient.Builder();

       builder.addInterceptor(new RspCheckInterceptor()); //添加检查拦截器
       
       if (AppConfig.DEBUG){
            builder.addInterceptor(LoginInterceptor);
        }
        ...
    }
```

   ```java
    public class RspCheckInterceptor implements Interceptor{

        @Override
        public Response intercept(Chain chain) throws IOException {
            Response response = chain.proceed(chain.request());
                try {
                    ResponseBody rspBody = response.body();
                    JSONObject jsonObject = new JSONObject(InterceptorUtils.getRspData(rspBody));
                    int status = jsonObject.getInt("status");
                    if (status < 200 || status >= 300){
                        throw new IOException(jsonObject.getString("msg"));
                    }
                } catch (JSONException e) {
                    e.printStackTrace();
                    throw new IOException("parase data error");
                }catch (Exception e){
                    if (e instanceof IOException){
                        throw (IOException)e;
                    }
                }

            return response;
        }
    }
```

## 实现列表数据缓存,断网时从缓存中获取列表数据
有时候我们可能有这样的需求,有网络从后台获取列表数据,无网络从缓存里获取列表数据。那么我们要如何简单的实现这样的需求呢?
retrofit的网络层是采用OKhttp。那么我们可以设置okhttp的cache目录和添加okhttp的网络拦截器来实现这样需求

```java
private void initRetrofit() {
    ...
    builder.addInterceptor(new RspParseInterceptor());
    if (AppConfig.DEBUG){
        builder.addInterceptor(LoginInterceptor);
    }

    builder.addNetworkInterceptor(new RspCacheControllerInterceptor()); //添加缓存控制拦截器
    File cacheFile = new File(AppConfig.HTTP_CACHE_PAth);
    Cache cache = new Cache(cacheFile,AppConfig.CACHE_SIZE);
    builder.cache(cache);

    builder.connectTimeout(15, TimeUnit.SECONDS);
    ...
}
```


```java
public class RspCacheControllerInterceptor implements Interceptor {
    private final int maxAge = 60 * 60 * 24 *7;
    private final int maxStale = 60 * 60 * 24 * 28; // tolerate 4-weeks stale

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        // Add Cache Control only for GET methods
        if (request.method().equals("GET")) {
            Response originalResponse = chain.proceed(chain.request());
            if (NetworkUtils.isNetworkConnected(PWApplication.getInstance())) {
                return originalResponse.newBuilder()
                        .header("Cache-Control", "public, max-age=" + maxAge)
                        .build();
            } else {

                return originalResponse.newBuilder()
                        .header("Cache-Control", "public, only-if-cached, max-stale=" + maxStale)
                        .build();
            }
        }

        Response originalResponse = chain.proceed(request);
        return originalResponse;
    }
}
```
一般情况下我们应该只缓存查询API的数据（查询请求一般采用get）,所以在拦截器中我们需要对请求进行判断是否为get请求,是则进行缓存(缓存的具体控制可查看http请求头Cache-Control相关的资料)。当断网时,我们将Cache-Control设置为only-if-cached,那么该请求只会从缓存中查询是否有该请求记录,有则返回缓存数据,没有则返回错误.到了这一步我们基本把该做的事都做完了，最后就剩下如何去使用了。



## 使用
进行用户登录
请求方式为post,url为http:127.0.0.1:8080/pw/user/${userName}
请求参数为userName和pwd
请求成功的响应数据
```json
{
   "status":200,
   "data":{
      "userName":"test"
      "token":"abcdefg123456789"
      "uid":"1"
   }
}
```
### 1.设置baseUrl
```java
public class AppConfig {
    ...
    public static final String BASE_URL = "http:127.0.0.1:8080/pw";
    ...
}
```
### 2.根据返回的data数据构建user实体
```java
public class User {
    private long uid;
    private String userName;
    private String token;

    public long getUid() {return uid;}

    public void setUid(long uid) {this.uid = uid;}
    
    public String getToken() {
        return token;
    }

    public void setToken(String token) {
        this.token = token;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }
}
```
### 3.构建请求API
```java
public interface LoginApi {
    @FormUrlEncoded
    @POST("user/{userName}")
    Call<WrapperRspEntity<User>> loginReq(@Path("userName") String userName,@Field("pwd") String pwd);
}
```
### 4.进行请求
```java
RetrofitManager.getInstance()
                .createReq(LoginApi.class)
                .loginReq("test1", "123456")
                .enqueue(new Callback<WrapperRspEntity<User>>() {
                    @Override
                    public void onResponse(Call<WrapperRspEntity<User>> call, Response<WrapperRspEntity<User>> response) {
                        AppLog.d("userName="+response.body().getData().getUserName());
                    }

                    @Override
                    public void onFailure(Call<WrapperRspEntity<User>> call, Throwable t) {
                        AppLog.d("errorMsg="+t.getMessage());
                    }
                });
```


# 四.结合rxjava使用
### 1.引入依赖
```gradle
compile 'io.reactivex:rxandroid:1.2.1'
compile 'io.reactivex:rxjava:1.1.9'
compile 'com.squareup.retrofit2:adapter-rxjava:2.0.0'
```
### 2.修改LoginApi
```java
public interface LoginApi {
    @FormUrlEncoded
    @POST("user/{userName}")
    Observable<WrapperRspEntity<User>> loginReq(@Path("userName") String userName,@Field("pwd") String pwd);
}
```
### 3.修改请求方式
```java
RetrofitManager.getInstance()
                .createReq(LoginApi.class)
                .loginReq("test1", "123456")
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<WrapperRspEntity<User>>() {
            @Override
            public void onCompleted() {
            }

            @Override
            public void onError(Throwable e) {
                AppLog.d("errorMsg="+t.getMessage());
            }

            @Override
            public void onNext(WrapperRspEntity<User> userWrapperRspEntity) {
                AppLog.d("userName="+userWrapperRspEntity.getData().getUserName);
            }
        });
```