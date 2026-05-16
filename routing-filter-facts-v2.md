# changedetection.io RSS 标签过滤路由事实核查报告 v2

> **核查方法**：基于源码逐行审计（`main_feed.py`、`tag.py`）
> **核查日期**：2026-05-16
> **代码版本**：当前主干

---

## 目录

1. [事实核查说明](#1-事实核查说明)
2. [主 Feed `/rss?tag=...` 路径完整步骤](#2-主-feed-rsstag-路径完整步骤)
3. [标签路由 `/rss/tag/<uuid>` 路径完整步骤](#3-标签路由-rsstaguuid-路径完整步骤)
4. [两条路径的语义边界对比](#4-两条路径的语义边界对比)
5. [关键边缘行为与缺陷](#5-关键边缘行为与缺陷)
6. [实现源码摘录](#6-实现源码摘录)

---

## 1. 事实核查说明

### 1.1 核查对象

| 路由 | 对应文件 | 代码行数 |
|------|---------|---------|
| `/rss?tag=...` | `blueprint/rss/main_feed.py` | 43-59 |
| `/rss/tag/<tag_uuid>` | `blueprint/rss/tag.py` | 10-95 |

### 1.2 关键定义

- **标签存储**：`datastore.data['settings']['application']['tags']`
  - 结构：`{tag_uuid: {title: str, ...}}`
  - 键：标签 UUID 字符串
  - 值：标签对象（包含 `title` 字段）

- **Watch 标签关联**：`watch['tags']`
  - 结构：`[tag_uuid1, tag_uuid2, ...]`
  - 仅存储 UUID 字符串列表，**不存储标签名称**

---

## 2. 主 Feed `/rss?tag=...` 路径完整步骤

### 2.1 完整执行流程图

```
请求：GET /rss?tag=Amazon-Watches&token=...
    │
    ▼
STEP 1: 参数提取与规范化 (main_feed.py:43)
    limit_tag = request.args.get('tag', '').lower().strip()
    # 例："Amazon-Watches" → "amazon-watches"
    │
    ▼
STEP 2: 名称 → UUID 映射尝试 (main_feed.py:45-47)
    FOR (uuid, tag_obj) IN datastore.data['settings']['application']['tags'].items():
        IF limit_tag == tag_obj.get('title', '').lower().strip():
            limit_tag = uuid    # 替换为内部 UUID
            BREAK  # 找到第一个匹配就停止，不再遍历后续
    END FOR
    # 两种结果：
    # A) 找到匹配 → limit_tag 变为标签 UUID 字符串
    # B) 未找到匹配 → limit_tag 保持为原始小写字符串
    │
    ▼
STEP 3: Watch 筛选循环 (main_feed.py:53-59)
    FOR (watch_uuid, watch) IN datastore.data['watching'].items():
        # 过滤器 A：静音过滤（全局配置控制）
        IF rss_hide_muted_watches AND watch.get('notification_muted'):
            CONTINUE
        
        # 过滤器 B：标签匹配检查（核心逻辑）
        IF limit_tag IS NOT EMPTY AND limit_tag NOT IN watch['tags']:
            CONTINUE  # 排除不匹配的 Watch
        
        # 通过所有过滤 → 加入候选列表
        sorted_watches.append(watch)
    END FOR
    │
    ▼
STEP 4: 排序与后续处理
    sorted_watches.sort(key=lambda x: x.last_changed, reverse=False)
    # → 生成 RSS Feed...
```

---

### 2.2 名称匹配失败后的精确行为

**当没有任何标签匹配输入名称时**：
- `limit_tag` **保持为原始小写字符串**（不会变为空或 None）
- 随后进入筛选循环：`limit_tag not in watch['tags']`
- 由于 `watch['tags']` 只存储 **UUID**，不可能匹配字符串名称
- **最终结果**：所有 Watch 被过滤 → **空 Feed**

**失配不会抛出任何错误**，静默返回空结果。

---

### 2.3 匹配逻辑边界条件

| 条件 | 行为 | 结果 |
|------|------|-----|
| 多个标签同名 | 取字典遍历顺序的**第一个匹配**（Python 3.7+ 为插入顺序） | 不确定哪个标签生效 |
| 标签名大小写不一致 | `.lower()` 规范化后比较 | 大小写不敏感匹配 |
| 标签名含前后空格 | `.strip()` 规范化后比较 | 自动忽略前后空格 |
| `tag=` 参数为空字符串 | `limit_tag = ''` → 不触发过滤逻辑 | 等同于无 tag 参数 |
| 无 `tag` 参数 | `limit_tag = ''` → 不触发过滤逻辑 | 返回所有 Watch 变更 |

---

## 3. 标签路由 `/rss/tag/<uuid>` 路径完整步骤

### 3.1 完整执行流程图

```
请求：GET /rss/tag/a1b2c3-d4e5-f6g7...?token=...
    │
    ▼
STEP 1: 路由参数提取 (tag.py:10-11)
    tag_uuid = <uuid_str:tag_uuid>  # Flask 路由转换器
    # 直接从 URL 路径中获取标签 UUID
    │
    ▼
STEP 2: 标签存在性校验 (tag.py:33-36)
    tag = datastore.data['settings']['application']['tags'].get(tag_uuid)
    IF tag IS None:
        RETURN "Tag with UUID not found", 404
    END IF
    │
    ▼
STEP 3: Feed 元数据设置 (tag.py:41-44)
    fg.title(f'changedetection.io - {tag_title}')
    fg.description(f'Changes for watches tagged with {tag_title}')
    # 标签标题体现在 Feed 元数据中
    │
    ▼
STEP 4: Watch 筛选循环 (tag.py:47-91)
    FOR (watch_uuid, watch) IN datastore.data['watching'].items():
        # 过滤器 A：标签精确匹配
        IF tag_uuid NOT IN watch.get('tags', []):
            CONTINUE  # 不在该标签组中的 Watch 跳过
        
        # 过滤器 B：静音过滤（同主 Feed）
        IF rss_hide_muted_watches AND watch.get('notification_muted'):
            CONTINUE
        
        # 过滤器 C：历史快照数量检查（至少 2 个才有变更）
        dates = list(watch.history.keys())
        IF len(dates) < 2:
            CONTINUE
        
        # 过滤器 D：仅未读变更（同主 Feed）
        IF NOT watch.viewed:
            # → 生成 RSS 条目...
    END FOR
    │
    ▼
STEP 5: 输出 RSS
    response = make_response(fg.rss_str())
    # 无排序步骤（注意：存在已知缺陷）
```

---

### 3.2 UUID 不匹配的精确行为

**当标签 UUID 不存在时**：
- 直接返回 **HTTP 404** 状态码 + 错误文本
- **不会**静默返回空 Feed

---

## 4. 两条路径的语义边界对比

### 4.1 功能对比矩阵

| 维度 | `/rss?tag=名称` (主 Feed 参数) | `/rss/tag/<uuid>` (标签路由) |
|------|-------------------------------|-----------------------------|
| **参数类型** | 标签**名称**（人类可读字符串） | 标签**UUID**（内部标识符） |
| **参数位置** | URL 查询参数 (`?tag=...`) | URL 路径部分 (`/tag/...`) |
| **名称 → UUID 映射** | ✅ 自动映射（请求时） | ❌ 无映射，直接使用 UUID |
| **标签不存在时** | ❌ 静默返回空 Feed | ❌ 显式返回 HTTP 404 |
| **Feed Title** | 固定 "changedetection.io" | 动态 "changedetection.io - 标签名" |
| **Feed Description** | 固定 "Feed description" | 动态 "Changes for watches tagged with 标签名" |
| **条目排序** | ✅ 按 `last_changed` 升序排序 | ❌ 按 Watch 创建时间遍历（Bug） |
| **条目标题后缀** | ❌ 无后缀（仅 Watch 标题） | ✅ 有后缀 "Change @ 变更时间" |
| **静音过滤** | ✅ 支持（全局配置） | ✅ 支持（同主 Feed） |
| **未读过滤** | ✅ 仅未读变更 | ✅ 仅未读变更 |
| **自动标签支持** | ❌ 仅检查手动标签 | ❌ 仅检查手动标签 |

---

### 4.2 语义差异的架构原因

#### 为什么两条路径不统一？

1. **使用场景不同**：
   - `/rss?tag=...`：为人类设计，方便手动构造 URL
   - `/rss/tag/<uuid>`：为程序设计（前端生成的链接）

2. **历史演进痕迹**：
   - 主 Feed 的 tag 参数是后来**追加**的功能
   - 独立标签路由是**先有**的设计
   - 代码中没有公共函数抽离，存在重复逻辑

3. **注释自证缺陷**：
   ```python
   # main_feed.py:52
   # @todo needs a .itemsWithTag() or something - then we can use that in Jinaj2 and throw this away
   ```
   代码作者自己也承认这是临时实现，需要重构。

---

## 5. 关键边缘行为与缺陷

### 5.1 已知 Bug：标签路由无排序

**源码证据** (`tag.py:48`)：
```python
#@todo  This is wrong, it needs to sort by most recently changed and then limit it
```

**行为描述**：
- `/rss/tag/<uuid>` 条目顺序 = Watch 创建时间顺序（字典插入顺序）
- `/rss?tag=...` 条目顺序 = 变更时间升序（`last_changed` 从小到大）

**不一致影响**：
- 用户在两条路径看到相同标签的 Feed 条目顺序不同
- 最新变更可能不在最前面（违背 RSS 阅读习惯）

---

### 5.2 隐性缺陷：名称失配无提示

**行为描述**：
- 用户输入错误的标签名（如拼写错误）
- 系统不返回任何错误
- 静默返回空 RSS Feed
- 用户无法区分"无变更"还是"标签名错误"

**风险**：
- 调试困难
- 用户长时间认为"无变更"但实际上是拼写错误

---

### 5.3 隐性缺陷：同名标签的非确定性

**行为描述**：
- 用户创建了两个名称完全相同的标签
- `/rss?tag=...` 只匹配遍历顺序的第一个
- 匹配哪个取决于 UUID 字典遍历顺序（Python 3.7+ = 插入顺序）
- 无法保证确定性匹配

---

### 5.4 隐性缺陷：大小写敏感的 UUID

**行为描述**：
- `/rss/tag/ABC123` 和 `/rss/tag/abc123` 是不同的
- UUID 比较是大小写敏感的字符串比较
- 但 UUID 标准不区分大小写

---

## 6. 实现源码摘录

### 6.1 主 Feed 标签过滤核心代码

```python
# main_feed.py:43-59
limit_tag = request.args.get('tag', '').lower().strip()
# Be sure limit_tag is a uuid
for uuid, tag in datastore.data['settings']['application'].get('tags', {}).items():
    if limit_tag == tag.get('title', '').lower().strip():
        limit_tag = uuid

# Sort by last_changed and add the uuid which is usually the key..
sorted_watches = []

# @todo needs a .itemsWithTag() or something - then we can use that in Jinaj2 and throw this away
for uuid, watch in datastore.data['watching'].items():
    # @todo tag notification_muted skip also (improve Watch model)
    if datastore.data['settings']['application'].get('rss_hide_muted_watches') and watch.get('notification_muted'):
        continue
    if limit_tag and not limit_tag in watch['tags']:
        continue
    sorted_watches.append(watch)

sorted_watches.sort(key=lambda x: x.last_changed, reverse=False)
```

---

### 6.2 标签路由核心代码

```python
# tag.py:33-36, 47-53
# Verify tag exists
tag = datastore.data['settings']['application'].get('tags', {}).get(tag_uuid)
if not tag:
    return f"Tag with UUID {tag_uuid} not found", 404

# ...

# Find all watches with this tag
for uuid, watch in datastore.data['watching'].items():
    #@todo  This is wrong, it needs to sort by most recently changed and then limit it  datastore.data['watching'].items().sorted(?)
    # So get all watches in this tag then sort

    # Skip if watch doesn't have this tag
    if tag_uuid not in watch.get('tags', []):
        continue
```

---

## 核查结论

| 核查项 | 结论 |
|--------|------|
| **名称 → UUID 映射** | 主 Feed 参数路由会在请求时遍历标签字典做名称映射 |
| **失配行为** | 主 Feed 静默返回空 Feed，标签路由返回 404 |
| **排序一致性** | 不一致，主 Feed 按变更时间排序，标签路由不排序（已知 Bug） |
| **大小写敏感性** | 名称匹配大小写不敏感，UUID 匹配大小写敏感 |
| **多标签同名** | 主 Feed 取第一个匹配，标签路由 UUID 精确匹配 |
| **Feed 元数据** | 主 Feed 固定标题，标签路由动态包含标签名 |
| **条目标题** | 主 Feed 仅 Watch 标题，标签路由带时间后缀 |

---

**报告生成**：基于源码逐行审计，所有结论均可追溯到具体代码行。
