# Rust 项目结构 — 代码组织和模块划分

> Go 版本见 `../go/06-Go项目结构.md`，两者已充分对照。

---

## 🧩 三项目总览

系统由三个**独立 Git 仓库**组成，互不依赖：

| 项目 | 仓库名 | 技术栈 | 定位 |
|------|--------|--------|------|
| **服务端** | `book-server` | Rust/Axum + PG + S3 | API + 轻量 PDF 处理 + 搜索 |
| **导入 CLI** | `book-import` | Rust CLI | 重型处理 (OCR/EPUB/大文件) → .bookpkg |
| **阅读器** | `book-reader` | Flutter + Rust FFI | 跨平台阅读 + 本地离线搜索 |

三者唯一联系：`.bookpkg` 压缩包格式。详见 `14-导入CLI工具.md`。

---

## 📁 服务端项目目录树

```
pdf-manager/
│
├── src/                      # Rust 源码
│   ├── main.rs               # 应用入口（<50行）
│   │
│   ├── config/               # 配置管理
│   │   ├── mod.rs
│   │   ├── app_config.rs     # 配置结构和加载 (serde_yaml)
│   │   ├── database.rs       # PG 连接池 (sqlx::PgPool)
│   │   └── logger.rs         # 日志/追踪初始化 (tracing)
│   │
│   ├── models/               # 数据模型
│   │   ├── mod.rs
│   │   ├── user.rs           # 用户模型
│   │   ├── document.rs       # 文档模型
│   │   ├── bookmark.rs       # 标记模型（高亮/注释/书签）
│   │   ├── search.rs         # 搜索模型
│   │   ├── segment.rs        # 分词模型
│   │   └── opds.rs           # OPDS 数据模型（Feed/Entry/Link）
│   │
│   ├── repository/           # 数据访问层
│   │   ├── mod.rs
│   │   ├── user_repo.rs      # 用户仓库
│   │   ├── document_repo.rs  # 文档仓库
│   │   ├── bookmark_repo.rs  # 标记仓库
│   │   ├── search_repo.rs    # 搜索仓库
│   │   └── sync_repo.rs      # 阅读进度/同步仓库
│   │
│   ├── service/              # 业务逻辑层
│   │   ├── mod.rs
│   │   ├── auth_service.rs   # 认证服务
│   │   ├── document_service.rs  # 文档处理服务
│   │   ├── bookmark_service.rs  # 标记管理服务
│   │   ├── search_service.rs    # 搜索协调服务
│   │   ├── segment_service.rs   # 分词服务（jieba-rs）
│   │   ├── s3_service.rs        # S3 操作服务（上传/删除/预签名）
│   │   ├── opds_service.rs      # OPDS Feed 编解码服务
│   │   └── sync_service.rs      # 阅读进度同步服务
│   │
│   ├── handler/              # HTTP 处理层（Axum Router）
│   │   ├── mod.rs
│   │   ├── auth_handler.rs   # 认证接口
│   │   ├── document_handler.rs  # 文档接口
│   │   ├── bookmark_handler.rs  # 标记接口
│   │   ├── search_handler.rs    # 搜索接口
│   │   ├── admin_handler.rs     # 后台管理接口
│   │   └── opds_handler.rs      # OPDS 协议处理
│   │
│   ├── middleware/           # Axum 中间件
│   │   ├── mod.rs
│   │   ├── auth.rs           # Session + API Key 认证中间件 (from_fn)
│   │   ├── csrf.rs           # CSRF 防护中间件 (Double Submit Cookie)
│   │   ├── error.rs          # 错误处理中间件
│   │   ├── cors.rs           # CORS 中间件
│   │   └── rate_limit.rs     # 全局限流 (tower-governor)
│   │
│   ├── util/                 # 工具函数
│   │   ├── mod.rs
│   │   ├── errors.rs         # AppError 枚举 (thiserror) → HTTP 状态码映射
│   │   ├── log_mask.rs       # 日志脱敏（密码/token/email）
│   │   ├── response.rs       # 标准 JSON 响应
│   │   ├── validator.rs      # 数据验证 (validator derive)
│   │   ├── segment.rs        # jieba-rs 分词封装
│   │   ├── segment_cache.rs  # 分词缓存
│   │   ├── presigned_cache.rs # S3 签名 URL 缓存（Redis → PG → SDK）
│   │   ├── pdf.rs            # PDF 处理 (lopdf + pdf-extract)
│   │   ├── hash.rs           # 哈希和加密
│   │   └── memory.rs         # 内存监控
│   │
│   └── db/                   # 数据库相关
│       └── migrations/       # SQL 迁移脚本
│           ├── 001_init_schema.sql
│           ├── 002_add_indexes.sql
│           └── 003_add_constraints.sql
│
├── tests/                    # 集成测试（L3）
│   ├── common/
│   │   └── mod.rs            # setup_test_app() + setup_test_db()
│   ├── auth_test.rs          # 注册/登录/me API 测试
│   ├── document_test.rs      # 文档上传/列表 API 测试
│   ├── file_test.rs          # 文件代理 /files/{key} 测试
│   └── fixtures/
│       └── test_data.sql
│
├── scripts/                  # 脚本
│   ├── init-db.sh
│   ├── seed.sh
│   └── migrate.sh
│
├── config/                   # 配置文件
│   ├── app.yaml              # 应用配置（本地）
│   ├── app.prod.yaml         # 生产配置
│   ├── custom_dict.txt       # 自定义词库
│   ├── stopwords.txt         # 停用词表
│   └── sonic.cfg             # Sonic 搜索引擎配置
│
├── web/                      # 前端项目（不变）
│
├── docker-compose.yml
├── Dockerfile                # Rust 多阶段构建
├── Cargo.toml                # Rust 依赖
├── Cargo.lock                # 依赖锁定
├── .env.example
├── .gitignore
├── README.md
└── Makefile
```

