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

2. **元素排序（浏览器端，升序）**

   **核心文件**：`changedetectionio/content_fetchers/res/xpath_element_scraper.js:278`

   ```javascript
   // Sort the elements so we find the smallest one first, in other words, we find the smallest one matching in that area
   // so that we dont select the wrapping element by mistake and be unable to select what we want
   size_pos.sort((a, b) => (a.width * a.height > b.width * b.height) ? 1 : -1)
   ```

   **排序逻辑**：按元素面积（width × height）**从小到大**排序（升序）

**排序逻辑**：按元素面积（width × height）**从小到大**排序（升序）

**设计意图**：
- 注释写着"so we find the smallest one first"，希望优先匹配最小元素
- **但实际效果相反**：由于 `forEach` 会遍历所有匹配元素并不断覆盖 `currentSelection`，升序排列时小元素先匹配、大元素后匹配，**最终匹配的是最后一个（最大的）元素**
- 这个排序实际上会导致选中最外层的父容器元素，与注释意图相反

**排序时机**：在浏览器端采集完成后、数据返回前进行排序

3. **元素位置与属性采集**
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

4. **选择器生成策略（优先级从高到低）**
   - **策略1**：如果是 `<input>` 且有 `name` 属性，优先使用 `tagName[name="xxx"]`
   - **策略2**：向上遍历直到找到有 ID 的元素，生成 `#id > tag > tag` 形式的 CSS 选择器
   - **策略3**：回退到传统 XPath 生成算法

5. **现有过滤器回显**
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
        sortScrapedElementsBySize();  // 按面积从大到小排序（前端二次排序）
        setScale();                // 设置缩放比例
        reflowSelector();          // 初始化选择器
    });
}
```

#### 3.1.3 元素排序（前端，降序）

**核心文件**：`changedetectionio/static/js/visual-selector.js:69-76`

```javascript
function sortScrapedElementsBySize() {
    // Sort the currentSelections array by area (width * height) in descending order
    selectorData['size_pos'].sort((a, b) => {
        const areaA = a.width * a.height;
        const areaB = b.width * b.height;
        return areaB - areaA;
    });
}
```

**排序逻辑**：按元素面积（width × height）**从大到小**排序（降序）

**设计意图**：
- Canvas 渲染时，大元素先绘制，小元素后绘制
- 小元素会叠加在大元素之上，确保视觉层级正确
- 但鼠标悬停匹配时仍按浏览器端的升序遍历（优先匹配小元素）

**两次排序的对比**：

| 阶段 | 排序方向 | 目的 |
|------|----------|------|
| 浏览器端 | 升序（小→大） | 鼠标悬停时优先匹配最小元素，避免误选父容器 |
| 前端 | 降序（大→小） | Canvas 渲染时大元素先画，小元素后画，保证视觉层级 |

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
- 绘制结果保存为 `bounding_box` 格式：`x,y,width,height`

#### 4.2.3 坐标转换

```javascript
// 保存时转换回原始图像坐标
const naturalX = Math.round(drawnBox.x / xScale);
const naturalY = Math.round(drawnBox.y / yScale);
const naturalWidth = Math.round(drawnBox.width / xScale);
const naturalHeight = Math.round(drawnBox.height / yScale);
$('#bounding_box').val(`${naturalX},${naturalY},${naturalWidth},${naturalHeight}`);
```

#### 4.2.4 框选坐标保存格式

**核心文件**：`changedetectionio/processors/image_ssim_diff/forms.py:65-83`

```python
processor_config_bounding_box = StringField(
    _l('Bounding Box'),
    validators=[
        validators.Optional(),
        validators.Length(max=100),
        validate_bounding_box
    ],
    render_kw={"style": "display: none;", "id": "bounding_box"}
)

def validate_bounding_box(form, field):
    # 格式校验：必须是四个逗号分隔的非负整数
    if not re.match(r'^\d+,\d+,\d+,\d+$', field.data):
        raise ValidationError('Bounding box must be in format: x,y,width,height')
