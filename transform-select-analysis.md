# miniPaint 图像几何变换与选区处理分析报告

## 概述

本报告分析 miniPaint 浏览器端图片编辑器的图像几何变换和选区处理实现机制，涵盖选区数据结构、裁剪操作、旋转翻转、缩放插值以及自动裁边等核心功能。

---

## 1. 矩形选区数据结构与事件处理

### 1.1 选区数据结构定义

选区的核心数据结构定义在 [base-selection.js](src/js/core/base-selection.js) 中，是一个简单的矩形对象：

```javascript
// 选区数据结构
{
    x: null,        // 选区左上角 X 坐标
    y: null,        // 选区左上角 Y 坐标
    width: null,    // 选区宽度
    height: null    // 选区高度
}
```

在 [selection.js](src/js/tools/selection.js) 中，选区工具维护自己的选区状态：

```javascript
this.selection = {
    x: null,
    y: null,
    width: null,
    height: null,
};
```

**选区配置参数**（通过 `sel_config` 传入 `Base_selection_class`）：

| 参数 | 类型 | 说明 |
|------|------|------|
| `enable_background` | boolean | 是否显示半透明绿色背景 |
| `enable_borders` | boolean | 是否显示边框线 |
| `enable_controls` | boolean | 是否显示控制手柄 |
| `enable_rotation` | boolean | 是否启用旋转控制 |
| `enable_move` | boolean | 是否允许移动 |
| `keep_ratio` | boolean | 是否保持宽高比 |
| `crop_lines` | boolean | 是否显示三分法裁剪辅助线 |
| `data_function` | function | 返回当前选区数据的回调函数 |

### 1.2 鼠标事件到选区坐标的转换

事件处理流程在 [selection.js](src/js/tools/selection.js) 中实现：

**mousedown 阶段**：
```javascript
mousedown(e) {
    var mouse = this.get_mouse_info(e);
    // 创建新选区
    this.selection = {
        x: mouse.x,        // 记录起始点
        y: mouse.y,
        width: 0,
        height: 0,
    };
    this.type = 'create';
    this.selection_coords_from = {x: mouse.x, y: mouse.y};
}
```

**mousemove 阶段**：
```javascript
mousemove(e) {
    var mouse = this.get_mouse_info(e);
    if (this.type == 'create') {
        // 动态计算选区尺寸
        this.selection.width = mouse.x - mouse.click_x;
        this.selection.height = mouse.y - mouse.click_y;
        config.need_render = true;
    }
}
```

**mouseup 阶段**（处理负值坐标）：
```javascript
mouseup(e) {
    // 确保坐标非负
    if (details.width < 0) {
        x = x + details.width;  // 调整起点
    }
    if (details.height < 0) {
        y = y + details.height;
    }
    this.selection = {
        x: x,
        y: y,
        width: Math.abs(details.width),
        height: Math.abs(details.height),
    };
}
```

**坐标转换关键点**：
- `mouse.x` / `mouse.y`：当前鼠标位置（已转换为画布坐标）
- `mouse.click_x` / `mouse.click_y`：鼠标按下时的起始位置
- 选区宽高 = 当前位置 - 起始位置
- 负值表示反向拖拽，需要在 mouseup 时修正

### 1.3 虚线选框绘制

选框绘制在 [base-selection.js](src/js/core/base-selection.js) 的 `draw_selection()` 方法中实现：

**边框绘制（双线效果）**：
```javascript
// 外层白线
this.ctx.lineWidth = wholeLineWidth;  // 2 / config.ZOOM
this.ctx.strokeStyle = 'rgb(255, 255, 255)';
this.ctx.strokeRect(x - halfLineWidth, y - halfLineWidth, w + wholeLineWidth, h + wholeLineWidth);

// 内层黑线
this.ctx.lineWidth = halfLineWidth;   // 1 / config.ZOOM
this.ctx.strokeStyle = 'rgb(0, 0, 0)';
this.ctx.strokeRect(x - wholeLineWidth, y - wholeLineWidth, w + (wholeLineWidth * 2), h + (wholeLineWidth * 2));
```

