# 敏感信息遮蔽策略分析报告

## 一、概述

本报告分析 changedetection.io 项目中密码与敏感凭据在配置、通知和导出三个环节的遮蔽策略，识别统一规则与未覆盖角落。

---

## 二、敏感字段清单

### 2.1 系统级敏感字段
| 字段名 | 存储位置 | 用途 |
|--------|----------|------|
| `password` | `settings.application.password` | 登录密码（加盐哈希） |
| `api_access_token` | `settings.application.api_access_token` | API 访问令牌 |
| `rss_access_token` | `settings.application.rss_access_token` | RSS 访问令牌 |
| `llm.api_key` | `settings.application.llm.api_key` | LLM 服务商 API 密钥 |
| `notification_urls` | `settings.application.notification_urls` | Apprise 通知 URL（可能内嵌密码） |
| `extra_proxies[].proxy_url` | `settings.requests.extra_proxies` | 代理 URL（可能内嵌密码） |
| `headers` | `settings.headers` | HTTP 请求头（可能包含 Authorization） |

### 2.2 Watch 级敏感字段
| 字段名 | 存储位置 | 用途 |
|--------|----------|------|
| `notification_urls` | `watch.notification_urls` | 单 Watch 通知 URL |
| `headers` | `watch.headers` | 单 Watch HTTP 请求头 |
| `proxy` | `watch.proxy` | 单 Watch 代理配置 |

---

## 三、配置环节遮蔽策略

### 3.1 Web 表单渲染

