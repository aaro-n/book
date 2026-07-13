# Flutter + Rust 客户端方案 — 跨平台离线 PDF 阅读器

> **定位**：本方案是 Phase 3（完整版）的客户端替代方案，取代原计划的 Tauri/Electron。
> **核心理念**：Flutter 负责 UI 跨平台，Rust 负责本地计算（分词、搜索），实现**完全离线可用**的 PDF 阅读体验。

---

## 一、为什么选择 Flutter + Rust？

### 与候选方案对比

| | React Web（现方案） | Tauri/Electron | Flutter（纯 Dart） | **Flutter + Rust** |
|---|---|---|---|---|
| 平台覆盖 | Web only | 桌面 3 平台 | 6 平台 | 6 平台 |
| 离线模式 | ❌ 困难 | ⚠️ 可做 | ✅ | ✅ **最佳** |
| PDF 渲染 | PDF.js | 浏览器内核 | 原生级 | 原生级 |
| 中文分词 | ❌ 纯前端不可行 | 可内嵌 | dart_jieba (弱) | jieba-rs (强) |
| 本地全文搜索 | ❌ | 可内嵌 | SQLite FTS5 | tantivy (10x) |
| 移动端 | ❌ | ❌ | ✅ | ✅ |
| 原生性能 | 受浏览器限制 | 好 | 好 | **极致** |
| AI 编程效率 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |

---

## 二、整体架构

```
┌──────────────────────────────────────────────────────┐
│                   Flutter UI 层                       │
│                                                      │
│  ┌─────────┐ ┌──────────┐ ┌────────┐ ┌───────────┐ │
│  │ PDF阅读器│ │ 标记面板  │ │ 搜索框  │ │ 文档列表   │ │
│  │ (手势选 │ │ (高亮/注释│ │ (实时   │ │ (本地+云端 │ │
│  │  择/缩放)│ │  /书签)   │ │  联想)  │ │  双数据源) │ │
│  └────┬─────┘ └────┬─────┘ └───┬────┘ └─────┬─────┘ │
│       │             │           │             │       │
│  ┌────┴─────────────┴───────────┴─────────────┴───┐ │
│  │               状态管理层 (Riverpod)              │ │
│  │  ├─ AuthState       ├─ DocumentState            │ │
│  │  ├─ ReaderState     └─ SearchState              │ │
│  └──────────────────────┬───────────────────────────┘ │
│                         │                              │
│  ┌──────────────────────┴───────────────────────────┐ │
│  │              flutter_rust_bridge                  │ │
│  │           (自动生成 Dart ↔ Rust FFI)               │ │
│  └──────────────────────┬───────────────────────────┘ │
│                         │                              │
└─────────────────────────┼──────────────────────────────┘
                          │
┌─────────────────────────┴──────────────────────────────┐
│                     Rust 核心层                          │
│                                                         │
│  ┌──────────────┐ ┌───────────────┐ ┌───────────────┐ │
│  │ 分词引擎       │ │ 本地搜索引擎    │ │ PDF 处理      │ │
│  │ jieba-rs      │ │ tantivy       │ │ pdf-extract   │ │
│  │ ├─ 中文分词    │ │ ├─ 倒排索引    │ │ ├─ 文本提取    │ │
│  │ ├─ 词性标注    │ │ ├─ 高亮片段    │ │ ├─ 元数据读取  │ │
│  │ └─ 自定义词典  │ │ └─ 模糊搜索    │ │ └─ 目录解析    │ │
│  └──────┬───────┘ └──────┬────────┘ └──────┬────────┘ │
│         │                │                  │          │
│  ┌──────┴────────────────┴──────────────────┴──────┐  │
│  │             数据持久化层                           │  │
│  │  ├─ SQLite (sqflite)  ← 标记/书签/进度            │  │
│  │  ├─ 文件系统          ← PDF 缓存 + tantivy 索引   │  │
│  │  └─ 安全存储          ← Token/密钥                │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │             网络同步层                              │ │
│  │  ├─ HTTP Client   ← 与 Go 后端 REST API 通信       │ │
│  │  ├─ 离线队列      ← 无网络时暂存操作                │ │
│  │  └─ 冲突解决      ← 多端标记合并策略                 │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 三、技术栈详解

### 3.1 Flutter 侧

| 组件 | 选择 | 说明 |
|------|------|------|
| **框架** | Flutter 3.22+ | 最新 stable |
| **状态管理** | Riverpod 2+ | 编译期安全，比 Provider 更强 |
| **PDF 渲染** | `syncfusion_flutter_pdfviewer` | 商业免费（社区许可），标注能力最强 |
| **本地数据库** | `sqflite` + `drift` | drift 提供类型安全 SQL |
| **HTTP 客户端** | `dio` | 拦截器、重试、离线缓存 |
| **路由** | `go_router` | 声明式路由，支持深度链接 |
| **Rust 桥接** | `flutter_rust_bridge` v2 | 自动生成 Dart 绑定代码 |
| **图标/组件** | Material 3 | Flutter 内置 |

### 3.2 Rust 侧

| 组件 | 选择 | 说明 |
|------|------|------|
| **中文分词** | `jieba-rs` | 与后端同款算法，词典共享 |
| **全文搜索** | `tantivy` 0.22+ | Lucene 的 Rust 实现，性能远超 SQLite FTS5 |
| **PDF 文本提取** | `pdf-extract` | 提取纯文本供分词/索引 |
| **加密** | `ring` + `aes-gcm` | 本地数据加密 |
| **JSON 序列化** | `serde_json` | 与 Flutter 侧 JSON 通信 |

### 3.3 Cargo.toml（Rust 依赖）

```toml
[package]
name = "pdf_reader_core"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "staticlib"]  # 动态库 + 静态库

