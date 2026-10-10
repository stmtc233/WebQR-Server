# WebQR Server

这是一个用于接收并展示二维码数据的服务端项目。项目支持接收来自客户端（如 Android 应用）上传的二维码字符串，并在网页端实时生成和展示对应的二维码。

本项目提供了两种部署方式：**Node.js** 和 **Cloudflare Workers(推荐)**。您可以根据实际需求和服务器环境选择合适的部署方案。

> 该仓库大量内容为ai编写 请不要完全照搬

## 客户端

https://github.com/stmtc233/WebQR-Client


## 纯前端版本（无需服务器，无需 App）

`docs/` 目录是一个**完全在浏览器里运行**的版本：手机浏览器直接调用摄像头扫码，扫到的内容通过 **WebRTC 点对点**发送到另一台设备的网页上，并在网页上重新生成二维码。不需要部署 Node.js / Cloudflare Workers，也不需要安装 Android 客户端。

### 使用方法

1. 在电脑（或任意设备）上打开页面，点击 **「作为显示端打开」**，页面会显示一个 6 位房间号和配对二维码。
2. 用手机相机扫描配对二维码（或打开同一页面输入房间号），允许摄像头权限后对准要转发的二维码。
3. 扫到新内容后会立即同步到显示端，显示端重新生成二维码，并保留最近 5 条记录。

功能：原生 `BarcodeDetector` 识别（不支持的浏览器自动退回 jsQR）、变焦滑条、闪光灯、防重复发送、断线自动重连、多台扫码端同时连接、显示端记住房间号（扫码端链接可收藏）。

### 发布到公网（GitHub Pages）

整个 `docs/` 都是静态文件（依赖库已放在 `docs/vendor/`，不依赖外部 CDN），可以托管在任何静态网站服务上。用 GitHub Pages：

仓库 **Settings → Pages → Build and deployment → Source 选 “Deploy from a branch”**，分支选 `main`（或包含 `docs/` 的分支），目录选 `/docs`，保存。几分钟后即可通过 `https://stmtc233.github.io/WebQR-Server/` 访问。

> 摄像头只能在 HTTPS 下使用，GitHub Pages / Cloudflare Pages 等都自带 HTTPS。

### 原理与限制

- 设备之间建立连接需要一个**信令服务器**交换连接信息，默认使用 PeerJS 官方的免费公共服务器 `0.peerjs.com`；连接建立后二维码数据直接点对点传输，不经过任何服务器。
- 只配置了 STUN，没有 TURN 中继：两台设备都在对称型 NAT 后面（部分移动网络、公司网络）时可能连不上，这时让两台设备连同一个 Wi-Fi 一般就能解决。
- 如果公共信令服务器在你的网络环境下访问不稳定，可以自建 [peerjs-server](https://github.com/peers/peerjs-server)，然后在 `docs/config.js` 里填写地址。
- 显示端关闭后不保留历史；需要持久化请使用上面的 Cloudflare Workers 方案。


## 目录

- [部署方式一：Node.js](#部署方式一nodejs)
- [部署方式二(推荐)：Cloudflare Workers](#部署方式二推荐cloudflare-workers)
- [两种方式的区别与对比](#两种方式的区别与对比)
- [API 接口说明](#api-接口说明)

---

## 部署方式一：Node.js

适用于拥有云服务器（VPS）或本地服务器环境，支持持久化运行 Node.js 进程的场景。

### 特点
- **实时性高**：使用 `Socket.IO` (WebSocket) 技术，数据上传后网页端毫秒级同步更新。
- **环境要求**：需要安装 Node.js 环境。

### 部署步骤

1.  **安装依赖**
    在项目根目录下运行终端命令：
    ```bash
    npm install
    ```

2.  **启动服务**
    ```bash
    npm start
    # 或者
    node server.js
    ```
    服务默认运行在 `3000` 端口。

3.  **访问**
    浏览器访问 `http://localhost:3000` (或服务器 IP:3000)。

### 注意事项
- 该方式主要依赖内存转发，默认代码中未配置持久化存储，重启服务后之前的二维码记录会丢失。
- 网页端需要配合 `public` 目录下的前端文件使用（Socket.IO 客户端）。

---

## 部署方式二(推荐)：Cloudflare Workers

适用于无服务器环境，无需购买 VPS，利用 Cloudflare 的全球边缘网络进行部署。

### 特点
- **Serverless**：无需管理服务器，按需付费或免费额度通常够用。
- **数据持久化**：使用 Cloudflare D1 存储数据，即使没人访问，数据也会保留。
- **更新机制**：网页端通过轮询（Polling）方式获取最新数据。

### 注意事项
Cloudflare Workers分配的域名在大陆无法直接访问 请绑定自己域名

### 部署步骤

1.  **创建D1数据库**
    运行[D1.sql](Cloudflare/D1.sql)

2.  **部署Workers**
    直接在网页新建workers
    将的[Worker.js](Cloudflare/Worker.js)内容复制进去
    保存

3.  **绑定D1数据库**
    绑定-添加绑定-D1数据库
    变量名称为"DB"数据库选择刚才创建的数据库

### 踩坑
原本这里使用的是KV数据库 但是发现二维码不会变 后来发现是因为KV数据库有缓存 故这里使用的是D1数据库
    
--- 

## 两种方式的区别与对比

| 特性 | Node.js 部署 | Cloudflare Workers 部署 |
| :--- | :--- | :--- |
| **运行环境** | 需要服务器 (VPS/本地) | Serverless (无服务器) |
| **实时通讯** | Socket.IO (WebSocket) | HTTP 轮询 (Polling) |
| **实时性** | 极高 (即时推送) | 较高 (依赖轮询间隔，默认 1秒) |
| **数据存储** | 内存 (重启丢失) | Cloudflare D1 (持久化存储) |
| **维护成本** | 需维护服务器进程 | 无需维护，由 Cloudflare 托管 |
| **新用户访问** | 可能看不到历史二维码 | 可以看到最后一次上传的二维码 |
| **适用场景** | 这里的环境可控，对即时性要求极高 | 希望低成本/免费托管，无需维护服务器 |

---



## API 接口说明

无论使用哪种部署方式，上传数据的接口定义保持一致，方便客户端适配。

### 上传二维码数据

- **URL**: `/upload_qr`
- **Method**: `POST`
- **Content-Type**: `application/json`
- **Body**:
  ```json
  {
      "data": "您的二维码内容字符串"
  }
  ```

### 响应
- **200 OK**: Data received
- **400 Bad Request**: No data provided
