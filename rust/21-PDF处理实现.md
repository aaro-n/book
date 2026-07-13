# PDF 处理实现 — 文字层检测 + 文本提取 + 单页拆分

> **定位**：服务端 `util/pdf.rs` 的完整实现指南。处理 PDF 是服务端核心能力之一（`03-技术架构.md` 中定义的"轻量 PDF 提取"）。
> **内存预算**：≤ 60MB（服务端 512MB 总量的 ~12%）

---

## 一、处理范围

### 服务端做

| 操作 | 使用 crate | 内存 |
|------|-----------|------|
| 检测 PDF 是否有文字层 | `lopdf` 读前 3 页 | ~20MB |
| 提取全文字 + 分词 | `lopdf` + `pdf-extract` + `jieba-rs` | ~60MB |
| 提取封面（首页缩略图） | `lopdf` | ~10MB |

### 服务端不做（交给 CLI `14-导入CLI工具.md`）

| 操作 | 原因 |
|------|------|
| OCR 扫描件 | Tesseract 需要 200-400MB |
| 大文件 > 200MB | 内存扛不住 |
| 单页拆分为 WebP/PDF | 渲染耗内存，CLI 端处理 |
| 特殊语言（泰/阿）| 分词策略不同 |

---

## 二、文字层检测

### 2.1 核心逻辑

```rust
// src/util/pdf.rs

use lopdf::{Document, Object, ObjectId};
use tracing::{info, warn};

/// PDF 类型检测结果
#[derive(Debug, Clone, PartialEq)]
pub enum PdfType {
    /// 有文字层：可直接提取文本
    TextLayer {
        /// 前 3 页中至少 1 页有文本
        text_pages: Vec<u32>,
    },
    /// 扫描件：全是图片，无文本对象 → 需 OCR
    Scanned {
        reason: String,
    },
    /// 无法打开或损坏
    Corrupted {
        error: String,
    },
}

/// 检测 PDF 前 3 页是否有文字层。
///
/// 方法：检查每页的 Content Stream 中是否包含文本操作符
/// （BT/ET 块、Tj/TJ 操作符）。
///
/// 内存：~20MB（只加载前 3 页，不全量解析）
pub fn detect_pdf_type(data: &[u8]) -> PdfType {
    let doc = match Document::load_mem(data) {
        Ok(d) => d,
        Err(e) => return PdfType::Corrupted {
            error: format!("无法解析 PDF: {}", e),
        },
    };

    let page_ids = doc.get_pages().keys().take(3).copied().collect::<Vec<_>>();

    if page_ids.is_empty() {
        return PdfType::Corrupted {
            error: "PDF 无页面".into(),
        };
    }

    let mut text_pages = Vec::new();
    let max_pages_to_check = 3.min(page_ids.len());

    for &page_id in &page_ids[..max_pages_to_check] {
        if page_has_text(&doc, page_id) {
            text_pages.push(page_id.0 as u32);
        }
    }

    if text_pages.is_empty() {
        PdfType::Scanned {
            reason: format!(
                "前 {} 页均未检测到文本对象，可能是扫描件",
                max_pages_to_check
            ),
        }
    } else {
        PdfType::TextLayer { text_pages }
    }
}

/// 检查单页是否包含文本操作符
fn page_has_text(doc: &Document, page_id: ObjectId) -> bool {
    // 获取页面的 Contents 流
    let content = match get_page_content(doc, page_id) {
        Some(c) => c,
        None => return false,
    };

    // 文本操作符检查（不完整解析，只做字符串匹配）
    // BT = Begin Text, ET = End Text
    // Tj = Show text, TJ = Show text with positioning
    let text_operators = ["BT", "Tj", "TJ", "'", "\""];

    text_operators.iter().any(|op| {
        // 以操作符 + 换行/空格 的形式匹配，避免误匹配
        content.contains(&format!(" {} ", op))
            || content.contains(&format!("{}\n", op))
            || content.ends_with(op)
    })
}

/// 提取页面 Content Stream 为字符串
fn get_page_content(doc: &Document, page_id: ObjectId) -> Option<String> {
    use lopdf::content::Content;

    let content_data = doc.get_page_content(page_id).ok()?;
    let content = Content::decode(&content_data).ok()?;

    // 将 Content 操作合并为字符串用于检测
    let text: String = content
        .operations
        .iter()
        .map(|op| op.operator.clone())
        .collect::<Vec<_>>()
        .join(" ");

    Some(text)
}
```

