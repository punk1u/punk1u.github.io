---
title: "Spring Cloud Alibaba实践——认证授权" 
date: 2021-02-06T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——认证授权

在微服务架构中，一个绕不开的问题就是认证授权，比如必须得登录之后才能进行一些操作，而用户登录往往都是在一个专门的用户服务中完成，如果其他的服务也需要获取到用户登录后的状态才能进行相应操作，就需要从用户服务获得用户登录状态的认证授权。



## 有状态vs无状态

在单体应用中，都是采用`Session`的方式在服务端存储`用户状态`，保持`会话`，如果一个服务有多个实例，则将`用户状态`存储在第三方工具中，比如`Redis`等。但是在微服务中，普遍使用的方式是`无状态`存储的方式，即服务端不再记录用户的状态，这样做即缓解了服务端的性能压力，又避免了需要对`Redis`等工具的运维工作。



但是无状态也有它的缺点，比如不能像有状态那样有很强的控制能力，比如有状态时（比如Session），可以很方便地设置修改用户登录状态的过期时间，可以强行下线用户。



## JWT

比较有代表性的无状态认证授权方案是`JWT`，参考之前写的[分布式Session中的JWT的实现部分](https://www.punklu.tech/post/%E5%88%86%E5%B8%83%E5%BC%8Fsession/#jwt%E5%B7%A5%E4%BD%9C%E6%96%B9%E5%BC%8F)



### JWT组成

| 组成                | 作用                        | 内容示例                           |
| ------------------- | --------------------------- | ---------------------------------- |
| Header（头）        | 记录令牌类型、签名的算法等  | {"alg":"HS256,"typ":"JWT"}         |
| Payload（有效载荷） | 携带一些用户信息            | {"userId":"1","username":"punk1u"} |
| Signatrue(签名)     | 防止token被篡改，确保安全性 | 计算出来的签名，一个字符串         |



### JWT相关公式

Token = Base64(Header).Base64(Payload).Base64(Signature)

示例：aaaa.bbbbb.ccccc



Signatur = 使用Header指定的签名算法计算(Base64(header).Base64(payload),密钥)



示例：HS256("aaaa.bbbbb",密钥)





## 实现微服务间的用户认证



### 用户中心引入JWT

首先引入依赖：

```xml
<!-- 添加JWT依赖 -->
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.10.3</version>
</dependency>
```



添加配置项：

```yaml
JWT_KEY:
  punk1uJWTTokenSecret
JWT_EXPIRE_TIME:
  3600000
```

用于指定`JWT`加密解密时的`KEY`以及生成的`token`的过期时间。



编写相关`JWT`工具类：

```java
package tech.punklu.usercenter.util;

import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTCreator;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTDecodeException;
import com.auth0.jwt.exceptions.TokenExpiredException;
import com.auth0.jwt.interfaces.DecodedJWT;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.Date;
import java.util.Map;

@Component
@Slf4j
public class JWTOperater {

    @Value("${JWT_KEY}")
    private String JWT_KEY;

    @Value("${JWT_EXPIRE_TIME}")
    private Long JWT_EXPIRE_TIME;

    /**
     * 生成对应的JWT Token
     * @param map
     * @return
     */
    public String generateToken(Map<String,String> map){
        // 账号密码正确
        // 指定JWT使用的算法和对应的密钥key
        Algorithm algorithm = Algorithm.HMAC256(JWT_KEY);
        JWTCreator.Builder builder = JWT.create();
        for (String key : map.keySet()){
            builder.withClaim(key,map.get(key));
        }
        // 设置token过期时间
        String token =
                builder.withExpiresAt(new Date(System.currentTimeMillis() + JWT_EXPIRE_TIME))
                .sign(algorithm);
        return token;
    }

    /**
     * 从JWT Token中解析相应的数据
     * @param token JWT Token
     * @param key 要解析的数据的key
     * @return
     */
    public String getInfoFromJWT(String token,String key){
        // 指定JWT使用的算法和对应的密钥key
        Algorithm algorithm = Algorithm.HMAC256(JWT_KEY);
        JWTVerifier verifier = JWT.require(algorithm).build();
        try {
            DecodedJWT jwt = verifier.verify(token);
            // 返回该token对应的用户名
            return jwt.getClaim(key).asString();
        }catch (TokenExpiredException e){
            // token过期
            log.warn("token过期!",e);
        }catch (JWTDecodeException e){
            // 解码失败，token错误
            log.warn("token解码失败!",e);
        }

        return null;
    }
}

```



开发登录接口：

```java
package tech.punklu.usercenter.controller.user;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import tech.punklu.usercenter.domain.dto.user.UserWithJwtTokenRespDTO;
import tech.punklu.usercenter.domain.entity.user.User;
import tech.punklu.usercenter.service.user.UserService;
import tech.punklu.usercenter.util.JWTOperater;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/users")
@Slf4j
public class UserController {

    @Autowired
    private UserService userService;


    /**
     * 用户登录
     * @param loginUser
     * @return
     */
    @PostMapping("/login")
    public String login(@RequestBody  User loginUser){
        String userToken = this.userService.login(loginUser);
        return userToken;
    }
}

```



以及对应的`Service`方法：

```java
package tech.punklu.usercenter.service.user;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import tech.punklu.usercenter.dao.user.UserMapper;
import tech.punklu.usercenter.domain.entity.user.User;
import tech.punklu.usercenter.util.JWTOperater;

import java.util.HashMap;
import java.util.Map;

@Service
@Slf4j
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private JWTOperater jwtOperater;

    public String login(User user){
        User dbData = this.userMapper.selectByPrimaryKey(user);
        // 如果登录成功，生成对应的jwt token并返回
        if (dbData != null){
            Map<String,String> parameterMap = new HashMap<>();
            parameterMap.put("user_id",dbData.getId().toString());
            String token = jwtOperater.generateToken(parameterMap);
            return token;
        }else {
            log.warn("登录失败，不存在对应的用户!");
        }
        return null;
    }
}

```



启动用户中心，使用`Postman`访问`127.0.0.1:8081/users/login`，携带的`JSON`数据为`{"id":"1"}`，可以看到返回回来了`JWT`的`token`。将`token`解密可以看到对应的`Header`和`Payload`中的值，如下所示：

**Header:**

```json
{
    "typ": "JWT",
    "alg": "HS256"
}
```

**Payload:**

```json
{
    "user_id": "1",
    "exp": 1613057742
}
```



`Signature`是加密的，是拿不到对应的数据的。



### 实现登录状态检查

之前用户中心的`/users/login`接口，没有做相关验证就可调用查询。实际应用场景下应该是只有在登录了的情况下才能查询用户信息。



这里使用`AOP`的方式实现。



首先引入依赖：

```xml
<!-- 添加Spring AOP依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```



然后创建检查用户是否登录的注解：

```java
package tech.punklu.usercenter.auth;

/**
 * 检查用户是否已登录的注解
 */
public @interface CheckLogin {
}

```



定义用户未登录时要抛出并处理的异常：

```java
package tech.punklu.usercenter.security;

public class SecurityException extends RuntimeException{
}

```



以及`Spring MVC`的全局异常处理类：

```java
package tech.punklu.usercenter.advice;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionErrorHandler {

    /**
     * 统一处理用户未登录的异常
     * @param e
     * @return
     */
    @ExceptionHandler(SecurityException.class)
    public ResponseEntity<ErrorBody> error(SecurityException e){
        log.warn("发生SecurityException异常",e);
        ResponseEntity<ErrorBody> response = new ResponseEntity<ErrorBody>(
                ErrorBody.builder()
                .body("Token非法，用户不允许访问!~")
                .status(HttpStatus.UNAUTHORIZED.value())
                .build(),
                HttpStatus.UNAUTHORIZED
        );
        return response;
    }
}

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
class ErrorBody{
    private String body;
    private int status;
}

```



编写`Aspect`切面处理类：

```java
package tech.punklu.usercenter.auth;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import tech.punklu.usercenter.util.JWTOperater;

import javax.servlet.http.HttpServletRequest;

@Aspect
@Component
public class CheckLoginAspect {

    @Autowired
    private JWTOperater jwtOperater;

    /**
     * 检查用户是否登录
     * @param point
     * @return
     */
    @Around("@annotation(tech.punklu.usercenter.auth.CheckLogin)")
    public Object checjLogin(ProceedingJoinPoint point) {
        try {
            // 从Header中获取token
            RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
            ServletRequestAttributes attributes = (ServletRequestAttributes) requestAttributes;
            HttpServletRequest request = attributes.getRequest();
            String token = request.getHeader("X-Token");
            // 校验token是否合法，如果不合法直接抛异常，如果合法放行
            String id = jwtOperater.getInfoFromJWT(token, "user_id");
            if (StringUtils.isEmpty(id)){
                throw new SecurityException("Token不合法!");
            }
            // 如果校验成功，那么就将用户的id设置到request的attribute里面
            request.setAttribute("id",id);
            return  point.proceed();
        }catch (Throwable e){
            throw new SecurityException("Token不合法!");
        }
    }
}

```





最后，将检查的注解添加到接口上：

```java
package tech.punklu.usercenter.controller.user;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import tech.punklu.usercenter.auth.CheckLogin;
import tech.punklu.usercenter.domain.entity.user.User;
import tech.punklu.usercenter.service.user.UserService;

@RestController
@RequestMapping("/users")
@Slf4j
public class UserController {

    @Autowired
    private UserService userService;




    /**
     * 根据用户id查询对应的用户信息
     * @param id
     * @return
     */
    @GetMapping("/{id}")
    // 添加检查用户是否已登录的注解
    @CheckLogin
    public User findById(@PathVariable Integer id){
        return this.userService.findById(id);
    }
}

```



同理，给`内容中心`的`/shares/{id}`查询投稿信息接口也添加上相应检查用户是否已登录的的功能。



### Feign实现Token传递

在给`用户中心`、`内容中心`都添加上了相应的检查后，启动`内容中心`、`用户中心`，访问内容中心的`127.0.0.1:8082/shares/1?origin=browser`接口，携带上`X-Token`，可以发现虽然正常返回了投稿相关的信息，但是其中的`wxNickName`的值并没有正常返回，而是触发了`Feign`的默认降级处理规则，返回了一个默认值。因为这需要调用`用户中心`的`/users/{id}`接口才能查询到，但是因为直接调用的是`内容中心`的接口，虽然传递了`X-Token`，但是这个值并没有被传递到`Feign`客户端中，所以调用`用户中心`报错了。

实现方式包括：

1. `@RequestHeader`注解

   `@RequestHeader`是`Spring MVC`的一个注解，`Feign`支持`Spring MVC`注解，所以可以使用`@RequestHeader`。

2. `RequestInterceptor`拦截器

   

其中`@RequestHeader`需要对每个接口进行修改，当接口数量较多时改动很大。比如在这个例子中，需要先使用`@RequestHeader`给`/shares/{id}`接口上接收前台传过来的`X-Token`，还要在后边的`UserCenterFeignClient`这个`Feign`代理类上添加上对应的`@RequestHeader`注解表示将`X-Token`通过`Feign`传递到`用户中心`。



所以这里使用`RequestInterceptor`拦截器来实现`X-Token`的传递。



新建`Feign`拦截器扩展类：

```java
package tech.punklu.contentcenter.feignclient.interceptor;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.util.StringUtils;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;

/**
 * Feign Token传递的类
 */
public class TokenRelayRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate requestTemplate) {
        // 获取到token
        // 从Header中获取token
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        ServletRequestAttributes attributes = (ServletRequestAttributes) requestAttributes;
        HttpServletRequest request = attributes.getRequest();
        String token = request.getHeader("X-Token");
        // 判断token是否为空
        if (!StringUtils.isEmpty(token)){
            // 将token传递
            requestTemplate.header("X-Token",token);
        }
    }
}

```



然后在配置文件中添加上`Feign`全局拦截器配置项：

```yaml
feign:
  client:
    config:
      # 想要调用的微服务的名称,如果想配置全局的，只需要设置为default即可
      default:
        loggerLevel: full
        # 配置全局的Feign拦截器配置类，用于在微服务间传递token
        requestInterceptors:
          -tech.punklu.contentcenter.feignclient.interceptor.TokenRelayRequestInterceptor
```



重启`内容中心`，再次访问内容中心的`127.0.0.1:8082/shares/1?origin=browser`接口，携带上`X-Token`，可以发现已经正常返回了投稿相关的信息，其中的`wxNickName`的值也已经有正常返回，没有触发降级规则返回默认值，说明已经实现了`X-Token`在微服务间的正常传递。



### RestTemplate传递Token

如果项目不是使用的`Feign`而是使用的`RestTemplate`的方式的话，可以使用如下的两种方式实现微服务间的`Token`传递：

1. `exchange()`


   示例代码：

   ```java
   package tech.punklu.contentcenter;
   
   import lombok.extern.slf4j.Slf4j;
   import org.apache.commons.lang3.StringUtils;
   import org.springframework.http.HttpEntity;
   import org.springframework.http.HttpHeaders;
   import org.springframework.http.HttpMethod;
   import org.springframework.http.ResponseEntity;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.PathVariable;
   import org.springframework.web.bind.annotation.RestController;
   import org.springframework.web.client.RestTemplate;
   import tech.punklu.contentcenter.domain.dto.user.UserDTO;
   import tech.punklu.contentcenter.domain.entity.content.Share;
   
   import javax.servlet.http.HttpServletRequest;
   
   @Slf4j
   @RestController
   public class TestController {
   
       @Autowired
       private RestTemplate restTemplate;
   
       @GetMapping("/tokenRelay/{userId}")
       public ResponseEntity<UserDTO> tokenRelay(@PathVariable Integer userId, HttpServletRequest request) {
           String token = request.getHeader("X-Token");
           HttpHeaders headers = new HttpHeaders();
           headers.add("X-Token", token);
   
           return this.restTemplate
                   .exchange(
                           "http://user-center/users/{userId}",
                           HttpMethod.GET,
                           new HttpEntity<>(headers),
                           UserDTO.class,
                           userId
                   );
       }
   }
   
   ```

   

2. `ClientHttpRequestInterceptor`客户端拦截器

   `exchange()`也需要面临修改多个接口代码的问题，所以，最好的方法依然是使用拦截器实现。


   拦截器示例代码：

   ```java
   package tech.punklu.contentcenter;
   
   import org.springframework.http.HttpHeaders;
   import org.springframework.http.HttpRequest;
   import org.springframework.http.client.ClientHttpRequestExecution;
   import org.springframework.http.client.ClientHttpRequestInterceptor;
   import org.springframework.http.client.ClientHttpResponse;
   import org.springframework.web.context.request.RequestAttributes;
   import org.springframework.web.context.request.RequestContextHolder;
   import org.springframework.web.context.request.ServletRequestAttributes;
   
   import javax.servlet.http.HttpServletRequest;
   import java.io.IOException;
   
   public class RestTemplateTokenRelayInterceptor implements ClientHttpRequestInterceptor {
   
       @Override
       public ClientHttpResponse intercept(HttpRequest httpRequest, byte[] bytes, ClientHttpRequestExecution clientHttpRequestExecution) throws IOException {
           // 获取到token
           // 从Header中获取token
           RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
           ServletRequestAttributes attributes = (ServletRequestAttributes) requestAttributes;
           HttpServletRequest request = attributes.getRequest();
           String token = request.getHeader("X-Token");
   
           // 将获取到的Token添加到RestTemplate的Header里
           HttpHeaders headers = httpRequest.getHeaders();
           headers.add("X-Token",token);
           return clientHttpRequestExecution.execute(httpRequest,bytes);
       }
   }
   
   ```


   然后修改项目主类里声明`RestTemplate`实例的代码：

   ```java
   /**
    * 在Spring容器中，创建一个对象，类型是RestTemplate，
    * 名称/id是方法名
    * @return
   */
   @Bean
   @LoadBalanced
   @SentinelRestTemplate
   public RestTemplate restTemplate(){
       RestTemplate template = new RestTemplate();
       template.setInterceptors(
           Collections.singletonList(new RestTemplateTokenRelayInterceptor())
       );
       return template;
   }
   ```



至此，已经实现了微服务间的`用户登录认证`，主要通过无状态的`JWT`实现，微服务间的`授权`也可通过这个方式实现，比如对于投稿审核功能，可以在`JWT`中添加用户的角色信息，在`投稿审核接口`上通过`Spring AOP`功能和`用户登录认证`一样实现相关`角色`的审核功能。