---

## 📄 各文件职责说明

### src/main.rs（应用入口，<50行）

```rust
use axum::Router;
use tower_http::{cors::CorsLayer, trace::TraceLayer};
use tracing::info;

mod config;
mod handler;
mod middleware;
mod models;
mod repository;
mod service;
mod util;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. 加载配置
    let cfg = config::load()?;
    config::init_logger(&cfg);

    // 2. 初始化数据库连接池
    let pool = config::init_db(&cfg).await?;
    sqlx::migrate!("./src/db/migrations")
        .run(&pool).await?;

    // 3. 初始化 Redis
    let redis = config::init_redis(&cfg)?;

    // 4. 初始化 S3 客户端
    let s3 = config::init_s3(&cfg).await;

    // 5. 初始化分词引擎
    let segmenter = service::segment_service::init_segmenter(&cfg)?;

    // 6. 组装 AppState
    let state = AppState { pool, redis, s3, segmenter };

    // 7. 构建路由
    let app = handler::build_routes(state)
        .layer(CorsLayer::permissive())
        .layer(TraceLayer::new_for_http());

    // 8. 启动服务器
    let addr = format!("0.0.0.0:{}", cfg.server.port);
    info!("Server listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(&addr).await?;
    axum::serve(listener, app).await?;

    Ok(())
}

pub struct AppState {
    pub pool: sqlx::PgPool,
    pub redis: redis::aio::ConnectionManager,
    pub s3: aws_sdk_s3::Client,
    pub segmenter: service::Segmenter,
}
```

### handler/mod.rs（路由注册）

