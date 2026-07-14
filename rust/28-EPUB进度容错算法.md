# EPUB 进度容错算法 — 三重定位 + 位置插值

> **定位**：当 EPUB 内容变更（章节增删）后，如何恢复用户的阅读进度。
> **参考基准**：Komga `analyzeAndPersist` — 前后最近位置插值算法。
> **更新日期**：2026-07-14

---

## 一、问题场景

```
用户在 EPUB 第 3 章读了 35%
      │
      ▼
用户重新上传了同一 EPUB 的新版本
（章节增删/重排，总章节数从 20 变为 25）
      │
      ▼
旧的进度数据：{ href: "chapter3.xhtml", progression: 0.35, position: 3 }
新版本中 "chapter3.xhtml" 可能已不存在或位置已变
      │
      ▼
如何恢复用户的阅读位置？
```

---

## 二、三重定位

EPUB 进度存储三个维度的位置信息，任一维度匹配即可恢复：

```rust
// src/models/progression.rs

use serde::{Deserialize, Serialize};

/// EPUB 进度的三重定位
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EpubProgression {
    /// 维度1：资源 href（章节文件名）
    /// 最精确：如果新版本中此文件仍存在，直接恢复
    pub href: String,           // "chapter3.xhtml"

    /// 维度2：资源内进度 0.0 ~ 1.0
    /// 中等精确：即使 href 变了，章节内进度比例通常不变
    pub progression: f64,      // 0.35

    /// 维度3：出版物全局位置索引（从 1 开始）
    /// 最粗略：章节序号，用于 href 和 progression 都失败时的兜底
    pub position: u32,          // 3
}
```

| 维度 | 精确度 | 适用场景 | 失败条件 |
|------|--------|---------|---------|
| `href` | ⭐⭐⭐ | 章节文件名未变 | 文件被重命名/删除 |
| `progression` | ⭐⭐ | 章节内进度比例 | 章节内容大幅重写 |
| `position` | ⭐ | 章节序号 | 章节增删导致序号偏移 |

---

## 三、匹配算法

### 3.1 匹配流程

```
旧进度: { href: "chapter3.xhtml", progression: 0.35, position: 3 }
新版本: 25 个章节，href 列表 = ["ch1.xhtml", "ch2.xhtml", ..., "ch25.xhtml"]

1. href 精确匹配
   ├─ "chapter3.xhtml" 在新列表中？ → ✅ 成功 → 恢复到该章节 35% 处
   └─ 不在 → ↓

2. position 插值匹配
   ├─ 旧 position=3, 旧总数=20, 新总数=25
   ├─ 新 position = round(3 * 25 / 20) = 4
   ├─ 取新列表第 4 个章节
   └─ 恢复到该章节 35% 处（confidence=0.7）

3. progression 模糊匹配
   ├─ 旧 progression=0.35, 新总数=25
   ├─ 新 position = round(0.35 * 25) = 9
   ├─ 取新列表第 9 个章节
   └─ 恢复到该章节 35% 处（confidence=0.5）

4. 全部失败 → 重置到第 1 章
```

### 3.2 Rust 实现

```rust
// src/sync/progression.rs

/// EPUB 进度匹配结果
#[derive(Debug, Clone)]
pub enum ProgressionMatchResult {
    /// href 精确匹配
    ExactMatch {
        href: String,
        progression: f64,
    },
    /// position 插值匹配
    InterpolatedMatch {
        href: String,
        progression: f64,
        confidence: f64,
        match_type: MatchType,
    },
    /// 匹配失败，重置
    Reset,
}

#[derive(Debug, Clone)]
pub enum MatchType {
    Position,
    Progression,
}

/// 在新的 EPUB 资源列表中匹配旧进度
///
/// # 参数
/// - `old`: 旧的进度数据
/// - `new_resources`: 新版本的章节 href 列表（按阅读顺序）
/// - `new_total_positions`: 新版本的总章节数
pub fn match_epub_progression(
    old: &EpubProgression,
    new_resources: &[String],
    new_total_positions: u32,
) -> ProgressionMatchResult {
    if new_resources.is_empty() {
        return ProgressionMatchResult::Reset;
    }

    // 1. href 精确匹配
    if new_resources.contains(&old.href) {
        return ProgressionMatchResult::ExactMatch {
            href: old.href.clone(),
            progression: old.progression,
        };
    }

    // 2. position 插值匹配
    if old.position > 0 {
        let ratio = old.position as f64 / new_total_positions as f64;
        let new_position = (ratio * new_total_positions as f64).round() as u32;
        let new_position = new_position.min(new_total_positions).max(1);
        let idx = (new_position - 1) as usize;
        if idx < new_resources.len() {
            return ProgressionMatchResult::InterpolatedMatch {
                href: new_resources[idx].clone(),
                progression: old.progression,
                confidence: 0.7,
                match_type: MatchType::Position,
            };
        }
    }

    // 3. progression 模糊匹配
    let target_position = (old.progression * new_total_positions as f64).round() as u32;
    let target_position = target_position.min(new_total_positions).max(1);
    let idx = (target_position - 1) as usize;
    if idx < new_resources.len() {
        return ProgressionMatchResult::InterpolatedMatch {
            href: new_resources[idx].clone(),
            progression: old.progression,
            confidence: 0.5,
            match_type: MatchType::Progression,
        };
    }

    // 4. 兜底重置
    ProgressionMatchResult::Reset
}
```

