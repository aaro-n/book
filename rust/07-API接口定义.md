# API接口定义 — OPDS 主干 + 自定义扩展

## 🎯 设计原则

> **OPDS 只管发现和分发，同步由自建扩展 API 负责。**
> **OPDS 标准只做三件事：浏览书目、搜索内容、获取文件。没有同步功能。**

| 原则 | 说明 |
|------|------|
| **OPDS 主干** | OPDS 1.2 / 2.0 — 书目发现、全文搜索、文件获取（不含同步） |
| **同步协议** | Readium Locator（进度定位） + W3C Web Annotation（高亮/注释） |
| **扩展协议** | RESTful JSON — 上传、词典、管理 |
| **兼容同步** | KOReader Sync Protocol + WebDAV（第三方阅读器进度同步） |
| **认证** | Bearer SessionToken + API Key（详见 `15-认证与安全.md`） |
| **内容协商** | OPDS 路由通过 `Accept` 头区分 Atom XML / JSON-LD |
| **版本** | `/opds/` 无版本号（协议自带版本），`/api/v1/` 带版本 |

---

## 📡 路由全景

```
                    ┌────────────────────────┐
                    │  OPDS 主干（标准协议）   │
                    │  仅：发现 + 搜索 + 获取  │
                    ├────────────────────────┤
                    │ GET  /opds/catalog      │  书目导航 (Atom Feed)
                    │ GET  /opds/v2/catalog   │  书目目录 (JSON-LD)
                    │ GET  /opds/search       │  OpenSearch 描述文档
                    │ GET  /opds/catalog/search│ 全文搜索 (Atom)
                    │ GET  /opds/v2/catalog/search│全文搜索 (JSON-LD)
                    │ GET  /opds/acquisition/  │  获取/下载 PDF
                    └────────────────────────┘

                    ┌─────────────────────────────┐
                    │   自定义扩展（自建私有 API）  │
                    ├─────────────────────────────┤
                    │ GET  /files/*               │  文件代理（→ S3 签名 URL） │
                    │ POST /api/v1/auth/*         │  认证（注册/登录/Session）
                    │ POST /api/v1/documents/upload│ 文档上传
                    │ CRUD /api/v1/annotations/*  │  高亮/注释/书签（Web Annotation）
                    │ GET/PUT /api/v1/progression/  │ 阅读进度（OPDS Progression）
                    │ CRUD /api/v1/dictionaries/* │  自定义词典
                    │ GET/PUT /api/v1/admin/*     │  后台管理
                    │ GET  /health               │  健康检查
                    └─────────────────────────────┘

                    ┌─────────────────────────────────┐
                    │  第三方阅读器同步（可选兼容层）   │
                    ├─────────────────────────────────┤
                    │ /koreader-sync/*                │  KOReader Sync Protocol
                    │ /webdav/*                       │  WebDAV 进度文件
                    └─────────────────────────────────┘
```
> ★ 同步路由是私有 REST API，不属于 OPDS 标准。OPDS Feed 中通过自定义 `rel` 链接指向它们。

---

# 第一部分：OPDS 主干（标准协议）

> OPDS 1.2 规范范围：https://specs.opds.io/opds-1.2
> OPDS 2.0 规范范围：https://drafts.opds.io/opds-2.0

## 📚 1. OPDS 书目目录

### 1.1 根目录导航（OPDS 1.2 Navigation Feed）

- **GET** `/opds/catalog`
- **Accept**: `application/atom+xml`
- **Auth**: `Authorization: Bearer <session_token>` 或 URL 路径 `/opds/{apiKey}/catalog`

