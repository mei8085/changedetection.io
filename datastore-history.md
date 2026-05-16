# 文件型 DataStore 数据一致性机制报告

## 1. 概述

Changedetection.io 的文件型 DataStore 采用原子写入、分层存储和即时提交机制，确保 Watch 配置数据与历史快照数据的一致性。

**重要前提**：本机制仅保证单进程场景下的数据完整性，不支持多进程并发写入。多进程场景下需使用 Redis/SQL 后端。

本报告详细说明保存与提交顺序、历史读取路径以及崩溃恢复机制。

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
   - POSIX 保证原子性：系统调用层面不可中断
   - 原文件要么完整保留，要么被新文件完全替换
   - 仅保证单文件原子性，不保证多进程并发安全

5. 新文件目录 fsync (确保文件名元数据持久化)
   - 仅针对新创建的文件
   - dir_fd = os.open(parent_dir, os.O_RDONLY)
   - os.fsync(dir_fd)
```

**关键保证**：原子写入确保单进程下 watch.json 永远不会处于半写损坏状态。

**重要限制**：`os.replace()` 仅保证单个文件替换的系统调用原子性，不能解决多进程场景下的竞态条件（如进程 A 读取、进程 B 同时写入）。

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

## 5. 部分写入或崩溃后的恢复机制

### 5.1 崩溃场景分析

| 崩溃时间点 | 影响 |
|-----------|------|
| 写临时文件过程中 | 临时文件残留，原文件完整 |
| os.replace() 执行中 | POSIX 保证原子，单文件无损坏 |
| history.txt 追加中 | 可能出现不完整行 |
| 快照文件写入中 | 快照文件损坏 |

---

### 5.2 临时文件崩溃处理

#### 5.2.1 现有实现（代码行为）

**位置**：`save_json_atomic()` - file_saving_datastore.py:142-175行

```python
except Exception as e:
    # 1. 关闭文件描述符
    if not fd_closed:
        try:
            os.close(fd)
        except:
            pass
    # 2. 删除临时文件
    if os.path.exists(temp_path):
        try:
            os.unlink(temp_path)
        except:
            pass
```

**当前行为逐条说明**：

1. ✅ **异常捕获**：所有异常（包括磁盘满、权限错误等）都会进入异常处理分支
2. ✅ **文件描述符关闭**：尝试关闭可能未关闭的文件描述符，忽略关闭失败
3. ✅ **临时文件删除**：检查临时文件存在后尝试删除，忽略删除失败
4. ❌ **启动时残留清理**：崩溃后遗留的 `.tmp` 文件不会在下次启动时自动清理

#### 5.2.2 建议方案

| 改进项 | 建议实现 |
|--------|---------|
| 启动时清理残留 .tmp 文件 | datastore 初始化时遍历所有 watch 目录，删除 `*.tmp` 文件 |
| 临时文件时间戳校验 | 仅删除超过 N 分钟的临时文件，避免清理正在写入的文件 |

---

### 5.3 history.txt 崩溃处理

#### 5.3.1 现有实现（代码行为）

**位置**：`history` 属性 getter - Watch.py:442-491行

```python
if os.path.isfile(fname):
    logger.debug(f"Reading watch history index for {self.get('uuid')}")
    with open(fname, "r", encoding='utf-8') as f:
        for i in f.readlines():
            if ',' in i:  # 1. 验证行格式
                k, v = i.strip().split(',', 2)
                
                # 2. 路径安全验证 + 文件存在验证
                safe_data_dir = os.path.realpath(self.data_dir)
                snapshot_fname = os.path.basename(v.strip())
                resolved_path = os.path.realpath(os.path.join(self.data_dir, snapshot_fname))
                
                if not resolved_path.startswith(safe_data_dir + os.sep) and resolved_path != safe_data_dir:
                    continue  # 跳过不安全路径
                    
                if not os.path.exists(resolved_path):
                    continue  # 跳过不存在的文件
                    
                tmp_history[k] = resolved_path  # 仅保留有效条目
```

**当前行为逐条说明**：

1. ✅ **逐行验证格式**：每行必须包含逗号，否则被静默跳过
2. ✅ **路径安全检查**：通过 `os.path.realpath()` 确保不超出 data_dir
3. ✅ **文件存在验证**：引用的快照文件必须实际存在才加入内存索引
4. ❌ **不完整行处理**：崩溃导致的行截断（无换行符）会被 `readlines()` 读入，但因无逗号被跳过
5. ❌ **磁盘修复**：内存中跳过无效行，但不会重写修复 history.txt 文件
6. ❌ **启动时完整性检查**：无专门的启动时校验逻辑，仅在访问时验证

#### 5.3.2 建议方案

| 改进项 | 建议实现 |
|--------|---------|
| 启动时完整性校验 | datastore 加载完成后扫描所有 watch 的 history.txt |
| 不完整行检测 | 检查最后一行是否以换行符结尾，无则截断 |
| 修复后重写 | 检测到无效条目后，清理并重写 history.txt |
| 行格式校验增强 | 验证时间戳部分为纯数字，文件名部分合法 |

---

### 5.4 watch.json 崩溃处理

#### 5.4.1 现有实现（代码行为）

**位置**：`load_watch_from_file()` - file_saving_datastore.py:211-270行

```python
try:
    # 1. 文件大小校验
    file_size = os.path.getsize(watch_json)
    if file_size > 10 * 1024 * 1024:  # 10MB 上限
        logger.critical(f"CORRUPTED WATCH DATA: {uuid} 文件过大")
        return None
    
    # 2. JSON 解析验证
    if HAS_ORJSON:
        with open(watch_json, 'rb') as f:
            watch_data = orjson.loads(f.read())
    else:
        with open(watch_json, 'r', encoding='utf-8') as f:
            watch_data = json.load(f)
            
    return watch_data
    
