# 前端模块文档

[根目录](../CLAUDE.md) > **src (前端模块)**

---

## 变更记录 (Changelog)

| 日期 | 版本 | 变更内容 |
|------|------|----------|
| 2025-11-21 | v1.1.0 | 增量更新：补充组件结构详情、工具函数库、消息渲染机制 |
| 2025-11-21 | v1.0.0 | 初始化前端模块文档 |

---

## 模块职责

前端模块负责：
- 提供用户界面和交互体验
- 管理应用状态（Pinia）
- 调用后端 API
- 处理路由导航
- 国际化支持（中英法韩俄越土）
- 主题切换（亮/暗模式）

---

## 入口与启动

### 主入口文件
`src/main.ts`:
```typescript
async function bootstrap() {
  const app = createApp(App)
  setupAssets()          // 加载全局样式
  setupScrollbarStyle()  // 自定义滚动条
  setupStore(app)        // 初始化 Pinia
  setupI18n(app)         // 初始化国际化
  await setupRouter(app) // 初始化路由
  app.mount('#app')
}
```

### 启动命令
- **开发模式**: `pnpm dev` (端口: 1002)
- **构建**: `pnpm build`
- **预览**: `pnpm preview`

---

## 对外接口

### API 层 (src/api/)

核心 API 模块：

| 文件 | 职责 | 主要导出 |
|------|------|----------|
| `chat.ts` | ChatGPT 对话 API | `fetchChatAPIProcess`, `fetchSession`, `fetchVerify` |
| `mjapi.ts` | Midjourney 绘图 API | `subTask`, `flechTask`, `mjFetch`, `getSeed` |
| `suno.ts` | Suno 音乐生成 API | `sunoFetch`, 音乐生成相关函数 |
| `luma.ts` | Luma 视频生成 API | `lumaFetch`, 视频任务管理 |
| `kling.ts` | Kling 视频 API | - |
| `runway.ts` | Runway 视频 API | - |
| `viggle.ts` | Viggle 舞蹈 API | - |
| `ideo.ts` | Ideogram 绘图 API | - |
| `realtime.ts` | OpenAI Realtime API | 实时语音对话 |
| `openapi.ts` | OpenAI 通用 API | 图片编辑、TTS、Whisper 等 |

### API 调用模式

**示例：ChatGPT 对话**
```typescript
import { fetchChatAPIProcess } from '@/api'

await fetchChatAPIProcess({
  prompt: '你好',
  options: { conversationId, parentMessageId },
  onDownloadProgress: (event) => {
    // 处理流式响应
  }
})
```

**示例：Midjourney 绘图**
```typescript
import { subTask } from '@/api'

const chat = { /* Chat 对象 */ }
await subTask({
  drawText: 'a beautiful sunset',
  fileBase64: [],
  bot: 'MID_JOURNEY'
}, chat)
```

---

## 关键依赖与配置

### 核心依赖
- `vue@^3.2.47`: 核心框架
- `vue-router@^4.1.6`: 路由管理
- `pinia@^2.0.33`: 状态管理
- `naive-ui@^2.34.3`: UI 组件库
- `vite@^4.2.0`: 构建工具
- `axios`: HTTP 客户端
- `localforage`: 本地存储（IndexedDB）

### 配置文件
- **Vite**: `vite.config.ts`
- **TypeScript**: `tsconfig.json`
- **Tailwind**: `tailwind.config.js`
- **ESLint**: `.eslintrc.cjs`

### Vite 配置亮点
```typescript
// 路径别名
resolve: {
  alias: {
    '@': path.resolve(process.cwd(), 'src'),
  }
}

// 开发代理
proxy: {
  '/api': { target: viteEnv.VITE_APP_API_BASE_URL },
  '/mjapi': { target: viteEnv.VITE_APP_API_BASE_URL },
  '/openapi': { target: viteEnv.VITE_APP_API_BASE_URL },
  // ...
}
```

---

## 数据模型

### 状态管理架构 (Pinia)

```
src/store/
├── index.ts              # Store 导出
├── homeStore.ts          # 全局状态（会话、模块切换）
└── modules/
    ├── app/             # 应用配置（主题、语言）
    ├── auth/            # 认证状态
    ├── chat/            # 对话历史
    ├── prompt/          # 提示词管理
    ├── settings/        # 用户设置
    └── user/            # 用户信息
```

