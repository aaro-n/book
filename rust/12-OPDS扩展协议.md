# OPDS 协议完整参考 — Schema 定义 + Rust 实现指南

> 本文档提供 OPDS 协议在本项目中使用的完整数据模型、XML/JSON Schema 以及 Rust 实现代码。
> Go 版本见 `../go/12-OPDS扩展协议.md`

---

## 一、OPDS 数据模型定义

### 1.1 Rust 结构体（serde + quick-xml）

```rust
// src/models/opds.rs

use serde::{Deserialize, Serialize};

// ===== OPDS 1.2 (quick-xml serialize) =====

#[derive(Debug, Serialize)]
#[serde(rename = "feed")]
pub struct OPDSFeed {
    #[serde(rename = "@xmlns")]
    pub xmlns: String,               // "http://www.w3.org/2005/Atom"
    #[serde(rename = "@xmlns:opds")]
    pub xmlns_opds: String,          // "http://opds-spec.org/2010/catalog"
    #[serde(rename = "@xmlns:dcterms")]
    pub xmlns_dcterms: String,       // "http://purl.org/dc/terms/"
    pub id: String,
    pub title: String,
    pub updated: String,             // RFC3339
    pub author: AtomAuthor,
    #[serde(rename = "link", default)]
    pub links: Vec<AtomLink>,
    #[serde(rename = "entry", default)]
    pub entries: Vec<OPDSEntry>,
}

#[derive(Debug, Serialize)]
pub struct OPDSEntry {
    pub title: String,
    pub id: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub summary: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub content: Option<AtomContent>,
    #[serde(rename = "link", default)]
    pub links: Vec<AtomLink>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub updated: Option<String>,
    // Dublin Core
    #[serde(rename = "dcterms:publisher", skip_serializing_if = "Option::is_none")]
    pub publisher: Option<String>,
    #[serde(rename = "dcterms:issued", skip_serializing_if = "Option::is_none")]
    pub issued: Option<String>,
    #[serde(rename = "dcterms:language", skip_serializing_if = "Option::is_none")]
    pub language: Option<String>,
    #[serde(rename = "author", default)]
    pub authors: Vec<AtomAuthor>,
}

#[derive(Debug, Serialize)]
pub struct AtomAuthor {
    pub name: String,
}

#[derive(Debug, Serialize)]
pub struct AtomContent {
    #[serde(rename = "@type")]
    pub r#type: String,
    #[serde(rename = "$text")]
    pub text: String,
}

#[derive(Debug, Serialize)]
pub struct AtomLink {
    #[serde(rename = "@href")]
    pub href: String,
    #[serde(rename = "@rel")]
    pub rel: String,
    #[serde(rename = "@type")]
    pub r#type: String,
    #[serde(rename = "@title", skip_serializing_if = "Option::is_none")]
    pub title: Option<String>,
}

// ===== OPDS 2.0 (JSON-LD via serde_json) =====

#[derive(Debug, Serialize)]
pub struct OPDS2Catalog {
    pub metadata: OPDS2Metadata,
    #[serde(default)]
    pub links: Vec<OPDS2Link>,
    #[serde(default)]
    pub publications: Vec<OPDS2Publication>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub search_info: Option<SearchInfo>,
}

#[derive(Debug, Serialize)]
pub struct OPDS2Metadata {
    pub title: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub number_of_items: Option<i32>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub items_per_page: Option<i32>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub current_page: Option<i32>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub modified: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct OPDS2Publication {
    pub metadata: PublicationMetadata,
    #[serde(default)]
    pub links: Vec<OPDS2Link>,
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub images: Vec<OPDS2Image>,
}

#[derive(Debug, Serialize)]
pub struct PublicationMetadata {
    #[serde(rename = "@type")]
    pub r#type: String,              // "http://schema.org/Book"
    pub identifier: String,
    pub title: String,
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub author: Vec<NameOnly>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub publisher: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub language: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub published: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub description: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub number_of_pages: Option<i32>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub modified: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct NameOnly { pub name: String }

#[derive(Debug, Serialize)]
pub struct OPDS2Link {
    pub rel: String,
    pub href: String,
    #[serde(rename = "type")]
    pub r#type: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub title: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub templated: Option<bool>,
}

#[derive(Debug, Serialize)]
pub struct OPDS2Image {
    pub href: String,
    #[serde(rename = "type")]
    pub r#type: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub rel: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct SearchInfo {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub score: Option<f64>,
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub hits: Vec<SearchHit>,
}

#[derive(Debug, Serialize)]
pub struct SearchHit {
    pub page_num: i32,
    pub snippet: String,
}
```

