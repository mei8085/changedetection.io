# 备份还原时新旧数据存储合并分析报告

## 1. 概述

本文档分析了 changedetection.io 项目中备份还原时新旧数据存储的合并策略、冲突处理机制和处理顺序。

## 2. 核心数据结构

### 2.1 数据存储架构

系统采用 **UUID 目录 + JSON 文件** 的分布式存储结构：

- **Watch 数据**：每个监控项存储在 `{datastore_path}/{uuid}/watch.json`
- **Tag/Group 数据**：每个标签存储在 `{datastore_path}/{uuid}/tag.json`
- **全局设置**：存储在 `{datastore_path}/changedetection.json`
- **历史数据**：存储在各 UUID 目录下的 `history.txt`、`snapshot.txt` 等文件

### 2.2 内存数据结构

```python
datastore.data = {
    'watching': {  # UUID -> Watch 对象
        'uuid-1': <Watch object>,
        'uuid-2': <Watch object>
    },
    'settings': {
        'application': {
            'tags': {  # UUID -> Tag 对象
                'tag-uuid-1': <Tag object>,
                'tag-uuid-2': <Tag object>
            }
        }
    }
}
```

## 3. 备份文件结构

备份文件是一个 ZIP 压缩包，包含以下内容：

```
backup.zip/
├── changedetection.json          # 全局设置（可选）
├── url-watches.json              # 旧版格式（向后兼容，可选）
├── url-list.txt                  # URL 列表
├── url-list-with-tags.txt        # 带标签的 URL 列表
├── {watch-uuid}/
│   ├── watch.json               # Watch 配置
│   ├── history.txt              # 历史记录索引
│   ├── snapshot.txt             # 最近快照
│   └── ... (其他数据文件)
├── {tag-uuid}/
│   └── tag.json                 # Tag 配置
```

## 4. 合并算法

### 4.1 还原入口

还原功能由 `changedetectionio/blueprint/backups/restore.py` 中的 `import_from_zip` 函数实现（第 40-169 行）。

### 4.2 合并流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     备份还原流程                              │
├─────────────────────────────────────────────────────────────┤
│ 1. 读取 ZIP 文件并验证格式                                    │
│       ↓                                                      │
│ 2. 解压到临时目录                                              │
│       ↓                                                      │
│ 3. 扫描临时目录，识别 UUID 目录                                 │
│       ↓                                                      │
│ 4. 对每个 UUID 目录：                                          │
│    ├─ 是 Tag 目录？ ─→ 处理 Tag 合并                           │
│    │                       ↓                                 │
│    │                  冲突检查：                              │
│    │                  UUID 已存在？                           │
│    │                   ├─ 是 ─→ 检查 replace 标志              │
│    │                   │           ├─ True ─→ 覆盖            │
│    │                   │           └─ False ─→ 跳过           │
│    │                   └─ 否 ─→ 直接导入                       │
│    └─ 是 Watch 目录？ ─→ 处理 Watch 合并                       │
│                            ↓                                  │
│                       冲突检查：                              │
│                       UUID 已存在？                           │
│                        ├─ 是 ─→ 检查 replace 标志              │
│                        │           ├─ True ─→ 覆盖            │
│                        │           └─ False ─→ 跳过           │
│                        └─ 否 ─→ 直接导入                       │
│       ↓                                                      │
│ 5. 提交 datastore 设置（保存 changedetection.json）            │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 详细合并算法

#### 4.3.1 Tag（分组）合并

**核心逻辑**（restore.py:90-124）：

```python
# 伪代码
if include_groups and os.path.exists(tag_json_path):
    # 冲突检查
    if uuid in current_tags and not include_groups_replace:
        # 跳过现有 Tag
        skipped_groups += 1
        continue
    
    # 读取 Tag 数据
    tag_data = json.load(f)
    
    # 复制整个 UUID 目录（覆盖式）
    if os.path.exists(dst_dir):
        shutil.rmtree(dst_dir)  # 先删除旧目录
    shutil.copytree(entry.path, dst_dir)  # 复制新目录
    
    # 创建 Tag 对象并提交
    tag_obj = Tag.model(...)
    current_tags[uuid] = tag_obj
    tag_obj.commit()
```

