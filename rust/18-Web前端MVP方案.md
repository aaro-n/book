# Web 前端 MVP 方案 — React + Vite + TypeScript + Tailwind

> **设计原则**：AI 友好、零经验可开发、最小依赖、渐进增强。
> **参考基准**：Kavita (Angular 21)、Komga (Vue 2 + Vuetify)、Calibre-Web (Flask SSR + jQuery) 的实战分析。

---

## 一、三大项目前端方案调研结论

| | Kavita | Komga | Calibre-Web |
|---|---|---|---|
| **架构** | Angular 21 SPA | Vue 2 SPA | Flask SSR (Jinja2) |
| **UI 框架** | Bootstrap 5 + ng-bootstrap | Vuetify 2 (Material) | Bootstrap 3 + jQuery |
| **PDF 阅读** | ngx-extended-pdf-viewer (PDF.js) | Readium 引擎 | 原生 PDF.js |
| **EPUB 阅读** | 自研 EPUB.js 风格 | Readium 引擎 | EPUB.js |
| **漫画阅读** | 自研 Canvas 渲染器（6种模式） | v-carousel + 懒加载 | kthoom.js + Web Worker |
| **状态管理** | RxJS + Angular Signals | Vuex + persistedstate | 无（服务端 Session） |
| **实时通信** | SignalR WebSocket（30+ 事件） | SSE (EventSource) | AJAX (jQuery) |
| **构建工具** | Angular CLI (esbuild) | Vue CLI 5 (Webpack) | 无（传统 SSR） |

**关键发现**：
1. 阅读器是核心复杂度——三个项目都为 PDF/EPUB/漫画写了独立组件
2. Kavita 的 Canvas 自研太重，MVP 不需要；Komga 的 Vue 2 已过时
3. Calibre-Web 的 SSR 翻页整页刷新，2026 年不应该这么做
4. 实时通信不是必须的——Komga 用 SSE 做事件推送最轻量

---

## 二、技术选型

### 2.1 选型逻辑

> **核心理由**：React 是 AI 编程质量最高的前端框架。JSX 纯 JavaScript，AI 生成代码准确率远超 Angular 模板语法和 Vue SFC。

| | Angular | Vue | React ✅ |
|---|---|---|---|
| AI 生成准确率 | 低（装饰器 + 模板） | 中（SFC 多语法混合） | **高**（JSX = 纯 JS） |
| PDF 生态 | ngx-extended-pdf-viewer | 无成熟方案 | **react-pdf**（100万周下载） |
| 学习曲线 | 陡 | 平 | 平 |
| 包体积 | 大 | 中 | 小（Vite tree-shaking） |
| 本项目适合度 | 过度设计 | 可选 | **最合适** |

### 2.2 技术栈

```
React 19 + TypeScript 5.7
├── 构建: Vite 6
├── CSS: Tailwind CSS 4
├── UI 组件: shadcn/ui（复制源码，按需引入）
├── 路由: react-router-dom v7
├── 状态: zustand（1KB，无 boilerplate）
├── HTTP: fetch（原生，不装 axios）
├── PDF 阅读: react-pdf（基于 PDF.js）
├── EPUB 阅读: epubjs
├── 虚拟滚动: @tanstack/react-virtual
├── SSE: 原生 EventSource
├── 表单: react-hook-form + zod
└── 分词: jieba-rs WASM（仅在搜索输入框使用）
```

### 2.3 为什么是这些？

| 选择 | 替代方案 | 理由 |
|------|---------|------|
| **zustand** | Redux / Jotai | 最轻（1KB），API 最简，AI 最易生成 |
| **shadcn/ui** | MUI / Ant Design | 复制源码而非 npm 安装，打包体积最小 |
| **Tailwind** | CSS Modules | 原子化 CSS，AI 生成最准，零命名成本 |
| **react-pdf** | 直接 PDF.js | React 组件封装，省去 DOM 操作 |
| **@tanstack/react-virtual** | react-window | 支持动态高度（EPUB 章节不等高） |
| **fetch** | axios | 少一个依赖，Edge 环境原生支持 |
| **react-hook-form** | Formik | 性能最好（非受控），zod 集成最简 |
| **zod** | yup | TypeScript-first，类型推导完美 |

