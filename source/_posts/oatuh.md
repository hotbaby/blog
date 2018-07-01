---
title: OAuth
date: 2017-07-19 22:30:45
tags: [OAuth]
---

OAuth客户端使用一个访问令牌(access token)来访问受保护的资源，其中访问令牌包含了特殊作用域(specific scope), 生命周期，和其他的访问属性。访问令牌由授权服务器在资源拥有者授权之后颁发(issue)。客户端通过访问令牌资源服务器上受保护资源。

比如，一个终端用户(resource-owner)可以授权打印服务器(client)访问他的存储在图片分享服务(resource server)上的受保护的图片，但不用与打印服务器分享他的用户名和密码。他可以授权图片分享服务(authorization server)颁发一个特殊的证书(access token).

# Introduction

## Roles

| Name                 | Description                                                  |
| -------------------- | ------------------------------------------------------------ |
| resource owner       | 一个能授权访问受保护资源的实体                               |
| resource server      | 托管受保护资源的服务器，能处理通过访问令牌访问受保护资源的请求 |
| client               | 代表资源拥有者发起访问受保护资源的请求的应用程序             |
| authorization server | 在资源拥有者认证、授权之后，能够颁发访问令牌给客户端的服务器 |

## Protocol Flow

[![Abstract Protocol Flow](https://hotbaby.org/images/1500477626596.png)](https://hotbaby.org/images/1500477626596.png)Abstract Protocol Flow

## Authorization Grant

授权grant是一个证书表示资源拥有者已经授权，客户端可以用授权grant获取访问令牌。该规范定义4种授权类型：

- authorization code
- implicit
- resource owner password credentials
- client credentials

## Access Token

访问令牌是一种证书用来访问受保护的资源。一个访问令牌就是认证服务器颁发给客户端的字符串。令牌包含了特殊作用域，过期时间，授权的资源拥有者等信息。

## Refresh Token

刷新令牌也是一个证书用来获取访问令牌。刷新令牌是认证服务器发放给客户端的，在当访问令牌不可用或过期时，用来获取新的访问令牌。

刷新令牌是资源资源拥有者授权给客户端认证码。与访问令牌不同，刷新令牌只能与认证服务器通信，不能从资源服务器获取资源。

[![Refreshing an Expired Access Token](http://processon.com/chart_image/59660f76e4b09c8a2926274f.png)](http://processon.com/chart_image/59660f76e4b09c8a2926274f.png)Refreshing an Expired Access Token

# Client Registration

在使用OAuth协议之前，客户端需要在认证服务器注册应用。

客户端注册是让认证服务器能够识别、信任客户端，其中包括客户端类型，重定向URI等。

## Client Types

OAuth定义了两种客户端类型

- confidential - 客户端能够管理证书的机密性
- public - 客户端不能管理证书的机密性

## Client Identifier

认证服务器给已注册的客户端颁发一个客户端标识。对于认证服务器，客户端标识是唯一的。

## Client Authentication

如果客户端类型是confidential, 客户端和认证服务器建立一个客户端认证方法。Confidential客户端会向认证服务器声明它支持证书，比如密码或公私钥对。

# Obtaining Authorization

在请求访问令牌之前，客户端先要获取资源拥有者的授权。OAuth定义了4种授权类型: authorization code, implicit, resource owner password credential, 和client credentials.

## Authorization Code Grant

授权码授权类型用来获取访问令牌和刷新令牌。

认证码获取流程:

[![Authorization Code Flow](https://hotbaby.org/images/1500478492502.png)](https://hotbaby.org/images/1500478492502.png)Authorization Code Flow

### Authorization Request

`Content-Type: application/x-www-form-urlencoded`

| Parameter     | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| response_type | **REQUIRED**该值必须设置为”code”                             |
| client_id     | **REQUIRED**注册的应用的id                                   |
| redirect_uri  | **OPTIONAL**重定向URI，成功获取授权码后重定向到oauth客户端的URI |
| scope         | **OPTIONAL**作用域                                           |
| state         | **RECOMMENDED**客户端用来管理请求、回调状态的值。认证服务器的重定向到UA时，会包含这个值。**这个参数应该用来防止扩展请求伪造** |

```http
GET /o/authorize/?response_type=code&client_id=wt6Pvm2s3vbb8RPE7nlPlugwaMnj58UhFpk8bCPp&redirect_uri=http%3A%2F%2Flocalhost%3A8000%2Foauth%2Fauthorized%2F HTTP/1.0
Host: localhost:8000
Connection: close
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/57.0.2987.98 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch, br
Accept-Language: en-US,en;q=0.8
```

认证服务器会验证这个请求，保证请求中的所有参数都是有效的。如果请求是有效的，认证服务器需要资源拥有者授权。如果资源拥有者授权成功，授权服务器根据`redirect_uri`返回给UA一个重定向响应。

### Authorization Response

`Content-Type: application/x-www-form-urlencoded`

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| code      | **REQUIRED**认证服务器生成的授权码，为了防止泄露的危险，授权码必须在很短的时间过期。授权码生存期推荐为10min.客户端只能使用一次授权码。授权码与客户端id,重定向URI绑定 |
| state     | **REQUIRED**如果客户端请求中包含state,授权服务器必须要返回该值 |

```http
HTTP/1.0 302 Found
Date: Tue, 18 Jul 2017 04:12:00 GMT
Server: WSGIServer/0.1 Python/2.7.9
X-Frame-Options: SAMEORIGIN
Access-Control-Allow-Origin: *
Content-Type: text/html; charset=utf-8
Location: http://localhost:8000/oauth/authorized/?code=x1yxsYhAJ23pXNMXej4tB13dvMFTov
Vary: Cookie
```

#### Error Response

`Conent-Type: application/x-www-form-urlencoded`

| Parameter         | Description  |
| ----------------- | ------------ |
| error             | **REQUIRED** |
| error_description | **OPTIONAL** |
| error_uri         | **OPTIONAL** |
| state             | **REQUIRED** |

```
HTTP/1.1 302 Found
Location: https://client.example.com/cb?error=access_denied&state=xyz
```

### Access Token Request

`Conent-Type: application/x-www-form-urlencoded`

| Parameter    | Description                                  |
| ------------ | -------------------------------------------- |
| grant_type   | **REQUIRED**该值必须设置“authorization_code” |
| code         | **REQUIRED**授权服务器返回的code             |
| redirect_uri | **REQUIRED**重定向URI                        |
| client_id    | **REQUIRED**注册应用的ID                     |

```http
POST /o/token/ HTTP/1.0
Host: localhost:8000
Connection: close
Content-Length: 183
Accept-Encoding: gzip, deflate
Accept: */*
User-Agent: python-requests/2.18.1
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=x1yxsYhAJ23pXNMXej4tB13dvMFTov&redirect_uri=http%3A%2F%2Flocalhost%3A8000%2Foauth%2Fauthorized%2F&client_id=wt6Pvm2s3vbb8RPE7nlPlugwaMnj58UhFpk8bCPp
```

认证服务器必须要验证客户端的请求

- 对于confidential客户端，需要客户端认证
- 如果客户端包含认证信息，认证客户端
- 验证授权码是否有效
- 验证redirect_uri是否证券

### Access Token Response

如果获取访问令牌的请求是有效的而且认证通过，认证服务器需要发放访问令牌和可选的刷新令牌。如果请求是无效的或认证失败，需要返回一个错误响应。

```
HTTP/1.0 200 OK
Date: Tue, 18 Jul 2017 04:12:00 GMT
Server: WSGIServer/0.1 Python/2.7.9
X-Frame-Options: SAMEORIGIN
Content-Type: application/json
Pragma: no-cache
Cache-Control: no-store

{"access_token": "4C1gRp2TfYKN0tO45fqa6BkFZxTJNU", "token_type": "Bearer", "expires_in": 36000, "refresh_token": "oZX6XtV2OGSVRnvyOoa7qerWddOPKw", "scope": "read write"}
```

# Refreshing an Access Token

如果认证服务器向客户端颁发了刷新令牌，客户端可以使用刷新令牌更新访问令牌。

`Content-Type: application/x-www-form-urlencoded`

| Parameter     | Description                               |
| ------------- | ----------------------------------------- |
| grant_type    | **REQUIRED**该值必须设置为”refresh_token” |
| refresh_token | **REQUIRED**认证服务器颁发的刷新令牌      |
| scope         | **OPTIONAL**                              |

刷新令牌是长期存在证书用来更新访问令牌，刷新令牌必须要与客户端id绑定。此外，客户端必须向认证服务器提供认证信息。

```http
POST /o/token/ HTTP/1.0
Host: localhost:8000
Connection: close
Content-Length: 69
Accept-Encoding: gzip, deflate
Accept: */*
User-Agent: python-requests/2.18.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic d3Q2UHZtMnMzdmJiOFJQRTdubFBsdWd3YU1uajU4VWhGcGs4YkNQcDpWcUF3NDlwb3VXc0lUVU1OcWFrdzczQ0NSRVZzclMyRWd5SG9WeVJnekpWNWRlWnNKQUNBeVdkeEFJMXZRc2RaZk1JQmlCRlRFeUR5YUkyeTJJcU1vR3JEbkFNWE5ITWd4TFRJUDIxRXhIR0tkUmNTVlJ5RjRxOHA5STR5OWhqVQ==

grant_type=refresh_token&refresh_token=oZX6XtV2OGSVRnvyOoa7qerWddOPKw
```

Response:

```http
HTTP/1.0 200 OK
Date: Tue, 18 Jul 2017 06:47:50 GMT
Server: WSGIServer/0.1 Python/2.7.9
X-Frame-Options: SAMEORIGIN
Content-Type: application/json
Pragma: no-cache
Cache-Control: no-store

{"access_token": "gebkQX71vIwmksTsvleRKMzpdysLHY", "token_type": "Bearer", "expires_in": 36000, "refresh_token": "CbpaNXAqyexTQOuCvQf1RUoEN2EZeX", "scope": "read write"}
```

# Accessing Protected Resources

客户端可以同访问访问资源服务器受保护的资源。资源服务器必须要验证访问令牌，保证访问令牌没有过期，并且其作用域能覆盖此资源。

## Access Token Types

“bearer”令牌类型，参考[RFC6750](https://tools.ietf.org/html/rfc6750)

```http
GET /resource/1 HTTP/1.1
Host: example.com
Authorization: Bearer mF_9.B5f-4.1JqM
```

“mac”令牌类型

```http
GET /resource/1 HTTP/1.1
Host: example.com
Authorization: MAC id="h480djs93hd8",
               nonce="274312:dj83hs9s",
               mac="kDZvddkndxvhGRXZhvuDjEWhGeE="
```

## References

- <https://oauth.net/2/>
- <https://tools.ietf.org/html/rfc6749>
- <https://github.com/evonove/django-oauth-toolkit>
- <https://oauth.net/articles/authentication/>
- [https://www.oauth.com](https://www.oauth.com/)