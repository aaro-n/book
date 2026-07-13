# OPDS 协议完整参考 — Schema 定义 + Go 实现指南

> 本文档提供 OPDS 协议在本项目中使用的完整数据模型、XML/JSON Schema 以及 Go 实现代码。

---

## 一、OPDS 数据模型定义

### 1.1 Go 结构体

```go
// internal/models/opds.go

package models

import "encoding/xml"

// ===== OPDS 1.2 =====

// Feed 代表一个 OPDS 目录（Navigation 或 Acquisition）
type OPDSFeed struct {
    XMLName xml.Name     `xml:"http://www.w3.org/2005/Atom feed"`
    ID      string       `xml:"id"`
    Title   string       `xml:"title"`
    Updated string       `xml:"updated"`
    Author  AtomAuthor   `xml:"author"`
    Links   []AtomLink   `xml:"link"`
    Entries []OPDSEntry  `xml:"entry"`
}

// Entry 代表目录中的一个条目（导航节点或书目）
type OPDSEntry struct {
    Title   string       `xml:"title"`
    ID      string       `xml:"id"`
    Summary string       `xml:"summary,omitempty"`
    Content AtomContent  `xml:"content,omitempty"`
    Links   []AtomLink   `xml:"link"`
    Updated string       `xml:"updated,omitempty"`
    // 扩展：Dublin Core 元数据
    Publisher  string   `xml:"http://purl.org/dc/terms/ publisher,omitempty"`
    Issued     string   `xml:"http://purl.org/dc/terms/ issued,omitempty"`
    Language   string   `xml:"http://purl.org/dc/terms/ language,omitempty"`
    // 扩展：作者
    Authors    []AtomAuthor `xml:"author"`
}

type AtomAuthor struct {
    Name string `xml:"name"`
}

type AtomContent struct {
    Type string `xml:"type,attr"`
    Text string `xml:",chardata"`
}

// AtomLink OPDS 链接（核心概念）
type AtomLink struct {
    Href string `xml:"href,attr"`
    Rel  string `xml:"rel,attr"`
    Type string `xml:"type,attr"`
    // OPDS 扩展属性
    Title   string `xml:"http://opds-spec.org/2010/catalog title,attr,omitempty"`
    FacetGroup string `xml:"http://opds-spec.org/2010/catalog facetGroup,attr,omitempty"`
}

// ===== OPDS 2.0 (JSON-LD) =====

type OPDS2Catalog struct {
    Metadata     OPDS2Metadata    `json:"metadata"`
    Links        []OPDS2Link      `json:"links"`
    Publications []OPDS2Publication `json:"publications"`
    // 搜索用
    SearchInfo   *SearchInfo      `json:"searchInfo,omitempty"`
}

type OPDS2Metadata struct {
    Title          string `json:"title"`
    NumberOfItems  int    `json:"numberOfItems,omitempty"`
    ItemsPerPage   int    `json:"itemsPerPage,omitempty"`
    CurrentPage    int    `json:"currentPage,omitempty"`
    Modified       string `json:"modified,omitempty"`
}

type OPDS2Publication struct {
    Metadata PublicationMetadata `json:"metadata"`
    Links    []OPDS2Link         `json:"links"`
    Images   []OPDS2Image        `json:"images,omitempty"`
}

type PublicationMetadata struct {
    Type        string        `json:"@type"`
    Identifier  string        `json:"identifier"`
    Title       string        `json:"title"`
    Author      []NameOnly    `json:"author,omitempty"`
    Publisher   string        `json:"publisher,omitempty"`
    Language    string        `json:"language,omitempty"`
    Published   string        `json:"published,omitempty"`
    Description string        `json:"description,omitempty"`
    NumberOfPages int         `json:"numberOfPages,omitempty"`
    Modified    string        `json:"modified,omitempty"`
}

type NameOnly struct {
    Name string `json:"name"`
}

type OPDS2Link struct {
    Rel      string `json:"rel"`
    Href     string `json:"href"`
    Type     string `json:"type"`
    Title    string `json:"title,omitempty"`
    Templated bool  `json:"templated,omitempty"`
}

type OPDS2Image struct {
    Href string `json:"href"`
    Type string `json:"type"`
    Rel  string `json:"rel,omitempty"`
}

// 搜索扩展（OPDS 2.0 不原生支持，本项目扩展）
type SearchInfo struct {
    Score float64    `json:"score,omitempty"`
    Hits  []SearchHit `json:"hits,omitempty"`
}

type SearchHit struct {
    PageNum int    `json:"pageNum"`
    Snippet string `json:"snippet"`
}

// ===== OpenSearch =====

type OpenSearchDescription struct {
    XMLName     xml.Name             `xml:"http://a9.com/-/spec/opensearch/1.1/ OpenSearchDescription"`
    ShortName   string               `xml:"ShortName"`
    Description string               `xml:"Description"`
    URLs        []OpenSearchURL      `xml:"Url"`
    Languages   []string             `xml:"Language"`
}

type OpenSearchURL struct {
    Type     string `xml:"type,attr"`
    Template string `xml:"template,attr"`
}

// ===== Sync =====

type ReadingProgress struct {
    DocumentID int     `json:"document_id"`
    PageNum    int     `json:"page_num"`
    TotalPages int     `json:"total_pages,omitempty"`
    Percentage float64 `json:"percentage"`
    LastReadAt string  `json:"last_read_at,omitempty"`
    Device     string  `json:"device,omitempty"`
}

type SyncBookmarks struct {
    DocumentID int       `json:"document_id"`
    Bookmarks  []BookmarkItem `json:"bookmarks"`
}

type BookmarkItem struct {
    ID        int    `json:"id,omitempty"`
    PageNum   int    `json:"page_num"`
    Title     string `json:"title"`
    Note      string `json:"note,omitempty"`
    CreatedAt string `json:"created_at,omitempty"`
}
```