**Response** `200 OK`：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom"
      xmlns:opds="http://opds-spec.org/2010/catalog"
      xmlns:dcterms="http://purl.org/dc/terms/">

  <id>urn:uuid:pdf-manager-root</id>
  <title>我的 PDF 图书馆</title>
  <updated>2024-01-15T10:30:00Z</updated>
  <author><name>PDF Manager</name></author>

  <link rel="self" href="/opds/catalog"
        type="application/atom+xml;profile=opds-catalog;kind=navigation"/>
  <link rel="start" href="/opds/catalog"
        type="application/atom+xml;profile=opds-catalog;kind=navigation"/>
  <link rel="search" href="/opds/search"
        type="application/opensearchdescription+xml"/>

  <entry>
    <title>最近添加</title>
    <id>urn:uuid:recent</id>
    <content type="text">最近 30 天上传的文档</content>
    <link rel="subsection" href="/opds/catalog/recent"
          type="application/atom+xml;profile=opds-catalog;kind=acquisition"/>
  </entry>

  <entry>
    <title>按标签浏览</title>
    <id>urn:uuid:tags</id>
    <link rel="subsection" href="/opds/catalog/tags"
          type="application/atom+xml;profile=opds-catalog;kind=acquisition"/>
  </entry>

  <entry>
    <title>所有文档</title>
    <id>urn:uuid:all</id>
    <link rel="subsection" href="/opds/catalog/all"
          type="application/atom+xml;profile=opds-catalog;kind=acquisition"/>
  </entry>
</feed>
```

### 1.2 书目列表（OPDS 1.2 Acquisition Feed）

- **GET** `/opds/catalog/all?page=1&limit=20`
- **Accept**: `application/atom+xml`

**Response** `200 OK`：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom"
      xmlns:opds="http://opds-spec.org/2010/catalog"
      xmlns:dcterms="http://purl.org/dc/terms/">

  <id>urn:uuid:all-documents</id>
  <title>所有文档</title>
  <updated>2024-01-15T10:30:00Z</updated>

  <link rel="self"  href="/opds/catalog/all?page=1" type="application/atom+xml"/>
  <link rel="next"  href="/opds/catalog/all?page=2" type="application/atom+xml"/>

  <entry>
    <title>深入理解计算机系统（第3版）</title>
    <id>urn:uuid:550e8400-e29b-41d4-a716-446655440000</id>
    <author><name>Randal E. Bryant</name></author>
    <author><name>David R. O&apos;Hallaron</name></author>
    <dcterms:publisher>机械工业出版社</dcterms:publisher>
    <dcterms:issued>2016-11</dcterms:issued>
    <dcterms:language>zh</dcterms:language>
    <summary>计算机科学经典教材，涵盖数据表示、汇编、内存层次、链接等核心主题</summary>
    <updated>2024-01-10T08:00:00Z</updated>

    <!-- OPDS 标准链接：封面 -->
    <link rel="http://opds-spec.org/image/thumbnail"
          href="/api/v1/documents/101/cover"
          type="image/jpeg"/>

    <!-- OPDS 标准链接：获取 PDF -->
    <link rel="http://opds-spec.org/acquisition"
          href="/opds/acquisition/101"
          type="application/pdf"/>

    <!-- 自定义扩展链接：进度同步（OPDS Progression 草案） -->
    <link rel="http://opds-spec.org/progression"
          href="/api/v1/progression/101"
          type="application/opds-progression+json"
          title="阅读进度"/>

    <!-- 自定义扩展链接：标记同步（W3C Web Annotation） -->
    <link rel="http://www.w3.org/ns/oa#annotationService"
          href="/api/v1/annotations?document_id=101"
          type="application/ld+json"
          title="标记"/>
  </entry>
</feed>
```

### 1.3 书目目录（OPDS 2.0 JSON-LD）

- **GET** `/opds/v2/catalog`
- **Accept**: `application/opds+json`

