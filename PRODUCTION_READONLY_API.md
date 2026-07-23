# MaMage 生产环境只读 API 联调说明

本文档给前端联调使用。文档中的接口只读取生产数据，不提供创建、修改、删除和上传能力。

生产后端：

```text
https://mamage.wenyuli.site
```

当前可用的内网后端入口（2026-07-23 实测）：

```text
物理局域网：  http://10.100.65.147:8080
ZeroTier：    http://10.11.12.63:8080
```

两个地址的 `GET /api/health` 均返回 `{"status":"ok"}`。前端人员和 Mac Mini 处在同一校园局域网时优先使用物理局域网地址；不在同一局域网但加入了 ZeroTier 时使用 ZeroTier 地址。物理局域网 DHCP 地址可能轮换，ZeroTier 地址通常更稳定。

## 1. 推荐调用方式

### 1.1 本项目本地开发

在 `mamage-web` 目录执行：

```bash
npm install
npm run dev:prod
```

默认打开：

```text
http://localhost:5173
```

`dev:prod` 会把本地前端的 `/api`、`/uploads` 和 `/static` 请求代理到生产后端，并关闭前端直传对象存储。前端代码统一使用相对路径，不要把生产域名写死到组件里：

```js
const response = await fetch('/api/projects/63', {
  headers: token ? { Authorization: `Bearer ${token}` } : {},
});
const data = await response.json();
```

### 1.2 其他前端项目

开发服务器配置代理：

```js
{
  '/api': {
    target: 'https://mamage.wenyuli.site',
    changeOrigin: true,
    secure: true,
  },
  '/uploads': {
    target: 'https://mamage.wenyuli.site',
    changeOrigin: true,
    secure: true,
  },
}
```

然后仍然调用 `/api/...`。这样可以避免跨域，也不会把对象存储密钥暴露给浏览器。

### 1.3 内网开发代理

需要走物理局域网时，将开发服务器代理目标改为：

```bash
MAMAGE_BACKEND_URL=http://10.100.65.147:8080 \
MAMAGE_DISABLE_DIRECT_UPLOAD=1 \
WEBPACK_DEV_SERVER_PORT=5173 \
npx webpack serve --mode development
```

需要走 ZeroTier 时只替换目标地址：

```bash
MAMAGE_BACKEND_URL=http://10.11.12.63:8080 \
MAMAGE_DISABLE_DIRECT_UPLOAD=1 \
WEBPACK_DEV_SERVER_PORT=5173 \
npx webpack serve --mode development
```

本地页面仍然访问 `http://localhost:5173`，前端代码仍然调用 `/api/...`，不需要把内网 IP 写进业务组件。

### 1.4 认证

分享接口和健康检查不需要登录。组织内项目、照片、人脸等接口需要生产环境 JWT：

```http
Authorization: Bearer <生产环境 JWT>
```

不要把 JWT 写入代码、提交 Git 或发到群里。完整图库数据按账号所属组织隔离；没有 JWT 时，项目列表通常不会返回组织内数据。

## 2. 最常用的项目和照片接口

以下接口足够完成首页、相册页、照片详情和搜索页的前端预览。

### 健康检查

```http
GET /api/health
```

示例响应：

```json
{ "status": "ok" }
```

### 项目列表

```http
GET /api/projects?limit=24
GET /api/projects/list?page=1&pageSize=24&keyword=团代会
GET /api/projects/scenery
GET /api/projects/:projectId
```

示例：

```js
const project = await fetch('/api/projects/63').then((r) => r.json());
```

### 照片列表和详情

```http
GET /api/photos?projectId=63&limit=100
GET /api/photos?projectId=63&type=normal&limit=100
GET /api/photos/:photoId
GET /api/photos/scenery/random?limit=20
```

照片列表通常包含：

```text
id, projectId, url, thumbUrl, publicDownloadUrl, playbackUrl,
title, description, tags, type, aiStatus, aiScore, aiQuality,
timelineSectionId, timelineSectionName, photographerName,
createdAt, updatedAt
```

### 智能照片检索

```http
GET /api/photos/search?q=王婧琦&page=1&pageSize=20&sort=relevance
GET /api/photos/search?q=红色背景&projectId=63&page=1&pageSize=20&sort=newest
```

搜索参数：

| 参数 | 说明 |
| --- | --- |
| `q` | 搜索关键词，可以是人物、标签、项目名或描述 |
| `page` | 页码，从 1 开始 |
| `pageSize` | 每页数量，建议不超过 50 |
| `projectId` | 可选，相册范围 |
| `sort` | `relevance` 或 `newest` |
| `smart` | 可选，`1` 会启用 AI 增强检索，测试普通 UI 时建议不传，避免额外消耗 AI 配额 |

### 照片原图和缩略图读取

用于前端 Canvas 调色、直方图或原图预览：

```http
GET /api/photos/:photoId/pixel-source?variant=thumb
GET /api/photos/:photoId/pixel-source?variant=original
```

用于判断内网对象存储直连是否可用：

```http
GET /api/photos/direct-status
```

在允许的校园网 HTTP 入口下，可获取临时签名地址：