### 2.4 不装的依赖

| 不装 | 理由 |
|------|------|
| ❌ axios | `fetch` 原生够用 |
| ❌ Redux | zustand 替代 |
| ❌ MUI / Ant Design | shadcn/ui 替代 |
| ❌ moment.js | `Intl.DateTimeFormat` 原生 |
| ❌ lodash | `Array.map/filter/reduce` 原生够用 |

---

## 三、项目结构

```
web/
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.ts
│
├── public/
│   └── favicon.svg
│
└── src/
    ├── main.tsx                  # 入口
    ├── App.tsx                   # 根组件（路由 + 布局）
    │
    ├── components/               # 通用 UI 组件
    │   ├── ui/                   # shadcn/ui 组件（button, card, dialog...）
    │   ├── layout/
    │   │   ├── Shell.tsx         # 全局布局（导航栏 + 侧边栏 + 内容区）
    │   │   ├── Navbar.tsx        # 顶部导航
    │   │   └── Sidebar.tsx       # 侧边栏（书架分类）
    │   ├── BookCard.tsx          # 书架上的书卡片
    │   ├── BookGrid.tsx          # 书卡片网格
    │   ├── SearchBar.tsx         # 搜索框（含 jieba-rs WASM 分词）
    │   ├── PageViewer.tsx        # 单页渲染组件
    │   └── AnnotationLayer.tsx   # 高亮/注释覆盖层
    │
    ├── pages/                    # 页面级组件（每个 = 一个路由）
    │   ├── LoginPage.tsx         # 登录
    │   ├── RegisterPage.tsx      # 注册
    │   ├── HomePage.tsx          # 首页（最近阅读 + 统计）
    │   ├── LibraryPage.tsx       # 书架（分页网格）
    │   ├── SearchPage.tsx        # 搜索结果
    │   ├── BookDetailPage.tsx    # 书详情（元数据 + 标注列表）
    │   ├── ReaderPage.tsx        # 阅读器（核心页面）
    │   ├── SettingsPage.tsx      # 设置（主题、密码、API Key）
    │   └── AdminPage.tsx         # 管理（词典、系统状态）
    │
    ├── stores/                   # zustand 状态
    │   ├── authStore.ts          # 用户 + token
    │   ├── bookStore.ts          # 文档列表 + 搜索
    │   └── readerStore.ts        # 阅读器状态（当前页、进度、缩放）
    │
    ├── hooks/                    # 自定义 hooks
    │   ├── useApi.ts             # fetch 封装（自动带 token + 401 处理）
    │   ├── useSSE.ts             # SSE 事件订阅
    │   ├── useBookmarks.ts       # 书签 CRUD
    │   ├── useAnnotations.ts     # 标注 CRUD
    │   └── useProgression.ts     # 阅读进度同步
    │
    ├── lib/                      # 工具函数
    │   ├── api.ts                # API 端点常量 + 响应类型
    │   ├── csrf.ts               # CSRF token 读取
    │   ├── segment.ts            # jieba-rs WASM 封装
    │   └── utils.ts              # 通用工具（时间格式化等）
    │
    ├── types/                    # TypeScript 类型
    │   ├── document.ts           # Document, Page, Annotation
    │   ├── user.ts               # User, Session
    │   └── api.ts                # API 请求/响应类型
    │
    └── styles/
        └── globals.css           # Tailwind 指令 + 自定义变量
```

---

## 四、路由设计

