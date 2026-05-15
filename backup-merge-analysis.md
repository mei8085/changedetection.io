# 备份恢复与 Datastore 合并测试分析报告

## 目录
1. [测试框架与 Fixture 速查](#1-测试框架与-fixture-速查)
2. [Tag 分支测试用例（8 个可执行版本）](#2-tag-分支测试用例8-个可执行版本)
3. [Watch 分支测试用例（8 个可执行版本）](#3-watch-分支测试用例8-个可执行版本)
4. [混合场景测试用例（4 个核心组合）](#4-混合场景测试用例4-个核心组合)
5. [同一 UUID 双文件冲突场景（8 个精确验证）](#5-同一-uuid-双文件冲突场景8-个精确验证)
6. [关键发现总结](#6-关键发现总结)

---

## 1. 测试框架与 Fixture 速查

### 1.1 可用 Fixture 列表（直接使用）

| Fixture | 来源 | 用途 |
|---------|------|------|
| `client` | pytest-flask | Flask 测试客户端，用于 HTTP 请求 |
| `live_server` | pytest-flask | 运行中的 Flask 应用，可获取 datastore |
| `datastore_path` | tests/conftest.py | 测试专用的临时数据目录路径 |
| `measure_memory_usage` | tests/conftest.py | 内存用量监控（可选） |
| `live_server.app.config['DATASTORE']` | app config | 获取 Datastore 实例（核心对象） |

### 1.2 Datastore API 真实签名（修正！）

```python
# ================ 真实 API 签名（非常重要！）================

# 添加 Watch（只有第一个参数是 url，没有 uuid 参数！）
watch_uuid = datastore.add_watch(
    url="http://example.com",  # 唯一必填
    tag='',                    # 可选
    extras=None,               # 可选
    tag_uuids=None,            # 可选
    save_immediately=True      # 可选
)
# 返回值：新生成的 UUID

# 添加 Tag（只有一个参数 title，没有 uuid 参数！）
tag_uuid = datastore.add_tag(
    title="My Tag"  # 唯一必填
)
# 返回值：新生成的 UUID（或已存在的同名 Tag 的 UUID）

# 获取恢复函数（从 backups blueprint 导入）
from changedetectionio.blueprint.backups.restore import import_from_zip

# 完整签名
result = import_from_zip(
    zip_stream=test_zip,           # BytesIO 对象
    datastore=datastore,           # Datastore 实例
    include_groups=True/False,     # 开关 1
    include_groups_replace=True/False,  # 开关 2
    include_watches=True/False,    # 开关 3
    include_watches_replace=True/False, # 开关 4
)
# 返回值：字典
# {
#   'restored_groups': int,
#   'skipped_groups': int,
#   'restored_watches': int,
#   'skipped_watches': int
# }
```

### 1.3 标准测试模板（可直接复制）

```python
import io
import os
import json
from zipfile import ZipFile

def test_name(client, live_server, datastore_path):
    """测试描述"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    # ========== 步骤 1：获取 datastore 实例 ==========
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 步骤 2：准备前置数据（如需要）==========
    # 例如：预先创建一个 Tag/Watch 来测试"已存在"场景
    existing_uuid = datastore.add_tag(title="Existing Tag")
    
    # ========== 步骤 3：构造测试 zip ==========
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        # 添加 tag.json
        tag_data = {'title': 'Tag From Backup'}
        zf.writestr(f"{existing_uuid}/tag.json", json.dumps(tag_data))
        
        # 可选：添加额外文件（如 history.txt）用于验证
        zf.writestr(f"{existing_uuid}/history.txt", "some history data")
    
    test_zip.seek(0)
    
    # ========== 步骤 4：执行恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,        # 根据测试场景设置
        include_groups_replace=True,# 根据测试场景设置
        include_watches=True,       # 根据测试场景设置
        include_watches_replace=True, # 根据测试场景设置
    )
    
    # ========== 步骤 5：断言计数器（第一层验证）==========
    assert result['restored_groups'] == 1, \
        f"Expected restored_groups=1, got {result['restored_groups']}"
    assert result['skipped_groups'] == 0, \
        f"Expected skipped_groups=0, got {result['skipped_groups']}"
    assert result['restored_watches'] == 0, \
        f"Expected restored_watches=0, got {result['restored_watches']}"
    assert result['skipped_watches'] == 0, \
        f"Expected skipped_watches=0, got {result['skipped_watches']}"
    
    # ========== 步骤 6：双重校验 - 内存对象状态 ==========
    # Tag 校验
    restored_tag = datastore.data['settings']['application']['tags'].get(existing_uuid)
    assert restored_tag is not None, "Tag should exist in memory after restore"
    assert restored_tag['title'] == 'Tag From Backup', \
        f"Tag title mismatch: expected 'Tag From Backup', got '{restored_tag['title']}'"
    
    # Watch 校验（如果有）
    restored_watch = datastore.data['watching'].get(existing_uuid)
    assert restored_watch is None, "Watch should NOT exist in memory"
    
    # ========== 步骤 7：双重校验 - 磁盘文件状态 ==========
    uuid_dir = os.path.join(datastore_path, existing_uuid)
    assert os.path.exists(uuid_dir), f"UUID directory should exist: {uuid_dir}"
    
    tag_json_path = os.path.join(uuid_dir, 'tag.json')
    assert os.path.exists(tag_json_path), f"tag.json should exist: {tag_json_path}"
    
    watch_json_path = os.path.join(uuid_dir, 'watch.json')
    assert not os.path.exists(watch_json_path), \
        f"watch.json should NOT exist (only Tag was restored): {watch_json_path}"
    
    # 验证磁盘 JSON 内容
    with open(tag_json_path, 'r') as f:
        disk_data = json.load(f)
    assert disk_data['title'] == 'Tag From Backup', \
        f"tag.json on disk has wrong title: {disk_data['title']}"
```

---

## 2. Tag 分支测试用例（8 个可执行版本）

### 通用常量定义

```python
TITLE_EXISTING = "Existing Tag Title"
TITLE_NEW_FROM_BACKUP = "New Tag From Backup"
TITLE_UPDATED_IN_BACKUP = "Updated Title From Backup"
```

---

### 测试 T-001：include_groups=False → Tags 完全不恢复

**场景：** zip 中有 Tag，但用户明确不勾选"恢复 Groups"

**预期：** 所有计数器为 0，内存和磁盘都没有 Tag

```python
def test_t001_include_groups_false(client, live_server, datastore_path):
    """T-001: include_groups=False → Tags 完全不恢复"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    import uuid
    
    datastore = live_server.app.config['DATASTORE']
    
    # 构造 zip：包含 1 个新 Tag
    new_uuid = str(uuid.uuid4())
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{new_uuid}/tag.json", 
                   json.dumps({'title': TITLE_NEW_FROM_BACKUP}))
    test_zip.seek(0)
    
    # 执行恢复（关键参数：include_groups=False）
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=False,          # 关键开关关闭
        include_groups_replace=True,   # 不影响，因为 include_groups=False
        include_watches=True,
        include_watches_replace=True,
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_groups'] == 0
    assert result['skipped_groups'] == 0
    
    # ========== 双重校验 1：内存对象状态 ==========
    assert new_uuid not in datastore.data['settings']['application']['tags']
    
    # ========== 双重校验 2：磁盘文件状态 ==========
    tag_dir = os.path.join(datastore_path, new_uuid)
    assert not os.path.exists(tag_dir), \
        f"Tag directory should NOT exist: {tag_dir}"
```

---

### 测试 T-002：include_groups=False（同 T-001，replace=True 不影响）

与 T-001 完全相同，只需将 `include_groups_replace=True` 改为 `False`。

**关键结论：** include_groups_replace 参数在 include_groups=False 时完全不起作用。

---

### 测试 T-003：Tag 已存在 + include_groups_replace=False → 跳过

**场景：** Datastore 中已有一个 Tag，备份 zip 中包含相同 UUID 的 Tag，但用户勾选不替换。

**预期：** 
- `restored_groups=0, skipped_groups=1`
- Tag title 保持原值（不被替换）
- 目录不被重新复制（额外文件不会出现）

```python
def test_t003_tag_exists_no_replace(client, live_server, datastore_path):
    """T-003: Tag 已存在 + include_groups_replace=False → 跳过（不覆盖）"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据：预先创建 Tag ==========
    existing_uuid = datastore.add_tag(title=TITLE_EXISTING)
    
    # 构造 zip：包含相同 UUID 的 Tag，但 title 不同 + 额外文件
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{existing_uuid}/tag.json", 
                   json.dumps({'title': TITLE_UPDATED_IN_BACKUP}))
        # 这个文件不应该出现在磁盘上（因为跳过了替换，目录没被复制）
        zf.writestr(f"{existing_uuid}/extra_file.txt", "should not exist")
    test_zip.seek(0)
    
    # ========== 执行恢复（关键参数：include_groups_replace=False）==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=False,    # 关键：不替换已存在的
        include_watches=True,
        include_watches_replace=True,
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_groups'] == 0, \
        "Should NOT restore any group (existing not replaced)"
    assert result['skipped_groups'] == 1, \
        "Should skip 1 existing group"
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_tag = datastore.data['settings']['application']['tags'].get(existing_uuid)
    assert restored_tag is not None, "Tag should still exist"
    assert restored_tag['title'] == TITLE_EXISTING, \
        f"Tag title should NOT be replaced! Expected '{TITLE_EXISTING}', " \
        f"got '{restored_tag['title']}'"
    
    # ========== 双重校验 2：磁盘文件状态 ==========
    # 验证 tag.json 内容没有变化
    tag_json_path = os.path.join(datastore_path, existing_uuid, 'tag.json')
    with open(tag_json_path, 'r') as f:
        disk_data = json.load(f)
    assert disk_data['title'] == TITLE_EXISTING, \
        f"tag.json on disk should NOT change! Expected '{TITLE_EXISTING}', " \
        f"got '{disk_data['title']}'"
    
    # 关键验证：额外文件不存在（目录没被复制）
    extra_file_path = os.path.join(datastore_path, existing_uuid, 'extra_file.txt')
    assert not os.path.exists(extra_file_path), \
        "extra_file.txt should NOT exist (directory was NOT copied due to skip logic)"
```

---

### 测试 T-004：Tag 已存在 + include_groups_replace=True → 替换覆盖

**场景：** Datastore 中已有一个 Tag，备份 zip 中包含相同 UUID 的 Tag，用户勾选替换。

**预期：**
- `restored_groups=1, skipped_groups=0`
- Tag title 被更新为 zip 中的值
- 目录被删除重建，额外文件出现

```python
def test_t004_tag_exists_replace(client, live_server, datastore_path):
    """T-004: Tag 已存在 + include_groups_replace=True → 替换覆盖"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据：预先创建 Tag ==========
    existing_uuid = datastore.add_tag(title=TITLE_EXISTING)
    
    # 记录恢复前的目录 mtime（用于验证目录被删除重建）
    tag_dir = os.path.join(datastore_path, existing_uuid)
    mtime_before = os.path.getmtime(tag_dir)
    import time
    time.sleep(0.1)  # 确保 mtime 有差异
    
    # 构造 zip：包含相同 UUID 的 Tag，title 不同 + 额外文件
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{existing_uuid}/tag.json", 
                   json.dumps({'title': TITLE_UPDATED_IN_BACKUP}))
        zf.writestr(f"{existing_uuid}/history.txt", "history from backup")
    test_zip.seek(0)
    
    # ========== 执行恢复（关键参数：include_groups_replace=True）==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,     # 关键：允许替换
        include_watches=True,
        include_watches_replace=True,
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_groups'] == 1, "Should restore 1 group (existing replaced)"
    assert result['skipped_groups'] == 0, "Should skip 0 groups"
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_tag = datastore.data['settings']['application']['tags'].get(existing_uuid)
    assert restored_tag is not None, "Tag should still exist"
    assert restored_tag['title'] == TITLE_UPDATED_IN_BACKUP, \
        f"Tag title SHOULD be replaced! Expected '{TITLE_UPDATED_IN_BACKUP}', " \
        f"got '{restored_tag['title']}'"
    
    # ========== 双重校验 2：磁盘文件状态 ==========
    # 验证 tag.json 内容已更新
    tag_json_path = os.path.join(datastore_path, existing_uuid, 'tag.json')
    with open(tag_json_path, 'r') as f:
        disk_data = json.load(f)
    assert disk_data['title'] == TITLE_UPDATED_IN_BACKUP, \
        f"tag.json on disk SHOULD change! Expected '{TITLE_UPDATED_IN_BACKUP}', " \
        f"got '{disk_data['title']}'"
    
    # 验证目录确实被删除重建（mtime 应该更新）
    mtime_after = os.path.getmtime(tag_dir)
    assert mtime_after > mtime_before, \
        "Tag directory mtime should be updated (directory was deleted and recreated)"
    
    # 验证额外文件已被复制
    history_path = os.path.join(datastore_path, existing_uuid, 'history.txt')
    assert os.path.exists(history_path), "history.txt should exist (directory was copied)"
```

---

### 测试 T-005：Tag 不存在（新增）+ include_groups_replace=False

**场景：** 空 Datastore，恢复一个新 Tag，replace=False（对新增无影响）。

```python
def test_t005_tag_new_no_replace(client, live_server, datastore_path):
    """T-005: Tag 不存在（新增）+ replace=False → 正常恢复"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    import uuid
    
    datastore = live_server.app.config['DATASTORE']
    
    # 构造 zip：包含 1 个新 Tag
    new_uuid = str(uuid.uuid4())
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{new_uuid}/tag.json", 
                   json.dumps({'title': TITLE_NEW_FROM_BACKUP}))
    test_zip.seek(0)
    
    # 执行恢复
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=False,  # 对新增无影响
        include_watches=True,
        include_watches_replace=True,
    )
    
    # 断言计数器
    assert result['restored_groups'] == 1
    assert result['skipped_groups'] == 0
    
    # 双重校验 1：内存对象状态
    restored_tag = datastore.data['settings']['application']['tags'].get(new_uuid)
    assert restored_tag is not None
    assert restored_tag['title'] == TITLE_NEW_FROM_BACKUP
    
    # 双重校验 2：磁盘文件状态
    tag_json_path = os.path.join(datastore_path, new_uuid, 'tag.json')
    assert os.path.exists(tag_json_path)
    with open(tag_json_path, 'r') as f:
        assert json.load(f)['title'] == TITLE_NEW_FROM_BACKUP
```

---

### 测试 T-006：Tag 不存在（新增）+ include_groups_replace=True

与 T-005 完全相同，仅 `include_groups_replace=True`。

**关键结论：** replace 参数仅对"已存在"的实体有影响，新增时无论 True/False 都会被恢复。

---

### 测试 T-007：仅含已存在 Tag + include_groups_replace=False

场景：zip 中只有已存在的 Tag，没有新 Tag。

**预期：** `restored_groups=0, skipped_groups=1`，Tag title 保持不变。

---

### 测试 T-008：仅含已存在 Tag + include_groups_replace=True

场景：zip 中只有已存在的 Tag，没有新 Tag。

**预期：** `restored_groups=1, skipped_groups=0`，Tag title 被更新。

---

## 3. Watch 分支测试用例（8 个可执行版本）

### 通用常量定义

```python
URL_EXISTING = "http://existing.example.com"
URL_NEW_FROM_BACKUP = "http://new-from-backup.example.com"
URL_UPDATED_IN_BACKUP = "http://updated-from-backup.example.com"
```

---

### 测试 W-001：include_watches=False → Watches 完全不恢复

（结构与 T-001 对称，可直接复制）

```python
def test_w001_include_watches_false(client, live_server, datastore_path):
    """W-001: include_watches=False → Watches 完全不恢复"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    import uuid
    
    datastore = live_server.app.config['DATASTORE']
    
    # 构造 zip：包含 1 个新 Watch
    new_uuid = str(uuid.uuid4())
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{new_uuid}/watch.json", 
                   json.dumps({'url': URL_NEW_FROM_BACKUP}))
    test_zip.seek(0)
    
    # 执行恢复
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=False,          # 关键开关关闭
        include_watches_replace=True,
    )
    
    # 断言计数器
    assert result['restored_watches'] == 0
    assert result['skipped_watches'] == 0
    
    # 双重校验 1：内存对象状态
    assert new_uuid not in datastore.data['watching']
    
    # 双重校验 2：磁盘文件状态
    watch_dir = os.path.join(datastore_path, new_uuid)
    assert not os.path.exists(watch_dir)
```

---

### 测试 W-002：include_watches=False（同 W-001，replace=True 不影响）

与 W-001 完全相同。

---

### 测试 W-003：Watch 已存在 + include_watches_replace=False → 跳过

（结构与 T-003 对称，可直接复制）

```python
def test_w003_watch_exists_no_replace(client, live_server, datastore_path):
    """W-003: Watch 已存在 + include_watches_replace=False → 跳过（不覆盖）"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据：预先创建 Watch ==========
    existing_uuid = datastore.add_watch(url=URL_EXISTING)
    
    # 构造 zip：包含相同 UUID 的 Watch，但 URL 不同 + 额外文件
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{existing_uuid}/watch.json", 
                   json.dumps({'url': URL_UPDATED_IN_BACKUP}))
        zf.writestr(f"{existing_uuid}/history.txt", "should NOT exist after skip")
    test_zip.seek(0)
    
    # ========== 执行恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=True,
        include_watches_replace=False,    # 关键：不替换
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_watches'] == 0
    assert result['skipped_watches'] == 1
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_watch = datastore.data['watching'].get(existing_uuid)
    assert restored_watch is not None
    assert restored_watch['url'] == URL_EXISTING, \
        f"Watch URL should NOT be replaced! Expected '{URL_EXISTING}', " \
        f"got '{restored_watch['url']}'"
    
    # ========== 双重校验 2：磁盘文件状态 ==========
    watch_json_path = os.path.join(datastore_path, existing_uuid, 'watch.json')
    with open(watch_json_path, 'r') as f:
        disk_data = json.load(f)
    assert disk_data['url'] == URL_EXISTING, \
        f"watch.json should NOT be overwritten on disk"
    
    # 关键验证：额外文件不存在
    history_path = os.path.join(datastore_path, existing_uuid, 'history.txt')
    assert not os.path.exists(history_path), \
        "history.txt should NOT exist (directory was NOT copied due to skip logic)"
```

---

### 测试 W-004：Watch 已存在 + include_watches_replace=True → 替换覆盖

（结构与 T-004 对称，可直接复制）

```python
def test_w004_watch_exists_replace(client, live_server, datastore_path):
    """W-004: Watch 已存在 + include_watches_replace=True → 替换覆盖"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据：预先创建 Watch ==========
    existing_uuid = datastore.add_watch(url=URL_EXISTING)
    
    # 记录恢复前的目录 mtime
    watch_dir = os.path.join(datastore_path, existing_uuid)
    mtime_before = os.path.getmtime(watch_dir)
    import time
    time.sleep(0.1)
    
    # 构造 zip：包含相同 UUID 的 Watch，URL 不同 + 额外文件
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{existing_uuid}/watch.json", 
                   json.dumps({'url': URL_UPDATED_IN_BACKUP}))
        zf.writestr(f"{existing_uuid}/history.txt", "history from backup")
        zf.writestr(f"{existing_uuid}/snapshot.txt", "snapshot from backup")
    test_zip.seek(0)
    
    # ========== 执行恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=True,
        include_watches_replace=True,     # 关键：允许替换
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_watches'] == 1
    assert result['skipped_watches'] == 0
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_watch = datastore.data['watching'].get(existing_uuid)
    assert restored_watch is not None
    assert restored_watch['url'] == URL_UPDATED_IN_BACKUP, \
        f"Watch URL SHOULD be replaced! Expected '{URL_UPDATED_IN_BACKUP}', " \
        f"got '{restored_watch['url']}'"
    
    # ========== 双重校验 2：磁盘文件状态 ==========
    watch_json_path = os.path.join(datastore_path, existing_uuid, 'watch.json')
    with open(watch_json_path, 'r') as f:
        disk_data = json.load(f)
    assert disk_data['url'] == URL_UPDATED_IN_BACKUP, \
        f"watch.json SHOULD be overwritten on disk"
    
    # 验证目录确实被删除重建
    mtime_after = os.path.getmtime(watch_dir)
    assert mtime_after > mtime_before, "Watch directory mtime should be updated"
    
    # 验证额外文件已被复制
    history_path = os.path.join(datastore_path, existing_uuid, 'history.txt')
    assert os.path.exists(history_path), "history.txt should be restored from backup"
    
    snapshot_path = os.path.join(datastore_path, existing_uuid, 'snapshot.txt')
    assert os.path.exists(snapshot_path), "snapshot.txt should be restored from backup"
```

---

### 测试 W-005 ~ W-008（与 T-005 ~ T-008 对称）

结构与 Tag 分支完全一致，此处略。

---

## 4. 混合场景测试用例（4 个核心组合）

### 测试 M-001：四开关全 False → 什么都不恢复

**预期：** 所有计数器 = 0，目录都不创建

```python
def test_m001_all_switches_false(client, live_server, datastore_path):
    """M-001: 四开关全 False → 什么都不恢复"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    import uuid
    
    datastore = live_server.app.config['DATASTORE']
    
    # 构造 zip：同时包含 Tag 和 Watch
    tag_uuid = str(uuid.uuid4())
    watch_uuid = str(uuid.uuid4())
    
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{tag_uuid}/tag.json", json.dumps({'title': 'Test Tag'}))
        zf.writestr(f"{watch_uuid}/watch.json", json.dumps({'url': 'http://test.com'}))
    test_zip.seek(0)
    
    # 执行恢复：所有开关关闭
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=False,
        include_groups_replace=False,
        include_watches=False,
        include_watches_replace=False,
    )
    
    # 所有计数器都应该是 0
    assert result['restored_groups'] == 0
    assert result['skipped_groups'] == 0
    assert result['restored_watches'] == 0
    assert result['skipped_watches'] == 0
    
    # 双重校验：内存和磁盘都不存在
    assert tag_uuid not in datastore.data['settings']['application']['tags']
    assert watch_uuid not in datastore.data['watching']
```

---

### 测试 M-004：Tag 已存在 + Watch 已存在 + 双 replace=False → 都跳过

**场景：** Datastore 中同时存在 1 个 Tag 和 1 个 Watch，备份中包含相同 UUID 的两者，都不勾选替换。

**预期：** 
- `restored_groups=0, skipped_groups=1`
- `restored_watches=0, skipped_watches=1`
- Tag title 和 Watch URL 都保持原值

```python
def test_m004_both_exist_both_no_replace(client, live_server, datastore_path):
    """M-004: Tag 已存在 + Watch 已存在 + 双 replace=False → 都跳过"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据：创建 Tag 和 Watch ==========
    existing_tag_uuid = datastore.add_tag(title=TITLE_EXISTING)
    existing_watch_uuid = datastore.add_watch(url=URL_EXISTING)
    
    # 构造 zip：两者都有更新
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{existing_tag_uuid}/tag.json", 
                   json.dumps({'title': TITLE_UPDATED_IN_BACKUP}))
        zf.writestr(f"{existing_watch_uuid}/watch.json", 
                   json.dumps({'url': URL_UPDATED_IN_BACKUP}))
    test_zip.seek(0)
    
    # 执行恢复：两个 replace 都关闭
    result = result = result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=False,  # Tag 不替换
        include_watches=True,
        include_watches_replace=False, # Watch 不替换
    )
    
    # 计数器断言
    result = result
    assert result['restored_groups'] == 0
    assert result['skipped_groups'] == 1
    assert result['restored_watches'] == 0
    assert result['skipped_watches'] == 1
    
    # 双重校验：两者都保持原值
    restored_tag = datastore.data['settings']['application']['tags'].get(existing_tag_uuid)
    assert restored_tag['title'] == TITLE_EXISTING
    
    restored_watch = datastore.data['watching'].get(existing_watch_uuid)
    assert restored_watch['url'] == URL_EXISTING
```

---

### 测试 M-005：Tag 已存在（替换） + Watch 已存在（不替换）

**场景：** 两者都已存在，Tag 勾选替换，Watch 不勾选。

**预期：**
- Tag 被更新，Watch 保持原值
- 计数器：`restored_groups=1, skipped_groups=0`
- 计数器：`restored_watches=0, skipped_watches=1`

---

### 测试 M-006：Tag 已存在（不替换） + Watch 已存在（替换）

与 M-005 相反，Tag 不替换，Watch 替换。

---

## 5. 同一 UUID 双文件冲突场景（8 个精确验证）

### 关键背景：if-elif 结构的影响

**代码位置（restore.py 第 89-127 行）：**

```python
if include_groups and os.path.exists(tag_json_path):
    # ... Tag 处理逻辑 ...
    # 分支内的 continue 会跳到下一个 UUID！
    continue  # ← 关键！Watch 分支永远不会执行！
elif include_watches and os.path.exists(watch_json_path):
    # ... Watch 处理逻辑 ...
    # ← 如果进入了 if 分支，这里永远不会执行！
```

**重要发现：** `shutil.copytree(entry.path, dst_dir)` 复制整个目录！
- 如果目录中同时有 tag.json 和 watch.json，**两个文件都会被复制到磁盘**
- 但**内存中只有 Tag 对象被创建**（因为进入了 Tag 分支，continue 跳过了 Watch 分支）

---

### 场景 C-001：双文件 + 双开关 True = Watch 静默丢失

**输入构造：** 同一个 UUID 目录下同时有 tag.json 和 watch.json

**调用参数：**
- `include_groups=True`
- `include_groups_replace=True`
- `include_watches=True`（关键：用户勾选了恢复 Watch！）
- `include_watches_replace=True`

**可执行测试代码：**

```python
def test_c001_both_files_both_true_watch_silent_drop(client, live_server, datastore_path):
    """C-001: 同一 UUID 双文件 + 双开关 True = Watch 静默丢失"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    import uuid
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 输入构造：同一个 UUID 双文件 ==========
    conflict_uuid = str(uuid.uuid4())
    
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        # 同一个 UUID 目录！
        zf.writestr(f"{conflict_uuid}/tag.json", 
                   json.dumps({'title': 'Conflict Tag Title'}))
        zf.writestr(f"{conflict_uuid}/watch.json", 
                   json.dumps({'url': 'http://conflict-watch.com'}))
    test_zip.seek(0)
    
    # ========== 执行恢复：两个开关都开启 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=True,          # 关键：用户勾选了恢复 Watch！
        include_watches_replace=True,
    )
    
    # ========== 断言计数器 ==========
    # Tag 被正常恢复
    assert result['restored_groups'] == 1, "Tag should be restored"
    assert result['skipped_groups'] == 0
    
    # ⚠️ BUG！Watch 被静默丢弃！计数器也是 0！
    assert result['restored_watches'] == 0, \
        "⚠️ Watch was SILENTLY DROPPED! restored_watches=0 but include_watches=True"
    assert result['skipped_watches'] == 0, \
        "⚠️ skipped_watches is also 0! No indication that anything was skipped!"
    
    # ========== 双重校验 1：内存对象状态 ==========
    # Tag 正常存在
    tag_in_memory = datastore.data['settings']['application']['tags'].get(conflict_uuid)
    assert tag_in_memory is not None, "Tag should exist in memory"
    assert tag_in_memory['title'] == 'Conflict Tag Title'
    
    # ⚠️ BUG！Watch 在内存中不存在！
    watch_in_memory = datastore.data['watching'].get(conflict_uuid)
    assert watch_in_memory is None, \
        "⚠️ Watch should NOT exist in memory! SILENTLY DROPPED!"
    
    # ========== 双重校验 2：磁盘文件状态 ==========
    # 验证目录存在（因为 Tag 分支执行了 copytree）
    uuid_dir = os.path.join(datastore_path, conflict_uuid)
    assert os.path.exists(uuid_dir), "UUID directory should exist"
    
    # tag.json 正常存在
    tag_json_path = os.path.join(uuid_dir, 'tag.json')
    assert os.path.exists(tag_json_path), "tag.json should exist on disk"
    
    # ⚠️ 发现！watch.json 也在磁盘上！（因为 copytree 复制了整个目录）
    watch_json_path = os.path.join(uuid_dir, 'watch.json')
    assert os.path.exists(watch_json_path), \
        "⚠️ watch.json DOES exist on disk! (copytree copied entire directory)"
    
    # 结果：磁盘上有 watch.json，但内存中没有 Watch 对象！
    # 下次 reload 时这个 watch.json 会被忽略（因为 UUID 在 tags 字典中已存在）
    # 或者可能导致其他未定义行为
```

---

### 场景 C-002：双文件 + Tag True + Watch False

与 C-001 相同，唯一区别是 `include_watches=False`。

**预期：** 行为与 C-001 相同（Tag 恢复，Watch 不恢复），但这是预期行为（用户本来就没要恢复 Watch）。

---

### 场景 C-003：双文件 + Tag False + Watch True = Watch 正常恢复

**输入：** 同 C-001（同一个 UUID 双文件）

**调用参数：**
- `include_groups=False`（关键：不恢复 Tag）
- `include_groups_replace=True`
- `include_watches=True`
- `include_watches_replace=True`

**预期：** 因为 if 条件不满足（include_groups=False），所以走到 elif 分支，Watch 被正常恢复。

```python
result = import_from_zip(...)
# 预期计数器：
result['restored_groups'] == 0
result['restored_watches'] == 1  # ✅ Watch 正常恢复！

# 双重校验：
# Tag 不在内存中，Watch 在内存中
# 磁盘上两个 JSON 都存在（因为 copytree 复制了整个目录）
```

---

### 场景 C-004：双文件 + 双开关 False = 都不恢复（预期）

计数器全部为 0，正常行为。

---

### 场景 C-005：双文件 + Tag 已存在 + replace=False + Watch True = 双重静默丢失

**最隐蔽的 BUG！**

**前置数据：** Datastore 中已有一个 Tag（相同 UUID）

**输入：** 同一个 UUID 双文件（tag.json + watch.json）

**调用参数：**
- `include_groups=True`
- `include_groups_replace=False`（关键：不替换，会触发 continue！）
- `include_watches=True`（用户勾选了恢复 Watch！）
- `include_watches_replace=True`

**预期结果（严重 BUG）：**

```python
result = import_from_zip(...)
result['restored_groups'] == 0
result['skipped_groups'] == 1  # ← 只有 Tag 跳过被计数了

# ⚠️ Watch 分支完全没有执行！
result['restored_watches'] == 0
result['skipped_watches'] == 0  # ← 连跳过计数都没有！

# 内存中：
# - Tag 保持原值不变
# - Watch 完全不存在

# 磁盘上：
# - 目录完全没被复制（因为跳过了 shutil.copytree）
# - 所以两个 JSON 都没有变化
```

**可执行测试代码：**

```python
def test_c005_tag_exists_no_replace_watch_true_double_silent_drop(client, live_server, datastore_path):
    """C-005: Tag 已存在 + replace=False + Watch True = Watch 被双重静默丢弃"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据：已有 Tag ==========
    existing_uuid = datastore.add_tag(title=TITLE_EXISTING)
    
    # 构造 zip：同一个 UUID 双文件
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{existing_uuid}/tag.json", 
                   json.dumps({'title': TITLE_UPDATED_IN_BACKUP}))
        zf.writestr(f"{existing_uuid}/watch.json", 
                   json.dumps({'url': 'http://watch-from-backup.com'}))
    test_zip.seek(0)
    
    # ========== 执行恢复：Tag 不替换 + Watch 恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=False,  # 关键：会触发 continue！
        include_watches=True,          # 关键：用户想要恢复 Watch！
        include_watches_replace=True,
    )
    
    # ========== 断言计数器 ==========
    result = result
    assert result['restored_groups'] == 0
    assert result['skipped_groups'] == 1, "Tag skip was counted correctly"
    
    # ⚠️ 严重 BUG！Watch 完全被忽略，连跳过计数都没有！
    assert result['restored_watches'] == 0, "Watch was NOT restored"
    assert result['skipped_watches'] == 0, \
        "⚠️ Watch skip was NOT counted! SILENT DROP of Watch!"
    
    # ========== 双重校验 ==========
    # Tag 保持原值
    restored_tag = datastore.data['settings']['application']['tags'].get(existing_uuid)
    assert restored_tag['title'] == TITLE_EXISTING, "Tag title should not change"
    
    # Watch 完全不存在
    watch = datastore.data['watching'].get(existing_uuid)
    assert watch is None, "Watch was SILENTLY dropped, completely ignored"
    
    # 磁盘上：watch.json 没有新增（因为目录没被复制）
    watch_json_path = os.path.join(datastore_path, existing_uuid, 'watch.json')
    assert not os.path.exists(watch_json_path), \
        "watch.json should NOT appear (directory was not copied)"
```

---

### 场景 C-006：双文件 + Tag 已存在 + replace=True + Watch False

与 C-001 类似，Tag 被替换，Watch 不恢复（因为 include_watches=False，属于预期行为）。

---

### 场景 C-007：双文件 + Tag 已存在 + replace=True + Watch True

**场景：** Tag 已存在，用户勾选替换，同时勾选恢复 Watch。

**行为：** Tag 被正常替换，Watch 继续被静默丢弃（同 C-001）。

**计数器：** `restored_groups=1, skipped_groups=0, restored_watches=0, skipped_watches=0`

---

### 场景 C-008：双文件 + Tag 不存在 + replace 任意 + Watch True

**场景：** Tag 不存在，用户勾选恢复 Tag 和 Watch。

**行为：** Tag 被恢复（新增），Watch 被静默丢弃（同 C-001）。

---

## 6. 关键发现总结

### 发现 1：Tag 优先级永远高于 Watch（if-elif 结构）

- 如果同一个 UUID 目录下同时有 tag.json 和 watch.json
- 只要 `include_groups=True`，Tag 分支会执行并执行 `continue`
- Watch 分支永远不会执行，哪怕 `include_watches=True`！

### 发现 2：静默丢失没有任何指示

- 计数器不会增加（restored_watches=0，skipped_watches=0）
- 没有 warning 日志
- 用户完全不知道自己的 Watch 备份被忽略了

### 发现 3：磁盘和内存状态可能不一致

- Tag 分支执行了 `shutil.copytree()` 会复制整个目录
- 因此 watch.json 可能出现在磁盘上（C-001）
- 但内存中没有 Watch 对象
- 下次 datastore reload 时可能产生未知行为

### 发现 4：Tag 跳过也会导致 Watch 静默丢失

- 最隐蔽的场景：Tag 已存在且 replace=False
- 代码执行 `continue` 跳到下一个 UUID
- Watch 分支完全没机会执行
- 连 `skipped_watches` 计数都没有！

---

### 修复建议

1. **添加冲突检测日志：** 在处理 UUID 前，先检查是否同时有 tag.json 和 watch.json，如果有则记录 warning 日志。

2. **重构分支结构：** 将 `if-elif` 改为两个独立的 `if`，但添加 UUID 命名空间检查（同一个 UUID 不能同时是 Tag 和 Watch）。

3. **改进计数器：** 即使 Watch 分支没执行，也应该有某种计数（如 `conflicting_uuids`）来指示有异常发生。

---

## 附录：核心代码位置速查

| 功能 | 文件 | 行号 | 代码片段 |
|------|------|------|---------|
| Tag 分支 if 条件 | backups/restore.py | 89 | `if include_groups and os.path.exists(tag_json_path):` |
| Tag 跳过 continue | backups/restore.py | 94 | `continue` |
| Tag 恢复计数 | backups/restore.py | 122 | `restored_groups += 1` |
| Watch 分支 elif 条件 | backups/restore.py | 127 | `elif include_watches and os.path.exists(watch_json_path):` |
| Watch 跳过 continue | backups/restore.py | 131 | `continue` |
| Watch 恢复计数 | backups/restore.py | 154 | `restored_watches += 1` |
| 目录复制（Tag 分支） | backups/restore.py | 110-114 | `shutil.copytree(entry.path, dst_dir)` |
| 目录复制（Watch 分支） | backups/restore.py | 147 | `shutil.copytree(entry.path, dst_dir)` |
| add_tag 真实签名 | store/__init__.py | 947 | `def add_tag(self, title):` |
| add_watch 真实签名 | store/__init__.py | 674 | `def add_watch(self, url, tag='', extras=None, ...):` |
