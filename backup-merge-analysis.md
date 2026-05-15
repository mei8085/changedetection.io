# 备份恢复与 Datastore 合并分析报告

## 1. 系统架构概述

### 1.1 数据存储架构 (FileSavingDataStore)

```
datastore/
├── changedetection.json          # 全局设置文件（不包含 watches/tags）
├── {uuid}/                       # 每个 watch/tag 的独立目录
│   ├── watch.json               # watch 配置（独立存储）
│   ├── tag.json                 # tag 配置（独立存储）
│   ├── history.txt              # 历史变更记录
│   ├── snapshot.txt             # 最新快照内容
│   ├── last-checksum.txt        # 上一次校验和
│   ├── last-screenshot.png      # 截图（如有）
│   └── ...                      # 其他资源文件
├── url-list.txt                 # URL 列表（备份时生成）
├── url-list-with-tags.txt       # 带标签的 URL 列表
└── changedetection-{version}.json  # 版本升级时的备份
```

### 1.2 核心类继承关系

```
DataStore (ABC 抽象基类)
    ↓
FileSavingDataStore (文件存储实现)
    + save_json_atomic()         # 原子 JSON 写入
    + load_all_watches()         # 加载所有 watches
    + load_all_tags()            # 加载所有 tags
    ↓
DatastoreUpdatesMixin (Schema 更新)
    + run_updates()              # 执行 schema 升级
    ↓
ChangeDetectionStore (主存储类)
    + add_watch()
    + delete()
    + commit()
    + rehydrate_entity()
```

---

## 2. 备份功能实现分析

### 2.1 备份核心流程 (`backups/__init__.py:create_backup()`)

```python
def create_backup(datastore_path, watches: dict, tags: dict = None):
    # 1. 创建临时 zip 文件
    with zipfile.ZipFile(tmp_path, "w", compression=zipfile.ZIP_DEFLATED, compresslevel=8):
        
        # 2. 添加全局设置文件
        - changedetection.json  (新格式)
        - url-watches.json      (旧格式，向后兼容)
        
        # 3. 添加所有 tags 数据目录
        for uuid, tag in tags.items():
            for f in Path(tag.data_dir).glob('*'):
                zipObj.write(f, arcname=os.path.join(f.parts[-2], f.parts[-1]))
        
        # 4. 添加所有 watches 数据目录
        for uuid, w in watches.items():
            for f in Path(w.data_dir).glob('*'):
                zipObj.write(f, arcname=os.path.join(f.parts[-2], f.parts[-1]))
        
        # 5. 生成辅助文件
        - url-list.txt              # 纯 URL 列表
        - url-list-with-tags.txt    # 带标签的 URL 列表
    
    # 6. 原子重命名为最终备份文件
    os.rename(tmp_path, final_path)
```

### 2.2 备份触发与管理

| 功能 | 实现细节 |
|------|---------|
| **手动触发** | `/backups/request-backup` 路由，后台线程执行 |
| **自动备份** | 版本升级时自动备份 (`save_version_copy_json_db()`) |
| **备份列表** | 扫描 datastore 目录中 `changedetection-backup-*.zip` 文件 |
| **数量限制** | `MAX_NUMBER_BACKUPS` 环境变量控制，默认 100 |
| **下载支持** | 支持下载指定版本或最新版本 (`/backups/download/latest`) |

### 2.3 备份安全特性

- **原子写入**：先写入临时文件，完成后重命名，避免中途损坏
- **后台线程**：备份在独立线程执行，不阻塞 Web 请求
- **排除敏感文件**：`secret.txt` 等敏感文件不包含在备份中

---

## 3. 恢复功能实现分析

### 3.1 恢复核心流程 (`backups/restore.py:import_from_zip()`)

