# 文件型 DataStore 数据一致性机制报告

## 1. 概述

Changedetection.io 的文件型 DataStore 采用原子写入、分层存储和即时提交机制，确保 Watch 配置数据与历史快照数据的一致性。本报告详细说明保存与提交顺序、历史读取路径以及崩溃恢复机制。

---

## 2. 核心类架构

### 2.1 类继承关系

```
dict (Python 内置)
  ↓
watch_base (model/__init__.py)
  ↓         ↳ 提供基础数据结构、编辑追踪、commit() 框架
  |
EntityPersistenceMixin (model/persistence.py)
  ↓         ↳ 提供 _save_to_disk() 原子写入实现
  |
Watch.model (model/Watch.py)
            ↳ Watch 特定业务逻辑、历史管理
```

### 2.2 关键文件位置

| 文件 | 职责 |
|------|------|
| `store/file_saving_datastore.py` | 原子写入核心实现、watch.json/tag.json 保存 |
| `model/__init__.py` | watch_base 基类、commit() 框架、编辑追踪 |
| `model/persistence.py` | EntityPersistenceMixin 持久化混合类 |
| `model/Watch.py` | Watch 模型、历史记录管理、快照保存 |

---

## 3. 保存与提交顺序

### 3.1 数据修改追踪

Watch 继承自 dict，所有修改操作都会触发编辑标记：

```python
# watch_base.__setitem__ (model/__init__.py:370-383)
def __setitem__(self, key, value):
    super().__setitem__(key, value)
    self._mark_field_as_edited(key)  # 标记字段已编辑
```

**标记规则**（`_mark_field_as_edited`，326-368行）：
- 跳过 `__` 前缀的临时内存字段（不持久化）
- 跳过后端自动更新字段（如 `last_checked`、`fetch_time` 等）
- 仅用户可编辑字段会触发 `__watch_was_edited = True`

### 3.2 commit() 执行流程

当调用 `watch.commit()` 时（`watch_base.commit`，656-694行）：

```
Step 1: 检查 data_dir 和 UUID 是否有效
    ↓
Step 2: 调用 _get_commit_data() 获取数据快照
    │   - Watch._get_commit_data() (Watch.py:1064-1093)
    │   - 排除 processor_config_* 字段（单独存储）
    │   - 排除 __ 前缀临时字段
    │   - 规范化 browser_steps（无意义步骤转为空列表）
    │   - 持有 datastore.lock 防止并发修改
    ↓
Step 3: 调用 _save_to_disk() 持久化
    │   - EntityPersistenceMixin._save_to_disk() (persistence.py:52-84)
    │   - 自动确定实体类型 (watch/tag)
    │   - 调用 save_entity_atomic()
    ↓
Step 4: 原子写入文件
    └─ save_json_atomic() (file_saving_datastore.py:36-175)
```

### 3.3 原子写入机制

**`save_json_atomic()`** 核心流程（file_saving_datastore.py:36-175行）：

```
1. 创建临时文件
   tempfile.mkstemp(suffix='.tmp', prefix='json-', dir=parent_dir)

2. 序列化数据
   - orjson (优先) 或标准 json.dumps
   - 大小验证 (watch: 10MB, tag: 1MB)

3. 写入临时文件 + fsync (可选)
   - FORCE_FSYNC_DATA_IS_CRITICAL 环境变量控制
   - 默认: False (性能优先)
   - 设置为 True 时: os.fsync(fd) 强制刷盘

4. 原子替换 (os.replace)
   os.replace(temp_path, file_path)
   - POSIX 保证原子性
   - 原文件要么完整保留，要么被新文件完全替换

5. 新文件目录 fsync (确保文件名元数据持久化)
   - 仅针对新创建的文件
   - dir_fd = os.open(parent_dir, os.O_RDONLY)
   - os.fsync(dir_fd)
```

**关键保证**：原子写入确保 watch.json 永远不会处于半写损坏状态。

---

## 4. 历史记录数据路径

### 4.1 目录结构

```
datastore/
├── changedetection.json          # 全局设置
├── {uuid1}/                      # 单个 Watch 目录
│   ├── watch.json               # Watch 配置 (原子写入)
│   ├── history.txt              # 历史索引文件
│   ├── {timestamp}.txt          # 文本快照
│   ├── {timestamp}.txt.br       # Brotli 压缩文本快照
│   ├── {timestamp}.html.br      # 原始 HTML 快照
│   ├── {timestamp}.png/jpg     # 截图/二进制文件
│   ├── last-screenshot.png      # 最新截图
│   ├── last-error.txt           # 错误信息
│   ├── last-fetched.br          # 过滤前原始内容缓存
│   ├── favicon.{ext}            # 网站图标
│   └── {processor}.json         # 处理器配置 (如 restock_diff.json)
└── {uuid2}/
    └── ...
```

