# miniPaint 图像几何变换与选区处理代码分析报告

## 1. 矩形选区系统架构

### 1.1 数据结构定义

矩形选区的核心数据结构定义在 `tools/select.js` 和 `tools/selection.js` 中：

```javascript
// 选区数据对象结构
this.selection = {
    x: null,        // 选区左上角 X 坐标（相对于画布）
    y: null,        // 选区左上角 Y 坐标（相对于画布）
    width: null,    // 选区宽度
    height: null    // 选区高度
};
```

在 `core/base-selection.js` 中，选区通过 `settings_all` 对象以工具名称为 key 存储，支持多工具独立管理各自的选区状态：

```javascript
const settings_all = [];

// 选区配置结构
var sel_config = {
    enable_background: false,   // 是否填充背景色
    enable_borders: true,       // 是否绘制边框
    enable_controls: true,      // 是否显示控制点
    keep_ratio: true,           // 是否保持宽高比
    enable_rotation: true,      // 是否启用旋转
    enable_move: true,          // 是否允许移动
    data_function: function () {
        return config.layer;    // 数据源函数
    },
};
```

### 1.2 鼠标事件到选区坐标的转换

**坐标转换流程**（`core/base-tools.js`）：

1. **原始事件坐标获取**：
```javascript
get_mouse_coordinates_from_event(event) {
    var mouse_x = event.pageX - this.Base_gui.canvas_offset.x;
    var mouse_y = event.pageY - this.Base_gui.canvas_offset.y;

    // 适配缩放比例
    var global_pos = this.Base_layers.get_world_coords(mouse_x, mouse_y);
    mouse_x = global_pos.x;
    mouse_y = global_pos.y;

    return { x: mouse_x, y: mouse_y };
}
```

2. **拖拽选区创建**（`tools/crop.js`）：
```javascript
mousedown(e) {
    var mouse = this.get_mouse_info(e);
    // 记录起始点
    this.Base_selection.set_selection(mouse.x, mouse.y, 0, 0);
}

mousemove(e) {
    var mouse = this.get_mouse_info(e);
    // 计算宽度和高度（支持负值，表示反向拖拽）
    var width = mouse.x - mouse.click_x;
    var height = mouse.y - mouse.click_y;
    this.Base_selection.set_selection(null, null, width, height);
}
```

3. **坐标规范化**（`mouseup` 时处理负宽高）：
```javascript
// 确保坐标不为负
if (details.width < 0) {
    x = x + details.width;  // 向左偏移
}
if (details.height < 0) {
    y = y + details.height; // 向上偏移
}
this.selection = {
    x: x,
    y: y,
    width: Math.abs(details.width),
    height: Math.abs(details.height)
};
```

### 1.3 虚线选框绘制机制

**绘制流程**（`core/base-selection.js` 的 `draw_selection` 方法）：

选框采用**双边框策略**实现虚线效果（类似 Photoshop 的蚂蚁线）：

```javascript
// 外边框：白色实线
this.ctx.lineWidth = wholeLineWidth;  // 2 / config.ZOOM
this.ctx.strokeStyle = 'rgb(255, 255, 255)';
this.ctx.strokeRect(x - halfLineWidth, y - halfLineWidth, w + wholeLineWidth, h + wholeLineWidth);

// 内边框：黑色实线（较细）
this.ctx.lineWidth = halfLineWidth;   // 1 / config.ZOOM
this.ctx.strokeStyle = 'rgb(0, 0, 0)';
this.ctx.strokeRect(x - wholeLineWidth, y - wholeLineWidth, w + (wholeLineWidth * 2), h + (wholeLineWidth * 2));
```

**视觉效果**：黑白双边框在任何背景下都清晰可见，通过缩放因子 `config.ZOOM` 保持线宽恒定。