```json
{
  "metadata": {
    "title": "我的 PDF 图书馆",
    "numberOfItems": 42,
    "modified": "2024-01-15T10:30:00Z"
  },
  "links": [
    { "rel": "self",   "href": "/opds/v2/catalog", "type": "application/opds+json" },
    { "rel": "search", "href": "/opds/search?q={query}&page={page}", "type": "application/opds+json", "templated": true }
  ],
  "publications": [
    {
      "metadata": {
        "@type": "http://schema.org/Book",
        "identifier": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "title": "深入理解计算机系统（第3版）",
        "author": [{"name": "Randal E. Bryant"}, {"name": "David R. O'Hallaron"}],
        "publisher": "机械工业出版社",
        "language": "zh",
        "published": "2016-11",
        "description": "计算机科学经典教材",
        "numberOfPages": 737,
        "modified": "2024-01-10T08:00:00Z"
      },
      "links": [
        {"rel": "http://opds-spec.org/acquisition", "href": "/opds/acquisition/101", "type": "application/pdf"},
        {"rel": "http://opds-spec.org/progression", "href": "/api/v1/progression/101", "type": "application/opds-progression+json", "title": "阅读进度"},
        {"rel": "http://www.w3.org/ns/oa#annotationService", "href": "/api/v1/annotations?document_id=101", "type": "application/ld+json", "title": "标记"}
      ],
      "images": [
        {"href": "/api/v1/documents/101/cover", "type": "image/jpeg", "rel": "http://opds-spec.org/image/thumbnail"}
      ]
    }
  ]
}
```

---

## 🔍 2. OPDS 搜索

### 2.1 OpenSearch 描述文档

- **GET** `/opds/search`
- **Accept**: `application/opensearchdescription+xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OpenSearchDescription xmlns="http://a9.com/-/spec/opensearch/1.1/">
  <ShortName>PDF 图书馆搜索</ShortName>
  <Description>全文搜索 PDF 文档内容，支持中日韩分词</Description>
  <Url type="application/atom+xml"
       template="/opds/catalog/search?q={searchTerms}&amp;page={startPage?}&amp;language={language?}"/>
  <Url type="application/opds+json"
       template="/opds/v2/catalog/search?q={searchTerms}&amp;page={startPage?}&amp;language={language?}"/>
  <Language>zh</Language>
  <Language>ja</Language>
  <Language>ko</Language>
  <Language>en</Language>
</OpenSearchDescription>
```

### 2.2 执行搜索（OPDS 1.2）

- **GET** `/opds/catalog/search?q=人工智能&page=1&limit=20`
- **Accept**: `application/atom+xml`

```xml
<feed xmlns="http://www.w3.org/2005/Atom"
      xmlns:opensearch="http://a9.com/-/spec/opensearch/1.1/">
  <title>搜索：人工智能</title>
  <opensearch:totalResults>15</opensearch:totalResults>
  <opensearch:startIndex>1</opensearch:startIndex>
  <opensearch:itemsPerPage>20</opensearch:itemsPerPage>
  <entry>
    <title>深入理解计算机系统</title>
    <id>urn:uuid:550e8400-e29b-41d4-a716-446655440000</id>
    <summary>...&lt;em&gt;人工智能&lt;/em&gt;是计算机科学的重要分支...</summary>
    <link rel="http://opds-spec.org/acquisition" href="/opds/acquisition/101" type="application/pdf"/>
  </entry>
</feed>
```

### 2.3 执行搜索（OPDS 2.0 JSON-LD）

- **GET** `/opds/v2/catalog/search?q=人工智能&page=1`
- **Accept**: `application/opds+json`

```json
{
  "metadata": { "title": "搜索：人工智能", "numberOfItems": 15, "itemsPerPage": 20, "currentPage": 1 },
  "publications": [{
    "metadata": { "title": "深入理解计算机系统", "identifier": "urn:uuid:..." },
    "links": [{ "rel": "http://opds-spec.org/acquisition", "href": "/opds/acquisition/101", "type": "application/pdf" }],
    "searchInfo": {
      "score": 0.98,
      "hits": [
        { "pageNum": 12, "snippet": "...<em>人工智能</em>是计算机科学的重要分支..." },
        { "pageNum": 45, "snippet": "...<em>人工智能</em>系统的设计与实现..." }
      ]
    }
  }]
}
```

---

## 📥 3. OPDS 获取（Acquisition）

### 3.1 下载 PDF

- **GET** `/opds/acquisition/{document_id}`
- **Auth**: `Authorization: Bearer <session_token>` 或 `X-API-Key: <api_key>`

后端从 S3 获取预签名 URL，302 重定向（省流量），也可 `?format=url` 返回 JSON：
```json
{ "url": "https://s3.amazonaws.com/...", "expires_at": "2024-01-15T11:30:00Z" }
```

---