**合并策略**：
- 基于 UUID 的精确匹配
- 覆盖式替换（删除旧目录 → 复制新目录）
- 保留备份中的所有相关文件（包括历史数据等）

#### 4.3.2 Watch（监控项）合并

**核心逻辑**（restore.py:127-155）：

```python
# 伪代码
if include_watches and os.path.exists(watch_json_path):
    # 冲突检查
    if uuid in current_watches and not include_watches_replace:
        # 跳过现有 Watch
        skipped_watches += 1
        continue
    
    # 读取 Watch 数据
    watch_data = json.load(f)
    
    # 复制整个 UUID 目录（覆盖式）
    if os.path.exists(dst_dir):
        shutil.rmtree(dst_dir)  # 先删除旧目录
    shutil.copytree(entry.path, dst_dir)  # 复制新目录
    
    # 重新水化 Watch 对象并提交
    watch_obj = datastore.rehydrate_entity(uuid, watch_data)
    current_watches[uuid] = watch_obj
    watch_obj.commit()
```

**合并策略**：
- 与 Tag 相同的 UUID 精确匹配机制
- 覆盖式目录替换
- 保留备份中的所有文件（watch.json、history.txt、snapshot.txt 等）

### 4.4 处理顺序

**处理顺序**由目录扫描顺序决定（restore.py:78-87）：

```python
for entry in os.scandir(tmpdir):
    if not entry.is_dir():
        continue
    
    uuid = entry.name
    if not _UUID_RE.match(uuid):
        continue
    
    # 先检查是否是 Tag
    if include_groups and os.path.exists(tag_json_path):
        # 处理 Tag
        ...
    # 再检查是否是 Watch（注意是 elif，不是独立的 if）
    elif include_watches and os.path.exists(watch_json_path):
        # 处理 Watch
        ...
```

**关键特点**：
1. **目录级顺序**：由 `os.scandir()` 返回的顺序决定（通常是文件系统的顺序）
2. **类型优先级**：Tag 的检查在 Watch 之前（使用 `elif`）
3. **互斥处理**：同一个 UUID 目录只能是 Tag 或 Watch，不能同时处理

## 5. 冲突处理策略

### 5.1 冲突检测机制

冲突检测基于 **UUID 精确匹配**：

| 场景 | 检测条件 |
|------|---------|
| Tag 冲突 | `uuid in datastore.data['settings']['application']['tags']` |
| Watch 冲突 | `uuid in datastore.data['watching']` |

### 5.2 冲突解决选项

用户在还原时可以通过表单选择冲突处理策略：

```python
class RestoreForm(Form):
    include_groups = BooleanField('Include groups', default=True)
    include_groups_replace_existing = BooleanField(
        'Replace existing groups of the same UUID', 
        default=True
    )
    include_watches = BooleanField('Include watches', default=True)
    include_watches_replace_existing = BooleanField(
        'Replace existing watches of the same UUID', 
        default=True
    )
```

### 5.3 四种冲突处理组合

#### 组合 1：include=True, replace=True（默认）

```python
# 行为：覆盖现有条目
if uuid in current_tags:  # 冲突检测
    if include_groups_replace:  # True
        # 执行覆盖
        shutil.rmtree(dst_dir)      # 删除旧目录
        shutil.copytree(...)        # 复制新目录
        current_tags[uuid] = tag_obj # 更新内存
```

**结果**：备份数据优先，覆盖当前数据

#### 组合 2：include=True, replace=False

```python
# 行为：跳过现有条目
if uuid in current_tags and not include_groups_replace:  # 冲突且不替换
    skipped_groups += 1
    continue  # 直接跳过，不做任何处理
```

**结果**：当前数据优先，保留现有数据

#### 组合 3：include=False

```python
# 行为：完全不处理该类型
if include_groups:  # False，整个分支跳过
    ...
```

**结果**：该类型数据完全不导入