**裁剪辅助线**（三分法网格）：
```javascript
if(settings.crop_lines === true) {
    // 垂直线
    for(var part = 1; part < 3; part++) {
        this.ctx.moveTo(x + w / 3 * part - halfLineWidth, y);
        this.ctx.lineTo(x + w / 3 * part - halfLineWidth, y + h);
    }
    // 水平线
    for(var part = 1; part < 3; part++) {
        this.ctx.moveTo(x, y + h / 3 * part - halfLineWidth);
        this.ctx.lineTo(x + w, y + h / 3 * part - halfLineWidth);
    }
}
```

### 1.4 全局状态管理

选区在全局状态中的存在形式：

1. **工具级选区状态**：
   - `tools/select.js`：管理图层对象的选中状态（`config.layer`）
   - `tools/selection.js`：管理像素级选区（`this.selection`）
   - `tools/crop.js`：管理裁剪选区（`this.selection`）

2. **渲染触发机制**：
```javascript
// 选区变化时标记需要重绘
config.need_render = true;
```

3. **状态持久化**：
   - 通过 `app.Actions.Set_selection_action` 将选区变更加入撤销栈
   - `Reset_selection_action` 用于清除选区

4. **控制点交互状态**：
```javascript
this.selected_obj_positions = {};        // 8个方向控制点位置
this.selected_obj_rotate_position = {};  // 旋转控制点位置
this.mouse_lock = null;                  // 当前锁定操作（resize/rotate/move）
this.selected_object_drag_type = null;   // 拖拽类型（DRAG_TYPE_TOP/BOTTOM/LEFT/RIGHT 的组合）
```

---

## 2. 裁剪操作实现分析

### 2.1 裁剪执行流程

**核心方法**：`tools/crop.js` 的 `on_params_update()`

```javascript
async on_params_update() {
    // 1. 验证选区有效性
    if (selection.width == null || selection.width == 0 || selection.height == 0) {
        alertify.error('Empty selection');
        return;
    }
    
    // 2. 检查旋转图层（不支持旋转图层裁剪）
    for (var i in config.layers) {
        if(link.rotate > 0) {
            alertify.error('Crop on rotated layer is not supported...');
            return;
        }
    }
    
    // 3. 遍历所有图层执行裁剪
    for (var i in config.layers) {
        // 移动图层位置
        x -= parseInt(selection.x);
        y -= parseInt(selection.y);
        
        if (link.type == 'image') {
            // 4. 创建新 canvas 并绘制可见区域
            let canvas = document.createElement('canvas');
            canvas.width = crop_width / width_ratio;
            canvas.height = crop_height / height_ratio;
            
            ctx.translate(-left / width_ratio, -top / height_ratio);
            ctx.drawImage(link.link, 0, 0);
            
            // 5. 更新图层图像
            actions.push(new app.Actions.Update_layer_image_action(canvas, link.id));
        }
    }
    
    // 6. 更新画布尺寸
    actions.push(new app.Actions.Update_config_action({
        WIDTH: parseInt(selection.width),
        HEIGHT: parseInt(selection.height)
    }));
}
```

### 2.2 图层处理方式

**裁剪后图层处理策略**：

| 方面 | 处理方式 |
|------|----------|
| **图像数据** | 生成新的 canvas，仅包含选区内像素 |
| **原图层** | 通过 `Update_layer_image_action` 替换图像数据 |
| **图层位置** | 重新计算相对于新画布原点的坐标 |
| **尺寸属性** | 更新 `width`、`height`、`width_original`、`height_original` |

**关键代码**：
```javascript
// 计算裁剪后的图像尺寸（考虑拉伸比例）
let width_ratio = layer.width / layer.width_original;
let height_ratio = layer.height / layer.height_original;

// 创建新 canvas，尺寸基于原始图像坐标系
canvas.width = crop_width / width_ratio;
canvas.height = crop_height / height_ratio;

// 通过 translate 定位裁剪区域
ctx.translate(-left / width_ratio, -top / height_ratio);
ctx.drawImage(link.link, 0, 0);
```

### 2.3 画布尺寸变化

**裁剪后画布调整**：

```javascript
// 裁剪前：保存画布状态
new app.Actions.Prepare_canvas_action('undo'),

// 更新画布配置
new app.Actions.Update_config_action({
    WIDTH: parseInt(selection.width),
    HEIGHT: parseInt(selection.height)
}),

// 裁剪后：恢复画布状态
new app.Actions.Prepare_canvas_action('do')
```

