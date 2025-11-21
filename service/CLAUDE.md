# 后端模块文档

[根目录](../CLAUDE.md) > **service (后端模块)**

---

## 变更记录 (Changelog)

| 日期 | 版本 | 变更内容 |
|------|------|----------|
| 2025-11-21 | v1.0.0 | 初始化后端模块文档 |

---

## 模块职责

后端模块负责：
- 提供 RESTful API 服务
- 代理转发到各种 AI 服务（OpenAI, Midjourney, Suno, Luma 等）
- 处理用户认证与授权
- 文件上传与存储管理
- API 请求限流
- 防爆破验证

---

## 入口与启动

### 主入口文件
`service/src/index.ts`:
```typescript
const app = express()
const router = express.Router()

// 监听端口 3002
app.listen(3002, () => console.log('Server is running on port 3002'))
```

### 启动命令
- **开发模式**: `pnpm dev` (使用 esno watch)
- **构建**: `pnpm build` (使用 tsup)
- **生产模式**: `pnpm prod`

---

## 对外接口

### API 端点列表

#### 核心 ChatGPT 接口

| 路径 | 方法 | 描述 | 认证 |
|------|------|------|------|
| `/api/chat-process` | POST | 流式对话接口 | authV2 |
| `/api/config` | POST | 获取配置信息 | auth |
| `/api/session` | POST | 获取会话信息 | 无 |
| `/api/verify` | POST | 密码验证 | 无 |
| `/api/reg` | GET | 注册 Cookie | 无 |

#### Midjourney 代理

| 路径 | 方法 | 描述 |
|------|------|------|
| `/mjapi/*` | ALL | 代理到 `MJ_SERVER` |

#### OpenAI 通用接口

| 路径 | 方法 | 描述 |
|------|------|------|
| `/openapi/v1/upload` | POST | 文件上传 |
| `/openapi/v1/audio/transcriptions` | POST | Whisper 语音转文字 |
| `/openapi/v1/images/edits` | POST | 图片编辑 |
| `/openapi/*` | ALL | 代理到 OpenAI API |

#### 其他 AI 服务代理

| 路径 | 方法 | 描述 |
|------|------|------|
| `/sunoapi/*`, `/suno/*` | ALL | Suno 音乐生成 |
| `/luma/*`, `/pro/luma/*` | ALL | Luma 视频生成 |
| `/viggle/*`, `/pro/viggle/*` | ALL | Viggle 舞蹈生成 |
| `/viggle/asset`, `/pro/viggle/asset` | POST | Viggle 文件上传 |
| `/runway/*` | ALL | Runway 视频生成 |
| `/runwayml/*` | ALL | Runway ML 视频生成 |
| `/kling/*` | ALL | Kling 视频生成 |
| `/ideogram/*` | ALL | Ideogram 绘图 |
| `/ideogram/remix` | POST | Ideogram 图片上传 |
| `/pika/*` | ALL | Pika 视频生成 |
| `/udio/*` | ALL | Udio 音乐生成 |
| `/pixverse/*` | ALL | PixVerse 视频生成 |

#### Cloudflare R2 存储

| 路径 | 方法 | 描述 |
|------|------|------|
| `/openapi/pre_signed` | POST | 获取 R2 预签名 URL |

---

## 关键依赖与配置

### 核心依赖
```json
{
  "express": "^4.18.2",
  "chatgpt": "^5.1.2",
  "axios": "^1.3.4",
  "dotenv": "^16.0.3",
  "multer": "1.4.5-lts.1",
  "aws-sdk": "^2.1535.0",
  "express-http-proxy": "^2.0.0",
  "express-rate-limit": "^6.7.0",
  "https-proxy-agent": "^5.0.1",
  "socks-proxy-agent": "^7.0.0"
}
```

### 配置文件
- **package.json**: 依赖与脚本
- **tsup.config.ts**: 构建配置
- **service/.env**: 环境变量（敏感，不提交）

### 环境变量

详见根目录 [CLAUDE.md](../CLAUDE.md) 的"环境变量配置"部分。

---

## 数据模型

### ChatGPT 相关类型

`service/src/chatgpt/types.ts`:
```typescript
export interface ChatMessage {
  id: string
  text: string
  role: 'user' | 'assistant' | 'system'
  // ...
}

export interface RequestOptions {
  message: string
  lastContext?: ChatContext
  process?: (chat: ChatMessage) => void
  systemMessage?: string
  temperature?: number
  top_p?: number
}
```

### 通用类型

`service/src/types.ts`:
```typescript
export interface RequestProps {
  prompt: string
  options?: {
    conversationId?: string
    parentMessageId?: string
  }
  systemMessage?: string
  temperature?: number
  top_p?: number
}
```

