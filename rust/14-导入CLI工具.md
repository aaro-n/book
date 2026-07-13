# 导入 CLI 工具 — 独立 Rust 命令行工具

> **定位**：独立于服务端和阅读器客户端的纯 Rust CLI 工具。处理服务端不擅长的重型任务（OCR、EPUB 解析、大文件、特殊语言），输出标准 `.bookpkg` 压缩包上传到服务端。

---

## 一、为什么独立？

| 理由 | 说明 |
|------|------|
| **编译体积** | 服务端不需要 Tesseract/OCR 依赖，二进制小很多 |
| **依赖冲突** | 导入工具可能用不同版本的 PDF 库，互不干扰 |
| **独立发布** | CLI 工具可以单独发版，不需要动服务端和客户端 |
| **安全隔离** | 导入工具跑在用户电脑上，服务端不暴露重型处理能力 |

**三者的唯一联系**：`.bookpkg` 压缩包格式。

---

## 二、处理能力

### 服务端 vs CLI 分工

CLI 处理服务端**做不了或做不好**的场景。服务端只处理能力范围内的（详见 `03-技术架构.md`）。

| 场景 | 服务端 (512MB) | CLI | 说明 |
|------|:---:|:---:|------|
| PDF 文字版, CJK/空格语言, <200MB | ✅ | — | `pdf-extract` 内存 ~60MB |
| PDF 扫描件（无文字层） | ❌ | ✅ | Tesseract OCR 需 200-400MB |
| PDF > 200MB | ❌ | ✅ | 内存扛不住 |
| PDF 特殊语言（泰语/阿拉伯语） | ❌ | ✅ | 服务端不装对应分词 |
| EPUB, CJK/空格语言, <50MB | ✅ | — | 解压 + 去标签，内存 ~20MB |
| EPUB, 特殊语言/超大/大量图片 | — | ✅ | 需对应分词 + 图片渲染 |
| 任何 MOBI | — | ✅ | 服务端不做格式转换 |

### CLI 全部能力

| 能力 | 说明 |
|------|------|
| OCR 扫描件 | Tesseract OCR，支持多语言包 |
| EPUB 解析 | 解压 + 提取 XHTML + 目录 + 封面 |
| MOBI 解析 | 格式转换为 EPUB 后再处理 |
| 任意语言分词 | 自动检测 → jieba-rs / 空格切分 / Unicode 单词边界，全语言覆盖 |
| 重型 PDF | >200MB 大文件，特殊语言 PDF |
| 含大量图片的 EPUB | 渲染章节截图 → 打包书页 |
| 拆页渲染 | PDF → WebP 预览图 + 单页 PDF |
| 缩略图 | 生成书架列表用的小图 |
| 打包 | 所有产物打包为 `.bookpkg`，可选自动上传 |

---

## 三、.bookpkg 压缩包格式

`.bookpkg` = tar.gz 改名，一个文件包含所有产物：

```
三体.bookpkg
│
├── package.json               ← 元数据 + 分页内容 + 分词结果
│
├── original.epub              ← 整本原件（备份 / 客户端离线阅读）
│
├── chapters/                  ← EPUB：单章 XHTML（Web 原生渲染）
│   chapter-01.xhtml            ← 服务端直接流式渲染，不用 PDF.js
│   chapter-02.xhtml
│   ...
│
├── pages/                     ← PDF：单页 PDF（Web 阅读器，可选中文字）
│   page_001.pdf
│   ...
│
├── images/                    ← WebP 预览图（快速翻页 / 搜索结果）
│   page_001.webp
│   ...
│
├── thumbs/                    ← 缩略图（书架网格列表）
│   page_001.jpg
│   ...
│
├── images-epub/               ← EPUB 内嵌图片（封面、插图等）
│   cover.jpg
│   figure-01.png
│   ...
│
└── toc.json                   ← 目录结构
```

### package.json 结构

```json
{
  "version": "1.0",
  "source": "book-import v0.2.1",
  "document": {
    "title": "三体",
    "author": "刘慈欣",
    "file_hash": "sha256:abc123...",
    "page_count": 34,
    "language": "zh",
    "mime_type": "application/epub+zip",
    "cover_image": "images-epub/cover.jpg"
  },
  "pages": [
    {
      "page_num": 1,
      "raw_text": "第一章  科学边界\n...",
      "segmented_text": "第一章 科学 边界 ...",
      "chapter": "chapters/chapter-01.xhtml",
      "image": "images/page_001.webp",
      "thumb": "thumbs/page_001.jpg",
      "width": 1200,
      "height": 1700,
      "size_bytes": 204800
    }
  ],
  "toc": [
    { "level": 1, "title": "第一章 科学边界", "page": 1 },
    { "level": 1, "title": "第二章 台球", "page": 2 }
  ]
}
```