[dependencies]
flutter_rust_bridge = "2"

# 分词
jieba-rs = "0.7"

# 全文搜索
tantivy = "0.22"

# PDF
pdf-extract = "0.8"

# 序列化
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# 加密
ring = "0.17"
aes-gcm = "0.10"

# 错误处理
anyhow = "1"
thiserror = "2"
```

### 3.4 pubspec.yaml（Flutter 依赖）

```yaml
dependencies:
  flutter:
    sdk: flutter

  # 状态管理
  flutter_riverpod: ^2.5
  riverpod_annotation: ^2.3

  # PDF
  syncfusion_flutter_pdfviewer: ^25.1

  # 数据库
  sqflite: ^2.3
  drift: ^2.18
  sqlite3_flutter_libs: ^0.5

  # 网络
  dio: ^5.4

  # 路由
  go_router: ^14.0

  # Rust 桥接
  flutter_rust_bridge: ^2.0

  # 存储
  path_provider: ^2.1
  flutter_secure_storage: ^9.2

  # UI
  google_fonts: ^6.2
  flutter_colorpicker: ^1.1  # 高亮颜色选择

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_rust_bridge_codegen: ^2.0  # 代码生成器
  drift_dev: ^2.18
  build_runner: ^2.4
```

---

## 四、项目结构

```
pdf-reader/
│
├── lib/                            # Flutter (Dart)
│   ├── main.dart                   # 应用入口
│   │
│   ├── app/
│   │   ├── app.dart                # MaterialApp 配置
│   │   ├── router.dart             # 路由定义
│   │   └── theme.dart              # 主题/暗黑模式
│   │
│   ├── features/
│   │   ├── auth/
│   │   │   ├── login_page.dart
│   │   │   ├── register_page.dart
│   │   │   └── auth_provider.dart
│   │   │
│   │   ├── documents/
│   │   │   ├── document_list_page.dart
│   │   │   ├── document_detail_page.dart
│   │   │   └── document_provider.dart
│   │   │
│   │   ├── reader/
│   │   │   ├── reader_page.dart        ← PDF 阅读器主界面
│   │   │   ├── highlight_panel.dart    ← 高亮/注释侧边栏
│   │   │   ├── bookmark_panel.dart     ← 书签面板
│   │   │   └── reader_provider.dart
│   │   │
│   │   └── search/
│   │       ├── search_page.dart
│   │       ├── search_results_page.dart
│   │       └── search_provider.dart
│   │
│   ├── services/
│   │   ├── api_client.dart             ← HTTP 请求封装
│   │   ├── local_db_service.dart        ← SQLite 操作
│   │   ├── sync_service.dart            ← 离线同步
│   │   └── rust_bridge_service.dart     ← Rust FFI 调用封装
│   │
│   ├── models/
│   │   ├── user.dart
│   │   ├── document.dart
│   │   ├── highlight.dart
│   │   ├── bookmark.dart
│   │   └── search_result.dart
│   │
│   └── widgets/                        ← 通用组件
│       ├── loading_indicator.dart
│       ├── error_widget.dart
│       └── color_picker.dart
│
├── rust/                           # Rust 核心
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── api/
│       │   ├── mod.rs              ← flutter_rust_bridge 入口
│       │   ├── segmenter.rs        ← 分词 API
│       │   ├── search.rs           ← tantivy 搜索 API
│       │   └── pdf.rs              ← PDF 处理 API
│       │
│       ├── segmenter/
│       │   ├── mod.rs
│       │   ├── jieba_impl.rs       ← jieba-rs 封装
│       │   └── dict_manager.rs     ← 自定义词典管理
│       │
│       ├── search/
│       │   ├── mod.rs
│       │   ├── index_builder.rs    ← tantivy 索引构建
│       │   ├── query_parser.rs     ← 搜索查询解析
│       │   └── highlighter.rs     ← 搜索结果高亮
│       │
│       ├── pdf/
│       │   ├── mod.rs
│       │   └── extractor.rs        ← PDF 文本提取
│       │
│       └── sync/
│           ├── mod.rs
│           └── encrypt.rs          ← 本地数据加密
│
├── assets/                         # 静态资源
│   ├── dict/                       # 分词词典
│   │   ├── dict.txt                # jieba 默认词典
│   │   └── custom_dict.txt         # 自定义词典
│   └── fonts/                      # 字体文件
│
├── test/                           # 测试
├── integration_test/               # 集成测试
│
├── ios/                            # iOS 原生配置
├── android/                        # Android 原生配置
├── macos/                          # macOS 原生配置
├── windows/                        # Windows 原生配置
├── linux/                          # Linux 原生配置
│
├── pubspec.yaml                    # Flutter 依赖
├── analysis_options.yaml           # Dart 代码规范
└── Makefile                        # 构建脚本
```

---

## 五、核心功能实现

### 5.1 Rust 本地搜索 API

```rust
// rust/src/api/search.rs
use flutter_rust_bridge::frb;
use tantivy::schema::*;
use tantivy::{Index, IndexWriter, ReloadPolicy};
use jieba_rs::Jieba;