### 1.2 OPDS 链接关系（rel）速查表

| rel 值 | 含义 | 使用场景 |
|--------|------|---------|
| `self` | 当前文档自身 | 每个 Feed 必须 |
| `start` | 起始目录 | 根导航 Feed |
| `subsection` | 子目录 | 导航到子分类 |
| `http://opds-spec.org/acquisition` | 获取文件 | 下载 PDF |
| `http://opds-spec.org/image/thumbnail` | 缩略图 | 封面图片 |
| `http://opds-spec.org/progression` | 阅读进度 | OPDS Progression 草案 |
| `http://www.w3.org/ns/oa#annotationService` | 标记服务 | W3C Web Annotation |
| `search` | 搜索 | OpenSearch 描述文档 |
| `next` / `prev` / `first` / `last` | 分页 | Acquisition Feed |

> **注意**：`sync/progress`、`sync/bookmark` 等同步 rel **不属于 OPDS 标准**，已被替换为 `http://opds-spec.org/progression` 和 `http://www.w3.org/ns/oa#annotationService`。

---

## 二、Go Service 层实现

### 2.1 OPDS Service

```go
// internal/service/opds_service.go

package service

import (
    "fmt"
    "time"
    "myapp/internal/models"
)

type OPDSService struct {
    docRepo   DocumentRepository
    syncRepo  SyncRepository
}

// BuildNavigationFeed 构建 OPDS 1.2 导航 Feed
func (s *OPDSService) BuildNavigationFeed(baseURL string) *models.OPDSFeed {
    now := time.Now().UTC().Format(time.RFC3339)

    return &models.OPDSFeed{
        ID:      "urn:uuid:pdf-manager-root",
        Title:   "我的 PDF 图书馆",
        Updated: now,
        Author: models.AtomAuthor{Name: "PDF Manager"},
        Links: []models.AtomLink{
            {Href: baseURL + "/opds/catalog", Rel: "self",
                Type: "application/atom+xml;profile=opds-catalog;kind=navigation"},
            {Href: baseURL + "/opds/catalog", Rel: "start",
                Type: "application/atom+xml;profile=opds-catalog;kind=navigation"},
            {Href: baseURL + "/opds/search", Rel: "search",
                Type: "application/opensearchdescription+xml"},
        },
        Entries: []models.OPDSEntry{
            {
                Title:   "最近添加",
                ID:      "urn:uuid:recent",
                Content: models.AtomContent{Type: "text", Text: "最近 30 天上传的文档"},
                Links: []models.AtomLink{
                    {Href: baseURL + "/opds/catalog/recent", Rel: "subsection",
                        Type: "application/atom+xml;profile=opds-catalog;kind=acquisition"},
                },
            },
            {
                Title: "所有文档",
                ID:    "urn:uuid:all",
                Links: []models.AtomLink{
                    {Href: baseURL + "/opds/catalog/all", Rel: "subsection",
                        Type: "application/atom+xml;profile=opds-catalog;kind=acquisition"},
                },
            },
        },
    }
}

// BuildAcquisitionFeed 构建书目列表 Feed
func (s *OPDSService) BuildAcquisitionFeed(
    baseURL string,
    documents []Document,
    page, limit, total int,
) *models.OPDSFeed {
    now := time.Now().UTC().Format(time.RFC3339)

    feed := &models.OPDSFeed{
        ID:      "urn:uuid:all-documents",
        Title:   "所有文档",
        Updated: now,
        Links: []models.AtomLink{
            {Href: fmt.Sprintf("%s/opds/catalog/all?page=%d", baseURL, page), Rel: "self",
                Type: "application/atom+xml"},
        },
    }

    // 分页链接
    if page*limit < total {
        feed.Links = append(feed.Links, models.AtomLink{
            Href: fmt.Sprintf("%s/opds/catalog/all?page=%d", baseURL, page+1),
            Rel:  "next", Type: "application/atom+xml",
        })
    }

    // 书目条目
    for _, doc := range documents {
        entry := models.OPDSEntry{
            Title:     doc.Title,
            ID:        fmt.Sprintf("urn:uuid:%s", doc.UUID),
            Summary:   doc.Description,
            Updated:   doc.UpdatedAt.Format(time.RFC3339),
            Language:  doc.Language,
            Publisher: doc.Publisher,
            Issued:    doc.IssuedDate,
            Authors:   s.buildAuthors(doc.Authors),
            Links: []models.AtomLink{
                {Href: fmt.Sprintf("%s/api/v1/documents/%d/cover", baseURL, doc.ID),
                    Rel: "http://opds-spec.org/image/thumbnail", Type: "image/jpeg"},
                {Href: fmt.Sprintf("%s/opds/acquisition/%d", baseURL, doc.ID),
                    Rel: "http://opds-spec.org/acquisition", Type: "application/pdf"},
                {Href: fmt.Sprintf("%s/api/v1/progression/%d", baseURL, doc.ID),
                    Rel: "http://opds-spec.org/progression", Type: "application/opds-progression+json"},
                {Href: fmt.Sprintf("%s/api/v1/annotations?document_id=%d", baseURL, doc.ID),
                    Rel: "http://www.w3.org/ns/oa#annotationService", Type: "application/ld+json"},
            },
        }
        feed.Entries = append(feed.Entries, entry)
    }

    return feed
}

func (s *OPDSService) buildAuthors(authors []string) []models.AtomAuthor {
    result := make([]models.AtomAuthor, len(authors))
    for i, a := range authors {
        result[i] = models.AtomAuthor{Name: a}
    }
    return result
}

// BuildCatalogV2 构建 OPDS 2.0 JSON-LD
func (s *OPDSService) BuildCatalogV2(
    baseURL string,
    documents []Document,
) *models.OPDS2Catalog {
    now := time.Now().UTC().Format(time.RFC3339)

    catalog := &models.OPDS2Catalog{
        Metadata: models.OPDS2Metadata{
            Title:         "我的 PDF 图书馆",
            NumberOfItems: len(documents),
            Modified:      now,
        },
        Links: []models.OPDS2Link{
            {Rel: "self", Href: baseURL + "/opds/v2/catalog", Type: "application/opds+json"},
            {Rel: "search", Href: baseURL + "/opds/search?q={query}&page={page}",
                Type: "application/opds+json", Templated: true},
        },
        Publications: make([]models.OPDS2Publication, len(documents)),
    }

    for i, doc := range documents {
        catalog.Publications[i] = models.OPDS2Publication{
            Metadata: models.PublicationMetadata{
                Type:         "http://schema.org/Book",
                Identifier:   fmt.Sprintf("urn:uuid:%s", doc.UUID),
                Title:        doc.Title,
                Author:       s.buildNames(doc.Authors),
                Publisher:    doc.Publisher,
                Language:     doc.Language,
                Published:    doc.IssuedDate,
                Description:  doc.Description,
                NumberOfPages: doc.PageCount,
                Modified:     doc.UpdatedAt.Format(time.RFC3339),
            },
            Links: []models.OPDS2Link{
                {Rel: "http://opds-spec.org/acquisition",
                    Href: fmt.Sprintf("%s/opds/acquisition/%d", baseURL, doc.ID),
                    Type: "application/pdf"},
            },
            Images: []models.OPDS2Image{
                {Href: fmt.Sprintf("%s/api/v1/documents/%d/cover", baseURL, doc.ID),
                    Type: "image/jpeg", Rel: "http://opds-spec.org/image/thumbnail"},
            },
        }
    }

    return catalog
}

func (s *OPDSService) buildNames(authors []string) []models.NameOnly {
    result := make([]models.NameOnly, len(authors))
    for i, a := range authors {
        result[i] = models.NameOnly{Name: a}
    }
    return result
}
```

