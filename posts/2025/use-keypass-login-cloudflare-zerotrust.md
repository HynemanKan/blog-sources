---
date: 2025-12-21
title: 用 passKey 快速登录你的 cloudflare zeroTrust 应用
category: cloudflare大善人
tags:
- cloudflare-workers
- cloudflare-zeroTrust
- 白嫖
description: 使用worker实现passkey和oidc的桥接，数据存储在D1数据库。
---

# 用 passKey 快速登录你的 cloudflare zeroTrust 应用

## 前言

cloudflare这个大善人的 `zero trust` 服务有免费计划，对于个人来说他限制的50个席位那是绰绰有余，
配合`cloudflared`构建的隧道和`access Control`遍可以安全的在公网访问家中机器上跑的应用（web应用的静态资源甚至能吃到cdn），
但是默认配置的认证是`oneTime pin`就是每次给你发验证码，虽然也可以绑定`github`、`Azure AD`(微软账号)这些三方认证服务，
但对于我们这种web开发者，浏览器全清那是日常调试和debug中长有的操作，那么用这些认证服务势必要重新输入一遍密码。

那么有没有啥优雅而又安全的方法呢，首先想到的就是`passkey`,在win下`windows hello`、mac下的`faceID`和`touchID`跟别提`yubikey`、`1password`这些和系统类型无关的设备和服务了，都通过各种模式对passkey做了适配。

但是我们可以看到cloudflare官方支撑的认证中心中不包含`passkey`这也没什么问题，毕竟`passkey`只是一种认证手段。

![cloudflare support auth center](/imgs/posts/2025/cloudflare-support-auth-center.png)

那么秉持着白嫖到底的思路，就有了这片文章的实践，用`worker`配合`D1`数据库，实现了一个最小化的支持（也仅支持）passkey登录的OIDC服务。

## 先看效果

<iframe style="width:100%;height:600px" src="//player.bilibili.com/player.html?isOutside=true&aid=115757555909080&bvid=BV1p4qSBBESi&cid=34887699866&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