**控制手柄绘制**（圆形）：
```javascript
var corner = (x, y, dx, dy, drag_type, cursor) => {
    const circle = new Path2D();
    circle.arc(x + dx * block_size, y + dy * block_size, block_size / 2, 0, 2 * Math.PI);
    
    this.ctx.fill(circle);
    this.ctx.stroke(circle);
    
    // 注册位置用于点击检测
    this.selected_obj_positions[drag_type] = {
        cursor: cursor,
        path: circle,
    };
};
```

**手柄位置**（8个控制点 + 1个旋转点）：
- 四角：`nwse-resize` / `nesw-resize` 光标
- 四边中点：`ns-resize` / `ew-resize` 光标
- 旋转手柄：黄色圆形，位于选区上方

**旋转支持**：
```javascript
if (data.rotate != null && data.rotate != 0) {
    this.ctx.translate(data.x + data.width / 2, data.y + data.height / 2);
    this.ctx.rotate(data.rotate * Math.PI / 180);
    x = Math.round(-data.width / 2);
    y = Math.round(-data.height / 2);
}
```

### 1.4 选区在全局状态中的存在形式

选区通过以下方式存在于全局状态：

**1. 工具内部状态**：
```javascript
// selection.js
this.selection = { x, y, width, height };
```

**2. 通过 `data_function` 关联到配置**：
```javascript
var sel_config = {
    data_function: function () {
        return _this.selection;  // 返回选区引用
    },
};
```

**3. Base_selection_class 单例管理**：
```javascript
// base-selection.js
var settings_all = [];  // 存储所有工具的选区配置

find_settings() {
    var current_key = config.TOOL.name;
    for (var i in settings_all) {
        if (i == current_key)
            settings = settings_all[i];
    }
    settings.data = (settings.data_function).call();
    return settings;
}
```

**4. Action 系统状态管理**：
```javascript
// 通过 Action 记录选区变化
app.State.do_action(
    new app.Actions.Set_selection_action(x, y, width, height, previous_selection)
);
```

---

## 2. 裁剪操作实现分析

### 2.1 裁剪工作流程

[crop.js](src/js/tools/crop.js) 实现裁剪功能，核心流程如下：

**Step 1: 创建选区**（用户拖拽）
```javascript
mousedown(e) {
    this.Base_selection.set_selection(mouse.x, mouse.y, 0, 0);
}

mousemove(e) {
    var width = mouse.x - mouse.click_x;
    var height = mouse.y - mouse.click_y;
    
    // Ctrl 键保持宽高比
    if(e.ctrlKey == true || e.metaKey){
        var ratio = config.WIDTH / config.HEIGHT;
        // ... 计算等比尺寸
    }
    
    this.Base_selection.set_selection(null, null, width, height);
}
```

**Step 2: 执行裁剪**（`on_params_update` 方法）
```javascript
async on_params_update() {
    let actions = [];
    
    for (var i in config.layers) {
        var link = config.layers[i];
        
        // 1. 调整图层位置（相对于裁剪区域）
        x -= parseInt(selection.x);
        y -= parseInt(selection.y);
        
        if (link.type == 'image') {
            // 2. 计算需要裁剪的边界
            let left = (x < 0) ? -x : 0;
            let top = (y < 0) ? -y : 0;
            let right = (x + width > selection.width) ? x + width - selection.width : 0;
            let bottom = (y + height > selection.height) ? y + height - selection.height : 0;
            
            // 3. 创建新 canvas 并裁剪
            let canvas = document.createElement('canvas');
            canvas.width = crop_width / width_ratio;
            canvas.height = crop_height / height_ratio;
            
            ctx.translate(-left / width_ratio, -top / height_ratio);
            ctx.drawImage(link.link, 0, 0);
            
            // 4. 更新图层图像
            actions.push(new app.Actions.Update_layer_image_action(canvas, link.id));
        }
        
        // 5. 更新图层属性
        actions.push(new app.Actions.Update_layer_action(link.id, {x, y, width, height}));
    }
    
    // 6. 更新画布尺寸
    actions.push(new app.Actions.Update_config_action({
        WIDTH: parseInt(selection.width),
        HEIGHT: parseInt(selection.height)
    }));
}
```

### 2.2 图层处理方式

**原图层处理**：裁剪操作**直接修改原图层数据**，而非创建新图层。

具体修改内容：
- **图像数据**：通过 `Update_layer_image_action` 替换图层的 canvas 内容
- **位置属性**：调整 `x`、`y` 坐标
- **尺寸属性**：更新 `width`、`height`、`width_original`、`height_original`