```

**保存格式**：`x,y,width,height`（四个非负整数，逗号分隔）

**示例**：`100,200,300,400` 表示左上角 (100, 200)，宽 300px，高 400px

#### 4.2.5 回显条件

**回显触发时机**：
1. 页面加载时 `initializeDrawMode()` 被调用
2. 从隐藏表单字段读取已保存的值

**回显逻辑**：

```javascript
function initializeDrawMode() {
    // 1. 读取已保存的选择模式
    const savedMode = $selectionModeField.val();
    if (savedMode && (savedMode === 'element' || savedMode === 'draw')) {
        $selectorModeRadios.filter(`[value="${savedMode}"]`).prop('checked', true);
    }

    // 2. 读取已保存的框选坐标
    const existingBox = $boundingBoxField.val();
    if (existingBox) {
        try {
            const parts = existingBox.split(',').map(p => parseFloat(p));
            if (parts.length === 4) {
                // 转换为当前缩放后的坐标
                drawnBox = {
                    x: parts[0] * xScale,
                    y: parts[1] * yScale,
                    width: parts[2] * xScale,
                    height: parts[3] * yScale
                };
                drawBox();  // 重新绘制
            }
        } catch (e) {
            console.error('Failed to parse existing bounding box:', e);
        }
    }
}
```

**回显前置条件**：
- 处理器类型为 `image_ssim_diff`
- 隐藏字段 `#bounding_box` 有有效值
- 隐藏字段 `#selection_mode` 有有效值
- 截图已加载完成，缩放比例已计算

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

## 五、降级处理机制

### 5.1 数据就绪检查

**核心文件**：`changedetectionio/store/__init__.py:812-821`

```python
def visualselector_data_is_ready(self, watch_uuid):
    """
    Check if visual selector data (screenshot + elements) is ready.
    Returns: bool: True if both screenshot and elements data exist
    """
    has_screenshot = self._watch_resource_exists(watch_uuid, "last-screenshot.png")
    has_elements = self._watch_resource_exists(watch_uuid, "elements.deflate")
    return has_screenshot and has_elements
```

**检查时机**：编辑页面渲染时（`edit.py:340`）

### 5.2 截图加载失败

**核心文件**：`changedetectionio/static/js/visual-selector.js:107-111`

```javascript
$selectorBackgroundElem
    .on("error", () => {
        $fetchingUpdateNoticeElem.html(
            "<strong>Ooops!</strong> The VisualSelector tool needs at least one fetched page, " +
            "please unpause the watch and/or wait for the watch to complete fetching " +
            "and then reload this page."
        ).css('color', '#bb0000');
        $('#selector-current-xpath, #clear-selector').hide();
    })
```

**降级表现**：
- 显示红色错误提示
- 隐藏 XPath 显示和清除按钮
- Canvas 覆盖层不初始化

### 5.3 现有过滤器找不到

**核心文件**：`changedetectionio/static/js/visual-selector.js:126-136`

```javascript
function alertIfFilterNotFound() {
    let existingFilters = splitToList($includeFiltersElem.val());
    let sizePosXpaths = selectorData['size_pos'].map(sel => sel.xpath);

    for (let filter of existingFilters) {
        if (!sizePosXpaths.includes(filter)) {
            alert(`One or more of your existing filters was not found ` +
                  `and will be removed when a new filter is selected.`);
            break;
        }
    }
}
```

**降级表现**：
- 弹出警告提示用户
- 不阻止用户操作，但选择新过滤器时会覆盖旧的

### 5.4 不支持可视化选择器

**核心文件**：`changedetectionio/blueprint/ui/templates/edit.html:464-467`

```html
{% else %}
    <p>
        <strong>{{ _('Sorry, this functionality only works with fetchers that support Javascript and screenshots (such as playwright etc).') }}</strong>
    </p>
{% endif %}
```

**降级表现**：
- 显示不支持的提示信息
- 不渲染 Canvas 选择界面

---

## 六、与文本/图片 Diff 的衔接

### 6.1 规则存储机制对比

| 特性 | 文本 Diff（text_json_diff） | 图片 Diff（image_ssim_diff） |
|------|----------------------------|-----------------------------|
| 主配置文件 | `watch.json` | `watch.json` + `image_ssim_diff.json` |
| 选择器字段 | `include_filters`（保存在 watch.json） | `bounding_box`（保存在处理器配置） |
| 配置前缀 | 无（直接保存） | `processor_config_` 前缀 |
| 表单字段 | `include_filters` | `processor_config_bounding_box` |