---

## 三、Handler 层实现

```go
// internal/handler/opds_handler.go

package handler

import (
    "net/http"
    "strconv"
    "myapp/internal/models"
    "github.com/gin-gonic/gin"
)

type OPDSHandler struct {
    opdsService *service.OPDSService
    docService  *service.DocumentService
    syncService *service.SyncService
}

// NavigationFeed OPDS 1.2 根导航
func (h *OPDSHandler) NavigationFeed(c *gin.Context) {
    feed := h.opdsService.BuildNavigationFeed(c.Request.Host)
    c.XML(http.StatusOK, feed)
}

// AcquisitionFeed OPDS 1.2 书目列表
func (h *OPDSHandler) AcquisitionFeed(c *gin.Context) {
    userID := c.GetInt("user_id")
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    limit, _ := strconv.Atoi(c.DefaultQuery("limit", "20"))

    docs, total, err := h.docService.ListDocuments(c.Request.Context(), userID, page, limit)
    if err != nil {
        c.XML(http.StatusInternalServerError, errorFeed(err.Error()))
        return
    }

    feed := h.opdsService.BuildAcquisitionFeed(
        "http://"+c.Request.Host, docs, page, limit, total,
    )
    c.XML(http.StatusOK, feed)
}

// CatalogV2 OPDS 2.0 JSON-LD
func (h *OPDSHandler) CatalogV2(c *gin.Context) {
    userID := c.GetInt("user_id")
    docs, _, err := h.docService.ListDocuments(c.Request.Context(), userID, 1, 100)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    catalog := h.opdsService.BuildCatalogV2("http://"+c.Request.Host, docs)
    c.JSON(http.StatusOK, catalog)
}

// SearchFeed OPDS 1.2 搜索
func (h *OPDSHandler) SearchFeed(c *gin.Context) {
    userID := c.GetInt("user_id")
    query := c.Query("q")
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    limit, _ := strconv.Atoi(c.DefaultQuery("limit", "20"))

    results, total, err := h.docService.SearchContent(
        c.Request.Context(), userID, query, page, limit,
    )
    if err != nil {
        c.XML(http.StatusInternalServerError, errorFeed(err.Error()))
        return
    }

    feed := h.opdsService.BuildSearchFeed(
        "http://"+c.Request.Host, query, results, page, limit, total,
    )
    c.XML(http.StatusOK, feed)
}

// Download PDF 获取
func (h *OPDSHandler) Download(c *gin.Context) {
    userID := c.GetInt("user_id")
    docID, _ := strconv.Atoi(c.Param("id"))

    // 权限校验
    doc, err := h.docService.GetDocument(c.Request.Context(), userID, docID)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "文档不存在"})
        return
    }

    // 生成 S3 预签名 URL，302 重定向
    presignedURL, _ := h.docService.GetPresignedURL(doc.S3Key)
    if c.Query("format") == "url" {
        c.JSON(http.StatusOK, gin.H{"url": presignedURL})
        return
    }

    c.Redirect(http.StatusFound, presignedURL)
}

// GetProgress 阅读进度
func (h *OPDSHandler) GetProgress(c *gin.Context) {
    userID := c.GetInt("user_id")
    docID, _ := strconv.Atoi(c.Param("id"))

    progress, err := h.syncService.GetProgress(c.Request.Context(), userID, docID)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "无进度记录"})
        return
    }
    c.JSON(http.StatusOK, progress)
}

// UpdateProgress 更新进度
func (h *OPDSHandler) UpdateProgress(c *gin.Context) {
    userID := c.GetInt("user_id")
    docID, _ := strconv.Atoi(c.Param("id"))

    var progress models.ReadingProgress
    if err := c.ShouldBindJSON(&progress); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    progress.DocumentID = docID
    if err := h.syncService.UpdateProgress(c.Request.Context(), userID, &progress); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"status": "synced"})
}
```