```typescript
// src/App.tsx

const router = createBrowserRouter([
  {
    element: <Shell />,           // 带导航的布局
    children: [
      { path: "/",              element: <HomePage /> },
      { path: "/library",       element: <LibraryPage /> },
      { path: "/search",        element: <SearchPage /> },
      { path: "/books/:id",     element: <BookDetailPage /> },
      { path: "/read/:id",      element: <ReaderPage /> },
      { path: "/settings",      element: <SettingsPage /> },
      { path: "/admin",         element: <AdminPage /> },
    ],
  },
  {
    element: <AuthLayout />,      // 无导航的布局（登录页）
    children: [
      { path: "/login",         element: <LoginPage /> },
      { path: "/register",      element: <RegisterPage /> },
    ],
  },
]);
```

### 4.1 路由守卫

```typescript
// 认证守卫：未登录跳转 /login
function RequireAuth({ children }: { children: React.ReactNode }) {
  const token = useAuthStore(s => s.token);
  if (!token) return <Navigate to="/login" replace />;
  return children;
}

// Admin 守卫：非管理员跳首页
function RequireAdmin({ children }: { children: React.ReactNode }) {
  const user = useAuthStore(s => s.user);
  if (user?.role !== 'admin') return <Navigate to="/" replace />;
  return children;
}
```

### 4.2 页面功能清单（MVP）

| 路由 | 页面 | 核心功能 |
|------|------|---------|
| `/login` | 登录页 | 邮箱 + 密码登录，注册链接 |
| `/register` | 注册页 | 邮箱 + 用户名 + 密码 |
| `/` | 首页 | 最近阅读（封面 + 进度条）、快捷搜索、简单统计 |
| `/library` | 书架 | 分页网格（缩略图 + 标题）、排序、上传按钮 |
| `/search` | 搜索 | 搜索框（WASM 分词）、结果列表（含页码命中片段） |
| `/books/:id` | 书详情 | 元数据、页码列表、标注列表 |
| `/read/:id` | 阅读器 | PDF 渲染、翻页、高亮/注释（详见 §六） |
| `/settings` | 设置 | 修改密码、API Key 管理、主题切换 |
| `/admin` | 管理 | 自定义词典、搜索配置 |

---

## 五、状态管理（zustand）

### 5.1 authStore

```typescript
// src/stores/authStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AuthState {
  token: string | null;           // Session token（登录后设置）
  user: User | null;              // 当前用户信息
  setAuth: (token: string, user: User) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      setAuth: (token, user) => set({ token, user }),
      logout: () => set({ token: null, user: null }),
    }),
    {
      name: 'auth',               // localStorage key
      partialize: (state) => ({ token: state.token }), // 只持久化 token
    }
  )
);
```

### 5.2 bookStore

```typescript
// src/stores/bookStore.ts
import { create } from 'zustand';

interface BookState {
  books: Document[];
  total: number;
  page: number;
  searchQuery: string;
  isLoading: boolean;

  fetchBooks: (page?: number) => Promise<void>;
  search: (query: string) => Promise<void>;
  uploadBook: (file: File) => Promise<void>;
  deleteBook: (id: string) => Promise<void>;
}

export const useBookStore = create<BookState>((set, get) => ({
  books: [],
  total: 0,
  page: 1,
  searchQuery: '',
  isLoading: false,

  fetchBooks: async (page = 1) => {
    set({ isLoading: true });
    const res = await api.get(`/api/v1/documents?page=${page}&limit=24`);
    set({ books: res.items, total: res.total, page, isLoading: false });
  },

  search: async (query) => {
    set({ isLoading: true, searchQuery: query });
    const res = await api.get(`/api/v1/search?q=${encodeURIComponent(query)}`);
    set({ books: res.items, total: res.total, isLoading: false });
  },

  uploadBook: async (file) => {
    const form = new FormData();
    form.append('file', file);
    await api.post('/api/v1/documents/upload', form);
    get().fetchBooks(); // 刷新列表
  },

  deleteBook: async (id) => {
    await api.delete(`/api/v1/documents/${id}`);
    set(s => ({ books: s.books.filter(b => b.id !== id) }));
  },
}));
```

### 5.3 readerStore