except json.JSONDecodeError as e:
    logger.critical(f"CORRUPTED WATCH DATA: {uuid} JSON 解析失败")
    return None  # 跳过损坏的 watch
except ValueError as e:
    if "invalid json" in str(e).lower() or HAS_ORJSON:
        logger.critical(f"CORRUPTED WATCH DATA: {uuid} JSON 解析失败")
        return None
    raise
```

**当前行为逐条说明**：

1. ✅ **文件大小检查**：超过 10MB 视为损坏，跳过加载
2. ✅ **JSON 解析验证**：解析失败记录 critical 日志并返回 None
3. ✅ **损坏 watch 跳过**：加载失败的 watch 不会加入内存索引
4. ❌ **无自动恢复**：检测到损坏后不尝试从备份恢复
5. ❌ **无损坏标记**：损坏的 watch.json 文件保留在磁盘，下次启动继续报错

#### 5.4.2 建议方案

| 改进项 | 建议实现 |
|--------|---------|
| 自动备份机制 | 每次成功 commit() 后保留上一版本为 watch.json.bak |
| 损坏自动恢复 | 加载失败时尝试从 watch.json.bak 恢复 |
| 用户通知 | WebUI 中显示损坏的 watch 及恢复选项 |

---

## 6. 并发安全机制（单进程内）

### 6.1 datastore.lock 保护

- 所有 `commit()` 操作在获取数据快照时持有 `datastore.lock`
- 防止单进程内多线程并发修改导致的数据不一致
- Python `threading.Lock()` 实现
- **仅保护线程安全，不保护进程安全**

### 6.2 os.replace() 的实际边界

**正确理解**：
- `os.replace()` 是原子系统调用，不会产生半写文件
- 任何时刻文件要么是旧版本，要么是新版本
- 单进程内配合 datastore.lock 使用安全

**常见误解澄清**：
- ❌ **不等于多进程并发安全**：进程 A 读取后进程 B 写入，进程 A 基于旧数据计算后写入会产生丢失更新
- ❌ **不保证跨文件一致性**：同时写入 watch.json 和 history.txt 可能出现部分成功
- ❌ **不解决 NFS 分布式锁问题**：网络文件系统上原子性可能削弱

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

1. ✅ **单进程下 watch.json 永不损坏**：原子写入 + 临时文件替换
2. ✅ **崩溃后可启动**：损坏的 watch 会被跳过并记录日志
3. ✅ **history.txt 内存级容错**：读取时逐行验证，无效条目被跳过
4. ✅ **路径穿越防护**：所有文件访问限制在 data_dir 内
5. ✅ **异常时临时文件清理**：写入过程中异常会删除当前临时文件

### 8.2 潜在风险点

1. ⚠️ **崩溃残留临时文件**：进程崩溃后遗留的 `.tmp` 文件不会自动清理
2. ⚠️ **history.txt 磁盘级不修复**：内存中跳过但不重写修复文件
3. ⚠️ **快照文件写入时崩溃**：可能产生损坏的快照文件，下次读取失败
4. ⚠️ **fsync 默认关闭**：系统崩溃可能导致最近几秒数据丢失
5. ⚠️ **多进程场景不安全**：当前设计仅支持单进程部署

### 8.3 建议改进优先级

| 优先级 | 改进 | 对应章节 | 收益 |
|--------|------|---------|------|
| 高 | 启动时清理残留 .tmp 文件 | 5.2.2 | 防止磁盘空间泄漏 |
| 高 | history.txt 启动时检测并重写 | 5.3.2 | 防止历史索引累积损坏 |
| 中 | watch.json 自动备份机制 | 5.4.2 | 极端情况下可恢复配置 |
| 低 | WAL 预写日志用于 history.txt | 5.3.2 | 强一致场景使用 |
| 低 | 快照文件校验和验证 | 4.2 | 检测静默损坏的快照 |

---

**报告生成时间**：2026-05-17
**修订版本**：v2 - 明确单进程边界，拆分现有实现和建议方案
**基于代码版本**：file_saving_datastore.py, Watch.py, model/__init__.py