### 1.2 OPDS 链接关系（rel）速查表

| rel 值 | 含义 | 使用场景 |
|--------|------|---------|
| `self` | 当前文档自身 | 每个 Feed 必须 |
| `start` | 起始目录 | 根导航 Feed |
| `subsection` | 子目录 | 导航到子分类 |
| `http://opds-spec.org/acquisition` | 获取文件 | 下载 PDF |
| `http://opds-spec.org/image/thumbnail` | 缩略图 | 封面图片 |
| `search` | 搜索 | OpenSearch 描述文档 |
| `next` / `prev` / `first` / `last` | 分页 | Acquisition Feed |

> **注意**：`sync/progress`、`sync/bookmark` 等同步 rel **不属于 OPDS 标准**。本项目进度同步使用 OPDS Progression 草案格式（`rel="http://opds-spec.org/progression"`），高亮/注释使用 W3C Web Annotation（`rel="http://www.w3.org/ns/oa#annotationService"`），两者都是私有 API。

---

## 二、Rust Service 层实现

```rust
// src/service/opds_service.rs

use chrono::Utc;
use crate::models::opds::*;

pub struct OPDSService;

impl OPDSService {
    pub fn new() -> Self { Self }

    pub fn build_navigation_feed(&self, base_url: &str) -> OPDSFeed {
        let now = Utc::now().to_rfc3339();
        OPDSFeed {
            xmlns: "http://www.w3.org/2005/Atom".into(),
            xmlns_opds: "http://opds-spec.org/2010/catalog".into(),
            xmlns_dcterms: "http://purl.org/dc/terms/".into(),
            id: "urn:uuid:pdf-manager-root".into(),
            title: "我的 PDF 图书馆".into(),
            updated: now.clone(),
            author: AtomAuthor { name: "PDF Manager".into() },
            links: vec![
                AtomLink {
                    href: format!("{}/opds/catalog", base_url), rel: "self".into(),
                    r#type: "application/atom+xml;profile=opds-catalog;kind=navigation".into(), title: None,
                },
                AtomLink {
                    href: format!("{}/opds/search", base_url), rel: "search".into(),
                    r#type: "application/opensearchdescription+xml".into(), title: None,
                },
            ],
            entries: vec![
                OPDSEntry {
                    title: "最近添加".into(), id: "urn:uuid:recent".into(),
                    summary: Some("最近 30 天上传的文档".into()),
                    content: Some(AtomContent { r#type: "text".into(), text: "最近 30 天上传的文档".into() }),
                    links: vec![AtomLink {
                        href: format!("{}/opds/catalog/recent", base_url), rel: "subsection".into(),
                        r#type: "application/atom+xml;profile=opds-catalog;kind=acquisition".into(), title: None,
                    }],
                    updated: Some(now), publisher: None, issued: None, language: None, authors: vec![],
                },
                OPDSEntry {
                    title: "所有文档".into(), id: "urn:uuid:all".into(),
                    summary: None, content: None,
                    links: vec![AtomLink {
                        href: format!("{}/opds/catalog/all", base_url), rel: "subsection".into(),
                        r#type: "application/atom+xml;profile=opds-catalog;kind=acquisition".into(), title: None,
                    }],
                    updated: None, publisher: None, issued: None, language: None, authors: vec![],
                },
            ],
        }
    }

    pub fn build_catalog_v2(&self, base_url: &str, documents: &[Document]) -> OPDS2Catalog {
        let publications: Vec<OPDS2Publication> = documents.iter().map(|doc| {
            OPDS2Publication {
                metadata: PublicationMetadata {
                    r#type: "http://schema.org/Book".into(),
                    identifier: format!("urn:uuid:{}", doc.uuid),
                    title: doc.title.clone(),
                    author: doc.authors.iter().map(|a| NameOnly { name: a.clone() }).collect(),
                    publisher: doc.publisher.clone(),
                    language: doc.language.clone(),
                    published: doc.issued_date.clone(),
                    description: doc.description.clone(),
                    number_of_pages: Some(doc.page_count),
                    modified: Some(doc.updated_at.to_rfc3339()),
                },
                links: vec![OPDS2Link {
                    rel: "http://opds-spec.org/acquisition".into(),
                    href: format!("{}/opds/acquisition/{}", base_url, doc.id),
                    r#type: "application/pdf".into(),
                    title: None, templated: None,
                }],
                images: vec![OPDS2Image {
                    href: format!("{}/api/v1/documents/{}/cover", base_url, doc.id),
                    r#type: "image/jpeg".into(),
                    rel: Some("http://opds-spec.org/image/thumbnail".into()),
                }],
            }
        }).collect();

        OPDS2Catalog {
            metadata: OPDS2Metadata {
                title: "我的 PDF 图书馆".into(),
                number_of_items: Some(documents.len() as i32),
                modified: Some(Utc::now().to_rfc3339()),
                items_per_page: None, current_page: None,
            },
            links: vec![
                OPDS2Link {
                    rel: "self".into(), href: format!("{}/opds/v2/catalog", base_url),
                    r#type: "application/opds+json".into(), title: None, templated: None,
                },
                OPDS2Link {
                    rel: "search".into(), href: format!("{}/opds/search?q={{query}}&page={{page}}", base_url),
                    r#type: "application/opds+json".into(), title: None, templated: Some(true),
                },
            ],
            publications,
            search_info: None,
        }
    }
}
```