```rust
use axum::{middleware, Router, routing::{get, post, put, delete}};
use super::AppState;

pub fn build_routes(state: AppState) -> Router {
    // ===== OPDS 主干（标准协议）=====
    let opds = Router::new()
        .route("/opds/catalog", get(opds_handler::navigation_feed))
        .route("/opds/catalog/{*path}", get(opds_handler::acquisition_feed))
        .route("/opds/search", get(opds_handler::search_descriptor))
        .route("/opds/catalog/search", get(opds_handler::search_feed))
        .route("/opds/v2/catalog", get(opds_handler::catalog_v2))
        .route("/opds/v2/catalog/search", get(opds_handler::search_v2))
        .route("/opds/acquisition/{id}", get(opds_handler::download))
        .route("/opds/acquisition/{id}", get(opds_handler::download));

    // ===== 进度同步（OPDS Progression 草案格式）=====
    let progression = Router::new()
        .route("/api/v1/progression", get(progression_handler::list_all))         // GET ?since=
        .route("/api/v1/progression/{id}",
            get(progression_handler::get).put(progression_handler::update));

    // ===== 标记（W3C Web Annotation + Locator fragments）=====
    let annotations = Router::new()
        .route("/api/v1/annotations", get(annotation_handler::list_by_doc))       // GET ?document_id=&since=
        .route("/api/v1/annotations/{id}",
            post(annotation_handler::create)
            .put(annotation_handler::update)
            .delete(annotation_handler::delete));

    // ===== 认证（扩展 API）=====
    let auth = Router::new()
        .route("/api/v1/auth/register", post(auth_handler::register))
        .route("/api/v1/auth/login", post(auth_handler::login))
        .route("/api/v1/auth/logout", post(auth_handler::logout))
        .route("/api/v1/auth/me", get(auth_handler::me))
        .route("/api/v1/auth/password", put(auth_handler::change_password));

    // ===== API Key 管理（需认证）=====
    let api_keys = Router::new()
        .route("/api/v1/auth/api-keys",
            get(auth_handler::list_api_keys).post(auth_handler::create_api_key))
        .route("/api/v1/auth/api-keys/{id}", delete(auth_handler::delete_api_key))
        .route_layer(middleware::from_fn_with_state(
            state.clone(),
            middleware::auth::require_auth,
        ));

    // ===== 需认证的扩展 API（标记已统一为 W3C Web Annotation）=====
    let api = Router::new()
        .route("/api/v1/documents/upload", post(document_handler::upload))
        .route("/api/v1/documents/{id}/status", get(document_handler::status))
        .route("/api/v1/dictionaries",
            get(admin_handler::list_dict)
            .post(admin_handler::add_word))
        .route("/api/v1/dictionaries/{id}", delete(admin_handler::delete_word))
        .route_layer(middleware::from_fn_with_state(
            state.clone(),
            middleware::auth::require_auth,
        ));

    // ===== 管理员路由 =====
    let admin = Router::new()
        .route("/api/v1/admin/segmenter",
            get(admin_handler::get_segmenter_config)
            .put(admin_handler::update_segmenter_config))
        .route("/api/v1/admin/search-engine",
            get(admin_handler::get_search_config)
            .put(admin_handler::update_search_config))
        .route("/api/v1/admin/segmenter/test", post(admin_handler::test_segment))
        .route("/api/v1/admin/metrics", get(admin_handler::metrics))
        .route_layer(middleware::from_fn_with_state(
            state.clone(),
            middleware::auth::require_admin,
        ));

    // ===== 健康检查 =====
    let health = Router::new()
        .route("/health", get(|| async { axum::Json(serde_json::json!({"status": "ok"})) }));

    Router::new()
        .merge(opds)
        .merge(progression)
        .merge(annotations)
        .merge(auth)
        .merge(api_keys)
        .merge(api)
        .merge(admin)
        .merge(health)
        .with_state(state)
}
```

### handler/opds_handler.rs（OPDS Handler 示例）

```rust
use axum::{extract::{State, Query, Path}, http::StatusCode, response::IntoResponse, Json};
use serde::Deserialize;
use super::AppState;
use crate::util::errors::AppError;

#[derive(Deserialize)]
pub struct CatalogQuery {
    pub page: Option<i32>,
    pub limit: Option<i32>,
}

pub async fn navigation_feed(
    State(state): State<AppState>,
) -> Result<axum::response::Response, AppError> {
    let feed = state.services.opds.build_navigation_feed();
    Ok((
        StatusCode::OK,
        [("content-type", "application/atom+xml")],
        quick_xml::se::to_string(&feed)?
    ).into_response())
}

pub async fn acquisition_feed(
    State(state): State<AppState>,
    Query(params): Query<CatalogQuery>,
    axum::extract::RequestParts(parts): _,
) -> Result<impl IntoResponse, AppError> {
    let page = params.page.unwrap_or(1);
    let limit = params.limit.unwrap_or(20);

    let (docs, total) = state.services
        .document
        .list_documents(page, limit)
        .await?;

    let feed = state.services.opds
        .build_acquisition_feed(&docs, page, limit, total);

    Ok((
        StatusCode::OK,
        [("content-type", "application/atom+xml")],
        quick_xml::se::to_string(&feed)?
    ))
}
```

---

## 📦 Cargo.toml 依赖