# 第二部分：同步与标记 API（Readium 生态标准）

> **进度同步**：OPDS Progression 草案（简单、内置 device、支持冲突解决）
> **高亮/注释**：W3C Web Annotation + Readium Locator fragments
> 所有字段均已做 PDF 适配。全部为私有 API，不属于 OPDS 标准。
>
> 详细研究见 `13-同步方案研究.md`（分析了 Komga、Kavita、Calibre-Web 等 10 个项目的同步机制）

---

## 🔄 4. 阅读进度同步（OPDS Progression 草案格式）

> 格式依据：`opds-community/drafts` 的 `opds-progression-1.0.md`
> **为什么不用 Locator 做进度同步？** Progression 比 Locator 更简单（只有 4 个核心字段），内置 device 追踪和 modified 时间戳，专为"读到哪了"设计。

### 4.1 获取进度

- **GET** `/api/v1/progression/{document_id}`
- **Response** `200 OK` — OPDS Progression 格式：
```json
{
  "title": "深入理解计算机系统（第3版）",
  "modified": "2024-01-15T10:30:00Z",
  "device": {
    "id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
    "name": "PDF Manager Web"
  },
  "progression": 0.057,
  "references": [
    "#page=42"
  ]
}
```

**字段说明（PDF 适配）**：
| 字段 | 含义 | 示例 |
|------|------|------|
| `progression` | 全书进度 0.0 ~ 1.0 | `0.057` (= 第42页 / 737页) |
| `references` | 媒体片段 URI 数组 | `["#page=42"]` |
| `device.id` | 设备唯一 ID（UUID） | 多端同步时区分"iPad读到哪 vs 手机读到哪" |
| `modified` | 最后更新时间 | 用于冲突解决（两台设备同时更新，以新的为准） |

### 4.2 更新进度

- **PUT** `/api/v1/progression/{document_id}`
- **Body**（OPDS Progression 格式）:
```json
{
  "progression": 0.058,
  "references": ["#page=43"],
  "device": {
    "id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
    "name": "PDF Manager Web"
  }
}
```
- **Response** `200 OK`:
```json
{ "status": "synced", "modified": "2024-01-15T10:31:00Z" }
```

### 4.3 OPDS Feed 自动发现

在 OPDS Feed 中嵌入 Progression 端点链接，让兼容客户端自动发现：

```json
{
  "rel": "http://opds-spec.org/progression",
  "href": "/api/v1/progression/101",
  "type": "application/opds-progression+json"
}
```

### 4.4 获取所有文档的进度列表（增量同步）

- **GET** `/api/v1/progression?since=2024-01-14T00:00:00Z`
- **Response**：只返回 `modified > since` 的进度记录
- 用于客户端"自从上次同步以来，哪些文档的进度变了"

---

## 🖍️ 5. 高亮与注释（W3C Web Annotation + Readium Locator）

> W3C Web Annotation 规范：https://www.w3.org/TR/annotation-model/
> Locator fragments 规范：`readium.org/architecture/models/locators/`
> **PDF 适配**：`target.selector` 使用 PDF 页面坐标（Media Fragment URI），而非 EPUB 的 DOM 选择器。

### 5.1 Readium Locator 在标记中的角色

高亮/注释用 W3C Web Annotation 的 JSON-LD 作为数据模型，但其 `target.selector.value` 使用 Readium Locator 的 fragments 格式：

| 格式 | fragments 示例 |
|------|---------------|
| PDF | `"page=42"`、`"page=42&xywh=120,340,180,15"` |
| EPUB | `"#chapter1"`、`"#:~:text=重要的段落"` |
| 音频 | `"#t=67"` |

### 5.2 Web Annotation 核心结构

```json
{
  "@context": "http://www.w3.org/ns/anno.jsonld",
  "type": "Annotation",
  "motivation": "highlighting",      // "highlighting" | "commenting" | "bookmarking"
  "body": {
    "type": "TextualBody",
    "value": "用户的笔记文字",        // （仅当 motivation=commenting 时）
    "format": "text/plain"
  },
  "target": {
    "source": "/docs/101",           // 文档标识
    "selector": {
      "type": "FragmentSelector",
      "value": "page=42&xywh=120,340,180,15"  // PDF 坐标片段
    }
  }
}
```