---

## 三、Handler 层实现

```rust
// src/handler/opds_handler.rs

use axum::{
    extract::{State, Query, Path},
    http::{StatusCode, header},
    response::IntoResponse,
    Json,
};
use serde::Deserialize;
use crate::{AppState, AppError};

#[derive(Deserialize)]
pub struct CatalogParams {
    pub page: Option<i32>,
    pub limit: Option<i32>,
    pub q: Option<String>,
}

/// OPDS 1.2 根导航
pub async fn navigation_feed(
    State(state): State<AppState>,
) -> Result<impl IntoResponse, AppError> {
    let feed = state.services.opds.build_navigation_feed(
        &format!("http://{}", state.base_url)
    );
    let xml = quick_xml::se::to_string(&feed)?;
    Ok((StatusCode::OK, [(header::CONTENT_TYPE, "application/atom+xml")], xml))
}

/// OPDS 1.2 书目列表
pub async fn acquisition_feed(
    State(state): State<AppState>,
    Query(params): Query<CatalogParams>,
) -> Result<impl IntoResponse, AppError> {
    let page = params.page.unwrap_or(1);
    let limit = params.limit.unwrap_or(20);

    let (docs, total) = state.services.document.list(page, limit).await?;
    let feed = state.services.opds.build_acquisition_feed(&docs, page, limit, total);

    let xml = quick_xml::se::to_string(&feed)?;
    Ok((StatusCode::OK, [(header::CONTENT_TYPE, "application/atom+xml")], xml))
}

/// OPDS 2.0 JSON-LD
pub async fn catalog_v2(
    State(state): State<AppState>,
) -> Result<Json<OPDS2Catalog>, AppError> {
    let docs = state.services.document.list_all().await?;
    let catalog = state.services.opds.build_catalog_v2(
        &format!("http://{}", state.base_url), &docs
    );
    Ok(Json(catalog))
}

/// PDF 下载（302 重定向到 S3 预签名 URL）
pub async fn download(
    State(state): State<AppState>,
    Path(doc_id): Path<i32>,
) -> Result<impl IntoResponse, AppError> {
    let presigned_url = state.services.document.get_presigned_url(doc_id).await?;
    Ok(axum::response::Redirect::temporary(&presigned_url))
}
```