#### 3.1.1 LLM API Key
- **实现位置**: [blueprint/settings/__init__.py:36-41](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/settings/__init__.py#L36-L41)
- **策略**: 使用 WTForms `PasswordField`，在 GET 请求时显式清空值：
  ```python
  default['llm'] = LLMSettings.model_validate(
      datastore.data['settings']['application'].get('llm') or {}
  ).model_dump()
  default['llm']['api_key'] = ''  # 显式清空
  ```
- **表单提交处理**: [blueprint/settings/__init__.py:108-111](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/settings/__init__.py#L108-L111)
  - 空值提交时保留原有密钥，不清空
  ```python
  if not (llm_form_input.get('api_key') or '').strip():
      llm_form_input.pop('api_key', None)
  ```

#### 3.1.2 登录密码
- **实现位置**: [forms.py:92-100](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/forms.py#L92-L100), [blueprint/settings/__init__.py:86-88](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/settings/__init__.py#L86-L88)
- **策略**: 
  - 使用自定义 `SaltyPasswordField`
  - 空值或 `False` 提交时，从更新字典中删除该键，保留原有密码
  ```python
  if 'password' in app_update and not app_update['password']:
      del (app_update['password'])
  ```

#### 3.1.3 API Access Token
- **实现位置**: [blueprint/settings/__init__.py:263-270](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/settings/__init__.py#L263-L270)
- **策略**: 
  - 设置页面显示完整 token（[settings.html:237](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/settings/__init__.py#L237)）
  - 通过 `/reset-api-key` 接口单独重置

### 3.2 环境变量覆盖
- LLM 配置可通过环境变量 `LLM_API_KEY`、`LLM_MODEL`、`LLM_API_BASE` 覆盖
- 环境变量优先级高于 datastore 存储
- 环境变量配置的字段在 UI 中标记为只读

---

## 四、API 响应环节遮蔽策略

### 4.1 内部字段过滤机制

#### 4.1.1 `strip_internal_api_fields()`
- **实现位置**: [api/__init__.py:153-170](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/api/__init__.py#L153-L170)
- **功能**: 过滤两类字段：
  1. `__` 前缀的临时/内部字段（如 `__check_status`）
  2. `SYSTEM_MANAGED_NON_SPEC_FIELDS` 中定义的系统管理字段

#### 4.1.2 `SYSTEM_MANAGED_NON_SPEC_FIELDS`
- **定义位置**: [model/schema_utils.py:21-32](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/model/schema_utils.py#L21-L32)
- **过滤列表**:
  ```python
  SYSTEM_MANAGED_NON_SPEC_FIELDS = frozenset({
      'last_check_status',
      'last_filter_config_hash',
      'restock',
      '_llm_result',
      '_llm_intent',
      '_llm_change_summary',
      'llm_prefilter',
      'llm_evaluation_cache',
      'llm_last_tokens_used',
      'llm_tokens_used_cumulative',
  })
  ```

#### 4.1.3 只读字段保护
- **实现位置**: [api/Watch.py:200-214](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/api/Watch.py#L200-L214)
- **策略**: PUT 请求时过滤 `readonly` 字段和 `@property` 计算属性

### 4.2 LLM API Key 专项保护

#### 4.2.1 安全测试覆盖
- **测试文件**: [tests/test_llm_api_key_security.py](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/tests/test_llm_api_key_security.py)
- **测试覆盖的端点**:
  - `GET /api/v1/watch/<uuid>` - 单 Watch 查询
  - `GET /api/v1/watch` - Watch 列表
  - `PUT /api/v1/watch/<uuid>` - Watch 更新响应
  - `GET /api/v1/tag/<uuid>` - 单 Tag 查询
  - `GET /api/v1/tags` - Tag 列表
  - `GET /api/v1/systeminfo` - 系统信息
  - `GET/POST/PUT /api/v1/notifications` - 通知配置
  - `GET /api/v1/search` - 搜索
  - `GET /api/v1/full-spec` - OpenAPI 规范
  - 设置页面 HTML 源码

#### 4.2.2 测试方法（"金丝雀"测试）
```python
CANARY_KEY = 'sk-CANARY-SECRET-DO-NOT-EXPOSE-12345'
# 注入测试密钥后，检查所有 API 响应是否包含该密钥
def _key_in_response(response, key=CANARY_KEY) -> bool:
    body = response.data.decode('utf-8', errors='replace')
    return key in body
```

#### 4.2.3 凭据泄露防护（GHSA-g36r-fm2p-87xm）
- **场景**: 防止 CSRF 攻击将存储的 API key 发送到攻击者控制的服务器
- **防护逻辑**: 当 `api_base` 与存储值不匹配时，必须显式提供 `api_key`，否则拒绝请求
- **测试**: [tests/test_llm_api_key_security.py:487-545](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/tests/test_llm_api_key_security.py#L487-L545)

#### 4.2.4 SSRF 防护（GHSA-jrxm-qjfh-g54f）
- **场景**: 防止通过 LLM `api_base` 指向内网/本地服务进行 SSRF 攻击
- **防护逻辑**: 默认拒绝 `127.0.0.1`、`localhost`、`10.0.0.0/8`、`192.168.0.0/16`、`169.254.169.254` 等地址
- **绕过方式**: 设置环境变量 `ALLOW_IANA_RESTRICTED_ADDRESSES=true`

### 4.3 通知 URL API 响应
- **实现位置**: [api/Notifications.py:10-19](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/api/Notifications.py#L10-L19)
- **当前策略**: **无遮蔽**，直接返回原始 URL 列表
  ```python
  return {
      'notification_urls': notification_urls,
  }, 200
  ```
- **风险**: Apprise URL 可能包含明文密码（如 `mailto://user:pass@host`、`slack://xoxb-TOKEN@CHANNEL`）

---

## 五、通知环节遮蔽策略

### 5.1 通知 URL 处理

#### 5.1.1 通知发送流程
- **实现位置**: [notification/handler.py:416-510](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/notification/handler.py#L416-L510)
- **日志记录**:
  ```python
  logger.info(f">> Process Notification: AppRise start notifying '{url}'")
  ```
  - **风险**: 日志中直接输出完整 URL，可能包含密码

#### 5.1.2 通知调试日志
- **实现位置**: [flask_app.py:1098-1107](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/flask_app.py#L1098-L1107)
- **记录内容**:
  ```python
  notification_debug_log += ["{} - SENDING - {}".format(
      now.strftime("%c"), 
      json.dumps(sent_obj)  # 包含完整的 url、title、body
  )]
  ```
- **风险**: 
  - `sent_obj['url']` 包含完整通知 URL（可能有密码）
  - `sent_obj['original_context']` 包含完整的通知上下文
  - 日志保留最近 100 条，可通过 `/settings/notification-logs` 查看

### 5.2 Apprise 库日志捕获
- **实现位置**: [notification/handler.py:416-507](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/notification/handler.py#L416-L507)
- **日志级别**: DEBUG
- **风险**: Apprise 内部日志可能包含敏感信息

---

## 六、导出/备份环节遮蔽策略

### 6.1 备份创建流程
- **实现位置**: [blueprint/backups/__init__.py:16-91](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/backups/__init__.py#L16-L91)
- **备份内容**:
  1. `changedetection.json` - 全局配置（包含所有敏感字段明文）
  2. `{uuid}/watch.json` - 每个 Watch 的配置
  3. `{uuid}/tag.json` - 每个 Tag 的配置
  4. `{uuid}/*` - 历史快照、截图等数据
  5. `url-list.txt` / `url-list-with-tags.txt` - URL 列表

### 6.2 当前策略
- **无任何遮蔽**，所有敏感信息以明文形式存储在备份 ZIP 中
- **备份下载**: [blueprint/backups/__init__.py:144-166](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/backups/__init__.py#L144-L166)
  - 需要登录认证
  - 文件名验证防止路径遍历

### 6.3 导入流程
- **实现位置**: [blueprint/imports/importer.py](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/imports/importer.py)
- **行为**: 直接导入备份中的所有配置，包括敏感信息

---

## 七、未覆盖角落与风险分析

### 7.1 高风险区域

#### 🔴 风险 1: 通知 URL 密码明文泄露
- **位置**: API 响应、日志、调试日志
- **场景**: 
  - `GET /api/v1/notifications` 返回完整 URL
  - `notification_debug_log` 记录完整 `sent_obj`
  - Apprise 日志输出完整 URL
- **示例风险 URL**:
  ```
  mailto://user:password@smtp.example.com
  slack://xoxb-1234567890-abcdef@T00000000/B00000000
  discord://WEBHOOK_ID/WEBHOOK_TOKEN
  ```
- **影响**: 拥有 API 访问权限或日志访问权限的用户可获取通知服务凭据

#### 🔴 风险 2: 备份文件包含所有敏感信息明文
- **位置**: 备份 ZIP 文件
- **场景**: 备份文件泄露或未妥善保管
- **影响**: 所有 API 密钥、密码、通知凭据全部泄露

#### 🔴 风险 3: 代理 URL 密码泄露
- **位置**: `proxy_list` 属性、API 响应
- **场景**: 代理 URL 包含认证信息
- **示例**: `http://user:pass@proxy.example.com:8080`

#### 🟡 风险 4: HTTP 头认证信息泄露
- **位置**: `headers.txt`、`settings.headers`、`watch.headers`
- **场景**: 
  - `Authorization: Bearer <token>` 头存储在配置中
  - 备份、API 响应、日志中可能包含
- **防护**: 部分在 `strip_internal_api_fields` 中未被过滤

#### 🟡 风险 5: 系统日志敏感信息
- **位置**: 应用日志（loguru）
- **场景**:
  ```python
  logger.info(f">> Process Notification: AppRise start notifying '{url}'")
  logger.error(f"Notification worker ... Error {str(e)}")
  ```
- **风险**: 错误信息可能包含敏感上下文

### 7.2 现有防护机制的局限性

| 机制 | 覆盖范围 | 局限性 |
|------|----------|--------|
| `PasswordField` 清空 | LLM API Key、登录密码 | 仅 Web 表单，不覆盖 API |
| `strip_internal_api_fields` | LLM 运行时状态、内部字段 | 不覆盖 notification_urls、headers、proxy_url |
| LLM API Key 专项测试 | LLM API Key | 仅覆盖 LLM 密钥，不覆盖其他敏感字段 |
| 登录认证 | 备份下载、设置页面 | 不保护备份文件本身的内容 |
| SSRF 防护 | LLM api_base | 不覆盖通知 URL、代理 URL |
| 凭据泄露防护 | LLM api_key + api_base 组合 | 仅覆盖 LLM 场景 |

---

## 八、统一遮蔽规则建议

### 8.1 建议新增的敏感字段列表
```python
SENSITIVE_FIELDS = frozenset({
    # 系统级
    'api_access_token',
    'rss_access_token',
    'password',
    # LLM
    'llm.api_key',
    # 通知
    'notification_urls',
    # 代理
    'proxy_url',
    'extra_proxies',
    # HTTP 头
    'headers',
})
```

### 8.2 URL 密码遮蔽算法
```python
import re
from urllib.parse import urlparse, urlunparse

def mask_url_password(url: str) -> str:
    """
    遮蔽 URL 中的密码部分
    例: http://user:pass@host.com -> http://user:****@host.com
    """
    try:
        parsed = urlparse(url)
        if parsed.password:
            # 重建 netloc，替换密码
            netloc = parsed.hostname
            if parsed.username:
                netloc = f"{parsed.username}:****"
                if parsed.hostname:
                    netloc += f"@{parsed.hostname}"
            if parsed.port:
                netloc += f":{parsed.port}"
            
            masked = parsed._replace(netloc=netloc)
            return urlunparse(masked)
        return url
    except Exception:
        return url
```

### 8.3 建议的遮蔽规则

#### 规则 1: API 响应统一过滤
- **应用位置**: 所有 API 端点返回前
- **处理方式**:
  - `notification_urls`: 对每个 URL 应用密码遮蔽
  - `headers`: 移除或遮蔽 `Authorization`、`Proxy-Authorization`、`Cookie` 等头
  - `extra_proxies[].proxy_url`: 应用密码遮蔽
  - `api_access_token`、`rss_access_token`: 可考虑只显示后 4 位

#### 规则 2: 日志统一过滤
- **应用位置**: 所有 `logger.info()`、`logger.error()` 等调用前
- **处理方式**:
  - 通知 URL 先遮蔽再记录
  - 错误信息中可能包含的敏感信息进行过滤

#### 规则 3: 调试日志过滤
- **应用位置**: `notification_debug_log` 写入前
- **处理方式**:
  - `sent_obj['url']` 遮蔽密码
  - 移除 `sent_obj['original_context']` 或进行敏感信息扫描

#### 规则 4: 备份可选遮蔽
- **应用位置**: 备份创建时
- **处理方式**:
  - 新增选项："创建安全备份（遮蔽敏感信息）"
  - 遮蔽后的备份可用于分享但无法直接恢复

---

## 九、代码参考索引

### 核心遮蔽实现
- API 字段过滤: [api/__init__.py:153-170](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/api/__init__.py#L153-L170)
- 系统管理字段定义: [model/schema_utils.py:21-32](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/model/schema_utils.py#L21-L32)
- LLM API Key 表单处理: [blueprint/settings/__init__.py:36-41](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/settings/__init__.py#L36-L41), [108-111](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/settings/__init__.py#L108-L111)

### 安全测试
- LLM API Key 安全测试: [tests/test_llm_api_key_security.py](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/tests/test_llm_api_key_security.py)
- API 内部字段过滤测试: [tests/test_api.py:409-457](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/tests/test_api.py#L409-L457)

### 高风险区域
- 通知 URL API: [api/Notifications.py:10-19](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/api/Notifications.py#L10-L19)
- 通知调试日志: [flask_app.py:1098-1107](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/flask_app.py#L1098-L1107)
- 备份创建: [blueprint/backups/__init__.py:16-91](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/blueprint/backups/__init__.py#L16-L91)
- 通知发送日志: [notification/handler.py:430](file:///d:/fz/0601-2/solo-dogfeeding/code/23-changedetection.io/changedetectionio/notification/handler.py#L430)

---

## 十、总结

### 现有优势
1. ✅ LLM API Key 有完善的专项保护和测试
2. ✅ 内部/临时字段有统一过滤机制
3. ✅ Web 表单密码字段使用 `PasswordField` 不回显
4. ✅ API 访问有 token 认证
5. ✅ LLM 场景有 SSRF 和凭据泄露防护

### 主要缺失
1. ❌ 通知 URL 中的密码无任何遮蔽（API、日志、调试日志）
2. ❌ 代理 URL 中的密码无遮蔽
3. ❌ HTTP 认证头无遮蔽
4. ❌ 备份文件包含所有敏感信息明文
5. ❌ 缺少统一的敏感字段清单和遮蔽工具函数
6. ❌ 系统日志无敏感信息扫描过滤

### 优先级建议
1. **高优先级**: 通知 URL 密码遮蔽（API 响应 + 日志）
2. **高优先级**: 备份文件敏感信息警告/可选遮蔽
3. **中优先级**: 代理 URL 密码遮蔽
4. **中优先级**: HTTP 认证头遮蔽
5. **低优先级**: 统一敏感字段管理工具类
