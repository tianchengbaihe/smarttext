# HTTP API 参考

> 版本: v1.0.0  
> Base URL: `http://localhost:8088`

Smart Text 服务提供 RESTful HTTP API，用于文档管理和函数执行。

---

## 1. 通用约定

### 1.1 请求格式

- **Content-Type**: `application/json`
- **编码**: UTF-8
- **路径参数**: URL 编码

### 1.2 响应格式

所有响应均为 JSON，结构统一：

**成功响应**:
```json
{
  "data": { ... }
}
```

**错误响应**:
```json
{
  "error": {
    "code": "error_code",
    "message": "错误描述",
    "details": { ... }
  }
}
```

### 1.3 HTTP 状态码

| 状态码 | 含义 |
|--------|------|
| `200 OK` | 请求成功 |
| `201 Created` | 创建成功 |
| `400 Bad Request` | 请求参数错误 |
| `404 Not Found` | 资源不存在 |
| `409 Conflict` | 资源冲突（如重复） |
| `500 Internal Server Error` | 服务器内部错误 |

---

## 2. API 端点列表

### 2.1 健康检查

```
GET /api/v1/health
```

检查服务健康状态。

**响应示例**:
```json
{
  "data": {
    "status": "ok",
    "documentCount": 42,
    "time": "2026-03-15T10:30:00Z",
    "features": {
      "sse": true
    }
  }
}
```

---

### 2.2 列出所有文档

```
GET /api/v1/documents
```

获取所有文档的摘要列表。

**查询参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `namespace` | string | 否 | 按命名空间过滤 |
| `id` | string | 否 | 按文档 ID 过滤 |
| `q` | string | 否 | 关键字搜索 |
| `status` | string | 否 | 按状态过滤 (draft/published) |
| `page` | integer | 否 | 页码（从 1 开始） |
| `pageSize` | integer | 否 | 每页数量 |

**响应示例**:
```json
{
  "data": [
    {
      "namespace": "health",
      "id": "hypertension-visit-assistant",
      "text": "高血压门诊随访助手",
      "description": "使用血压阈值库生成分层结论",
      "latestVersion": "1.0.0",
      "latestStatus": "published",
      "latestPublishedVersion": "1.0.0",
      "versions": ["1.0.0", "0.9.0"],
      "versionSummaries": [
        {
          "uri": "smart://health/hypertension-visit-assistant@1.0.0",
          "namespace": "health",
          "id": "hypertension-visit-assistant",
          "version": "1.0.0",
          "text": "高血压门诊随访助手",
          "description": "使用血压阈值库生成分层结论",
          "status": "published",
          "sourceFormat": "yaml",
          "readFormat": "json",
          "importCount": 1,
          "functionCount": 2
        }
      ]
    }
  ]
}
```

**分页响应**（当提供 page/pageSize 时）:
```json
{
  "data": {
    "items": [ ... ],
    "total": 100,
    "page": 1,
    "pageSize": 10,
    "totalPages": 10
  }
}
```

---

### 2.3 列出所有命名空间

```
GET /api/v1/namespaces
```

获取所有命名空间列表。

**响应示例**:
```json
{
  "data": ["common", "health", "medical"]
}
```

---

### 2.4 获取文档详情

```
GET /api/v1/documents/{namespace}/{id}/{version}
```

获取文档的完整信息。

**路径参数**:

| 参数 | 说明 |
|------|------|
| `namespace` | 命名空间 |
| `id` | 文档 ID |
| `version` | 版本号（或使用 `latest`） |

**响应示例**:
```json
{
  "data": {
    "uri": "smart://health/hypertension-visit-assistant@1.0.0",
    "namespace": "health",
    "id": "hypertension-visit-assistant",
    "version": "1.0.0",
    "text": "高血压门诊随访助手",
    "description": "使用血压阈值库生成分层结论",
    "status": "published",
    "as": "bp-assistant",
    "imports": [
      { "uri": "smart://health/hypertension-thresholds@1.1.0" }
    ],
    "references": [
      {
        "uri": "smart://health/hypertension-thresholds@1.1.0",
        "namespace": "health",
        "id": "hypertension-thresholds",
        "version": "1.1.0"
      }
    ],
    "history": [
      {
        "uri": "smart://health/hypertension-visit-assistant@1.0.0",
        "version": "1.0.0",
        "status": "published",
        "isCurrent": true
      }
    ],
    "sourceFormat": "yaml",
    "rawSource": "原始 YAML 内容",
    "mergedDocument": {
      "id": "hypertension-visit-assistant",
      "version": "1.0.0",
      "read": {
        "format": "json",
        "body": { ... }
      },
      "execute": {
        "functions": [ ... ]
      }
    },
    "read": {
      "format": "json",
      "body": { ... }
    },
    "resolvedFormat": "json",
    "resolvedRead": { ... },
    "functionNames": ["classify_bp", "get_followup_days"],
    "functions": [ ... ]
  }
}
```