```python
def import_from_zip(zip_stream, datastore, include_groups, include_groups_replace, include_watches, include_watches_replace):
    """
    从 zip 备份文件恢复数据
    
    流程:
    1. 安全检查 (大小限制、路径遍历防护)
    2. 解压到临时目录
    3. 扫描 UUID 目录，识别 tag/watch
    4. 复制目录到 datastore
    5. 重新水化实体对象
    6. 提交更改
    """
```

### 3.1 恢复安全机制

| 安全措施 | 实现细节 | 位置 |
|---------|---------|------|
| **上传大小限制** | `MAX_RESTORE_UPLOAD_MB`，默认 256MB | `restore.py:19` |
| **解压大小限制** | `MAX_RESTORE_DECOMPRESSED_MB`，默认 1GB | `restore.py:21` |
| **Zip Slip 防护** | 检查所有成员路径，防止路径遍历攻击 | `restore.py:71-74` |
| **文件类型验证** | 仅接受 `.zip` 文件 | `restore.py:199-201` |
| **后台线程执行** | 恢复操作在后台线程执行，避免阻塞 | `restore.py:229-244` |

### 3.2 合并策略

恢复过程采用**选择性合并**策略，支持：

```
恢复选项:
├─ 恢复 Groups (Tags)
│   └─ 替换已存在的同 UUID Groups
└─ 恢复 Watches
    └─ 替换已存在的同 UUID Watches
```

### 3.3 恢复流程 (`restore.py:import_from_zip()`)

```
1. 读取上传的 ZIP 文件流
   ↓
2. 安全校验
   ├─ 验证 zip 文件格式有效性
   ├─ 检查解压后总大小 (防止 zip bomb)
   └─ 检查路径遍历攻击 (Zip Slip)
   ↓
3. 解压到临时目录
   ↓
4. 扫描 UUID 目录
   ├─ 对于每个 UUID 目录
   │   ├─ 检查是否存在 tag.json → Tag 恢复流程
   │   └─ 检查是否存在 watch.json → Watch 恢复流程
   ├─ 检查是否已存在
   │   ├─ 存在且允许替换 → 删除旧目录，复制新目录
   │   ├─ 存在但不允许替换 → 跳过
   │   └─ 不存在 → 直接复制目录
   └─ 重新水化 (rehydrate) 为对象
   ↓
5. 提交更改到 changedetection.json
```

### 3.4 关键代码段分析

**安全检查：**
```python
# 解压大小限制检查
total_uncompressed = sum(m.file_size for m in zf.infolist())
if total_uncompressed > _MAX_DECOMPRESSED_BYTES:
    raise ValueError(...)

# Zip Slip 路径遍历防护
for member in zf.infolist():
    member_dest = os.path.realpath(os.path.join(resolved_dest, member.filename))
    if not member_dest.startswith(resolved_dest + os.sep) and member_dest != resolved_dest:
        raise ValueError(...)
```

### 3.5 实体重新水化 (Rehydration)

恢复后，从文件重建为内存对象的过程：

```python
# Tags 重新水化
tag_obj = Tag.model(
    datastore_path=datastore.datastore_path,
    __datastore=datastore.data,
    default=tag_data  # 从 tag.json 读取
)

# Watches 重新水化
watch_obj = datastore.rehydrate_entity(uuid, watch_data)
# 内部调用 get_custom_watch_obj_for_processor()
# 根据 processor 类型创建对应 Watch 子类
```

---

## 4. 恢复操作路由

| 功能 | 路由 | 说明 |
|------|------|------|
| **恢复页面** | `GET /backups/restore` | 显示恢复选项表单 |
| **执行恢复** | `POST /backups/restore/start` | 上传 zip 并启动后台恢复线程 |

### 4.1 恢复表单选项

```
恢复选项:
├─ 📁 选择备份 ZIP 文件
├─ ☐ 包含 Groups (Tags)
│   └─ ☐ 替换已存在的同 UUID Groups
├─ ☐ 包含 Watches
│   └─ ☐ 替换已存在的同 UUID Watches
└─ [恢复备份] 按钮
```