### 核心 Store

**chatStore** (`src/store/modules/chat/index.ts`):
- 管理对话历史 (`history[]`)
- 管理对话内容 (`chat[]`)
- 当前激活会话 (`active`)
- 上下文模式 (`usingContext`)

**authStore** (`src/store/modules/auth/index.ts`):
- 认证状态
- Token 管理

**homeStore** (`src/store/homeStore.ts`):
- 当前激活的模块（chat/draw/music/video/dance）
- 会话配置
- 任务更新通知

### 类型定义

位于 `src/typings/`:
- `chat.d.ts`: 对话相关类型
- `global.d.ts`: 全局类型
- `env.d.ts`: 环境变量类型

---

## 路由结构

`src/router/index.ts`:

| 路径 | 组件 | 功能 |
|------|------|------|
| `/chat/:uuid?` | `chat/index.vue` | ChatGPT 对话 |
| `/g/:gid` | `chat/index.vue` | GPTs (ChatGPT 插件) |
| `/m/:gid` | `chat/index.vue` | 模型切换 |
| `/s/t` | `chat/index.vue` | 设置页面 |
| `/draw/:uuid?` | `mj/draw.vue` | Midjourney 绘图 |
| `/music/:uuid?` | `suno/music.vue` | Suno 音乐 |
| `/video/:uuid?` | `luma/video.vue` | 视频生成 |
| `/dance/:uuid?` | `viggle/dance.vue` | Viggle 舞蹈 |
| `/wav/:uuid?` | `wav/wav.vue` | OpenAI Realtime |

---

## 视图层结构

### 主要视图模块

**1. Chat 模块** (`src/views/chat/`)
- **职责**: ChatGPT 对话、GPTs、模型切换
- **核心组件**:
  - `index.vue`: 主对话界面
  - `components/Message/`: 消息展示组件
  - `components/Header/`: 顶部工具栏
  - `layout/`: 布局与侧边栏

**2. MJ 模块** (`src/views/mj/`)
- **职责**: Midjourney 绘图、图片编辑、混图、人脸替换
- **核心组件**:
  - `draw.vue`: 绘图主界面
  - `aiDrawInput.vue`: 绘图输入
  - `drawList.vue`: 绘图列表
  - `aiCanvas.vue`: 画布编辑
  - `aiFace.vue`: 人脸替换
  - `aiBlend.vue`: 混图

**3. Suno 模块** (`src/views/suno/`)
- **职责**: AI 音乐生成
- **核心组件**:
  - `music.vue`: 音乐主界面
  - `mcInput.vue`: 音乐输入
  - `mcList.vue`: 音乐列表
  - `player.vue`: 音乐播放器

**4. Luma 模块** (`src/views/luma/`)
- **职责**: AI 视频生成（支持多种服务）
- **核心组件**:
  - `video.vue`: 视频主界面
  - `lumaInput.vue`: Luma 输入
  - `pikaInput.vue`: Pika 输入
  - `runwayInput.vue`: Runway 输入
  - `runmlInput.vue`: Runway ML 输入
  - `pixInput.vue`: PixVerse 输入

**5. Viggle 模块** (`src/views/viggle/`)
- **职责**: AI 舞蹈生成
- **核心组件**:
  - `dance.vue`: 舞蹈主界面
  - `dcInput.vue`: 舞蹈输入
  - `dcList.vue`: 舞蹈列表

**6. WAV 模块** (`src/views/wav/`)
- **职责**: OpenAI Realtime API 实时语音对话
- **核心组件**:
  - `wav.vue`: 实时语音界面
  - `an_main.vue`: 主控制器

---

## Chat 组件详解 (增量更新)

### 组件层级结构

```
src/views/chat/
├── index.vue                    # 主对话页面
├── components/
│   ├── index.ts                 # 组件导出
│   ├── Header/
│   │   └── index.vue           # 顶部工具栏
│   └── Message/
│       ├── index.vue           # 消息容器组件
│       ├── Avatar.vue          # 头像组件
│       └── Text.vue            # 文本渲染组件
└── hooks/
    ├── useChat.ts              # 聊天操作 hooks
    ├── useScroll.ts            # 滚动控制 hooks
    └── useUsingContext.ts      # 上下文控制 hooks
```