### 5.3 创建高亮

- **POST** `/api/v1/documents/{doc_id}/annotations`
- **Body**（W3C Web Annotation）:
```json
{
  "@context": "http://www.w3.org/ns/anno.jsonld",
  "type": "Annotation",
  "motivation": "highlighting",
  "target": {
    "source": "/docs/101",
    "selector": {
      "type": "FragmentSelector",
      "value": "page=42&xywh=120,340,180,15"
    }
  },
  "body": {
    "type": "TextualBody",
    "value": "加速比 = 1/((1-α) + α/k)",
    "format": "text/plain"
  },
  "style": { "color": "#FFEB3B" }
}
```
- **Response** `201 Created`: 返回完整 Annotation，含服务端生成的 `id` 和时间戳。

> **`selector.value` 格式说明（PDF 专用）**：
> `page=42&xywh=120,340,180,15` 表示第 42 页，坐标 (x=120, y=340)，宽 180，高 15。
> 这是 Media Fragment URI 风格，替代 EPUB 的 DOM 选择器。

### 5.4 motivation 对照表（扩展）

| motivation | 本项目含义 | 对应 `annotation_type` | body 是否必需 |
|-----------|-----------|----------------------|--------------|
| `highlighting` | 高亮/划线 | `highlight` / `underline` / `squiggly` / `strikethrough` | 否 |
| `commenting` | 带文字笔记 | `note` / `sticky_note` / `freetext` | 是 |
| `bookmarking` | 书签 | `bookmark` | 否 |
| `drawing` | 图形标注 | `freehand` / `circle` / `rectangle` / `line` / `polygon` / `stamp` | 可选 |

### 5.5 创建标注（全类型）

Flutter 客户端可创建全类型。Web 端可创建文本型标注，图形型为只读渲染。

- **POST** `/api/v1/documents/{doc_id}/annotations`
- **Body**（W3C Web Annotation 扩展）:
```json
{
  "@context": "http://www.w3.org/ns/anno.jsonld",
  "type": "Annotation",
  "motivation": "drawing",
  "target": {
    "source": "/docs/101",
    "selector": {
      "type": "FragmentSelector",
      "value": "page=15"
    }
  },
  "body": [
    {
      "type": "TextualBody",
      "value": "这个公式重要",
      "format": "text/plain"
    },
    {
      "type": "Image",
      "value": "data:image/svg+xml;base64,...",
      "format": "image/svg+xml"
    }
  ],
  "style": {
    "type": "freehand",
    "color": "#FF0000",
    "strokeWidth": 2.0
  },
  "device": { "id": "urn:uuid:a1b2c3d4-..." }
}
```

### 5.6 获取文档所有标注

- **GET** `/api/v1/documents/{doc_id}/annotations`
- **Query**: `?page=42` 只取第 42 页；`?type=freehand,highlight` 过滤类型；`?search=人工智能` 全职搜索
- **Response** `200 OK`:
```json
{
  "@context": "http://www.w3.org/ns/anno.jsonld",
  "type": "AnnotationCollection",
  "total": 15,
  "items": [ /* Annotation 数组 */ ]
}
```

### 5.7 更新/删除

- **PUT** `/api/v1/annotations/{id}` — 更新 `body.value`、`style.color`、`stroke_data`
- **DELETE** `/api/v1/annotations/{id}` — 软删除（`is_deleted=true`）

> **Web 端渲染**：GET 标注后，文本型 → CSS 绝对定位覆盖；图形型 → 读取 `stroke_data` 中的 SVG path 用 Canvas/SVG 绘制。Web 端无法创建手写/涂鸦，但可**完整展示**所有类型。

---

## 📤 6. 文档上传（扩展）

- **POST** `/api/v1/documents/upload`
  - `multipart/form-data`, file (pdf/epub) + title + tags
  - 服务端自动检测文档类型，能力范围内直接处理，超范围的返回提示
- **POST** `/api/v1/documents/import`
  - `multipart/form-data`, file (.bookpkg) — CLI 预处理的打包文件直接导入