---

### 2.5 读取文档内容

```
GET /api/v1/documents/{namespace}/{id}/{version}/read
```

获取文档的 `read` 块内容（解析 import 后的合并结果）。

**响应示例**:
```json
{
  "data": {
    "uri": "smart://health/hypertension-visit-assistant@1.0.0",
    "format": "json",
    "content": {
      "service_name": "高血压门诊随访助手",
      "education_focus": ["..."]
    }
  }
}
```

---

### 2.6 执行函数

```
POST /api/v1/documents/{namespace}/{id}/{version}/execute/{function}
```

执行文档中的指定函数。

**路径参数**:

| 参数 | 说明 |
|------|------|
| `namespace` | 命名空间 |
| `id` | 文档 ID |
| `version` | 版本号 |
| `function` | 函数名 |

**请求头**:

| 头部 | 说明 |
|------|------|
| `Accept` | `application/json`（默认）或 `text/event-stream`（流式） |

**请求体**:
函数输入参数（JSON 对象）。

```json
{
  "systolic": 160,
  "diastolic": 100
}
```

#### 普通响应（Accept: application/json）

**响应示例**:
```json
{
  "data": {
    "uri": "smart://health/hypertension-visit-assistant@1.0.0",
    "function": "classify_bp",
    "result": "高血压2级",
    "durationMs": 15
  }
}
```

#### 流式响应（Accept: text/event-stream）

用于 LLM 等需要流式输出的场景。

**响应头**:
```http
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

**SSE 事件格式**:

```
data: {"type": "connected"}

data: {"type": "chunk", "content": "部分内容..."}

data: {"type": "reasoning", "content": "推理过程..."}

data: {"type": "complete", "content": "完整内容", "durationMs": 2500}
```

**事件类型**:

| 类型 | 说明 |
|------|------|
| `connected` | 连接成功 |
| `chunk` | 内容块更新 |
| `reasoning` | 推理过程（仅 LLM） |
| `complete` | 执行完成 |
| `error` | 执行错误 |

---

### 2.7 获取开发者手册

```
GET /api/v1/docs/developer-guide
```

获取内置的开发者指南（Markdown 格式）。

**响应**: Markdown 文本

---

## 3. 数据类型定义

### DocumentSummary

```typescript
{
  namespace: string;
  id: string;
  text?: string;
  description?: string;
  latestVersion?: string;
  latestStatus?: string;
  latestPublishedVersion?: string;
  versions: string[];
  versionSummaries: VersionSummary[];
}
```

### VersionSummary

```typescript
{
  uri: string;
  namespace: string;
  id: string;
  version: string;
  text?: string;
  description?: string;
  status?: string;
  sourceFormat: string;
  readFormat: string;
  importCount: number;
  functionCount: number;
  isCurrent?: boolean;
}
```

### FunctionDescriptor

```typescript
{
  name: string;
  text?: string;
  description?: string;
  params: FunctionParam[];
  returns?: FunctionReturn;
  type: string;
  runtime?: string;
  origin: "local" | "imported";
  // ... 类型特定字段
}
```

---

## 4. 错误码参考

| 错误码 | HTTP 状态 | 说明 |
|--------|-----------|------|
| `bad_request` | 400 | 请求参数错误 |
| `invalid_json` | 400 | JSON 解析失败 |
| `not_found` | 404 | 文档或函数不存在 |
| `duplicate` | 409 | 文档已存在（内容相同） |
| `version_exists` | 409 | 版本号已存在（内容不同） |
| `execute_failed` | 400 | 函数执行失败 |

---

## 5. 相关文档

- [技术白皮书](./whitepaper.md)
- [协议规范](./protocol.md)
- [MCP 接口参考](./mcp-reference.md)
