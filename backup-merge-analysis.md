# 备份恢复与 Datastore 合并机制深度分析报告

## 目录
1. [存储主体架构](#1-存储主体架构)
2. [核心组合关系](#2-核心组合关系)
3. [恢复分支逻辑详解](#3-恢复分支逻辑详解)
4. [同一实体的覆盖与跳过顺序](#4-同一实体的覆盖与跳过顺序)
5. [更新机制的执行流程](#5-更新机制的执行流程)
6. [测试验证方式分析](#6-测试验证方式分析)
7. [代码分支覆盖情况](#7-代码分支覆盖情况)
8. [关键发现与建议](#8-关键发现与建议)

---

## 1. 存储主体架构

### 1.1 实体类继承体系

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

### 1.2 关键类职责划分

| 类/模块 | 核心职责 | 关键方法 |
|--------|---------|---------|
| **watch_base** | 通用字段定义、commit 入口逻辑、通用行为 | `commit()`, `_get_commit_data()` |
| **EntityPersistenceMixin** | 实体类型识别、调用底层原子写入 | `_save_to_disk()` |
| **save_entity_atomic** | 文件级原子写入、大小校验 | `save_json_atomic()` |
| **Watch.model** | Watch 专用逻辑、历史记录管理 | `rehydrate_entity()` (datastore) |
| **Tag.model** | Tag 专用逻辑、URL 匹配规则 | `matches_url()` |

### 1.3 文件系统存储结构

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

## 2. 核心组合关系

### 2.1 存储主体与持久化机制的组合

```
┌─────────────────────────────────────────────────────────────────┐
│                  存储主体 × 持久化机制 组合矩阵                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 组合 1: Watch + 文件系统存储                              │  │
│  │   = watch_base (字段定义 + commit)                       │  │
│  │   + EntityPersistenceMixin (_save_to_disk)              │  │
│  │   + save_entity_atomic (原子写入)                        │  │
│  │                                                           │  │
│  │   结果: {uuid}/watch.json (10MB 限制)                    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 组合 2: Tag + 文件系统存储                                │  │
│  │   = watch_base (字段定义 + commit)                       │  │
│  │   + EntityPersistenceMixin (_save_to_disk)              │  │
│  │   + save_entity_atomic (原子写入)                        │  │
│  │                                                           │  │
│  │   结果: {uuid}/tag.json (1MB 限制)                       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 组合 3: 全局设置 + 文件系统存储                            │  │
│  │   = datastore.commit()                                   │  │
│  │   + save_json_atomic (直接调用)                          │  │
│  │                                                           │  │
│  │   结果: changedetection.json                              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 组合关键点

**关键点 1：EntityPersistenceMixin 的动态类型识别**

```python
# persistence.py: _determine_entity_type()
# 通过类继承层级动态识别实体类型（类级别缓存）
for base_class in inspect.getmro(cls):
    module_name = base_class.__module__
    if module_name.startswith('changedetectionio.model.'):
        # "changedetectionio.model.Watch" -> "watch"
        return module_name.split('.')[-1].lower()

# 结果:
# - Watch.model → entity_type = 'watch' → filename = 'watch.json' → max_size = 10MB
# - Tag.model → entity_type = 'tag' → filename = 'tag.json' → max_size = 1MB
```

**关键点 2：watch_base.commit() 的通用流程**

```python
# model/__init__.py: commit()
def commit(self):
    # 1. 校验：必须有 data_dir 和 UUID
    if not self.data_dir:
        logger.error("Cannot commit without datastore_path")
        return
    
    # 2. 获取待提交数据（子类可过滤）
    data_dict = self._get_commit_data()
    
    # 3. 委托给 Mixin 的 _save_to_disk()
    self._save_to_disk(data_dict, uuid)
```

---

## 3. 恢复分支逻辑详解

### 3.1 import_from_zip 完整分支树

```python
# restore.py: import_from_zip()
#
# ┌─────────────────────────────────────────────────────────────────┐
# │                  恢复函数完整分支树                               │
# └─────────────────────────────────────────────────────────────────┘
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

## 4. 同一实体的覆盖与跳过顺序

### 4.1 优先级规则：Tag 优先于 Watch

**关键发现：一个 UUID 目录中如果同时存在 tag.json 和 watch.json，只会恢复 Tag**

```python
# restore.py 第 89-131 行
# ┌─────────────────────────────────────────────────────────┐
# │ 注意: 这是 if-elif 结构，不是两个独立的 if！              │
# └─────────────────────────────────────────────────────────┘

if include_groups and os.path.exists(tag_json_path):
    # --- Tag 恢复逻辑 ---
    # (如果进入此分支，后续 elif 不会执行)
elif include_watches and os.path.exists(watch_json_path):
    # --- Watch 恢复逻辑 ---
    # (只有当 tag.json 不存在或 include_groups=False 时才会执行)
```

**结论：**
| 目录内容 | include_groups | include_watches | 实际恢复 |
|---------|---------------|-----------------|---------|
| tag.json + watch.json | True | True | **仅恢复 Tag** (优先级更高) |
| tag.json + watch.json | False | True | 恢复 Watch |
| tag.json + watch.json | True | False | 恢复 Tag |
| 只有 tag.json | True | True | 恢复 Tag |
| 只有 watch.json | True | True | 恢复 Watch |

### 4.2 覆盖 vs 跳过的判断顺序

```
对于每个实体（Tag 或 Watch）：
┌─────────────────────────────────────────────────────────────────┐
│              覆盖 / 跳过 判断决策树                              │
└─────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────┐
                          │ UUID 是否已存在？       │
                          └─────────────────────────┘
                                         │
                       ┌─────────────────┴─────────────────┐
                       │                                   │
                       ▼                                   ▼
              ┌─────────────┐                    ┌──────────────┐
              │   不存在    │                    │    已存在     │
              └─────────────┘                    └──────────────┘
                       │                                   │
                       ▼                                   ▼
              ┌───────────────────┐          ┌─────────────────────────┐
              │ 直接恢复（新增）  │          │ replace 参数是否为 True？│
              └───────────────────┘          └─────────────────────────┘
                       │                                   │
                       ▼                       ┌───────────┴───────────┐
              ┌───────────────────┐          │                       │
              │ 1. 删除旧目录（如有）│         ▼                       ▼
              │ 2. 复制新目录        │  ┌─────────────┐       ┌─────────────┐
              │ 3. 重新水化对象      │  │   覆盖      │       │    跳过     │
              │ 4. 存入内存          │  └─────────────┘       └─────────────┘
              │ 5. commit() 持久化   │          │                       │
              └───────────────────┘          ▼                       ▼
                                           ┌──────────────────┐   ┌─────────────┐
                                           │ 同"不存在"流程  │   │ 计数 + 跳过 │
                                           └──────────────────┘   └─────────────┘
```

### 4.3 四种恢复选项的组合效果矩阵

| include_groups | include_groups_replace | include_watches | include_watches_replace | 效果 |
|---------------|------------------------|----------------|-------------------------|------|
| **False** | 任意 | **False** | 任意 | 完全不恢复 |
| **True** | **False** | **False** | 任意 | 仅新增 Tags，跳过已存在的 Tags |
| **True** | **True** | **False** | 任意 | 新增 + 覆盖 Tags |
| **False** | 任意 | **True** | **False** | 仅新增 Watches，跳过已存在的 Watches |
| **False** | 任意 | **True** | **True** | 新增 + 覆盖 Watches |
| **True** | **False** | **True** | **False** | 仅新增 Tags 和 Watches |
| **True** | **True** | **True** | **False** | Tags 全覆盖 + Watches 仅新增 |
| **True** | **False** | **True** | **True** | Tags 仅新增 + Watches 全覆盖 |
| **True** | **True** | **True** | **True** | Tags 和 Watches 全覆盖（测试用例使用此组合） |

---

## 5. 更新机制的执行流程

### 5.1 恢复时的完整数据流动

```
ZIP 文件
   │
   ▼
 解压到临时目录 ────────────┐
   │                         │ 安全检查：
   ▼                         │ - 总大小限制
遍历每个 UUID 目录            │ - 路径遍历防护
   │                         │
   ▼                         │
判断实体类型 (Tag / Watch)  │
   │                         │
   ▼                         │
覆盖 / 跳过 决策             │
   │                         │
   ├─ 跳过 → 计数 + continue │
   │                         │
   ▼                         │
删除目标目录（如果存在）◀────┘
   │
   ▼
复制新目录到 datastore
   │
   ▼
┌─────────────────────────────────────────────────────────┐
│ 对象重新水化 (Rehydration)                               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 对于 Tag:                                                │
│   Tag.model(                                            │
│       datastore_path=...,                               │
│       __datastore=...,                                  │
│       default=tag_data                                  │
│   )                                                     │
│   + 强制设置 processor='restock_diff'                   │
│                                                         │
│ 对于 Watch:                                              │
│   datastore.rehydrate_entity(uuid, watch_data)          │
│   → 根据 processor 字段选择对应的 Watch 子类             │
│     (text_json_diff, restock_diff, etc.)                │
└─────────────────────────────────────────────────────────┘
   │
   ▼
存入内存字典（current_tags / current_watches）
   │
   ▼
obj.commit() → 持久化到 {uuid}/(watch|tag).json
   │
   ▼
最后: datastore.commit() → 持久化 changedetection.json
```

### 5.2 commit() 调用层级

```python
# 恢复过程中有 N+1 次 commit 调用：
#
# 1. 每个 Tag 恢复 → tag_obj.commit() → tag.json
# 2. 每个 Watch 恢复 → watch_obj.commit() → watch.json
# 3. 最后 datastore.commit() → changedetection.json
#
#        ┌─────────────────────────────────────────────────┐
#        │              commit() 调用层级                  │
#        └─────────────────────────────────────────────────┘
#
# Watch.model.commit()
#     │
#     └── watch_base.commit()
#           ├── 校验 data_dir, uuid
#           ├── _get_commit_data() → 获取全部 dict 数据
#           └── EntityPersistenceMixin._save_to_disk()
#                 ├── _determine_entity_type() → 'watch'
#                 ├── filename = 'watch.json'
#                 ├── max_size_mb = 10
#                 └── save_entity_atomic()
#                       └── save_json_atomic()
#                             ├── tempfile.mkstemp()
#                             ├── 写入 JSON
#                             ├── os.replace() 原子重命名
#                             └── 可选 fsync
#
# Tag.model.commit() 完全相同，区别仅在于：
#   _determine_entity_type() → 'tag'
#   filename = 'tag.json'
#   max_size_mb = 1
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
#
# ┌─────────────────────────────────────────────────────────────┐
# │ 阶段 1: 准备数据 (第 130-136 行)                            │
# └─────────────────────────────────────────────────────────────┘
uuid = datastore.add_watch(url=watch_url)              # 创建 Watch
tag_uuid = datastore.add_tag(title="Tasty backup tag")  # 创建 Tag 1
tag_uuid2 = datastore.add_tag(title="Tasty backup tag number two")  # 创建 Tag 2
client.get(url_for("ui.form_watch_checknow"))  # 触发检查，生成历史
wait_for_all_checks(client)

# ┌─────────────────────────────────────────────────────────────┐
# │ 阶段 2: 创建并下载备份 (第 138-152 行)                      │
# └─────────────────────────────────────────────────────────────┘
client.get(url_for("backups.request_backup"))  # 触发后台备份
time.sleep(4)  # 等待备份线程完成
res = client.get(url_for("backups.download_backup", filename="latest"))
zip_data = res.data

# 验证 ZIP 内容（不是直接恢复验证，而是验证备份正确包含文件）
backup = ZipFile(io.BytesIO(zip_data))
names = backup.namelist()
assert f"{uuid}/watch.json" in names          # Watch 配置文件存在
assert f"{tag_uuid}/tag.json" in names        # Tag 1 配置文件存在
assert f"{tag_uuid2}/tag.json" in names       # Tag 2 配置文件存在

# ┌─────────────────────────────────────────────────────────────┐
# │ 阶段 3: 清空现有数据 (第 154-160 行)                        │
# └─────────────────────────────────────────────────────────────┘
datastore.delete('all')                              # 删除所有 Watches
client.get(url_for("tags.delete_all"))               # 删除所有 Tags
# 验证：确认数据已被清除
assert uuid not in datastore.data['watching']
assert tag_uuid not in datastore.data['settings']['application']['tags']

# ┌─────────────────────────────────────────────────────────────┐
# │ 阶段 4: 执行恢复 (第 162-178 行)                            │
# └─────────────────────────────────────────────────────────────┘
res = client.post(
    url_for("backups.restore.backups_restore_start"),
    data={
        'zip_file': (io.BytesIO(zip_data), 'backup.zip'),
        'include_groups': 'y',                          # 恢复 Tags
        'include_groups_replace_existing': 'y',         # 覆盖已存在的（虽然此时为空）
        'include_watches': 'y',                         # 恢复 Watches
        'include_watches_replace_existing': 'y',        # 覆盖已存在的
    },
    content_type='multipart/form-data'
)
time.sleep(2)  # 等待恢复线程完成

# ┌─────────────────────────────────────────────────────────────┐
# │ 阶段 5: 验证恢复结果 (第 180-202 行)                        │
# └─────────────────────────────────────────────────────────────┘
# Watch 验证
restored_watch = datastore.data['watching'].get(uuid)
assert restored_watch is not None                  # 对象存在
assert restored_watch['url'] == watch_url          # 关键数据正确
assert isinstance(restored_watch, Watch.model)      # 类型正确（重新水化成功）
assert restored_watch.history_n >= 1               # 历史记录恢复

# Tag 验证
restored_tags = datastore.data['settings']['application']['tags']
restored_tag = restored_tags.get(tag_uuid)
assert restored_tag is not None                    # 对象存在
assert restored_tag['title'] == "Tasty backup tag"  # 关键数据正确
assert isinstance(restored_tag, Tag.model)          # 类型正确（重新水化成功）
```

### 6.3 安全测试的特殊验证方式

**Zip Slip 测试：直接调用底层函数**
```python
# 不通过 Web 接口，直接调用 import_from_zip()
# 原因：可以精确控制 zip 内容（否则通过 Flask 文件上传可能会被 sanitize）
with pytest.raises(ValueError, match="Zip Slip"):
    import_from_zip(
        zip_stream=malicious_zip,
        datastore=datastore,
        include_groups=True,
        include_groups_replace=True,
        include_watches=True,
        include_watches_replace=True,
    )
```

**Zip Bomb 测试：动态 monkey-patch**
```python
# 动态修改限制值（不影响其他测试）
original_limit = restore_mod._MAX_DECOMPRESSED_BYTES
try:
    restore_mod._MAX_DECOMPRESSED_BYTES = 50 * 1024  # 临时改为 50KB
    with pytest.raises(ValueError, match="decompressed size"):
        import_from_zip(...)
finally:
    restore_mod._MAX_DECOMPRESSED_BYTES = original_limit  # 恢复原值
```

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
| ✅ tag_obj.commit() 持久化 | 间接覆盖 | `test_backup_restore`（验证对象存在） |
| ✅ watch_obj.commit() 持久化 | 间接覆盖 | `test_backup_restore`（验证对象存在） |
| ✅ datastore.commit() 持久化 | 间接覆盖 | `test_backup_restore` |

### 7.2 未覆盖分支（关键发现！）

| 分支 | 覆盖情况 | 说明 |
|------|---------|------|
| ❌ 无效 UUID 目录跳过 | 未覆盖 | 目录名不是 UUID 格式时的 warning + continue |
| ❌ tag.json JSON 解析失败 | 未覆盖 | tag.json 损坏或无效时的 error + continue |
| ❌ watch.json JSON 解析失败 | 未覆盖 | watch.json 损坏或无效时的 error + continue |
| ❌ **Tag 已存在且不替换 (include_groups_replace=False)** | **未覆盖** | **最重要的缺失分支！** |
| ❌ **Watch 已存在且不替换 (include_watches_replace=False)** | **未覆盖** | **最重要的缺失分支！** |
| ❌ Tag 已存在且替换 (include_groups_replace=True) | 未覆盖 | 删除旧目录后复制新目录的逻辑 |
| ❌ Watch 已存在且替换 (include_watches_replace=True) | 未覆盖 | 删除旧目录后复制新目录的逻辑 |
| ❌ include_groups=False 时跳过所有 Tags | 未覆盖 | 不恢复 Tags 的情况 |
| ❌ include_watches=False 时跳过所有 Watches | 未覆盖 | 不恢复 Watches 的情况 |
| ❌ 目录既无 tag.json 也无 watch.json | 未覆盖 | 静默跳过的情况 |
| ❌ commit() 时无 data_dir 的错误处理 | 未覆盖 | 异常路径 |
| ❌ commit() 时无 uuid 的错误处理 | 未覆盖 | 异常路径 |
| ❌ 压缩后大小超过限制 (save_json_atomic) | 未覆盖 | 异常路径 |

### 7.3 最高优先级的缺失测试场景

**场景 A：恢复时目标实体已存在，且不允许替换**
```
预期行为:
1. 现有 Watch W1 (uuid=xxx) + Tag T1 (uuid=yyy) 存在于 datastore
2. 备份 zip 中包含相同 UUID 的 W1' 和 T1'
3. 恢复时使用 include_groups_replace=False, include_watches_replace=False
4. 结果: W1 和 T1 保持不变，计数 skipped_watches +=1, skipped_tags +=1
```

**场景 B：恢复时目标实体已存在，且允许替换**
```
预期行为:
1. 现有 Watch W1 (uuid=xxx) 内容为 "A"
2. 备份 zip 中 W1' 内容为 "B"
3. 恢复时 include_watches_replace=True
4. 结果: W1 被替换为 "B"，旧目录被删除，新目录被复制
```

---

## 8. 关键发现与建议

### 8.1 关键发现

**发现 1：Tag 恢复优先级高于 Watch（if-elif 结构）**
- 一个 UUID 目录同时有 tag.json 和 watch.json 时，只会恢复 Tag
- Watch 恢复逻辑永远不会执行
- 这种设计可能隐含假设：Tag 和 Watch 的 UUID 空间完全隔离
- 风险：如果 UUID 生成逻辑有重叠，Watch 会被静默丢弃

**发现 2：恢复分支覆盖率严重不足**
- 8 个主要恢复分支中仅覆盖了 3 个（清空后恢复的 happy path）
- 最重要的"已存在且不替换"逻辑完全没有测试覆盖
- 安全分支测试覆盖较好，但业务逻辑分支测试缺失

**发现 3：恢复结果只有日志记录，缺乏结构化反馈**
- import_from_zip 返回计数字典 `{restored_groups, skipped_groups, ...}`
- 但后台线程执行时，这个返回值被丢弃了（线程无返回值）
- 用户无法知道到底恢复了什么、跳过了什么

**发现 4：同一个 UUID 下同时存在 Tag 和 Watch 时的静默行为**
- 当备份 zip 中一个 UUID 目录同时有 tag.json 和 watch.json
- 用户选择同时恢复 Tags 和 Watches 时
- 实际只会恢复 Tag，Watch 被静默跳过
- 没有日志、没有警告、没有用户反馈

### 8.2 改进建议

**建议 1：补充核心分支测试**

优先级从高到低：
1. **已存在 + 不替换** → 验证跳过逻辑和计数正确
2. **已存在 + 替换** → 验证目录删除、复制、更新正确
3. **include_groups=False / include_watches=False** → 验证不恢复
4. **损坏 JSON** → 验证异常跳过、日志记录
5. **无效 UUID 目录** → 验证 warning 日志

**建议 2：改进 Tag/Watch 冲突处理**

选项 A：**在备份创建时确保 UUID 不冲突**
- 备份时检查 watches 和 tags 的 UUID 是否有重叠
- 如有冲突，记录 warning

选项 B：**在恢复时检测并报告冲突**
```python
# 恢复时添加检测
if os.path.exists(tag_json_path) and os.path.exists(watch_json_path):
    logger.warning(f"UUID {uuid} contains both tag.json and watch.json, only Tag will be restored")
```

选项 C：**改为两个独立 if，先处理 Tag 再处理 Watch**
```python
# 当前：if-elif → 只能处理一个
# 建议：if + if → 两者都尝试（但仍需检测 UUID 冲突）
if include_groups and os.path.exists(tag_json_path):
    process_tag()
if include_watches and os.path.exists(watch_json_path):
    process_watch()  # 现在可以执行了！
```

**建议 3：保存恢复结果供用户查看**
```python
# 恢复线程完成后，将结果保存到 datastore 或临时文件
restore_result = import_from_zip(...)
# 保存：{timestamp: restore_result}
# 用户可以通过 UI 查看：恢复了 X 个，跳过了 Y 个
```

### 8.3 代码优化点

**优化 1：消除魔法字符串 'y'**
```python
# 当前代码（restore.py 第 187-246 行）
include_groups = request.form.get('include_groups') == 'y'
# 建议：使用常量或更明确的布尔值转换
```

**优化 2：增加冲突检测日志**
```python
# 在 if include_groups ... 前添加
has_tag = os.path.exists(tag_json_path)
has_watch = os.path.exists(watch_json_path)
if has_tag and has_watch:
    logger.warning(f"UUID {uuid} contains both entity types, priority: Tag > Watch")
```

**优化 3：恢复完成后发送通知信号**
```python
# 使用 blinker signal 通知恢复完成
from blinker import signal
restore_completed = signal('restore_completed')
restore_completed.send(result=restore_result)
```

---

## 附录：核心代码位置速查表

| 功能 | 文件 | 行号范围 |
|------|------|---------|
| import_from_zip 主逻辑 | `backups/restore.py` | 40-169 |
| Tag 恢复分支 | `backups/restore.py` | 89-124 |
| Watch 恢复分支 | `backups/restore.py` | 127-155 |
| 覆盖/跳过判断 | `backups/restore.py` | 90-94, 128-131 |
| 实体持久化 Mixin | `model/persistence.py` | 37-84 |
| watch_base 基类 | `model/__init__.py` | 15-690 |
| commit() 方法 | `model/__init__.py` | 649-690 |
| 原子写入 | `store/file_saving_datastore.py` | 36-176 |
| 备份恢复测试 | `tests/test_backup.py` | 122-261 |
