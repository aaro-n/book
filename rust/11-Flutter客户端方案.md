# Flutter + Rust 客户端方案 — 跨平台离线 PDF 阅读器

> **定位**：本方案是 Phase 3（完整版）的客户端替代方案，取代原计划的 Tauri/Electron。
> **核心理念**：Flutter 负责 UI 跨平台，Rust 负责本地计算（分词、搜索），实现**完全离线可用**的 PDF 阅读体验。

> **前端分词**：客户端 jieba-rs FFI 分词用户查询 → 发送已分词结果到服务端 → 服务端直接走索引路径。搜索体验和服务端分层策略详见 `05-分词搜索方案.md`。

---

## 一、为什么选择 Flutter + Rust？

### 与候选方案对比

| | React Web（现方案） | Tauri/Electron | Flutter（纯 Dart） | **Flutter + Rust** |
|---|---|---|---|---|
| 平台覆盖 | Web only | 桌面 3 平台 | 6 平台 | 6 平台 |
| 离线模式 | ❌ 困难 | ⚠️ 可做 | ✅ | ✅ **最佳** |
| PDF 渲染 | PDF.js | 浏览器内核 | 原生级 | 原生级 |
| 中文分词 | ❌ 纯前端不可行 | 可内嵌 | dart_jieba (弱) | jieba-rs FFI (强) |
| 本地全文搜索 | ❌ | 可内嵌 | SQLite FTS5 | SQLite FTS5（从服务端同步分词数据） |
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
│  │ 分词引擎       │ │ 离线搜索入口    │ │ PDF 处理      │ │
│  │ jieba-rs      │ │ (查询分词后    │ │ pdf-extract   │ │
│  │ ├─ 中文分词    │ │  调 SQLite)    │ │ ├─ 文本提取    │ │
│  │ ├─ 词性标注    │ │               │ │ ├─ 元数据读取  │ │
│  │ └─ 自定义词典  │ │               │ │ └─ 目录解析    │ │
│  └──────┬───────┘ └──────┬────────┘ └──────┬────────┘ │
│         │                │                  │          │
│  ┌──────┴────────────────┴──────────────────┴──────┐  │
│  │             数据持久化层                           │  │
│  │  ├─ SQLite (sqflite)  ← 标记/书签/进度/FTS5搜索    │  │
│  │  ├─ 文件系统          ← PDF 缓存                   │  │
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
| **中文分词** | `jieba-rs` | 与后端同款算法，词典共享。仅负责分词查询词和离线导入时的文本分词 |
| **PDF 文本提取** | `pdf-extract` | 提取纯文本供分词/索引 |
| **加密** | `ring` + `aes-gcm` | 本地数据加密 |
| **JSON 序列化** | `serde_json` | 与 Flutter 侧 JSON 通信 |

> **离线搜索**：由 Dart 侧 SQLite FTS5 直接实现，不需要 Tantivy。Rust 层仅保留 jieba-rs 分词（查询时用）。

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
│   │   ├── local_db_service.dart        ← SQLite 操作（含 FTS5）
│   │   ├── local_search_service.dart    ← FTS5 离线搜索
│   │   ├── sync_service.dart            ← 离线同步（增量拉取分词数据）
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
│       │   └── pdf.rs              ← PDF 处理 API
│       │
│       ├── segmenter/
│       │   ├── mod.rs
│       │   ├── jieba_impl.rs       ← jieba-rs 封装
│       │   └── dict_manager.rs     ← 自定义词典管理
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

### 5.1 离线搜索架构：从服务端同步分词数据

**核心思路**：服务端在导入时已完成 jieba-rs 分词 → `segmented_text`。客户端下载分词数据 → 直接写入 SQLite FTS5 → 离线全文搜索。

```
服务端 DB: segmented_text = "人工 智能 正在 改变 世界"
        ↓ 增量同步 API
客户端 SQLite: INSERT INTO search_index VALUES ('人工 智能 正在 改变 世界')
        ↓
FTS5 unicode61 分词器按空格切 → ["人工","智能","正在","改变","世界"]
        ↓
用户搜索: jieba-rs FFI 分词 → "人工 智能"
        ↓
SELECT * FROM search_index WHERE search_index MATCH '人工 智能'
```

**为什么不用 Tantivy？** 文本已在服务端分好词，SQLite FTS5 + `unicode61` 按空格建索引完全够用。省去 Rust 侧 Tantivy 的 ~5MB 依赖和额外索引文件。

### 5.2 Rust 侧：仅保留分词（查询时用）