/// 搜索结果
#[frb]
pub struct SearchResult {
    pub doc_id: i64,
    pub page_num: i32,
    pub snippet: String,    // 带高亮标签的片段
    pub score: f32,
}

/// 创建本地搜索索引（应用首次启动时调用）
#[frb]
pub fn create_index(index_path: String) -> anyhow::Result<()> {
    let mut schema_builder = Schema::builder();
    schema_builder.add_i64_field("doc_id", INDEXED | STORED);
    schema_builder.add_i32_field("page_num", INDEXED | STORED);
    schema_builder.add_text_field("content", TEXT | STORED);  // 分词后文本

    let schema = schema_builder.build();
    let index = Index::create_in_dir(&index_path, schema)?;

    // 注册 jieba 分词器
    let tokenizer = TextTokenizer::for_language(
        tantivy::tokenizer::Language::Chinese
    );
    index.tokenizers().register("jieba", tokenizer);

    Ok(())
}

/// 向索引中添加文档
#[frb]
pub fn add_to_index(
    index_path: String,
    doc_id: i64,
    page_num: i32,
    raw_text: String,
) -> anyhow::Result<()> {
    let segmenter = Jieba::new();
    let words: Vec<&str> = segmenter.cut(&raw_text, false);
    let segmented = words.join(" ");

    let index = Index::open_in_dir(&index_path)?;
    let mut writer: IndexWriter = index.writer(50_000_000)?;

    let schema = index.schema();
    let doc_id_field = schema.get_field("doc_id").unwrap();
    let page_field = schema.get_field("page_num").unwrap();
    let content_field = schema.get_field("content").unwrap();

    let mut doc = Document::default();
    doc.add_i64(doc_id_field, doc_id);
    doc.add_i32(page_field, page_num);
    doc.add_text(content_field, &segmented);

    writer.add_document(doc)?;
    writer.commit()?;

    Ok(())
}