### 4.2 历史快照保存流程

**`save_history_blob()`**（Watch.py:653-729行）：

```
1. 确保数据目录存在
   ensure_data_dir_exists()

2. 根据内容类型处理
   ├─ 二进制内容 (bytes)
   │  ├─ puremagic 检测文件类型
   │  ├─ 保存为 {snapshot_id}.{ext}
   │  └─ 原子写入 (_write_atomic)
   │
   └─ 文本内容 (str)
      ├─ 超过阈值且启用 Brotli → {snapshot_id}.txt.br
      └─ 否则 → {snapshot_id}.txt

3. 追加到 history.txt (原子追加)
   with open(index_fname, 'a', encoding='utf-8') as f:
       f.write(f"{timestamp},{snapshot_fname}\n")
       f.flush()
       os.fsync(f.fileno())  # 强制刷盘

4. 更新内存状态
   self.__newest_history_key = timestamp
   self.__history_n += 1

5. 历史裁剪 (如配置了最大长度)
   if maxlen and self.__history_n > maxlen:
       self.history_trim(newest_n_items=maxlen)
```

### 4.3 历史读取流程

**`history` 属性**（Watch.py:442-491行）：

```
1. 读取 history.txt
   for line in open(history.txt):
       timestamp, filename = line.strip().split(',', 2)

2. 路径安全验证
   - 解析为绝对路径: os.path.realpath(os.path.join(data_dir, filename))
   - 确保路径在 data_dir 内（防止路径穿越）
   - 验证文件实际存在

3. 返回有序字典
   {timestamp: full_filepath}
```

**`get_history_snapshot()`**（Watch.py:553-605行）：

```
1. 路径安全检查
   safe_data_dir = os.path.realpath(self.data_dir)
   确保快照路径在 safe_data_dir 内

2. 文件类型判断
   ├─ 二进制文件 (.png, .jpg, .pdf, etc.)
   │  └─ 直接返回原始字节
   │
   └─ 文本文件
      ├─ 查找 .br 压缩版本
      ├─ 不存在则查找未压缩版本
      └─ Brotli 解压返回字符串
```

---

## 5. 部分写入或崩溃后的恢复思路

### 5.1 崩溃场景分析

| 崩溃时间点 | 影响 | 恢复策略 |
|-----------|------|---------|
| 写临时文件过程中 | 临时文件残留，原文件完整 | 下次启动时清理 .tmp 文件 |
| os.replace() 执行中 | POSIX 保证原子，无影响 | 无需恢复 |
| history.txt 追加中 | 可能出现不完整行 | 启动时验证并截断到最后完整行 |
| 快照文件写入中 | 快照文件损坏 | 重新检查时重新生成 |

### 5.2 现有恢复机制

#### 5.2.1 watch.json 完整性保护

- **原子写入**：崩溃时最多丢失最后一次提交，原文件保持完整
- **JSON 解析验证**：加载时解析失败会记录错误并跳过该 watch
  ```python
  # load_watch_from_file() (file_saving_datastore.py:211-270行)
  except json.JSONDecodeError as e:
      logger.critical(f"CORRUPTED WATCH DATA: {uuid}...")
      return None  # 跳过损坏的 watch
  ```

#### 5.2.2 history.txt 启动时验证

当前实现中，读取 history.txt 时每行都会：
1. 验证格式（包含逗号）
2. 解析出时间戳和文件名
3. 验证文件实际存在
4. 跳过无效/损坏的条目

#### 5.2.3 临时文件清理

`save_json_atomic()` 中的异常处理：
```python
except Exception as e:
    # 关闭文件描述符
    if not fd_closed:
        try: os.close(fd)
        except: pass
    # 删除临时文件
    if os.path.exists(temp_path):
        try: os.unlink(temp_path)
        except: pass
```

### 5.3 建议增强的恢复策略

#### 策略1：启动时完整数据一致性检查