---

## 中间件

### 认证中间件 (`service/src/middleware/auth.ts`)

**authV2**: 主认证中间件
- 检查 `AUTH_SECRET_KEY`
- 验证请求头 `Authorization` 或 `x-ptoken`
- 防爆破：记录错误次数和 IP

**auth**: 简单认证中间件

**verify**: 验证密码接口
- POST `/api/verify`
- 返回 token (Cookie)

**turnstileCheck**: Cloudflare Turnstile 验证

### 限流中间件 (`service/src/middleware/limiter.ts`)

使用 `express-rate-limit`:
```typescript
export const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100, // 最多 100 次请求
})
```

---

## 代理服务实现

### Midjourney 代理 (`/mjapi/*`)

```typescript
app.use('/mjapi', authV2, proxy(process.env.MJ_SERVER, {
  proxyReqPathResolver: (req) => req.originalUrl.replace('/mjapi', ''),
  proxyReqOptDecorator: (proxyReqOpts) => {
    if (process.env.MJ_API_SECRET)
      proxyReqOpts.headers['mj-api-secret'] = process.env.MJ_API_SECRET
    proxyReqOpts.headers['Mj-Version'] = pkg.version
    return proxyReqOpts
  }
}))
```

### OpenAI 代理 (`/openapi/*`)

```typescript
app.use('/openapi', authV2, turnstileCheck, proxy(API_BASE_URL, {
  proxyReqPathResolver: (req) => req.originalUrl.replace('/openapi', ''),
  proxyReqOptDecorator: (proxyReqOpts) => {
    proxyReqOpts.headers['Authorization'] = 'Bearer ' + process.env.OPENAI_API_KEY
    proxyReqOpts.headers['Mj-Version'] = pkg.version
    return proxyReqOpts
  }
}))
```

### 其他服务代理 (`service/src/myfun.ts`)

**关键代理函数**:
- `sunoProxy`: Suno 音乐生成代理
- `lumaProxy`: Luma 视频生成代理
- `viggleProxy`: Viggle 舞蹈生成代理
- `viggleProxyFileDo`: Viggle 文件上传处理
- `runwayProxy`, `runwaymlProxy`: Runway 视频代理
- `klingProxy`: Kling 视频代理
- `ideoProxy`, `ideoProxyFileDo`: Ideogram 代理
- `pikaProxy`: Pika 视频代理
- `udioProxy`: Udio 音乐代理
- `pixverseProxy`: PixVerse 视频代理
- `GptImageEdit`: GPT 图片编辑

---

## 文件上传与存储

### 本地存储

使用 `multer` 处理文件上传：
```typescript
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    const uploadPath = `./uploads/${formattedDate()}/`
    if (!fs.existsSync(uploadPath)) fs.mkdirSync(uploadPath)
    cb(null, uploadPath)
  },
  filename: (req, file, cb) => {
    const filename = Date.now() + path.extname(file.originalname)
    cb(null, filename)
  }
})
```

访问路径: `/uploads/{date}/{filename}`

### Cloudflare R2 存储

使用 AWS S3 SDK 连接 R2：
```typescript
const R2Client = () => {
  const s3 = new AWS.S3({
    endpoint: new AWS.Endpoint(`https://${accountId}.r2.cloudflarestorage.com`),
    region: 'auto',
    credentials: new AWS.Credentials(accessKeyId, accessKeySecret),
    signatureVersion: 'v4',
  })
  return s3
}
```

获取预签名 URL:
```typescript
POST /openapi/pre_signed
{
  "file_name": ".jpg",
  "ContentType": "image/jpeg"
}