/// 本地全文搜索
#[frb]
pub fn local_search(
    index_path: String,
    query: String,
    limit: i32,
) -> anyhow::Result<Vec<SearchResult>> {
    // 1. 对查询分词
    let segmenter = Jieba::new();
    let terms: Vec<String> = segmenter
        .cut(&query, false)
        .iter()
        .filter(|w| w.len() > 1)
        .map(|w| w.to_string())
        .collect();

    // 2. 构造 tantivy 查询
    let index = Index::open_in_dir(&index_path)?;
    let reader = index.reader_builder()
        .reload_policy(ReloadPolicy::OnCommitWithDelay)
        .try_into()?;
    let searcher = reader.searcher();
    let schema = index.schema();
    let content_field = schema.get_field("content").unwrap();

    let query_str = terms.join(" OR ");
    let query_parser = QueryParser::for_index(&index, vec![content_field]);
    let query = query_parser.parse_query(&query_str)?;

    // 3. 执行搜索
    let top_docs = searcher.search(&query, &TopDocs::with_limit(limit as usize))?;
    let doc_id_field = schema.get_field("doc_id").unwrap();
    let page_field = schema.get_field("page_num").unwrap();

    let mut results = Vec::new();
    for (_score, doc_address) in top_docs {
        let doc = searcher.doc(doc_address)?;
        results.push(SearchResult {
            doc_id: doc.get_first(doc_id_field).and_then(|v| v.as_i64()).unwrap_or(0),
            page_num: doc.get_first(page_field).and_then(|v| v.as_i32()).unwrap_or(0),
            snippet: "...".into(),  // 实际应用中用 highlighter
            score: _score,
        });
    }

    Ok(results)
}
```

### 5.2 Dart 侧调用

```dart
// lib/services/rust_bridge_service.dart
import 'package:flutter_rust_bridge/flutter_rust_bridge.dart';
import 'package:pdf_reader/rust/api/search.dart' as rust_search;

class LocalSearchService {
  final String _indexPath;

  LocalSearchService(this._indexPath);

  /// 初始化本地索引（首次启动）
  Future<void> initIndex() async {
    await rust_search.createIndex(indexPath: _indexPath);
  }

  /// 为文档建立本地搜索索引
  Future<void> indexDocument(int docId, String rawText) async {
    // 按段落/页面拆分
    final pages = rawText.split('\n\n--- Page Break ---\n\n');
    for (var i = 0; i < pages.length; i++) {
      await rust_search.addToIndex(
        indexPath: _indexPath,
        docId: docId,
        pageNum: i + 1,
        rawText: pages[i],
      );
    }
  }

  /// 本地搜索
  Future<List<rust_search.SearchResult>> search(String query) async {
    return await rust_search.localSearch(
      indexPath: _indexPath,
      query: query,
      limit: 20,
    );
  }
}
```

### 5.3 离线同步策略

```dart
// lib/services/sync_service.dart
class SyncService {
  final ApiClient _api;
  final LocalDbService _localDb;
  final _pendingQueue = <SyncOperation>[];

  /// 用户操作时：先写本地，加入同步队列
  Future<void> createHighlight(HighlightInput input) async {
    // 1. 立即写入本地 SQLite（用户无感知）
    final localId = await _localDb.insertHighlight(input);

    // 2. 加入同步队列
    _pendingQueue.add(SyncOperation.createHighlight(input, localId));

    // 3. 如果有网络，立即同步
    if (await hasNetwork()) {
      await _flushQueue();
    }
  }

  /// 联网后：批量同步
  Future<void> _flushQueue() async {
    while (_pendingQueue.isNotEmpty) {
      final op = _pendingQueue.first;
      try {
        switch (op.type) {
          case SyncType.createHighlight:
            final remoteId = await _api.createHighlight(op.data);
            await _localDb.updateHighlightRemoteId(op.localId, remoteId);
          // ... 其他操作
        }
        _pendingQueue.removeAt(0);
      } catch (e) {
        // 失败保留在队列中，下次重试
        break;
      }
    }
  }

  /// 应用启动时：拉取云端更新
  Future<void> pullFromCloud() async {
    final lastSync = await _localDb.getLastSyncTime();
    final updates = await _api.getUpdatesSince(lastSync);
    for (final update in updates) {
      await _localDb.upsert(update);
    }
    await _localDb.updateLastSyncTime(DateTime.now());
  }
}
```

---

## 六、PDF 阅读器实现要点

### 6.1 渲染方案

```dart
// lib/features/reader/reader_page.dart
class ReaderPage extends ConsumerStatefulWidget {
  final int documentId;
  const ReaderPage({required this.documentId});
  // ...
}