### 2.2 在 document_service 中使用

```rust
// src/service/document_service.rs

use crate::util::pdf::{detect_pdf_type, PdfType};

pub async fn upload_document(
    state: &AppState,
    user_id: i32,
    file_data: &[u8],
    file_name: &str,
) -> Result<Document, AppError> {
    // 1. 检测 PDF 类型
    let pdf_type = detect_pdf_type(file_data);

    match pdf_type {
        PdfType::Corrupted { error } => {
            return Err(AppError::BadRequest(format!("PDF 文件损坏: {}", error)));
        }
        PdfType::Scanned { reason } => {
            return Err(AppError::BadRequest(format!(
                "{}。请使用 CLI 工具（book-import）进行 OCR 后上传 .bookpkg",
                reason
            )));
        }
        PdfType::TextLayer { .. } => {
            // 文字版 PDF → 继续处理
            info!("PDF 有文字层，继续提取文本");
        }
    }

    // 2. 提取文本 + 分词（见第三节）
    // 3. 上传到 S3（见 16-S3存储方案.md）
    // ...
}
```

---

## 三、文本提取 + 分词

### 3.1 逐页提取文本

```rust
// src/util/pdf.rs (续)

use pdf_extract::extract_text;
use jieba_rs::Jieba;

/// 提取 PDF 全文（逐页）。
///
/// 返回 Vec<(page_num, raw_text)>
/// page_num 从 1 开始。
pub fn extract_text_per_page(data: &[u8]) -> Result<Vec<(u32, String)>, AppError> {
    // pdf-extract 提供全文档一次性提取
    let full_text = extract_text(data)
        .map_err(|e| AppError::Internal(format!("PDF 文本提取失败: {}", e)))?;

    // pdf-extract 默认用换页符分隔页面，尝试按页分割
    let pages: Vec<(u32, String)> = full_text
        .split('\u{0C}') // 换页符 (form feed)
        .enumerate()
        .filter(|(_, text)| !text.trim().is_empty())
        .map(|(i, text)| ((i + 1) as u32, text.to_string()))
        .collect();

    if pages.is_empty() {
        return Err(AppError::Internal("PDF 未提取到任何文本".into()));
    }

    Ok(pages)
}

/// 用 jieba-rs 对文本分词。
/// 输入：原始文本 "我喜欢阅读PDF文档"
/// 输出：空格分隔的词 "我 喜欢 阅读 PDF 文档"
pub fn segment_text(jieba: &Jieba, raw_text: &str) -> String {
    jieba
        .cut(raw_text, true) // true = hmm 模式
        .iter()
        .filter(|w| !w.trim().is_empty())
        .collect::<Vec<_>>()
        .join(" ")
}

/// 完整处理：提取文本 → 分词 → 返回结构化数据
pub fn process_pdf_text(
    data: &[u8],
    jieba: &Jieba,
) -> Result<Vec<PageContent>, AppError> {
    let pages = extract_text_per_page(data)?;

    let result = pages
        .into_iter()
        .map(|(page_num, raw_text)| {
            let segmented_text = segment_text(jieba, &raw_text);
            PageContent {
                page_num,
                raw_text,
                segmented_text,
            }
        })
        .collect();

    Ok(result)
}

/// 单页内容结构
#[derive(Debug, Clone)]
pub struct PageContent {
    pub page_num: u32,
    pub raw_text: String,
    pub segmented_text: String,
}

/// 获取总页数
pub fn get_page_count(data: &[u8]) -> Result<u32, AppError> {
    let doc = Document::load_mem(data)
        .map_err(|e| AppError::Internal(format!("无法打开 PDF: {}", e)))?;
    Ok(doc.get_pages().len() as u32)
}
```