> **PDF 格式**：`page_num` = 物理页码；`page_pdf` = 单页 PDF 路径（可选）
> **EPUB 格式**：`page_num` = 章节序号；`chapter` = XHTML 章节路径；无 `page_pdf` 字段

服务端处理 `.bookpkg`：

```
解包 → original.pdf/epub → S3 (归档)
     → chapters/*.xhtml  → S3 (EPUB 章节，Web 原生渲染)
     → pages/*.pdf       → S3 (PDF 按页存取)
     → images/*.webp     → S3
     → thumbs/*.jpg      → S3
     → package.json      → 逐页 INSERT INTO document_content
```

### 语言处理总览

CLI 的目标是**处理任何语言**。策略自动检测，用户无需手动指定：

```
任何语言的文本
      │
      ▼
┌─ 自动检测 ──────────────────────────────┐
│                                         │
│  CJK (中/日/韩)     → jieba-rs 分词     │
│  含空格 (英/俄/阿…)  → split_whitespace  │
│  其余所有语言        → Unicode 单词边界   │
│                                         │
│  全语言覆盖 ✅                            │
└─────────────────────────────────────────┘
```

| 策略 | 触发条件 | 示例语言 | 实现 |
|------|---------|---------|------|
| **jieba-rs** | 语言标记为 `zh`/`ja`/`ko` 或自动检测为 CJK | 中文、日文、韩文 | `jieba.cut(text, true)` |
| **空格切分** | 文本包含空格字符 | 英文、法语、俄文、**阿拉伯文**、希伯来文等 | `text.split_whitespace()` |
| **Unicode 单词边界** | 兜底：非 CJK 且无空格 | 泰语、高棉语、老挝语、缅甸语、**任何未知语言** | `UnicodeSegmentation::split_word_bounds()` |

> **阿拉伯语**：本身按空格分隔词，自动命中策略 2。前端不需要任何处理。
> **泰语/高棉语等**：自动命中策略 3。前端用 `Intl.Segmenter` 浏览器原生 API。
> **任何未知语言**：策略 3 兜底。`split_word_bounds()` 基于 Unicode 标准 Annex #29，覆盖所有定义了单词边界的文字系统。

---

## 四、配置

### 配置文件 `~/.config/book-import/config.toml`

```toml
[server]
# 服务端地址，--upload 时用到
url = "https://my-books.example.com"
# API 密钥（建议用环境变量 BOOK_TOKEN 覆盖）
token = ""

[ocr]
# OCR 引擎: "tesseract" | "none"
# none = 遇到扫描件直接报错跳过
engine = "tesseract"
# OCR 语言包，逗号分隔
languages = ["chi_sim", "chi_tra", "eng", "jpn"]
# 多语言时是否自动检测（慢但准）
auto_language = false
# 临时文件目录，默认系统临时目录
temp_dir = "/tmp/book-import"

[pages]
# 单页图片渲染 DPI
image_dpi = 150
# 图片格式: "webp" | "png" | "jpg"
image_format = "webp"
# WebP 质量 (1-100)
image_quality = 75
# 是否生成单页 PDF
generate_pdf = true
# 缩略图宽度（等比缩放）
thumb_width = 200

[segmentation]
# 中文分词引擎: "jieba"
engine = "jieba"
# 自定义词典路径
user_dict = "~/.config/book-import/dict.txt"
# 语言处理：默认自动检测，以下为手动覆盖（一般不需要改）
# override_lang = "zh"  # 强制按中文分词

[processing]
# 并发页数（OCR/渲染时并行处理）
concurrency = 4
# 单文件大小上限，超过则拒绝 OCR（避免内存爆炸）
max_file_size_mb = 500

[output]
# 默认输出目录，为空则输出在当前目录
dir = ""
# 压缩等级 (1-9)
compression = 6
# 是否保留中间文件（调试用）
keep_temp = false

[logging]
level = "info"  # trace | debug | info | warn | error
```