class _ReaderPageState extends ConsumerState<ReaderPage> {
  final PdfViewerController _pdfController = PdfViewerController();
  final TextSelectionControls _selectionControls = MaterialTextSelectionControls();

  @override
  Widget build(BuildContext context) {
    final doc = ref.watch(documentProvider(widget.documentId));

    return Scaffold(
      appBar: AppBar(title: Text(doc.title)),
      body: Stack(
        children: [
          // PDF 渲染层
          SfPdfViewer.network(
            doc.s3Url,
            controller: _pdfController,
            onTextSelectionChanged: _onTextSelected,  // ← 选中文本触发
          ),

          // 高亮覆盖层（读取本地/云端标记绘制）
          HighlightOverlay(
            highlights: ref.watch(highlightsProvider(widget.documentId)),
          ),
        ],
      ),

      // 底部工具栏
      bottomSheet: ReaderToolbar(
        onHighlight: _addHighlight,
        onBookmark: _addBookmark,
        onSearch: _openSearch,
      ),
    );
  }

  void _onTextSelected(PdfTextSelectionChangedDetails details) {
    if (details.selectedText == null) return;

    showModalBottomSheet(
      context: context,
      builder: (_) => HighlightPanel(
        selectedText: details.selectedText!,
        pageNum: _pdfController.pageNumber,
        bounds: details.globalSelectedRegion!,  // 坐标
      ),
    );
  }
}
```

### 6.2 高亮覆盖层

```dart
// lib/features/reader/highlight_overlay.dart
class HighlightOverlay extends StatelessWidget {
  final List<Highlight> highlights;

  @override
  Widget build(BuildContext context) {
    return IgnorePointer(
      child: CustomPaint(
        painter: _HighlightPainter(highlights),
        size: Size.infinite,
      ),
    );
  }
}

class _HighlightPainter extends CustomPainter {
  final List<Highlight> highlights;

  @override
  void paint(Canvas canvas, Size size) {
    for (final h in highlights) {
      final rect = Rect.fromLTWH(
        h.bbox.x, h.bbox.y, h.bbox.width, h.bbox.height,
      );
      final color = Color(int.parse(h.color.replaceFirst('#', '0xFF')));
      canvas.drawRect(
        rect,
        Paint()..color = color.withOpacity(0.3),
      );
    }
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) => true;
}
```

---

## 七、Rust ↔ Flutter 数据模型

### 共享类型定义

```rust
// rust/src/api/models.rs
#[frb]
pub struct LocalDocument {
    pub id: i64,
    pub title: String,
    pub page_count: i32,
    pub cached_path: Option<String>,  // 本地缓存路径
    pub indexed: bool,                 // 是否已建 tantivy 索引
}
```

```dart
// lib/models/local_document.dart
// flutter_rust_bridge 自动从上面的 Rust struct 生成
// 也可手动定义并手动转换
class LocalDocument {
  final int id;
  final String title;
  final int pageCount;
  final String? cachedPath;
  final bool indexed;

  LocalDocument({required this.id, required this.title, ...});
}
```

---

## 八、构建与部署

### 8.1 构建命令

```bash
# 1. 生成 Rust ↔ Dart 绑定代码
flutter_rust_bridge_codegen generate

# 2. Android
flutter build apk --release

# 3. iOS
flutter build ios --release

# 4. macOS
flutter build macos --release

# 5. Windows
flutter build windows --release

# 6. Linux
flutter build linux --release
```

### 8.2 包体积预估

| 平台 | 安装包大小 | 说明 |
|------|-----------|------|
| Android APK | ~60 MB | 含 jieba 词典 (~15MB) + tantivy 索引 |
| iOS IPA | ~50 MB | App Store 瘦身后 |
| macOS DMG | ~40 MB | 桌面端最小 |
| Windows MSI | ~45 MB | 含 MSVC runtime |
| Linux AppImage | ~38 MB | 静态链接 |

### 8.3 CI/CD（GitHub Actions）

```yaml
name: Build All Platforms

on:
  push:
    tags: ['v*']