---

## 三-B、Progression Handler（OPDS Progression 草案）

```rust
// src/handler/progression_handler.rs

use axum::{extract::{State, Query, Path}, Json};
use serde::Deserialize;

#[derive(Debug, Serialize, Deserialize)]
pub struct ProgressionDoc {
    pub title: Option<String>,
    pub modified: String,                          // RFC3339
    pub device: ProgressionDevice,
    pub progression: f64,                          // 0.0 ~ 1.0
    pub references: Vec<String>,                   // ["#page=42"]
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ProgressionDevice {
    pub id: String,                                // "urn:uuid:..."
    pub name: String,                              // "PDF Manager iOS"
}

#[derive(Deserialize)]
pub struct SinceQuery {
    pub since: Option<String>,                     // RFC3339 timestamp
}

/// 获取单个文档进度（OPDS Progression 格式）
pub async fn get(
    State(state): State<AppState>,
    Path(doc_id): Path<i32>,
) -> Result<Json<ProgressionDoc>, AppError> {
    let prog = state.services.progression.get(doc_id).await?;
    Ok(Json(prog))
}

/// 更新进度
pub async fn update(
    State(state): State<AppState>,
    Path(doc_id): Path<i32>,
    Json(body): Json<ProgressionDoc>,
) -> Result<Json<serde_json::Value>, AppError> {
    state.services.progression.update(doc_id, &body).await?;
    Ok(Json(serde_json::json!({"status": "synced", "modified": body.modified})))
}

/// 增量同步：获取自某个时间点以来的所有进度变更
pub async fn list_all(
    State(state): State<AppState>,
    Query(params): Query<SinceQuery>,
) -> Result<Json<Vec<ProgressionDoc>>, AppError> {
    let since = params.since.as_deref().unwrap_or("1970-01-01T00:00:00Z");
    let progressions = state.services.progression.list_since(since).await?;
    Ok(Json(progressions))
}
```

---

## 四、中间件实现

```rust
// src/middleware/auth.rs

use axum::{
    extract::{State, Request},
    http::StatusCode,
    middleware::Next,
    response::Response,
};
use crate::{AppState, models::UserContext, util::errors::AppError};

/// Session Token 认证中间件（详见 15-认证与安全.md）
/// 从 Cookie / Authorization Header 提取 Session Token，Redis 验证后注入 UserContext
pub async fn require_auth(
    State(state): State<AppState>,
    mut req: Request,
    next: Next,
) -> Result<Response, AppError> {
    let token = extract_session_token(&req)?;

    let session_key = format!("session:{token}");
    let user_json: String = state.redis.get(&session_key).await
        .map_err(|_| AppError::Unauthorized("未登录或会话已过期".into()))?;

    let user: UserContext = serde_json::from_str(&user_json)
        .map_err(|_| AppError::Unauthorized("会话数据异常".into()))?;

    // 续期：每次请求刷新 TTL
    let _ = state.redis.expire(&session_key, 604800).await; // 7 天

    req.extensions_mut().insert(user);
    Ok(next.run(req).await)
}
```

---

## 五、数据库补充字段（与 Go 版相同，语言无关）

```sql
-- 迁移脚本: 004_opds_metadata.sql
ALTER TABLE documents ADD COLUMN IF NOT EXISTS uuid UUID DEFAULT gen_random_uuid();
ALTER TABLE documents ADD COLUMN IF NOT EXISTS publisher VARCHAR(255);
ALTER TABLE documents ADD COLUMN IF NOT EXISTS issued_date VARCHAR(50);
ALTER TABLE documents ADD COLUMN IF NOT EXISTS authors TEXT[];
ALTER TABLE documents ADD COLUMN IF NOT EXISTS cover_s3_key VARCHAR(500);
ALTER TABLE documents ADD COLUMN IF NOT EXISTS tags TEXT[];

CREATE TABLE IF NOT EXISTS reading_progress (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    document_id INTEGER NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    page_num INTEGER NOT NULL DEFAULT 1,
    percentage REAL NOT NULL DEFAULT 0,
    device_id VARCHAR(64),
    device_name VARCHAR(100),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, document_id)
);
```

