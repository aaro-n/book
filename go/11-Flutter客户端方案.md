# Flutter + Go 后端 客户端方案 — 跨平台 PDF 阅读器

> **定位**：本方案是 Phase 3（完整版）的客户端方案，取代原计划的 Tauri/Electron。
> **核心理念**：Flutter 负责 UI 跨平台，Go 后端负责全部计算（分词、搜索），通过 REST API 通信，本地 SQLite 提供基础离线能力。
>
> Rust 版对照见 `../rust/11-Flutter客户端方案.md`（含 Rust 原生本地计算层）。

---

## 一、为什么选择 Flutter + Go 后端？

### 与候选方案对比

| | React Web（现方案） | Tauri/Electron | **Flutter + Go 后端** | Flutter + Rust |
|---|---|---|---|---|
| 平台覆盖 | Web only | 桌面 3 平台 | 6 平台 | 6 平台 |
| 离线模式 | ❌ 困难 | ⚠️ 可做 | ✅ 基础离线 | ✅ **最佳** |
| PDF 渲染 | PDF.js | 浏览器内核 | 原生级 | 原生级 |
| 中文分词 | 后端 API | 可内嵌 | 后端 Go (GSE 原生) | jieba-rs (强) |
| 本地全文搜索 | ❌ | 可内嵌 | SQLite FTS5 | tantivy (10x) |
| 移动端 | ❌ | ❌ | ✅ | ✅ |
| 原生性能 | 受浏览器限制 | 好 | 好 | **极致** |
| **AI 编程效率** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **开发复杂度** | 低 | 中 | **中** | 高 |

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
│  │                数据与服务层                         │ │
│  │  ├─ dio (HTTP)      ← 与 Go 后端 REST API 通信     │ │
│  │  ├─ sqflite/drift   ← 本地 SQLite (标记/进度)      │ │
│  │  ├─ path_provider   ← PDF 本地缓存                 │ │
│  │  └─ secure_storage  ← Token/密钥                   │ │
│  └──────────────────────┬───────────────────────────┘ │
│                         │                              │
└─────────────────────────┼──────────────────────────────┘
                          │ HTTPS