---

## 5. 测试覆盖分析

### 5.1 备份功能测试 (`test_backup.py`)

| 测试用例 | 覆盖内容 | 位置 |
|---------|---------|------|
| `test_backup` | 完整备份流程，验证 zip 内容、格式 | `test_backup.py:12-81` |
| `test_watch_data_package_download` | 单个 watch 数据打包下载 | `test_backup.py:83-119` |
| `test_backup_restore` | 完整备份-恢复闭环测试，验证 watch/tag 正确恢复 | `test_backup.py:122-202` |
| `test_backup_restore_zip_slip_rejected` | Zip Slip 路径遍历攻击防护 | `test_backup.py:205-226` |
| `test_backup_restore_zip_bomb_rejected` | Zip Bomb 压缩炸弹攻击防护 | `test_backup.py:229-261` |

### 5.2 关键测试流程

**完整备份恢复测试流程：**

```python
# 1. 设置测试数据
uuid = datastore.add_watch(url=test_url)
tag_uuid = datastore.add_tag(title="Test Tag")

# 2. 创建备份
client.get(url_for("backups.request_backup"))

# 3. 下载备份
res = client.get(url_for("backups.download_backup", filename="latest"))
zip_data = res.data

# 4. 清空现有数据
datastore.delete('all')

# 5. 执行恢复
client.post(
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

# 6. 验证恢复结果
assert restored_watch['url'] == test_url
assert restored_tag['title'] == "Test Tag"
```

---

## 6. 核心安全机制

### 6.1 原子写入机制 (`file_saving_datastore.py:save_json_atomic()`)

```
原子写入流程:
1. 创建临时文件 (tempfile.mkstemp)
2. 写入数据到临时文件
3. 可选：fsync 强制刷盘 (FORCE_FSYNC_DATA_IS_CRITICAL)
4. 原子重命名临时文件 → 目标文件 (os.replace)
5. 可选：目录 fsync (仅新文件)

优势:
- 防止中途写入损坏
- 兼容 NFS/NAS 网络存储
- 崩溃后数据一致性
```

### 6.2 线程安全

```python
# 使用 threading.Lock 保护关键操作
with self.lock:
    # 修改数据结构
    self.__data['watching'][uuid].update(update_obj)
```

### 3.6 数据合并关键逻辑

恢复时的核心合并逻辑：

```python
# 对于每个 UUID 目录
for entry in os.scandir(tmpdir):
    if not entry.is_dir():
        continue
    uuid = entry.name
    
    # --- Tags (Groups) 恢复 ---
    if include_groups and os.path.exists(tag_json_path):
        if uuid in current_tags and not include_groups_replace:
            skipped_groups += 1
            continue
        
        # 删除旧目录（如果存在），复制新目录
        dst_dir = os.path.join(datastore.datastore_path, uuid)
        if os.path.exists(dst_dir):
            shutil.rmtree(dst_dir)
        shutil.copytree(entry.path, dst_dir)
        
        # 重新水化 tag 对象
        tag_obj = Tag.model(...)
        current_tags[uuid] = tag_obj
        tag_obj.commit()
    
    # --- Watches 恢复 ---
    elif include_watches and os.path.exists(watch_json_path):
        if uuid in current_watches and not include_watches_replace:
            skipped_watches += 1
            continue
        
        # 删除旧目录（如果存在），复制新目录
        dst_dir = os.path.join(datastore.datastore_path, uuid)
        if os.path.exists(dst_dir):
            shutil.rmtree(dst_dir)
        shutil.copytree(entry.path, dst_dir)
        
        # 重新水化 watch 对象
        watch_obj = datastore.rehydrate_entity(uuid, watch_data)
        current_watches[uuid] = watch_obj
        watch_obj.commit()

# 最后提交全局设置
datastore.commit()
```

---