```python
# 建议添加到 datastore 初始化流程
def run_data_consistency_check():
    for uuid in watch_dirs:
        # 1. 验证 watch.json
        try:
            with open(watch_json) as f:
                json.load(f)
        except:
            # 尝试从备份恢复或标记为损坏
            handle_corrupted_watch(uuid)

        # 2. 验证 history.txt
        history_file = os.path.join(uuid_dir, 'history.txt')
        if os.path.exists(history_file):
            validate_and_repair_history(history_file)

        # 3. 验证快照文件存在性
        # 对于 history.txt 中列出的每个快照
        # 验证文件存在且可读取
```

#### 策略2：history.txt 双写或 WAL

```python
# 建议实现预写日志 (Write-Ahead Log)
# 追加新历史条目前先写 WAL
def save_history_with_wal():
    # 1. 先写 WAL
    wal_path = os.path.join(data_dir, 'history.wal')
    with open(wal_path, 'w') as f:
        f.write(f"{timestamp},{snapshot_fname}\n")
    os.fsync(f)

    # 2. 再追加到主文件
    with open(history_txt, 'a') as f:
        f.write(...)
    os.fsync(f)

    # 3. 删除 WAL
    os.unlink(wal_path)

# 启动时检查 WAL 存在则重放
if os.path.exists(wal_path):
    replay_wal_entry(wal_path)
```

#### 策略3：自动备份 watch.json

```python
# 每次 commit 前先备份上一版本
def commit_with_backup():
    watch_json = os.path.join(data_dir, 'watch.json')
    if os.path.exists(watch_json):
        backup = os.path.join(data_dir, 'watch.json.bak')
        shutil.copy2(watch_json, backup)

    # 然后执行正常 commit
```

#### 策略4：部分写入检测

```python
# 检测 history.txt 中的不完整行
def repair_history_txt(history_path):
    lines = []
    with open(history_path, 'r') as f:
        for line in f:
            line = line.strip()
            # 验证行格式: timestamp,filename
            if ',' in line and len(line.split(',', 1)) == 2:
                timestamp, filename = line.split(',', 1)
                if timestamp.isdigit():  # 时间戳应为数字
                    lines.append(line)

    # 重写完整的 history.txt
    with open(history_path, 'w') as f:
        f.write('\n'.join(lines) + '\n')
```

---

## 6. 并发安全机制

### 6.1 datastore.lock 保护

- 所有 `commit()` 操作在获取数据快照时持有 `datastore.lock`
- 防止并发修改导致的数据不一致
- Python `threading.Lock()` 实现

### 6.2 文件系统级原子性

- `os.replace()` 是原子系统调用
- 多进程同时写入同一文件不会导致内容损坏
- 最后写入者获胜

---

## 7. 关键配置参数

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `FORCE_FSYNC_DATA_IS_CRITICAL` | False | 强制 fsync 刷盘，True = 一致性优先，False = 性能优先 |
| `DISABLE_BROTLI_TEXT_SNAPSHOT` | False | 禁用 Brotli 文本压缩 |
| `SNAPSHOT_BROTLI_COMPRESSION_THRESHOLD` | 20KB | 启用 Brotli 压缩的最小大小 |
| `FILTER_FAILURE_NOTIFICATION_SEND_DEFAULT` | True | 过滤器失败时发送通知 |
| `MINIMUM_SECONDS_RECHECK_TIME` | 3 | 最小重检查间隔 |

---

## 8. 总结

### 8.1 现有保证

1. ✅ **watch.json 永不损坏**：原子写入 + 临时文件替换
2. ✅ **崩溃后可启动**：损坏的 watch 会被跳过并记录日志
3. ✅ **部分历史丢失不影响整体**：history.txt 条目逐行验证
4. ✅ **路径穿越防护**：所有文件访问限制在 data_dir 内

### 8.2 潜在风险点

1. ⚠️ history.txt 追加时崩溃可能产生不完整行
2. ⚠️ 快照文件写入时崩溃可能产生损坏文件
3. ⚠️ fsync 默认关闭，系统崩溃可能导致最近几秒数据丢失

### 8.3 建议改进优先级

| 优先级 | 改进 | 收益 |
|--------|------|------|
| 高 | history.txt 启动时修复 | 防止崩溃后历史索引损坏 |
| 中 | watch.json 自动备份 | 极端情况下可恢复 |
| 低 | WAL 预写日志 | 强一致场景使用 |
| 低 | 快照文件校验和 | 检测损坏快照 |

---

**报告生成时间**：2026-05-17
**基于代码版本**：file_saving_datastore.py, Watch.py, model/__init__.py