**行为特征**：
- 画布尺寸变为选区尺寸
- 所有图层坐标相对于新画布重新计算
- 位于选区外的图层内容被裁切
- 支持撤销/重做（通过 Action 机制）

---

## 3. 旋转与翻转实现

### 3.1 旋转实现方式

**实现策略**：使用 CSS/Canvas 变换属性（非像素矩阵运算）

**核心代码**（`modules/image/rotate.js`）：

```javascript
// 旋转仅更新角度属性，不修改像素数据
app.State.do_action(
    new app.Actions.Bundle_action('rotate_layer', 'Rotate Layer', [
        new app.Actions.Update_layer_action(config.layer.id, {
            rotate: new_rotate  // 0-360 度
        }),
        ...this.check_sizes(new_rotate)  // 调整画布/图层位置
    ])
);
```

**渲染时应用旋转**（由渲染引擎处理）：
```javascript
// 在 base-selection.js 的 draw_selection 中
if (data.rotate != null && data.rotate != 0) {
    this.ctx.translate(data.x + data.width / 2, data.y + data.height / 2);
    this.ctx.rotate(data.rotate * Math.PI / 180);
    x = Math.round(-data.width / 2);
    y = Math.round(-data.height / 2);
}
```

### 3.2 画布包围盒计算

**旋转后包围盒计算**（`check_sizes` 方法）：

```javascript
check_sizes(new_rotate) {
    let actions = [];
    var w = config.layer.width;
    var h = config.layer.height;

    // 计算旋转后的包围盒尺寸
    var o = new_rotate * Math.PI / 180;
    var new_x = w * Math.abs(Math.cos(o)) + h * Math.abs(Math.sin(o));
    var new_y = w * Math.abs(Math.sin(o)) + h * Math.abs(Math.cos(o));

    // 四舍五入
    new_x = Math.ceil(Math.round(new_x * 1000) / 1000);
    new_y = Math.ceil(Math.round(new_y * 1000) / 1000);

    // 如果超出画布，扩展画布
    if (new_x > config.WIDTH || new_y > config.HEIGHT) {
        var dx = Math.ceil(new_x - config.WIDTH) / 2;
        var dy = Math.ceil(new_y - config.HEIGHT) / 2;
        
        actions.push(
            new app.Actions.Update_layer_action(config.layer.id, {
                x: config.layer.x + dx,
                y: config.layer.y + dy
            }),
            new app.Actions.Update_config_action({
                WIDTH: new_width,
                HEIGHT: new_height
            })
        );
    }
    return actions;
}
```

**算法说明**：
- 使用三角函数计算旋转后矩形的外接矩形
- `new_x = w*|cos(θ)| + h*|sin(θ)|`
- `new_y = w*|sin(θ)| + h*|cos(θ)|`

### 3.3 翻转实现方式

**实现策略**：使用 Canvas transform API

**核心代码**（`modules/image/flip.js`）：

```javascript
flip(mode) {
    // 1. 获取图层 canvas
    var canvas = this.Base_layers.convert_layer_to_canvas(null, true);
    
    // 2. 创建目标 canvas
    var canvas2 = document.createElement('canvas');
    canvas2.width = canvas.width;
    canvas2.height = canvas.height;
    var ctx2 = canvas2.getContext("2d");
    
    // 3. 应用变换并绘制
    if (mode == 'vertical') {
        ctx2.scale(1, -1);
        ctx2.drawImage(canvas, 0, canvas2.height * -1);
    }
    else if (mode == 'horizontal') {
        ctx2.scale(-1, 1);
        ctx2.drawImage(canvas, canvas2.width * -1, 0);
    }
    
    // 4. 更新图层图像
    return app.State.do_action(
        new app.Actions.Update_layer_image_action(canvas2)
    );
}
```

**特点**：
- 翻转是**破坏性操作**，直接修改像素数据
- 使用 `ctx.scale()` 实现，性能高效
- 与旋转不同，翻转会生成新的图像数据