- **GET** `/api/v1/documents/{id}/status`
  - 状态: `processing` → `ready` → `error`
  - `error` 时附带 `hint` 字段提示下一步操作

---

## 📎 6.1. 文档内容访问（扩展）

> **完整设计见** `16-S3存储方案.md`。所有 S3 文件通过 302 跳转到签名 URL（含 CDN）。
> 签名 URL 缓存：Redis → PG → S3 SDK，有效期 30 天。

| API | 说明 |
|-----|------|
| **GET** `/files/{s3_key}` | 统一文件代理端点 → 302 → CDN/S3 |
| **GET** `/api/v1/documents/{id}/pages?format=url` | 返回所有页面的完整 URL 列表（Flutter 客户端一次性获取，缓存 30 天） |
| **GET** `/api/v1/documents/{id}/cover` | 封面（→ 302 → CDN/S3） |

> 前端 HTML：`<img src="/files/{doc_id}/thumbs/001.jpg">`、`<embed src="/files/{doc_id}/pages/001.pdf">`
> Flutter：`GET /api/v1/documents/{id}/pages?format=url` 一次性拿全部 URL 存本地。

---

## 🔐 7. 认证模块（扩展）

> **完整设计见** `15-认证与安全.md`。此处仅列出端点摘要。
>
> 认证方案：Session Token（Redis, 7天）+ API Key（SHA256 存储）。不用 JWT。

| 方法 | 路径 | 说明 |
|------|------|------|
| **POST** | `/api/v1/auth/register` | 注册（注册即登录，返回 token） |
| **POST** | `/api/v1/auth/login` | 登录（返回 token + Set-Cookie） |
| **POST** | `/api/v1/auth/logout` | 登出（删除 session） |
| **GET** | `/api/v1/auth/me` | 获取当前用户信息 |
| **PUT** | `/api/v1/auth/password` | 修改密码（踢出所有设备） |
| **GET** | `/api/v1/auth/api-keys` | 列出 API Key |
| **POST** | `/api/v1/auth/api-keys` | 创建 API Key（明文仅展示一次） |
| **DELETE** | `/api/v1/auth/api-keys/{id}` | 删除 API Key |

---

## 📖 8. 自定义词典管理（扩展）

- **GET** `/api/v1/dictionaries`
- **POST** `/api/v1/dictionaries` — 副作用：触发全服务热更新
- **POST** `/api/v1/dictionaries/import`
- **GET** `/api/v1/dictionaries/export`
- **DELETE** `/api/v1/dictionaries/{id}`

---

## ⚙️ 9. 后台管理（扩展）

- **GET/PUT** `/api/v1/admin/segmenter`
- **GET/PUT** `/api/v1/admin/search-engine`
- **POST** `/api/v1/admin/segmenter/test`
- **GET** `/api/v1/admin/metrics`

---

## 🏥 10. 健康检查

- **GET** `/health` → `{ "status": "ok", "db": "ok", "redis": "ok", "s3": "ok" }`

---

# 第三部分：第三方阅读器同步兼容层（可选）

> 以下协议与 OPDS 完全独立，用于让第三方阅读器（KOReader 等）也能同步进度。

## 📡 KOReader Sync Protocol（推荐）

KOReader 有一套轻量的专属同步 API。可通过 Docker 自建 `koreader-sync-server`。

- **项目地址**：https://github.com/koreader/koreader-sync-server
- **Docker 部署**：`ghcr.io/koreader/koreader-sync-server`
- **作用**：KOReader 多设备间进度秒级同步
- **本项目如何对接**：KOReader 通过 OPDS 从本项目获取 PDF，通过自建 sync-server 同步进度

## 📂 WebDAV 方案（最通用）

绝大多数第三方阅读器（静读天下 Moon+ Reader、KOReader、KyBook 3 等）支持 WebDAV 同步。

- **原理**：阅读器将进度/书签打包为 JSON/SQLite 文件，通过 WebDAV 上传/下载
- **缺点**：不同阅读器间不互通（静读天下的进度 ≠ KOReader 的进度）
- **本项目如何对接**：可部署 WebDAV 服务（如 Nginx + WebDAV 模块），让用户的第三方阅读器自行同步