jobs:
  rust-core:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd rust/ && cargo test
      - run: cd rust/ && cargo clippy -- -D warnings

  flutter-android:
    needs: rust-core
    uses: ./.github/workflows/flutter-build.yml
    with:
      platform: android

  flutter-ios:
    needs: rust-core
    uses: ./.github/workflows/flutter-build.yml
    with:
      platform: ios

  flutter-desktop:
    needs: rust-core
    strategy:
      matrix:
        os: [macos-latest, windows-latest, ubuntu-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter build ${{ matrix.os == 'macos-latest' && 'macos' || matrix.os == 'windows-latest' && 'windows' || 'linux' }} --release
```

---

## 九、开发路线图

### Phase 3a：Flutter 客户端 MVP（4 周）

| 周 | 任务 |
|----|------|
| 第 1 周 | Flutter 项目搭建 + `flutter_rust_bridge` 集成 + Rust 分词 API 跑通 |
| 第 2 周 | PDF 阅读器基础（渲染 + 翻页 + 缩放）+ 与后端 API 对接 |
| 第 3 周 | 高亮/注释/书签（本地 SQLite + 远端同步） |
| 第 4 周 | 本地全文搜索（tantivy 建索引 + 查询 + 结果高亮） |

### Phase 3b：离线完善（2 周）

| 周 | 任务 |
|----|------|
| 第 5 周 | 离线模式（离线队列 + 冲突解决 + 首次启动引导） |
| 第 6 周 | 暗黑模式 + 自定义词典同步 + 性能优化 |

### Phase 3c：发版（2 周）

| 周 | 任务 |
|----|------|
| 第 7 周 | 各平台打包 + 测试 + Bug 修复 |
| 第 8 周 | 应用商店上架（App Store / Google Play / 官网下载） |

---

## 十、风险与缓解

| 风险 | 等级 | 缓解措施 |
|------|------|----------|
| `flutter_rust_bridge` 生成代码失败 | 🟡 中 | 锁定版本，CI 中验证生成 |
| tantivy 索引体积过大 | 🟢 低 | 仅索引文本，不存 PDF；压缩索引 |
| jieba 词典包体大 | 🟢 低 | 提供"精简词典"选项（~5MB） |
| 不同平台 PDF 渲染不一致 | 🟡 中 | `syncfusion` 多平台测试；备选 `pdfx` |
| AI 无法很好生成 FFI 代码 | 🟡 中 | 提供模板代码，AI 只修改业务逻辑 |

---

## 十一、与后端的分工

```
┌─────────────────────────────────────────────────┐
│                  Go 后端（不变）                   │
│                                                  │
│  ✅ 用户认证（JWT 签发/验证）                      │
│  ✅ 文档元数据管理（上传/列表/删除）                │
│  ✅ S3 文件存储（PDF 原文）                        │
│  ✅ 服务端搜索引擎（Elasticsearch/Meilisearch）     │
│  ✅ 标记数据权威存储（PG）                          │
│  ✅ 自定义词典管理（PG + 热更新）                   │
│  ✅ 多端同步 API（提供 since 时间戳增量拉取）       │
└─────────────────────────────────────────────────┘
                         ↑ REST API
┌────────────────────────┴────────────────────────┐
│              Flutter + Rust 客户端                │
│                                                  │
│  ✅ PDF 本地渲染 + 缓存                           │
│  ✅ 本地中文分词 (jieba-rs)                       │
│  ✅ 本地全文搜索 (tantivy)                        │
│  ✅ 标记本地存储 (SQLite)                         │
│  ✅ 离线操作队列 + 自动同步                       │
│  ✅ 冲突解决（最后写入胜出 / 用户选择）            │
└─────────────────────────────────────────────────┘
```

---

## 十二、总结

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Flutter + Rust 是 PDF 阅读器客户端的最优解               │
│                                                          │
│  ✅ 一次编写，6 个平台分发                                 │
│  ✅ 真正的离线可用（分词 + 搜索 + 存储全部本地）           │
│  ✅ 原生性能（PDF 渲染 > 浏览器 10x）                     │
│  ✅ 后端零改动（共用同一套 REST API）                     │
│  ✅ 未来可扩展笔记/同步/协作                               │
│                                                          │
│  ⚠️ 学习成本高：Dart + Rust + FFI 三件套                  │
│  ⚠️ AI 编程效率中等：FFI 是薄弱地带                       │
│  ⚠️ 建议 MVP 阶段用 React Web，Phase 3 再启动客户端       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```
