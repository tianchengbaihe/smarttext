# Smart Text 协议规范 v1.0.0

---

**版本**: 1.0.0 | **协议日期**: 2026-03-15 | **许可证**: MIT

---

![Smart Text Document Structure](docs/smart-text-structure.png)

![Smart Text Directory Layout](docs/smart-text-directory.png)

---

## 目录

1. [概述](#1-概述)
2. [快速上手](#2-快速上手)
   - 2.1 [示例：高血压治疗指南](#21-示例高血压治疗指南)
   - 2.2 [示例：健康风险评估助手](#22-示例健康风险评估助手)
   - 2.3 [目录布局方式](#23-目录布局方式)
3. [协议速览](#3-协议速览)
4. [核心概念](#4-核心概念)
5. [函数类型详解](#5-函数类型详解)
6. [HTTP API](#6-http-api)
7. [MCP 接口](#7-mcp-接口)
8. [详细规范](#8-详细规范)
   - 8.4 [文件存储（单文件）](#84-文件存储单文件)
   - 8.5 [文件存储（目录布局）](#85-文件存储目录布局)

---

## 1. 概述

### 1.1 什么是 Smart Text

**Smart Text** 是一种 AI 时代下整合传统计算能力和 AI 智能能力的以 YAML 为入口的文本文件格式：

| 内容 | 对应块 | 说明 |
|------|--------|------|
| **文档身份** | 顶层字段 | uri 统一标识，支持引用互连 |
| **静态知识** | `contents` | 可读取的知识内容 |
| **可执行能力** | `services` | 可调用的服务声明 |
| **引用能力** | `import` | 可引用的内容，支持片段级引用 |
| **元数据** | `meta` | 描述信息、标签、扩展字段 |

### 1.2 一句话定义

> Smart Text = 版本化的知识内容 + 可执行的知识服务 + 可引用的知识网络

---

## 2. 快速上手

### 2.1 示例：高血压诊疗指南

`contents` 中存放 Markdown 格式的医学知识，`services` 提供智能问答能力：

```yaml
# smart://health/htn-guide@1.0.0
namespace: health
id: htn-guide
version: "1.0.0"

meta:
  title: "高血压诊疗指南"
  tags: [高血压, 心血管疾病, 诊疗指南]
  author: "医疗专家组"

contents:
  format: text/markdown
  body: |
    # 高血压分类与处理
    
    ## 正常血压 {#normal}
    - 收缩压 < 120 mmHg 且舒张压 < 80 mmHg
    - 建议：保持健康生活方式
    
    ## 1级高血压 {#stage1}
    - 收缩压 140-159 mmHg 或舒张压 90-99 mmHg
    - 建议：生活方式干预 + 考虑药物治疗
    
    ## 2级高血压 {#stage2}
    - 收缩压 ≥ 160 mmHg 或舒张压 ≥ 100 mmHg
    - 建议：立即启动药物治疗
    
    ## 高血压危象 {#critical}
    - 收缩压 ≥ 180 mmHg 或舒张压 ≥ 120 mmHg
    - 建议：立即就医，评估靶器官损害

services:
  - name: query
    text: "指南查询"
    type: tools/llm
    system: 你是一位全科医生，基于指南回答问题。
    expr: |
      参考指南：
      {{$contents.raw()}}
      
      患者问题：{{input.question}}
      
      请基于上述指南给出专业建议。
```

**调用：**

```bash
curl -X POST http://localhost:8088/api/v1/documents/health/htn-guide/1.0.0/services/query \
  -H "Content-Type: application/json" \
  -d '{"question": "血压 150/95 需要吃药吗？"}'
```

### 2.2 示例：健康风险评估助手

展示 services 的组合编排能力（code/js + flow/switch + tools/llm），通过 import 复用 2.1 的高血压指南：

```yaml
# smart://health/risk-assistant@1.0.0
namespace: health
id: risk-assistant
version: "1.0.0"

meta:
  title: "健康风险评估助手"
  author: "张三"
  tags: [风险评估, 血压]

import:
  # 引入完整指南
  - smart://health/htn-guide@1.0.0
  # 引入特定片段（必须用引号包裹，避免 # 被识别为注释）
  - "smart://health/htn-guide@1.0.0#stage1"
  - "smart://health/htn-guide@1.0.0#stage2"
  - "smart://health/htn-guide@1.0.0#critical"

contents:
  format: application/yaml
  body: |
    healthy_tips: "保持健康生活方式，定期监测血压"

services:
  # 1. 参数校验与标准化（code/js）
  - name: normalize_input
    type: code/js
    expr: |
      const sys = $utils.to_number(input.systolic);
      const dia = $utils.to_number(input.diastolic);
      if (isNaN(sys) || isNaN(dia)) {
        throw new Error("血压值格式错误");
      }
      return { systolic: sys, diastolic: dia, age: input.age };

  # 2. 分级评估路由（flow/switch）
  - name: classify_risk
    type: flow/switch
    cases:
      - when:
          type: code/cel
          expr: input.systolic >= 180 || input.diastolic >= 120
        action:
          type: code/cel                      # 内联返回，无需定义 service
          expr: |
            {
              level: "critical",
              message: "血压危急，请立即就医！",
              action: "拨打急救电话或前往急诊"
            }
      - when:
          type: code/cel
          expr: input.systolic >= 140 || input.diastolic >= 90
        action:
          ref: high_risk                      # 引用其他 service
    default:
      type: code/template
      expr: |
        您的血压 {{input.systolic}}/{{input.diastolic}} mmHg 处于正常范围。
        建议：{{$contents.yaml("healthy_tips")}}

  # 3. 高风险：基于指南的 AI 建议（tools/llm）
  - name: high_risk
    type: tools/llm
    system: 你是专业医生，基于高血压指南给出建议。
    expr: |
      参考指南：
      {{$import.htn_guide.contents.raw()}}
      
      患者血压：{{input.systolic}}/{{input.diastolic}} mmHg
      请根据指南给出风险评估和建议。

  # 4. 完整评估流程（flow/sequence）
  - name: assess
    type: flow/sequence
    steps:
      - normalize_input
      - classify_risk
    merge: last
```

**调用：**

```bash
# 单接口完成：校验 → 分级 → 智能建议
curl -X POST http://localhost:8088/api/v1/documents/health/risk-assistant/1.0.0/services/assess \
  -H "Content-Type: application/json" \
  -d '{"systolic": 155, "diastolic": 98}'
```

### 2.3 目录布局方式

除了单一 YAML 文件，也可以使用目录结构组织文档：

```
health/risk_assistant/1.0.0/
├── main.yaml          # 可为空或省略
├── meta.yaml          # 可选元数据
├── contents.md        # 或 .json / .yaml
├── services/
│   ├── normalize_input.js
│   └── high_risk.yaml
└── docs/              # 本地依赖（自动导入）
    ├── thresholds.yaml
    ├── prompts.md
    └── helpers.js
```

**规则：**
- 命名用 `_` 不用 `.`
- `contents.*` 只能有一个
- `docs/` 下文件作为本地依赖自动导入

---

## 3. 协议速览

### 3.1 核心结构

| 块 | 用途 |
|----|------|
| **身份** | `id`, `version`, `namespace` 唯一标识 |
| **元数据** | `meta` - 标题、标签、作者、扩展字段 |
| **知识** | `contents` - Text、Markdown、YAML、JSON |
| **能力** | `services` - 可调用的服务列表 |
| **引用** | `import` - 复用其他文档、引用片段 |

### 3.2 内容格式

参考 MIME 声明不同格式的内容

| format 值 | 说明 | body 类型 |
|-----------|------|-----------|
| `text/plain` | 纯文本 | string |
| `text/markdown` | Markdown 富文本 | string |
| `application/json` | JSON 数据 | object/array |
| `application/yaml` | YAML 数据 | object/array |

### 3.3 服务类型

通过三种服务实现逻辑（code）、编排（flow）、接入服务（tools）全场景需求，每种服务可扩展。

| 类型 | 语法 | 说明 |
|------|--------|------|
| `code/js` | ES5.1 | JavaScript 代码执行 |
| `code/cel` | CEL | 表达式计算 |
| `code/template` | Mustache | 模板渲染 |
| `flow/if` / `flow/switch` / `flow/sequence` / `flow/loop` | 自定义 | 流程控制 |
| `tools/http` | 自定义 | HTTP 外部调用 |
| `tools/llm` | 自定义 | 大模型调用 |

### 3.4 扩展

`extensions/` 目录下的 JS 等脚本自动注册到 `$` 命名空间。

---

## 4. 核心概念

### 4.1 Document（文档）

```yaml
id: example-doc           # 文档标识
version: "1.0.0"          # 语义化版本
namespace: default        # 命名空间

meta:
  title: "示例文档"       # 展示名称
  description: "描述"     # 文档描述
  tags: [tag1, tag2]      # 标签列表
  author: "作者"
  created_at: "2026-03-15"
  x-custom-field: "扩展值" # 应用层自定义字段
```

**URI 格式**: `smart://<namespace>/<id>@<version>`

示例: `smart://health/hypertension-thresholds@1.1.0`

### 4.2 Contents（静态知识）

存放不依赖外部输入的静态内容，支持多种格式：

```yaml
# Markdown 格式 - 适合大段文档
contents:
  format: text/markdown
  body: |
    # 操作指南
    1. 第一步...
    2. 第二步...
```

```yaml
# YAML 格式 - 适合结构化问答
contents:
  format: application/yaml
  body:
    faq:
      q1: 
        question: "常见问题"
        answer: "详细回答内容..."
```

```yaml
# JSON 格式 - 适合配置数据
contents:
  format: application/json
  body:
    threshold: 100
    enabled: true
```

#### 内容片段（Fragments）

YAML 格式的内容，每个顶层 key 可独立引用：

```yaml
contents:
  format: application/yaml
  body:
    # 这些 key 可通过 smart://...#medications 引用
    medications:
      first_line: [氨氯地平]
    thresholds:
      stage1: {sys: 140, dia: 90}
```

Markdown 格式的内容，可通过 `{#anchor}` 定义锚点：

```markdown
## 药物治疗 {#medications}
内容...

## 生活方式 {#lifestyle}
内容...
```

### 4.3 Services（可执行能力）

定义可调用的服务：

```yaml
services:
  - name: my-service        # 服务标识符
    text: "我的服务"         # 展示名称
    type: code/js           # 服务类型
    # ... 类型特定配置
```

### 4.4 Import（引用机制）

通过 URI 引用其他文档，支持完整文档或片段级引用：

```yaml
import:
  # 简化形式 - 引入完整文档
  - smart://health/thresholds@1.1.0
  
  # 完整形式 - 指定别名
  - uri: smart://common/utils@1.0.0
    as: utils
    
  # 片段引用 - 必须用引号包裹
  - "smart://health/htn-guide@1.0.0#medications"
  - "smart://health/htn-guide@1.0.0#critical"
```

**引用规则**:
- 完整文档：按主文档知识的结构合并被引用文档的知识（文本追加，结构化内容合并），追加被引用文档的服务
- 片段引用（`#fragment`）：仅加载指定片段的内容，可通过 `$import.<alias>.contents.fragment("fragment_name")` 访问

---

## 5. 函数类型详解

### 5.1 本地计算

#### code/js

```yaml
- name: calculate
  type: code/js
  timeout_ms: 5000                    # 超时时间（默认 5000）
  expr: |
    // 访问输入
    const val = input.x;
    
    // 访问 contents 中的知识
    const cfg = $contents.json("threshold");
    const doc = $contents.raw();     // 读取原始文本（如 Markdown）
    
    // 访问导入文档
    const imported = $import.utils.contents.json("config");
    
    // 调用其他函数
    const result = $call("other", {x: 1});
    
    // 工具函数
    $utils.to_number(x);
    $utils.coalesce(a, b);
    $utils.json_encode(obj);
    $utils.now();
    
    console.log("debug");            // 日志输出
    
    return result;
```

#### code/cel

```yaml
- name: check_age
  type: code/cel
  expr: input.age >= 18 && input.status == "active"
```

#### code/template

Mustache 语法，支持 `[[ ]]` 内嵌 CEL 表达式：

```yaml
- name: generate_text
  type: code/template
  expr: |
    您好，{{input.name}}！
    总计：[[price * quantity]]
    状态：{{$call("get_status", input)}}。
```

### 5.2 流程控制

#### flow/if

```yaml
- name: conditional
  type: flow/if
  condition:
    type: code/cel
    expr: input.valid == true
  then:
    ref: process
  default:
    ref: reject
```

**then/default 说明**：
- `ref: <name>` - 引用其他 service 执行
- `{type: <type>, expr: ...}` - 内联代码执行

#### flow/switch

```yaml
- name: classify
  type: flow/switch
  cases:
    - when:
        type: code/cel
        expr: input.score >= 90
      action:
        ref: handle_a                      # 引用其他 service
    - when:
        type: code/cel
        expr: input.score >= 80
      action:
        type: code/cel                     # 内联代码
        expr: "B"
  default:
    type: code/cel
    expr: "C"
```

**action 说明**：
- `ref: <name>` - 引用其他 service 执行
- `{type: <type>, expr: ...}` - 内联代码执行

#### flow/sequence

```yaml
- name: pipeline
  type: flow/sequence
  steps:
    - validate
    - transform
    - save
  merge: last                       # last | all | object
```

#### flow/loop

```yaml
- name: batch_process
  type: flow/loop
  over: input.items                 # 数组路径
  as: item                          # 迭代变量名
  do: process_item                  # 每个元素执行的函数
  concurrency: 5                    # 并发数（默认 1）
```

### 5.3 外部工具

#### tools/http

`url`、`headers`、`body` 支持模板语法（Mustache + CEL `[[ ]]`）：

```yaml
- name: api_call
  type: tools/http
  method: POST
  url: https://api.example.com/data
  headers:
    Authorization: Bearer {{$contents.json("api_key")}}
  body:
    key: "{{input.value}}"
  response_path: data.result
  timeout_ms: 10000
```

#### tools/llm

`system` 和 `expr` 均支持模板语法（Mustache + CEL `[[ ]]`）：

```yaml
- name: ask_ai
  type: tools/llm
  model: gpt-4
  system: "你是助手。上下文：{{$contents.raw()}}"
  expr: "{{input.question}}"
  temperature: 0.7
  max_tokens: 2000
```

---

## 6. HTTP API

### 6.1 基础

- **Base URL**: `http://localhost:8088`
- **Content-Type**: `application/json`

### 6.2 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/health` | 健康检查 |
| GET | `/api/v1/documents` | 列出文档 |
| POST | `/api/v1/documents` | 上传文档 |
| GET | `/api/v1/documents/{namespace}/{id}/{version}` | 获取文档 |
| GET | `/api/v1/documents/{namespace}/{id}/{version}/contents` | 获取 contents 内容 |
| GET | `/api/v1/documents/{namespace}/{id}/{version}/contents/{fragment}` | 获取指定片段 |
| POST | `/api/v1/documents/{namespace}/{id}/{version}/status` | 更新状态 |
| POST | `/api/v1/documents/{namespace}/{id}/{version}/services/{name}` | 执行服务 |
| POST | `/api/v1/documents/{namespace}/{id}/{version}/preview/{name}` | 预览 LLM 提示词 |

### 6.3 响应格式

**成功**:
```json
{ "data": { ... } }
```

**错误**:
```json
{ "error": { "code": "...", "message": "..." } }
```

### 6.4 流式输出

LLM 函数支持 SSE 流式响应：

```bash
curl -H "Accept: text/event-stream" \
  http://localhost:8088/api/v1/documents/health/htn-guide/1.0.0/services/query
```

事件类型: `connected`, `chunk`, `reasoning`, `complete`, `error`

---

## 7. MCP 接口

### 7.1 端点

```
POST /mcp
```

### 7.2 工具列表

| 工具名 | 说明 |
|--------|------|
| `smarttext.list` | 查询文档（支持按 namespace、tags 过滤） |
| `smarttext.read` | 读取文档或指定片段 |
| `smarttext.execute` | 执行函数 |

---

## 8. 详细规范

### 8.1 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | ✅ | 文档标识符（小写、数字、横线、下划线） |
| `version` | string | ✅ | 版本号 `A.B.C` 或 `A.B.C.D` |
| `namespace` | string | ✅ | 命名空间（小写字母开头，`a-z0-9_-`，1-32 字符） |
| `meta` | object | ❌ | 元数据块 |
| `import` | array | ❌ | 引用列表 |
| `contents` | object | ✅ | 静态知识块 |
| `services` | array | ❌ | 可执行服务列表 |

### 8.2 Meta 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `title` | string | 展示名称 |
| `description` | string | 文档描述 |
| `tags` | array | 标签列表，用于检索和分类 |
| `author` | string | 作者 |
| `created_at` | string | 创建时间（ISO 8601 格式） |
| `updated_at` | string | 最后更新时间（ISO 8601 格式） |
| `status` | string | `draft` / `published` |
| `x-*` | any | 应用层扩展字段 |

### 8.3 约束规则

**id 规则**:
- 长度：1-64 字符
- 字符：`a-z`、`0-9`、`-`、`_`
- 必须以小写字母开头

**version 规则**:
- 格式：`A.B.C` 或 `A.B.C.D`，数字

**import URI 规则**:
```
smart://<namespace>/<id>[@<version>][#<fragment>]
```

- 无 `@version`：使用当前文档版本作为 fallback
- 有 `#fragment`：仅加载指定片段（必须用引号包裹避免 YAML 注释）

### 8.4 文件存储（单文件）

```
<data-dir>/<namespace>/<id>/<version>/main.yaml
```

示例:
```
data/docs/health/bp-assistant/1.0.0/main.yaml
```

### 8.5 文件存储（目录布局）

对于复杂文档，可以使用目录布局替代单一 YAML：

```
<data-dir>/<namespace>/<id>/<version>/
├── main.yaml          # 可选，可为空
├── meta.yaml          # 可选元数据
├── contents.md        # 或 contents.json / contents.yaml（三选一）
├── services/          # 服务声明
│   └── query.js       # 或 .yaml
└── docs/              # 本地依赖（自动导入）
    ├── thresholds.yaml
    ├── prompts.md
    └── helpers.js
```

**规则：**
- `id` 必须用 `_` 不能用 `.`（如 `risk_assistant` 而非 `risk.assistant`）
- 内容格式：`contents.md`、`contents.json`、`contents.yaml` 只能存在一个
- 服务命名：`services/query.js` 或 `services.query.js`，冲突即报错
- `docs/` 下文件作为本地依赖自动导入

---

## 附录

### A. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-03-15 | 初始稳定版本 |

### B. 许可证

MIT License - 详见 LICENSE 文件