// 返回
{
  "status": "Success",
  "data": {
    "up": "预签名上传 URL",
    "url": "公开访问 URL"
  }
}
```

### 自定义文件服务

通过 `FILE_SERVER` 环境变量指定外部文件服务：
```typescript
if (process.env.FILE_SERVER) {
  // 转发到自定义文件服务
  const formData = new FormData()
  formData.append('file', fileBuffer, { filename: originalname })
  await axios.post(process.env.FILE_SERVER, formData)
}
```

---

## ChatGPT API 封装

### 核心逻辑 (`service/src/chatgpt/index.ts`)

**API 模式选择**:
- 如果有 `OPENAI_API_KEY`: 使用官方 API (`ChatGPTAPI`)
- 如果有 `OPENAI_ACCESS_TOKEN`: 使用非官方代理 (`ChatGPTUnofficialProxyAPI`)

**chatReplyProcess**:
```typescript
async function chatReplyProcess(options: RequestOptions) {
  const { message, lastContext, process, systemMessage, temperature, top_p } = options

  const response = await api.sendMessage(message, {
    timeoutMs,
    systemMessage,
    completionParams: { model, temperature, top_p },
    parentMessageId: lastContext?.parentMessageId,
    onProgress: (partialResponse) => {
      process?.(partialResponse) // 流式返回
    }
  })

  return response
}
```

**代理支持**:
- SOCKS 代理: `SOCKS_PROXY_HOST`, `SOCKS_PROXY_PORT`
- HTTPS 代理: `HTTPS_PROXY`, `ALL_PROXY`

---

## 测试与质量

### 当前状态
- **单元测试**: 无
- **集成测试**: 无
- **代码覆盖率**: 未测量

### 代码质量工具
- **ESLint**: 使用 `@antfu/eslint-config`
- **TypeScript**: 启用严格模式

### 推荐测试策略
1. **单元测试**: 使用 Jest 测试工具函数
2. **API 测试**: 使用 Supertest 测试端点
3. **Mock**: 使用 nock 模拟外部 API

---

## 常见问题 (FAQ)

### Q1: 如何添加新的代理服务？
1. 在 `service/src/myfun.ts` 创建代理函数
2. 在 `service/src/index.ts` 添加路由：
   ```typescript
   app.use('/newservice', authV2, newServiceProxy)
   ```

### Q2: 如何调试代理请求？
- 查看服务器控制台日志
- 设置环境变量 `OPENAI_API_DISABLE_DEBUG=false` 开启 ChatGPT API 调试

### Q3: 文件上传失败怎么办？
1. 检查 `API_UPLOADER` 是否设置
2. 检查上传目录权限 (`./uploads/`)
3. 检查文件大小限制 (`UPLOAD_IMG_SIZE`)

### Q4: 如何配置防爆破？
设置环境变量：
```bash
AUTH_SECRET_KEY=mysecret
AUTH_SECRET_ERROR_COUNT=3  # 3次错误
AUTH_SECRET_ERROR_TIME=10  # 锁定10分钟
```

**注意**: Nginx 需要配置 `proxy_set_header X-Forwarded-For $remote_addr;`

### Q5: 如何使用 R2 存储？
设置环境变量：
```bash
R2_DOMAIN=https://your-domain.com
R2_BUCKET_NAME=your-bucket
R2_ACCOUNT_ID=your-account-id
R2_KEY_ID=your-key-id
R2_KEY_SECRET=your-key-secret
```

---

## 相关文件清单

### 核心文件
```
service/
├── src/
│   ├── index.ts              # 主入口，所有路由定义
│   ├── chatgpt/
│   │   ├── index.ts          # ChatGPT API 封装
│   │   └── types.ts          # ChatGPT 类型定义
│   ├── middleware/
│   │   ├── auth.ts           # 认证中间件
│   │   └── limiter.ts        # 限流中间件
│   ├── myfun.ts              # 多服务代理函数
│   ├── types.ts              # 通用类型
│   └── utils/
│       ├── index.ts          # 工具函数
│       └── is.ts             # 类型判断
├── package.json              # 依赖配置
├── tsup.config.ts            # 构建配置
└── .env                      # 环境变量（不提交）
```

### 构建产物
- `build/index.mjs`: 编译后的生产代码

---

## 安全建议

1. **环境变量**: 永远不要将 `.env` 文件提交到版本控制
2. **API Key**: 使用环境变量存储敏感密钥
3. **防爆破**: 启用 `AUTH_SECRET_ERROR_COUNT` 和 `AUTH_SECRET_ERROR_TIME`
4. **CORS**: 根据需要限制跨域访问
5. **限流**: 使用 `express-rate-limit` 防止 API 滥用
6. **HTTPS**: 生产环境使用 HTTPS
7. **Nginx**: 配置反向代理时正确设置 `X-Forwarded-For`

---

## 性能优化建议

1. **缓存**: 使用 Redis 缓存频繁访问的数据
2. **连接池**: 复用 HTTP 连接
3. **压缩**: 启用 gzip/brotli 压缩
4. **负载均衡**: 使用 PM2 或 Kubernetes 实现多实例
5. **日志**: 使用结构化日志（winston）

---

## 下一步改进建议

1. **测试覆盖**: 添加完整的单元测试和集成测试
2. **错误处理**: 统一错误处理中间件
3. **日志系统**: 使用 winston 或 pino
4. **监控**: 集成 Prometheus + Grafana
5. **API 文档**: 使用 Swagger/OpenAPI 自动生成文档
6. **类型安全**: 减少 `any` 使用
7. **数据库**: 如需持久化，集成 PostgreSQL/MongoDB

---

**返回**: [根目录文档](../CLAUDE.md)
