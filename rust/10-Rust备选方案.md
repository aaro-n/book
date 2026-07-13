# Rust 备选技术方案 — 与 Go 方案完整对照

> **定位**：本文档作为现有 Go 方案的平行备选，供团队在技术选型时对比参考。
> **前提**：前端（React + TypeScript）、数据库（PostgreSQL 18）、基础设施（S3/Redis/搜索引擎）保持不变，仅替换后端语言。

---

## 一、技术栈对照表

### 1.1 核心组件映射

| 层级 | Go 方案 | Rust 方案 | 选型理由 |
|------|---------|-----------|----------|
| **语言** | Go 1.22+ | Rust 1.80+ (stable) | 零成本抽象，无 GC |
| **异步运行时** | Goroutine (内置) | Tokio | 业界标准，生态最完善 |
| **Web 框架** | Gin v1.9+ | Axum 0.7+ | 基于 Tower 中间件，类型安全，与 Tokio 深度集成 |
| **数据库驱动** | lib/pq | sqlx 0.8+ | 编译期 SQL 校验， async 原生支持 |
| **ORM 备选** | — | SeaORM 1.0+ | 如需要 ORM 层，SeaORM 是最成熟的异步 ORM |
| **缓存客户端** | go-redis v8 | redis-rs 0.25+ | 官方客户端，async 支持 |
| **S3 客户端** | AWS SDK Go v2 | aws-sdk-s3 (Rust) | AWS 官方 Rust SDK，1.0 已发布 |
| **中文分词** | gse + jieba (Go) | jieba-rs | Rust 版结巴，成熟稳定 |
| **PDF 处理** | pdfcpu | lopdf + pdf-extract | lopdf 读/写，pdf-extract 提取文本 |
| **JWT 认证** | golang-jwt v5 | jsonwebtoken | 功能完整 |
| **日志/追踪** | zap | tracing + tracing-subscriber | 结构化日志 + 分布式追踪 |
| **序列化** | encoding/json | serde + serde_json | 编译期序列化，零成本 |
| **配置管理** | viper | config + serde_yaml | 类型安全反序列化 |
| **参数校验** | go-playground/validator | validator | derive 宏自动生成 |

### 1.2 Cargo.toml 依赖清单

```toml
[package]
name = "pdf-manager"
version = "0.1.0"
edition = "2021"

[dependencies]
# 异步运行时
tokio = { version = "1", features = ["full"] }

# Web 框架
axum = "0.7"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace", "auth"] }

# 数据库
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "chrono", "uuid"] }

# Redis
redis = { version = "0.25", features = ["tokio-comp", "connection-manager"] }

# S3
aws-sdk-s3 = "1"
aws-config = "1"
aws-credential-types = "1"

# 中文分词
jieba-rs = "0.7"

# PDF 处理
lopdf = "0.34"
pdf-extract = "0.8"

# JWT
jsonwebtoken = "9"

# 序列化
serde = { version = "1", features = ["derive"] }
serde_json = "1"

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

# 工具
dotenvy = "0.15"

[dev-dependencies]
tower = { version = "0.4", features = ["util"] }  # 测试用
axum-test = "15"                                    # 集成测试
```

---

## 二、项目结构对照