---

## 四、内容协商实现

```go
// internal/middleware/opds.go

package middleware

import (
    "net/http"
    "strings"
    "github.com/gin-gonic/gin"
)

// OPDSContentNegotiation 自动选择 OPDS 1.2 或 2.0 格式
// 用法：GET /opds/catalog/all 默认返回 Atom XML
//       Accept: application/opds+json 返回 JSON-LD
func OPDSContentNegotiation() gin.HandlerFunc {
    return func(c *gin.Context) {
        accept := c.GetHeader("Accept")

        if strings.Contains(accept, "application/opds+json") {
            c.Set("opds_version", "2.0")
        } else {
            c.Set("opds_version", "1.2") // 默认
        }
        c.Next()
    }
}

// Routes 注册所有 OPDS 路由
func SetupOPDSRoutes(r *gin.Engine, h *handler.OPDSHandler) {
    opds := r.Group("/opds")
    opds.Use(OPDSContentNegotiation())

    // 导航 + 目录
    opds.GET("/catalog", h.NavigationFeed)
    opds.GET("/catalog/*path", h.AcquisitionFeed)
    opds.GET("/catalog/search", h.SearchFeed)

    // OPDS 2.0
    opds.GET("/v2/catalog", h.CatalogV2)
    opds.GET("/v2/catalog/search", h.SearchV2)

    // OpenSearch 描述
    opds.GET("/search", h.SearchDescriptor)

    // 获取
    opds.GET("/acquisition/:id", h.Download)

    // ===== 进度同步（OPDS Progression 草案）=====
    progression := r.Group("/api/v1/progression")
    progression.GET("/:id", h.GetProgress)
    progression.PUT("/:id", h.UpdateProgress)
    progression.GET("", h.ListAll)

    // ===== 标记（W3C Web Annotation）=====
    annotations := r.Group("/api/v1/annotations")
    annotations.POST("", h.CreateAnnotation)
    annotations.GET("", h.ListAnnotations)
    annotations.GET("/:id", h.GetAnnotation)
    annotations.PUT("/:id", h.UpdateAnnotation)
    annotations.DELETE("/:id", h.DeleteAnnotation)
}
```