---

## 🔀 11. 内容协商

| Accept 头 | 格式 | 适用路由 |
|-----------|------|---------|
| `application/atom+xml` | OPDS 1.2 Atom Feed | `/opds/catalog*` |
| `application/opds+json` | OPDS 2.0 JSON-LD | `/opds/v2/*` |
| `application/opensearchdescription+xml` | OpenSearch | `/opds/search` |
| `application/json` | 默认 JSON | `/api/v1/*` |
| `application/ld+json` | W3C Annotation | `/api/v1/annotations/*` |

---

## 📊 12. 路由对照表

| 功能 | 路由 | 协议归属 |
|------|------|---------|
| 书目浏览 | `GET /opds/catalog/*` | **OPDS 1.2 标准** |
| 书目浏览 | `GET /opds/v2/catalog` | **OPDS 2.0 标准** |
| 全文搜索 | `GET /opds/catalog/search` | **OPDS 1.2 + OpenSearch** |
| PDF 下载 | `GET /opds/acquisition/:id` | **OPDS 1.2 标准** |
| 进度同步 | `GET/PUT /api/v1/progression/:id` | **OPDS Progression 草案** |
| 增量同步 | `GET /api/v1/progression?since=...` | **OPDS Progression 草案** |
| 高亮/注释/书签 | `/api/v1/annotations/*` | **W3C Web Annotation + Locator fragments** |
| 上传 | `/api/v1/documents/upload` | 私有 API |
| 词典 | `/api/v1/dictionaries/*` | 私有 API |
| 管理 | `/api/v1/admin/*` | 私有 API |
| 认证 | `/api/v1/auth/*` | 私有 API |

---

## 🔗 13. 第三方阅读器兼容

实现 OPDS 后，以下阅读器可直接接入**书目浏览和下载**：

| 阅读器 | 平台 | OPDS 支持 | 同步方式 |
|--------|------|----------|---------|
| **KOReader** | 桌面/手机/E-ink | OPDS 1.2 | KOReader Sync Server 或 WebDAV |
| **FBReader** | 全平台 | OPDS 1.2 + 2.0 | 无内置同步 |
| **Marvin** | iOS | OPDS 1.2 | 无内置同步 |
| **Calibre** | 桌面 | OPDS 1.2 客户端 | 无内置同步 |
| **静读天下 Moon+** | Android | OPDS 1.2 | WebDAV |
| **自研 Flutter 客户端** | 全平台 | OPDS 2.0 + 私有 API | Readium Locator + Web Annotation |

> **第三方阅读器的进度同步 ≠ 本项目高亮/注释同步**。KOReader 等通过自带 sync-server/WebDAV 同步进度；高亮/注释只能通过本项目的 W3C Web Annotation API。

---

## 📚 14. Readium 生态参考资料

| 资源 | 地址 | 用途 |
|------|------|------|
| **Readium 架构总览** | `github.com/readium/architecture` | 理解 Streamer / Navigator / Publication Server 分工 |
| **WebPub Manifest 规范** | `github.com/readium/webpub-manifest` | 图书元数据的 JSON 清单格式（`readingOrder`, `resources`） |
| **OPDS 2.0 规范** | `specs.opds.io/opds-2.0.html` | 分发协议：目录 Feed + 获取链接 |
| **Readium Locator 规范** | `readium.org/architecture/models/locators/` | 进度定位数据结构（position, progression, totalProgression） |
| **W3C Web Annotation** | `w3.org/TR/annotation-model/` | 高亮/注释的标准 JSON-LD 模型 |
| **Readium Kotlin Toolkit** | `github.com/readium/kotlin-toolkit` | Android 客户端参考实现（如何生成 Locator） |

> **WebPub Manifest** 在本项目中的意义：PDF 的 `readingOrder` 就是页面列表（`page=1` 到 `page=N`），`resources` 包含封面图片、字体等。可在 `/opds/v2/catalog` 中嵌入 WebPub Manifest 链接，让 Readium 兼容阅读器直接理解文档结构。