#### 组合 4：include=True, replace=True（无冲突）

```python
# 行为：直接导入
if uuid in current_tags:  # False，无冲突
    # 跳过冲突检查，直接导入
    shutil.copytree(...)
    current_tags[uuid] = tag_obj
```

**结果**：直接导入新数据

### 5.4 冲突处理决策矩阵

| 条目类型 | 存在于备份 | 存在于当前 | include | replace | 结果 |
|---------|----------|----------|--------|--------|------|
| Tag | ✅ | ❌ | ✅ | 任意 | ✅ 导入 |
| Tag | ✅ | ✅ | ✅ | ✅ | 🔄 覆盖 |
| Tag | ✅ | ✅ | ✅ | ❌ | ⏭️ 跳过 |
| Tag | ✅ | 任意 | ❌ | 任意 | ❌ 忽略 |
| Watch | ✅ | ❌ | ✅ | 任意 | ✅ 导入 |
| Watch | ✅ | ✅ | ✅ | ✅ | 🔄 覆盖 |
| Watch | ✅ | ✅ | ✅ | ❌ | ⏭️ 跳过 |
| Watch | ✅ | 任意 | ❌ | 任意 | ❌ 忽略 |

## 6. 边界条件分析

### 6.1 文件系统边界

#### 6.1.1 Zip Slip 路径遍历防护（restore.py:71-75）

```python
for member in zf.infolist():
    member_dest = os.path.realpath(os.path.join(resolved_dest, member.filename))
    if not member_dest.startswith(resolved_dest + os.sep) and member_dest != resolved_dest:
        raise ValueError(f"Zip Slip path traversal detected: {member.filename!r}")
```

**保护机制**：
- 使用 `os.path.realpath()` 解析符号链接和相对路径
- 检查解压后的路径是否在临时目录范围内
- 发现路径遍历攻击直接抛出异常

#### 6.1.2 Zip Bomb 解压大小限制（restore.py:64-69）

```python
total_uncompressed = sum(m.file_size for m in zf.infolist())
if total_uncompressed > _MAX_DECOMPRESSED_BYTES:
    raise ValueError(
        f"Backup archive decompressed size ({total_uncompressed // (1024 * 1024)} MB) "
        f"exceeds the {_MAX_DECOMPRESSED_BYTES // (1024 * 1024)} MB limit"
    )
```

**限制参数**（环境变量可配置）：
- `MAX_RESTORE_UPLOAD_MB`：上传文件大小限制（默认 256MB）
- `MAX_RESTORE_DECOMPRESSED_MB`：解压后总大小限制（默认 1GB）

#### 6.1.3 文件大小验证

**Watch 文件限制**（file_saving_datastore.py:225-234）：
- 单个 watch.json 最大 10MB
- 超过限制的文件被视为损坏，跳过加载

**Tag 文件限制**（file_saving_datastore.py:348-358）：
- 单个 tag.json 最大 1MB
- 超过限制的文件被视为损坏，跳过加载

### 6.2 数据完整性边界

#### 6.2.1 JSON 格式错误处理

```python
try:
    with open(tag_json_path, 'r', encoding='utf-8') as f:
        tag_data = json.load(f)
except (json.JSONDecodeError, IOError) as e:
    logger.error(f"Restore: failed to read tag.json for {uuid}: {e}")
    continue  # 跳过此条目，继续处理其他
```

**策略**：单个文件损坏不影响整体还原，仅跳过该条目

#### 6.2.2 UUID 格式验证

```python
_UUID_RE = re.compile(
    r'^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$',
    re.IGNORECASE,
)

if not _UUID_RE.match(uuid):
    logger.warning(f"Restore: skipping non-UUID directory {uuid!r}")
    continue
```

**验证规则**：仅处理标准 UUID v4 格式的目录

### 6.3 并发边界

#### 6.3.1 还原线程管理

