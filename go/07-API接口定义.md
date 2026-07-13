# API接口定义 — OPDS 主干 + 自定义扩展（Go 版本）

## 🎯 设计原则

> **OPDS 只管发现和分发，同步由自建扩展 API 负责。**
> **OPDS 标准只做三件事：浏览书目、搜索内容、获取文件。没有同步功能。**

| 原则 | 说明 |
|------|------|
| **OPDS 主干** | OPDS 1.2 / 2.0 — 书目发现、全文搜索、文件获取（不含同步） |
| **进度同步** | OPDS Progression 草案格式 — `{progression, references, device, modified}` |
| **标记协议** | W3C Web Annotation — 高亮/注释/书签（motivation 区分类型） |
| **扩展协议** | RESTful JSON — 上传、词典、管理 |
| **认证** | Bearer JWT |
| **内容协商** | OPDS 路由通过 `Accept` 头区分 Atom XML / JSON-LD |

---

## 📡 路由全景

```
                    ┌────────────────────────┐
                    │  OPDS 主干（标准协议）   │
                    │  仅：发现 + 搜索 + 获取  │
                    ├────────────────────────┤
                    │ GET  /opds/catalog      │
                    │ GET  /opds/v2/catalog   │
                    │ GET  /opds/search       │
                    │ GET  /opds/catalog/search│
                    │ GET  /opds/acquisition/  │
                    └────────────────────────┘

                    ┌─────────────────────────────┐
                    │   私有扩展 API（Go Handler） │
                    ├─────────────────────────────┤
                    │ POST /api/v1/auth/*          │
                    │ POST /api/v1/documents/upload│
                    │ GET/PUT /api/v1/progression/ │ ★ OPDS Progression 草案
                    │ CRUD /api/v1/annotations/*   │ ★ W3C Web Annotation
                    │ CRUD /api/v1/dictionaries/*  │
                    │ GET/PUT /api/v1/admin/*      │
                    │ GET  /health                │
                    └─────────────────────────────┘
```

---

## 📚 1. OPDS 书目目录（标准协议，与 Rust 版相同，略）

## 🔍 2. OPDS 搜索（标准协议，与 Rust 版相同，略）

## 📥 3. OPDS 获取（标准协议，与 Rust 版相同，略）

---

# 第二部分：同步与标记 API（Go 版本）

## 🔄 4. 阅读进度同步（OPDS Progression 草案）

### 4.1 获取进度
- **GET** `/api/v1/progression/{document_id}`
- **Response** `200 OK`:
```json
{
  "title": "深入理解计算机系统（第3版）",
  "modified": "2024-01-15T10:30:00Z",
  "device": { "id": "urn:uuid:...", "name": "PDF Manager Web" },
  "progression": 0.057,
  "references": ["#page=42"]
}
```

### 4.2 更新进度
- **PUT** `/api/v1/progression/{document_id}`
- **Body**: `{ "progression": 0.058, "references": ["#page=43"], "device": {...} }`

### 4.3 增量同步
- **GET** `/api/v1/progression?since=2024-01-14T00:00:00Z`
- **Response**: 只返回 `modified > since` 的进度记录

---

## 🖍️ 5. 高亮与注释（W3C Web Annotation）

### 5.1 创建标记
- **POST** `/api/v1/annotations`
- **Body**:
```json
{
  "@context": "http://www.w3.org/ns/anno.jsonld",
  "type": "Annotation",
  "motivation": "highlighting",
  "target": { "source": "/docs/101", "selector": { "type": "FragmentSelector", "value": "page=42&xywh=120,340,180,15" }},
  "body": { "type": "TextualBody", "value": "笔记内容", "format": "text/plain" },
  "style": { "color": "#FFEB3B" }
}
```

### 5.2 获取标记
- **GET** `/api/v1/annotations?document_id=101&motivation=highlighting&since=2024-01-14T00:00:00Z`

### 5.3 更新/删除
- **PUT** `/api/v1/annotations/{id}`
- **DELETE** `/api/v1/annotations/{id}`

### 5.4 motivation 对照
| motivation | 含义 | body |
|-----------|------|------|
| `highlighting` | 纯高亮 | 否 |
| `commenting` | 带笔记的高亮 | 是 |
| `bookmarking` | 书签 | 否 |

---

## 📤 6. 文档上传 / 🔐 7. 认证 / 📖 8. 词典 / ⚙️ 9. 管理 / 🏥 10. 健康检查

（与 Rust 版相同，略）

---

## 📊 路由对照：旧 vs 新（Go 版迁移）

| 旧路由（已废弃） | 新路由 | 协议 |
|------|------|------|
| `GET /opds/sync/progress/:id` | `GET /api/v1/progression/:id` | OPDS Progression |
| `GET/POST /opds/sync/bookmark/:id` | `/api/v1/annotations` (motivation=bookmarking) | W3C Web Annotation |
| `* /api/v1/highlights/*` | `/api/v1/annotations` (motivation=highlighting) | W3C Web Annotation |
| `* /api/v1/annotations/*` | `/api/v1/annotations` (motivation=commenting) | W3C Web Annotation |
| `* /api/v1/bookmarks/*` | `/api/v1/annotations` (motivation=bookmarking) | W3C Web Annotation |
