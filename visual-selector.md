# 可视化选择器（Visual Selector）完整流程分析

## 一、概述

可视化选择器是 changedetection.io 的核心功能之一，允许用户通过直观的可视化界面选择网页上的元素，自动生成内容抽取规则。整个流程涵盖从浏览器抓取、数据回传、覆盖层渲染、用户交互、规则转换到最终的差异检测的完整链路。

---

## 二、浏览器抓取与数据回传阶段

### 2.1 浏览器端元素数据采集

**核心文件**：`changedetectionio/content_fetchers/res/xpath_element_scraper.js`

#### 2.1.1 采集流程

1. **元素遍历策略**
   - 从 `document.body` 开始递归遍历所有子元素
   - 只收集指定类型的可见元素：`div,span,form,table,tbody,tr,td,a,p,ul,li,h1,h2,h3,h4,header,footer,section,article,aside,details,main,nav,summary,button`
   - 过滤条件：`display !== 'none'`、`visibility !== 'hidden'`、`contentVisibility !== 'hidden'`

2. **元素位置与属性采集**
   ```javascript
   size_pos.push({
       xpath: xpath_result,           // 元素的 XPath 或 CSS 选择器
       width: Math.round(bbox['width']),    // 元素宽度
       height: Math.round(bbox['height']),  // 元素高度
       left: Math.floor(bbox['left']),      // 元素左偏移
       top: Math.floor(bbox['top']) + scroll_y,  // 元素顶部偏移（含滚动）
       tagName: element.tagName.toLowerCase(), // 标签名
       tagtype: element.type,      // input 类型
       isClickable: computedStyle.cursor === "pointer", // 是否可点击
       fontSize: computedStyle.getPropertyValue('font-size'),
       fontWeight: computedStyle.getPropertyValue('font-weight'),
       hasDigitCurrency: hasDigitCurrency,  // 是否包含货币数字
       label: label
   });
   ```

3. **选择器生成策略（优先级从高到低）**
   - **策略1**：如果是 `<input>` 且有 `name` 属性，优先使用 `tagName[name="xxx"]`
   - **策略2**：向上遍历直到找到有 ID 的元素，生成 `#id > tag > tag` 形式的 CSS 选择器
   - **策略3**：回退到传统 XPath 生成算法

4. **现有过滤器回显**
   - 对已保存的 `include_filters`，重新在页面上查找并标记 `highlight_as_custom_filter: true`
   - 支持 XPath 和 CSS 选择器两种格式

#### 2.1.2 数据返回格式
```javascript
return JSON.stringify({
    'size_pos': size_pos,      // 元素位置数组
    'browser_width': window.innerWidth  // 浏览器宽度，用于前端缩放
});
```

### 2.2 Playwright 抓取集成

**核心文件**：`changedetectionio/content_fetchers/playwright.py:385-398`

```python
# 在浏览器中执行元素采集脚本
self.xpath_data = await self.page.evaluate(XPATH_ELEMENT_JS, {
    "visualselector_xpath_selectors": visualselector_xpath_selectors,
    "max_height": MAX_TOTAL_HEIGHT
})
```

同时抓取：
- 页面截图：`capture_full_page_async()`
- 页面内容：`await self.page.content()`
- 库存数据：`INSTOCK_DATA_JS`

### 2.3 数据持久化

**核心文件**：`changedetectionio/model/Watch.py:1174-1190`

```python
def save_xpath_data(self, data, as_error=False):
    import json
    import zlib
    
    target_path = os.path.join(str(self.data_dir), "elements.deflate")
    with open(target_path, 'wb') as f:
        if not isinstance(data, str):
            f.write(zlib.compress(json.dumps(data).encode()))
        else:
            f.write(zlib.compress(data.encode()))
```

- 元素数据：`elements.deflate`（zlib 压缩的 JSON）
- 截图文件：`last-screenshot.png`

---

## 三、前端数据获取与覆盖层渲染

### 3.1 数据获取

**核心文件**：`changedetectionio/static/js/visual-selector.js:138-162`

#### 3.1.1 静态资源路由

**核心文件**：`changedetectionio/flask_app.py:794-809`

```python
if group == 'visual_selector_data':
    watch_directory = str(os.path.join(datastore_o.datastore_path, filename))
    if os.path.isfile(os.path.join(watch_directory, "elements.deflate")):
        response = make_response(send_from_directory(watch_directory, "elements.deflate"))
        response.headers['Content-Type'] = 'application/json'
        response.headers['Content-Encoding'] = 'deflate'
```

#### 3.1.2 前端数据加载