---

## 五、数据库补充字段

为支持 OPDS 元数据，`documents` 表需补充以下字段：

```sql
-- 迁移脚本: 004_opds_metadata.sql
ALTER TABLE documents ADD COLUMN IF NOT EXISTS uuid UUID DEFAULT gen_random_uuid();
ALTER TABLE documents ADD COLUMN IF NOT EXISTS publisher VARCHAR(255);
ALTER TABLE documents ADD COLUMN IF NOT EXISTS issued_date VARCHAR(50);
ALTER TABLE documents ADD COLUMN IF NOT EXISTS authors TEXT[];       -- PostgreSQL 数组
ALTER TABLE documents ADD COLUMN IF NOT EXISTS cover_s3_key VARCHAR(500);  -- 封面图存储
ALTER TABLE documents ADD COLUMN IF NOT EXISTS tags TEXT[];

-- 阅读进度同步表
CREATE TABLE IF NOT EXISTS reading_progress (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    document_id INTEGER NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    page_num INTEGER NOT NULL DEFAULT 1,
    percentage REAL NOT NULL DEFAULT 0,
    device VARCHAR(200),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, document_id)
);

CREATE INDEX idx_progress_user ON reading_progress(user_id);
CREATE INDEX idx_progress_doc ON reading_progress(document_id);
```