---

## 4. 缩放与自动裁剪

### 4.1 缩放插值算法

**支持三种缩放模式**（`modules/image/resize.js`）：

| 模式 | 算法 | 适用场景 | 特点 |
|------|------|----------|------|
| **Lanczos** | Lanczos 重采样 | 高质量缩放 | 使用 Pica 库，质量最高，支持放大 |
| **Hermite** | Hermite 插值 | 快速缩小 | 仅支持缩小，速度较快 |
| **Basic** | 最近邻/双线性 | 简单场景 | Canvas 原生 drawImage，最快 |

**Lanczos 实现**（使用 Pica 库）：
```javascript
if (mode == "Lanczos") {
    var tmp_data = document.createElement("canvas");
    tmp_data.width = width;
    tmp_data.height = height;
    
    await this.pica.resize(canvas, tmp_data, {
        alpha: true,  // 保留透明度
    })
    .then((result) => {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        canvas.width = width;
        canvas.height = height;
        ctx.drawImage(tmp_data, 0, 0, width, height);
    });
}
```

**Hermite 实现**：
```javascript
else if (mode == "Hermite") {
    // 使用 hermite-resize 库
    this.Hermite.resample_single(canvas, width, height, true);
}
```

**Basic 实现**：
```javascript
else {
    // 简单 canvas 缩放
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

### 4.2 抗锯齿处理

**锐化选项**：
```javascript
if (sharpen == true) {
    var imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    var filtered = this.ImageFilters.Sharpen(imageData, 1);
    ctx.putImageData(filtered, 0, 0);
}
```

- 使用 `ImageFilters.Sharpen` 卷积滤镜
- 在缩放后应用，减少模糊

### 4.3 Trim 自动裁剪实现

**边界检测算法**（`modules/image/trim.js`）：

```javascript
get_trim_info(layer_id, trim_white, power) {
    // 获取像素数据
    var img = ctx.getImageData(0, 0, canvas.width, canvas.height);
    var imgData = img.data;

    // 从四个方向扫描
    // 1. 检测顶部空白
    main1:
    for (var y = 0; y < img.height; y++) {
        for (var x = 0; x < img.width; x++) {
            var k = ((y * (img.width * 4)) + (x * 4));
            if (imgData[k + 3] <= power)  // Alpha 通道检查
                continue; // 透明
            if (trim_white == true && imgData[k] >= 255 - power && 
                imgData[k + 1] >= 255 - power && imgData[k + 2] >= 255 - power)
                continue; // 白色
            break main1;  // 找到非空像素
        }
        top++;
    }
    
    // 2. 检测左侧（类似逻辑）
    // 3. 检测底部（反向扫描）
    // 4. 检测右侧（反向扫描）
}
```

**扫描策略**：
- **Top**：从上到下逐行扫描
- **Left**：从左到右逐列扫描
- **Bottom**：从下到上逐行扫描
- **Right**：从右到左逐列扫描

**可配置参数**：
- `trim_white`：是否将白色视为空白
- `power`：透明度/颜色阈值（0-255）

### 4.4 性能评估

**Trim 性能分析**：

| 场景 | 时间复杂度 | 空间复杂度 | 优化建议 |
|------|-----------|-----------|----------|
| 小图 (<1MP) | O(n) | O(1) | 即时处理，无感知延迟 |
| 中图 (1-10MP) | O(n) | O(1) | 可能产生 100-500ms 延迟 |
| 大图 (>10MP) | O(n) | O(1) | 建议使用 Web Worker |

**潜在性能瓶颈**：
1. **全像素遍历**：每次 trim 都需要读取整个 ImageData
2. **同步执行**：在主线程执行，阻塞 UI
3. **重复计算**：没有缓存机制

**优化方向**：
```javascript
// 1. 使用抽样检测减少计算量
for (var y = 0; y < img.height; y += 2) {  // 隔行扫描
    for (var x = 0; x < img.width; x += 2) {  // 隔列扫描
        // ...
    }
}