```javascript
function bootstrapVisualSelector() {
    $selectorBackgroundElem
        .on('load', () => {
            c = document.getElementById("selector-canvas");
            ctx = c.getContext("2d");
            fetchData();  // 异步获取元素数据
        })
        .attr("src", screenshot_url);
}

function fetchData() {
    $.ajax({
        url: watch_visual_selector_data_url,
        context: document.body
    }).done((data) => {
        selectorData = data;
        sortScrapedElementsBySize();  // 按面积从大到小排序
        setScale();                // 设置缩放比例
        reflowSelector();          // 初始化选择器
    });
}
```

### 3.2 覆盖层渲染

**核心文件**：`changedetectionio/static/js/visual-selector.js:175-280`

#### 3.2.1 Canvas 层级结构

```
#selector-wrapper (相对定位容器)
├── img#selector-background (截图背景)
└── canvas#selector-canvas (覆盖层，z-index 高于背景)
```

#### 3.2.2 缩放计算

```javascript
function setScale() {
    selectorImageRect = selectorImage.getBoundingClientRect();
    $selectorCanvasElem.attr({
        'height': selectorImageRect.height,
        'width': selectorImageRect.width
    });
    xScale = selectorImageRect.width / selectorImage.naturalWidth;
    yScale = selectorImageRect.height / selectorImage.naturalHeight;
}
```

#### 3.2.3 元素高亮渲染

```javascript
// 悬停高亮
function drawHighlight(sel) {
    ctx.strokeRect(
        sel.left * xScale, 
        sel.top * yScale, 
        sel.width * xScale, 
        sel.height * yScale
    );
    ctx.fillRect(/* 同上 */);
}

// 已选元素高亮（灰化背景，红色边框）
function highlightCurrentSelected() {
    xctx.fillStyle = 'rgba(205,205,205,0.95)';  // 灰化
    xctx.clearRect(0, 0, c.width, c.height);
    currentSelections.forEach(sel => {
        xctx.strokeRect(/* 绘制红色边框 */);
    });
}
```

---

## 四、用户交互与选择转换

### 4.1 元素选择模式

#### 4.1.1 鼠标悬停预览

```javascript
function handleMouseMove(e) {
    selectorData['size_pos'].forEach(sel => {
        if (e.offsetY > sel.top * yScale && 
            e.offsetY < sel.top * yScale + sel.height * yScale &&
            e.offsetX > sel.left * yScale && 
            e.offsetX < sel.left * yScale + sel.width * yScale) {
            setCurrentSelectedText(sel.xpath);  // 显示当前 XPath
            drawHighlight(sel);
        }
    })
}
```

#### 4.1.2 点击确认选择

```javascript
function handleMouseDown() {
    // Shift 键按下时追加到列表，否则替换
    currentSelections = appendToList 
        ? [...currentSelections, currentSelection] 
        : [currentSelection];
    highlightCurrentSelected();
    updateFiltersText();
}
```

#### 4.1.3 多选支持

- 按住 Shift 键可多选多个元素
- 选择结果自动去重

### 4.2 绘制框选模式（Image SSIM Diff 专用）

**核心文件**：`changedetectionio/static/js/visual-selector.js:282-648`

#### 4.2.1 模式切换

```javascript
// 两种模式：element（元素选择）和 draw（绘制框选）
const isImageProcessor = $('input[value="image_ssim_diff"]').is(':checked');
```

#### 4.2.2 绘制交互