```rust
// rust/src/api/segmenter.rs
use flutter_rust_bridge::frb;
use jieba_rs::Jieba;
use std::sync::OnceLock;

static JIEBA: OnceLock<Jieba> = OnceLock::new();

fn get_jieba() -> &'static Jieba {
    JIEBA.get_or_init(|| Jieba::new())
}

/// 分词用户搜索查询（前端调用，离线可用）
#[frb]
pub fn segment_query(query: String) -> String {
    let jieba = get_jieba();
    jieba.cut(&query, true).join(" ")
    // "人工智能技术" → "人工 智能 技术"
}

/// 分词批量文本（离线导入时用，本地自建索引）
#[frb]
pub fn segment_batch(texts: Vec<String>) -> Vec<String> {
    let jieba = get_jieba();
    texts.into_iter()
        .map(|t| jieba.cut(&t, true).join(" "))
        .collect()
}
```

### 5.3 Dart 侧：SQLite FTS5 搜索 + 从服务端同步

```dart
// lib/services/local_search_service.dart

class LocalSearchService {
  final Database _db;
  final ApiClient _api;

  /// 建 FTS5 虚拟表（应用首次启动时）
  Future<void> initSchema() async {
    await _db.execute('''
      CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
        doc_id,
        page_num,
        title,
        segmented_text,
        tokenize='unicode61'  -- 按空格切分，匹配已分词文本
      )
    ''');
  }

  /// 从服务端增量同步分词数据
  Future<void> syncFromServer({DateTime? since}) async {
    final pages = await _api.syncDocumentContent(since: since);

    final batch = _db.batch();
    for (final p in pages) {
      batch.insert('search_index', {
        'doc_id': p.docId,
        'page_num': p.pageNum,
        'title': p.title,
        'segmented_text': p.segmentedText,  // 已分词，直接存
      }, conflictAlgorithm: ConflictAlgorithm.replace);
    }
    await batch.commit(noResult: true);
  }

  /// 本地搜索（离线可用）
  Future<List<SearchResult>> localSearch(String rawQuery) async {
    // 1. 前端 jieba-rs FFI 分词查询词
    final segmented = await RustLib.instance.api
        .crateApiSegmenterSegmentQuery(query: rawQuery);
    // rawQuery = "人工智能技术"
    // segmented = "人工 智能 技术"

    // 2. FTS5 MATCH
    return _db.rawQuery('''
      SELECT doc_id, page_num, title,
             snippet(search_index, 1, '<em>', '</em>', '...', 40) AS snippet,
             bm25(search_index) AS score
      FROM search_index
      WHERE search_index MATCH ?
      ORDER BY score
      LIMIT 20
    ''', [segmented]).then((rows) => rows.map(_toResult).toList());
  }
}
```

### 5.4 服务端配套 API

```
GET /api/v1/documents/sync-content?since=2024-01-01T00:00:00Z
```

```json
[
  {
    "doc_id": 101,
    "title": "人工智能导论",
    "page_num": 5,
    "segmented_text": "人工 智能 正在 改变 世界"
  }
]
```

**增量同步**：客户端记录 `last_synced_at`，每次只拉新内容。100 本以内全量同步无压力（~20MB），超过 100 本增量同步（只拉新书）。

### 5.5 离线搜索流程

```
┌── 在线时 ─────────────────────────────────────┐
│  ① 首次打开 App                                │
│     → GET /api/v1/documents/sync-content?since=0 │
│     → 全量拉取 segmented_text                   │
│     → 写入 SQLite FTS5                          │
│                                               │
│  ② 后续增量                                    │
│     → GET ?since=上次同步时间                   │
│     → 只拉新书/新页                              │
│     → 追加 FTS5 索引                            │
│                                               │
│  ③ 用户搜索                                    │
│     → jieba-rs FFI 分词查询词                   │
│     → SQLite FTS5 MATCH（毫秒级）               │
│     → snippet() 高亮 + bm25() 排序              │
│     → 在线时再搜云端补充                         │
└───────────────────────────────────────────────┘

┌── 离线时 ─────────────────────────────────────┐
│  用户搜 "人工智能"                              │
│     → jieba-rs FFI → "人工 智能"               │
│     → SQLite FTS5 MATCH                        │
│     → 秒出结果 ✅                               │
│                                               │
│  不需要下载 PDF 原件                            │
│  不需要重复提取文字                              │
│  不需要重复分词                                 │
│  不需要 Tantivy 依赖                            │
└───────────────────────────────────────────────┘
```

