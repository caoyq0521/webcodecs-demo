# WebCodecs 性能测试 Demo

基于 **[web-demuxer](https://github.com/bilibili/web-demuxer)**（WASM 解封装）+ **WebCodecs.VideoDecoder**（浏览器原生硬解）的视频帧性能测试工具集。

无需任何构建步骤，无需安装 npm 包，所有依赖均通过 CDN 按需加载。

## Demo 列表

| Demo | 说明 |
|---|---|
| [`webcodecs-demo.html`](webcodecs-demo.html) | **单路视频性能测试** — 测量解析、读取编码样本、解码、Canvas 渲染各阶段耗时，支持预览指定帧 |
| [`multi-video-demo.html`](multi-video-demo.html) | **多路视频同步测试** — 模拟 1–16 路视频并发解码，测量首帧就绪时间与性能预算（33ms/帧） |

## 运行方式

### 方式一：本地 HTTP 服务器（推荐）

由于 WebCodecs 和 WASM 要求通过 HTTP 协议访问（不能直接打开 `file://`），需要启一个本地服务：

```bash
# Python 3
cd /path/to/webcodecs-demo
python3 -m http.server 8080
# 访问 http://localhost:8080

# 或 Node.js（需全局安装 serve）
npx serve .
```

### 方式二：GitHub Pages（无需服务器）

1. 将此仓库 Push 到 GitHub
2. 进入仓库 **Settings → Pages**
3. Source 选择 `main` 分支，根目录 `/`，Save
4. 稍等片刻，访问 `https://<your-username>.github.io/<repo-name>/`

### 方式三：Netlify / Vercel Drop

将项目目录拖拽到 [Netlify Drop](https://app.netlify.com/drop) 即可一键部署静态站点。

## 环境要求

| 要求 | 说明 |
|---|---|
| **浏览器** | Chrome 107+ 或 Edge 107+（WebCodecs + H.265 硬解支持） |
| **协议** | HTTP 或 HTTPS（不支持 `file://` 直接打开） |
| **网络** | 首次加载时需访问 jsDelivr CDN（`cdn.jsdelivr.net`）下载 web-demuxer JS 和 WASM 文件 |
| **视频格式** | H.264（AVC）或 H.265（HEVC）编码的 MP4/MOV 文件 |

## 性能指标说明

### 单路 Demo

| 指标 | 说明 |
|---|---|
| **解析耗时** | WebDemuxer 初始化 + 文件加载 + 元数据读取 + 获取解码器配置的总时间 |
| **读取样本（平均/帧）** | `readAVPacket` 读取全部编码样本的总时间 / 样本数 |
| **解码耗时（平均/帧）** | `VideoDecoder` 解码全部帧的总时间 / 帧数（硬件加速） |
| **渲染耗时（第50帧）** | `ctx.drawImage(videoFrame)` 绘制到 Canvas 的时间 |

### 多路 Demo

在单路指标基础上新增：

| 指标 | 说明 |
|---|---|
| **解码耗时（平均/路/帧）** | 总解码时间 / 路数 / 每路帧数 |
| **首帧就绪（最慢路）** | 从开始解码到所有路都输出第一帧的墙钟时间（用户感知的等待时间） |
| **首帧就绪（各路均值）** | 各路首帧就绪时间的平均值 |
| **性能预算** | 平均帧时是否 ≤ 33ms（30fps 实时播放预算） |

## 技术实现

```
本地视频文件
    │
    ├─ WebDemuxer（web-demuxer WASM via jsDelivr CDN）
    │   └─ readAVPacket() → EncodedVideoChunk[]
    │
    ├─ VideoDecoder（WebCodecs 浏览器原生硬解）
    │   └─ output: VideoFrame
    │
    └─ Canvas 2D
        └─ ctx.drawImage(VideoFrame)
```

**关键设计原则：**
- `VideoFrame` 在渲染完成后立即 `close()`，避免显存泄漏
- 多路测试中，Demuxer 实例并发初始化（`Promise.allSettled`），单路失败不影响其他路
- 解码时从关键帧（keyframe）开始下发，保证 delta 帧可正常解码
- API 兼容层同时支持 web-demuxer v2（`readMediaPacket`）和 v3（`readAVPacket`）

## 文件结构

```
webcodecs-demo/
├── index.html              # 导航首页
├── webcodecs-demo.html     # 单路视频性能测试
├── multi-video-demo.html   # 多路视频同步测试
└── README.md               # 本文档
```

## 相关链接

- [web-demuxer GitHub](https://github.com/bilibili/web-demuxer)
- [WebCodecs API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)
- [VideoDecoder - MDN](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder)
- [WebCodecs 兼容性表](https://caniuse.com/webcodecs)