```
pdf-manager/                        pdf-manager/
├── cmd/main/main.go                ├── src/
│   (单一入口)                       │   ├── main.rs          ← 入口（<50行）
│                                   │   │
├── internal/                       │   ├── config/
│   ├── config/   (配置)             │   │   ├── mod.rs        ← 配置加载
│   ├── models/   (数据模型)         │   │   ├── database.rs   ← 数据库连接池
│   ├── repository/(数据访问)        │   │   └── logger.rs     ← 日志初始化
│   ├── service/  (业务逻辑)         │   │
│   ├── handler/  (HTTP处理)         │   ├── models/
│   ├── middleware/(中间件)          │   │   ├── mod.rs
│   ├── util/     (工具)             │   │   ├── user.rs
│   └── db/migrations/               │   │   ├── document.rs
│                                   │   │   ├── bookmark.rs
├── tests/                          │   │   └── search.rs
│                                   │   │
├── config/app.yaml                 │   ├── repository/
├── web/   (前端)                    │   │   ├── mod.rs
│                                   │   │   ├── user_repo.rs
├── Dockerfile                      │   │   ├── document_repo.rs
├── docker-compose.yml              │   │   ├── bookmark_repo.rs
├── go.mod / go.sum                 │   │   └── search_repo.rs
│                                   │   │
│                                   │   ├── service/
│                                   │   │   ├── mod.rs
│                                   │   │   ├── auth_service.rs
│                                   │   │   ├── document_service.rs
│                                   │   │   ├── bookmark_service.rs
│                                   │   │   ├── search_service.rs
│                                   │   │   ├── segment_service.rs
│                                   │   │   └── s3_service.rs
│                                   │   │
│                                   │   ├── handler/
│                                   │   │   ├── mod.rs
│                                   │   │   ├── auth_handler.rs
│                                   │   │   ├── document_handler.rs
│                                   │   │   ├── bookmark_handler.rs
│                                   │   │   ├── search_handler.rs
│                                   │   │   └── admin_handler.rs
│                                   │   │
│                                   │   ├── middleware/
│                                   │   │   ├── mod.rs
│                                   │   │   ├── auth.rs
│                                   │   │   └── error.rs
│                                   │   │
│                                   │   └── util/
│                                   │       ├── mod.rs
│                                   │       ├── errors.rs
│                                   │       ├── response.rs
│                                   │       ├── segment.rs
│                                   │       ├── pdf.rs
│                                   │       └── hash.rs
│                                   │
│                                   ├── migrations/   ← SQL 迁移
│                                   ├── tests/
│                                   ├── config/
│                                   │   ├── app.yaml
│                                   │   ├── custom_dict.txt
│                                   │   └── stopwords.txt
│                                   ├── web/          ← 前端（不变）
│                                   ├── scripts/
│                                   ├── Dockerfile
│                                   ├── docker-compose.yml
│                                   └── Cargo.toml / Cargo.lock
```

**关键差异**：
- Go 用 `internal/` 包级隔离；Rust 用 `src/` 模块树 + `pub(crate)` 可见性控制
- Go 入口在 `cmd/main/`（多入口能力）；Rust 入口在 `src/main.rs`（默认单入口）
- Go 的 `go.mod` 声明模块路径；Rust 的 `Cargo.toml` 声明 crate 元信息

---

## 三、代码实现对照

### 3.1 应用入口

| Go (`cmd/main/main.go`) | Rust (`src/main.rs`) |
|------------------------|---------------------|
| goroutine 隐式并发 | `#[tokio::main]` 显式声明运行时 |
| `panic(err)` 启动失败 | `anyhow::Result` + `?` 传播 |
| `defer db.Close()` | `Drop` trait 自动释放 |

**Go：**
```go
func main() {
    cfg, err := config.Load()
    if err != nil {
        panic(err)
    }
    db, err := config.InitDB(cfg)
    if err != nil {
        panic(err)
    }
    defer db.Close()

    segmenter, _ := initSegmenter(cfg)
    repos := repository.NewRepositories(db)
    services := service.NewServices(repos, segmenter)
    handlers := handler.NewHandlers(services)

    r := gin.Default()
    setupRoutes(r, handlers)
    r.Run(fmt.Sprintf(":%d", cfg.Server.Port))
}
```

**Rust：**
```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cfg = config::load()?;
    config::init_logger(&cfg);

    let pool = config::init_db(&cfg).await?;
    let segmenter = init_segmenter(&cfg)?;
    let redis = config::init_redis(&cfg)?;
    let s3 = config::init_s3(&cfg).await;

    let state = AppState::new(pool, redis, s3, segmenter);

    let app = router::build(state)
        .layer(tower_http::cors::CorsLayer::permissive())
        .layer(tower_http::trace::TraceLayer::new_for_http());

    let addr = format!("0.0.0.0:{}", cfg.server.port);
    tracing::info!("Server listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(&addr).await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

### 3.2 路由注册

**Go (Gin)：**
```go
func setupRoutes(r *gin.Engine, h *handler.Handlers) {
    auth := r.Group("/api/v1/auth")
    auth.POST("/register", h.Auth.Register)
    auth.POST("/login", h.Auth.Login)

    api := r.Group("/api/v1")
    api.Use(middleware.AuthRequired())
    api.POST("/documents/upload", h.Document.Upload)
}
```

**Rust (Axum)：**
```rust
pub fn build(state: AppState) -> Router {
    let public = Router::new()
        .route("/api/v1/auth/register", post(auth::register))
        .route("/api/v1/auth/login", post(auth::login))
        .route("/health", get(health_check));

    let protected = Router::new()
        .route("/api/v1/documents", post(document::upload).get(document::list))
        .route("/api/v1/documents/{id}", get(document::get_by_id))
        .route("/api/v1/search", get(search::search))
        .layer(axum::middleware::from_fn_with_state(
            state.clone(),
            auth::require_auth,
        ));

    Router::new()
        .merge(public)
        .merge(protected)
        .with_state(state)
}
```

### 3.3 Handler 实现

**Go：**
```go
func (h *AuthHandler) Register(c *gin.Context) {
    var req RegisterInput
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    user, err := h.service.Register(c.Request.Context(), &req)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    c.JSON(201, user)
}
```

**Rust：**
```rust
#[derive(Debug, Deserialize, Validate)]
pub struct RegisterInput {
    #[validate(email)]
    pub email: String,
    #[validate(length(min = 8))]
    pub password: String,
}

