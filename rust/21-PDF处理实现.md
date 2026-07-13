# PDF 处理实现 — 纯文字提取 + 分词

> **定位**：服务端 PDF 处理仅做**纯文字提取**，不提取坐标，不渲染，不修改源文件。
> **架构**：服务端提取文字 → jieba-rs 分词 → 存入 PG 用于搜索；前端用 PDF.js 负责渲染和高亮。
> **内存预算**：≤ 60MB（服务端 512MB 总量的 ~12%）

---

## 一、处理范围

### 服务端做

| 操作 | 使用 crate | 内存 |
|------|-----------|------|
| 检测 PDF 是否有文字层 | `pdf-extract` | ~20MB |
| 提取全文字 + 分词 | `pdf-extract` + `jieba-rs` | ~60MB |
| 存入 PG 用于搜索 | `sqlx` | ~5MB |

### 服务端不做（交给前端）

| 操作 | 原因 |
|------|------|
| PDF 渲染 | 前端 PDF.js 处理 |
| 高亮坐标计算 | 前端 PDF.js 计算 |
| 标注叠加显示 | 前端 SVG/Canvas 渲染 |

---

## 二、文字层检测 + 文本提取

### 2.1 核心逻辑

```rust
// src/pdf/extract.rs

use pdf_extract::{extract_text, extract_text_by_pages};
use tracing::{info, warn};

/// PDF 类型检测结果
#[derive(Debug, Clone, PartialEq)]
pub enum PdfType {
    /// 有文字层：可直接提取文本
    TextLayer { page_count: u32 },
    /// 扫描件：全是图片，无文本
    Scanned { reason: String },
    /// 无法打开或损坏
    Corrupted { error: String },
}

/// 检测并提取 PDF 文本
pub fn process_pdf(data: &[u8]) -> Result<(PdfType, Vec<PageContent>), AppError> {
    // 尝试提取文本
    let pages = extract_text_by_pages(data)
        .map_err(|e| AppError::Internal(format!("PDF 提取失败: {}", e)))?;

    if pages.is_empty() {
        return Ok((PdfType::Scanned {
            reason: "PDF 未提取到任何文本，可能是扫描件".into(),
        }, vec![]));
    }

    let page_count = pages.len() as u32;
    let contents: Vec<PageContent> = pages
        .into_iter()
        .enumerate()
        .map(|(i, text)| PageContent {
            page_num: (i + 1) as u32,
            raw_text: text,
            segmented_text: String::new(), // 分词在下一步处理
        })
        .collect();

    Ok((PdfType::TextLayer { page_count }, contents))
}

/// 单页内容结构
#[derive(Debug, Clone)]
pub struct PageContent {
    pub page_num: u32,
    pub raw_text: String,
    pub segmented_text: String,
}
```

---

## 三、内存与性能

### 3.1 内存预算

| 操作 | 峰值内存 | 说明 |
|------|---------|------|
| `extract_text_by_pages` | ~20MB | pdf-extract 全量解析 |
| `process_pdf` + jieba | ~60MB | 文本 + 分词 |

### 3.2 文件大小限制

- 服务端处理上限：**200MB**
- 超过限制的 PDF 应使用 CLI 工具预处理后上传 `.bookpkg`

---

## 四、单元测试

```rust
// src/pdf/extract.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_segment_text() {
        let jieba = Jieba::new();
        let result = jieba.cut("我喜欢阅读PDF文档", true).join(" ");
        assert!(!result.is_empty());
        assert!(result.contains(' '));
    }

    #[test]
    fn test_size_check() {
        assert!(check_pdf_size(100 * 1024 * 1024).is_ok());
        assert!(check_pdf_size(201 * 1024 * 1024).is_err());
    }

    #[test]
    fn test_corrupted_pdf() {
        let jieba = Jieba::new();
        let garbage = b"not a pdf file at all";
        let result = process_pdf(garbage, &jieba);
        assert!(matches!(result, Ok((PdfType::Corrupted { .. }, _))));
    }
}
```

---

## 五、前端 PDF 方案

服务端只负责文字提取，前端用 **PDF.js** 处理：

```
┌─────────────────────────────────────┐
│         服务端（Rust）               │
│                                     │
│  PDF 上传 → pdf-extract 提取纯文本  │
│              → jieba-rs 分词        │
│              → 存入 PG document_content │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         前端（浏览器）               │
│                                     │
│  PDF.js 渲染 PDF                    │
│  PDF.js 提取文本坐标（前端计算）     │
│  用户选择高亮 → 计算坐标 → 存服务端  │
│  SVG 叠加层渲染高亮标注              │
└─────────────────────────────────────┘
```

---

## 七、Crash 安全

```rust
// std::panic::catch_unwind 包裹所有 PDF 操作
// pdf-extract 解析异常 PDF 时可能 panic，用 catch_unwind 兜底

use std::panic;

pub fn safe_process_pdf(
    data: &[u8],
    jieba: &Jieba,
) -> Result<(PdfType, Vec<PageContent>), AppError> {
    panic::catch_unwind(panic::AssertUnwindSafe(|| {
        process_pdf(data, jieba)
    }))
    .map_err(|_| AppError::Internal("PDF 处理时发生内部错误（panic）".into()))?
}
```

---

## 八、与现有文档的关系

| 文档 | 使用本模块 |
|------|-----------|
| `03-技术架构.md` | 服务端处理能力边界表 ↔ `process_pdf` 返回 `PdfType::Scanned` |
| `04-数据库设计.md` | `process_pdf` → `document_content` 表逐页入库 |
| `06-Rust项目结构.md` | 代码放入 `src/pdf/extract.rs` |
| `14-导入CLI工具.md` | `PdfType::Scanned` 时提示用户用 CLI |
