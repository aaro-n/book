# Web Annotation 验证方案

> **定位**：在 Rust 后端实现 W3C Web Annotation 数据模型验证，参考 `other/validate-web-annotation/` 项目。
> **更新日期**：2026-07-13

---

## 一、参考项目

`other/validate-web-annotation/` 提供了 Node.js 实现的 W3C Web Annotation 验证：

```javascript
const validateAnnotation = require('validate-web-annotation');

const anno = {
  '@context': 'http://www.w3.org/ns/anno.jsonld',
  id: 'http://example.org/anno1',
  type: 'Annotation',
  body: { type: 'TextualBody', value: 'Comment' },
  target: 'http://example.com/page1',
};

validateAnnotation(anno); // true/false + errors
```

**核心验证规则**（来自 `wadm-schema.js`）：
- `@context` 必须是 `"http://www.w3.org/ns/anno.jsonld"` 或包含该 IRI 的数组
- `type` 必须是 `"Annotation"`
- `id` 必须是 IRI 格式
- `target` 必须存在（IRI 或 SpecificResource）
- `bodyValue` 和 `body` 不能同时存在

---

## 二、Rust 验证方案

### 2.1 使用 `validator` crate

在 `Cargo.toml` 中添加：

```toml
[dependencies]
validator = { version = "0.18", features = ["derive"] }
url = "2.5"    # IRI 验证
```

### 2.2 Annotation 结构体验证

```rust
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, Validate)]
#[serde(rename_all = "camelCase")]
pub struct Annotation {
    /// JSON-LD 上下文
    #[serde(rename = "@context")]
    pub context: serde_json::Value,
    
    /// 类型标识
    #[serde(rename = "type")]
    #[validate(length(equal = 10, message = "must be 'Annotation'"))]
    pub anno_type: String,
    
    /// 唯一 IRI 标识
    #[validate(custom(function = "validate_iri"))]
    pub id: String,
    
    /// 标注目标（必需）
    #[validate(nested)]
    pub target: AnnotationTarget,
    
    /// 标注正文（可选）
    #[validate(nested)]
    pub body: Option<AnnotationBody>,
    
    /// 动机
    pub motivation: AnnotationMotivation,
    
    /// 创建时间
    pub created: String,
    
    /// 修改时间
    pub modified: Option<String>,
}

/// IRI 格式验证
fn validate_iri(iri: &str) -> Result<(), validator::ValidationError> {
    url::Url::parse(iri)
        .map(|_| ())
        .map_err(|_| {
            let mut err = validator::ValidationError::new("invalid_iri");
            err.add_param("value".into(), &iri);
            Err(err)
        })
}
```

### 2.3 @context 验证

```rust
impl Annotation {
    /// 验证 @context 是否符合 W3C 规范
    pub fn validate_context(&self) -> bool {
        match &self.context {
            serde_json::Value::String(s) => {
                s == "http://www.w3.org/ns/anno.jsonld"
            }
            serde_json::Value::Array(arr) => {
                arr.iter().any(|v| {
                    v.as_str() == Some("http://www.w3.org/ns/anno.jsonld")
                })
            }
            _ => false,
        }
    }
}
```

### 2.4 bodyValue 与 body 互斥验证

```rust
impl Annotation {
    /// W3C 约束：bodyValue 和 body 不能同时存在
    pub fn validate_body_exclusive(&self) -> bool {
        // MVP 阶段简化处理：有 body 就不允许 bodyValue
        self.body.is_some() || self.body_value.is_none()
    }
}
```

---

## 三、验证测试用例

参考 `validate-web-annotation/test.js` 中的测试案例：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_basic_annotation() {
        let json = r#"{
            "@context": "http://www.w3.org/ns/anno.jsonld",
            "id": "http://example.org/anno1",
            "type": "Annotation",
            "target": "http://example.com/page1",
            "motivation": "highlighting",
            "created": "2024-01-15T10:00:00Z"
        }"#;
        
        let anno: Annotation = serde_json::from_str(json).unwrap();
        assert!(anno.validate_context());
    }
    
    #[test]
    fn test_invalid_context() {
        let json = r#"{
            "@context": "invalid",
            "id": "http://example.org/anno1",
            "type": "Annotation",
            "target": "http://example.com/page1",
            "motivation": "highlighting",
            "created": "2024-01-15T10:00:00Z"
        }"#;
        
        let anno: Annotation = serde_json::from_str(json).unwrap();
        assert!(!anno.validate_context());
    }
    
    #[test]
    fn test_textual_body() {
        let json = r#"{
            "@context": "http://www.w3.org/ns/anno.jsonld",
            "id": "http://example.org/anno1",
            "type": "Annotation",
            "target": "http://example.com/page1",
            "motivation": "commenting",
            "created": "2024-01-15T10:00:00Z",
            "body": {
                "type": "TextualBody",
                "value": "用户笔记",
                "format": "text/plain"
            }
        }"#;
        
        let anno: Annotation = serde_json::from_str(json).unwrap();
        assert!(matches!(anno.body, Some(AnnotationBody::Textual(_))));
    }
}
```

---

## 四、API 层验证

在 Axum handler 中使用验证：

```rust
use axum::{Json, extract::State, http::StatusCode};
use validator::Validate;

async fn create_annotation(
    State(pool): State<PgPool>,
    Json(payload): Json<CreateAnnotationRequest>,
) -> Result<(StatusCode, Json<AnnotationResponse>), AppError> {
    // 验证输入
    payload.validate()
        .map_err(|e| AppError::ValidationError(e.to_string()))?;
    
    // 写入数据库
    let id = insert_annotation(&pool, &payload).await?;
    
    Ok((StatusCode::CREATED, Json(AnnotationResponse { id })))
}
```

---

## 五、验证层级总结

```
┌─────────────────────────────────────┐
│  第 1 层：JSON 解析 (serde)          │  类型匹配
├─────────────────────────────────────┤
│  第 2 层：结构验证 (validator)       │  必需字段、IRI 格式、长度
├─────────────────────────────────────┤
│  第 3 层：业务验证 (自定义函数)      │  @context 检查、body 互斥
├─────────────────────────────────────┤
│  第 4 层：数据库约束 (SQL)           │  外键、唯一性、NOT NULL
└─────────────────────────────────────┘
```

---

## 六、MVP 验证清单

| 验证项 | 规则 | 优先级 |
|--------|------|--------|
| `@context` | 必须包含 W3C anno.jsonld | ✅ MVP |
| `type` | 必须为 "Annotation" | ✅ MVP |
| `id` | 必须是 IRI 格式 | ✅ MVP |
| `target` | 必须存在 | ✅ MVP |
| `motivation` | 必须是合法值 | ✅ MVP |
| `bodyValue` vs `body` | 互斥 | ✅ MVP |
| `created` | ISO 8601 格式 | ✅ MVP |
| `selector.exact` | TextQuoteSelector 时非空 | ✅ MVP |
| `selector.value` | FragmentSelector 时非空 | Phase 2 |
| `body.format` | 必须是合法 MIME | Phase 2 |