**关键代码**：
```javascript
// 创建新 canvas 裁剪图像
let canvas = document.createElement('canvas');
let ctx = canvas.getContext("2d");
canvas.width = crop_width / width_ratio;
canvas.height = crop_height / height_ratio;

ctx.translate(-left / width_ratio, -top / height_ratio);
ctx.drawImage(link.link, 0, 0);

// 直接更新原图层
actions.push(new app.Actions.Update_layer_image_action(canvas, link.id));
```

### 2.3 画布尺寸变化

**裁剪后画布尺寸会变化**，变为选区的尺寸：

```javascript
actions.push(
    new app.Actions.Update_config_action({
        WIDTH: parseInt(selection.width),
        HEIGHT: parseInt(selection.height)
    })
);
```

**尺寸变化流程**：
1. 所有图层位置相对于裁剪区域左上角重新计算
2. 画布宽高更新为选区宽高
3. 超出裁剪区域的图像部分被丢弃

### 2.4 特殊处理

**旋转图层限制**：
```javascript
// 检查是否有旋转的图层
for (var i in config.layers) {
    if(link.rotate > 0){
        alertify.error('Crop on rotated layer is not supported. Convert it to raster to continue.');
        return;
    }
}
```

**边界控制**：
```javascript
selection.x = Math.max(selection.x, 0);
selection.y = Math.max(selection.y, 0);
selection.width = Math.min(selection.width, config.WIDTH);
selection.height = Math.min(selection.height, config.HEIGHT);
```

---

## 3. 图像旋转与翻转实现

### 3.1 旋转实现

[rotate.js](src/js/modules/image/rotate.js) 提供旋转功能：

**旋转方式**：使用 **Canvas Transform API**，而非像素矩阵运算

**实现原理**：
```javascript
// 旋转通过修改图层属性实现
config.layer.rotate = new_rotate;

// 渲染时使用 canvas transform
ctx.translate(data.x + data.width / 2, data.y + data.height / 2);
ctx.rotate(data.rotate * Math.PI / 180);
```

**旋转角度处理**：
```javascript
rotate_handler(data, can_resize = true) {
    var value = parseInt(data.rotate);
    if (data.right_angle != 'Custom') {
        value = parseInt(data.right_angle);  // 0, 90, 180, 270
    }
    
    // 规范化到 0-360 范围
    if (value < 0) value = 360 + value;
    if (value >= 360) value = value - 360;
}
```

### 3.2 旋转包围盒计算

**任意角度旋转后的包围盒计算**：
```javascript
check_sizes(new_rotate) {
    var w = config.layer.width;
    var h = config.layer.height;
    
    var o = new_rotate * Math.PI / 180;
    
    // 计算旋转后的包围盒尺寸
    var new_x = w * Math.abs(Math.cos(o)) + h * Math.abs(Math.sin(o));
    var new_y = w * Math.abs(Math.sin(o)) + h * Math.abs(Math.cos(o));
    
    // 如果超出画布，扩展画布
    if (new_x > config.WIDTH || new_y > config.HEIGHT) {
        let new_width = config.WIDTH;
        let new_height = config.HEIGHT;
        
        if (new_x > config.WIDTH) {
            dx = Math.ceil(new_x - new_width) / 2;
            new_width = new_x;
        }
        if (new_y > config.HEIGHT) {
            dy = Math.ceil(new_y - new_height) / 2;
            new_height = new_y;
        }
        
        // 移动图层并扩展画布
        actions.push(new app.Actions.Update_layer_action(config.layer.id, {
            x: config.layer.x + dx,
            y: config.layer.y + dy
        }));
        actions.push(new app.Actions.Update_config_action({
            WIDTH: new_width,
            HEIGHT: new_height
        }));
    }
}
```

**包围盒公式**：
- 新宽度 = |原宽度 × cos(θ)| + |原高度 × sin(θ)|
- 新高度 = |原宽度 × sin(θ)| + |原高度 × cos(θ)|

### 3.3 翻转实现

[flip.js](src/js/modules/image/flip.js) 实现翻转功能：

**翻转方式**：使用 **Canvas scale + drawImage API**

**垂直翻转**：
```javascript
if (mode == 'vertical') {
    ctx2.scale(1, -1);  // Y 轴镜像
    ctx2.drawImage(canvas, 0, canvas2.height * -1);
}
```

