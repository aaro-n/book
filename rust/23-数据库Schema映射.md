# 数据库 Schema 映射 — Rust 数据模型 → PostgreSQL

> **定位**：将 `20-数据模型实现.md` 中的 Rust 结构体映射到 `04-数据库设计.md` 的实际表结构。
> **核心原则**：**旁路（Sidecar）存储** — PDF 源文件永远不修改，所有标注数据存 PG JSONB。
> **更新日期**：2026-07-13

---

## 一、映射原则

1. **Rust 模型先行**：先在 `models/` 中定义 Serde 结构体（文档 20），再映射到 SQL
2. **JSONB 存复杂结构**：`Locator`、`Selector` 等嵌套结构存为 JSONB，应用层反序列化
3. **平面字段建索引**：`document_id`、`page_num`、`motivation` 等查询字段单独列
4. **W3C 兼容**：保留 `@context`、`id`(IRI)、`type` 等 W3C 必需字段

---

## 二、Annotation 表（增强版）

在 `04-数据库设计.md` 基础上，增加 W3C Web Annotation 字段：

```sql
CREATE TABLE annotations (
    id SERIAL PRIMARY KEY,
    
    -- W3C Web Annotation 字段
    w3c_id VARCHAR(255) UNIQUE,               -- Annotation.id (IRI)
    w3c_context TEXT DEFAULT 'http://www.w3.org/ns/anno.jsonld',
    
    -- 关联
    document_id INTEGER NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- 位置（简化为页码，用于快速查询）
    page_num INTEGER NOT NULL,
    
    -- === Locator 字段（Readium 通用位置） ===
    -- 对应 src/models/locator.rs 的 Locator 结构体
    locator_href TEXT,                        -- Locator.href
    locator_type TEXT,                        -- Locator.type (MIME)
    locator_locations JSONB,                  -- Locator.locations (完整 JSON)
    locator_text JSONB,                       -- Locator.text (完整 JSON)
    
    -- === Selector 字段（W3C 选择器） ===
    -- 对应 src/models/annotation.rs 的 Selector enum
    selector_type TEXT,                       -- "text_quote" | "fragment" | "css"
    selector_data JSONB,                      -- 完整 Selector JSON
    
    -- === Body 字段（用户笔记） ===
    text_body TEXT,                           -- TextualBody.value
    body_format VARCHAR(20),                  -- "text/plain" | "text/html"
    
    -- 标注类型 + 样式
    annotation_type VARCHAR(20) NOT NULL,
    color VARCHAR(7) DEFAULT '#FFFF00',
    opacity REAL DEFAULT 0.3,
    motivation VARCHAR(20) NOT NULL,
    
    -- W3C 时间
    created_iso VARCHAR(35),                  -- ISO 8601 创建时间
    modified_iso VARCHAR(35),                 -- ISO 8601 修改时间
    
    -- 同步字段（见文档 13）
    device_id VARCHAR(64),
    is_deleted BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ann_doc_page ON annotations(document_id, page_num);
CREATE INDEX idx_ann_user ON annotations(user_id);
CREATE INDEX idx_ann_motivation ON annotations(motivation);
CREATE INDEX idx_ann_selector_type ON annotations(selector_type);
```

---

## 三、Rust ↔ SQL 字段映射表

### 3.1 Locator → 数据库字段

| Rust 字段 | 类型 | 数据库字段 | 说明 |
|-----------|------|-----------|------|
| `Locator.href` | String | `locator_href` | 资源路径 |
| `Locator.media_type` | String | `locator_type` | MIME 类型 |
| `Locator.title` | Option\<String\> | — | 可选，不存 |
| `Locator.locations` | Option\<Locations\> | `locator_locations` | JSONB |
| `Locator.text` | Option\<LocatorText\> | `locator_text` | JSONB |

### 3.2 Selector → 数据库字段

| Rust 字段 | 类型 | 数据库字段 | 说明 |
|-----------|------|-----------|------|
| `Selector::TextQuoteSelector` | TextQuoteSelector | `selector_type="text_quote"` | `selector_data` 存完整 JSON |
| `Selector::FragmentSelector` | FragmentSelector | `selector_type="fragment"` | 同上 |
| `Selector::CssSelector` | CssSelector | `selector_type="css"` | 同上 |

### 3.3 Annotation → 数据库字段

| Rust 字段 | 类型 | 数据库字段 | 说明 |
|-----------|------|-----------|------|
| `Annotation.id` | String | `w3c_id` | W3C IRI |
| `Annotation.target` | AnnotationTarget | 分解存储 | source → page_num, selector → selector_data |
| `Annotation.body` | Option\<AnnotationBody\> | `text_body` + `body_format` | 展开存储 |
| `Annotation.motivation` | AnnotationMotivation | `motivation` | 字符串 |
| `Annotation.created` | String | `created_iso` | ISO 8601 |

---

## 四、Rust 读写示例

```rust
// 写入标注
async fn create_annotation(
    pool: &PgPool,
    anno: &Annotation,
    locator: &Locator,
    doc_id: i32,
    user_id: i32,
) -> Result<i32> {
    let page_num = locator.page_number().unwrap_or(1);
    
    sqlx::query!(
        r#"INSERT INTO annotations 
           (w3c_id, document_id, user_id, page_num,
            locator_href, locator_type, locator_locations, locator_text,
            selector_type, selector_data,
            text_body, body_format,
            annotation_type, motivation, created_iso)
           VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15)"#,
        anno.id,
        doc_id,
        user_id,
        page_num,
        locator.href,
        locator.media_type,
        serde_json::to_value(&locator.locations)?,
        serde_json::to_value(&locator.text)?,
        "text_quote",  // 根据实际 selector 类型
        serde_json::to_value(&anno.target)?,
        match &anno.body {
            Some(AnnotationBody::Textual(tb)) => Some(&tb.value),
            _ => None,
        },
        "text/plain",
        "highlight",
        format!("{:?}", anno.motivation).to_lowercase(),
        anno.created,
    )
    .execute(pool)
    .await?;
    
    Ok(1)
}

// 读取标注 → 还原为 Rust 结构体
async fn get_annotation(pool: &PgPool, id: i32) -> Result<(Annotation, Locator)> {
    let row = sqlx::query!(
        r#"SELECT w3c_id, w3c_context, locator_href, locator_type,
                  locator_locations, locator_text,
                  selector_data, text_body, motivation,
                  created_iso, modified_iso
           FROM annotations WHERE id = $1"#,
        id
    )
    .fetch_one(pool)
    .await?;
    
    // 从 JSONB 还原 Locator
    let locator = Locator {
        href: row.locator_href.unwrap_or_default(),
        media_type: row.locator_type.unwrap_or_default(),
        title: None,
        locations: serde_json::from_value(row.locator_locations.unwrap())?,
        text: serde_json::from_value(row.locator_text.unwrap())?,
    };
    
    // 从 JSONB 还原 Annotation
    let annotation = Annotation { ... };
    
    Ok((annotation, locator))
}
```

---

## 五、JSONB 存储示例

### locator_locations 字段

```json
{
  "fragments": ["page=42"],
  "progression": 0.65,
  "position": 42,
  "total_progression": 0.65
}
```

### locator_text 字段

```json
{
  "before": "这是前文",
  "highlight": "高亮的文本内容",
  "after": "这是后文"
}
```

### selector_data 字段（TextQuoteSelector）

```json
{
  "type": "TextQuoteSelector",
  "exact": "高亮的文本内容",
  "prefix": "这是前文",
  "suffix": "这是后文"
}
```