### Message 组件 (`components/Message/index.vue`)

**职责**: 渲染单条消息，包括用户输入和 AI 响应

**关键功能**:
- 消息复制（`copyToClip`）
- 消息重新生成（`handleRegenerate`）
- TTS 语音朗读
- 切换原始文本/Markdown 预览
- 消息编辑/删除

**Props 接口**:
```typescript
interface Props {
  dateTime?: string
  text?: string
  inversion?: boolean  // 是否为用户消息
  error?: boolean
  loading?: boolean
  chat: Chat.Chat
  index: number
}
```

**事件发射**:
- `regenerate`: 重新生成回复
- `delete`: 删除消息
- `edit`: 编辑消息

### Text 组件 (`components/Message/Text.vue`)

**职责**: 渲染消息文本内容，支持多种格式

**核心功能**:
- Markdown 渲染（使用 `markdown-it`）
- 代码高亮（使用 `highlight.js`）
- LaTeX 公式支持（使用 `markdown-it-katex`）
- 思考过程处理（`<think>` 标签转换为引用格式）

**条件渲染**:
- Midjourney 图片 (`mjText.vue`)
- DALL-E 图片 (`dallText.vue`)
- TTS 音频 (`ttsText.vue`)
- Whisper 转录 (`whisperText.vue`)
- API Key 错误提示 (`aiTextSetting.vue`)
- 认证错误提示 (`aiSetAuth.vue`)

### Header 组件 (`components/Header/index.vue`)

**职责**: 对话页面顶部工具栏

**功能**:
- 显示当前对话标题
- 导出对话
- 清空对话
- 模型切换弹窗（`aiModel.vue`）
- 侧边栏折叠控制

---

## MJ 模块详解 (增量更新)

### 组件层级结构

```
src/views/mj/
├── index.ts                     # 组件导出
├── draw.vue                     # 绘图主页面
├── layout.vue                   # 布局组件
├── drawList.vue                 # 绘图任务处理逻辑
├── aiSider.vue                  # 侧边栏导航
├── aiSiderInput.vue             # 侧边栏输入区域
├── aiDrawInput.vue              # 绘图输入主组件（Tab 容器）
├── aiDrawInputItem.vue          # MJ 绘图输入表单
├── aiDall.vue                   # DALL-E 输入
├── aiIdeoInput.vue              # Ideogram 输入
├── aiFace.vue                   # 换脸功能
├── aiBlend.vue                  # 混图功能
├── aiCanvas.vue                 # 画布编辑
├── aiEditImage.vue              # 图片编辑
├── aiEditVidoe.vue              # 视频编辑
├── aiFooter.vue                 # 底部组件
├── aiGallery.vue                # 图库抽屉
├── aiGalleryItem.vue            # 图库项
├── aiGpts.vue                   # GPTs 列表
├── aiGptsAdd.vue                # 添加 GPTs
├── aiGptsCom.vue                # GPTs 组件
├── aiGptInput.vue               # GPT 输入
├── aiListText.vue               # 列表文本
├── aiMic.vue                    # 麦克风
├── aiMobileMenu.vue             # 移动端菜单
├── aiModel.vue                  # 模型选择器
├── aiModelServer.vue            # 服务端模型
├── aiMsg.vue                    # 消息组件
├── aiOther.vue                  # 其他功能
├── aiSetAuth.vue                # 认证设置
├── aiSetServer.vue              # 服务器设置
├── aiTextSetting.vue            # 文本设置
├── mjText.vue                   # MJ 文本渲染
├── mjTextAttr.vue               # MJ 属性显示
├── dallText.vue                 # DALL-E 文本
├── ttsText.vue                  # TTS 文本
├── whisperText.vue              # Whisper 文本
└── myTest.vue                   # 测试组件
```

### aiDrawInput.vue（绘图输入主组件）

**职责**: 绘图功能的 Tab 容器，整合多种绘图服务

**Tab 结构**:
1. **MidJourney** - 包含子 Tab:
   - 绘图 (`aiDrawInputItem.vue`)
   - 换脸 (`aiFace.vue`)
   - 混图 (`aiBlend.vue`)
2. **Dall.E** (`aiDall.vue`)
3. **IdeoGram** (`aiIdeoInput.vue`)