```python
# 检查是否有正在运行的还原
if any(t.is_alive() for t in restore_threads):
    flash("A restore is already running, check back in a few minutes", "error")
    return redirect(...)

# 启动新的还原线程
restore_thread = threading.Thread(
    target=import_from_zip,
    kwargs={...},
    daemon=True,
    name="BackupRestore"
)
restore_thread.start()
```

**策略**：
- 同一时间只允许一个还原操作
- 还原在后台线程执行，不阻塞请求
- 线程标记为 daemon，进程退出时自动终止

#### 6.3.2 文件操作原子性

```python
def save_json_atomic(file_path, data_dict, ...):
    # 1. 写入临时文件
    fd, temp_path = tempfile.mkstemp(suffix='.tmp', ...)
    os.write(fd, data)
    os.close(fd)
    
    # 2. 原子重命名
    os.replace(temp_path, file_path)
```

**保护**：使用临时文件 + 原子重命名确保写入操作的原子性

### 6.4 数据类型边界

#### 6.4.1 Tag 与 Watch 互斥

```python
# 使用 elif，确保同一目录只处理一种类型
if include_groups and os.path.exists(tag_json_path):
    # 处理 Tag
    ...
elif include_watches and os.path.exists(watch_json_path):
    # 处理 Watch
    ...
```

**规则**：
- 同一 UUID 目录不能同时包含 tag.json 和 watch.json
- 优先识别为 Tag（由于 `elif`）
- 实际业务中 Tag 和 Watch 应该有不同的 UUID

#### 6.4.2 目录复制边界

```python
# 复制前先删除目标目录
if os.path.exists(dst_dir):
    shutil.rmtree(dst_dir)
shutil.copytree(entry.path, dst_dir)
```

**影响**：
- 覆盖式复制，目标目录的所有内容被替换
- 包括 watch.json 之外的其他文件（history.txt、snapshot.txt 等）
- 这意味着历史数据也会被还原

## 7. 合并后的数据持久化

### 7.1 持久化流程

```
┌─────────────────────────────────────────────────────────────┐
│                      数据持久化流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 复制 UUID 目录到数据存储目录                                │
│     ├─ {tmp}/{uuid}/ → {datastore}/{uuid}/                   │
│     └─ 包括所有文件（watch.json, history.txt 等）             │
│                                                              │
│  2. 对象提交（针对 Tag 和 Watch）                               │
│     ├─ tag_obj.commit()                                      │
│     │   └─ 确保 tag.json 被正确写入                           │
│     └─ watch_obj.commit()                                    │
│         └─ 确保 watch.json 被正确写入                         │
│                                                              │
│  3. 全局设置提交                                              │
│     └─ datastore.commit()                                    │
│         └─ 保存 changedetection.json（包含 tags 引用）        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 双重存储机制

Tag 采用双重存储：
1. **文件存储**：`{uuid}/tag.json`
2. **设置存储**：`changedetection.json` 中的 `settings.application.tags`

Watch 采用单文件存储：
- **文件存储**：`{uuid}/watch.json`
- 内存中通过 `datastore.data['watching']` 访问

## 8. 与备份创建的对应关系

### 8.1 备份内容（backups/__init__.py:16-91）

```python
def create_backup(datastore_path, watches: dict, tags: dict = None):
    # 1. 添加全局设置
    if os.path.isfile(changedetection_json):
        zipObj.write(changedetection_json, arcname="changedetection.json")
    
    # 2. 添加 Tag 数据目录
    for uuid, tag in (tags or {}).items():
        for f in Path(tag.data_dir).glob('*'):
            zipObj.write(f, arcname=os.path.join(f.parts[-2], f.parts[-1]))
    
    # 3. 添加 Watch 数据目录
    for uuid, w in watches.items():
        for f in Path(w.data_dir).glob('*'):
            zipObj.write(f, arcname=os.path.join(f.parts[-2], f.parts[-1]))