---

---

## 三-B、Go Progression Handler（OPDS Progression 草案）

```go
// internal/handler/progression_handler.go

package handler

import (
    "net/http"
    "strconv"
    "time"
    "github.com/gin-gonic/gin"
)

type ProgressionDoc struct {
    Title       string           `json:"title,omitempty"`
    Modified    string           `json:"modified"`
    Device      ProgressionDevice `json:"device"`
    Progression float64          `json:"progression"`
    References  []string         `json:"references"`
}

type ProgressionDevice struct {
    ID   string `json:"id"`    // "urn:uuid:..."
    Name string `json:"name"`  // "PDF Manager Go"
}

// Get 获取单个文档进度
func (h *ProgressionHandler) Get(c *gin.Context) {
    docID, _ := strconv.Atoi(c.Param("id"))
    prog, err := h.service.GetProgression(c.Request.Context(), docID)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "无进度记录"})
        return
    }
    c.JSON(http.StatusOK, prog)
}

// Update 更新进度
func (h *ProgressionHandler) Update(c *gin.Context) {
    docID, _ := strconv.Atoi(c.Param("id"))
    var doc ProgressionDoc
    if err := c.ShouldBindJSON(&doc); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    doc.Modified = time.Now().UTC().Format(time.RFC3339)
    if err := h.service.UpdateProgression(c.Request.Context(), docID, &doc); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"status": "synced", "modified": doc.Modified})
}

// ListAll 增量同步：获取自某时间点以来的所有进度变更
func (h *ProgressionHandler) ListAll(c *gin.Context) {
    since := c.DefaultQuery("since", "1970-01-01T00:00:00Z")
    progressions, err := h.service.ListSince(c.Request.Context(), since)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, progressions)
}
```

---

## 六、验证清单

上线前逐项检查：

- [ ] `GET /opds/catalog` 返回合法 Atom XML（KOReader 可解析）
- [ ] `GET /opds/search` 返回合法 OpenSearchDescription XML
- [ ] `GET /opds/catalog/search?q=人工智能` 中文搜索正确
- [ ] `GET /opds/acquisition/{id}` 返回 PDF 或 302 重定向
- [ ] `GET /opds/v2/catalog` 返回合法 JSON-LD
- [ ] 自定义同步 API `PUT /api/v1/progression/{id}` 进度持久化
- [ ] 增量同步 `GET /api/v1/progression?since=...` 返回变更列表
- [ ] KOReader 中添加 OPDS 地址后可浏览/搜索/下载