---

## 六、验证清单

- [ ] `GET /opds/catalog` 返回合法 Atom XML（KOReader 可解析）
- [ ] `GET /opds/search` 返回合法 OpenSearchDescription XML
- [ ] `GET /opds/catalog/search?q=人工智能` 中文搜索正确
- [ ] `GET /opds/acquisition/{id}` 返回 PDF 或 302 重定向
- [ ] `GET /opds/v2/catalog` 返回合法 JSON-LD
- [ ] **自定义同步 API** `PUT /api/v1/progression/{id}` 进度持久化
- [ ] 增量同步 `GET /api/v1/progression?since=...` 返回变更列表
- [ ] KOReader Sync Server 可自建实现多设备进度同步（可选）

---

## 七、同步方案补充（非 OPDS）

> **OPDS 标准不含同步功能。** 以下为可选的第三方同步方案。

### 7.1 KOReader Sync Protocol（推荐）

KOReader 有专属的轻量同步 API，通过 Docker 自建：

```yaml
# docker-compose.yml 追加
  koreader-sync:
    image: ghcr.io/koreader/koreader-sync-server:latest
    container_name: pdf-koreader-sync
    ports:
      - "3030:3030"
    environment:
      KOSYNC_DB_DRIVER: SQLite
      KOSYNC_DB_PATH: /data/kosync.db
    volumes:
      - koreader_sync:/data
```

**作用**：用户在 KOReader 中同时配置：
1. OPDS 地址 → `https://your-server/opds/catalog`（浏览+下载 PDF）
2. 同步地址 → `https://your-server:3030`（进度秒级同步）

### 7.2 WebDAV（最通用）

WebDAV 被绝大多数阅读器支持（静读天下、KOReader、KyBook 3 等）：

```yaml
  webdav:
    image: bytemark/webdav
    container_name: pdf-webdav
    environment:
      AUTH_TYPE: Digest
      USERNAME: reader
      PASSWORD: ${WEBDAV_PASSWORD:-change_me}
    volumes:
      - webdav_data:/var/lib/dav
    ports:
      - "8081:80"
```

用户在各阅读器中填入 WebDAV 地址即可自动同步进度文件。

> **重要限制**：不同阅读器之间的进度不互通（静读天下的进度文件格式 ≠ KOReader 的格式）。跨阅读器的同步只能通过本项目的自建私有 API。

### 7.3 同步方案选择

| 场景 | 方案 |
|------|------|
| 只用自己的 Flutter 客户端 | 私有 API `PUT /api/v1/progression/{id}`（OPDS Progression 格式） |
| 同时用 KOReader | KOReader Sync Server（进度） + OPDS（下载） |
| 同时用多种第三方阅读器 | WebDAV（各大阅读器各自同步自己的进度文件） |
| 统一管理所有标记（高亮/注释） | 只能通过私有 API，第三方阅读器不支持 |

---

## 📎 补充规范（参考 other/ 开源项目实践）

> 以下内容从 OPDS Drafts（`drafts/opds-progression-1.0.md`、`drafts/authentication-for-opds-1.0.md`）、
> Komga（`interfaces/api/kosync/`）、Readium Architecture（`server/`）中提炼。

### 8. OPDS Progression references 字段

> 来源：`drafts/opds-progression-1.0.md`

OPDS Progression 1.0 草案定义了 `references` 字段，用于在 PDF 等文档中提供更精确的位置引用。

#### 8.1 支持的引用格式

| 文档类型 | 引用格式 | 示例 |
|----------|---------|------|
| PDF | `#page=N` | `#page=42` |
| HTML | `#id` | `chapter1.html#section2` |
| 文本片段 | `#:~:text=...` | `#:~:text=关键词` |
| 音视频 | `#t=秒数` | `#t=67` |

#### 8.2 Rust 模型补充

