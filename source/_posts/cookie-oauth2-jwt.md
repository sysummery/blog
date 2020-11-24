---
title: 从Cookie到OAuth2再到JWT
date: 2019-08-15 17:00:39
tags:
    - cookie
    - oauth2
    - jwt
photos:
    - ["http://sysummery.top/jwt.jpeg"]
---
cookie加session认证是比较传统的方法，但是随着分布式系统的出现，这种认证方式局限性就渐渐显露出来了。OAuth2认证主要是用于使用公共平台的身份去登陆第三方网站。JWT是一种简单的认证方式，适合分布式系统。今天想总结一下工作几年来使用的三种认证方式
<!--more-->
## Cookie认证
Cookie认证是每次客户端向服务器请求的时候都会在http报文的header中传送一个字段叫Cookie，他的格式是形如这样的
![](http://sysummery.top/cookie.jpg)
以分号分隔，比如Cookie:a=b;c=d, 那么到达服务器以后以php为例会有一个全局能访问的数组$\_COOKIE保存这两个键值对。
```php
$var_a = $_COOKIE['a'] //值为b
$var_c = $_COOKIE['c'] //值为d 
```

Cookie并不是专门用来认证的，实时上他是用来和服务器交换数据的。因为http协议是无状态的，因此每次服务器返回给客户端的时候在返回的header里面会告诉浏览器保存哪些信息，然后再次请求这个域名的时候要带上这些信息，通过这种形式来模拟“有状态”。

![](http://sysummery.top/setcookie.jpg)

所以Cookie认证就是在发送的Cookie里面放一个用于表示认证的字段，这个字段的值就是一个凭证，这个凭证就像一把钥匙，能够与服务器上的“数据”相联系，这个数据叫做session。当服务器获得了浏览器传过来的“钥匙”的时候就会通过这把钥匙去找相应的session数据，session数据在哪？有可能在本机的文件系统，也有可能是在redis集群里面。使用redis的好处除了读取和写入的速度要快于文件之外还有一个就是对分布式系统支持良好。假如你的一个服务有a,b,c三台服务器，第一次请求负载均衡到了a机器上，理应session文件应该存储到a服务器的一个文件夹中，第二次负载到了b机器上，这时候b机器没有相应的文件，之前写到a机器上的数据就不见了。服务器获取到session数据后会加载到$\_SESSION超全局数组中。

最后说一下曾今困扰我的一个问题，就是session过期的问题。比如我的请求在10:30，服务端设立的session过期时间是30分钟，那么在11:00session应该就是过期的。但是如果我10:31再次请求的时候，服务端会认为这个session的过期时间是11:01，session的过期时间是每次请求都会更新的。

## OAuth2 认证
OAuth2的应用场景是第三方应用授权登录：在APP或者网页接入一些第三方应用时，时常会需要用户登录另一个合作平台，比如QQ，微博，微信的授权登录,第三方应用通过oauth2方式获取用户信息。

他的流程图大致如下
![](http://sysummery.top/oauth2.png)

1. 用户访问一个网站A，并在网站的登陆页面选择使用"微信"登陆。
2. 客户端扫描微信登陆的二维码，此时A网站向微信服务器发起授权请求，这个请求里面会告诉微信服务器哪个微信用户想使用微信信息登陆A网站。
3. 微信服务器收到A网站的请求后会推送消息给用户让用户选择是否同意A网站使用用户的微信名或者微信头像等信息。网站A与微信服务器的tcp连接是一直连接着的，在http应用层面是通过不断的轮训来”保持连接“的。
4. 用户同意
5. 微信服务器会把网站A对微信服务器的请求转到A网站提前设定好的重定向地址中，同时还会附加一个code
6. A网站通过code和A网站之前已经向微信服务器申请的app_id和app_secret向微信服务器申请调用微信用户信息接口的access_token和refresh_token。
7. A网站使用access_token向微信的接口获取用户的用户名和头像
8. 做一些用户信息处理，比如把该用户的信息存在自己的服务器
9. 返回用户的头像和用户名给浏览器

所以整个过程简单再说一下：先是用户选择微信登陆，然后网站A向微信服务器发起请求，微信服务器收到用户的确认后把请求重定向到网站A的服务器，网站A获取到用户的信息后返回给客户端，至此整个过程结束。

## JWT认证
传统的cookie与session认证在分布式系统里显得很无力很苍白。我们当然可以使用一个redis集群来整一个分布式的session。不过，还有一个方法就是使用JWT(json web token)

他的原理其实很简单，每次客户端登陆成功后，服务器都会为这个用户产生一个认证的字符串，作为之后权限认证的token并放在返回的header中。之后客户端在请求的时候会在请求的header里面附上这个jwt token。服务器收到这个token后会进行一系列的验证，然后判断token的有效性。

那么这个token长得啥样字呢？如下
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

一个jwt的token由3部分组成，分别称作header、payload、signature。header和payload原本都是json格式的数据，他们分别经过base64编码变成两段字符串。最后将这两段字符串使用\.连接起来使用一个固定的密匙和加密算法生成第三段signature。

### header
jwt的头部承载两部分信息：

1. 声明类型，这里是jwt
2. 声明加密的算法 通常直接使用 HMAC SHA256

```js
{
  'typ': 'JWT',
  'alg': 'HS256'
}
```
然后将头部进行base64加密（该加密是可以对称解密的),构成了第一部分。
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

### payload
载荷就是存放有效信息的地方。这个名字像是特指飞机上承载的货品，这些有效信息包含三个部分。

1. 标准中注册的声明
2. 公共的声明
3. 私有的声明

标准中注册的声明 (建议但不强制使用) ：

1. iss: jwt签发者
2. sub: jwt所面向的用户
3. aud: 接收jwt的一方
4. exp: jwt的过期时间，这个过期时间必须要大于签发时间
5. nbf: 定义在什么时间之前，该jwt都是不可用的.
6. iat: jwt的签发时间
7. jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。

公共的声明 ：
公共的声明可以添加任何的信息，一般添加用户的相关信息或其他业务需要的必要信息.但不建议添加敏感信息，因为该部分在客户端可解密.

私有的声明 ：
私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64是对称解密的，意味着该部分信息可以归类为明文信息。

定义一个payload:
```js
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```
然后将其进行base64加密，得到Jwt的第二部分。
```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
```
### signature
把上两步生成的字符串连接起来使用一个固定的secret和加密算法生成signature。然后把这三部分连接起来就是整个token。

### 服务器验证token

1. 首先先使用secret和加密算法重新对token的第一部分和第二部分加密得到signature，检查得到的signature是否与token的第三部分相同，如果不相同说明token可能是别人伪造的。
2. 解析token的payload，检查token有没有过期等信息
3. 应用层也要对token进行验证，比如payload中有没有user_id字段，如果有就可以通过user_id找当前登陆的用户，如果没有告诉客户端重新登陆。

如果是分布式服务，那么只要每台机器的加密算法和加密secret都相同那么无论请求负载均衡到哪台机器上都是一样的。