```

**备份特点**：
- 备份每个 UUID 目录下的所有文件（不仅是 JSON）
- 保留完整的目录结构
- 便于完整还原（包括历史数据）

### 8.2 还原与备份的对称性

| 操作 | 处理内容 | 目录结构 |
|-----|---------|---------|
| 备份 | 遍历 UUID 目录，添加所有文件 | `{uuid}/` 目录完整保留 |
| 还原 | 解压 UUID 目录，复制所有文件 | 完整恢复 `{uuid}/` 目录 |

## 9. 潜在风险与注意事项

### 9.1 数据丢失风险

**场景**：当 `replace=True` 时，现有数据被完全覆盖

```python
# 风险：现有目录被删除，无法恢复
if os.path.exists(dst_dir):
    shutil.rmtree(dst_dir)  # 永久删除
shutil.copytree(entry.path, dst_dir)
```

**建议**：
- 还原前建议先创建当前数据的备份
- 使用 `replace=False` 测试还原效果

### 9.2 历史数据覆盖

**场景**：还原时历史数据（history.txt、snapshot.txt）也会被覆盖

**影响**：
- 如果备份较旧，会丢失备份后产生的历史记录
- 没有历史数据的增量合并机制

### 9.3 标签引用一致性

**场景**：Watch 中的 `tags` 字段引用 Tag UUID

**潜在问题**：
- 如果还原 Watch 但不还原对应的 Tag，会产生无效引用
- 系统需要处理无效的 Tag UUID 引用

### 9.4 线程安全

**场景**：还原在后台线程执行，主线程继续服务请求

**潜在问题**：
- 还原期间数据处于不一致状态
- 用户可能在还原未完成时操作数据

**建议**：
- 检查 `restore_running` 状态
- 考虑在还原期间暂停或限制某些操作

## 10. 测试覆盖分析

### 10.1 现有测试

测试文件：`changedetectionio/tests/test_backup.py`

#### test_backup_restore（第 122-202 行）
- 测试完整的备份-还原流程
- 验证 Watch 和 Tag 的正确恢复
- 测试 `replace=True` 的场景

#### test_backup_restore_zip_slip_rejected（第 205-226 行）
- 测试 Zip Slip 路径遍历防护
- 验证恶意 ZIP 被拒绝

#### test_backup_restore_zip_bomb_rejected（第 229-261 行）
- 测试 Zip Bomb 解压大小限制
- 验证过大的压缩包被拒绝

### 10.2 测试覆盖缺口

**未测试的场景**：
1. `replace=False` 的冲突处理
2. `include=False` 的类型过滤
3. 部分成功、部分失败的还原
4. 并发还原尝试
5. 损坏文件的跳过处理
6. UUID 格式错误的目录处理

## 11. 总结

### 11.1 核心合并策略

1. **UUID 精确匹配**：基于 UUID 进行条目匹配
2. **覆盖式合并**：冲突时选择覆盖或跳过，无增量合并
3. **目录级复制**：整个 UUID 目录被复制，保留所有文件
4. **用户可控**：通过表单选项控制冲突处理行为

### 11.2 处理顺序

1. 按文件系统顺序扫描 UUID 目录
2. 优先处理 Tag（先检查 tag.json）
3. 再处理 Watch（使用 elif）
4. 同一 UUID 目录只处理一种类型

### 11.3 边界条件保护

- ✅ Zip Slip 路径遍历防护
- ✅ Zip Bomb 解压大小限制
- ✅ 文件大小限制
- ✅ JSON 格式错误处理
- ✅ UUID 格式验证
- ✅ 单线程还原保护
- ⚠️ 数据丢失风险（依赖用户决策）

### 11.4 关键文件位置

| 功能 | 文件位置 | 关键行号 |
|-----|---------|---------|
| 还原逻辑 | `blueprint/backups/restore.py` | 40-169 |
| 冲突检测 | `blueprint/backups/restore.py` | 91-94, 128-131 |
| 目录复制 | `blueprint/backups/restore.py` | 111-114, 144-147 |
| 表单配置 | `blueprint/backups/restore.py` | 29-37 |
| 备份创建 | `blueprint/backups/__init__.py` | 16-91 |
| 原子保存 | `store/file_saving_datastore.py` | 36-176 |
| 数据加载 | `store/file_saving_datastore.py` | 211-448 |