pub async fn register(
    State(state): State<AppState>,
    Json(input): Json<RegisterInput>,
) -> Result<(StatusCode, Json<UserResponse>), AppError> {
    input.validate()?;
    let user = state.auth_service.register(input).await?;
    Ok((StatusCode::CREATED, Json(user.into())))
}
```

### 3.4 错误处理

**Go（自定义错误类型）：**
```go
type AppError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func (e *AppError) Error() string { return e.Message }
```

**Rust（thiserror + Axum 集成）：**
```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("未找到: {0}")]
    NotFound(String),

    #[error("未授权: {0}")]
    Unauthorized(String),

    #[error("参数校验失败: {0}")]
    Validation(String),

    #[error("数据库错误: {0}")]
    Database(#[from] sqlx::Error),

    #[error("内部错误: {0}")]
    Internal(#[from] anyhow::Error),
}

impl axum::response::IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match &self {
            AppError::NotFound(m) => (StatusCode::NOT_FOUND, m.clone()),
            AppError::Unauthorized(m) => (StatusCode::UNAUTHORIZED, m.clone()),
            AppError::Validation(m) => (StatusCode::BAD_REQUEST, m.clone()),
            AppError::Database(_) => (StatusCode::INTERNAL_SERVER_ERROR, "数据库错误".into()),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "内部错误".into()),
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

### 3.5 Repository 层

**Go：**
```go
func (r *UserRepo) Create(ctx context.Context, user *models.User) error {
    return r.db.QueryRowContext(ctx,
        `INSERT INTO users (email, username, password_hash)
         VALUES ($1, $2, $3) RETURNING id, created_at`,
        user.Email, user.Username, user.PasswordHash,
    ).Scan(&user.ID, &user.CreatedAt)
}
```

**Rust（sqlx 编译期校验 SQL）：**
```rust
pub async fn create(pool: &PgPool, user: &CreateUser) -> Result<User, sqlx::Error> {
    sqlx::query_as!(
        User,
        r#"INSERT INTO users (email, username, password_hash, display_name)
           VALUES ($1, $2, $3, $4)
           RETURNING id, email, username, display_name, created_at, updated_at"#,
        user.email,
        user.username,
        user.password_hash,
        user.display_name,
    )
    .fetch_one(pool)
    .await
}
```

> `sqlx::query_as!` 在编译期连接数据库检查 SQL 语法和返回类型，字段不匹配直接编译报错。

---

## 四、内存占用对照

这是 Rust 最大的优势之一。

| 场景 | Go | Rust | 差异 |
|------|-----|------|------|
| 空载运行 | 15-25 MB | 5-8 MB | Rust 省 60%+ |
| 加载 GSE/Jieba 分词词典 | +20-110 MB | +15-80 MB | Rust 省 ~25% |
| 处理单个 PDF 上传 (10MB) | +30-50 MB (GC 延迟释放) | +10-15 MB (立即释放) | Rust 峰值更低 |
| 稳态运行 (100 并发) | 80-200 MB | 30-60 MB | Rust 省 60-70% |
| **512MB 限制下的余量** | 较紧张 (Mixed 模式) | 非常充裕 | Rust 明显胜出 |

**Rust 优势来源**：
- **无 GC**：不会出现"已不用但 GC 未回收"的内存占用
- **枚举紧凑**：`Option<Box<T>>` 可编译为单个指针，Go 的 `interface{}` 有额外开销
- **静态分发**：泛型在编译期单态化，无虚表开销

### Tokio vs Goroutine 内存

| | Goroutine | Tokio Task |
|---|-----------|------------|
| 初始栈大小 | 2-4 KB | ~64 bytes (无栈) |
| 10 万并发内存 | ~200-400 MB | ~6-8 MB |
| 调度器开销 | 有栈协程 | 无栈 async/await |

---

## 五、中文分词对照

这是切换到 Rust 风险最大的部分。

| 能力 | Go (GSE) | Rust (jieba-rs) | 差距分析 |
|------|----------|-----------------|----------|
| 基础分词 | ✅ | ✅ | 持平 |
| 自定义词典 | ✅ `AddToken()` | ✅ `add_word()` | 持平 |
| 热更新词典 | ✅ (GSE 原生支持) | ⚠️ 需自行实现 reload | Rust 需额外开发 |
| 词性标注 | ✅ | ✅ | 持平 |
| 搜索引擎模式 | ✅ | ⚠️ `cut_for_search` | jieba-rs 支持但文档较少 |
| 内存占用 | 20-40 MB (GSE) | 15-30 MB | Rust 略优 |
| AI 编程友好度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Go 的 GSE API 更直观 |

**Rust 热更新词典实现方案**：

```rust
use std::sync::RwLock;

pub struct HotReloadSegmenter {
    inner: RwLock<jieba_rs::Jieba>,
}

impl HotReloadSegmenter {
    /// 从数据库加载最新词典并热更新
    pub async fn reload_from_db(&self, pool: &PgPool) -> Result<(), AppError> {
        let words = sqlx::query_as!(CustomWord, "SELECT term, frequency FROM custom_dictionary WHERE is_enable = true")
            .fetch_all(pool).await?;

        let mut new_seg = jieba_rs::Jieba::new();
        for w in &words {
            new_seg.add_word(&w.term, None, Some(w.frequency as usize));
        }

        // 原子替换
        let mut writer = self.inner.write().unwrap();
        *writer = new_seg;
        Ok(())
    }
}
```

> 因为 jieba-rs 没有内置热更新 API，需要包装 `RwLock` 实现。Go 的 GSE 原生支持，更省事。

---

## 六、Docker 部署对照

### Rust Dockerfile（多阶段构建）

```dockerfile
# ===== 构建阶段 =====
FROM rust:1.80-alpine AS builder

RUN apk add --no-cache musl-dev pkgconfig openssl-dev

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src/ src/
COPY config/ config/
COPY migrations/ migrations/

# 编译为静态链接二进制
RUN cargo build --release && \
    strip target/release/pdf-manager

# ===== 运行阶段 =====
FROM alpine:3.20

RUN apk add --no-cache ca-certificates tzdata

COPY --from=builder /app/target/release/pdf-manager /usr/local/bin/
COPY --from=builder /app/config/ /etc/pdf-manager/
COPY --from=builder /app/migrations/ /etc/pdf-manager/migrations/

EXPOSE 8080
CMD ["pdf-manager"]
```

### 镜像大小对比

| | Go | Rust |
|---|-----|------|
| 构建镜像 | ~600 MB | ~1.2 GB（首次，依赖多） |
| 运行镜像 (Alpine) | ~15-25 MB | ~10-18 MB |
| Scratch 镜像 | ~8 MB (CGO_ENABLED=0) | ~5 MB (musl 静态链接) |

### 编译时间对比

| | Go | Rust |
|---|-----|------|
| 首次编译 (冷缓存) | ~30 秒 | 2-5 分钟 |
| 增量编译 | 1-3 秒 | 10-30 秒（依赖数量决定） |
| CI/CD 友好度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐（需缓存 target/） |

---

## 七、AI 编程效率对照

这是零经验团队最需要关注的部分。

| 场景 | Go | Rust | 说明 |
|------|-----|------|------|
| CRUD 开发 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Go 的简单性让 AI 一次生成可用率很高 |
| 并发编程 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Goroutine 心智负担极低，Tokio 有陡峭学习曲线 |
| 错误处理 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | `?` 运算符很强大，但错误类型组合有时让 AI 困惑 |
| 序列化/反序列化 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | serde 是 Rust 最强生态之一，AI 用得很好 |
| PDF 处理 | ⭐⭐⭐⭐ | ⭐⭐⭐ | Go 的 pdfcpu 更成熟，AI 训练数据更多 |
| 中文分词 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | GSE 生态更丰富，jieba-rs 训练数据较少 |
| 数据库操作 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | sqlx 编译期校验减少了运行时 SQL 错误 |
| 总体 | **高** | **中** | AI 生成 Rust 代码的返工率预计为 Go 的 2-3 倍 |

### AI 修复循环对比

```
Go 典型流程：
  提示词 → AI生成 → 编译通过 → 运行 → 可能有小bug → 修1-2轮 → 完成

Rust 典型流程：
  提示词 → AI生成 → 编译失败 → AI修复 → 编译失败 → AI修复 → 编译通过
  → 运行 → 发现 .clone() 滥用 → AI重构 → 编译通过 → 完成
```

**结论**：Rust 编译器能更早发现错误，但 AI 达到"能编译且高质量"需要更多轮次（3-5 轮 vs Go 的 1-2 轮）。

---

## 八、综合分析

### 8.1 优势矩阵

| 维度 | Go 胜 | 平手 | Rust 胜 |
|------|--------|------|---------|
| 内存占用 | | | ✅ +60% |
| 运行性能 | | | ✅ +20-50% |
| 类型安全 | | | ✅ (无 nil) |
| 编译速度 | ✅ | | |
| 开发速度 | ✅ | | |
| AI 编程效率 | ✅ | | |
| 中文分词生态 | ✅ | | |
| 部署镜像大小 | | | ✅ 更小 |
| 并发模型简洁 | ✅ | | |
| 数据库类型安全 | | | ✅ (编译期 SQL) |
| 错误信息质量 | | | ✅ |
| 学习曲线 | ✅ | | |

### 8.2 风险清单

| 风险 | 严重程度 | 缓解措施 |
|------|----------|----------|
| AI 生成代码可用率低 | 🔴 高 | 拆分更小的模块，增加更多代码示例 |
| 中文分词热更新需自研 | 🟡 中 | 用 `RwLock` 包装，投入约 2-3 天开发 |
| jieba-rs 社区活跃度 | 🟡 中 | 可用 IK 分词器（ES）绕过应用层分词 |
| 团队成员 Rust 经验 | 🔴 高 | 先小范围试点，编写内部 Rust 速查手册 |
| 编译时间长影响迭代 | 🟢 低 | sccache 缓存 + CI 预编译 |

### 8.3 最终建议

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│   ✅ 保持 Go（推荐）                                   │
│   ├─ 适用：零经验 + AI 编程 + 快速交付                  │
│   ├─ 优点：开发速度快，AI 可用率高，风险低               │
│   └─ 缺点：512MB 内存较紧张                            │
│                                                      │
│   ⚠️ 迁移 Rust（适合有条件的团队）                       │
│   ├─ 适用：有 Rust 经验 + 追求极致性能 + 内存敏感        │
│   ├─ 优点：内存省 60%+，性能更好，类型更安全             │
│   └─ 缺点：AI 返工率高，分词生态弱，学习曲线陡           │
│                                                      │
│   🔀 混合方案（折中推荐）                               │
│   ├─ 主业务层：Go（快速迭代）                           │
│   ├─ 分词引擎：Rust FFI 库（高性能 + 省内存）            │
│   └─ 优点：两全其美，风险可控                           │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 九、如果决定用 Rust 的迁移路径

### 第一阶段：验证（1 周）
- [ ] 搭建 Rust 项目骨架（`src/main.rs` + Cargo.toml）
- [ ] 实现 `/health` + `/api/v1/auth/register` 一个完整链路
- [ ] 验证 Docker 构建 + 运行
- [ ] 评估 AI 编程实际效率

### 第二阶段：核心（2-3 周）
- [ ] 实现认证模块（register/login/JWT）
- [ ] 实现文档上传 + S3 集成
- [ ] 实现 PDF 文本提取
- [ ] 实现分词服务 + 热更新

### 第三阶段：完善（2-3 周）
- [ ] 标记管理（highlights/annotations/bookmarks）
- [ ] 全文搜索
- [ ] 后台管理
- [ ] 单元测试 + 集成测试

### 第四阶段：上线（1 周）
- [ ] 性能压测
- [ ] CI/CD 流水线
- [ ] 文档完善

---

## 十、参考资料

| 组件 | 文档链接 |
|------|---------|
| Axum | https://docs.rs/axum |
| sqlx | https://docs.rs/sqlx |
| jieba-rs | https://docs.rs/jieba-rs |
| lopdf | https://docs.rs/lopdf |
| Tokio | https://tokio.rs |
| Tracing | https://docs.rs/tracing |