- 鼠标按下开始绘制，移动调整大小，释放完成
- 支持拖拽移动已绘制的框
- 支持四角调整大小
- 绘制结果保存为 `bounding_box` 格式：`x,y,width,height

#### 4.2.3 坐标转换

```javascript
// 保存时转换回原始图像坐标
const naturalX = Math.round(drawnBox.x / xScale);
const naturalY = Math.round(drawnBox.y / yScale);
$('#bounding_box').val(`${naturalX},${naturalY},${naturalWidth},${naturalHeight}`);
```

### 4.3 选择结果转换为抽取规则

**核心文件**：`changedetectionio/static/js/visual-selector.js:164-173`

```javascript
function updateFiltersText() {
    let uniqueSelections = new Set(
        currentSelections.map(sel => 
            (sel[0] === '/' ? `xpath:${sel.xpath}` : sel.xpath)
    );
    $includeFiltersElem.val(
        Array.from(uniqueSelections).join("\n")
    );
}
```

转换规则：
- XPath 格式：`/html/body/div[1]` → `xpath:/html/body/div[1]`
- CSS 选择器：`#content > .item` → 直接保存

---

## 五、与文本/图片 Diff 的衔接

### 5.1 文本 Diff 流程

**核心文件**：`changedetectionio/processors/text_json_diff/processor.py`

#### 5.1.1 过滤器配置合并

```python
class FilterConfig:
    @property
    def include_filters(self):
        # 三级合并：watch → tags → global
        watch_rules = self.watch.get('include_filters', [])
        tag_rules = self.datastore.get_tag_overrides_for_watch(...)
        return list(dict.fromkeys(watch_rules + tag_rules))
```

#### 5.1.2 内容过滤执行

**核心文件**：`changedetectionio/html_tools.py:136-152`

```python
def include_filters(include_filters, html_content):
    from bs4 import BeautifulSoup
    soup = BeautifulSoup(html_content, "html.parser")
    r = soup.select(include_filters, separator="")
    for element in r:
        html_block += str(element)
    return html_block
```

#### 5.1.3 差异检测流程

```
1. 抓取页面内容 → 2. 应用 include_filters 过滤 → 3. 文本提取与转换
   → 4. 与历史版本对比 → 5. 生成差异报告
```

### 5.2 图片 Diff 流程

**核心文件**：`changedetectionio/processors/image_ssim_diff/`

#### 5.2.1 框选区域应用

- 从 `processor_config` 读取 `bounding_box`
- 对截图进行裁剪
- 使用 SSIM（结构相似性）算法对比

---

## 六、规则保存与持久化

### 6.1 表单提交

**核心文件**：`changedetectionio/blueprint/ui/edit.py:180-238`

```python
if request.method == 'POST' and form.validate():
    datastore.data['watching'][uuid].update(form.data)
    datastore.data['watching'][uuid].commit()
```

### 6.2 保存字段

| 字段 | 说明 |
|------|------|
| `include_filters` | XPath/CSS 选择器列表，每行一个 |
| `processor_config` | 处理器特定配置（如 bounding_box） |

### 6.3 配置哈希校验

```python
def get_filter_config_hash(self):
    config = {
        'include_filters': sorted(self.include_filters),
        'subtractive_selectors': sorted(self.subtractive_selectors),
        # ... 其他配置
    }
    return hashlib.md5(json.dumps(config).encode()).hexdigest()
```

---

## 七、完整数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                     浏览器抓取阶段                        │
├─────────────────────────────────────────────────────────────────┤
│  Playwright 打开页面 → 执行 XPATH_ELEMENT_JS → 采集元素 │
│  位置、XPath → 生成截图 → 返回 xpath_data + screenshot  │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     数据持久化阶段                        │
├─────────────────────────────────────────────────────────────────┤
│  save_xpath_data() → elements.deflate                   │
│  save_screenshot() → last-screenshot.png                │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     前端渲染阶段                                │
├─────────────────────────────────────────────────────────────────┤
│  加载截图 → 获取 elements.deflate → Canvas 绘制覆盖层         │
│  → 鼠标交互 → 选择元素/绘制框选                    │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     规则转换与保存                              │
├─────────────────────────────────────────────────────────────────┤
│  选择结果 → include_filters → 表单提交 → 保存到 watch.json │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     差异检测阶段                        │
├─────────────────────────────────────────────────────────────────┤
│  下次抓取 → 应用过滤器 → 提取内容 → 对比差异 → 通知   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、关键技术点

### 8.1 性能优化

1. **元素排序**：按面积从小到大排序，优先选择最小匹配元素
2. **数据压缩**：使用 zlib 压缩元素数据，节省磁盘空间
3. **分块截图**：大页面分块抓取后拼接，避免 GPU 内存溢出
4. **配置哈希**：通过哈希检测配置变化，避免不必要的重新检测

### 8.2 兼容性处理

1. **选择器降级**：CSS 选择器生成失败时回退到 XPath
2. **元素可见性判断**：综合判断 display、visibility、contentVisibility
3. **滚动偏移处理**：正确计算滚动后的元素位置

### 8.3 安全性

1. **访问控制**：元素数据接口遵循密码保护要求
2. **XSS 防护**：所有用户输入的选择器在服务器端重新执行验证

---

## 九、代码文件索引

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| 浏览器元素采集 | `changedetectionio/content_fetchers/res/xpath_element_scraper.js` | 浏览器端元素位置与XPath采集 |
| Playwright 抓取 | `changedetectionio/content_fetchers/playwright.py` | 浏览器自动化与数据抓取 |
| 数据持久化 | `changedetectionio/model/Watch.py` | 元素数据与截图保存 |
| 前端交互 | `changedetectionio/static/js/visual-selector.js` | 覆盖层渲染与用户交互 |
| 编辑页面 | `changedetectionio/blueprint/ui/edit.py` | 编辑页面路由与表单处理 |
| 编辑模板 | `changedetectionio/blueprint/ui/templates/edit.html` | 可视化选择器UI结构 |
| 静态资源 | `changedetectionio/flask_app.py` | 元素数据与截图提供 |
| 文本过滤 | `changedetectionio/html_tools.py` | CSS/XPath 过滤器执行 |
| 文本Diff处理器 | `changedetectionio/processors/text_json_diff/processor.py` | 文本差异检测 |
| 图片Diff处理器 | `changedetectionio/processors/image_ssim_diff/` | 图片差异检测 |