```typescript
// src/stores/readerStore.ts
import { create } from 'zustand';

interface ReaderState {
  docId: string | null;
  currentPage: number;
  totalPages: number;
  pageUrls: PageUrl[];            // 预先生成的所有页面签名 URL
  scale: number;                  // 缩放 0.5 ~ 3.0
  fitting: 'width' | 'height' | 'auto';

  loadBook: (docId: string) => Promise<void>;
  goToPage: (page: number) => void;
  setScale: (scale: number) => void;
}

export const useReaderStore = create<ReaderState>((set, get) => ({
  docId: null,
  currentPage: 1,
  totalPages: 0,
  pageUrls: [],
  scale: 1.0,
  fitting: 'width',

  loadBook: async (docId) => {
    // 一次性获取所有页面的签名 URL（30 天有效）
    const res = await api.get(`/api/v1/documents/${docId}/pages?format=url`);
    set({
      docId,
      totalPages: res.page_count,
      pageUrls: res.pages,
      currentPage: 1,
    });
  },

  goToPage: (page) => {
    if (page >= 1 && page <= get().totalPages) {
      set({ currentPage: page });
    }
  },

  setScale: (scale) => set({ scale }),
}));
```

---

## 六、阅读器方案

### 6.1 对页渲染流程

> 架构见 `03-技术架构.md`，页面分为三层：缩略图（Layer 1）→ 预览图（Layer 2）→ 单页 PDF（Layer 3）。

```
翻到第 42 页：
  ① 立即显示 Layer 2 预览图: <img src="pageUrls[41].image_url" />   (秒开)
  ② 后台懒加载 Layer 3 单页 PDF: react-pdf <Document file={pageUrls[41].pdf_url} />
  ③ PDF 加载完成后，无缝替换预览图（透明度过渡）
```

### 6.2 ReaderPage 组件骨架

```tsx
// src/pages/ReaderPage.tsx

export function ReaderPage() {
  const { id } = useParams<{ id: string }>();
  const { currentPage, totalPages, pageUrls, scale, fitting, loadBook, goToPage } = useReaderStore();
  const { annotations, fetchAnnotations } = useAnnotations();

  useEffect(() => {
    if (id) loadBook(id);
  }, [id]);

  useEffect(() => {
    if (id) fetchAnnotations(id, currentPage);
  }, [id, currentPage]);

  if (!pageUrls.length) return <Loading />;

  const current = pageUrls[currentPage - 1];

  return (
    <div className="h-screen flex flex-col bg-gray-900">
      {/* 顶部工具栏 */}
      <ReaderToolbar
        page={currentPage}
        total={totalPages}
        onPageChange={goToPage}
        scale={scale}
        onScaleChange={setScale}
      />

      {/* 页面渲染区 */}
      <div className="flex-1 overflow-hidden flex items-center justify-center">
        <PageRenderer
          imageUrl={current.image_url}      // Layer 2: 预览图（秒开）
          pdfUrl={current.pdf_url}           // Layer 3: 单页 PDF（懒加载替换）
          scale={scale}
          fitting={fitting}
        />
        <AnnotationLayer annotations={annotations} scale={scale} />
      </div>

      {/* 底部缩略图导航 */}
      <ThumbnailStrip
        pages={pageUrls}
        current={currentPage}
        onSelect={goToPage}
      />
    </div>
  );
}
```

### 6.3 PageRenderer — 两层渲染

```tsx
// src/components/PageRenderer.tsx

import { useState } from 'react';
import { Document, Page } from 'react-pdf';

export function PageRenderer({ imageUrl, pdfUrl, scale, fitting }: Props) {
  const [pdfReady, setPdfReady] = useState(false);

  return (
    <div className="relative" style={{ transform: `scale(${scale})` }}>
      {/* Layer 2: 预览图 — 永远显示，秒开 */}
      <img
        src={imageUrl}
        alt=""
        className={pdfReady ? 'opacity-0' : 'opacity-100'}
      />

      {/* Layer 3: 单页 PDF — 懒加载，加载完替换预览图 */}
      <Document
        file={pdfUrl}
        onLoadSuccess={() => setPdfReady(true)}
        className={pdfReady ? 'absolute inset-0 opacity-100' : 'absolute inset-0 opacity-0'}
      >
        <Page
          pageNumber={1}
          width={fitting === 'width' ? containerWidth : undefined}
          renderTextLayer={true}     // 文字可选中
          renderAnnotationLayer={false} // 我们自己画标注
        />
      </Document>
    </div>
  );
}
```