**水平翻转**：
```javascript
else if (mode == 'horizontal') {
    ctx2.scale(-1, 1);  // X 轴镜像
    ctx2.drawImage(canvas, canvas2.width * -1, 0);
}
```

**完整流程**：
```javascript
flip(mode) {
    // 1. 获取图层 canvas
    var canvas = this.Base_layers.convert_layer_to_canvas(null, true);
    
    // 2. 创建目标 canvas
    var canvas2 = document.createElement('canvas');
    canvas2.width = canvas.width;
    canvas2.height = canvas.height;
    
    // 3. 应用变换
    ctx2.scale(1, -1);  // 或 scale(-1, 1)
    ctx2.drawImage(canvas, ...);
    
    // 4. 更新图层
    return app.State.do_action(
        new app.Actions.Update_layer_image_action(canvas2)
    );
}
```

### 3.4 技术对比

| 操作 | 实现方式 | 是否修改原图 | 性能 |
|------|----------|--------------|------|
| 旋转 | Canvas Transform API | 否（修改属性） | 高（GPU 加速） |
| 翻转 | Canvas scale + drawImage | 是（创建新 canvas） | 高 |

---

## 4. 缩放插值与透明边缘裁剪

### 4.1 缩放插值算法

[resize.js](src/js/modules/image/resize.js) 提供三种缩放模式：

#### 模式一：Lanczos（默认推荐）

```javascript
if (mode == "Lanczos") {
    var tmp_data = document.createElement("canvas");
    tmp_data.width = width;
    tmp_data.height = height;
    
    await this.pica.resize(canvas, tmp_data, {
        alpha: true,  // 保持透明通道
    });
}
```

