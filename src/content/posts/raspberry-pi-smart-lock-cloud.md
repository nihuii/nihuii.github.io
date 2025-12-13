---
title: 进阶玩法：基于树莓派的智能门锁系统 (微信云托管版)
published: 2025-12-14
description: 告别本地服务器和内网穿透！利用 Docker 和微信云托管，打造一个真正可随时随地访问的“Serverless”智能门锁。
image: ""
tags: [树莓派, 微信云托管, Docker, MQTT, 微信小程序]
category: 硬件开发
draft: false
---

> 在上一篇文章[《基于树莓派的物联网全能智能门锁系统(服务器本地部署)》](/posts/raspberry-pi-smart-lock-version1)中，我们介绍了一个需要在本地电脑运行 Flask 服务器的方案。
>
> 虽然那个方案可行，但它有一个痛点：**电脑必须一直开机，且为了外网访问需要复杂的内网穿透配置**。
>
> 今天，我们来一次架构升级！保持树莓派硬件端代码不变，将服务端迁移至**微信云托管 (WeChat Cloud Hosting)**，实现真正的云端控制。

## 💡 为什么要升级到云托管？

与之前的本地部署版本相比，新版本主要解决了以下问题：

1.  **无需公网 IP / 内网穿透**：微信云托管通过内网链路直接与小程序通信，不需要你有域名或公网 IP。
2.  **Serverless 体验**：基于容器化部署（Docker），无需维护服务器运维，按量计费（有免费额度），成本极低。
3.  **更安全**：小程序与云托管之间走微信私有链路，天然支持 HTTPS，比本地 HTTP 更安全。

## 🔗 项目开源

本项目包含**硬件端（不变）**、**云托管服务端（Cloud）**、**新版小程序（WeChat）**三部分代码。

- **GitHub 仓库地址**：[https://github.com/nihuii/raspberry-pi-smart-lock](https://github.com/nihuii/raspberry-pi-smart-lock)
- **GitHub 仓库地址**：[nihuii/wechat-mini-program: 代码为《基于树莓派的智能门锁》项目的微信小程序端代码和微信云托管（服务器端）代码](https://github.com/nihuii/wechat-mini-program)

---

## 🏗️ 系统架构对比

### 1. 硬件端 (Hardware) - **保持不变**
树莓派端的 Python 代码（`GraduationProject` 目录）不需要任何修改。它依然负责：
* 人脸识别与 GPIO 控制。
* 作为 MQTT Client，监听来自 Broker 的开锁指令。

### 2. 服务端 (Cloud Server) - **核心升级**
* **旧版**：Python Flask 代码直接运行在本地电脑上 (`cloud_server`)。
* **新版**：Python Flask 代码被打包成 **Docker 镜像**，运行在微信云托管的容器中 (`Cloud` 目录)。

### 3. 小程序端 (Mini Program) - **通信方式改变**
* **旧版**：使用 `wx.request` 请求本地 IP 地址 (`wechat_miniprogram`)。
* **新版**：使用 `wx.cloud.callContainer` 直接调用云托管服务 (`WeChat`)，无需配置域名。

---

## ☁️ 核心代码解析

### 1. 服务端容器化 (Dockerfile)

在 `Cloud` 文件夹中，我新增了 `Dockerfile`，这是部署到云托管的关键。它告诉云平台如何构建我们的 Python 环境：

```dockerfile
# 使用轻量级的 Python 基础镜像
FROM python:3.9-slim

# 设置工作目录
WORKDIR /app

# 复制当前目录内容到容器中
COPY . /app

# 安装依赖 (Flask, paho-mqtt 等)
RUN pip install --no-cache-dir -r requirements.txt

# 暴露端口 (微信云托管默认监听 80)
EXPOSE 80

# 启动 Flask 应用
CMD ["python3", "app.py"]
```

### 2. 小程序调用云服务

在 `WeChat` 新版小程序代码中，我们不再需要硬编码服务器的 IP 地址，而是直接通过“环境 ID”调用服务。

查看 `miniprogram/app.js` 或 API 调用部分，你会发现代码变成了这样：

JavaScript

```
// 旧版写法：
// wx.request({
//   url: '[http://192.168.1.100:5000/api/remote_unlock](http://192.168.1.100:5000/api/remote_unlock)',
//   ...
// })

// 新版写法 (使用 callContainer)：
wx.cloud.callContainer({
  "config": {
    "env": "你的云托管环境ID" 
  },
  "path": "/api/remote_unlock", // 直接写路由
  "header": {
    "X-WX-SERVICE": "flask-service" // 服务名称
  },
  "method": "POST",
  "data": {
    "token": "你的密钥"
  }
})
```

这种方式不仅速度快，而且自动处理了跨域和鉴权问题。

------

## 🚀 部署指南 (云托管版)

硬件部分的连接和树莓派代码部署请参考[上一篇博客][基于树莓派的物联网全能智能门锁系统(服务器本地部署) - Firefly](https://nihuii.github.io/posts/raspberry-pi-smart-lock-version1/)，这里只介绍云端和小程序的部署。

### 第一步：开通微信云托管

1. 登录 [微信公众平台](https://mp.weixin.qq.com/)，进入“云托管”。
2. 创建一个新的环境（Environment），记下 **环境 ID**。
3. 创建一个服务，命名为 `flask-service`。

### 第二步：部署服务端 (Cloud)

你有两种方式部署代码：

- **方式 A（推荐）：** 将 GitHub 仓库授权给微信云托管，选择 `Cloud` 目录作为根目录，勾选“代码变更自动部署”。
- **方式 B（手动）：** 在云托管控制台点击“版本管理” -> “上传代码包”，将 `Cloud` 文件夹打包上传。

部署成功后，你会在控制台看到“部署成功”的状态，并且可以通过公网访问流水线测试接口。

### 第三步：配置小程序 (WeChat)

1. 使用微信开发者工具打开 `wechat_mini_program/WeChat` 目录。
2. 找到 `miniprogram/envList.js` (或者 `app.js` 中的配置位置)，填入你的云托管 **环境 ID**。
3. **重要**：确保你的 MQTT Broker 地址是公网可访问的（因为云容器在腾讯云内部，树莓派在家里，它们需要通过一个公网 MQTT Broker 也就是“消息队列”来传话）。

### 第四步：联调测试

1. 树莓派运行 `main_pro.py`，确保连接上 MQTT。
2. 云托管服务显示“运行中”。
3. 在小程序点击“远程开锁”。
4. **流程跑通**：小程序 -> 微信云链路 -> 云托管 Flask 容器 -> 发布 MQTT 消息 -> 树莓派收到 -> **开锁成功！** 🔓

------

## 📝 总结

通过这次改造，我们将一个依赖本地环境的硬件项目，成功进化为了一个现代化的 IoT 应用。

- **Cloud (服务端)**：利用 Docker 实现了“一次构建，处处运行”。
- **WeChat (前端)**：利用云原生能力，实现了免配置域名的安全通信。

如果你希望你的智能门锁不仅能在局域网玩，还能在上班、出差时随时查看状态，那么强烈建议你尝试这个“云托管版本”！

如有疑问，欢迎在 GitHub 提 Issue 交流：https://github.com/nihuii/raspberry-pi-smart-lock