```toml
[package]
name = "pdf-manager"
version = "0.1.0"
edition = "2021"

[features]
default = []
tantivy-search = ["tantivy"]  # 开启第 3 层 Tantivy 搜索

[dependencies]
# 异步运行时
tokio = { version = "1", features = ["full"] }

# Web 框架
axum = "0.7"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace", "auth"] }

# 数据库
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "chrono", "uuid", "migrate"] }

# Redis
redis = { version = "0.25", features = ["tokio-comp", "connection-manager"] }

# S3
aws-sdk-s3 = "1"
aws-config = "1"

# 中文分词
jieba-rs = "0.7"

# 嵌入式全文搜索（第 3 层，可选）
tantivy = { version = "0.22", optional = true }

# PDF 处理
lopdf = "0.34"
pdf-extract = "0.8"

# JWT
jsonwebtoken = "9"

# 序列化
serde = { version = "1", features = ["derive"] }
serde_json = "1"
quick-xml = { version = "0.36", features = ["serialize"] }  # OPDS Atom XML

# 配置
config = "0.14"
serde_yaml = "0.9"

# 日志
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# 密码哈希
bcrypt = "0.15"

# UUID
uuid = { version = "1", features = ["v4", "serde"] }

# 时间
chrono = { version = "0.4", features = ["serde"] }

# 错误处理
anyhow = "1"
thiserror = "2"

# 参数校验
validator = { version = "0.18", features = ["derive"] }

# 环境变量
dotenvy = "0.15"

[dev-dependencies]
axum-test = "15"
tokio-test = "0.4"
```

---

## 🔄 Go → Rust 文件映射速查

| Go 文件 | Rust 文件 | 关键变化 |
|---------|----------|---------|
| `cmd/main/main.go` | `src/main.rs` | `gin` → `axum`, `defer` → `Drop` |
| `internal/handler/opds_handler.go` | `src/handler/opds_handler.rs` | `c.JSON()` → `axum::Json()` |
| `internal/service/opds_service.go` | `src/service/opds_service.rs` | 结构体方法 → `impl` 块 |
| `internal/middleware/auth.go` | `src/middleware/auth.rs` | Gin 中间件 → Axum `from_fn` |
| `internal/util/errors.go` | `src/util/errors.rs` | 自定义 error → `thiserror::Error` derive |
| `internal/util/segment.go` | `src/util/segment.rs` | `gse/jieba` → `jieba-rs` |
| `internal/repository/user_repo.go` | `src/repository/user_repo.rs` | `database/sql` → `sqlx::query_as!` |
| `go.mod` | `Cargo.toml` | Go modules → Cargo crates |

---

## 📎 补充：项目结构增强（参考 other/ 开源项目实践）

> 以下文件结构从 Kavita（C#）、Komga（Kotlin）等项目提炼，补充到现有项目结构中。

### 新增文件清单

```
src/
├── util/
│   ├── events.rs             ← 新增：领域事件总线（参考 Komga ApplicationEventPublisher）
│   └── cache.rs              ← 新增：缓存辅助函数（参考 Readium Architecture）
├── service/
│   └── document_lifecycle.rs ← 新增：文档生命周期管理（参考 Komga BookLifecycle）
├── handler/
│   └── kosync_handler.rs     ← 新增：KOReader Sync 兼容层（参考 Komga kosync/）
└── middleware/
    └── cache.rs              ← 新增：ETag 缓存中间件（参考 Readium server/caching.md）
```

### AppState 扩展

```rust
// src/main.rs — 在现有 AppState 中新增事件总线

pub struct AppState {
    pub pool: sqlx::PgPool,
    pub redis: redis::aio::ConnectionManager,
    pub s3: aws_sdk_s3::Client,
    pub segmenter: service::Segmenter,
    pub event_bus: util::EventBus,  // 新增：领域事件总线
}
```

### 路由注册扩展

```rust
// src/handler/mod.rs — 新增路由

pub fn build_routes(state: AppState) -> Router {
    // ...existing routes...

    // ===== KOReader Sync 兼容层（可选）=====
    let kosync = Router::new()
        .route("/kosync/sync", post(kosync_handler::push_progress))
        .route("/kosync/sync", get(kosync_handler::pull_progress));

    Router::new()
        // ...existing routes...
        .nest("/kosync", kosync)
        .with_state(state)
}
```

### Cargo.toml 补充依赖

```toml
# src/util/events.rs — 已在 tokio features 中包含（broadcast channel）
tokio = { version = "1", features = ["full"] }

# src/handler/mod.rs — cached_file 需要 MIME 类型猜测
mime_guess = "2"

# src/middleware/cache.rs — ETag 计算
md-5 = "0.10"
```
