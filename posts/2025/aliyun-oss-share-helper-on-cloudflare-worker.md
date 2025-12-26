---
date: 2025-06-12
title: 用 worker 便捷的分发你OSS中的文件
category: cloudflare大善人
tags:
- cloudflare-workers
- cloudflare-kv
- 白嫖
- aliyun-oss
description: 使用worker实现aliyun oss 下载链接的签名，数据存储在kv。
---

# 用 worker 便捷的分发你OSS中的文件

## 前言

给客户交付定制化的客户端软件的时候，有时候文件比较大微信这种IM软件发不了，走网盘又都需要VIP会员不然就是小水管，让客户开会员这体验也不好。

再加上我个人平时会把一些重要的资料和常用的文件放在aliyun oss中，重要资料用`深度冷归档存储`备份，常用的、要提供下载的放在`标准存储`（不存电影啥的，这个比各种XX网盘的vip要便宜多了）

![oss share page](/imgs/posts/2025/oss-share-page.png)

::: warning 警告 
注意这个分享一遍用于点对点分享，如果发不在公共环境（如论坛等）将会产生昂贵的出相流量账单，项目作者概不负责 
:::

## 实现

- 前端：Vue 3 + TypeScript + Naive UI
- 后端：Cloudflare Worker + TypeScript
- 存储：阿里云 OSS
- 构建工具：Vite
- 部署工具：Wrangler

### cloudflare worker runtime

一开始并秉持着不要重复造轮子的想法优先考虑了阿里云官方实现的node sdk [ali-oss](https://www.npmjs.com/package/ali-oss),
结果发现运行报错，报找不到`http:request`,照道理这个是node的标准库，不应该找不到才是，后来通过阅读cloudflare官方文档发现，worker的runtime是一个裁切过的沙箱环境，
根据文档，发现尚未对`http:request`做兼容处理，那没办法只能自己搓请求算签名了，在用cloudflare提供的`fetch`方法请求。

worker runtime的其他API兼容情况，请查看官方文档 [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)

> 写于2025.12.26
> 
> 根据官网最新描述，目前已对`http:request`进行兼容。
> 后续会尝试使用官方SDK替换目前手搓的接口。


## 具体实现

请看 [GitHub](https://github.com/HynemanKan/aliyun-oss-share-cfworker)

## 使用方法

- 请按照项目中`readme.md`文档完成部署。
- 由于~~安全~~偷懒对原因，没做文档选择界面，请在阿里云的OSS控制台中复制路径，进行长效URL的签发。