**事件**:
- `drawSent`: 发送绘图任务
- `close`: 关闭面板

### aiSider.vue（侧边栏导航）

**职责**: 左侧功能导航栏

**导航项**:
- Chat（对话）
- Draw（绘图）
- Music（音乐）
- Video（视频）
- GPTs（应用商店）
- Gallery（图库）
- Dance（舞蹈）
- Realtime（实时语音）
- Settings（设置）

**特性**:
- 支持菜单禁用 (`isDisableMenu`)
- 响应式布局（移动端隐藏）
- 头像显示
- 百度/Google 统计集成

### drawList.vue（绘图任务处理）

**职责**: 处理绘图任务提交和状态更新

**核心功能**:
- 提交各类绘图任务（MJ/DALL-E/换脸/混图）
- 监听 `homeStore.myData.act` 处理任务动作
- 本地存储图片（`localSaveAny`）
- 任务状态轮询和更新

**任务类型**:
- `face`: 换脸
- `blend`: 混图
- `gpt.dall-e-3`: DALL-E 绘图
- `shorten`: 提示词精简
- `mj.edit.video`: 视频编辑
- `mj.edit.image`: 图片编辑
- 普通 MJ 绘图

### aiModel.vue（模型选择器）

**职责**: AI 模型配置面板

**配置项**:
- 模型选择（支持 100+ 模型）
- 自定义模型
- 历史对话数
- 最大 Token 数
- 系统提示词
- 高级参数（temperature, top_p, presence_penalty, frequency_penalty）
- TTS 语音选择

**支持的模型系列**:
- OpenAI: GPT-3.5, GPT-4, GPT-4o, o1, o1-mini
- Google: Gemini Pro
- Anthropic: Claude 3 系列
- DeepSeek: R1, V3
- xAI: Grok 3 系列

---

## 工具函数库 (增量更新)

### src/utils/ 目录结构

```
src/utils/
├── copy.ts              # 剪贴板操作
├── wav_renderer.ts      # 音频波形渲染
├── functions/
│   ├── index.ts         # 日期工具函数
│   └── debounce.ts      # 防抖函数
├── is/
│   └── index.ts         # 类型判断函数
├── request/
│   ├── index.ts         # HTTP 请求封装
│   └── axios.ts         # Axios 实例配置
└── storage/
    └── index.ts         # 本地存储封装
```

### HTTP 请求 (`request/index.ts`)

**接口定义**:
```typescript
interface HttpOption {
  url: string
  data?: any
  method?: string
  headers?: any
  onDownloadProgress?: (progressEvent: AxiosProgressEvent) => void
  signal?: GenericAbortSignal
  beforeRequest?: () => void
  afterRequest?: () => void
}

interface Response<T = any> {
  data: T
  message: string | null
  status: string
}
```

**主要导出**:
- `get<T>()`: GET 请求
- `post<T>()`: POST 请求
- 默认导出 `post`

**特性**:
- 自动处理认证失败（刷新页面）
- 支持流式下载进度
- 请求前后钩子

### 本地存储 (`storage/index.ts`)

**功能**: 带过期时间的 localStorage 封装

**默认过期时间**: 7 天

**接口**:
```typescript
const storage = createLocalStorage({ expire?: number | null })

storage.set(key, data)   // 存储数据
storage.get(key)         // 获取数据（自动检查过期）
storage.remove(key)      // 删除数据
storage.clear()          // 清空所有
```

**预定义实例**:
- `ls`: 7 天过期
- `ss`: 永不过期

### 类型判断 (`is/index.ts`)

提供完整的类型守卫函数：

```typescript
isNumber(value)      // 数字
isString(value)      // 字符串
isBoolean(value)     // 布尔
isNull(value)        // null
isUndefined(value)   // undefined
isObject(value)      // 对象
isArray(value)       // 数组
isFunction(value)    // 函数
isDate(value)        // 日期
isRegExp(value)      // 正则
isPromise(value)     // Promise
isSet(value)         // Set
isMap(value)         // Map
isFile(value)        // File
```

### 剪贴板操作 (`copy.ts`)

```typescript
function copyToClip(text: string): Promise<string>
```

使用传统的 `execCommand('copy')` 方式复制文本。