┌─────────────────────────┴──────────────────────────────┐
│                     Go 后端                             │
│                                                         │
│  ┌──────────────┐ ┌───────────────┐ ┌───────────────┐ │
│  │ 分词引擎       │ │ 全文搜索引擎    │ │ PDF 处理      │ │
│  │ GSE + Jieba   │ │ PG tsvector    │ │ pdfcpu        │ │
│  │ ├─ 中文分词    │ │ ├─ 倒排索引     │ │ ├─ 文本提取    │ │
│  │ ├─ 词性标注    │ │ ├─ 高亮片段     │ │ ├─ 元数据读取  │ │
│  │ └─ 热更新词典  │ │ └─ 模糊搜索     │ │ └─ 目录解析    │ │
│  └──────────────┘ └───────────────┘ └───────────────┘ │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │              API 层 (Gin)                           │ │
│  │  ├─ /api/v1/auth/*        ← 认证                   │ │
│  │  ├─ /api/v1/documents/*   ← 文档 CRUD               │ │
│  │  ├─ /api/v1/search        ← 全文搜索                │ │
│  │  ├─ /api/v1/annotations/* ← 标记同步                │ │
│  │  └─ /api/v1/sync/*        ← 进度同步                │ │
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
| **本地数据库** | `sqflite` + `drift` | drift 提供类型安全 SQL，含 FTS5 全文搜索 |
| **HTTP 客户端** | `dio` | 拦截器、重试、离线缓存 |
| **路由** | `go_router` | 声明式路由，支持深度链接 |
| **图标/组件** | Material 3 | Flutter 内置 |

### 3.2 Go 后端（已有，不变）

| 组件 | 选择 | 说明 |
|------|------|------|
| **Web 框架** | Gin v1.9+ | 轻量高性能 |
| **中文分词** | GSE + Jieba (Go) | 热更新原生支持 |
| **全文搜索** | PostgreSQL tsvector + GIN | 应用层 GSE 分词 → tsvector 存储 |
| **PDF 处理** | pdfcpu | 纯 Go，文本提取 + 元数据 |
| **数据库** | PostgreSQL 18 | 权威数据存储 |
| **缓存** | Redis | 热点数据 |
| **S3** | AWS SDK Go v2 | PDF 文件存储 |

### 3.3 pubspec.yaml（Flutter 依赖）

```yaml
dependencies:
  flutter:
    sdk: flutter

  # 状态管理
  flutter_riverpod: ^2.5
  riverpod_annotation: ^2.3

  # PDF
  syncfusion_flutter_pdfviewer: ^25.1

  # 数据库（本地 SQLite + FTS5 离线搜索）
  sqflite: ^2.3
  drift: ^2.18
  sqlite3_flutter_libs: ^0.5

  # 网络
  dio: ^5.4

  # 路由
  go_router: ^14.0

  # 存储
  path_provider: ^2.1
  flutter_secure_storage: ^9.2

  # UI
  google_fonts: ^6.2
  flutter_colorpicker: ^1.1  # 高亮颜色选择

  # JSON 序列化
  json_annotation: ^4.9
  freezed_annotation: ^2.4

dev_dependencies:
  flutter_test:
    sdk: flutter
  drift_dev: ^2.18
  build_runner: ^2.4
  json_serializable: ^6.8
  freezed: ^2.5
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
│   │       ├── search_page.dart        ← 搜索入口
│   │       ├── search_results_page.dart
│   │       └── search_provider.dart
│   │
│   ├── services/
│   │   ├── api_client.dart             ← dio 封装 + 拦截器
│   │   ├── local_db_service.dart        ← SQLite 操作 (drift)
│   │   ├── sync_service.dart            ← 离线同步队列
│   │   └── cache_service.dart           ← PDF 本地缓存管理
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
├── test/                           # 单元测试
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

> 与 Rust 版的关键区别：**无 `rust/` 目录、无 `flutter_rust_bridge`**。离线搜索用 SQLite FTS5，分词/搜索核心能力在 Go 后端。

---

## 五、核心功能实现

### 5.1 搜索架构（在线优先 + 离线回退）

```dart
// lib/services/search_service.dart

enum SearchMode { online, offline, hybrid }

class SearchService {
  final ApiClient _api;
  final LocalDbService _localDb;

  /// 搜索策略：在线优先，离线回退
  Future<SearchResult> search(String query, {int docId = 0}) async {
    if (await _hasNetwork) {
      try {
        // 在线：调用 Go 后端 API（GSE 分词 + PG tsvector）
        return await _api.search(query, docId: docId);
      } catch (e) {
        // 网络故障 → 回退到本地 SQLite FTS5
        return _localSearch(query, docId: docId);
      }
    } else {
      // 离线：本地 SQLite FTS5
      return _localSearch(query, docId: docId);
    }
  }

  /// 本地 SQLite FTS5 搜索（离线能力）
  Future<SearchResult> _localSearch(String query, {int docId = 0}) async {
    // SQLite FTS5 支持前缀匹配和简单分词
    return await _localDb.searchWithFTS5(query, docId: docId);
  }
}
```

### 5.2 SQLite FTS5 本地索引

```dart
// lib/services/local_db_service.dart (drift 定义)

// drift 表定义
@DataClassName('DocumentContent')
class DocumentContents extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get documentId => integer()();
  IntColumn get pageNum => integer()();
  TextColumn get rawText => text()();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
}

// FTS5 虚拟表（用于全文搜索）
// drift 不支持直接创建 FTS5，需要用自定义 SQL
class LocalDbService {
  Future<void> createFTS5Table(Database db) async {
    await db.execute('''
      CREATE VIRTUAL TABLE IF NOT EXISTS document_content_fts 
      USING fts5(document_id, page_num, content, tokenize='unicode61')
    ''');
  }

  Future<void> indexDocument(int docId, int pageNum, String text) async {
    final db = await _database;
    await db.execute(
      'INSERT INTO document_content_fts(document_id, page_num, content) VALUES (?, ?, ?)',
      [docId, pageNum, text],
    );
  }

  Future<List<Map<String, dynamic>>> searchWithFTS5(
    String query, {int docId = 0}
  ) async {
    final db = await _database;
    // FTS5 查询语法：支持前缀 *
    final ftsQuery = query.split(' ').map((w) => '"$w"*').join(' AND ');

    var sql = '''
      SELECT document_id, page_num, snippet(document_content_fts, 1, '<b>', '</b>', '...', 40) as snippet
      FROM document_content_fts
      WHERE document_content_fts MATCH ?
    ''';
    final args = <dynamic>[ftsQuery];

    if (docId > 0) {
      sql += ' AND document_id = ?';
      args.add(docId);
    }

    sql += ' ORDER BY rank LIMIT 20';
    return await db.rawQuery(sql, args);
  }
}
```

### 5.3 离线同步策略

```dart
// lib/services/sync_service.dart

enum SyncStatus { pending, syncing, synced, failed }

class SyncOperation {
  final String id;
  final SyncType type;
  final Map<String, dynamic> data;
  final int localId;
  SyncStatus status;
  int retryCount;

  SyncOperation({
    required this.type, required this.data, required this.localId,
    this.status = SyncStatus.pending, this.retryCount = 0,
  }) : id = const Uuid().v4();
}

class SyncService {
  final ApiClient _api;
  final LocalDbService _localDb;
  final _pendingQueue = <SyncOperation>[];

  /// 用户操作时：先写本地，加入同步队列
  Future<void> createHighlight(HighlightInput input) async {
    final localId = await _localDb.insertHighlight(input);
    _pendingQueue.add(SyncOperation(
      type: SyncType.createHighlight, data: input.toJson(), localId: localId,
    ));

    if (await _hasNetwork()) {
      await _flushQueue();
    }
  }

  /// 联网后：批量同步
  Future<void> _flushQueue() async {
    final toSync = _pendingQueue
        .where((op) => op.status == SyncStatus.pending)
        .toList();

    for (final op in toSync) {
      op.status = SyncStatus.syncing;
      try {
        switch (op.type) {
          case SyncType.createHighlight:
            final remoteId = await _api.createHighlight(op.data);
            await _localDb.updateRemoteId(op.localId, remoteId);
          case SyncType.updateHighlight:
            await _api.updateHighlight(op.data['id'], op.data);
          case SyncType.deleteHighlight:
            await _api.deleteHighlight(op.data['id']);
        }
        op.status = SyncStatus.synced;
        _pendingQueue.remove(op);
      } catch (e) {
        op.status = SyncStatus.failed;
        op.retryCount++;
        if (op.retryCount > 10) {
          _pendingQueue.remove(op); // 放弃超过 10 次重试的
        }
        break; // 失败暂停，下次再试
      }
    }
  }

  /// 应用启动时：拉取云端更新（增量）
  Future<void> pullFromCloud() async {
    final lastSync = await _localDb.getLastSyncTime();
    final updates = await _api.getUpdatesSince(lastSync);
    for (final update in updates) {
      await _localDb.upsert(update);
    }
    await _localDb.updateLastSyncTime(DateTime.now());
  }

  /// 网络状态监听
  Future<bool> _hasNetwork() async {
    try {
      final result = await InternetAddress.lookup('google.com');
      return result.isNotEmpty && result[0].rawAddress.isNotEmpty;
    } catch (_) {
      return false;
    }
  }
}
```

### 5.4 Go 后端搜索 API 调用

```dart
// lib/services/api_client.dart

class ApiClient {
  final Dio _dio;

  ApiClient({required String baseUrl})
    : _dio = Dio(BaseOptions(
        baseUrl: baseUrl,
        connectTimeout: const Duration(seconds: 10),
        receiveTimeout: const Duration(seconds: 30),
      )) {
    _dio.interceptors.addAll([
      AuthInterceptor(),
      LogInterceptor(requestBody: true, responseBody: true),
      CacheInterceptor(),
    ]);
  }

  /// 调用 Go 后端 GSE 分词 + PG tsvector 搜索
  Future<SearchResponse> search(String query, {int docId = 0}) async {
    final params = <String, dynamic>{'q': query};
    if (docId > 0) params['document_id'] = docId;

    final response = await _dio.get('/api/v1/search', queryParameters: params);
    return SearchResponse.fromJson(response.data);
  }

  /// 获取文档内容（用于离线缓存时建本地 FTS5 索引）
  Future<DocumentContent> getDocumentContent(int docId) async {
    final response = await _dio.get('/api/v1/documents/$docId/content');
    return DocumentContent.fromJson(response.data);
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
}

class _ReaderPageState extends ConsumerState<ReaderPage> {
  final PdfViewerController _pdfController = PdfViewerController();

  @override
  Widget build(BuildContext context) {
    final doc = ref.watch(documentProvider(widget.documentId));

    return Scaffold(
      appBar: AppBar(title: Text(doc.title)),
      body: Stack(
        children: [
          // PDF 渲染层 — 从 Go 后端 S3 签名 URL 加载
          SfPdfViewer.network(
            doc.s3Url,   // Go 后端签发临时 S3 URL
            controller: _pdfController,
            onTextSelectionChanged: _onTextSelected,
          ),

          // 高亮覆盖层
          HighlightOverlay(
            highlights: ref.watch(highlightsProvider(widget.documentId)),
          ),
        ],
      ),
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
        bounds: details.globalSelectedRegion!,
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

## 七、数据模型（纯 Dart，无 FFI）

```dart
// lib/models/document.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'document.freezed.dart';
part 'document.g.dart';

@freezed
class Document with _$Document {
  const factory Document({
    required int id,
    required String title,
    required String s3Key,
    required int pageCount,
    String? description,
    @Default('auto') String language,
    @Default(false) bool isDeleted,
    required DateTime createdAt,
    required DateTime updatedAt,
  }) = _Document;

  factory Document.fromJson(Map<String, dynamic> json) =>
      _$DocumentFromJson(json);
}

// lib/models/highlight.dart
@freezed
class Highlight with _$Highlight {
  const factory Highlight({
    required int id,
    required int documentId,
    required int pageNum,
    required BBox bbox,
    required String highlightedText,
    String? note,
    @Default('#FFFF00') String color,
    required DateTime createdAt,
  }) = _Highlight;

  factory Highlight.fromJson(Map<String, dynamic> json) =>
      _$HighlightFromJson(json);
}

@freezed
class BBox with _$BBox {
  const factory BBox({
    required double x,
    required double y,
    required double width,
    required double height,
  }) = _BBox;

  factory BBox.fromJson(Map<String, dynamic> json) => _$BBoxFromJson(json);
}
```

---

## 八、构建与部署

### 8.1 构建命令

```bash
# 1. 生成 freezed/json_serializable 代码
dart run build_runner build --delete-conflicting-outputs

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

# 7. Web (可选，已有 React Web)
flutter build web --release
```

### 8.2 包体积预估（纯 Dart，无原生 Rust）

| 平台 | 安装包大小 | 说明 |
|------|-----------|------|
| Android APK | ~30 MB | 纯 Dart，无 Rust native 库 |
| iOS IPA | ~25 MB | App Store 瘦身后 |
| macOS DMG | ~20 MB | 桌面端最小 |
| Windows MSI | ~25 MB | Flutter 引擎 |
| Linux AppImage | ~22 MB | 无额外依赖 |

> 相比 Flutter+Rust 方案的 ~60MB，纯 Dart 方案包体减半。代价是离线搜索只能依赖 SQLite FTS5。

### 8.3 CI/CD（GitHub Actions）

```yaml
name: Build All Platforms

on:
  push:
    tags: ['v*']

jobs:
  flutter-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter test
      - run: flutter analyze

  flutter-android:
    needs: flutter-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter build apk --release

  flutter-ios:
    needs: flutter-test
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter build ios --release --no-codesign

  flutter-desktop:
    needs: flutter-test
    strategy:
      matrix:
        os: [macos-latest, windows-latest, ubuntu-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: |
          case "${{ matrix.os }}" in
            macos-latest)   flutter build macos --release ;;
            windows-latest) flutter build windows --release ;;
            ubuntu-latest)  flutter build linux --release ;;
          esac
```

---

## 九、开发路线图

### Phase 3a：Flutter 客户端 MVP（3 周）

| 周 | 任务 |
|----|------|
| 第 1 周 | Flutter 项目搭建 + 与 Go 后端 API 对接（登录、文档列表） |
| 第 2 周 | PDF 阅读器基础（渲染 + 翻页 + 缩放）+ 高亮/书签 UI |
| 第 3 周 | 搜索 UI + 离线 SQLite FTS5 索引 + 离线同步队列 |

### Phase 3b：离线完善（2 周）

| 周 | 任务 |
|----|------|
| 第 4 周 | 离线模式完整测试（离线队列 + 冲突解决 + 首次启动引导） |
| 第 5 周 | 暗黑模式 + 性能优化 + PDF 缓存管理 |

### Phase 3c：发版（2 周）

| 周 | 任务 |
|----|------|
| 第 6 周 | 各平台打包 + 测试 + Bug 修复 |
| 第 7 周 | 应用商店上架（App Store / Google Play / 官网下载） |

---

## 十、风险与缓解

| 风险 | 等级 | 缓解措施 |
|------|------|----------|
| SQLite FTS5 中文搜索不准确 | 🟡 中 | 在线时走 Go 后端 API（GSE 分词精度高）；离线 FTS5 仅作降级 |
| 大文档离线索引慢 | 🟢 低 | 后台线程建索引，分批提交 |
| PDF 缓存占用大 | 🟢 低 | LRU 策略，最大缓存 500MB |
| 离线同步冲突 | 🟡 中 | "最后写入胜出" 策略 + 用户手动选择 |
| 不同平台 PDF 渲染不一致 | 🟡 中 | `syncfusion` 多平台测试；备选 `pdfx` |

---

## 十一、与后端的分工

```
┌─────────────────────────────────────────────────┐
│                  Go 后端                          │
│                                                  │
│  ✅ 用户认证（JWT 签发/验证）                      │
│  ✅ 文档元数据管理（上传/列表/删除）                │
│  ✅ S3 文件存储（PDF 原文 + 临时签名 URL）         │
│  ✅ 中文分词（GSE + Jieba 原生热更新）             │
│  ✅ 全文搜索（PG tsvector + GIN 索引）            │
│  ✅ 标记数据权威存储（PG）                          │
│  ✅ 自定义词典管理（PG + 热更新）                   │
│  ✅ 多端同步 API（since 时间戳增量拉取）            │
└─────────────────────────────────────────────────┘
                         ↑ REST API (HTTPS)
┌────────────────────────┴────────────────────────┐
│              Flutter 客户端                       │
│                                                  │
│  ✅ PDF 本地渲染 + 缓存                           │
│  ✅ 在线搜索：调用 Go 后端 API（GSE 分词精度高）   │
│  ✅ 离线搜索：SQLite FTS5（降级方案）              │
│  ✅ 标记本地存储 (SQLite)                          │
│  ✅ 离线操作队列 + 自动同步                       │
│  ✅ 冲突解决（最后写入胜出 / 用户选择）            │
│  ✅ S3 临时签名 URL（Go 后端签发，安全可控）       │
└─────────────────────────────────────────────────┘
```

---

## 十二、Flutter+Go vs Flutter+Rust 对比

| 维度 | Flutter + Go 后端 | Flutter + Rust 本地 |
|------|-------------------|---------------------|
| 离线分词 | ❌ 需网络 | ✅ jieba-rs 本地 |
| 离线搜索精度 | 🟡 SQLite FTS5 一般 | ✅ tantivy 极高 |
| 安装包大小 | ~30 MB | ~60 MB |
| 开发复杂度 | **低**（纯 Dart） | 高（Dart + Rust + FFI） |
| AI 编程效率 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 热更新能力 | ✅ Flutter OTA | ⚠️ Rust 需重新编译 |
| 后端改动 | 零 | 零 |
| 适用场景 | **联网为主的阅读** | 完全离线阅读 |

---

## 十三、总结

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Flutter + Go 后端 是 MVP 阶段最务实的客户端方案          │
│                                                          │
│  ✅ 一次编写，6 个平台分发                                 │
│  ✅ 在线搜索精度高（Go 后端 GSE 分词）                    │
│  ✅ 基础离线可用（SQLite FTS5 + 离线队列）                 │
│  ✅ 原生性能（PDF 渲染 > 浏览器 10x）                     │
│  ✅ 开发效率高（纯 Dart，AI 友好）                        │
│  ✅ 包体积小（~30MB，比 Rust 方案减半）                   │
│  ✅ Go 后端零改动（共用同一套 REST API）                  │
│                                                          │
│  ⚠️ 离线搜索精度不如 Rust tantivy（可接受降级）           │
│  ⚠️ 离线分词不可用（搜索走 GSE 分词 API 或 FTS5 前缀匹配）│
│  ⚠️ MVP 阶段 React Web 先行，Phase 3 再启动客户端        │
│                                                          │
│  📌 升级路径：如需极致离线体验 → 参考 Rust 版本加本地层    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

> **Rust 版完整离线方案**见 `../rust/11-Flutter客户端方案.md`