## 4. 备份与恢复关键代码位置

| 功能 | 文件位置 | 核心函数/方法 |
|------|---------|-------------|
| **备份创建** | `backups/__init__.py` | `create_backup()` |
| **备份下载** | `backups/__init__.py` | `download_backup()` |
| **恢复流程** | `backups/restore.py` | `import_from_zip()` |
| **恢复路由** | `backups/restore.py` | `construct_restore_blueprint()` |
| **数据合并逻辑** | `backups/restore.py` | `import_from_zip()` |
| **原子写入** | `store/file_saving_datastore.py` | `save_json_atomic()` |
| **重新水化** | `store/__init__.py` | `rehydrate_entity()` |
| **实体保存** | `model/Watch.py` | `commit()` |

---

## 5. 数据合并流程详解

### 5.1 恢复时的实体识别

```
UUID 目录识别:
├─ 存在 tag.json → 识别为 Tag (Group)
├─ 存在 watch.json → 识别为 Watch
└─ 其他 → 忽略
```

### 5.2 合并策略选项

恢复时提供4个控制选项，实现灵活的数据合并：

```python
# 恢复选项
include_groups = True/False                    # 是否恢复 Tags
include_groups_replace = True/False            # 是否替换已存在的 Tag
include_watches = True/False                   # 是否恢复 Watches
include_watches_replace = True/False           # 是否替换已存在的 Watch
```

### 5.3 目录替换机制

```python
# 关键代码: 目录替换逻辑
dst_dir = os.path.join(datastore.datastore_path, uuid)
if os.path.exists(dst_dir):
    shutil.rmtree(dst_dir)  # 先删除整个旧目录
shutil.copytree(entry.path, dst_dir)  # 再复制新目录
```

### 5.4 重新水化 (Rehydration)

```python
# Tags 重新水化
tag_obj = Tag.model(
    datastore_path=datastore.datastore_path,
    __datastore=datastore.data,
    default=tag_data  # 从 tag.json 读取的数据
)
current_tags[uuid] = tag_obj
tag_obj.commit()  # 写入 tag.json

# Watches 重新水化
watch_obj = datastore.rehydrate_entity(uuid, watch_data)
# 根据 processor 类型选择对应的 Watch 子类
# 如: text_json_diff, restock_diff 等
current_watches[uuid] = watch_obj
watch_obj.commit()  # 写入 watch.json
```

### 3.7 安全机制

| 安全措施 | 实现 | 说明 |
|---------|------|------|
| **文件类型限制** | `FileAllowed(['zip'])` | WTForms 验证仅接受 zip 文件 | `restore.py:30-32` |
| **上传大小限制** | `MAX_RESTORE_UPLOAD_MB` | 默认 256MB，防止超大文件上传 | `restore.py:19` |
| **解压大小限制** | `MAX_RESTORE_DECOMPRESSED_MB` | 默认 1GB，防止 zip bomb | `restore.py:21` |
| **路径遍历防护** | `os.path.realpath()` 检查 | 防止 Zip Slip 攻击 | `restore.py:71-74` |
| **线程安全** | `threading.Thread` + 状态检查 | 后台执行，防止阻塞 | `restore.py:187-246` |

### 3.8 备份文件结构

```
changedetection-backup-{timestamp}.zip
├── changedetection.json          # 全局设置
├── {watch-uuid}/                 # Watch 目录
│   ├── watch.json               # Watch 配置
│   ├── history.txt              # 历史记录
│   ├── snapshot.txt             # 快照
│   └── ...                      # 其他资源
├── {tag-uuid}/                   # Tag 目录
│   └── tag.json                 # Tag 配置
├── url-list.txt                  # URL 列表
└── url-list-with-tags.txt        # URL + 标签列表
```

---

## 6. 与 Datastore 的交互点

### 6.1 备份时 Datastore 访问