---

## 四、单元测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_exact_href_match() {
        let old = EpubProgression {
            href: "chapter3.xhtml".into(),
            progression: 0.35,
            position: 3,
        };
        let new_resources: Vec<String> = (1..=25)
            .map(|i| format!("chapter{}.xhtml", i))
            .collect();

        let result = match_epub_progression(&old, &new_resources, 25);
        match result {
            ProgressionMatchResult::ExactMatch { href, progression } => {
                assert_eq!(href, "chapter3.xhtml");
                assert!((progression - 0.35).abs() < 0.001);
            }
            _ => panic!("Expected ExactMatch"),
        }
    }

    #[test]
    fn test_position_interpolation() {
        let old = EpubProgression {
            href: "old_chapter3.xhtml".into(), // 不在新列表中
            progression: 0.35,
            position: 3,
        };
        let new_resources: Vec<String> = (1..=25)
            .map(|i| format!("chapter{}.xhtml", i))
            .collect();

        let result = match_epub_progression(&old, &new_resources, 25);
        match result {
            ProgressionMatchResult::InterpolatedMatch {
                href, confidence, match_type, ..
            } => {
                assert_eq!(href, "chapter4.xhtml"); // round(3 * 25 / 20) = 4
                assert!((confidence - 0.7).abs() < 0.001);
                assert!(matches!(match_type, MatchType::Position));
            }
            _ => panic!("Expected InterpolatedMatch"),
        }
    }

    #[test]
    fn test_progression_fallback() {
        let old = EpubProgression {
            href: "missing.xhtml".into(),
            progression: 0.35,
            position: 0, // position 无效
        };
        let new_resources: Vec<String> = (1..=25)
            .map(|i| format!("chapter{}.xhtml", i))
            .collect();

        let result = match_epub_progression(&old, &new_resources, 25);
        match result {
            ProgressionMatchResult::InterpolatedMatch {
                href, confidence, match_type, ..
            } => {
                assert_eq!(href, "chapter9.xhtml"); // round(0.35 * 25) = 9
                assert!((confidence - 0.5).abs() < 0.001);
                assert!(matches!(match_type, MatchType::Progression));
            }
            _ => panic!("Expected InterpolatedMatch"),
        }
    }

    #[test]
    fn test_reset_when_all_fail() {
        let old = EpubProgression {
            href: "missing.xhtml".into(),
            progression: -1.0, // 无效
            position: 0,
        };
        let new_resources = vec!["ch1.xhtml".to_string()];

        let result = match_epub_progression(&old, &new_resources, 1);
        assert!(matches!(result, ProgressionMatchResult::Reset));
    }
}
```

---

## 五、与进度同步 API 的集成

```rust
// src/service/sync_service.rs

/// 文档重新导入后，尝试迁移旧进度
pub async fn migrate_progression(
    pool: &PgPool,
    doc_id: i32,
    user_id: i32,
    old_progression: &EpubProgression,
    new_resources: &[String],
) -> Result<ProgressionMatchResult> {
    let new_total = new_resources.len() as u32;
    let result = match_epub_progression(old_progression, new_resources, new_total);

    match &result {
        ProgressionMatchResult::ExactMatch { href, progression } => {
            // 更新进度表
            update_progression(pool, doc_id, user_id, href, *progression).await?;
        }
        ProgressionMatchResult::InterpolatedMatch { href, progression, confidence, .. } => {
            // 更新进度表 + 标记置信度
            update_progression(pool, doc_id, user_id, href, *progression).await?;
            // 在 metadata 中记录置信度，前端可提示"位置可能已变化"
            set_progression_metadata(pool, doc_id, user_id, "match_confidence", confidence).await?;
        }
        ProgressionMatchResult::Reset => {
            // 重置到第 1 章
            if let Some(first) = new_resources.first() {
                update_progression(pool, doc_id, user_id, first, 0.0).await?;
            }
        }
    }

    Ok(result)
}
```

---

## 六、前端提示

当进度匹配使用了插值（`InterpolatedMatch`），前端应显示提示：

```
┌──────────────────────────────────────────────┐
│ ℹ️ 位置已自动调整                              │
│                                              │
│ 此文档的新版本与旧版本有差异，                   │
│ 您的阅读位置已从"第 3 章"调整到"第 4 章"。       │
│ 如果位置不对，请手动翻到正确页面。               │
│                                              │
│ [知道了]                                      │
└──────────────────────────────────────────────┘
```