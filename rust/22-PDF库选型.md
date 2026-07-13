# PDF 库选型 — Rust 方案对比

> **定位**：基于实际调研，选择最适合服务端 PDF 文字提取的 Rust 库。
> **更新日期**：2026-07-13
> **架构分工**：服务端只做 PDF 纯文字提取 → 存数据库用于搜索；网页端用 PDF.js 渲染和高亮。

---

## 一、核心需求

服务端只需要：
1. **PDF 纯文字提取** — 从 PDF 中提取文字内容
2. **分词** — 用 jieba-rs 分词后存入数据库

> **不需要**：服务端不需要提取坐标、不需要渲染、不需要写入 PDF。坐标和高亮由前端 PDF.js 负责。

---

## 二、候选库对比

| 库 | 版本 | 纯文本提取 | 坐标提取 | 纯 Rust | 许可证 | 推荐度 |
|---|---|---|---|---|---|---|
| **pdf-extract** | 0.12 | ✅ 优秀 | ❌ 无 | ✅ | Apache-2.0 | ⭐⭐⭐⭐⭐ |
| **lopdf** | 0.44 | ⚠️ 手动解析 | ⚠️ 手动 | ✅ | MIT | ⭐⭐ |
| **pdf-oxide** | 0.3 | ✅ | ✅ 完整 | ✅ | MIT/Apache | ⭐⭐⭐⭐ |
| **mupdf-sys** | 绑定 | ✅ | ✅ | ❌ (C绑定) | AGPL | ⭐⭐⭐ |

---

## 三、推荐方案：pdf-extract（纯文字提取）

### 3.1 为什么选回 pdf-extract？

服务端架构简化后，**只需要纯文字提取**，不需要坐标：

| 功能 | 服务端 | 前端 |
|------|--------|------|
| 文字提取 | ✅ Rust 后端 | — |
| 坐标计算 | — | ✅ PDF.js |
| 高亮渲染 | — | ✅ PDF.js + SVG |
| 分词搜索 | ✅ jieba-rs + PG | — |

`pdf-extract` 是纯文字提取的最佳选择：
- API 简单：`pdf_extract::extract_text(path)?` 一行搞定
- 与 lopdf 同源，稳定可靠
- 无需处理坐标数据，减少服务端复杂度

### 3.2 Cargo.toml

```toml
[dependencies]
pdf-extract = "0.12"   # 纯文字提取
jieba-rs = "0.7"       # 中文分词
```

### 3.3 核心代码示例

```rust
// src/pdf/extract.rs

use pdf_extract::extract_text;

/// 提取 PDF 全文（纯文本，无坐标）
pub fn extract_full_text(pdf_path: &str) -> Result<String> {
    let text = extract_text(pdf_path)?;
    Ok(text)
}

/// 按页分割文本（用换页符分隔）
pub fn extract_text_by_pages(pdf_path: &str) -> Result<Vec<(u32, String)>> {
    let full_text = extract_text(pdf_path)?;
    let pages = full_text
        .split('\u{0C}') // 换页符
        .enumerate()
        .filter(|(_, t)| !t.trim().is_empty())
        .map(|(i, t)| ((i + 1) as u32, t.to_string()))
        .collect();
    Ok(pages)
}
```

### 3.4 处理流程

```
PDF 上传 → pdf-extract 提取纯文本 → jieba-rs 分词 → 存入 PG document_content 表
```

---

## 五、为什么不选 mupdf-sys

- **AGPL 许可证**：商业项目需付费许可
- **C 绑定依赖**：需要 C 编译器，增加部署复杂度
- **二进制体积大**：不符合"纯 Rust"目标

---

## 六、前端 PDF 解决方案

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

## 七、PDF.js 浏览器端参考

从 `other/pdfjs-express/` 看，浏览器端高亮方案是：
```
PDF.js getTextContent() → TextItem[] { str, transform[] }
→ transform[4] = x, transform[5] = y, width, height
→ 在 canvas 上叠加高亮 div
```

后端只需提供对应的坐标数据（存入 PG JSONB），前端即可回放高亮。