### 6.2 处理器配置提取

**核心文件**：`changedetectionio/processors/__init__.py:472-498`

```python
def extract_processor_config_from_form_data(form_data):
    """
    Extract processor_config_* fields from form data and return separate dicts.
    IMPORTANT: This function modifies form_data in-place by removing processor_config_* fields.
    """
    processor_config_data = {}

    for field_name in list(form_data.keys()):
        if field_name.startswith('processor_config_'):
            config_key = field_name.replace('processor_config_', '')
            processor_config_data[config_key] = form_data[field_name]
            del form_data[field_name]  # 从主配置中移除

    return processor_config_data
```

**处理流程**：
1. 表单提交后，提取所有 `processor_config_` 前缀的字段
2. 移除前缀后保存到独立字典
3. 从主 form_data 中删除这些字段，避免保存到 watch.json

### 6.3 处理器配置保存

**核心文件**：`changedetectionio/processors/__init__.py:428-469`

```python
def save_processor_config(datastore, watch_uuid, config_data):
    watch = datastore.data['watching'].get(watch_uuid)
    processor_name = watch.get('processor', 'text_json_diff')
    
    processor_instance = difference_detection_processor(datastore, watch_uuid)
    config_filename = f'{processor_name}.json'
    processor_instance.update_extra_watch_config(config_filename, config_data)
```

**保存位置**：
- 文本 Diff：无额外配置文件，`include_filters` 直接保存在 `watch.json`
- 图片 Diff：`{watch_data_dir}/image_ssim_diff.json`

### 6.4 处理器配置读取

**核心文件**：`changedetectionio/blueprint/ui/edit.py:132-162`

```python
if request.method == 'GET' and processor_name:
    try:
        processor_instance = difference_detection_processor(datastore, uuid)
        config_filename = f'{processor_name}.json'
        processor_config = processor_instance.get_extra_watch_config(config_filename)

        if processor_config:
            for config_key, config_value in processor_config.items():
                target_field = getattr(form, f'processor_config_{config_key}', None)
                if target_field is not None:
                    for sub_key, sub_value in config_value.items():
                        sub_field = target_field.form._fields.get(sub_key)
                        if sub_field is not None:
                            sub_field.data = sub_value
    except Exception as e:
        logger.warning(f"Failed to load processor config: {e}")
```

**读取流程**：
1. GET 请求时，从处理器配置文件读取配置
2. 将配置填充到对应的表单字段
3. 前端 JavaScript 从隐藏字段读取并回显

### 6.5 表单提交时的处理

**核心文件**：`changedetectionio/blueprint/ui/edit.py:192-195`

```python
# Handle processor-config-* fields separately (save to JSON, not datastore)
# IMPORTANT: These must NOT be saved to url-watches.json, only to the processor-specific JSON file
processor_config_data = processors.extract_processor_config_from_form_data(form.data)
processors.save_processor_config(datastore, uuid, processor_config_data)
```

**执行顺序**：
1. 提取处理器特定配置
2. 保存处理器配置到独立 JSON 文件
3. 剩余字段保存到 watch.json
4. 更新 datastore

### 6.6 差异检测时的应用

#### 6.6.1 文本 Diff 流程

```
1. 从 watch.json 读取 include_filters
2. FilterConfig 三级合并（watch → tags → global）
3. 按过滤器类型分路径执行
   ├─ XPath 过滤器 → elementpath 库
   └─ CSS 选择器 → BeautifulSoup4
4. 提取文本内容
5. 与历史版本对比
```

#### 6.6.2 图片 Diff 流程

```
1. 从 image_ssim_diff.json 读取 bounding_box
2. 裁剪截图到指定区域
3. 使用 SSIM 算法计算结构相似性
4. 比较差异百分比与阈值
5. 判断是否触发变更
```

---

## 七、规则保存与持久化

### 7.1 表单提交

**核心文件**：`changedetectionio/blueprint/ui/edit.py:180-238`

```python
if request.method == 'POST' and form.validate():
    datastore.data['watching'][uuid].update(form.data)
    datastore.data['watching'][uuid].commit()
```