```http
GET /api/photos/:photoId/direct-url?variant=thumb
GET /api/photos/:photoId/direct-url?variant=original
GET /api/photos/:photoId/direct-url?variant=public
GET /api/photos/:photoId/direct-url?variant=playback
```

签名地址只在短时间内有效。前端优先使用接口返回的 `thumbUrl`、`url`、`publicDownloadUrl` 和 `playbackUrl`，不要自己拼对象存储地址。

## 3. 完整只读路由表

下表是当前生产后端的全部主要 `GET` 数据路由。除特别标注外，带“需 `photos.view`”的接口都需要登录并携带 JWT。

### 公共路由

| 路由 | 认证 | 用途 |
| --- | --- | --- |
| `GET /api/health` | 无 | 服务健康检查 |
| `GET /api/auth/providers` | 无 | 查看密码登录、钉钉登录是否启用 |
| `GET /api/organizations?q=关键词&limit=50` | 无 | 读取组织列表 |
| `GET /api/share/:shareCode?limit=100&offset=0` | 无 | 读取公开分享相册 |
| `GET /api/wechat-preview/:token` | 无 | 读取公众号排版预览 HTML |
| `GET /api/wx-img?url=<微信图片地址>` | 无 | 读取白名单微信图片代理 |
| `GET /api/image/<object-key>` | 签名或按生产配置 | 读取对象存储中的图片、视频媒体流 |
| `GET /uploads/<path>` | 按生产配置 | 兼容旧的本地静态媒体地址 |

公开分享接口是最适合不接入登录功能的前端演示入口。它只返回分享码允许访问的照片。

### 项目路由

| 路由 | 认证 | 用途 |
| --- | --- | --- |
| `GET /api/projects?limit=10` | 可带 JWT | 首页项目列表 |
| `GET /api/projects/list?page=1&pageSize=6&keyword=` | 可带 JWT | 分页、关键词项目列表 |
| `GET /api/projects/scenery` | 可带 JWT | 风景相册列表及预览图 |
| `GET /api/projects/:projectId` | 可带 JWT | 相册详情、时间轴、照片摘要 |

### 照片路由

| 路由 | 认证 | 用途 |
| --- | --- | --- |
| `GET /api/photos?projectId=:id&limit=100` | 需 `photos.view` | 相册照片列表 |
| `GET /api/photos/scenery/random?limit=20` | 需 `photos.view` | 随机风景照片 |
| `GET /api/photos/search?...` | 登录或演示模式 | 智能照片搜索 |
| `GET /api/photos/:photoId` | 需 `photos.view` | 单张照片详情 |
| `GET /api/photos/:photoId/pixel-source?variant=thumb` | 需 `photos.view` | 缩略图像素源 |
| `GET /api/photos/:photoId/pixel-source?variant=original` | 需 `photos.view` | 原图像素源 |
| `GET /api/photos/direct-status` | 需 `photos.view` | 查询内网直连能力 |
| `GET /api/photos/:photoId/direct-url?variant=...` | 需 `photos.view` | 获取临时对象存储签名地址 |
| `GET /api/photos/:photoId/faces` | 需 `photos.view` | 读取照片已有的人脸框 |
| `GET /api/photos/group-rescue/:jobId` | 需 `photos.edit` | 查询已有合照救场任务 |
| `GET /api/photos/zip-direct` | 需 `photos.view` | 查询当前用户已有直连打包任务 |
| `GET /api/photos/zip-direct/:jobId` | 需 `photos.view` | 查询指定直连打包任务 |
| `GET /api/photos/zip?ticket=:ticket` | 一次性票据 | 下载已有票据对应的 ZIP，不用于普通数据查询 |
| `GET /api/photos/zip?shareCode=:code&photoIds=1,2` | 分享码 | 下载分享范围内的 ZIP，不用于普通数据查询 |

`/api/photos/zip-direct*` 只是查询已经存在的任务。不要为了测试 UI 主动创建打包任务，因为创建任务使用的是写接口，并会占用服务器和对象存储资源。

### 人脸和人物档案

| 路由 | 认证 | 用途 |
| --- | --- | --- |
| `GET /api/faces?photoId=:id` | 需 `photos.view` | 读取照片人脸列表 |
| `GET /api/faces?personId=:id` | 需 `photos.view` | 读取人物关联人脸 |
| `GET /api/faces/cluster/config` | 需 `photos.view` | 读取人脸聚类阈值 |
| `GET /api/faces/:faceId/person` | 需 `photos.view` | 读取人脸对应人物档案 |
| `GET /api/faces/:faceId` | 需 `photos.view` | 读取单个人脸档案 |
| `GET /api/faces/person?faceId=:id` | 需 `photos.view` | 按人脸读取人物档案 |
| `GET /api/faces/profile?personId=:id` | 需 `photos.view` | 读取人物档案 |
| `GET /api/persons?page=1&pageSize=20&q=` | 需 `photos.view` | 人物档案列表 |
| `GET /api/persons/:personId` | 需 `photos.view` | 单个人物档案 |
| `GET /api/persons/:personId/photos` | 需 `photos.view` | 人物关联照片 |
| `GET /api/photos/:photoId/faces` | 需 `photos.view` | 照片人脸框和识别结果 |