```rust
// src/models/sync.rs

#[derive(Debug, Serialize, Deserialize)]
pub struct ProgressionUpdate {
    /// 章节标题（可选，用于上下文识别）
    #[serde(skip_serializing_if = "Option::is_none")]
    pub title: Option<String>,

    /// 最后阅读时间
    pub modified: chrono::DateTime<chrono::Utc>,

    /// 设备信息
    pub device: DeviceInfo,

    /// 总进度百分比 (0.0 ~ 1.0)
    pub progression: f64,

    /// 精确位置引用（URI 数组）
    /// 例如：["#page=42"] 或 ["chapter1.html#section2"]
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub references: Vec<String>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct DeviceInfo {
    /// 设备唯一标识（URI 格式，如 urn:uuid:...）
    pub id: String,
    /// 设备显示名称
    pub name: String,
}
```

#### 8.3 响应示例

```json
{
    "title": "第3章 - 程序的机器级表示",
    "modified": "2024-07-15T14:30:00Z",
    "device": {
        "id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
        "name": "KOReader (Pixel 8)"
    },
    "progression": 0.345,
    "references": ["#page=187"]
}
```

#### 8.4 自动发现

在 OPDS 2.0 Feed 中，Progression 服务通过 `rel` 和 `type` 自动发现：

```json
{
    "href": "/api/v1/progression/101",
    "type": "application/opds-progression+json",
    "rel": "http://opds-spec.org/progression"
}
```

---

### 9. OPDS 认证发现协议

> 来源：`drafts/authentication-for-opds-1.0.md`

OPDS 认证规范要求服务端提供 `application/opds-authentication+json` 发现文档，
让客户端自动识别认证方式，无需硬编码。

#### 9.1 认证发现端点

```rust
// src/handler/opds_handler.rs

/// GET /opds/authentication
/// 返回认证发现文档，**不需要认证**即可访问
pub async fn auth_descriptor() -> Json<serde_json::Value> {
    Json(serde_json::json!({
        "id": "urn:uuid:pdf-manager-auth",
        "title": "PDF Manager",
        "description": "登录以访问你的 PDF 图书馆",
        "links": [
            {
                "rel": "logo",
                "href": "/assets/logo.png",
                "type": "image/png"
            },
            {
                "rel": "help",
                "href": "https://your-server.com/help",
                "type": "text/html"
            },
            {
                "rel": "register",
                "href": "/api/v1/auth/register",
                "type": "application/json"
            }
        ],
        "authentication": [
            {
                "type": "http://opds-spec.org/auth/oauth/password",
                "links": [
                    {
                        "rel": "authenticate",
                        "href": "/api/v1/auth/login",
                        "type": "application/json"
                    }
                ]
            }
        ]
    }))
}
```

#### 9.2 在 OPDS Feed 中引用

需要认证的链接通过 `properties.authenticate` 指向发现文档：

```json
{
    "href": "/api/v1/progression/101",
    "type": "application/opds-progression+json",
    "rel": "http://opds-spec.org/progression",
    "properties": {
        "authenticate": {
            "href": "/opds/authentication",
            "type": "application/opds-authentication+json"
        }
    }
}
```

#### 9.3 路由注册

```rust
// src/handler/mod.rs — 认证发现不需要 JWT
let opds = Router::new()
    .route("/opds/authentication", get(opds_handler::auth_descriptor))  // 无需认证
    .route("/opds/catalog", get(opds_handler::navigation_feed))         // 需要认证
    .route("/opds/v2/catalog", get(opds_handler::catalog_v2));          // 需要认证
```

---

### 10. KOReader Sync 协议详细实现

> 来源：Komga `interfaces/api/kosync/` + Calibre-Web `kobo.py` / `kobo_sync_status.py`

#### 10.1 协议核心接口

```rust
// src/handler/kosync_handler.rs

/// KOReader Sync 协议接口
///
/// 注意：KOReader 使用 MD5 哈希标识文档（非 SHA256）
/// 需要在上传时同时计算 MD5 和 SHA256 两种哈希

/// POST /kosync/sync — 上报阅读进度
#[derive(Debug, Deserialize)]
pub struct KoSyncPushRequest {
    /// 文档的 MD5 哈希
    pub document: String,
    /// 进度字符串（KOReader 内部格式，因文件类型而异）
    pub progress: String,
    /// 进度百分比 (0.0 ~ 1.0)
    pub percentage: f64,
    /// 设备标识（显示名称）
    pub device: String,
    /// 设备 ID（唯一标识）
    pub device_id: String,
}

/// GET /kosync/sync?document={md5} — 拉取阅读进度
#[derive(Debug, Serialize)]
pub struct KoSyncPullResponse {
    pub document: String,
    pub progress: String,
    pub percentage: f64,
    pub updated_at: String,  // ISO 8601
}
```