### 日期工具 (`functions/index.ts`)

```typescript
function getCurrentDate(): string  // 返回 "YYYY-M-D" 格式
```

---

## 测试与质量

### 当前状态
- **单元测试**: 无
- **集成测试**: 无
- **代码覆盖率**: 未测量

### 代码质量工具
- **ESLint**: 使用 `@antfu/eslint-config`
- **TypeScript**: 启用严格模式
- **Husky**: Git hooks
- **Lint-staged**: 提交前自动 lint

### 推荐测试策略
1. **组件测试**: 使用 Vitest + Vue Test Utils
2. **Store 测试**: 测试 Pinia actions 和 getters
3. **API Mock**: 使用 MSW (Mock Service Worker)

---

## 常见问题 (FAQ)

### Q1: 如何添加新的 AI 服务？
1. 在 `src/api/` 创建新的 API 文件
2. 在 `src/views/` 创建对应的视图组件
3. 在 `src/router/index.ts` 添加路由
4. 在后端 `service/src/index.ts` 添加代理路由

### Q2: 如何修改主题颜色？
- 全局主题配置在 `src/store/modules/app/`
- Tailwind 配置在 `tailwind.config.js`
- Naive UI 主题可通过 `NaiveProvider` 组件配置

### Q3: 图片如何存储？
- 本地开发: 使用 `localforage` (IndexedDB)
- 生产环境:
  - Cloudflare R2 (通过 `R2_*` 环境变量)
  - 服务器本地上传目录 (`/uploads`)
  - 自定义文件服务 (`FILE_SERVER`)

### Q4: 如何调试 API 调用？
1. 在 localStorage 设置 `debug=1`
2. 打开浏览器控制台
3. 查看 `[mjgpt]` 前缀的日志输出

### Q5: 如何切换语言？
- 通过 `src/store/modules/app/` 修改 `language` 状态
- 支持的语言: `zh-CN`, `zh-TW`, `en-US`, `fr-FR`, `ko-KR`, `ru-RU`, `tr-TR`, `vi-VN`

### Q6: 如何处理思考过程（Thinking）?
- Text.vue 组件会自动将 `<think>...</think>` 标签转换为 Markdown 引用格式
- 显示为 "Thinking..." 前缀的引用块

---

## 相关文件清单

### 核心文件
```
src/
├── main.ts                    # 应用入口
├── App.vue                    # 根组件
├── api/                       # API 层
│   ├── index.ts              # API 导出
│   ├── chat.ts               # ChatGPT API
│   ├── mjapi.ts              # Midjourney API
│   ├── suno.ts               # Suno API
│   ├── luma.ts               # Luma API
│   └── ...
├── store/                     # Pinia 状态管理
│   ├── index.ts
│   ├── homeStore.ts
│   └── modules/
├── router/                    # Vue Router
│   ├── index.ts
│   └── permission.ts
├── views/                     # 视图组件
│   ├── chat/
│   ├── mj/
│   ├── suno/
│   ├── luma/
│   ├── viggle/
│   └── wav/
├── components/                # 公共组件
│   ├── common/
│   └── custom/
├── utils/                     # 工具函数
│   ├── request/              # HTTP 请求
│   ├── storage/              # 本地存储
│   └── functions/            # 通用函数
├── locales/                   # 国际化
│   ├── zh-CN.ts
│   ├── en-US.ts
│   └── ...
└── typings/                   # 类型定义
    ├── chat.d.ts
    ├── global.d.ts
    └── env.d.ts
```

### 配置文件
- `vite.config.ts`: Vite 构建配置
- `tsconfig.json`: TypeScript 配置
- `tailwind.config.js`: Tailwind CSS 配置
- `.eslintrc.cjs`: ESLint 规则
- `postcss.config.js`: PostCSS 配置

---

## 下一步改进建议

1. **测试覆盖**: 添加 Vitest 单元测试
2. **类型安全**: 减少 `any` 使用，完善接口定义
3. **性能优化**:
   - 使用虚拟滚动优化长列表
   - 懒加载大型组件
   - 优化图片加载策略
4. **无障碍**: 添加 ARIA 标签，提升可访问性
5. **错误边界**: 使用 Vue 3 错误处理机制

---

**返回**: [根目录文档](../CLAUDE.md)
