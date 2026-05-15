# 备份恢复与 Datastore 合并机制深度分析报告

## 目录
1. [核心计数字段统一定义](#1-核心计数字段统一定义)
2. [四开关恢复分支断言映射矩阵](#2-四开关恢复分支断言映射矩阵)
3. [同一 UUID 双文件执行顺序与日志信号](#3-同一-uuid-双文件执行顺序与日志信号)
4. [存储主体架构](#4-存储主体架构)
5. [恢复分支逻辑详解](#5-恢复分支逻辑详解)
6. [测试验证方式分析](#6-测试验证方式分析)
7. [代码分支覆盖情况](#7-代码分支覆盖情况)
8. [关键发现与建议](#8-关键发现与建议)

---

## 1. 核心计数字段统一定义

### 1.1 字段命名规范（严格统一）

| 字段名 | 类型 | 含义 | 生效条件 |
|--------|------|------|---------|
| **restored_groups** | int | 成功恢复的 Tag 数量 | `include_groups=True` 时递增 |
| **skipped_groups** | int | 跳过的已存在 Tag 数量 | `include_groups=True AND include_groups_replace=False AND uuid exists` 时递增 |
| **restored_watches** | int | 成功恢复的 Watch 数量 | `include_watches=True` 时递增 |
| **skipped_watches** | int | 跳过的已存在 Watch 数量 | `include_watches=True AND include_watches_replace=False AND uuid exists` 时递增 |

### 1.2 代码中实际赋值位置

```python
# restore.py 第 90-94 行 - Tag 跳过计数
if uuid in current_tags and not include_groups_replace:
    logger.debug(f"Restore: skipping existing group {uuid} (replace not requested)")
    skipped_groups += 1
    continue

# restore.py 第 122 行 - Tag 恢复计数
restored_groups += 1
logger.success(f"Restore: group '{title}' ({uuid}) restored")

# restore.py 第 128-131 行 - Watch 跳过计数
if uuid in current_watches and not include_watches_replace:
    logger.debug(f"Restore: skipping existing watch {uuid} (replace not requested)")
    skipped_watches += 1
    continue

# restore.py 第 154 行 - Watch 恢复计数
restored_watches += 1
logger.success(f"Restore: watch '{url}' ({uuid}) restored")
```

### 1.3 命名空间约定说明

| 场景 | 术语 | 备注 |
|------|------|------|
| **代码内部** | `groups` | `import_from_zip` 函数内的变量和日志统一使用 `groups` |
| **UI 展示** | `tags` | 用户界面可能显示为 "Tags" 或 "标签" |
| **数据结构** | `tags` | `datastore.data['settings']['application']['tags']` |
| **文件名** | `tag.json` | 每个 Tag 目录下的 JSON 文件名 |

> 🔍 **重要：恢复代码内部 100% 使用 `_groups` 后缀，不使用 `_tags`。**
> 变量命名完全一致：`current_tags` 是个例外（来自 datastore 数据结构），但计数全部使用 `*_groups`。

---

## 2. 四开关恢复分支断言映射矩阵

### 2.1 测试用例设计总览

四个开关：`include_groups(G)`、`include_groups_replace(Rg)`、`include_watches(W)`、`include_watches_replace(Rw)`

每个开关 2 种状态 → 总计 2^4 = **16 种独立分支组合**

### 2.2 分支映射表（可直接转换为测试代码）

#### 前置条件约定

| 符号 | 含义 |
|------|------|
| `T1` | 已存在 Tag（恢复前 datastore 中有此 UUID） |
| `T1'` | 备份 zip 中的 T1（内容可能不同） |
| `W1` | 已存在 Watch（恢复前 datastore 中有此 UUID） |
| `W1'` | 备份 zip 中的 W1（内容可能不同） |
| `T2`, `W2` | 备份 zip 中存在，但 datastore 中不存在（新增） |
| `{T1, T2}` | zip 中包含 2 个 Tag 目录 |
| `{W1, W2}` | zip 中包含 2 个 Watch 目录 |

---

### 矩阵 1：Tag 相关分支（8 种组合）

| 测试 ID | G | Rg | W | Rw | 输入构造 | 期望计数值 | 期望对象状态 |
|---------|---|---|---|---|---------|-----------|------------|
| **T-001** | ❌ | ❌ | ✅ | ✅ | zip={T1, T2}, datastore={T1} | restored_groups=0, skipped_groups=0 | T1 保持不变，T2 不创建 |
| **T-002** | ❌ | ✅ | ✅ | ✅ | zip={T1, T2}, datastore={T1} | restored_groups=0, skipped_groups=0 | T1 保持不变，T2 不创建（Rg 不影响因为 G=False） |
| **T-003** | ✅ | ❌ | ✅ | ✅ | zip={T1, T2}, datastore={T1} | restored_groups=1, skipped_groups=1 | T1 保持不变（未替换），T2 被新增 |
| **T-004** | ✅ | ✅ | ✅ | ✅ | zip={T1, T2}, datastore={T1} | restored_groups=2, skipped_groups=0 | T1 被替换为 T1'，T2 被新增 |
| **T-005** | ✅ | ❌ | ✅ | ✅ | zip={T2}, datastore={} | restored_groups=1, skipped_groups=0 | T2 被新增（无冲突） |
| **T-006** | ✅ | ✅ | ✅ | ✅ | zip={T2}, datastore={} | restored_groups=1, skipped_groups=0 | T2 被新增（Rg 对新增无影响） |
| **T-007** | ✅ | ❌ | ✅ | ✅ | zip={T1}, datastore={T1} | restored_groups=0, skipped_groups=1 | T1 保持不变（全部跳过） |
| **T-008** | ✅ | ✅ | ✅ | ✅ | zip={T1}, datastore={T1} | restored_groups=1, skipped_groups=0 | T1 被替换（全部恢复） |

> Tag 分支可观测日志信号：
> - 跳过：`logger.debug("Restore: skipping existing group {uuid} (replace not requested)")`
> - 恢复：`logger.success("Restore: group '{title}' ({uuid}) restored")`

---

### 矩阵 2：Watch 相关分支（8 种组合）

| 测试 ID | G | Rg | W | Rw | 输入构造 | 期望计数值 | 期望对象状态 |
|---------|---|---|---|---|---------|-----------|------------|
| **W-001** | ✅ | ✅ | ❌ | ❌ | zip={W1, W2}, datastore={W1} | restored_watches=0, skipped_watches=0 | W1 保持不变，W2 不创建 |
| **W-002** | ✅ | ✅ | ❌ | ✅ | zip={W1, W2}, datastore={W1} | restored_watches=0, skipped_watches=0 | W1 保持不变，W2 不创建（Rw 不影响因为 W=False） |
| **W-003** | ✅ | ✅ | ✅ | ❌ | zip={W1, W2}, datastore={W1} | restored_watches=1, skipped_watches=1 | W1 保持不变（未替换），W2 被新增 |
| **W-004** | ✅ | ✅ | ✅ | ✅ | zip={W1, W2}, datastore={W1} | restored_watches=2, skipped_watches=0 | W1 被替换为 W1'，W2 被新增 |
| **W-005** | ✅ | ✅ | ✅ | ❌ | zip={W2}, datastore={} | restored_watches=1, skipped_watches=0 | W2 被新增（无冲突） |
| **W-006** | ✅ | ✅ | ✅ | ✅ | zip={W2}, datastore={} | restored_watches=1, skipped_watches=0 | W2 被新增（Rw 对新增无影响） |
| **W-007** | ✅ | ✅ | ✅ | ❌ | zip={W1}, datastore={W1} | restored_watches=0, skipped_watches=1 | W1 保持不变（全部跳过） |
| **W-008** | ✅ | ✅ | ✅ | ✅ | zip={W1}, datastore={W1} | restored_watches=1, skipped_watches=0 | W1 被替换（全部恢复） |

> Watch 分支可观测日志信号：
> - 跳过：`logger.debug("Restore: skipping existing watch {uuid} (replace not requested)")`
> - 恢复：`logger.success("Restore: watch '{url}' ({uuid}) restored")`

---

### 矩阵 3：混合场景分支（覆盖 16 种组合的关键子集）

| 测试 ID | G | Rg | W | Rw | 输入构造 | 期望计数值 | 期望对象状态 |
|---------|---|---|---|---|---------|-----------|------------|
| **M-001** | ❌ | ❌ | ❌ | ❌ | zip={T1, W1}, datastore={} | restored_*=0, skipped_*=0 | 什么都不恢复 |
| **M-002** | ❌ | ❌ | ✅ | ✅ | zip={T1, W1}, datastore={} | restored_groups=0, restored_watches=1 | W1 被恢复，T1 不恢复 |
| **M-003** | ✅ | ✅ | ❌ | ❌ | zip={T1, W1}, datastore={} | restored_groups=1, restored_watches=0 | T1 被恢复，W1 不恢复 |
| **M-004** | ✅ | ❌ | ✅ | ❌ | zip={T1, W1}, datastore={T1, W1} | restored_groups=0, skipped_groups=1, restored_watches=0, skipped_watches=1 | T1、W1 均保持不变 |
| **M-005** | ✅ | ✅ | ✅ | ❌ | zip={T1, W1}, datastore={T1, W1} | restored_groups=1, skipped_groups=0, restored_watches=0, skipped_watches=1 | T1 被替换，W1 保持不变 |
| **M-006** | ✅ | ❌ | ✅ | ✅ | zip={T1, W1}, datastore={T1, W1} | restored_groups=0, skipped_groups=1, restored_watches=1, skipped_watches=0 | T1 保持不变，W1 被替换 |
| **M-007** | ✅ | ✅ | ✅ | ✅ | zip={T1, W1}, datastore={T1, W1} | restored_groups=1, skipped_groups=0, restored_watches=1, skipped_watches=0 | T1、W1 均被替换 |
| **M-008** | ✅ | ❌ | ✅ | ❌ | zip={T2, W2}, datastore={T1, W1} | restored_groups=1, skipped_groups=0, restored_watches=1, skipped_watches=0 | T2、W2 新增，T1、W1 不变 |

---

### 2.3 可执行断言模板（Python）

```python
from changedetectionio.blueprint.backups.restore import import_from_zip

def assert_restore_scenario(
    # 四开关参数
    include_groups, include_groups_replace,
    include_watches, include_watches_replace,
    # 输入数据
    zip_content,          # zip 内的 UUID-内容 映射
    initial_datastore,    # 初始 datastore 状态
    # 期望输出
    expected_restored_groups, expected_skipped_groups,
    expected_restored_watches, expected_skipped_watches,
    # 期望对象状态： {uuid: expected_title_or_url}
    expected_object_states
):
    """
    通用恢复场景断言模板
    
    使用示例：
        assert_restore_scenario(
            include_groups=True, include_groups_replace=False,
            include_watches=True, include_watches_replace=True,
            zip_content={
                'uuid-t1': {'tag.json': {'title': 'T1-new'}},
                'uuid-w1': {'watch.json': {'url': 'http://w1-new.com'}},
            },
            initial_datastore={'tags': {'uuid-t1': {'title': 'T1-old'}}, 'watches': {'uuid-w1': {'url': 'http://w1-old.com'}}},
            expected_restored_groups=0, expected_skipped_groups=1,
            expected_restored_watches=1, expected_skipped_watches=0,
            expected_object_states={
                'uuid-t1': 'T1-old',    # 未替换
                'uuid-w1': 'http://w1-new.com',  # 已替换
            }
        )
    """
    # 步骤 1：构造测试 zip
    test_zip = create_test_zip(zip_content)
    
    # 步骤 2：初始化 datastore 状态
    datastore = setup_datastore(initial_datastore)
    
    # 步骤 3：执行恢复（捕获返回值）
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=include_groups,
        include_groups_replace=include_groups_replace,
        include_watches=include_watches,
        include_watches_replace=include_watches_replace
    )
    
    # 步骤 4：断言计数值
    assert result['restored_groups'] == expected_restored_groups, \
        f"restored_groups mismatch: expected {expected_restored_groups}, got {result['restored_groups']}"
    assert result['skipped_groups'] == expected_skipped_groups, \
        f"skipped_groups mismatch: expected {expected_skipped_groups}, got {result['skipped_groups']}"
    assert result['restored_watches'] == expected_restored_watches, \
        f"restored_watches mismatch: expected {expected_restored_watches}, got {result['restored_watches']}"
    assert result['skipped_watches'] == expected_skipped_watches, \
        f"skipped_watches mismatch: expected {expected_skipped_watches}, got {result['skipped_watches']}"
    
    # 步骤 5：断言对象状态
    for uuid, expected_value in expected_object_states.items():
        if uuid in datastore.data['settings']['application']['tags']:
            actual = datastore.data['settings']['application']['tags'][uuid].get('title')
            assert actual == expected_value, \
                f"Tag {uuid} title mismatch: expected {expected_value}, got {actual}"
        elif uuid in datastore.data['watching']:
            actual = datastore.data['watching'][uuid].get('url')
            assert actual == expected_value, \
                f"Watch {uuid} url mismatch: expected {expected_value}, got {actual}"
```

---

## 3. 同一 UUID 双文件执行顺序与日志信号

### 3.1 问题背景

**关键代码结构：**
```python
# restore.py 第 89-155 行
if include_groups and os.path.exists(tag_json_path):
    # --- Tag 恢复分支 ---
    # (执行后 continue 跳到下一个目录)
elif include_watches and os.path.exists(watch_json_path):
    # --- Watch 恢复分支 ---
    # (只有 if 分支不执行才会走到这里)
```

**结论：这是 `if-elif` 结构，**不是**两个独立的 `if`！**

### 3.2 执行顺序决策树

```
对单个 UUID 目录进行恢复处理：
├── 先检查 tag.json 是否存在 AND include_groups=True
│   ├── 是 → 进入 Tag 分支：
│   │   ├── 执行覆盖/跳过逻辑
│   │   ├── 更新 restored_groups 或 skipped_groups
│   │   └── continue → 结束此 UUID 处理（跳过 Watch 分支）
│   │
│   └── 否 → 继续检查 watch.json
│       ├── 检查 watch.json 是否存在 AND include_watches=True
│       │   ├── 是 → 进入 Watch 分支：
│       │   │   ├── 执行覆盖/跳过逻辑
│       │   │   └── 更新 restored_watches 或 skipped_watches
│       │   │
│       │   └── 否 → 静默跳过（无日志）
│       │
│       └── 结束
│
└── 结果：
    如果同一 UUID 目录同时含有 tag.json 和 watch.json：
    → 当 include_groups=True 时，Watch 分支永远不会执行！
```

### 3.3 组合枚举表（2 × 2 × 2 = 8 种场景）

| 场景 | tag.json | watch.json | include_groups | include_watches | 实际行为 | 计数值变化 | 日志信号 |
|------|---------|-----------|---------------|-----------------|---------|-----------|---------|
| **C-001** | ✅ | ✅ | ✅ | ✅ | 只恢复 Tag，Watch 静默丢失 | restored_groups +1 | Tag 分支的 success 日志 |
| **C-002** | ✅ | ✅ | ✅ | ❌ | 只恢复 Tag | restored_groups +1 | Tag 分支的 success 日志 |
| **C-003** | ✅ | ✅ | ❌ | ✅ | 只恢复 Watch | restored_watches +1 | Watch 分支的 success 日志 |
| **C-004** | ✅ | ✅ | ❌ | ❌ | 都不恢复 | 无变化 | 无日志（静默跳过） |
| **C-005** | ✅ | ❌ | ✅ | ✅ | 只恢复 Tag | restored_groups +1 | 正常 Tag 日志 |
| **C-006** | ❌ | ✅ | ✅ | ✅ | 只恢复 Watch | restored_watches +1 | 正常 Watch 日志 |
| **C-007** | ✅ | ❌ | ❌ | ✅ | 都不恢复 | 无变化 | 无日志 |
| **C-008** | ❌ | ✅ | ✅ | ❌ | 都不恢复 | 无变化 | 无日志 |

### 3.4 关键场景的可观测日志信号

**场景 C-001：双文件 + 双开启 = Watch 丢失**
```python
# 输入构造
zip_content = {
    'same-uuid-123': {
        'tag.json': {'title': 'My Group'},
        'watch.json': {'url': 'http://example.com'}
    }
}
include_groups = True
include_watches = True

# 可观测日志：
logger.success("Restore: group 'My Group' (same-uuid-123) restored")
# 🔴 注意：没有任何 Watch 相关日志！也没有警告说明 Watch 被丢弃！

# 计数值结果：
restored_groups = 1
skipped_groups = 0
restored_watches = 0    # 🔴 静默丢失
skipped_watches = 0
```

**场景 C-003：双文件 + 只开 Watch = 正常恢复**
```python
# 输入构造
zip_content = {
    'same-uuid-123': {
        'tag.json': {'title': 'My Group'},
        'watch.json': {'url': 'http://example.com'}
    }
}
include_groups = False
include_watches = True

# 可观测日志：
logger.success("Restore: watch 'http://example.com' (same-uuid-123) restored")

# 计数值结果：
restored_groups = 0
skipped_groups = 0
restored_watches = 1
skipped_watches = 0
```

**场景 C-001 + 已存在 + 不替换：Tag 跳过，Watch 也跳过**
```python
# 输入构造
datastore 已有 same-uuid-123 Tag
zip_content same-uuid-123 含 tag.json + watch.json
include_groups=True, include_groups_replace=False
include_watches=True

# 可观测日志：
logger.debug("Restore: skipping existing group same-uuid-123 (replace not requested)")
# 🔴 Watch 分支也不会执行，因为走到 Tag 的 continue 了！

# 计数值结果：
skipped_groups = 1
restored_groups = 0
restored_watches = 0    # 🔴 即使 include_watches=True，也不会检查 Watch！
skipped_watches = 0     # 🔴 即使 UUID 已存在，也不会递增 skipped_watches！
```

> ⚠️ **严重风险警告：** 当同一 UUID 目录同时有两种文件时，如果 Tag 分支触发了 `continue`（不管是恢复成功还是跳过），**Watch 分支的所有逻辑都不会执行**，包括：
> 1. 不会检查 watch.json 是否存在
> 2. 不会检查 Watch 是否已存在
> 3. 不会递增 skipped_watches
> 4. 完全静默——没有日志、没有警告、没有任何反馈

---

## 4. 存储主体架构

### 4.1 实体类继承体系

```
┌─────────────────────────────────────────────────────────────────┐
│                        实体类继承体系                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  watch_base(dict)                 (changedetectionio/model/)   │
│       │                                                          │
│       ├─────────── 通用字段定义 (url, title, tags, etc.)        │
│       ├─────────── commit() 方法 (持久化入口)                   │
│       └─────────── _get_commit_data() 方法 (序列化数据)          │
│                           │                                        │
│         ┌─────────────────┴─────────────────┐                    │
│         │                                   │                    │
│  EntityPersistenceMixin              EntityPersistenceMixin      │
│  (持久化能力混入)                        (持久化能力混入)         │
│         │                                   │                    │
│         ▼                                   ▼                    │
│  Watch.model (Watch.py)               Tag.model (Tag.py)         │
│  {uuid}/watch.json                       {uuid}/tag.json         │
│  max_size: 10MB                          max_size: 1MB          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 关键类职责划分

| 类/模块 | 核心职责 | 关键方法 |
|--------|---------|---------|
| **watch_base** | 通用字段定义、commit 入口逻辑、通用行为 | `commit()`, `_get_commit_data()` |
| **EntityPersistenceMixin** | 实体类型识别、调用底层原子写入 | `_save_to_disk()` |
| **save_entity_atomic** | 文件级原子写入、大小校验 | `save_json_atomic()` |
| **Watch.model** | Watch 专用逻辑、历史记录管理 | `rehydrate_entity()` (datastore) |
| **Tag.model** | Tag 专用逻辑、URL 匹配规则 | `matches_url()` |

### 4.3 文件系统存储结构

```
datastore/
├── changedetection.json              # 全局设置 (不含 watches/tags)
│   ├── settings
│   │   ├── application
│   │   │   └── tags: {}             # 仅存 UUID 引用（运行时对象）
│   │   ├── requests
│   │   └── headers
│   └── app_guid, build_sha, etc.
│
├── {watch-uuid}/                     # 每个 Watch 独立目录
│   ├── watch.json                    # Watch 配置（原子写入）
│   ├── history.txt                   # 变更历史记录
│   ├── snapshot.txt                  # 最新快照内容
│   ├── last-checksum.txt             # 校验和
│   ├── last-screenshot.png           # 截图（如有）
│   └── ...
│
└── {tag-uuid}/                       # 每个 Tag 独立目录
    └── tag.json                      # Tag 配置（原子写入）
```

---

## 5. 恢复分支逻辑详解

### 5.1 import_from_zip 完整分支树

```python
# restore.py: import_from_zip()
#
# 入口: import_from_zip(zip_stream, datastore, 
#                      include_groups, include_groups_replace,
#                      include_watches, include_watches_replace)
#     │
#     ├── 步骤 1: 准备数据
#     │     ├── current_tags = datastore.data['settings']['application']['tags']
#     │     └── current_watches = datastore.data['watching']
#     │
#     ├── 步骤 2: 解压 ZIP（安全检查）
#     │     ├── 检查解压后总大小 (防止 Zip Bomb)
#     │     │     └── IF total_uncompressed > LIMIT → ValueError
#     │     └── 检查路径遍历 (防止 Zip Slip)
#     │           └── IF realpath 跳出 tmpdir → ValueError
#     │
#     └── 步骤 3: 扫描每个 UUID 目录
#           │
#           ├── 检查: 是否为有效目录？
#           │     └── NO → continue（跳过）
#           │
#           ├── 检查: 目录名是否为有效 UUID 格式？
#           │     └── NO → logger.warning + continue（跳过）
#           │
#           ├── 检查: 目录中是否有 tag.json？
#           │     │
#           │     └── YES + include_groups=True → ──────────────┐
#           │                                                     │
#           │                                                     ▼
#           │                                         ┌─────────────────────┐
#           │                                         │ Tag 恢复分支          │
#           │                                         └─────────────────────┘
#           │                                         │
#           │                                         ├── 检查: UUID 是否已存在？
#           │                                         │     ├── YES
#           │                                         │     │   └── 检查: include_groups_replace?
#           │                                         │     │         ├── False → skipped_groups +=1 + continue
#           │                                         │     │         └── True  → 继续（覆盖模式）
#           │                                         │     └── NO → 继续（新增模式）
#           │                                         │
#           │                                         ├── 读取 tag.json
#           │                                         │     └── JSON 解析失败 → logger.error + continue
#           │                                         │
#           │                                         ├── 目录操作
#           │                                         │     ├── IF 目标目录存在 → shutil.rmtree() 删除
#           │                                         │     └── shutil.copytree() 复制新目录
#           │                                         │
#           │                                         ├── 对象重新水化
#           │                                         │     ├── tag_data['uuid'] = uuid
#           │                                         │     ├── tag_data['processor'] = 'restock_diff'
#           │                                         │     └── Tag.model() 创建对象
#           │                                         │
#           │                                         ├── 存入内存 + 持久化
#           │                                         │     ├── current_tags[uuid] = tag_obj
#           │                                         │     ├── tag_obj.commit() → 写入 tag.json
#           │                                         │     └── restored_groups += 1
#           │                                         │
#           │                                         └── END → continue（处理下一个目录）
#           │
#           ├── 检查: 目录中是否有 watch.json？
#           │     │
#           │     └── YES + include_watches=True → ─────────────┐
#           │                                                     │
#           │                                                     ▼
#           │                                         ┌─────────────────────┐
#           │                                         │ Watch 恢复分支        │
#           │                                         └─────────────────────┘
#           │                                         │
#           │                                         ├── 检查: UUID 是否已存在？
#           │                                         │     ├── YES
#           │                                         │     │   └── 检查: include_watches_replace?
#           │                                         │     │         ├── False → skipped_watches +=1 + continue
#           │                                         │     │         └── True  → 继续（覆盖模式）
#           │                                         │     └── NO → 继续（新增模式）
#           │                                         │
#           │                                         ├── 读取 watch.json
#           │                                         │     └── JSON 解析失败 → logger.error + continue
#           │                                         │
#           │                                         ├── 目录操作
#           │                                         │     ├── IF 目标目录存在 → shutil.rmtree() 删除
#           │                                         │     └── shutil.copytree() 复制新目录
#           │                                         │
#           │                                         ├── 对象重新水化
#           │                                         │     ├── watch_data['uuid'] = uuid
#           │                                         │     └── datastore.rehydrate_entity() → 创建 Watch 对象
#           │                                         │
#           │                                         ├── 存入内存 + 持久化
#           │                                         │     ├── current_watches[uuid] = watch_obj
#           │                                         │     ├── watch_obj.commit() → 写入 watch.json
#           │                                         │     └── restored_watches += 1
#           │                                         │
#           │                                         └── END → continue（处理下一个目录）
#           │
#           └── 其他情况（既无 tag.json 也无 watch.json）
#                 └── 静默跳过（无日志 + continue）
#
#     └── 步骤 4: 全局持久化
#           └── datastore.commit() → 写入 changedetection.json
```

---

## 6. 测试验证方式分析

### 6.1 现有测试用例覆盖情况

| 测试用例 | 位置 | 覆盖分支 | 验证方式 |
|---------|------|---------|---------|
| **test_backup_restore** | `test_backup.py:122-202` | 完整备份恢复闭环（清空后恢复所有） | 1. 创建 1 watch + 2 tags<br>2. 备份 → 清空 → 恢复<br>3. 验证 watch URL 正确<br>4. 验证 tag title 正确<br>5. 验证对象类型正确 (Watch.model / Tag.model)<br>6. 验证 watch 历史记录正确 |
| **test_backup_restore_zip_slip_rejected** | `test_backup.py:205-226` | Zip Slip 安全防护 | 1. 创建恶意 zip (包含 `../escaped.txt`)<br>2. 直接调用 `import_from_zip()`<br>3. 断言抛出 `ValueError` 且消息包含 "Zip Slip" |
| **test_backup_restore_zip_bomb_rejected** | `test_backup.py:229-261` | Zip Bomb 安全防护 | 1. 临时修改 `_MAX_DECOMPRESSED_BYTES` = 50KB<br>2. 创建压缩率极高的 zip (100KB 零数据)<br>3. 直接调用 `import_from_zip()`<br>4. 断言抛出 `ValueError` 且消息包含 "decompressed size"<br>5. finally 恢复原始限制 |

### 6.2 test_backup_restore 详细验证流程

```python
# 测试关键点分析 (test_backup.py 第 122-202 行)

# 阶段 1: 准备数据
uuid = datastore.add_watch(url=watch_url)
tag_uuid = datastore.add_tag(title="Tasty backup tag")
tag_uuid2 = datastore.add_tag(title="Tasty backup tag number two")

# 阶段 2: 创建并下载备份
client.get(url_for("backups.request_backup"))
time.sleep(4)
res = client.get(url_for("backups.download_backup", filename="latest"))
zip_data = res.data

# 验证 ZIP 内容
backup = ZipFile(io.BytesIO(zip_data))
names = backup.namelist()
assert f"{uuid}/watch.json" in names
assert f"{tag_uuid}/tag.json" in names
assert f"{tag_uuid2}/tag.json" in names

# 阶段 3: 清空现有数据
datastore.delete('all')
client.get(url_for("tags.delete_all"))

# 阶段 4: 执行恢复（使用全 True 组合：G=True, Rg=True, W=True, Rw=True）
res = client.post(
    url_for("backups.restore.backups_restore_start"),
    data={
        'zip_file': (io.BytesIO(zip_data), 'backup.zip'),
        'include_groups': 'y',
        'include_groups_replace_existing': 'y',
        'include_watches': 'y',
        'include_watches_replace_existing': 'y',
    },
    content_type='multipart/form-data'
)
time.sleep(2)

# 阶段 5: 验证恢复结果（只验证 happy path）
restored_watch = datastore.data['watching'].get(uuid)
assert restored_watch is not None
assert restored_watch['url'] == watch_url
assert isinstance(restored_watch, Watch.model)
assert restored_watch.history_n >= 1

restored_tags = datastore.data['settings']['application']['tags']
assert restored_tags.get(tag_uuid)['title'] == "Tasty backup tag"
```

### 6.3 现有测试覆盖缺口（对照第 2 节矩阵）

**已覆盖：**
- ✅ M-007：全 True + 清空 = 全恢复

**完全未覆盖：**
- ❌ T-001 ~ T-008：Tag 分支的 8 种组合
- ❌ W-001 ~ W-008：Watch 分支的 8 种组合
- ❌ M-001 ~ M-006：混合场景的 6 种组合
- ❌ C-001 ~ C-008：同一 UUID 双文件场景 8 种组合

**总计：30 种分支组合，仅覆盖 1 种 → 覆盖率 3.3%**

---

## 7. 代码分支覆盖情况

### 7.1 已覆盖分支

| 分支 | 覆盖情况 | 测试用例 |
|------|---------|---------|
| ✅ 解压大小限制检查 | 完全覆盖 | `test_backup_restore_zip_bomb_rejected` |
| ✅ Zip Slip 路径遍历检查 | 完全覆盖 | `test_backup_restore_zip_slip_rejected` |
| ✅ Tag.json 存在 + include_groups=True | 覆盖 | `test_backup_restore` |
| ✅ Watch.json 存在 + include_watches=True | 覆盖 | `test_backup_restore` |
| ✅ 目标目录不存在（新增） | 覆盖 | `test_backup_restore`（清空后恢复） |
| ✅ tag_obj.commit() 持久化 | 间接覆盖 | `test_backup_restore` |
| ✅ watch_obj.commit() 持久化 | 间接覆盖 | `test_backup_restore` |
| ✅ datastore.commit() 持久化 | 间接覆盖 | `test_backup_restore` |

### 7.2 未覆盖分支清单

| 分支 | 覆盖情况 | 对应测试矩阵 ID |
|------|---------|----------------|
| ❌ Tag 已存在 + include_groups_replace=False → 跳过 | 未覆盖 | T-003, T-007 |
| ❌ Tag 已存在 + include_groups_replace=True → 替换 | 未覆盖 | T-004, T-008 |
| ❌ Watch 已存在 + include_watches_replace=False → 跳过 | 未覆盖 | W-003, W-007 |
| ❌ Watch 已存在 + include_watches_replace=True → 替换 | 未覆盖 | W-004, W-008 |
| ❌ include_groups=False 时跳过所有 Tags | 未覆盖 | T-001, T-002 |
| ❌ include_watches=False 时跳过所有 Watches | 未覆盖 | W-001, W-002 |
| ❌ 同一 UUID 双文件时 Watch 被丢弃 | 未覆盖 | C-001 ~ C-004 |
| ❌ 无效 UUID 目录跳过 | 未覆盖 | - |
| ❌ tag.json JSON 解析失败 | 未覆盖 | - |
| ❌ watch.json JSON 解析失败 | 未覆盖 | - |
| ❌ 目录既无 tag.json 也无 watch.json | 未覆盖 | - |

---

## 8. 关键发现与建议

### 8.1 关键发现

**发现 1：命名一致性良好，但术语体系需澄清**
- ✅ 恢复计数统一使用 `_groups` 后缀（`restored_groups`, `skipped_groups`）
- ✅ 变量命名在 `import_from_zip` 函数内完全一致
- ⚠️ 但 UI/展示层可能用 "Tags"，需注意术语映射

**发现 2：同一 UUID 双文件场景存在静默数据丢失风险**
- `if-elif` 结构导致 Tag 分支 `continue` 后，Watch 分支完全不执行
- 没有日志、没有警告、没有计数反馈
- 即使 Watch 分支也应该跳过并递增 `skipped_watches`，实际上也不会执行

**发现 3：恢复分支覆盖率极低（约 3.3%）**
- 30 种分支组合仅覆盖 1 种（全清空后全恢复的 happy path）
- 最核心的合并逻辑（已存在 + 跳过/替换）完全没有测试
- 双文件冲突场景完全没有测试覆盖

**发现 4：恢复结果无法通过测试断言验证**
- 当前测试通过 Flask 后台线程执行恢复，`import_from_zip` 的返回值（计数字典）被丢弃
- 测试无法直接获取 `restored_groups` 等值进行断言
- 只能间接验证最终对象状态，无法验证"跳过"逻辑（因为状态不变，无法区分"未恢复" vs "不存在"）

### 8.2 改进建议

**建议 1：立即补充最高优先级测试（按第 2 节矩阵）**

优先级 1（P0）：核心合并逻辑
- T-003：Tag 已存在 + 不替换 → 验证跳过逻辑
- T-004：Tag 已存在 + 替换 → 验证替换逻辑
- W-003：Watch 已存在 + 不替换 → 验证跳过逻辑
- W-004：Watch 已存在 + 替换 → 验证替换逻辑

优先级 2（P1）：开关隔离测试
- T-001：include_groups=False → Tags 不恢复
- W-001：include_watches=False → Watches 不恢复

优先级 3（P2）：风险场景测试
- C-001：双文件 + 双开启 → 验证 Watch 静默丢失（当前行为）或添加警告日志（改进后）

**建议 2：增加双文件检测与日志告警**

```python
# 在 restore.py 第 89 行前添加检测
has_tag = os.path.exists(tag_json_path)
has_watch = os.path.exists(watch_json_path)

if has_tag and has_watch:
    logger.warning(
        f"UUID {uuid} directory contains BOTH tag.json and watch.json. "
        f"Will process only Tag (priority), Watch will be SILENTLY SKIPPED. "
        f"This is likely a backup file corruption issue!"
    )
```

**建议 3：让测试可直接获取恢复结果**

```python
# 方法 A：提供同步调用入口（供测试使用）
# 在 backups/__init__.py 添加测试专用函数
def restore_backup_sync(zip_file, datastore, **options):
    """同步恢复备份（测试专用），返回计数字典"""
    return import_from_zip(zip_file, datastore, **options)

# 方法 B：将恢复结果写入 datastore，持久化可查
# 恢复完成后写入 datastore：
datastore.data['last_restore_result'] = {
    'timestamp': time.time(),
    'restored_groups': restored_groups,
    'skipped_groups': skipped_groups,
    'restored_watches': restored_watches,
    'skipped_watches': skipped_watches,
}
datastore.commit()
```

**建议 4：考虑重构分支结构**

将 `if-elif` 改为两个独立的 `if`，但增加 UUID 空间检查：
```python
# 当前：if-elif → 潜在数据丢失
if include_groups and os.path.exists(tag_json_path):
    process_tag()
elif include_watches and os.path.exists(watch_json_path):
    process_watch()

# 建议：两个独立 if + 跨类型 UUID 冲突检测
if include_groups and os.path.exists(tag_json_path):
    process_tag()
    
if include_watches and os.path.exists(watch_json_path):
    if uuid in current_tags:
        logger.error(f"UUID {uuid} conflict: already exists as Tag, skipping Watch restore")
    else:
        process_watch()
```

---

## 附录：核心代码位置速查表

| 功能 | 文件 | 行号范围 |
|------|------|---------|
| import_from_zip 主逻辑 | `backups/restore.py` | 40-169 |
| Tag 恢复分支（if 分支） | `backups/restore.py` | 89-124 |
| Watch 恢复分支（elif 分支） | `backups/restore.py` | 127-155 |
| Tag 跳过计数赋值 | `backups/restore.py` | 90-94 |
| Watch 跳过计数赋值 | `backups/restore.py` | 128-131 |
| restored_groups 递增 | `backups/restore.py` | 122 |
| restored_watches 递增 | `backups/restore.py` | 154 |
| commit() 方法 | `model/__init__.py` | 649-690 |
| 实体持久化 Mixin | `model/persistence.py` | 37-84 |
| 现有备份恢复测试 | `tests/test_backup.py` | 122-261 |