### 7.2 保存字段

| 字段 | 保存位置 | 说明 |
|------|----------|------|
| `include_filters` | watch.json | XPath/CSS 选择器列表，每行一个 |
| `processor_config_bounding_box` | image_ssim_diff.json | 图片 Diff 的框选区域 |
| `processor_config_selection_mode` | image_ssim_diff.json | 选择模式（element/draw） |

### 7.3 配置哈希校验

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

## 八、完整数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                     浏览器抓取阶段                                │
├─────────────────────────────────────────────────────────────────┤
│  Playwright 打开页面 → 执行 XPATH_ELEMENT_JS → 采集元素         │
│  位置、XPath → 按面积升序排序 → 生成截图 → 返回 xpath_data      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     数据持久化阶段                                │
├─────────────────────────────────────────────────────────────────┤
│  save_xpath_data() → elements.deflate                           │
│  save_screenshot() → last-screenshot.png                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     前端渲染阶段                                │
├─────────────────────────────────────────────────────────────────┤
│  加载截图 → 获取 elements.deflate → 按面积降序排序              │
│  → Canvas 绘制覆盖层 → 鼠标交互 → 选择元素/绘制框选             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     规则转换与保存                              │
├─────────────────────────────────────────────────────────────────┤
│  文本选择 → include_filters → watch.json                        │
│  框选坐标 → processor_config_bounding_box → image_ssim_diff.json │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     差异检测阶段                                │
├─────────────────────────────────────────────────────────────────┤
│  文本Diff: 应用 include_filters → 提取文本 → 对比差异           │
│  图片Diff: 应用 bounding_box → 截图裁剪 → SSIM 对比             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、关键技术点

### 9.1 性能优化

1. **元素排序**：两次排序机制——浏览器端升序、前端降序，配合 `forEach` 覆盖逻辑实现最小元素优先命中
2. **数据压缩**：使用 zlib 压缩元素数据，节省磁盘空间
3. **分块截图**：大页面分块抓取后拼接，避免 GPU 内存溢出
4. **配置哈希**：通过哈希检测配置变化，避免不必要的重新检测

### 9.2 兼容性处理

1. **选择器降级**：CSS 选择器生成失败时回退到 XPath
2. **元素可见性判断**：综合判断 display、visibility、contentVisibility
3. **滚动偏移处理**：正确计算滚动后的元素位置

### 9.3 安全性

1. **访问控制**：元素数据接口遵循密码保护要求
2. **XSS 防护**：所有用户输入的选择器在服务器端重新执行验证

### 9.4 降级处理

1. **数据就绪检查**：双文件检查确保截图和元素数据同时存在
2. **截图加载失败**：友好的错误提示和引导
3. **过滤器失效**：警告用户但不阻止操作
4. **功能不支持**：明确的提示信息

---

## 十、代码文件索引

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| 浏览器元素采集 | `changedetectionio/content_fetchers/res/xpath_element_scraper.js` | 浏览器端元素位置与XPath采集 |
| Playwright 抓取 | `changedetectionio/content_fetchers/playwright.py` | 浏览器自动化与数据抓取 |
| 数据持久化 | `changedetectionio/model/Watch.py` | 元素数据与截图保存 |
| 前端交互 | `changedetectionio/static/js/visual-selector.js` | 覆盖层渲染与用户交互 |
| 编辑页面 | `changedetectionio/blueprint/ui/edit.py` | 编辑页面路由与表单处理 |
| 编辑模板 | `changedetectionio/blueprint/ui/templates/edit.html` | 可视化选择器UI结构 |
| 静态资源 | `changedetectionio/flask_app.py` | 元素数据与截图提供 |
| 文本过滤 | `changedetectionio/html_tools.py` | CSS 选择器 / XPath 过滤器分路径执行 |
| 文本Diff处理器 | `changedetectionio/processors/text_json_diff/processor.py` | 文本差异检测 |
| 图片Diff处理器 | `changedetectionio/processors/image_ssim_diff/` | 图片差异检测 |
| 处理器配置管理 | `changedetectionio/processors/__init__.py` | 处理器配置提取与保存 |
| 数据就绪检查 | `changedetectionio/store/__init__.py` | 可视化选择器数据就绪检查 |
