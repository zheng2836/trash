[ChatGPT_APK_逆向分析报告.md](https://github.com/user-attachments/files/27084301/ChatGPT_APK_.md)
# trash# ChatGPT Android APK 逆向分析报告

> **版本**: `1.2026.097`  
> **包名**: `com.openai.chatgpt`  
> **文件大小**: 48MB（6 个 DEX）  
> **最低 SDK**: Android 32 (Android 12)  
> **混淆**: R8/ProGuard（类名已混淆）

---

## 一、认证系统（鉴权流程）

### 1.1 认证架构

使用 **Auth0 Android SDK** + **OpenAI 自定义 Auth 服务器**，流程为标准 OAuth 2.0 + PKCE（Proof Key for Code Exchange），专为移动端设计（无 client_secret 暴露）。

```
┌──────────────┐     PKCE授权码流程      ┌──────────────────────────┐
│  ChatGPT App │ ───────────────────────> │ auth.openai.com          │
│              │ <─────────────────────── │ (基于Auth0构建)          │
│  存储Token:  │      access_token        └──────────────────────────┘
│  com.auth0.* │      refresh_token
└──────────────┘      id_token
```

### 1.2 Auth 端点详情

| 端点 | 方法 | 作用 |
|------|------|------|
| `https://auth.openai.com/api/accounts/authorize` | GET | 发起授权（浏览器打开） |
| `https://auth.openai.com/oauth/pre_token` | POST | 预认证 token（PKCE 第一步） |
| `https://auth.openai.com/oauth/token` | POST | 换取 access_token + refresh_token |
| `https://auth.openai.com/oauth/revoke` | POST | 吊销 token（注销） |
| `https://api.openai.com/auth` | - | Auth 相关辅助 |
| `https://api.openai.com/mfa` | - | MFA 多因素认证 |

### 1.3 OAuth 授权码请求参数

```http
GET https://auth.openai.com/api/accounts/authorize?
    response_type=code
    &client_id={CLIENT_ID}
    &redirect_uri=com.openai.chatgpt://auth.openai.com/android/com.openai.chatgpt/callback
    &scope=openid%20profile%20email%20offline_access
    &state={随机state}
    &nonce={随机nonce}
    &code_challenge={BASE64URL(SHA256(code_verifier))}
    &code_challenge_method=S256
    &audience={AUDIENCE}
    &connection={connection}
    &prompt=login
```

### 1.4 Token 换取请求

```http
POST https://auth.openai.com/oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "client_id": "{CLIENT_ID}",
  "code": "{authorization_code}",
  "redirect_uri": "com.openai.chatgpt://auth.openai.com/android/com.openai.chatgpt/callback",
  "code_verifier": "{原始code_verifier}"
}
```

**返回：**
```json
{
  "access_token": "eyJ...",
  "refresh_token": "...",
  "id_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "openid profile email offline_access"
}
```

### 1.5 Token 刷新请求

```http
POST https://auth.openai.com/oauth/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "client_id": "{CLIENT_ID}",
  "refresh_token": "{refresh_token}"
}
```

### 1.6 OAuth Redirect URI 方案（来自 AndroidManifest）

```
# 自定义 URI Scheme（用于 Auth0 SDK 回调）
com.openai.chatgpt://auth.openai.com/android/com.openai.chatgpt/callback
com.openai.chatgpt://auth0.openai.com/...
com.openai.chatgpt://auth0-dev.openai.com/...
com.openai.chatgpt://auth.api.openai.org/...

# 深链接回调
com.openai.chat://auth/ext_callback
com.openai.chat://auth/logout
com.openai.chat://auth/email_verification

# HTTPS 回调（App Links）
https://platform.openai.com/auth/ext_callback
https://platform.openai.com/auth/logout
```

### 1.7 Token 本地存储键（SharedPreferences）

```
com.auth0.access_token         # Bearer Token
com.auth0.refresh_token        # 刷新令牌
com.auth0.id_token             # OpenID JWT
com.auth0.expires_at           # 过期时间戳
com.auth0.cache_expires_at     # 缓存过期
com.auth0.scope                # 已授权范围
com.auth0.token_type           # 通常为 "Bearer"
```

### 1.8 Token 刷新策略

- 提前刷新：`early_token_refresh_days_v2`
- 接近过期刷新：`ACCESS_TOKEN_REFRESH_TRIGGER_TYPE_NEAR_EXPIRY`
- 手动刷新：`ACCESS_TOKEN_REFRESH_TRIGGER_TYPE_MANUAL`
- 禁止后台刷新：`disallow_token_refresh_in_background_v2`

---

## 二、文字对话 API

### 2.1 Base URL

```
https://android.chat.openai.com
```

### 2.2 核心对话端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/backend-api/alder/conversation` | POST | **主要对话 API**（SSE 流式） |
| `/backend-api/alder/conversation/prepare` | POST | 对话预处理（获取 turn_id 等） |
| `/backend-api/f/conversation` | POST | 备用对话端点（旧版） |
| `/backend-api/f/conversation/prepare` | POST | 备用预处理 |
| `/backend-anon/` | - | 匿名对话（无账号） |
| `/public-api/` | - | 公开 API |

### 2.3 对话请求结构（POST `/backend-api/alder/conversation`）

```http
POST https://android.chat.openai.com/backend-api/alder/conversation
Authorization: Bearer {access_token}
Content-Type: application/json
OAI-Client-Type: android
OAI-Device-Id: {device_uuid}
OAI-Package-Name: com.openai.chatgpt
User-Agent: ChatGPT/1.2026.097 (Android {version}; {device_info})
ChatGPT-Account-ID: {account_id}
ChatGPT-Residency-Region: {region}
x-oai-request-id: {request_uuid}
x-oai-convo-session-id: {session_id}
x-oai-turn-trace-id: {trace_id}
oai-android-play-integrity-token: {play_integrity_token}
```

### 2.4 请求 Body

```json
{
  "action": "next",
  "model": "gpt-4o",
  "messages": [
    {
      "id": "{message_uuid}",
      "author": {
        "role": "user"
      },
      "content": {
        "content_type": "text",
        "parts": ["用户输入的文字内容"]
      },
      "metadata": {},
      "status": "finished_successfully"
    }
  ],
  "conversation_id": "{conversation_uuid}",
  "parent_message_id": "{parent_uuid}",
  "stream": true,
  "variant": "...",
  "variants": [],
  "next": null,
  "suggestions": [],
  "timezone": "Asia/Shanghai",
  "system_hints": []
}
```

**关键字段说明：**

| 字段 | 说明 |
|------|------|
| `action` | `"next"`（新消息）/ `"variant"`（重新生成） |
| `model` | 见下方模型列表 |
| `conversation_id` | 不存在时由服务器新建 |
| `parent_message_id` | 上一条消息 ID，首次可用空 UUID |
| `stream` | `true` 时使用 SSE 流式响应 |

### 2.5 支持的模型 Slug

从 DEX 中提取到的模型标识符：

```
gpt-4o          # GPT-4o（默认）
gpt-5           # GPT-5
gpt-5-mini      # GPT-5 Mini
o1-pro          # o1 Pro
auto            # 自动选择
```

内部测试 slug（混淆/实验性）：
```
o01r, o0hl2, o0p08, o2p0, o33j, o3o3
o47i, o6o6, o7o7, o8o8, o9eq0o, o9n0, o9o9
```

### 2.6 SSE 流式响应格式

```
data: {"type": "delta", "content": {"type": "text", "text": "Hello"}}
data: {"type": "delta", "content": {"type": "text", "text": " world"}}
data: {"finish_reason": "stop", "message_id": "...", "turn_id": "..."}
data: [DONE]
```

关键响应字段：`delta`, `finish_reason`, `turn_id`, `turn_session_id`, `conversation_id`, `message_id`, `model_slug`

---

## 三、其他 API 端点

### 3.1 遥测 & 分析

```
POST https://android.chat.openai.com/ces/v1/telemetry/intake  # 遥测上报
POST https://android.chat.openai.com/ces/statsc/flush         # 统计刷新
POST https://ab.chatgpt.com/v1                                 # A/B 测试
```

### 3.2 实时语音（WebRTC/WebSocket）

```
WSS  https://android.chat.openai.com/realtime/  # 实时语音 WebSocket
WSS  https://realtime.chatgpt.com/v1            # Realtime API
```

### 3.3 Sora 视频

```
https://sora.chatgpt.com/backend/  # 视频生成 API
```

### 3.4 付费订阅

```
https://api-paywalls.revenuecat.com/        # RevenueCat 内购
https://api-production.8-lives-cat.io/      # RevenueCat 备用
https://api-diagnostics.revenuecat.com/     # RevenueCat 诊断
```

### 3.5 安全验证

```
POST Play Integrity Token → 请求头: oai-android-play-integrity-token
POST hCaptcha Token       → 人机验证
```

---

## 四、请求头完整列表

| Header | 值 / 格式 | 说明 |
|--------|-----------|------|
| `Authorization` | `Bearer {access_token}` | 认证 |
| `User-Agent` | `ChatGPT/1.2026.097 (Android ...)` | 客户端标识 |
| `OAI-Client-Type` | `android` | 平台类型 |
| `OAI-Device-Id` | UUID | 设备 ID |
| `OAI-Package-Name` | `com.openai.chatgpt` | 包名 |
| `Content-Type` | `application/json` | 请求体格式 |
| `ChatGPT-Account-ID` | `{account_id}` | 账户 ID |
| `ChatGPT-Residency-Region` | `{region}` | 数据驻留区域 |
| `Chatgpt-Project-ID` | `{project_id}` | 项目 ID（可选） |
| `x-oai-request-id` | UUID | 请求追踪 ID |
| `x-oai-convo-session-id` | UUID | 会话 ID |
| `x-oai-turn-trace-id` | UUID | Turn 追踪 ID |
| `x-oai-network-transport` | - | 网络传输类型 |
| `oai-android-play-integrity-token` | token | Google Play 完整性验证 |

---

## 五、应用架构关键组件

| 组件 | 类名 |
|------|------|
| 主 Activity | `com.openai.chatgpt.MainActivity` |
| 对话流服务 | `com.openai.feature.conversations.impl.coordinator.ConversationStreamingService` |
| Auth0 重定向 | `com.auth0.android.provider.RedirectActivity` |
| 实时语音服务 | `com.openai.voice.webrtc.VoiceModeForegroundService` |
| Assistant 代理 | `com.openai.feature.assistant.impl.AssistantProxyActivity` |
| 群组聊天（Calpico） | `com.openai.feature.calpico.impl.*` |
| 通知服务 | `com.openai.feature.notification.impl.NotificationService` |

---

## 六、注意事项

1. **Play Integrity**：每次对话请求都需附带 `oai-android-play-integrity-token`，该 token 由 Google Play Integrity API 生成，有效期短，是反爬核心机制。
2. **PKCE 无 Secret**：OAuth 流程使用 PKCE，无 `client_secret` 硬编码，安全性较高。
3. **代码混淆**：核心业务类已被 R8 混淆（如 `com.openai.chatgpt.app.NQ.LQaW`），完整还原需 jadx 反编译。
4. **SSE 协议**：对话响应使用 Server-Sent Events 流式推送，非标准 HTTP 轮询。