### 5.6 离线同步策略

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
| Android APK | ~45 MB | 含 jieba 词典 (~15MB)，无 Tantivy |
| iOS IPA | ~40 MB | App Store 瘦身后 |
| macOS DMG | ~30 MB | 桌面端最小 |
| Windows MSI | ~35 MB | 含 MSVC runtime |
| Linux AppImage | ~28 MB | 静态链接 |

> 去除 Tantivy 后，每平台减少约 5MB。SQLite FTS5 与 sqflite 共用同一数据库文件。离线搜索索引从服务端同步，不额外占安装包体积。

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
| 第 4 周 | 离线全文搜索（SQLite FTS5 建表 + 从服务端同步分词数据 + 查询 + bm25 排序） |

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
| SQLite FTS5 大数据量性能 | 🟢 低 | 100 本内全量同步 < 20MB；超过 100 本增量同步 |
| jieba 词典包体大 | 🟢 低 | 提供"精简词典"选项（~5MB） |
| 不同平台 PDF 渲染不一致 | 🟡 中 | `syncfusion` 多平台测试；备选 `pdfx` |

---

## 十一、与后端的分工

```
┌─────────────────────────────────────────────────┐
│                 Rust 后端（不变）                  │
│                                                  │
│  ✅ 用户认证（JWT 签发/验证）                      │
│  ✅ 文档元数据管理（上传/列表/删除）                │
│  ✅ S3 文件存储（PDF/EPUB 原文 + 拆分页面）         │
│  ✅ 服务端搜索引擎（四层架构）                      │
│  ✅ 标记数据权威存储（PG + W3C Web Annotation）     │
│  ✅ 自定义词典管理（PG + 热更新）                   │
│  ✅ 分词数据同步 API（供客户端下载 segmented_text）  │
└─────────────────────────────────────────────────┘
                         ↑ REST API
┌────────────────────────┴────────────────────────┐
│              Flutter + Rust 客户端                │
│                                                  │
│  ✅ PDF 本地渲染 + 缓存                           │
│  ✅ 本地中文分词 (jieba-rs FFI)                    │
│  ✅ 离线全文搜索 (SQLite FTS5，从服务端同步分词数据)│
│  ✅ 标记本地存储 (SQLite) + W3C Annotation 同步    │
│  ✅ 离线操作队列 + 自动同步                       │
│  ✅ 冲突解决（最后写入胜出 / 用户选择）            │
└─────────────────────────────────────────────────┘
```

---

## 十二、标注能力路线图

### 三阶段规划

```
Phase 1 MVP（90% 日常场景）:
  ✅ 高亮（多色）
  ✅ 下划线 / 波浪线
  ✅ 删除线
  ✅ 文字笔记
  ✅ 书签

Phase 2 进阶（iPad 笔记场景）:
  ⬜ 手写/涂鸦 (freehand)
  ⬜ 画圈/矩形/箭头
  ⬜ 便签贴 (sticky_note)
  ⬜ 图章（⭐重要 / ❓疑问 / ✅已读）

Phase 3 完整（审阅/校对场景）:
  ⬜ 文字框（页面任意位置打字）
  ⬜ 多边形/折线
  ⬜ 文档内跳转链接
  ⬜ 遮盖/涂黑
```

### 跨端兼容原则

| 端 | 创建能力 | 展示能力 | 说明 |
|----|:---:|:---:|------|
| **Flutter 客户端** | 全部类型 | 全部类型 | 原生绘制，完整能力 |
| **Web 端** | 文本型（Phase 1） | 全部类型 | 图形型通过 SVG/Canvas 只读渲染 |

> 统一存储于 `annotations` 表（详见 `04-数据库设计.md`），W3C Web Annotation JSON-LD 格式（详见 `07-API接口定义.md`）。多端同步冲突处理见 `13-同步方案研究.md`。

### 标注可搜索性

| 类型 | 可搜索 | 方式 |
|------|:---:|------|
| highlight/underline/squiggly/strikethrough | ✅ | 高亮文本 + jieba-rs 分词 → `segmented_text` |
| note/sticky_note/freetext | ✅ | 笔记文字 + 分词 |
| stamp | ✅ | 图章文字（"重要"/"疑问"） |
| freehand/circle/rectangle/line/polygon | ❌ | 仅按 page_num + type 过滤 |
| link | ❌ | 仅按 page_num 过滤 |
| redact | ❌ | 管理中可见，普通不可见 |

---

## 十三、总结

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