**特点**：
- 使用 [Pica](https://github.com/nodeca/pica) 库
- Lanczos3 滤波器，高质量重采样
- 支持放大和缩小
- 自动抗锯齿

#### 模式二：Hermite

```javascript
else if (mode == "Hermite") {
    this.Hermite.resample_single(canvas, width, height, true);
}
```

**特点**：
- 使用 [hermite-resize](https://github.com/viliusle/Hermite-resize) 库
- Hermite 插值算法
- **仅支持缩小**，放大时自动切换到 Lanczos
- 较快的处理速度

**限制检查**：
```javascript
if (mode == "Hermite" && (width > canvas.width || height > canvas.height)) {
    alertify.warning('Scaling up is not supported in Hermite, using Lanczos.');
    mode = "Lanczos";
}
```

#### 模式三：Basic（基础）

```javascript
else {
    var tmp_data = document.createElement("canvas");
    tmp_data.width = canvas.width;
    tmp_data.height = canvas.height;
    tmp_data.getContext("2d").drawImage(canvas, 0, 0);
    
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    canvas.width = width;
    canvas.height = height;
    
    ctx.drawImage(tmp_data, 0, 0, width, height);
}
```

**特点**：
- 使用浏览器原生 `drawImage` 缩放
- 浏览器自动选择插值算法（通常为双线性）
- 速度最快，质量一般

### 4.2 抗锯齿处理

**1. Pica 库内置抗锯齿**：
```javascript
await this.pica.resize(canvas, tmp_data, {
    alpha: true,  // 透明通道正确处理
});
```

**2. 可选锐化后处理**：
```javascript
if (sharpen == true) {
    var imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    var filtered = _this.ImageFilters.Sharpen(imageData, 1);
    ctx.putImageData(filtered, 0, 0);
}
```

**锐化作用**：
- 补偿缩放造成的模糊
- 增强边缘细节
- 使用 ImageFilters 库的 Sharpen 滤镜

### 4.3 Trim 自动裁边实现

[trim.js](src/js/modules/image/trim.js) 实现透明边缘检测和裁剪：

#### 像素边界检测算法

```javascript
get_trim_info(layer_id, trim_white, power) {
    var img = ctx.getImageData(0, 0, canvas.width, canvas.height);
    var imgData = img.data;
    
    // 检测顶部边界
    main1:
    for (var y = 0; y < img.height; y++) {
        for (var x = 0; x < img.width; x++) {
            var k = ((y * (img.width * 4)) + (x * 4));
            
            // 检查透明度
            if (imgData[k + 3] <= power)
                continue;  // 透明像素，跳过
            
            // 检查白色（可选）
            if (trim_white == true && 
                imgData[k] >= 255 - power && 
                imgData[k + 1] >= 255 - power &&
                imgData[k + 2] >= 255 - power)
                continue;  // 白色像素，跳过
            
            break main1;  // 找到有效像素，停止
        }
        top++;
    }
    
    // 同理检测 left、bottom、right
}
```

**检测条件**：
- **透明像素**：`alpha <= power`（默认 power=0）
- **白色像素**：`R >= 255-power && G >= 255-power && B >= 255-power`

#### 四边界检测顺序

```javascript
// 1. Top: 从上到下逐行扫描
for (var y = 0; y < img.height; y++) {
    for (var x = 0; x < img.width; x++) { ... }
}

// 2. Left: 从左到右逐列扫描
for (var x = 0; x < img.width; x++) {
    for (var y = 0; y < img.height; y++) { ... }
}

// 3. Bottom: 从下到上逐行扫描
for (var y = img.height - 1; y >= 0; y--) {
    for (var x = img.width - 1; x >= 0; x--) { ... }
}

// 4. Right: 从右到左逐列扫描
for (var x = img.width - 1; x >= 0; x--) {
    for (var y = img.height - 1; y >= 0; y--) { ... }
}
```

### 4.4 大图性能分析

#### Trim 性能瓶颈

**时间复杂度**：O(W × H) 最坏情况（全透明图像）

**性能问题**：
1. **四次遍历**：top、left、bottom、right 各遍历一次
2. **getImageData 开销**：一次性读取整个图像数据
3. **无提前终止优化**：即使已找到边界仍继续扫描

**大图场景**：
- 4000×3000 图像 = 12M 像素 = 48MB RGBA 数据
- 四次遍历 = 192MB 内存访问
- 预估耗时：100-500ms（取决于设备）

**优化建议**：
```javascript
// 可优化为单次遍历
for (var y = 0; y < img.height; y++) {
    for (var x = 0; x < img.width; x++) {
        if (is_valid_pixel(x, y)) {
            top = Math.min(top, y);
            left = Math.min(left, x);
            bottom = Math.max(bottom, y);
            right = Math.max(right, x);
        }
    }
}
```

#### Resize 性能对比

| 模式 | 算法 | 大图性能 | 内存占用 |
|------|------|----------|----------|
| Lanczos | Pica (WebWorker) | 中等（可并行） | 高 |
| Hermite | JS 单线程 | 较慢 | 中 |
| Basic | 浏览器原生 | 最快 | 低 |

---

## 5. 代码组织评估与透视变换扩展

### 5.1 目录结构与职责分工

```
src/js/
├── tools/                    # 交互式工具（用户操作）
│   ├── select.js            # 对象选择工具
│   ├── selection.js         # 矩形选区工具
│   └── crop.js              # 裁剪工具
│
├── modules/image/           # 图像处理模块（批量操作）
│   ├── rotate.js            # 旋转模块
│   ├── flip.js              # 翻转模块
│   ├── resize.js            # 缩放模块
│   └── trim.js              # 裁边模块
│
└── core/
    └── base-selection.js    # 选区基础类（绘制与交互）
```

### 5.2 职责划分

| 类别 | tools/ | modules/image/ |
|------|--------|----------------|
| **定位** | 交互式工具 | 图像处理模块 |
| **触发方式** | 用户画布操作 | 菜单/快捷键 |
| **状态管理** | 维护选区状态 | 无状态，纯函数 |
| **UI 交互** | 鼠标拖拽、键盘 | 弹窗参数配置 |
| **撤销支持** | 通过 Action 系统 | 通过 Action 系统 |

### 5.3 功能重叠分析

**存在的重叠**：

1. **选区绘制**：
   - `tools/select.js`：选择整个图层对象
   - `tools/selection.js`：创建像素选区
   - `tools/crop.js`：创建裁剪选区
   - 三者都使用 `Base_selection_class`

2. **图像裁剪**：
   - `tools/crop.js`：基于选区裁剪画布
   - `modules/image/trim.js`：自动裁剪透明边缘
   - 功能相似但触发方式不同

**职责清晰的地方**：
- `tools/` 负责用户交互和选区管理
- `modules/image/` 负责图像处理算法
- `core/base-selection.js` 提供选区绘制基础设施

### 5.4 新增自由透视变换改动点

如需新增透视变换功能，需要改动以下文件：

#### 1. 新建工具文件

**`src/js/tools/perspective.js`**：
```javascript
class Perspective_tool_class extends Base_tools_class {
    constructor(ctx) {
        super();
        this.name = 'perspective';
        this.corners = [
            {x: 0, y: 0},           // 左上
            {x: width, y: 0},       // 右上
            {x: width, y: height},  // 右下
            {x: 0, y: height}       // 左下
        ];
    }
    
    // 四角拖拽控制点
    // 透视网格预览
    // 确认/取消操作
}
```

#### 2. 新建图像处理模块

**`src/js/modules/image/perspective.js`**：
```javascript
class Image_perspective_class {
    apply(layer_id, corners) {
        // 使用 Canvas 2D 透视变换或 WebGL
        // 或使用 CSS matrix3d 计算
    }
    
    // 计算透视变换矩阵
    // 应用像素重采样
}
```

#### 3. 修改选区基础类

**`src/js/core/base-selection.js`**：
```javascript
// 新增四角控制点绘制
draw_perspective_handles(corners) {
    // 绘制四个角点
    // 绘制透视网格线
}

// 新增透视选区检测
check_perspective_hit(mouse) {
    // 检测是否点击角点
}
```

#### 4. 更新配置

**`src/js/config.js`**：
```javascript
config.TOOLS = [
    // ...
    {
        name: 'perspective',
        title: 'Perspective Transform',
        attributes: {
            mode: 'free',
        },
    },
];
```

#### 5. 新增 Action

**`src/js/app.js`** 或新建 **`src/js/actions/perspective-action.js`**：
```javascript
class Perspective_action extends Base_action {
    constructor(layer_id, old_corners, new_corners) {
        // 记录透视变换参数
        // 支持撤销/重做
    }
}
```

#### 6. 技术选型建议

| 方案 | 优点 | 缺点 |
|------|------|------|
| Canvas 2D + 手动插值 | 兼容性好 | 性能差，实现复杂 |
| WebGL | 性能好 | 兼容性差，学习曲线陡 |
| CSS matrix3d | 简单 | 仅限 DOM 元素 |
| 第三方库（如 glfx.js） | 功能完整 | 增加依赖 |

**推荐方案**：使用 Canvas 2D + 透视网格插值算法

```javascript
// 透视变换核心算法
function perspectiveTransform(srcCanvas, corners) {
    const dstCanvas = document.createElement('canvas');
    const ctx = dstCanvas.getContext('2d');
    
    // 计算目标尺寸（包围盒）
    // 遍历目标像素，反向映射到源像素
    // 双线性插值采样
    
    return dstCanvas;
}
```

---

## 6. 总结

### 6.1 架构特点

1. **选区系统**：简洁的矩形数据结构，通过 `Base_selection_class` 统一绘制和交互
2. **变换操作**：优先使用 Canvas 原生 API（transform、scale），性能优秀
3. **状态管理**：通过 Action 系统实现撤销/重做，数据流清晰
4. **模块化**：tools/ 与 modules/ 分离，职责明确

### 6.2 优化建议

1. **Trim 性能**：改为单次遍历检测边界
2. **Resize 模式**：默认使用 Lanczos，提供质量/速度选项
3. **选区重叠**：考虑抽象出 `SelectionManager` 统一管理
4. **透视变换**：可作为独立模块扩展，不影响现有架构

### 6.3 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 选区基础类 | [src/js/core/base-selection.js](src/js/core/base-selection.js) |
| 对象选择工具 | [src/js/tools/select.js](src/js/tools/select.js) |
| 矩形选区工具 | [src/js/tools/selection.js](src/js/tools/selection.js) |
| 裁剪工具 | [src/js/tools/crop.js](src/js/tools/crop.js) |
| 旋转模块 | [src/js/modules/image/rotate.js](src/js/modules/image/rotate.js) |
| 翻转模块 | [src/js/modules/image/flip.js](src/js/modules/image/flip.js) |
| 缩放模块 | [src/js/modules/image/resize.js](src/js/modules/image/resize.js) |
| 裁边模块 | [src/js/modules/image/trim.js](src/js/modules/image/trim.js) |

---

*报告生成时间：2026-04-12*
