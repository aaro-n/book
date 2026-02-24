# API接口定义 - RESTful API设计

## 🎯 设计原则

- **风格**：RESTful API
- **格式**：JSON
- **认证**：Bearer Token (JWT)
- **版本**：/api/v1
- **状态码**：使用标准HTTP状态码 (200, 201, 400, 401, 403, 404, 500)

---

## 🔐 认证模块 (Auth)

### 注册
- **POST** `/api/v1/auth/register`
- **Body**: `{ "email": "user@example.com", "password": "strongPassword123" }`
- **Response**: `201 Created` - `{ "token": "jwt_token...", "user": { "id": 1, ... } }`

### 登录
- **POST** `/api/v1/auth/login`
- **Body**: `{ "email": "user@example.com", "password": "params" }`
- **Response**: `200 OK` - `{ "token": "jwt_token...", "user": { "id": 1, ... } }`

### 获取当前用户
- **GET** `/api/v1/auth/me`
- **Headers**: `Authorization: Bearer <token>`
- **Response**: `200 OK` - `{ "id": 1, "email": "...", "display_name": "..." }`

---

## 📄 文档模块 (Documents)

### 上传文档
- **POST** `/api/v1/documents`
- **Content-Type**: `multipart/form-data`
- **Body**: `file=@book.pdf`
- **Response**: `201 Created` - `{ "id": 101, "title": "book.pdf", "status": "processing" }`
- **说明**: 触发后台解析任务（S3上传 -> 每页文字提取 -> 分词 -> 索引）

### 获取文档列表
- **GET** `/api/v1/documents`
- **Query**: `page=1&limit=20&sort=created_at&order=desc`
- **Response**: `200 OK` - `{ "items": [ ... ], "total": 50 }`

### 获取文档详情
- **GET** `/api/v1/documents/:id`
- **Response**: `200 OK` - `{ "id": 101, "title": "...", "page_count": 200, "s3_key": "..." }`

### 获取文档内容（用于阅读器）
- **GET** `/api/v1/documents/:id/content`
- **Response**: `200 OK` - `{ "s3_url": "presigned_url_to_pdf" }`
- **说明**: 返回S3预签名URL，前端直接请求S3或通过Nginx代理

### 删除文档
- **DELETE** `/api/v1/documents/:id`
- **Response**: `204 No Content`

---

## 🖍️ 标记模块 (Highlights & Annotations)

### 创建高亮/笔记
- **POST** `/api/v1/documents/:id/highlights`
- **Body**: 
  ```json
  {
    "page_num": 5,
    "color": "#ff0000",
    "text": "Selected text",
    "rects": [ { "x": 10, "y": 20, "w": 100, "h": 10 } ],
    "note": "My thoughts..."
  }
  ```
- **Response**: `201 Created` - `{ "id": 501, ... }`

### 获取文档的所有标记
- **GET** `/api/v1/documents/:id/highlights`
- **Response**: `200 OK` - `[ { "id": 501, "page_num": 5, "note": "...", ... } ]`

### 更新笔记
- **PUT** `/api/v1/highlights/:id`
- **Body**: `{ "note": "Updated thoughts...", "color": "#00ff00" }`
- **Response**: `200 OK`

### 删除标记
- **DELETE** `/api/v1/highlights/:id`
- **Response**: `204 No Content`

---

## 🔍 搜索模块 (Search)

### 全文搜索
- **GET** `/api/v1/search`
- **Query**: 
  - `q=人工智能` (关键词)
  - `type=document` (范围: document | highlight)
  - `doc_id=101` (可选，限制在某书内搜索)
- **Response**: `200 OK`
  ```json
  {
    "hits": [
      {
        "doc_id": 101,
        "page_num": 12,
        "text_highlight": "未来...<em>人工智能</em>...的发展",
        "score": 0.98
      }
    ],
    "total": 15,
    "took_ms": 10
  }
  ```

---

## ⚙️ 系统状态 (System)

### 健康检查
- **GET** `/health`
- **Response**: `200 OK` - `{ "status": "ok", "db": "ok", "redis": "ok" }`


---

## 📖 词典管理模块 (Dictionary)

### 获取所有自定义词汇
- **GET** `/api/v1/dictionaries`
- **Response**: `200 OK`
  ```json
  [
    { "id": 1, "term": "Transformer", "frequency": 100, "is_enable": true },
    { "id": 2, "term": "深度学习", "frequency": 50, "is_enable": true }
  ]
  ```

### 添加新词汇
- **POST** `/api/v1/dictionaries`
- **Body**: `{ "term": "Copilot", "frequency": 100 }`
- **Response**: `201 Created` - `{ "id": 3, "term": "Copilot" }`
- **说明**: 系统会自动发布变更事件，所有服务实例即时加载新词。

### 删除词汇
- **DELETE** `/api/v1/dictionaries/:id`
- **Response**: `204 No Content`