### 3.2 与 document_service 集成

```rust
// src/service/document_service.rs — 上传流程关键代码

use crate::util::pdf::{get_page_count, process_pdf_text, PageContent};

async fn process_and_store_document(
    state: &AppState,
    user_id: i32,
    file_data: &[u8],
    file_hash: &str,
    file_name: &str,
    file_size: i64,
) -> Result<Document, AppError> {
    // 1. 获取页数
    let page_count = get_page_count(file_data)?;

    // 2. 提取文本 + 分词
    let pages = process_pdf_text(file_data, &state.segmenter)?;

    // 3. 计算 file_hash + 入库 documents 表
    let s3_key = format!("{}/original.pdf", uuid::Uuid::new_v4());
    let doc = sqlx::query_as!(
        DocumentRow,
        r#"INSERT INTO documents (user_id, title, s3_key, file_hash, file_size_bytes, page_count)
           VALUES ($1, $2, $3, $4, $5, $6) RETURNING *"#,
        user_id, file_name, s3_key, file_hash, file_size, page_count as i32,
    )
    .fetch_one(&state.pool)
    .await?;

    // 4. 逐页入库 document_content 表
    for page in &pages {
        sqlx::query!(
            r#"INSERT INTO document_content (document_id, page_num, raw_text, segmented_text)
               VALUES ($1, $2, $3, $4)
               ON CONFLICT (document_id, page_num) DO UPDATE
               SET raw_text = $3, segmented_text = $4, updated_at = NOW()"#,
            doc.id, page.page_num as i32, page.raw_text, page.segmented_text,
        )
        .execute(&state.pool)
        .await?;
    }

    // 5. 上传原始文件到 S3
    state.s3.put_object()
        .bucket(&state.config.s3.bucket)
        .key(&s3_key)
        .body(file_data.to_vec().into())
        .send()
        .await?;

    Ok(doc)
}
```

---

## 四、封面提取（首页缩略图）

```rust
// src/util/pdf.rs (续)

use image::{DynamicImage, ImageFormat};

/// 从 PDF 第一页提取封面图。
///
/// 使用 lopdf 定位首页 → 提取 XObject 图片 → 转换为 JPEG。
/// 如果首页无内嵌图片，返回 None（用默认封面）。
pub fn extract_cover(data: &[u8]) -> Result<Option<Vec<u8>>, AppError> {
    let doc = Document::load_mem(data)
        .map_err(|e| AppError::Internal(format!("无法打开 PDF: {}", e)))?;

    // 获取第一页
    let page_ids: Vec<_> = doc.get_pages().keys().copied().collect();
    let first_page_id = match page_ids.first() {
        Some(id) => *id,
        None => return Ok(None),
    };

    // 获取页面 resources → XObject → 找第一个 Image
    let resources = match doc.get_page_resources(first_page_id) {
        Ok(r) => r,
        Err(_) => return Ok(None),
    };

    // 遍历 XObject 找图片
    for (_, obj_id) in resources.xobjects.iter().flatten() {
        if let Ok(obj) = doc.get_object(*obj_id) {
            if let Ok(stream) = obj.as_stream() {
                // 检查是否为 Image 子类型
                if stream.dict.get(b"Subtype").ok()
                    .and_then(|s| s.as_name().ok())
                    .map(|n| n == b"Image")
                    .unwrap_or(false)
                {
                    // 读取图片流内容
                    let img_data = stream.content.clone();
                    // lopdf 提取的原始图片可能需转换
                    // 直接返回原始字节流（前端解码）
                    return Ok(Some(img_data));
                }
            }
        }
    }

    Ok(None) // 无封面图片
}
```

---

## 五、内存与性能

### 5.1 内存预算