### 6.4 AnnotationLayer — 高亮/注释覆盖层

```tsx
// src/components/AnnotationLayer.tsx

export function AnnotationLayer({ annotations, scale }: Props) {
  return (
    <div className="absolute inset-0 pointer-events-none">
      {annotations.map(a => (
        <div
          key={a.id}
          className="absolute pointer-events-auto cursor-pointer"
          style={{
            left: a.bbox.x * scale,
            top: a.bbox.y * scale,
            width: a.bbox.w * scale,
            height: a.bbox.h * scale,
            backgroundColor: a.color,
            opacity: a.annotation_type === 'highlight' ? 0.3 : undefined,
            borderBottom: a.annotation_type === 'underline' ? `2px solid ${a.color}` : undefined,
          }}
          onClick={() => showNotePopover(a)}
        >
          {a.annotation_type === 'note' && <NotePin color={a.color} />}
        </div>
      ))}
    </div>
  );
}
```

### 6.5 翻页逻辑

```typescript
// 键盘翻页
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if (e.key === 'ArrowRight' || e.key === 'ArrowDown') goToPage(currentPage + 1);
    if (e.key === 'ArrowLeft' || e.key === 'ArrowUp') goToPage(currentPage - 1);
  };
  window.addEventListener('keydown', handler);
  return () => window.removeEventListener('keydown', handler);
}, [currentPage]);

// 触屏滑动翻页
const { onTouchStart, onTouchEnd } = useSwipeGesture({
  onSwipeLeft: () => goToPage(currentPage + 1),
  onSwipeRight: () => goToPage(currentPage - 1),
});
```

### 6.6 阅读进度自动保存

```typescript
// src/hooks/useProgression.ts

export function useProgression(docId: string, currentPage: number) {
  const lastSaved = useRef(currentPage);

  useEffect(() => {
    const timer = setInterval(async () => {
      if (currentPage !== lastSaved.current) {
        lastSaved.current = currentPage;
        await api.put(`/api/v1/progression/${docId}`, {
          progression: currentPage / totalPages,
          references: [`#page=${currentPage}`],
          device: { id: deviceId, name: 'PDF Manager Web' },
        });
      }
    }, 5000); // 每 5 秒保存一次

    return () => clearInterval(timer);
  }, [docId, currentPage]);

  // 页面卸载时立即保存
  useEffect(() => {
    return () => {
      api.put(`/api/v1/progression/${docId}`, { /* ... */ });
    };
  }, []);
}
```

---

## 七、后端通信

### 7.1 fetch 封装（useApi）

```typescript
// src/hooks/useApi.ts

const API_BASE = import.meta.env.VITE_API_BASE || '';

async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = useAuthStore.getState().token;

  const headers: HeadersInit = {
    ...options.headers,
  };

  if (token) {
    headers['Authorization'] = `Bearer ${token}`;
  }

  // CSRF token（Web 端 cookie 登录场景）
  if (!token) {
    const csrf = getCsrfToken();
    if (csrf) headers['X-CSRF-Token'] = csrf;
  }

  const res = await fetch(`${API_BASE}${path}`, { ...options, headers });

  if (res.status === 401) {
    useAuthStore.getState().logout();
    window.location.href = '/login';
    throw new Error('Unauthorized');
  }

  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new ApiError(res.status, body.error?.message || res.statusText);
  }

  return res.json();
}