```python
# 1. 从 datastore 获取 watches 字典
watches = datastore.data.get("watching")

# 2. 从 datastore 获取 tags 字典
tags = datastore.data['settings']['application'].get('tags', {})

# 3. 遍历每个 watch 的 data_dir 添加到 zip
for uuid, w in watches.items():
    for f in Path(w.data_dir).glob('*'):
        zipObj.write(f, arcname=os.path.join(f.parts[-2], f.parts[-1]))
```

### 6.2 恢复时 Datastore 交互

```python
# 1. 获取当前 datastore 中的 tags 和 watches
current_tags = datastore.data['settings']['application'].get('tags', {})
current_watches = datastore.data['watching']

# 2. 恢复后更新 datastore 字典
current_tags[uuid] = tag_obj
current_watches[uuid] = watch_obj

# 3. 提交全局设置
datastore.commit()
```

---

## 7. 关键安全机制

### 7.1 原子写入机制 (`file_saving_datastore.py:save_json_atomic()`)

```
原子写入流程:
1. 创建临时文件 (tempfile.mkstemp)
2. 写入数据到临时文件
3. 可选：fsync 强制刷盘 (FORCE_FSYNC_DATA_IS_CRITICAL)
4. 原子重命名临时文件 → 目标文件 (os.replace)
5. 可选：目录 fsync (仅新文件)

优势:
- 防止中途写入损坏
- 兼容 NFS/NAS 网络存储
- 崩溃后数据一致性
```

### 7.2 Zip Bomb 防护

```python
# 检查所有文件的解压后总大小
total_uncompressed = sum(m.file_size for m in zf.infolist())
if total_uncompressed > _MAX_DECOMPRESSED_BYTES:
    raise ValueError(
        f"Backup archive decompressed size ({total_uncompressed // (1024 * 1024)} MB) "
        f"exceeds the {_MAX_DECOMPRESSED_BYTES // (1024 * 1024)} MB limit"
    )
```

### 7.3 Zip Slip 路径遍历防护

```python
# 解析真实路径，防止 ../ 等路径遍历
resolved_dest = os.path.realpath(tmpdir)
for member in zf.infolist():
    member_dest = os.path.realpath(os.path.join(resolved_dest, member.filename))
    if not member_dest.startswith(resolved_dest + os.sep) and member_dest != resolved_dest:
        raise ValueError(f"Zip Slip path traversal detected in backup archive: {member.filename!r}")
```

---

## 8. 测试覆盖分析

### 8.1 备份恢复测试流程

```python
# 1. 准备测试数据
uuid = datastore.add_watch(url=test_url)
tag_uuid = datastore.add_tag(title="Test Tag")

# 2. 创建备份
client.get(url_for("backups.request_backup"))
time.sleep(4)  # 等待备份线程完成

# 3. 下载备份
res = client.get(url_for("backups.download_backup", filename="latest"))
zip_data = res.data

# 4. 清空数据
datastore.delete('all')

# 5. 执行恢复
client.post(
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
time.sleep(2)  # 等待恢复线程完成

# 6. 验证恢复结果
restored_watch = datastore.data['watching'].get(uuid)
assert restored_watch is not None
assert restored_watch['url'] == test_url
assert isinstance(restored_watch, Watch.model)
```

### 8.2 安全防护测试

### 3.9 与 Datastore 的交互点

```python
# 备份时访问 Datastore
watches = datastore.data.get("watching")
tags = datastore.data['settings']['application'].get('tags', {})

# 恢复时更新 Datastore
current_tags = datastore.data['settings']['application'].get('tags', {})
current_watches = datastore.data['watching']
# 更新后提交
datastore.commit()
```

---

## 9. 已知限制与潜在优化

### 9.1 当前限制

| 限制 | 说明 |
|------|------|
| **UUID 冲突处理** | 基于 UUID 精确匹配，不支持智能合并 |
| **增量恢复** | 不支持，每次都是完整覆盖或跳过 |
| **冲突提示** | 仅计数，不显示具体冲突项 |
| **大备份处理** | 整个 zip 读入内存，超大型备份可能内存不足 |