| 操作 | 峰值内存 | 说明 |
|------|---------|------|
| `detect_pdf_type` | ~20MB | 只解析前 3 页，不全量 |
| `extract_text_per_page` | ~40MB | pdf-extract 全量解析 |
| `process_pdf_text` + jieba | ~60MB | 文本 + 分词 |
| `extract_cover` | ~25MB | 首页解析 |

### 5.2 文件大小限制

```rust
/// 服务端处理上限
const MAX_PDF_SIZE: u64 = 200 * 1024 * 1024; // 200MB

pub fn check_pdf_size(file_size: u64) -> Result<(), AppError> {
    if file_size > MAX_PDF_SIZE {
        Err(AppError::BadRequest(format!(
            "PDF 文件 {}MB 超过服务端处理上限 200MB。请使用 CLI 工具（book-import）处理后上传 .bookpkg",
            file_size / 1024 / 1024
        )))
    } else {
        Ok(())
    }
}
```

### 5.3 流式/分页处理（大文件优化）

对于接近 200MB 上限的文件，逐页提取比分词更省内存：

```rust
/// 流式处理：逐页提取文本，边提取边释放。
/// 用于 100-200MB 的大 PDF。
pub fn extract_text_streaming(
    data: &[u8],
    jieba: &Jieba,
) -> Result<Vec<PageContent>, AppError> {
    let doc = Document::load_mem(data)
        .map_err(|e| AppError::Internal(format!("无法打开 PDF: {}", e)))?;

    let page_ids: Vec<_> = doc.get_pages().keys().copied().collect();
    let mut results = Vec::with_capacity(page_ids.len());

    for (i, &page_id) in page_ids.iter().enumerate() {
        let page_num = (i + 1) as u32;

        // 逐页提取文本（pdf-extract 不支持逐页，用 lopdf Content 解码）
        let raw_text = match extract_single_page_text(&doc, page_id) {
            Ok(t) => t,
            Err(e) => {
                warn!("第 {} 页文本提取失败: {}", page_num, e);
                continue;
            }
        };

        if raw_text.trim().is_empty() {
            continue;
        }

        let segmented_text = segment_text(jieba, &raw_text);
        results.push(PageContent { page_num, raw_text, segmented_text });
    }

    Ok(results)
}

/// 逐页提取：解码单页 Content Stream，提取文本操作符参数
fn extract_single_page_text(
    doc: &Document,
    page_id: ObjectId,
) -> Result<String, AppError> {
    use lopdf::content::Content;
    use lopdf::Object;

    let content_data = doc.get_page_content(page_id)
        .map_err(|e| AppError::Internal(format!("无法读取第 {:?} 页内容: {}", page_id, e)))?;

    let content = Content::decode(&content_data)
        .map_err(|e| AppError::Internal(format!("Content 解码失败: {}", e)))?;

    let mut texts = Vec::new();

    for operation in &content.operations {
        match operation.operator.as_ref() {
            "Tj" | "TJ" | "'" | "\"" => {
                // 文本显示操作符：提取操作数中的字符串
                for operand in &operation.operands {
                    if let Ok(s) = operand.as_str() {
                        texts.push(s.to_string());
                    } else if let Ok(array) = operand.as_array() {
                        // TJ 操作符：数组形式，包含字符串和偏移
                        for item in array {
                            if let Ok(s) = item.as_str() {
                                texts.push(s.to_string());
                            }
                        }
                    }
                }
            }
            _ => {}
        }
    }

    Ok(texts.join(""))
}
```

---

## 六、单元测试