export const api = {
  get: <T>(path: string) => request<T>(path),
  post: <T>(path: string, body?: unknown) =>
    request<T>(path, { method: 'POST', body: JSON.stringify(body), headers: { 'Content-Type': 'application/json' } }),
  put: <T>(path: string, body?: unknown) =>
    request<T>(path, { method: 'PUT', body: JSON.stringify(body), headers: { 'Content-Type': 'application/json' } }),
  delete: <T>(path: string) => request<T>(path, { method: 'DELETE' }),
};
```

### 7.2 SSE 事件订阅（useSSE）

```typescript
// src/hooks/useSSE.ts

export function useSSE() {
  const token = useAuthStore(s => s.token);

  useEffect(() => {
    if (!token) return;

    const es = new EventSource(`/api/v1/events?token=${token}`);

    es.addEventListener('progress-created', (e) => {
      const data = JSON.parse(e.data);
      // 更新阅读进度（其他设备同步过来的）
    });

    es.addEventListener('annotation-changed', (e) => {
      // 标注被其他设备修改
    });

    es.addEventListener('document-imported', (e) => {
      // 新书入库 → 刷新书架
      useBookStore.getState().fetchBooks();
    });

    es.onerror = () => {
      // 连接断开，5 秒后重连（EventSource 默认行为）
    };

    return () => es.close();
  }, [token]);
}
```

### 7.3 文件上传

```tsx
// 上传 PDF/EPUB → 跳转书架
async function handleUpload(file: File) {
  const form = new FormData();
  form.append('file', file);

  const res = await fetch('/api/v1/documents/upload', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${token}` },
    body: form,
    // 不设 Content-Type，让浏览器自动设 multipart/form-data + boundary
  });

  if (!res.ok) throw new Error('Upload failed');
  // 刷新书架
  useBookStore.getState().fetchBooks();
}
```

### 7.4 文件代理访问

```html
<!-- 所有 S3 文件走统一代理端点，服务端 302 跳转到签名 URL -->
<img src="/files/{doc_id}/thumbs/001.jpg" />
<img src="/files/{doc_id}/images/001.webp" />
<img src="/files/{doc_id}/cover.jpg" />
```

```typescript
// 阅读器不走代理，直接使用预生成的签名 URL（已在 readerStore.pageUrls 中）
<Document file={pageUrls[41].pdf_url} />
```

---

## 八、搜索方案

### 8.1 jieba-rs WASM 前端分词

> 方案详见 `05-分词搜索方案.md`。前端用 WASM 分词，服务端零分词依赖。

```typescript
// src/lib/segment.ts

let segmenter: typeof import('jieba-rs') | null = null;

async function initSegmenter() {
  if (!segmenter) {
    segmenter = await import('jieba-rs');
    // 首次加载 ~2-5MB WASM，浏览器缓存
  }
}

export async function segmentQuery(input: string): Promise<string> {
  // 检测是否含 CJK 字符
  if (!/[一-龠々〆〤\u4E00-\u9FFF\u3400-\u4DBF]/.test(input)) {
    return input; // 非 CJK → 按空格切即可
  }

  try {
    await initSegmenter();
    const words = segmenter!.cut(input);
    return words.join(' ');
  } catch {
    // WASM 加载失败 → 降级，服务端走 LIKE 兜底
    return input;
  }
}
```

### 8.2 SearchBar 组件

```tsx
// src/components/SearchBar.tsx

export function SearchBar() {
  const [query, setQuery] = useState('');
  const navigate = useNavigate();

  async function handleSearch() {
    const segmented = await segmentQuery(query);
    navigate(`/search?q=${encodeURIComponent(segmented)}`);
  }

  return (
    <div className="flex gap-2">
      <Input
        value={query}
        onChange={e => setQuery(e.target.value)}
        onKeyDown={e => e.key === 'Enter' && handleSearch()}
        placeholder="搜索书名或内容..."
      />
      <Button onClick={handleSearch}>搜索</Button>
    </div>
  );
}
```

---

## 九、构建配置

### 9.1 package.json

```json
{
  "name": "pdf-manager-web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^7.0.0",
    "react-pdf": "^9.0.0",
    "pdfjs-dist": "^4.0.0",
    "react-hook-form": "^7.0.0",
    "@hookform/resolvers": "^3.0.0",
    "zod": "^3.0.0",
    "zustand": "^5.0.0",
    "jieba-rs": "^0.7.0",
    "@tanstack/react-virtual": "^3.0.0",
    "lucide-react": "^0.400.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "typescript": "~5.7.0",
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "tailwindcss": "^4.0.0",
    "@tailwindcss/vite": "^4.0.0"
  }
}
```

### 9.2 vite.config.ts

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    port: 5173,
    proxy: {
      '/api': 'http://localhost:8080',   // API 代理到 Rust 后端
      '/opds': 'http://localhost:8080',
      '/files': 'http://localhost:8080',
    },
  },
  build: {
    outDir: 'dist',
    rollupOptions: {
      output: {
        manualChunks: {
          'pdf': ['react-pdf', 'pdfjs-dist'],      // PDF 渲染单独分片，按需加载
          'jieba': ['jieba-rs'],                    // WASM 分词单独分片
        },
      },
    },
  },
});
```

### 9.3 构建产物部署

```
# 开发阶段
npm run dev       # Vite dev server，代理到 localhost:8080