#### 10.2 KOReader 认证

KOReader Sync 使用独立的 HTTP Basic 风格认证（非 JWT）：

```rust
/// KOReader 使用 X-Auth-User + X-Auth-Key 头认证
/// X-Auth-Key = MD5(password)
pub async fn kosync_auth(headers: HeaderMap) -> Result<i32, AppError> {
    let username = headers.get("X-Auth-User")
        .and_then(|v| v.to_str().ok())
        .ok_or_else(|| AppError::Unauthorized("Missing X-Auth-User".into()))?;

    let key = headers.get("X-Auth-Key")
        .and_then(|v| v.to_str().ok())
        .ok_or_else(|| AppError::Unauthorized("Missing X-Auth-Key".into()))?;

    // 验证用户名和 key（key 是密码的 MD5）
    let user_id = verify_kosync_credentials(username, key).await?;
    Ok(user_id)
}
```

#### 10.3 双哈希存储

为兼容 KOReader，文档需要同时存储 MD5 和 SHA256：

```rust
// src/models/document.rs

#[derive(Debug, Serialize, Deserialize)]
pub struct Document {
    pub id: i32,
    pub user_id: i32,
    pub title: String,
    pub s3_key: String,
    pub sha256: String,     // 用于文件去重和标准校验
    pub md5: String,        // 用于 KOReader Sync 兼容
    pub page_count: i32,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

#### 10.4 路由注册

```rust
// KOReader Sync 兼容层路由（独立认证中间件）
let kosync = Router::new()
    .route("/kosync/sync", post(kosync_handler::push_progress))
    .route("/kosync/sync", get(kosync_handler::pull_progress))
    .layer(middleware::from_fn(kosync_handler::kosync_auth_middleware));
```

#### 10.5 KOReader Sync vs 私有 API 对照

| 特性 | KOReader Sync | 私有 API (OPDS Progression) |
|------|:---:|:---:|
| 认证方式 | X-Auth-User/Key (MD5) | Bearer Session Token |
| 文档标识 | MD5 哈希 | 文档 ID |
| 进度格式 | 字符串 (KOReader 内部格式) | 百分比 (0.0~1.0) |
| 高亮/注释 | ❌ 不支持 | ✅ Web Annotation |
| 设备识别 | device + device_id | DeviceInfo (id + name) |
| 精确位置引用 | ❌ | ✅ references 数组 |

---

### 11. HTTP 缓存策略（OPDS 场景）

> 来源：Readium Architecture `server/caching.md`

Publication Server 规范要求对不同类型的资源采用不同的缓存策略：

| 资源类型 | 策略 | Header 示例 |
|----------|------|------------|
| OPDS Feed / 目录 | ETag + 304 重验证 | `ETag: "abc123"`, `Cache-Control: private, no-cache` |
| PDF 文件下载 | 强缓存 | `Cache-Control: public, max-age=86400` |
| 封面图片 | 强缓存 + ETag | `Cache-Control: public, max-age=86400`, `ETag: "..."` |
| 认证发现文档 | 不缓存 | `Cache-Control: no-store` |
| API 响应 | 不缓存或短缓存 | `Cache-Control: private, max-age=60` |

**Manifest（目录）缓存要点**：
- 用 manifest 内容的 hash 作为 ETag
- 不使用 `Cache-Control`、`Last-Modified` 或 `Expires`
- 必须响应 `If-None-Match`，匹配时返回 `304 Not Modified`

**静态资源缓存要点**：
- 使用 `Cache-Control: public, max-age=86400`（至少 24 小时）
- 字体、CSS、JS 等资源的缓存对 HTML 渲染性能有显著正面影响