```rust
// src/util/pdf.rs

#[cfg(test)]
mod tests {
    use super::*;

    /// 测试用：生成一个最简单的文字版 PDF
    fn create_minimal_text_pdf() -> Vec<u8> {
        use lopdf::{Document, Object, Stream, Dictionary};
        use lopdf::content::{Content, Operation};

        let mut doc = Document::with_version("1.7");

        // 创建一个简单页面，含一个文本 "Hello"
        let content = Content {
            operations: vec![
                Operation::new("BT", vec![]),
                Operation::new("Tf", vec!["F1".into(), 12.0.into()]),
                Operation::new("Td", vec![100.0.into(), 700.0.into()]),
                Operation::new("Tj", vec!["Hello PDF".into()]),
                Operation::new("ET", vec![]),
            ],
        };

        let content_stream = Stream::new(
            Dictionary::new(),
            content.encode().unwrap(),
        );

        let content_id = doc.add_object(content_stream);

        // 页面字典
        let mut page = Dictionary::new();
        page.set("Type", "Page");
        page.set("Contents", vec![content_id]);

        let page_id = doc.add_object(page);

        // 页面树
        let mut pages = Dictionary::new();
        pages.set("Type", "Pages");
        pages.set("Kids", vec![page_id.into()]);
        pages.set("Count", 1.into());

        doc.catalog_mut().unwrap().set("Pages", doc.add_object(pages));

        let mut buf = Vec::new();
        doc.save_to(&mut buf).unwrap();
        buf
    }

    #[test]
    fn test_detect_text_layer() {
        let pdf = create_minimal_text_pdf();
        let result = detect_pdf_type(&pdf);
        assert!(matches!(result, PdfType::TextLayer { .. }));
    }

    #[test]
    fn test_detect_scanned_pdf() {
        // 空内容流 = 无文本操作符 → 判为扫描件
        use lopdf::{Document, Dictionary};
        let mut doc = Document::with_version("1.7");
        let page = Dictionary::new();
        let page_id = doc.add_object(page);

        let mut pages = Dictionary::new();
        pages.set("Type", "Pages");
        pages.set("Kids", vec![page_id.into()]);
        pages.set("Count", 1.into());
        doc.catalog_mut().unwrap().set("Pages", doc.add_object(pages));

        let mut buf = Vec::new();
        doc.save_to(&mut buf).unwrap();

        let result = detect_pdf_type(&buf);
        assert!(matches!(result, PdfType::Scanned { .. }));
    }

    #[test]
    fn test_corrupted_pdf() {
        let garbage = b"not a pdf file at all";
        let result = detect_pdf_type(garbage);
        assert!(matches!(result, PdfType::Corrupted { .. }));
    }

    #[test]
    fn test_size_check() {
        assert!(check_pdf_size(100 * 1024 * 1024).is_ok());
        assert!(check_pdf_size(201 * 1024 * 1024).is_err());
    }

    #[test]
    fn test_segment_text() {
        let jieba = Jieba::new();
        let result = segment_text(&jieba, "我喜欢阅读PDF文档");
        // 分词结果应包含空格分隔的词
        assert!(!result.is_empty());
        assert!(result.contains(' '));
    }
}
```

---

## 七、Crash 安全

```rust
// std::panic::catch_unwind 包裹所有 PDF 操作
// lopdf 解析异常 PDF 时可能 panic，用 catch_unwind 兜底

use std::panic;

pub fn safe_detect_pdf_type(data: &[u8]) -> PdfType {
    panic::catch_unwind(panic::AssertUnwindSafe(|| {
        detect_pdf_type(data)
    }))
    .unwrap_or_else(|_| PdfType::Corrupted {
        error: "PDF 解析时发生内部错误（panic），文件可能严重损坏".into(),
    })
}

pub fn safe_process_pdf_text(
    data: &[u8],
    jieba: &Jieba,
) -> Result<Vec<PageContent>, AppError> {
    panic::catch_unwind(panic::AssertUnwindSafe(|| {
        process_pdf_text(data, jieba)
    }))
    .map_err(|_| AppError::Internal("PDF 处理时发生内部错误（panic）".into()))?
}
```

---

## 八、与现有文档的关系

| 文档 | 使用本模块 |
|------|-----------|
| `03-技术架构.md` | 服务端处理能力边界表 ↔ `detect_pdf_type` 输出 `PdfType::Scanned` |
| `04-数据库设计.md` | `extract_text_per_page` → `document_content` 表逐页入库 |
| `06-Rust项目结构.md` | 代码放入 `src/util/pdf.rs` |
| `14-导入CLI工具.md` | `PdfType::Scanned` 时提示用户用 CLI |