### 相似度和 AI 结果读取

| 路由 | 认证 | 用途 |
| --- | --- | --- |
| `GET /api/similarity/groups?projectId=:id` | 需 `photos.view` | 读取相似照片分组 |
| `GET /api/similarity/pairs?projectId=:id&minScore=0.8` | 需 `photos.view` | 读取相似照片对 |
| `GET /api/similarity/groups/simple?projectId=:id` | 登录或演示模式 | 读取默认相似照片分组 |
| `GET /api/ai/news/jobs/:jobId` | 需 `ai.generate` | 查询新闻稿 AI 任务 |
| `GET /api/ai/news/batches/:batchId` | 需 `ai.generate` | 查询新闻稿批次 |
| `GET /api/wechat-style/blocks` | 需 `ai.generate` | 读取已保存的排版样式块 |
| `GET /api/wechat-compositions` | 需 `ai.generate` | 读取排版收藏列表 |
| `GET /api/wechat-compositions/:id` | 需 `ai.generate` | 读取单条排版收藏 |
| `GET /api/favorites` | 需 `ai.generate` | 读取用户收藏 |

相似度接口可能在照片很多时计算量较大，做 UI 联调时建议先限制在单个项目，并使用较小的照片数量。

### ZIP 和任务查询的说明

`GET /api/photos/zip-direct*` 只查询已经存在的任务；`GET /api/photos/zip` 会实际开始文件流下载，可能消耗服务器带宽。它们不适合作为普通页面初始化请求。前端只做列表、详情和预览时可以完全不调用这些接口。

### 用户读取

| 路由 | 认证 | 用途 |
| --- | --- | --- |
| `GET /api/users/me` | 登录 | 当前登录用户信息 |
| `GET /api/users/all` | 管理员 | 读取用户列表，不含密码 |

## 4. 前端调用模板

```js
const API = ''; // 本地代理时保持为空；生产页面也可以保持为空

function getToken() {
  return localStorage.getItem('mamage_jwt_token') || '';
}

async function readApi(path, options = {}) {
  const headers = new Headers(options.headers || {});
  const token = getToken();
  if (token) headers.set('Authorization', `Bearer ${token}`);

  const response = await fetch(`${API}${path}`, {
    ...options,
    method: 'GET',
    headers,
  });

  if (!response.ok) {
    const message = await response.text();
    throw new Error(`${response.status}: ${message}`);
  }

  const contentType = response.headers.get('content-type') || '';
  return contentType.includes('application/json')
    ? response.json()
    : response.text();
}

const projects = await readApi('/api/projects/list?page=1&pageSize=24');
const album = await readApi('/api/projects/63');
const photos = await readApi('/api/photos?projectId=63&limit=100');
const search = await readApi('/api/photos/search?q=团代会&page=1&pageSize=20');
```

## 5. 媒体地址使用规则

接口返回的媒体字段可能是绝对地址，也可能是相对地址：

```js
function assetUrl(value) {
  if (!value) return '';
  return /^https?:\/\//i.test(value) ? value : `${location.origin}${value.startsWith('/') ? value : `/${value}`}`;
}

const thumb = assetUrl(photo.thumbUrl || photo.url);
const original = assetUrl(photo.url);
const publicDownload = assetUrl(photo.publicDownloadUrl || photo.url);
const playback = assetUrl(photo.playbackUrl || photo.url);
```

建议：

- 网格使用 `thumbUrl`。
- 照片查看器使用 `url`，调色/直方图使用 `pixel-source`。
- 公网单张下载优先使用 `publicDownloadUrl`。
- 视频播放优先使用 `playbackUrl`，并保留浏览器 Range 请求。
- 不要在前端读取或保存 S3/COS Access Key、Secret Key。

## 6. 明确不要调用的接口

为了保证本次联调不修改生产数据，不要调用以下方法：

```text
POST /api/upload/*                    # 上传
POST /api/projects                    # 新建相册
POST /api/projects/:id/update         # 修改相册
DELETE /api/projects/:id              # 删除相册
POST /api/photos/delete               # 删除照片
PATCH /api/photos/:id                 # 修改照片
POST /api/photos/zip-ticket           # 创建打包下载任务
POST /api/photos/zip-direct           # 创建直连打包任务
POST /api/photos/:id/rendered         # 生成渲染结果
POST /api/faces/find-me               # 人脸搜索任务
POST /api/faces/find-me/share         # 分享页人脸搜索任务
POST /api/wechat-style/*              # 抓取或保存排版
POST /api/wechat-compositions         # 保存排版收藏
POST /api/ai/news/*                   # 生成 AI 新闻稿
```

`POST /api/users/login` 只用于获得 JWT，不会创建照片或修改图库内容。若前端没有现成 JWT，可以让测试人员通过正式登录页登录后再联调；不要把账号密码硬编码进前端。

特别注意，下面这个接口虽然是 `GET`，但加上 `detect=1` 后会触发人脸检测并写入结果：

```text
GET /api/photos/:photoId/faces?detect=1
```

只读联调时只能调用不带 `detect=1` 的版本。