# 生产构建
npm run build     # → dist/ 目录

# 部署方式 A：Rust 后端托管静态文件
#   复制 dist/ → book-server/static/
#   Axum: .nest_service("/", ServeDir::new("static"))

# 部署方式 B：Nginx 反代
#   Nginx serve dist/ + proxy /api/* → Rust:8080
```

---

## 十、分阶段实施

### Phase 1（MVP，2 周）

| 功能 | 页面 |
|------|------|
| 登录/注册 | LoginPage, RegisterPage |
| 书架展示 | LibraryPage（缩略图网格 + 分页） |
| PDF 阅读器 | ReaderPage（预览图 + react-pdf 替换） |
| 简单搜索 | SearchPage（WASM 分词） |
| 书详情 | BookDetailPage |
| 文件上传 | LibraryPage 上传按钮 |

### Phase 2（1 周）

| 功能 | 页面 |
|------|------|
| 高亮 + 笔记 | AnnotationLayer + 选中文字弹窗 |
| 阅读进度同步 | useProgression + SSE |
| 书签 | 阅读器内书签按钮 |
| 设置 | SettingsPage（密码修改） |

### Phase 3（1 周）

| 功能 | 页面 |
|------|------|
| EPUB 阅读器 | epubjs 集成 |
| API Key 管理 | SettingsPage |
| 管理面板 | AdminPage（词典） |
| 虚拟滚动 | LibraryPage 大书库性能优化 |

---

## 十一、性能目标

| 指标 | 目标 | 方案 |
|------|------|------|
| 首页加载 | <2s | Vite code-split，`pdf`/`jieba` 懒加载 |
| 翻页响应 | <200ms | 预览图秒开，PDF 后台替换 |
| 书架 1000 本书 | <100ms 渲染 | @tanstack/virtual 虚拟滚动 |
| 搜索响应 | <500ms | WASM 前端分词 + 服务端 tsvector/Tantivy |
| 打包体积 | <500KB（首屏） | shadcn/ui 按需引入，不装大型 UI 库 |

---

## 十二、与现有文档的衔接

| 文档 | 相关内容 | 本文档补充 |
|------|---------|-----------|
| `03-技术架构.md` | React Web 占位 | 完整技术栈 + 项目结构 |
| `05-分词搜索方案.md` | WASM 前端分词 | SearchBar 组件 + segmentQuery |
| `07-API接口定义.md` | 所有 API 端点 | fetch 封装 + useApi hook |
| `15-认证与安全.md` | Session + CSRF | authStore + CSRF token 读取 |
| `16-S3存储方案.md` | 签名 URL + /files 代理 | ReaderPage 两层渲染 + 302 处理 |
| `17-缓存与安全策略.md` | CORS 白名单 | Vite proxy 开发配置 |