// 2. 二分查找替代线性扫描
// 3. Web Worker 异步处理
```

---

## 5. 代码组织方式评估

### 5.1 目录结构分工

```
src/js/
├── tools/              # 交互式工具（用户直接操作）
│   ├── select.js       # 图层选择/变换工具
│   ├── selection.js    # 像素选区工具
│   ├── crop.js         # 裁剪工具
│   └── ...             # 画笔、橡皮等其他工具
├── modules/image/      # 图像处理模块（菜单命令）
│   ├── rotate.js       # 旋转
│   ├── flip.js         # 翻转
│   ├── resize.js       # 缩放
│   ├── trim.js         # 自动裁剪
│   ├── translate.js    # 平移
│   └── size.js         # 画布尺寸
└── core/
    ├── base-selection.js   # 选区绘制基类
    ├── base-tools.js       # 工具基类
    └── base-layers.js      # 图层管理
```

### 5.2 职责划分

| 目录 | 职责 | 触发方式 | 操作对象 |
|------|------|----------|----------|
| `tools/` | 交互式工具 | 鼠标/键盘交互 | 图层/选区 |
| `modules/image/` | 批量图像处理 | 菜单命令 | 图层像素数据 |
| `core/` | 基础设施 | 内部调用 | 全局状态/渲染 |

### 5.3 功能重叠与职责不清

**存在的问题**：

1. **裁剪功能分散**：
   - `tools/crop.js`：交互式裁剪（带选区）
   - `modules/image/trim.js`：自动裁剪（基于透明度）
   - **建议**：统一裁剪接口，区分自动/手动模式

2. **尺寸调整多处实现**：
   - `modules/image/resize.js`：图层缩放（插值算法）
   - `modules/image/size.js`：画布尺寸调整
   - `tools/select.js`：拖拽缩放（无插值）
   - **建议**：提取公共缩放逻辑

3. **旋转实现不一致**：
   - `tools/select.js`：交互式旋转（仅更新属性）
   - `modules/image/rotate.js`：精确角度旋转（同上）
   - **建议**：旋转操作统一走 Action 机制

### 5.4 新增透视变换的改动点

如果要实现自由透视变换（Perspective Transform），需要修改以下文件：

**核心改动**：

1. **`core/base-selection.js`**：
   - 添加 4 个角点控制（替代现有的 8 个方向控制）
   - 实现透视变换矩阵计算
   - 添加透视预览渲染

2. **`tools/select.js`**：
   - 添加透视变换模式切换
   - 处理 4 角点拖拽逻辑

3. **新增 `modules/image/perspective.js`**：
```javascript
class Image_perspective_class {
    // 1. 计算透视变换矩阵
    // 2. 应用变换到 canvas
    // 3. 使用 ctx.setTransform() 或手动像素映射
}
```

4. **`src/js/config.js`**：
   - 添加透视变换工具配置

5. **渲染管线**：
   - 修改图层渲染逻辑，支持 `transform` 矩阵
   - 或采用离屏 canvas 预渲染方案

**实现方案对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| CSS transform | 性能好，简单 | 仅矩形变换，不支持透视 |
| Canvas setTransform | 支持矩阵变换 | 需要计算 3x3 矩阵 |
| 像素重采样 | 质量高 | 性能差，复杂 |

---

## 6. 总结

### 6.1 架构亮点

1. **Action 机制**：所有操作通过 Action 封装，支持完善的撤销/重做
2. **分层渲染**：选区、图层、背景分离渲染，逻辑清晰
3. **插值算法选择**：提供质量与性能的平衡选项
4. **模块化设计**：工具与图像处理分离，便于扩展

### 6.2 改进建议

1. **性能优化**：
   - Trim 操作使用 Web Worker 异步化
   - 大图像缩放添加进度提示

2. **代码重构**：
   - 提取公共几何变换接口
   - 统一裁剪/缩放/旋转的 Action 封装

3. **功能增强**：
   - 支持非矩形选区（多边形、套索）
   - 添加透视变换支持
   - 实现智能参考线（对齐辅助）