### 环境变量

```bash
export BOOK_SERVER_URL="https://my-books.example.com"
export BOOK_TOKEN="sk-xxxx"              # API 密钥
export BOOK_OCR_LANGS="chi_sim,eng"      # 默认 OCR 语言
export BOOK_CONCURRENCY=4                # 默认并发数
```

### 配置优先级

```
命令行参数  >  环境变量  >  配置文件  >  默认值
```

---

## 五、命令行使用

```bash
# 基本用法
book-import ./三体.epub              # → 生成 三体.bookpkg
book-import ./战争与和平.pdf          # → 生成 战争与和平.bookpkg

# 处理并直接上传
book-import --upload ./百年孤独.pdf   # → 生成 → 上传 → 服务端自动导入

# 批量导入
book-import --batch ./books/          # → 遍历目录逐个处理

# 只预览，不生成文件
book-import --dry-run ./扫描书.pdf
# 输出: 🔍 检测: PDF扫描件, 342页, 预计OCR耗时 5min

# 覆盖配置
book-import --config ~/my-config.toml \     # 指定配置文件
            --server https://alt-server.com \# 覆盖服务端地址
            --ocr-langs "chi_sim,eng" \      # 只装中英文OCR
            --image-dpi 200 \               # 高清渲染
            --image-format png \            # 无损
            --no-pdf \                      # 不生成单页PDF
            --concurrency 2 \               # 低配电脑
            --output ~/books/ \             # 输出到指定目录
            --verbose \
            ./三体.epub
```

### 场景化预设

```bash
# 低配电脑
book-import --concurrency 1 --image-dpi 100 --no-pdf ./书.pdf

# 高清归档
book-import --image-dpi 300 --image-format png --compression 1 ./珍藏.pdf

# 批量导入并上传
book-import --upload --batch ./books/

# 只提取文字+分词，不生成图片（纯搜索用）
book-import --no-images --no-pdf ./论文集合.pdf
```

---

## 六、默认值（零配置可用）

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| OCR 引擎 | `tesseract` | 扫描件自动 OCR |
| OCR 语言 | `chi_sim,eng` | 中英文 |
| 图片 DPI | 150 | 适合屏幕阅读 |
| 图片格式 | `webp` | 体积最小 |
| 并发数 | CPU 核心数 | 自动检测 |
| 自定义词典 | 无 | 用 jieba 内置词典 |
| 输出目录 | 当前目录 | `./书名.bookpkg` |

---

## 七、项目结构（独立仓库）

```
book-import/                    # 独立 Git 仓库
├── Cargo.toml                  # 依赖: clap, tesseract, epub, pdf, jieba-rs, flate2, tar
├── src/
│   ├── main.rs                 # CLI 入口 (clap)
│   ├── config.rs               # 配置加载 (toml + env)
│   ├── pdf/
│   │   ├── detect.rs           # PDF 类型检测 (文字版 vs 扫描件)
│   │   ├── extract.rs          # 文字提取 (lopdf)
│   │   └── render.rs           # 页面渲染 (pdfium/poppler)
│   ├── epub/
│   │   └── parser.rs           # EPUB 解析
│   ├── ocr/
│   │   └── tesseract.rs        # Tesseract OCR 封装
│   ├── segmenter.rs            # jieba-rs 分词 + 空格语言 + Unicode 断字
│   ├── package.rs              # .bookpkg 打包/解包
│   └── upload.rs               # HTTP 上传到服务端
└── config/
    └── default.toml            # 默认配置
```

---

## 八、与 Go 版本对照

| 能力 | Rust CLI | Go CLI |
|------|----------|--------|
| OCR | ✅ Tesseract FFI | ⚠️ 需 CGO |
| PDF 渲染 | ✅ pdfium/poppler | ⚠️ 依赖系统库 |
| EPUB 解析 | ✅ 纯 Rust | ✅ 纯 Go |
| 分词 (CJK) | ✅ jieba-rs | ✅ gse/jieba |
| 分词 (空格语言) | ✅ split_whitespace | ✅ strings.Fields |
| 分词 (泰语等) | ✅ unicode-segmentation | ✅ golang.org/x/text |
| 打包 | ✅ flate2+tar | ✅ archive/tar |
| 单二进制 | ✅ 静态链接 | ✅ 静态编译 |
