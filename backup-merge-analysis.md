# 备份恢复测试分析 - 可执行测试用例版

## 目录
1. [测试框架入口模式说明](#1-测试框架入口模式说明)
2. [Tag 分支测试用例（8 个）](#2-tag-分支测试用例8-个)
3. [Watch 分支测试用例（8 个）](#3-watch-分支测试用例8-个)
4. [混合场景测试用例（8 个）](#4-混合场景测试用例8-个)
5. [同一 UUID 双文件冲突场景（8 个）](#5-同一-uuid-双文件冲突场景8-个)
6. [关键发现与测试优先级建议](#6-关键发现与测试优先级建议)

---

## 1. 测试框架入口模式说明

### 1.1 两种测试入口模式

| 模式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **模式 A：直接调用 `import_from_zip()`** | 精确捕获返回值、异常；可以直接断言 4 个计数字段；速度快 | 不经过 Flask 路由层，不测试后台线程逻辑 | 核心业务分支覆盖、计数器验证、安全测试 |
| **模式 B：通过 `client.post()` 调用 REST API** | 端到端完整测试；覆盖路由、表单解析、后台线程 | 无法直接获取计数值（后台线程丢弃返回值）；需要 sleep 等待；只能验证最终状态 | 集成测试、happy path 验证 |

### 1.2 推荐的测试函数签名（参考 `test_backup_restore_zip_slip_rejected`）

```python
# 所有核心分支测试统一使用：模式 A（直接调用） + pytest 风格
def test_xxx(client, live_server, datastore_path):
    """测试说明"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    # 1. 准备数据
    datastore = live_server.app.config['DATASTORE']
    
    # 2. 构造测试 zip（参考 zip_slip 测试的写法）
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr("uuid-xxx/tag.json", '{"title": "xxx"}')
        zf.writestr("uuid-xxx/watch.json", '{"url": "http://xxx"}')
    test_zip.seek(0)
    
    # 3. 调用恢复函数（可以直接捕获返回值）
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True/False,
        include_groups_replace=True/False,
        include_watches=True/False,
        include_watches_replace=True/False,
    )
    
    # 4. 断言计数值（关键！）
    assert result['restored_groups'] == X, "restored_groups mismatch"
    assert result['skipped_groups'] == Y, "skipped_groups mismatch"
    assert result['restored_watches'] == Z, "restored_watches mismatch"
    assert result['skipped_watches'] == W, "skipped_watches mismatch"
    
    # 5. 双重校验：目录状态 + 对象状态
    # ...
```

---

## 2. Tag 分支测试用例（8 个）

### 通用约定

```python
# 所有测试共享的常量
TAG_UUID_EXISTING = "aaaaaaaa-aaaa-4aaa-aaaa-aaaaaaaaaaaa"
TAG_UUID_NEW = "bbbbbbbb-bbbb-4bbb-bbbb-bbbbbbbbbbbb"
TITLE_EXISTING = "Existing Tag"
TITLE_NEW_IN_ZIP = "New Tag from Backup"
TITLE_UPDATED_IN_ZIP = "Updated Tag Title"
```

---

### 测试 T-001：include_groups=False → Tags 完全不恢复

**场景：** zip 中有 Tag，但用户明确不勾选"恢复 Groups"

**前置数据：** datastore 为空

**调用参数：**
- `include_groups=False`
- `include_groups_replace=False` （不影响，因为 include_groups=False）
- `include_watches=True`
- `include_watches_replace=True`

**可执行测试代码：**
```python
def test_t001_include_groups_false(client, live_server, datastore_path):
    """T-001: include_groups=False → Tags 完全不恢复"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # 构造 zip：包含 1 个 Tag
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{TAG_UUID_NEW}/tag.json", 
                   json.dumps({"title": TITLE_NEW_IN_ZIP}))
    test_zip.seek(0)
    
    # 执行恢复
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=False,          # 关键！
        include_groups_replace=False,
        include_watches=True,
        include_watches_replace=True,
    )
    
    # 断言计数器
    assert result['restored_groups'] == 0, "Should not restore any groups"
    assert result['skipped_groups'] == 0, "Should not skip any groups"
    
    # 双重校验 1：内存对象状态
    assert TAG_UUID_NEW not in datastore.data['settings']['application']['tags'], \
        "Tag should not exist in memory"
    
    # 双重校验 2：文件系统目录状态
    tag_dir = os.path.join(datastore_path, TAG_UUID_NEW)
    assert not os.path.exists(tag_dir), \
        f"Tag directory should NOT be created: {tag_dir}"
```

---

### 测试 T-002：include_groups=False（同 T-001，replace=True 不影响）

场景与断言完全同 T-001，仅 `include_groups_replace=True`。

**关键断言：** replace=True 不改变任何结果，因为 if 分支根本不进入。

---

### 测试 T-003：Tag 已存在 + include_groups_replace=False → 跳过

**场景：** datastore 已有相同 UUID Tag，恢复时不勾选"替换"

**前置数据构造：**
```python
# 在恢复前预先创建一个 Tag
datastore.add_tag(title=TITLE_EXISTING, uuid=TAG_UUID_EXISTING)
```

**调用参数：**
- `include_groups=True`
- `include_groups_replace=False` （关键！不替换）
- `include_watches=True`
- `include_watches_replace=True`

**可执行测试代码：**
```python
def test_t003_tag_exists_no_replace(client, live_server, datastore_path):
    """T-003: Tag 已存在 + include_groups_replace=False → 跳过（不覆盖）"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据 ==========
    # 先创建已存在的 Tag
    tag_uuid = datastore.add_tag(title=TITLE_EXISTING)
    
    # 构造 zip：包含相同 UUID 的 Tag，但 title 不同
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{tag_uuid}/tag.json", 
                   json.dumps({"title": TITLE_UPDATED_IN_ZIP}))
    test_zip.seek(0)
    
    # ========== 执行恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=False,    # 关键！不替换
        include_watches=True,
        include_watches_replace=True,
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_groups'] == 0, "Should restore 0 groups (existing not replaced)"
    assert result['skipped_groups'] == 1, "Should skip 1 existing group"
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_tag = datastore.data['settings']['application']['tags'].get(tag_uuid)
    assert restored_tag is not None, "Tag should still exist"
    assert restored_tag['title'] == TITLE_EXISTING, \
        f"Tag title should NOT be replaced! Expected '{TITLE_EXISTING}', got '{restored_tag['title']}'"
    
    # ========== 双重校验 2：文件系统目录状态 ==========
    # 校验 tag.json 内容没有被覆盖
    tag_json_path = os.path.join(datastore_path, tag_uuid, 'tag.json')
    with open(tag_json_path, 'r') as f:
        file_content = json.load(f)
    assert file_content['title'] == TITLE_EXISTING, \
        f"tag.json should NOT be overwritten! Expected '{TITLE_EXISTING}', got '{file_content['title']}'"
    
    # 校验目录没有被删除重建（可选，用 mtime 检查）
    # 逻辑：如果目录被删除重建，mtime 会比恢复前更新
```

---

### 测试 T-004：Tag 已存在 + include_groups_replace=True → 替换覆盖

**场景：** datastore 已有相同 UUID Tag，恢复时勾选"替换"

**前置数据构造：** 同 T-003（先创建一个 Tag）

**调用参数：**
- `include_groups=True`
- `include_groups_replace=True` （关键！允许替换）

**可执行测试代码：**
```python
def test_t004_tag_exists_replace(client, live_server, datastore_path):
    """T-004: Tag 已存在 + include_groups_replace=True → 替换覆盖"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据 ==========
    # 先创建已存在的 Tag
    tag_uuid = datastore.add_tag(title=TITLE_EXISTING)
    
    # 记录恢复前的目录 mtime（用于验证目录被删除重建）
    tag_dir = os.path.join(datastore_path, tag_uuid)
    mtime_before = os.path.getmtime(tag_dir)
    time.sleep(0.1)  # 确保 mtime 有差异
    
    # 构造 zip：包含相同 UUID 的 Tag，title 不同
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{tag_uuid}/tag.json", 
                   json.dumps({"title": TITLE_UPDATED_IN_ZIP}))
    test_zip.seek(0)
    
    # ========== 执行恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,     # 关键！允许替换
        include_watches=True,
        include_watches_replace=True,
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_groups'] == 1, "Should restore 1 group (existing replaced)"
    assert result['skipped_groups'] == 0, "Should skip 0 groups"
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_tag = datastore.data['settings']['application']['tags'].get(tag_uuid)
    assert restored_tag is not None, "Tag should still exist"
    assert restored_tag['title'] == TITLE_UPDATED_IN_ZIP, \
        f"Tag title SHOULD be replaced! Expected '{TITLE_UPDATED_IN_ZIP}', got '{restored_tag['title']}'"
    
    # ========== 双重校验 2：文件系统目录状态 ==========
    # 校验 tag.json 内容已被覆盖
    tag_json_path = os.path.join(datastore_path, tag_uuid, 'tag.json')
    with open(tag_json_path, 'r') as f:
        file_content = json.load(f)
    assert file_content['title'] == TITLE_UPDATED_IN_ZIP, \
        f"tag.json SHOULD be overwritten! Expected '{TITLE_UPDATED_IN_ZIP}', got '{file_content['title']}'"
    
    # 校验目录确实被删除重建（mtime 应该更新）
    mtime_after = os.path.getmtime(tag_dir)
    assert mtime_after > mtime_before, \
        "Tag directory mtime should be updated (directory was deleted and recreated)"
```

---

### 测试 T-005：Tag 不存在（新增） + include_groups_replace=False

**场景：** datastore 为空，恢复 zip 中新增 Tag，replace=False（不影响，因为不存在）

**调用参数：**
- `include_groups=True`
- `include_groups_replace=False`
- `include_watches=True`
- `include_watches_replace=True`

**可执行测试代码：**
```python
def test_t005_tag_new_no_replace(client, live_server, datastore_path):
    """T-005: Tag 不存在（新增）+ replace=False → 正常恢复"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # 构造 zip：包含 1 个新 Tag
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{TAG_UUID_NEW}/tag.json", 
                   json.dumps({"title": TITLE_NEW_IN_ZIP}))
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
    assert result['restored_groups'] == 1, "Should restore 1 new group"
    assert result['skipped_groups'] == 0, "Should skip 0 groups"
    
    # 双重校验 1：内存对象状态
    restored_tag = datastore.data['settings']['application']['tags'].get(TAG_UUID_NEW)
    assert restored_tag is not None, "New tag should exist in memory"
    assert restored_tag['title'] == TITLE_NEW_IN_ZIP
    
    # 双重校验 2：文件系统目录状态
    tag_json_path = os.path.join(datastore_path, TAG_UUID_NEW, 'tag.json')
    assert os.path.exists(tag_json_path), "tag.json should be created on filesystem"
    with open(tag_json_path, 'r') as f:
        assert json.load(f)['title'] == TITLE_NEW_IN_ZIP
```

---

### 测试 T-006：Tag 不存在（新增） + include_groups_replace=True

与 T-005 完全相同，仅 `include_groups_replace=True`。结果完全一致。

**关键断言：** replace=True 对新增无影响，计数器与对象状态与 T-005 相同。

---

### 测试 T-007：仅含已存在 Tag + include_groups_replace=False

场景：zip 中只有已存在的 Tag，没有新 Tag。

**关键断言：**
- `restored_groups == 0`
- `skipped_groups == 1`
- Tag title 保持原值不变

---

### 测试 T-008：仅含已存在 Tag + include_groups_replace=True

场景：zip 中只有已存在的 Tag，没有新 Tag。

**关键断言：**
- `restored_groups == 1`
- `skipped_groups == 0`
- Tag title 已更新为 zip 中的值

---

## 3. Watch 分支测试用例（8 个）

### 通用约定

```python
WATCH_UUID_EXISTING = "cccccccc-cccc-4ccc-cccc-cccccccccccc"
WATCH_UUID_NEW = "dddddddd-dddd-4ddd-dddd-dddddddddddd"
URL_EXISTING = "http://existing.example.com"
URL_NEW_IN_ZIP = "http://new-from-backup.example.com"
URL_UPDATED_IN_ZIP = "http://updated-from-backup.example.com"
```

---

### 测试 W-001：include_watches=False → Watches 完全不恢复

（结构与 T-001 对称）

**可执行测试代码：**
```python
def test_w001_include_watches_false(client, live_server, datastore_path):
    """W-001: include_watches=False → Watches 完全不恢复"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # 构造 zip：包含 1 个 Watch
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{WATCH_UUID_NEW}/watch.json", 
                   json.dumps({"url": URL_NEW_IN_ZIP}))
    test_zip.seek(0)
    
    # 执行恢复
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=False,          # 关键！
        include_watches_replace=True,
    )
    
    # 断言计数器
    assert result['restored_watches'] == 0, "Should not restore any watches"
    assert result['skipped_watches'] == 0, "Should not skip any watches"
    
    # 双重校验 1：内存对象状态
    assert WATCH_UUID_NEW not in datastore.data['watching'], \
        "Watch should not exist in memory"
    
    # 双重校验 2：文件系统目录状态
    watch_dir = os.path.join(datastore_path, WATCH_UUID_NEW)
    assert not os.path.exists(watch_dir), \
        f"Watch directory should NOT be created: {watch_dir}"
```

---

### 测试 W-002：include_watches=False（同 W-001，replace=True 不影响）

与 W-001 对称，仅 `include_watches_replace=True`。结果完全一致。

---

### 测试 W-003：Watch 已存在 + include_watches_replace=False → 跳过

（结构与 T-003 对称）

**可执行测试代码：**
```python
def test_w003_watch_exists_no_replace(client, live_server, datastore_path):
    """W-003: Watch 已存在 + include_watches_replace=False → 跳过（不覆盖）"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据 ==========
    # 先创建已存在的 Watch
    watch_uuid = datastore.add_watch(url=URL_EXISTING)
    
    # 构造 zip：包含相同 UUID 的 Watch，但 URL 不同
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{watch_uuid}/watch.json", 
                   json.dumps({"url": URL_UPDATED_IN_ZIP}))
        # 可选：加入额外的 history.txt 等文件验证目录复制
        zf.writestr(f"{watch_uuid}/history.txt", "fake history from backup")
    test_zip.seek(0)
    
    # ========== 执行恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=True,
        include_watches_replace=False,    # 关键！不替换
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_watches'] == 0, "Should restore 0 watches (existing not replaced)"
    assert result['skipped_watches'] == 1, "Should skip 1 existing watch"
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_watch = datastore.data['watching'].get(watch_uuid)
    assert restored_watch is not None, "Watch should still exist in memory"
    assert restored_watch['url'] == URL_EXISTING, \
        f"Watch URL should NOT be replaced! Expected '{URL_EXISTING}', got '{restored_watch['url']}'"
    
    # ========== 双重校验 2：文件系统目录状态 ==========
    # 校验 watch.json 内容没有被覆盖
    watch_json_path = os.path.join(datastore_path, watch_uuid, 'watch.json')
    with open(watch_json_path, 'r') as f:
        file_content = json.load(f)
    assert file_content['url'] == URL_EXISTING, \
        f"watch.json should NOT be overwritten! Expected '{URL_EXISTING}', got '{file_content['url']}'"
    
    # 关键：备份中的额外文件（如 history.txt）不应该出现（因为目录根本没被复制）
    history_path = os.path.join(datastore_path, watch_uuid, 'history.txt')
    assert not os.path.exists(history_path), \
        "history.txt from backup should NOT exist (directory not copied due to skip logic)"
```

---

### 测试 W-004：Watch 已存在 + include_watches_replace=True → 替换覆盖

（结构与 T-004 对称）

**可执行测试代码：**
```python
def test_w004_watch_exists_replace(client, live_server, datastore_path):
    """W-004: Watch 已存在 + include_watches_replace=True → 替换覆盖"""
    from changedetectionio.blueprint.backups.restore import import_from_zip
    
    datastore = live_server.app.config['DATASTORE']
    
    # ========== 前置数据 ==========
    # 先创建已存在的 Watch
    watch_uuid = datastore.add_watch(url=URL_EXISTING)
    
    # 记录恢复前的目录 mtime
    watch_dir = os.path.join(datastore_path, watch_uuid)
    mtime_before = os.path.getmtime(watch_dir)
    time.sleep(0.1)
    
    # 构造 zip：包含相同 UUID 的 Watch，URL 不同 + 额外文件
    test_zip = io.BytesIO()
    with ZipFile(test_zip, 'w') as zf:
        zf.writestr(f"{watch_uuid}/watch.json", 
                   json.dumps({"url": URL_UPDATED_IN_ZIP}))
        # 加入额外的历史文件（用于验证目录被完整复制）
        zf.writestr(f"{watch_uuid}/history.txt", "backup-history-content")
        zf.writestr(f"{watch_uuid}/snapshot.txt", "backup-snapshot-content")
    test_zip.seek(0)
    
    # ========== 执行恢复 ==========
    result = import_from_zip(
        zip_stream=test_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=True,
        include_watches_replace=True,     # 关键！允许替换
    )
    
    # ========== 断言计数器 ==========
    assert result['restored_watches'] == 1, "Should restore 1 watch (existing replaced)"
    assert result['skipped_watches'] == 0, "Should skip 0 watches"
    
    # ========== 双重校验 1：内存对象状态 ==========
    restored_watch = datastore.data['watching'].get(watch_uuid)
    assert restored_watch is not None, "Watch should still exist in memory"
    assert restored_watch['url'] == URL_UPDATED_IN_ZIP, \
        f"Watch URL SHOULD be replaced! Expected '{URL_UPDATED_IN_ZIP}', got '{restored_watch['url']}'"
    
    # ========== 双重校验 2：文件系统目录状态 ==========
    # 校验 watch.json 内容已被覆盖
    watch_json_path = os.path.join(datastore_path, watch_uuid, 'watch.json')
    with open(watch_json_path, 'r') as f:
        file_content = json.load(f)
    assert file_content['url'] == URL_UPDATED_IN_ZIP, \
        f"watch.json SHOULD be overwritten! Expected '{URL_UPDATED_IN_ZIP}', got '{file_content['url']}'"
    
    # 校验目录确实被删除重建（mtime 应该更新）
    mtime_after = os.path.getmtime(watch_dir)
    assert mtime_after > mtime_before, \
        "Watch directory mtime should be updated (directory was deleted and recreated)"
    
    # 校验额外文件（history.txt, snapshot.txt）已被复制过来
    history_path = os.path.join(datastore_path, watch_uuid, 'history.txt')
    assert os.path.exists(history_path), "history.txt should be restored from backup"
    
    snapshot_path = os.path.join(datastore_path, watch_uuid, 'snapshot.txt')
    assert os.path.exists(snapshot_path), "snapshot.txt should be restored from backup"
    
    # 验证文件内容正确性
    with open(history_path, 'r') as f:
        assert f.read() == "backup-history-content", "history.txt content should match backup"
```

---

### 测试 W-005 ~ W-008（与 T-005 ~ T-008 对称）

结构与 Tag 分支完全一致，不赘述。

---

## 4. 混合场景测试用例（8 个）

### 测试 M-001：四开关全 False → 什么都不恢复

**调用参数：**
```python
include_groups=False,
include_groups_replace=False,
include_watches=False,
include_watches_replace=False,
```

**关键断言：**
- 所有计数器 = 0
- 所有目录都未创建

---

### 测试 M-004：Tag 已存在 + Watch 已存在 + 双 replace=False → 都跳过

**场景：** datastore 中已有 1 个 Tag + 1 个 Watch，备份 zip 中相同 UUID，不勾选任何替换选项。

**关键断言：**
- `restored_groups == 0, skipped_groups == 1`
- `restored_watches == 0, skipped_watches == 1`
- Tag title 和 Watch URL 均保持原值不变

---

### 测试 M-005：Tag 已存在（替换） + Watch 已存在（不替换）

**调用参数：**
```python
include_groups=True,
include_groups_replace=True,    # Tag 允许替换
include_watches=True,
include_watches_replace=False,  # Watch 不允许替换
```

**关键断言：**
- Tag title 被更新
- Watch URL 保持原值
- 计数器：`restored_groups=1, skipped_groups=0`
- 计数器：`restored_watches=0, skipped_watches=1`

---

## 5. 同一 UUID 双文件冲突场景（8 个）

### 核心风险点回顾

```python
# restore.py 第 89-131 行
if include_groups and os.path.exists(tag_json_path):
    # Tag 分支执行后 CONTINUE，Watch 分支永远不触发！
    # 注意：即使 Watch 文件存在、即使 include_watches=True！
    ...
    continue  # ← 关键：跳到下一个 UUID
elif include_watches and os.path.exists(watch_json_path):
    # ← 永远不会走到这里！
    ...
```

---

### 场景 C-001：双文件 + 双开关 True = Watch 静默丢失

**输入样例构造：**
```python
# 同一个 UUID 目录下同时存在两个文件！
CONFLICT_UUID = "eeeeeeee-eeee-4eee-eeee-eeeeeeeeeeee"

test_zip = io.BytesIO()
with ZipFile(test_zip, 'w') as zf:
    # 同一个 UUID 下！
    zf.writestr(f"{CONFLICT_UUID}/tag.json", 
               json.dumps({"title": "ConflictTag"}))
    zf.writestr(f"{CONFLICT_UUID}/watch.json", 
               json.dumps({"url": "http://conflict-watch.com"}))
test_zip.seek(0)
```

**调用参数：**
```python
include_groups=True,
include_groups_replace=True,
include_watches=True,          # 关键：用户勾选了恢复 Watch
include_watches_replace=True,
```

**预期计数结果：**
```python
assert result['restored_groups'] == 1, "Tag 被恢复"
assert result['skipped_groups'] == 0
assert result['restored_watches'] == 0, "⚠️ Watch 静默丢失！计数器也是 0！"
assert result['skipped_watches'] == 0, "⚠️ 连 skipped_watches 都不递增！"
```

**预期可观测日志：**
```
[SUCCESS] Restore: group 'ConflictTag' (eeeeeeee-eeee-4eee-eeee-eeeeeeeeeeee) restored

⚠️ 注意：没有任何 Watch 相关日志！没有警告！没有错误！完全静默！
```

**双重校验断言：**
```python
# 校验 1：Tag 正常存在
tag = datastore.data['settings']['application']['tags'].get(CONFLICT_UUID)
assert tag is not None, "Tag should be restored"
assert tag['title'] == "ConflictTag"

# 校验 2：Watch 不存在！（静默丢失）
watch = datastore.data['watching'].get(CONFLICT_UUID)
assert watch is None, "⚠️ Watch should NOT exist! SILENTLY DROPPED!"

# 校验 3：文件系统
tag_json_path = os.path.join(datastore_path, CONFLICT_UUID, 'tag.json')
assert os.path.exists(tag_json_path), "tag.json should exist"

watch_json_path = os.path.join(datastore_path, CONFLICT_UUID, 'watch.json')
assert not os.path.exists(watch_json_path), \
    "⚠️ watch.json should NOT exist! The entire zip directory was copied, " \
    "but since we went into Tag branch, does watch.json exist on disk?"
    # 🔴 这里可能有 BUG：目录被复制了，所以 watch.json 在磁盘上！
    # 但内存中没有 Watch 对象！
```

---

### 场景 C-002：双文件 + Tag 开关 True + Watch 开关 False

结果与 C-001 相同。唯一区别是用户本来就没要 Watch，不算 bug。

---

### 场景 C-003：双文件 + Tag 开关 False + Watch 开关 True = Watch 正常恢复

**输入样例：** 同 C-001（同一个 UUID 双文件）

**调用参数：**
```python
include_groups=False,       # 关键：不恢复 Tag
include_groups_replace=True,
include_watches=True,       # 但恢复 Watch
include_watches_replace=True,
```

**预期计数结果：**
```python
assert result['restored_groups'] == 0
assert result['skipped_groups'] == 0
assert result['restored_watches'] == 1, "Watch 正常恢复（因为 if 分支不进入，走到了 elif 分支）"
assert result['skipped_watches'] == 0
```

**预期可观测日志：**
```
[SUCCESS] Restore: watch 'http://conflict-watch.com' (eeeeeeee-eeee-4eee-eeee-eeeeeeeeeeee) restored
```

---

### 场景 C-004：双文件 + 双开关 False = 都不恢复（预期）

计数器全部为 0，无日志。

---

### 场景 C-001 + 已存在 + 不替换：双重静默丢失

**前置数据：** datastore 中已有 Tag（同 UUID）

**输入样例：** 同 C-001

**调用参数：**
```python
include_groups=True,
include_groups_replace=False,  # 关键：Tag 已存在且不替换 → CONTINUE
include_watches=True,          # 用户勾选了恢复 Watch！
include_watches_replace=True,
```

**预期结果（非常隐蔽的 BUG！）：**

```python
# 因为 Tag 分支触发了 CONTINUE！
assert result['restored_groups'] == 0
assert result['skipped_groups'] == 1  # ← 只有这个递增了

# 🔴 严重问题：
assert result['restored_watches'] == 0  # 预期：Watch 应该被恢复！
assert result['skipped_watches'] == 0   # 连跳过计数都没有！
```

**这是目前代码中最严重的逻辑缺陷！**

解释：
1. 代码先检查 tag.json 是否存在 → 是
2. 然后检查 include_groups=True → 是
3. 然后进入 Tag 分支逻辑
4. 发现 UUID 已存在且 replace=False → 执行 `continue`（第 94 行）
5. `continue` 跳到下一个 UUID 循环
6. **Watch 分支永远不被执行！哪怕 include_watches=True！**
7. 结果：不仅 Watch 没恢复，连 `skipped_watches` 都不递增！完全静默！

---

## 6. 关键发现与测试优先级建议

### 6.1 最高优先级（P0）必须立即补充的测试

1. **T-003：Tag 已存在 + 不替换**（核心合并逻辑）
2. **T-004：Tag 已存在 + 替换**（核心合并逻辑）
3. **W-003：Watch 已存在 + 不替换**（核心合并逻辑）
4. **W-004：Watch 已存在 + 替换**（核心合并逻辑）
5. **C-001：同一 UUID 双文件 Watch 静默丢失**（安全/数据完整性）
6. **C-001 + 已存在：双重静默丢失**（最隐蔽的 BUG）

### 6.2 次高优先级（P1）需要补充的测试

1. T-001：include_groups=False
2. W-001：include_watches=False
3. M-004：双已存在 + 双不替换
4. M-005：Tag 替换 + Watch 不替换交叉测试

### 6.3 发现的最严重逻辑缺陷

**缺陷 ID：RESTORE-CONTINUE-BRANCH-SKIP**

**描述：** 当同一个 UUID 目录下同时有 tag.json 和 watch.json 时：
- 如果 Tag 分支的任何条件触发了 `continue`（包括：已存在且不替换、JSON 解析失败等）
- 则 Watch 分支完全不执行，哪怕 `include_watches=True`
- 且 `skipped_watches` 计数器不递增
- 且没有任何日志警告
- 完全静默的数据丢失

**修复建议：**

选项 A：先检测冲突，再处理
```python
# 在第 89 行前加入：
has_tag = os.path.exists(tag_json_path)
has_watch = os.path.exists(watch_json_path)
if has_tag and has_watch:
    logger.warning(
        f"UUID {uuid} has BOTH tag.json and watch.json! This is an invalid backup. "
        f"Only Tag will be processed, Watch will be IGNORED. "
        f"include_watches={include_watches} but has NO effect!"
    )
```

选项 B：改为两个独立的 if（不推荐，因为 UUID 命名空间本应隔离）
```python
if include_groups and os.path.exists(tag_json_path):
    process_tag()  # 不要在这里 continue！
    
if include_watches and os.path.exists(watch_json_path):
    if uuid in current_tags:
        logger.error(f"UUID {uuid} conflict! Exists as Tag already, cannot restore Watch.")
    else:
        process_watch()
```

---

## 附录：测试文件覆盖清单

| 测试用例 | 优先级 | 目前是否存在 | 建议位置 |
|---------|--------|-------------|---------|
| T-001 | P1 | ❌ 缺失 | test_backup.py |
| T-003 | P0 | ❌ 缺失 | test_backup.py |
| T-004 | P0 | ❌ 缺失 | test_backup.py |
| W-001 | P1 | ❌ 缺失 | test_backup.py |
| W-003 | P0 | ❌ 缺失 | test_backup.py |
| W-004 | P0 | ❌ 缺失 | test_backup.py |
| M-004 | P1 | ❌ 缺失 | test_backup.py |
| M-005 | P1 | ❌ 缺失 | test_backup.py |
| C-001 | P0 | ❌ 缺失 | test_backup.py |
| C-001 + 已存在 | P0 | ❌ 缺失 | test_backup.py |
| Zip Slip | 安全 | ✅ 已有 | test_backup.py:205 |
| Zip Bomb | 安全 | ✅ 已有 | test_backup.py:229 |
| Full restore (happy path) | P2 | ✅ 已有 | test_backup.py:122 |

---

## 附录：核心代码位置速查（精确定位）

| 功能 | 文件 | 行号 | 代码片段 |
|------|------|------|---------|
| Tag 分支 if 条件 | backups/restore.py | 89 | `if include_groups and os.path.exists(tag_json_path):` |
| Tag 跳过判断 | backups/restore.py | 90-91 | `if uuid in current_tags and not include_groups_replace:` |
| Tag 跳过计数 | backups/restore.py | 92 | `skipped_groups += 1` |
| Tag 跳过 continue | backups/restore.py | 93 | `continue` |
| Tag 恢复计数 | backups/restore.py | 122 | `restored_groups += 1` |
| Watch 分支 elif 条件 | backups/restore.py | 127 | `elif include_watches and os.path.exists(watch_json_path):` |
| Watch 跳过判断 | backups/restore.py | 128-129 | `if uuid in current_watches and not include_watches_replace:` |
| Watch 跳过计数 | backups/restore.py | 130 | `skipped_watches += 1` |
| Watch 跳过 continue | backups/restore.py | 131 | `continue` |
| Watch 恢复计数 | backups/restore.py | 154 | `restored_watches += 1` |
| 目录删除（覆盖前） | backups/restore.py | 112-113, 145-146 | `if os.path.exists(dst_dir): shutil.rmtree(dst_dir)` |
| 目录复制 | backups/restore.py | 114, 147 | `shutil.copytree(entry.path, dst_dir)` |