仓库地址  [GitHub](https://github.com/HynemanKan/passkey-openid-cloudflare-worker/)

## 核心协议

让我们先看看两个涉及到的协议

### passkey/webauthn

这一类操作就是挑战本质上就是一个签名挑战的过程。

整体的注册流程和认证流程如下，在浏览器环境具体实现使用的是webauthnAPI；秉承着有轮子就不造新的原则使用现成的npm库`@passwordless-id/webauthn` 实现。

```mermaid
sequenceDiagram
    participant 用户 as 用户(User)
    participant 客户端 as 客户端(Web/APP)
    participant 服务端 as 服务端(Server)
    participant 验证器 as 验证器(Authenticator)<br/>(安全芯片/指纹/面容)

    %% ========== Passkey 注册流程 ==========
    note over 用户,验证器: 【Passkey 注册阶段】
    用户->>客户端: 发起账号注册请求(如:点击"注册Passkey")
    客户端->>服务端: 发送注册初始化请求(含用户信息、客户端信息)
    服务端->>服务端: 生成注册挑战(Challenge)、用户凭证ID
    服务端->>客户端: 返回注册参数<br/>(挑战、RP ID、用户信息、算法等)
    客户端->>验证器: 调用WebAuthn API创建凭证<br/>(navigator.credentials.create)
    验证器->>用户: 请求本地验证(指纹/面容/密码确认)
    用户->>验证器: 完成本地身份验证
    验证器->>验证器: 生成公私钥对(私钥存储在验证器内)
    验证器->>客户端: 返回注册结果<br/>(公钥、凭证ID、签名等)
    客户端->>服务端: 提交注册结果(含公钥、签名)
    服务端->>服务端: 验证签名和挑战合法性
    服务端->>客户端: 注册成功响应(存储公钥+凭证ID关联用户)
    客户端->>用户: 提示Passkey注册完成

    %% ========== Passkey 登录流程 ==========
    note over 用户,验证器: 【Passkey 登录阶段】
    用户->>客户端: 发起登录请求(如:点击"用Passkey登录")
    客户端->>服务端: 发送登录初始化请求(含RP ID、用户标识)
    服务端->>服务端: 生成登录挑战(Challenge)
    服务端->>客户端: 返回登录参数<br/>(挑战、RP ID、允许的凭证ID列表等)
    客户端->>验证器: 调用WebAuthn API获取断言<br/>(navigator.credentials.get)
    验证器->>用户: 请求本地验证(指纹/面容/密码确认)
    用户->>验证器: 完成本地身份验证
    验证器->>验证器: 使用私钥对挑战签名生成断言
    验证器->>客户端: 返回登录断言<br/>(签名、凭证ID、用户存在证明等)
    客户端->>服务端: 提交登录断言(含签名、凭证ID)
    服务端->>服务端: 用存储的公钥验证签名合法性
    服务端->>客户端: 登录成功响应(生成会话Token)
    客户端->>用户: 提示登录成功并进入系统
```
### OIDC（openID connect）

这一块以csdn为首的国内社区会告诉你这个和OAuth2.0是一样的，是基于OAuth2.0实现的，确实整体的流程是大差不差的，但是让我看看cloudflare后台配置页需要哪些参数。

![oidc cloudflare config](/imgs/posts/2025/oidc-config.png)

其中这四个`应用 ID` `客户端密码` `身份验证 URL` `令牌 URL` 都和 OAuth2.0 要的一模一样；但最后一个是`证书URL`不是我们常见的用户信息查询接口。

`OIDC` 在调用认证的时候所构造的前台跳转参数`scope`所给的值中会包含`openid`

根据`OIDC`规范，携带这个参数的时候在`令牌 URL`的返回体中包含一个非对称加密算法签发的openID（jwt token），payload中需要包含scope中其他请求的信息。


应用会去`证书URL`的地址获取`jwks`即公钥，通过公钥验证jwt的签名是否正确，并从payload中获取用户身份。

整体流程如下
```mermaid
sequenceDiagram
    participant User as 用户
    participant Client as 客户端应用(我方)
    participant AuthServer as 授权服务器(对接平台)
    participant JWKS_Endpoint as JWKS端点(AuthServer下)
    %% 1. 初始化授权请求（同基础流程，但scope仅需openid）
    User->>Client: 发起登录请求（点击登录按钮）
    Client->>AuthServer: 重定向到授权端点
    note over Client,AuthServer: 请求参数：<br/>response_type=code、client_id、redirect_uri、<br/>scope=openid、state、nonce(防重放)
    note over AuthServer: 验证client_id/redirect_uri合法性
    %% 2. 授权服务器认证用户
    AuthServer->>User: 展示登录/授权页面
    User->>AuthServer: 输入账号密码并确认授权
    AuthServer->>AuthServer: 验证用户身份，生成授权码(code)
    %% 3. 授权码回调
    AuthServer->>Client: 重定向到redirect_uri
    note over AuthServer,Client: 携带参数：code(授权码)、state(客户端原始值)
    Client->>Client: 验证state（防CSRF攻击）
    %% 4. 换取令牌（核心：获取带签名的JWT令牌）
    Client->>AuthServer: 向令牌端点发送请求
    note over Client,AuthServer: 请求参数：<br/>grant_type=authorization_code、code、<br/>client_id、client_secret、redirect_uri
    AuthServer->>AuthServer: 验证授权码/客户端身份/redirect_uri
    AuthServer->>Client: 返回令牌响应（JWT格式）
    note over AuthServer,Client: 响应体：<br/>id_token(JWT)、access_token(JWT)、<br/>token_type=Bearer、expires_in、iss、aud
    %% 5. 核心：通过JWKS校验JWT签名（对接平台核心逻辑）
    Client->>Client: 解析id_token的header，提取jwk_uri/alg
    note over Client: JWT Header示例：<br/>{ "alg": "RS256", "kid": "xxx", "jwk_uri": "https://xxx/.well-known/jwks.json" }
    
    %% 5.1 获取JWKS公钥集（首次请求缓存，后续复用）
    Client->>JWKS_Endpoint: 请求JWKS公钥集（GET）
    note over Client,JWKS_Endpoint: 请求地址：<br/>从id_token header的jwk_uri或OIDC发现文档获取
    JWKS_Endpoint->>Client: 返回JWKS(JSON格式)
    note over JWKS_Endpoint,Client: JWKS包含：<br/>公钥列表（kty、alg、kid、n、e等RSA参数）、<br/>密钥过期时间、密钥ID(kid)
    %% 5.2 校验JWT签名（核心步骤）
    Client->>Client: 1. 匹配kid：从JWKS中找到与id_token header中kid一致的公钥<br/>2. 验证算法：确认alg（如RS256）与JWKS中一致<br/>3. 校验签名：用公钥验证id_token的签名完整性<br/>4. 验证JWT声明：iss(发行方)、aud(受众)、exp(过期时间)、nonce(防重放)
    note over Client: 签名不通过/声明非法 → 拒绝登录<br/>签名通过/声明合法 → 确认token有效
    %% 6. 完成登录
    Client->>User: 登录成功，建立会话
    note over Client: 可解析id_token的payload获取用户标识(sub)、昵称等信息（无需请求UserInfo）
```

## 整体实现

那么在~~踩坑~~了解完所有协议后，实现就很简单了。（后续内容可能AI味道会比较重，毕竟我已经工作模式初步融入了部分Vibe Coding，写文档这中事情当然就让AI代劳了）

### 前端技术栈
- **Vue 3** - 采用 Composition API 和 `<script setup>` 语法
- **TypeScript** - 提供完整的类型安全
- **Naive UI** - 现代化组件库，支持主题定制
- **Vue Router** - 单页面应用路由管理
- **Vite** - 快速的构建工具和开发服务器

### 后端技术栈
- **Cloudflare Workers** - 无服务器边缘计算平台
- **D1 Database** - Cloudflare 原生 SQLite 数据库
- **Drizzle ORM** - 类型安全的数据库操作库
- **@tsndr/cloudflare-worker-router** - 轻量级 HTTP 路由器
- **@tsndr/cloudflare-worker-jwt** - JWT Token 处理库

### webauthn方案
- **@passwordless-id/webauthn** - WebAuthn API 封装库

## 具体实现

请看 [GitHub](https://github.com/HynemanKan/passkey-openid-cloudflare-worker/)

## 使用方案

- 请按照项目中`readme.md`文档完成部署。
- 由于~~安全~~偷懒对原因，没有做管理面板，需要创建绑定连接的话，请使用cloudflare d1 的管理面板手动在`binding`表中添加数据并构造如下url访问。

```txt
https://${your-domain}/binding?ticket=${challenge_id}
```
