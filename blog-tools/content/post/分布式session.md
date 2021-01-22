---
title: "分布式session" 
date: 2021-01-06T12:29:39+08:00
draft: false
tags: ["分布式","笔记","JAVA"]
categories:
  - "分布式"
  - "笔记"
  - "JAVA"
---



# 分布式session



## 传统Session



### 传统Session实现

传统单机Session实现方式：

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @GetMapping("/login")
    public String login(@RequestParam String username,
                        @RequestParam String password,
                        HttpSession session){
        // 账号密码正确
        session.setAttribute("login_user",username);
        return "登录成功";
    }

    @GetMapping("/info")
    public String info(HttpSession session){
        return "当前登录的是：" + session.getAttribute("login_user");
    }


}
```



访问`http://127.0.0.1:8080/user/login?username=admin&password=admin`接口登录，可以在浏览器开发工具中的Network中看到请求中的`Response Headers`列中发现下面的key-value值：

```
Set-Cookie: JSESSIONID=EB47B8EC38FBA2E7974DFC0F8C2C9ABA; Path=/; HttpOnly
```

这条响应即是服务器端返回的用于标识Session的Cookie-Id。



登录后再访问`http://127.0.0.1:8080/user/info`，可以在请求的`Request Headers` 中发现:

```
Cookie: JSESSIONID=EB47B8EC38FBA2E7974DFC0F8C2C9ABA
```

即为登录时服务器返回的Cookie-Id，此Cookie和对应的Session在服务器端同样有所记录，便于后续前台请求发送到后台时，后台可以确定Session是否存在。



### Cookie的跨域问题

在使用`http://127.0.0.1:8080/user/login?username=admin&password=admin`接口登录后，使用`http://127.0.0.1:8080/user/info`可以看到对应的登录信息，但是当使用：

```
http:/localhost:8080/user/info
```

访问时却获取不到之前的登录信息，在浏览器开发工具的Application项中的Storage下的Cookies中可以选择对应的Cookie项看到，此时的Domain值是`localhost`，而之前`127.0.0.1`的请求中的Domain则是`127.0.0.1`，也就是说，虽然是在同一个浏览器下，但是Cookie只能被相同Domain（域名）下的请求访问到。如果是不同的Domain发起的请求，各个请求携带的Cookie-Id是不一样的。也就导致了Session的不同。可以在前台Application中的Storage下的Cookies修改Cookie值模拟对应的Session。



## Spring-Session实践

引入Spring-Session依赖和对应的Redis依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```



配置对应的Spring-Session和Redis的配置:

```properties
spring.redis.host=127.0.0.1
spring.redis.port=6379

# 定义Spring-Session使用的存储工具
spring.session.store-type=redis
# Spring Session自定义过期时间
spring.session.timeout=3600
# Spring Session自定义在Redis中的key的前缀,默认值:spring-session
spring.session.redis.namespace=login_user
```



然后启动项目后访问`http://127.0.0.1:8080/user/login?username=admin&password=1`登录后，再访问`http://127.0.0.1:8080/user/info`可以看到正常的登录信息。此时在Redis中可以看到如下的三个值：

```
spring:session:expirations:1610427360000
spring:session:sessions:e4ce5829-d85f-455e-acf9-5b53c7c35a26
spring:session:sessions:expires:e4ce5829-d85f-455e-acf9-5b53c7c35a26
```

代表此登录用户的Session信息已存入Redis中，此时重启应用，再次访问`http://127.0.0.1:8080/user/info`依然可以访问到之前的登录信息，因为此时Session信息不是存在Tomcat容器中，而是存在第三方的Redis中。



## Token+Redis自定义Session组件

在移动端等领域，基于Cookie的Session解决方案并不适用，且`Token+Redis`实现的方案自由度更高。



```java
@Autowired
private StringRedisTemplate stringRedisTemplate;

@GetMapping("/loginWithToken")
public String loginWithToken(@RequestParam String username,
							@RequestParam String password){
    // 账号密码正确
    String key = "token_" + UUID.randomUUID().toString();
    stringRedisTemplate.opsForValue().set(key,username,3600, TimeUnit.SECONDS);
    return key;
}

@GetMapping("/infoWithToken")
    public String infoWithToken(@RequestParam String token){
    return "当前登录的是：" + stringRedisTemplate.opsForValue().get(token);
}
```



## JWT实现

前面的分布式Session实现方式都依赖于Redis中间件，会增加系统复杂性，且如果要使用对应的加密信息还需手动编写对应的加密逻辑。可以使用JWT解决这两个问题



JSON Web Token (JWT)是一个开放标准(RFC 7519)，它定义了一种紧凑的、自包含的方式，用于作为JSON对象在各方之间安全地传输信息。该信息可以被验证和信任，因为它是数字签名的。



JSON Web Token由三部分组成，它们之间用圆点(.)连接。这三部分分别是：

