# 代理选择策略完整说明

## 概述

本文档详细说明了 changedetection.io 项目中为每次抓取选择代理的整套策略，包括全局代理池配置、watch 级别覆盖、no-proxy 判定顺序、请求库与浏览器 fetcher 之间的代理传递，以及失败回退机制。

---

## 1. 全局代理池配置

### 1.1 代理池的实际来源

全局代理池（`proxy_list`）仅由以下两个来源组成，定义于 `store/__init__.py:825-853`：

#### 1.1.1 本地配置文件来源 (proxies.json)
- 从 `datastore_path/proxies.json` 文件加载
- 支持 orjson (优先) 和标准 json 格式
- 这是主要的代理配置来源

#### 1.1.2 UI 配置来源 (extra_proxies)
- 从 `settings['requests']['extra_proxies']` 中读取
- 每个代理包含 `proxy_name` 和 `proxy_url` 字段
- 键名格式：`ui-{index}{proxy_name}`
- 通过 Web 界面设置的额外代理

#### 1.1.3 No-Proxy 选项的注入时机
**No-proxy 不是代理池的来源，而是在代理池构建完成后自动注入的选项**：

在 `store/__init__.py:850-851` 中：
```python
if proxy_list and strtobool(os.getenv('ENABLE_NO_PROXY_OPTION', 'True')):
    proxy_list["no-proxy"] = {'label': "No proxy", 'url': ''}
```

- 注入条件：代理池非空 + `ENABLE_NO_PROXY_OPTION=True`（默认启用）
- 注入时机：在加载完 proxies.json 和 extra_proxies 之后
- 该选项的 URL 为空字符串，表示不使用代理

> **重要**：