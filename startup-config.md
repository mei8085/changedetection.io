# changedetection.io 启动配置体系分析

## 1. 启动入口与初始化调用链

### 1.1 入口点

```
changedetection.py  →  changedetectionio.main()
```

[changedetection.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetection.py#L7-L8) 仅做转发，真正逻辑在 [changedetectionio/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/__init__.py#L181) 的 `main()` 中。

### 1.2 初始化顺序（按时间线）

| 阶段 | 代码位置 | 做了什么 |
|------|----------|----------|
| ① 模块级全局配置 | `__init__.py` 顶部 | 设置 `multiprocessing.set_start_method('spawn')`；设置 `MALLOC_ARENA_MAX` |
| ② CLI 参数解析 | `main()` L198-L323 | getopt 解析 `-h/-p/-d/-l/-s/-P` 等；预处理 `-u/-r/-b` |
| ③ 环境变量 → 运行时变量 | `main()` L204-L206 | `LISTEN_HOST`→`host`，`PORT`→`port`；`LOGGER_LEVEL`→日志级别 |
| ④ 创建 DataStore | `main()` L383 | `ChangeDetectionStore(datastore_path, ...)` — **核心配置装载点** |
| ⑤ 创建 Flask App | `main()` L428 | `changedetection_app(app_config, datastore)` |
| ⑥ 启动 Worker 池 | `flask_app.py` L996-L998 | `worker_pool.start_workers(n_workers, ...)` |
| ⑦ 启动后台线程 | `flask_app.py` L1004-L1021 | ticker、notification_runner、version_checker |
| ⑧ 启动 HTTP 服务 | `main()` L686-L697 | `socketio.run(app, ...)` 或 `app.run(...)` |

---

## 2. 配置装载层：三层合并模型

配置的最终值由三层叠加而成，优先级从高到低：

```
环境变量 (ENV)  >  配置文件 (JSON)  >  默认值 (代码硬编码)
```

但**并非所有配置项都严格遵循此顺序**，不同子系统有各自的解析策略，详见第 3 节。

### 2.1 第一层：代码默认值

#### 全局默认值 — App.model.base_config

定义于 [model/App.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/App.py#L20-L85)，是一个 dict 类的类属性：

```python
base_config = {
    'watching': {},
    'settings': {
        'headers': {},
        'requests': {
            'time_between_check': {'weeks': None, 'days': None, 'hours': 3, ...},
            'timeout': int(getenv("DEFAULT_SETTINGS_REQUESTS_TIMEOUT", "45")),
            'workers': int(getenv("DEFAULT_SETTINGS_REQUESTS_WORKERS", "5")),
            'default_ua': { 'html_requests': getenv("DEFAULT_SETTINGS_HEADERS_USERAGENT", ...) },
        },
        'application': {
            'fetch_backend': getenv("DEFAULT_FETCH_BACKEND", "html_requests"),
            'notification_body': default_notification_body,
            'notification_format': default_notification_format,
            ...
        }
    }
}
```

**关键特征**：`base_config` 中的默认值在**类定义时**就已求值，部分字段直接内嵌了 `os.getenv()` 调用——这意味着环境变量可以在默认值层面就介入优先级链。

#### Watch 级默认值 — watch_base.\_\_init\_\_

定义于 [model/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L174-L306)，每个 Watch 对象创建时自动填充：

```python
self.update({
    'fetch_backend': 'system',          # 'system' 意味着回退到全局设置
    'notification_format': USE_SYSTEM_DEFAULT_NOTIFICATION_FORMAT_FOR_WATCH,
    'filter_failure_notification_send': strtobool(os.getenv('FILTER_FAILURE_NOTIFICATION_SEND_DEFAULT', 'True')),
    'time_between_check_use_default': True,
    ...
})
```

**关键设计**：很多字段默认为 `None` 或 `'system'`，显式表达"未设置，使用上级配置"的语义。

#### LLM 默认值 — LLMSettings (Pydantic)

定义于 [model/LLMSettings.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/LLMSettings.py#L21-L56)，这是项目中**唯一使用 Pydantic** 的配置模型：

```python
class LLMSettings(BaseModel):
    model_config = ConfigDict(extra='forbid')   # 严格模式，拒绝未声明的字段
    enabled: bool = True
    thinking_budget: int = 0
    max_summary_tokens: int = 3000
    model: str = ''
    api_key: str = ''
    ...
```

读取时通过 `LLMSettings.model_validate(stored_dict_or_empty)` 水合，写入时通过 `.model_dump()` 序列化。存储层始终是纯 dict，Pydantic 仅作为验证/类型层。

### 2.2 第二层：配置文件 (JSON)

#### 存储结构

```
{datastore_path}/
├── changedetection.json        ← 全局设置（不含 watching/tags）
├── proxies.json                ← 外部代理列表
├── headers.txt                 ← 全局额外 HTTP 请求头
├── {uuid}/
│   ├── watch.json              ← 单个 Watch 配置
│   ├── tag.json                ← 单个 Tag 配置
│   ├── restock_diff.json      ← restock 处理器专用配置
│   ├── headers.txt             ← Watch 级额外 HTTP 请求头
│   └── ...                     ← 历史快照、截图等
└── {uuid}/
    ├── tag.json                ← Tag 持久化
    └── ...
```

#### 加载流程

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L190) `reload_state()` 的三种场景：

1. **已有 changedetection.json**：`_load_state()` → `_load_settings()` + `_load_watches()` + `_load_tags()` → `_rehydrate_watches()` + `_rehydrate_tags()` → `run_updates()`
2. **仅有 url-watches.json（旧格式）**：先从旧格式加载 → 运行 update_26 迁移 → 重新 `_load_state()`
3. **全新安装**：`init_fresh_install()` → 生成 app_guid/rss_access_token → 添加默认 watch → `_save_settings()` → `_load_state()`

#### 合并机制 — `_apply_settings()`

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L102-L134) 中 `_apply_settings()` 使用 `dict.update()` 将 JSON 数据合并到默认值之上：

```python
def _apply_settings(self, settings_data):
    if 'headers' in settings_data['settings']:
        self.__data['settings']['headers'].update(settings_data['settings']['headers'])
    if 'requests' in settings_data['settings']:
        self.__data['settings']['requests'].update(settings_data['settings']['requests'])
    if 'application' in settings_data['settings']:
        self.__data['settings']['application'].update(settings_data['settings']['application'])
    if 'watching' in settings_data:
        self.__data['watching'].update(settings_data['watching'])
```

**重要**：这是浅层 `dict.update()`，意味着 JSON 文件中的键会**覆盖**默认值中的同名键，但不会递归合并嵌套 dict（整个嵌套 dict 被替换）。

#### Watch 反序列化（Rehydration）

1. [file_saving_datastore.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/file_saving_datastore.py#L273) `load_all_watches()` 扫描所有 `{uuid}/watch.json`
2. 每个 watch_dict 传入 [store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L321-L335) `rehydrate_entity()`，构造 `Watch.model(default=watch_dict)`
3. Watch.\_\_init\_\_ 先填入默认值，再 `self.update(kw['default'])` 用 JSON 数据覆盖——JSON 中的字段**覆盖**默认值

#### Tag 加载的双重来源

1. `_apply_settings()` 从 changedetection.json 的 `settings.application.tags` 读取
2. `_load_tags()` 从独立的 `{uuid}/tag.json` 文件读取，**后者覆盖前者**

```python
# store/__init__.py _load_tags():
if tags:
    self.__data['settings']['application']['tags'].update(tags)
```

### 2.3 第三层：环境变量 (ENV)

环境变量是**最高优先级**的配置源，但其介入方式不统一，分为以下几类：

#### A. 启动时一次性解析（CLI 级覆盖）

| 环境变量 | 被谁读取 | 覆盖谁 | 默认值 |
|----------|----------|--------|--------|
| `LISTEN_HOST` | `main()` | CLI `-h` 参数 | `"0.0.0.0"` |
| `PORT` | `main()` | CLI `-p` 参数 | `5000` |
| `LOGGER_LEVEL` | `main()` | CLI `-l` 参数 | `"DEBUG"` |
| `SSL_CERT_FILE` / `SSL_PRIVKEY_FILE` | `main()` | CLI `-s` 参数 | `'cert.pem'` / `'privkey.pem'` |
| `SALTED_PASS` | `User.check_password()` | JSON 存储的密码 | — |

#### B. 在默认值定义时嵌入（App.model 层面）

这些环境变量通过 `getenv()` 嵌入 `base_config`，若环境变量有值则**替代硬编码默认值**，但会被后续 JSON 文件的 `update()` 覆盖：

| 环境变量 | 位置 | 默认值 |
|----------|------|--------|
| `DEFAULT_SETTINGS_REQUESTS_TIMEOUT` | App.py | `"45"` |
| `DEFAULT_SETTINGS_REQUESTS_WORKERS` | App.py | `"5"` |
| `DEFAULT_SETTINGS_HEADERS_USERAGENT` | App.py | Chrome UA 字符串 |
| `DEFAULT_FETCH_BACKEND` | App.py | `"html_requests"` |

**优先级**：`ENV` > `硬编码默认值`，但 JSON 配置文件可覆盖这两者。

#### C. 运行时动态解析（每次读取时判定）

这类环境变量在代码运行过程中每次被读取时实时判定，**始终覆盖 JSON 存储值**：

| 环境变量 | 读取位置 | 行为 |
|----------|----------|------|
| `FETCH_WORKERS` | flask_app.py L996, worker_pool.py | 覆盖 `settings.requests.workers` |
| `BASE_URL` | store/\_\_init\_\_.py L590-L591 | 覆盖 `settings.application.base_url`（仅当 JSON 中未设置时） |
| `SALTED_PASS` | flask_app.py L533 | 覆盖 JSON 存储的密码 |
| `HIDE_REFERER` | flask_app.py L643 | 控制响应头 |
| `USE_X_SETTINGS` | flask_app.py L656 | 启用 ProxyFix 中间件 |
| `FLASK_SERVER_NAME` | flask_app.py L118-L119 | 设置 Flask SERVER_NAME |
| `FLASK_ENABLE_COMPRESSION` | flask_app.py L100 | 启用 Flask-Compress |
| `DISABLE_VERSION_CHECK` | flask_app.py L1019 | 禁用版本检查 |
| `NOTIFICATION_WORKERS` | flask_app.py L1007 | 通知 worker 数量 |
| `MINIMUM_SECONDS_RECHECK_TIME` | flask_app.py L1117 | 最小重检间隔 |
| `ENABLE_NO_PROXY_OPTION` | store/\_\_init\_\_.py L850, L868 | 在代理列表添加 "No proxy" 选项 |
| `PAGE_WATCH_LIMIT` | store/\_\_init\_\_.py L739 | Watch 数量上限 |

#### D. LLM 子系统的专属环境变量覆盖

LLM 子系统有**最规范**的环境变量覆盖机制，在 [llm/evaluator.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/llm/evaluator.py#L234-L257) 的 `get_llm_config()` 中实现：

```python
def get_llm_config(datastore) -> dict | None:
    # 1. 环境变量优先
    env_model = os.getenv('LLM_MODEL', '').strip()
    if env_model:
        return {
            'model': env_model,
            'api_key': os.getenv('LLM_API_KEY', '').strip(),
            'api_base': os.getenv('LLM_API_BASE', '').strip(),
        }
    # 2. 回退到 datastore 设置（UI 配置）
    cfg = datastore.data['settings']['application'].get('llm') or {}
    if not cfg.get('model'):
        return None
    return cfg
```

| 环境变量 | 覆盖对象 |
|----------|----------|
| `LLM_MODEL` | `settings.application.llm.model` |
| `LLM_API_KEY` | `settings.application.llm.api_key` |
| `LLM_API_BASE` | `settings.application.llm.api_base` |
| `LLM_FEATURES_DISABLED` | 全局禁用 LLM 功能 |
| `LLM_MAX_INPUT_CHARS` | `settings.application.llm.max_input_chars` |
| `LLM_TOKEN_BUDGET_MONTH` | `settings.application.llm.token_budget_month` |
| `LLM_TIMEOUT` | LLM 客户端超时（默认 60s） |

#### E. 内容抓取器相关的环境变量

| 环境变量 | 读取位置 | 行为 |
|----------|----------|------|
| `PLAYWRIGHT_DRIVER_URL` | content_fetchers/\_\_init\_\_.py | 选择 Playwright 或 Selenium 作为浏览器抓取器 |
| `FAST_PUPPETEER_CHROME_FETCHER` | content_fetchers/\_\_init\_\_.py | 使用直接 Puppeteer 而非 Playwright |
| `SCREENSHOT_MAX_HEIGHT` | content_fetchers/\_\_init\_\_.py | 截图最大高度 |
| `SCREENSHOT_CHUNK_HEIGHT` | content_fetchers/\_\_init\_\_.py | 拼接阈值 |

---

## 3. 三级配置解析链（Watch → Tag → Global）

### 3.1 链式解析的概念模型

代码注释中反复提到的"梦想架构"是三级解析链：

```
Watch 自身设置  →  Tag/Group 设置  →  Global 设置
```

即：如果 Watch 上某字段为 `None`/`'system'`，则向上查找关联的 Tag，若 Tag 也未覆盖，最终回退到全局默认值。

### 3.2 当前实现：手动散落式解析

当前**没有**统一的解析函数，各消费方自行实现回退逻辑：

#### fetch_backend 解析

[Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L357-L389)：

```python
@property
def get_fetch_backend(self):
    if self.is_pdf:
        return 'html_requests'        # 特殊情况：PDF 始终用 requests
    return self.get('fetch_backend')   # 返回 watch 自身值，可能是 'system'
```

而在消费方（如 [Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L400-L402) `fetcher_supports_screenshots`）：

```python
fetcher_name = self.get_fetch_backend
if not fetcher_name or fetcher_name == 'system':
    fetcher_name = self._datastore['settings']['application'].get('fetch_backend', 'html_requests')
```

#### time_between_check 解析

[ticker_thread](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1196-L1221)：

```python
if watch.get('time_between_check_use_default'):
    time_schedule_limit = datastore.data['settings']['requests'].get('time_schedule_limit', {})
else:
    time_schedule_limit = watch.get('time_schedule_limit')

threshold = recheck_time_system_seconds if watch.get('time_between_check_use_default') else watch.threshold_seconds()
```

#### history_snapshot_max_length 解析

[Watch.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/Watch.py#L723)：

```python
maxlen = self.get('history_snapshot_max_length') or self.get_global_setting('application', 'history_snapshot_max_length')
```

#### 通知配置回退

[flask_app.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/flask_app.py#L1079-L1086) notification_runner：

```python
if not n_object.get('notification_body') and datastore.data['settings']['application'].get('notification_body'):
    n_object['notification_body'] = datastore.data['settings']['application'].get('notification_body')
# notification_title, notification_format 同理
```

### 3.3 Tag 覆盖 Watch 的机制

Tag 通过 `watch.get('tags')` 列表关联到 Watch。[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L936-L945) `get_tag_overrides_for_watch()` 可获取 Tag 对某属性的覆盖值。

目前 Tag 主要在 restock_diff 处理器中使用 override 机制：

```python
# processors/restock_diff/processor.py (示意):
for tag_uuid in watch.get('tags'):
    tag = datastore['settings']['application']['tags'][tag_uuid]
    if tag.get('overrides_watch'):
        restock_settings = tag.get('restock_settings', {})
        break
```

### 3.4 BASE_URL 的特殊三级解析

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L583-L597) `data` 属性：

```python
active_base_url = BASE_URL_NOT_SET_TEXT
if self.__data['settings']['application'].get('base_url'):
    active_base_url = self.__data['settings']['application'].get('base_url')   # JSON 配置
elif os.getenv('BASE_URL'):
    active_base_url = os.getenv('BASE_URL')                                   # 环境变量回退
```

**注意**：这里 JSON 优先于环境变量，与 LLM 子系统的优先级方向**相反**。

---

## 4. 全局共享对象与协同关系

### 4.1 核心对象图

```
main() 中的全局变量
├── datastore: ChangeDetectionStore   ← 单例，所有配置与运行时状态的真相来源
│   ├── __data: App.model (dict)      ← 内存中的完整数据树
│   │   ├── watching: {uuid: Watch}   ← 所有 Watch 对象
│   │   ├── settings.headers: dict
│   │   ├── settings.requests: dict
│   │   └── settings.application: dict ← 含 tags, llm, notification 等
│   ├── datastore_path: str
│   ├── lock: threading.Lock
│   └── generic_definition: Watch     ← 新建 watch 时的模板
│
└── app: Flask (由 changedetection_app 返回)
    ├── config['DATASTORE']: datastore ← Flask 可访问 datastore 引用
    ├── config['datastore_path']: str
    ├── config['exit']: Event          ← 优雅关闭信号
    ├── config['batch_mode']: bool
    └── 全局变量:
        ├── datastore (模块级)         ← flask_app.py 内的模块全局
        ├── update_q: RecheckPriorityQueue
        └── notification_q: NotificationQueue
```

### 4.2 对象间的引用关系

```
Watch 对象
  ├── _datastore → 指向 ChangeDetectionStore（共享引用，不深拷贝）
  ├── _datastore_path → datastore_path
  └── data_dir → os.path.join(datastore_path, uuid)

Tag 对象
  ├── _datastore → 同上
  └── _datastore_path → 同上

ChangeDetectionStore
  └── __data.settings.application.tags[uuid] → Tag 对象
  └── __data.watching[uuid] → Watch 对象

Flask App
  └── config['DATASTORE'] → ChangeDetectionStore
  └── flask_app.datastore (模块全局) → ChangeDetectionStore
```

### 4.3 Watch 对象的 get_global_setting() 方法

[watch_base](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/model/__init__.py#L583-L605) 提供了从 Watch 内部向上查找全局设置的标准方法：

```python
def get_global_setting(self, *path):
    if not self._datastore:
        return None
    try:
        value = self._datastore['settings']
        for key in path:
            value = value[key]
        return value
    except (KeyError, TypeError):
        return None
```

这是 Watch → Global 回退链的通用实现。

### 4.4 DataStore.data 属性的动态计算

[store/\_\_init\_\_.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/__init__.py#L582-L597) 的 `data` 属性**不是**简单返回 `__data`，而是在每次访问时动态注入 `active_base_url`：

```python
@property
def data(self):
    active_base_url = BASE_URL_NOT_SET_TEXT
    if self.__data['settings']['application'].get('base_url'):
        active_base_url = ...
    elif os.getenv('BASE_URL'):
        active_base_url = os.getenv('BASE_URL')
    d = self.__data
    d['settings']['application']['active_base_url'] = active_base_url.strip('" ')
    return d
```

这意味着 `datastore.data` 的返回值**带有副作用**（修改了 `__data` 中的 `active_base_url`）。

---

## 5. 优先级总结速查表

### 5.1 全局配置（App-level）

| 配置项 | 默认值来源 | JSON 覆盖 | ENV 覆盖 | 最终解析位置 |
|--------|-----------|-----------|----------|-------------|
| `requests.timeout` | `App.base_config` (或 `DEFAULT_SETTINGS_REQUESTS_TIMEOUT`) | ✅ changedetection.json | ⚠️ 仅在默认值层 | `data['settings']['requests']['timeout']` |
| `requests.workers` | `App.base_config` (或 `DEFAULT_SETTINGS_REQUESTS_WORKERS`) | ✅ | ✅ `FETCH_WORKERS` 实时覆盖 | flask_app.py L996 |
| `application.fetch_backend` | `App.base_config` (或 `DEFAULT_FETCH_BACKEND`) | ✅ | ❌ | `data['settings']['application']['fetch_backend']` |
| `application.base_url` | `None` | ✅ 优先 | ✅ `BASE_URL` 回退 | store data 属性 |
| `application.password` | `False` | ✅ | ✅ `SALTED_PASS` 优先 | User.check_password() |
| `application.notification_*` | `notification/__init__.py` | ✅ | ❌ | `data['settings']['application']` |
| `application.llm.*` | `LLMSettings` Pydantic 默认值 | ✅ | ✅ `LLM_MODEL/KEY/BASE` 优先 | evaluator.py get_llm_config() |
| `application.llm.max_input_chars` | `LLMSettings` (100000) | ✅ | ✅ `LLM_MAX_INPUT_CHARS` 优先 | evaluator.py _get_max_input_chars() |
| `application.llm.token_budget_month` | `LLMSettings` (0) | ✅ | ✅ `LLM_TOKEN_BUDGET_MONTH` 优先 | evaluator.py get_global_token_budget_month() |

### 5.2 Watch 级配置

| 配置项 | Watch 默认值 | Tag 覆盖 | Global 回退 | 回退标志 |
|--------|-------------|----------|------------|---------|
| `fetch_backend` | `'system'` | ❌ (未来支持) | ✅ `application.fetch_backend` | 值为 `'system'` |
| `time_between_check` | `time_between_check_use_default=True` | ❌ | ✅ `requests.time_between_check` | `time_between_check_use_default` |
| `notification_format` | `'System default'` | ✅ | ✅ `application.notification_format` | 值为 `'System default'` |
| `notification_body/title` | `None` | ✅ | ✅ `application.notification_*` | 值为 `None` |
| `history_snapshot_max_length` | `None` | ❌ | ✅ `application.history_snapshot_max_length` | 值为 `None` |
| `use_page_title_in_list` | `None` | ❌ | ✅ `application.ui.use_page_title_in_list` | 值为 `None` |
| `filter_failure_notification_send` | `True` (或 ENV) | ❌ | ❌ | 无回退 |

### 5.3 运行时行为覆盖

| 环境变量 | 覆盖时机 | 特点 |
|----------|----------|------|
| `FETCH_WORKERS` | 每次 Worker 启动/健康检查时 | 动态，始终优先于 JSON |
| `NOTIFICATION_WORKERS` | App 初始化时 | 固定 |
| `MINIMUM_SECONDS_RECHECK_TIME` | ticker 每次循环 | 动态 |
| `SALTED_PASS` | 每次密码校验 | 动态，完全绕过 JSON 存储密码 |
| `BASE_URL` | 每次访问 datastore.data | 动态，仅 JSON 无值时才生效 |
| `HIDE_REFERER` | 每次请求 | 动态 |
| `LLM_MODEL/KEY/BASE` | 每次 LLM 调用 | 动态，完全绕过 JSON LLM 配置 |
| `PLAYWRIGHT_DRIVER_URL` | 模块导入时 | 一次性，决定浏览器引擎选择 |
| `DISABLE_VERSION_CHECK` | App 初始化时 | 一次性 |
| `PAGE_WATCH_LIMIT` | 每次 add_watch | 动态 |
| `TESTING_SHUTDOWN_AFTER_DATASTORE_LOAD` | main() DataStore 加载后 | CI/CD 专用 |

---

## 6. Schema 迁移与配置演进

配置结构通过 schema version 控制向前兼容。定义于 [store/updates.py](file:///d:/fz/0601-2/solo-dogfeeding/code/4-changedetection.io/changedetectionio/store/updates.py) 的 `DatastoreUpdatesMixin`，当前最新版本为 **update_32**。

关键迁移：
- **update_26**: 单体 url-watches.json → 独立 watch.json 文件
- **update_29**: Tag 迁移到独立 tag.json 文件
- **update_30**: restock_settings 迁出 watch.json 到 restock_diff.json
- **update_31**: 扁平 `llm_*` 键折叠到嵌套 `application.llm.*`
- **update_32**: 清理废弃的 `max_tokens_per_check`，重命名 `max_tokens_cumulative`

每次迁移前自动创建 tarball 备份（`before-update-N-timestamp.tar.gz`）。

---

## 7. 关键设计洞察与不一致性

### 7.1 环境变量优先级方向不统一

| 子系统 | ENV vs JSON 优先级 |
|--------|-------------------|
| LLM 配置 | **ENV > JSON**（环境变量完全覆盖） |
| BASE_URL | **JSON > ENV**（JSON 有值则 ENV 被忽略） |
| SALTED_PASS | **ENV > JSON**（ENV 存在时完全绕过 JSON 密码） |
| Workers | **ENV > JSON**（FETCH_WORKERS 实时覆盖） |
| 超时/UA | **ENV 在默认值层**（被 JSON update 覆盖） |

### 7.2 dict.update() 的浅合并风险

`_apply_settings()` 使用浅层 `dict.update()`，这意味着对于嵌套结构（如 `settings.application.ui`），如果 JSON 中只存储了部分子键，整个嵌套 dict 会被替换而非合并。实际运行中因 `base_config` 已包含完整结构且 JSON 通常也写入完整结构，此问题较少暴露。

### 7.3 Watch → Tag → Global 解析散落

代码中缺少统一的 `resolve_setting(watch, key)` 函数，每个消费方自行实现回退逻辑。这导致：
- 不同位置可能实现不同的回退路径
- Tag 覆盖仅在 restock_diff 处理器中被真正使用
- 新增配置项时容易遗漏回退逻辑

### 7.4 Pydantic 仅用于 LLM

`LLMSettings` 是唯一使用 Pydantic 的模型，具有 `extra='forbid'` 的严格验证。其余所有配置（Watch、App、Tag）仍为 dict 继承，缺乏运行时类型检查。