1. Header
2. Payload
3. Signature



因此，一个典型的JWT看起来是这个样子的：

```
xxxxx.yyyyy.zzzzz
```



### Header

header典型的由两部分组成：token的类型（“JWT”）和算法名称（比如：HMAC SHA256或者RSA等等）。

例如：

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

然后，用Base64对这个JSON编码就得到JWT的第一部分



### Payload

JWT的第二部分是payload，它包含声明（要求）。声明是关于实体(通常是用户)和其他数据的声明。声明有三种类型: registered, public 和 private。

- Registered claims : 这里有一组预定义的声明，它们不是强制的，但是推荐。比如：iss (issuer), exp (expiration time), sub (subject), aud (audience)等。
- Public claims : 可以随意定义。
- Private claims : 用于在同意使用它们的各方之间共享信息，并且不是注册的或公开的声明。



例如：

```json
{
  "login_user": "admin",
  "user_id": 1,
  "exp": 1610465947
}
```

对payload进行Base64编码就得到JWT的第二部分

注意，不要在JWT的payload或header中放置敏感信息，除非它们是加密的。



### Signature

为了得到签名部分，你必须有编码过的header、编码过的payload、一个秘钥，签名算法是header中指定的那个，然对它们签名即可。

例如：

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

签名是用于验证消息在传递过程中有没有被更改，并且，对于使用私钥签名的token，它还可以验证JWT的发送方是否为它所称的发送方。



### JWT工作方式

在认证的时候，当用户用他们的凭证成功登录以后，一个JSON Web Token将会被返回。此后，token就是用户凭证了，你必须非常小心以防止出现安全问题。一般而言，你保存令牌的时候不应该超过你所需要它的时间。

无论何时用户想要访问受保护的路由或者资源的时候，用户代理（通常是浏览器）都应该带上JWT，服务器会根据设置好的密钥检查`Header`和`Signature`是否是正确的。



如果JWT的token是在授权头（Authorization header）中发送的，那么跨源资源共享(CORS)将不会成为问题，因为它不使用cookie。



### JWT和Session的差异

相同点是，它们都是存储用户信息；然而，Session是在服务器端的，而JWT是在客户端的。

Session方式存储用户信息的最大问题在于要占用大量服务器内存，增加服务器的开销。

而JWT方式将用户状态分散到了客户端中，可以明显减轻服务端的内存压力，服务器只需要根据传入的token进行计算即可，不再需要保留大量的Session信息。

Session的状态是存储在服务器端，客户端只有session id；而Token的状态是存储在客户端。



### 代码实现

引入JWT依赖：

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.10.3</version>
</dependency>
```



```java
private static String JWT_KEY = "encryt_key";

@GetMapping("/loginWithJwt")
public String loginWithJwt(@RequestParam String username,
							@RequestParam String password){
    // 账号密码正确
    // 指定JWT使用的算法和对应的密钥key
    Algorithm algorithm = Algorithm.HMAC256(JWT_KEY);
    String token = JWT.create()
    // 向JWT中存入数据
    .withClaim("login_user",username)
    .withClaim("user_id",1)
    // 设置token过期时间
    .withExpiresAt(new Date(System.currentTimeMillis() + 3600000))
    .sign(algorithm);
    return token;
}

@GetMapping("/infoWithJwt")
public String infoWithJwt(@RequestParam String token){
    // 指定JWT使用的算法和对应的密钥key
    Algorithm algorithm = Algorithm.HMAC256(JWT_KEY);
    JWTVerifier verifier = JWT.require(algorithm).build();
    try {
    DecodedJWT jwt = verifier.verify(token);
    // 返回该token对应的用户名
    return jwt.getClaim("login_user").asString();
    }catch (TokenExpiredException e){
    // token过期
    }catch (JWTDecodeException e){
    // 解码失败，token错误
    }

    return "error";
}
```



启动项目后，访问`http://127.0.0.1:8080/user/loginWithJwt?username=admin&password=1`之后，返回的token是一个类似于下面的值：

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpbl91c2VyIjoiYWRtaW4iLCJ1c2VyX2lkIjoxLCJleHAiOjE2MTA0NjU1MTh9.SnoRFLwj-ITZBZz41h0rZiUaEAMGI5HKvaByMjWajJA
```

再使用这个token作为参数访问：

```
http://127.0.0.1:8080/user/infoWithJwt?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpbl91c2VyIjoiYWRtaW4iLCJ1c2VyX2lkIjoxLCJleHAiOjE2MTA0NjU1MTh9.SnoRFLwj-ITZBZz41h0rZiUaEAMGI5HKvaByMjWajJA
```

可以得到登陆时在JWT中保存的变量值（用户的名字），同理也可获得对应的过期时间及其他值。



## 类似的OAuth2技术

使用OAuth2的场景：

1. 授权数据给第三方
2. 权限管理