### 9.2 优化建议

**1. 内存优化 - 流式处理大备份**
```python
# 当前实现: 整个 zip 读入内存
raw = zip_file.read(_MAX_UPLOAD_BYTES + 1)
zip_bytes = io.BytesIO(raw)

# 优化建议: 使用临时文件存储
with tempfile.NamedTemporaryFile(suffix='.zip', delete=False) as tmp:
    shutil.copyfileobj(zip_file, tmp)
# 然后从临时文件读取
```

**2. 智能合并 - 基于时间戳的冲突解决**
```python
# 建议添加: 比较新旧实体的时间戳，选择较新的版本
if uuid in current_watches:
    existing_time = current_watches[uuid].get('date_updated', 0)
    new_time = watch_data.get('date_updated', 0)
    if new_time > existing_time:
        # 执行替换
        pass
```

---

## 10. 关键代码位置汇总

| 功能模块 | 文件路径 | 核心函数/方法 |
|---------|---------|-------------|
| **备份创建** | `backups/__init__.py` | `create_backup()` |
| **备份下载** | `backups/__init__.py` | `download_backup()` |
| **恢复流程** | `backups/restore.py` | `import_from_zip()` |
| **恢复路由** | `backups/restore.py` | `construct_restore_blueprint()` |
| **数据合并逻辑** | `backups/restore.py` | `import_from_zip()` |
| **原子写入** | `store/file_saving_datastore.py` | `save_json_atomic()` |
| **重新水化** | `store/__init__.py` | `rehydrate_entity()` |
| **实体保存** | `model/Watch.py` | `commit()` |
| **测试用例** | `tests/test_backup.py` | 所有备份恢复相关测试 |

---

## 总结

### 备份恢复与 Datastore 合并的核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        备份恢复系统架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │  备份模块    │────▶│  ZIP 打包    │────▶│  下载/存储   │   │
│  │ backups/     │     │ create_backup│     │  线程安全    │   │
│  └──────────────┘     └──────────────┘     └──────────────┘   │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Datastore 数据存储                    │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  changedetection.json  +  {uuid}/ 目录结构                │   │
│  │  ├─ settings                                           │   │
│  │  ├─ watches  ──▶  {uuid}/watch.json                    │   │
│  │  └─ tags     ──▶  {uuid}/tag.json                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │  恢复模块    │◀────│  ZIP 解压    │◀────│  上传验证    │   │
│  │ restore/     │     │import_from_zip│     │  安全防护    │   │
│  └──────────────┘     └──────────────┘     └──────────────┘   │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    合并与重新水化                        │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  ├─ UUID 目录识别 (tag/watch)                           │   │
│  │  ├─ 存在性检查 + 替换策略                               │   │
│  │  ├─ 目录复制 + 原子替换                                 │   │
│  │  └─ rehydrate_entity() 重新水化对象                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 核心设计优势

1. **独立存储架构**：每个 watch/tag 拥有独立目录，便于备份恢复和原子操作
2. **灵活的合并策略**：通过4个选项控制恢复行为，支持部分恢复
3. **多层安全防护**：大小限制、路径遍历防护、线程安全等
4. **后台执行**：备份和恢复都在后台线程执行，不阻塞 Web 请求
5. **向后兼容**：支持新旧格式的备份文件

### 数据合并关键原则

| 原则 | 说明 |
|------|------|
| **UUID 精确匹配** | 基于 UUID 进行实体匹配和替换 |
| **目录级原子替换** | 删除整个旧目录，再复制新目录，确保完整性 |
| **重新水化对象** | 从 JSON 文件重建为 Python 对象，确保类型正确 |
| **全局设置最后提交** | 所有实体恢复后才提交 changedetection.json |
