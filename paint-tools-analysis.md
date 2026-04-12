# miniPaint 手绘/填充类工具横向分析报告

## 一、工具概述与事件框架

### 1.1 各工具功能简介

| 工具 | UI 功能描述 | 核心特性 |
|------|-------------|----------|
| **brush.js** | 画笔工具 | 支持压感（硬件/模拟）、柔边效果、线条平滑稳定、多笔触支持 |
| **pencil.js** | 铅笔工具 | 硬边像素风格、无抗锯齿、支持压感、复古像素画效果 |
| **erase.js** | 橡皮擦工具 | 支持圆形/矩形擦除、硬边/柔边模式、透明度控制 |
| **fill.js** | 油漆桶填充 | Flood Fill 算法、容差控制、连续/全局模式、抗锯齿选项 |
| **clone.js** | 克隆图章 | 从源位置采样绘制到目标位置、支持跨图层、柔边选项 |
| **gradient.js** | 渐变工具 | 线性/径向渐变、双色调、透明度控制、矢量图层支持 |

### 1.2 统一接入的基类与事件框架

所有 6 个工具都继承自 `Base_tools_class`（[base-tools.js](file:///Users/dengquan/Downloads/job/bytedance/code/prod/dogfooding-3-831/kimi/src/js/core/base-tools.js)）：

```javascript
// 统一继承关系
class Brush_class extends Base_tools_class
class Pencil_class extends Base_tools_class
class Erase_class extends Base_tools_class
class Fill_class extends Base_tools_class
class Clone_class extends Base_tools_class
class Gradient_class extends Base_tools_class
```

**基类提供的基础设施：**

| 功能 | 方法/属性 | 说明 |
|------|-----------|------|
| 鼠标状态追踪 | `config.mouse` | 统一存储 x/y、last_x/last_y、click_x/click_y、is_drag、speed_average 等 |
| 坐标转换 | `get_mouse_coordinates_from_event()` | 处理 zoom 和 canvas offset |
| 参数获取 | `getParams()` | 从 config.TOOL.attributes 读取工具参数 |
| 自定义光标 | `show_mouse_cursor()` | 显示圆形/矩形辅助光标 |
| 尺寸适配 | `adaptSize()` | 处理图层缩放后的坐标转换 |
| 事件绑定模板 | `default_events()` | 标准 mousedown/mousemove/mouseup 绑定 |

### 1.3 三阶段处理函数组织方式

各工具的事件处理组织方式分为两类：

**类型 A：使用 default_events() 标准模板（pencil, erase, gradient）**

```javascript
load() {
    this.default_events();  // 自动绑定 mousedown/mousemove/mouseup
}

// 直接实现三阶段回调
mousedown(e) { ... }
mousemove(e) { ... }
mouseup(e) { ... }
```

**类型 B：自定义事件绑定（brush, fill, clone）**

```javascript
// brush.js - 支持多指触控，需要维护 event_links 映射
load() {
    document.addEventListener('pointerdown', ...);
    document.addEventListener('pointermove', ...);
    // 同时支持 mouse 和 touch
}

dragStart(event) { ... }  // 分发到 mousedown_action
dragMove(event) { ... }   // 分发到 mousemove_action
dragEnd(event) { ... }    // 分发到 mouseup_action

// fill.js - 只有点击操作，无拖拽
load() {
    document.addEventListener('mousedown', ...);
    document.addEventListener('touchstart', ...);
}

// clone.js - 需要处理右键设置源点
dragStart/dragMove/dragEnd + mouseRightClick()
```

---

## 二、Pencil vs Brush 深度对比

### 2.1 核心差异概览

| 特性 | Pencil (铅笔) | Brush (画笔) |
|------|---------------|--------------|
| 边缘效果 | 硬边（无抗锯齿） | 柔边（抗锯齿） |
| 线条平滑 | 无 | 三次平滑插值 |
| 压感支持 | 硬件压感 only | 硬件压感 + 模拟速度压感 |
| 绘制方式 | `fillRect` 像素块 | `stroke` + `quadraticCurveTo` |
| 数据格式 | 一维数组 `[x, y, size]` | 二维数组分组 `[[[x,y,size], ...], ...]` |

### 2.2 鼠标移动时的插值/平滑处理

**Pencil - 无插值，直接连线段：**

```javascript
// pencil.js draw_simple_line()
for (var j = 0; j < distance; j++) {
    var x_tmp = Math.round(to_x + Math.cos(radiance) * j) - Math.floor(size / 2) - 1;
    var y_tmp = Math.round(to_y + Math.sin(radiance) * j) - Math.floor(size / 2) - 1;
    ctx.fillRect(x_tmp, y_tmp, size, size);  // 逐像素填充矩形
}
```

Pencil 在 `render_aliased()` 中直接遍历数据点，用 `draw_simple_line()` 在相邻点之间画直线，**不做任何曲线平滑**。

**Brush - 三次平滑插值（Stabilized Lines）：**

```javascript
// brush.js render_stabilized()
// 1. 复制最后一个点解决 Loose Ending
// 2. 三次平均滤波生成 temp_data1, temp_data2, temp_data
// 3. 使用 quadraticCurveTo 绘制平滑曲线

for (var i = 1; i < temp_data.length - 2; i = i+1) {
    c = (temp_data[i][0] + temp_data[i + 1][0]) / 2;
    d = (temp_data[i][1] + temp_data[i + 1][1]) / 2;
    ctx.quadraticCurveTo(temp_data[i][0], temp_data[i][1], c, d);
}
```

算法来源：[StackOverflow - Drawing smooth lines with canvas](https://stackoverflow.com/questions/7891740/drawing-smooth-lines-with-canvas/44810470#44810470)

### 2.3 快速移动时的断线处理

**Pencil - 不处理断线：**

Pencil 完全依赖 `mousemove` 事件的触发频率，如果移动过快导致采样点稀疏，画出的线条会出现明显断点。代码中没有针对此情况的补偿逻辑。

**Brush - 依赖 Canvas 的 stroke 自动处理：**

Brush 使用 `ctx.stroke()` 绘制路径，Canvas 会自动连接路径中的点。但对于压感模式（非稳定线条模式），Brush 也是逐段绘制直线：

```javascript
// brush.js render() 压感模式
for (var i = 1; i < group_n; i++) {
    ctx.beginPath();
    ctx.moveTo(group_data[i - 1][0], group_data[i - 1][1]);
    ctx.lineTo(group_data[i][0], group_data[i][1]);
    ctx.stroke();
}
```

**Erase - 显式处理快速移动间隙：**

```javascript
// erase.js - 在圆形模式下，用线段填充快速移动造成的间隙
if (type == 'move' && is_circle == true && mouse_last_x != false && mouse_last_y != false) {
    ctx.save();
    ctx.globalCompositeOperation = 'destination-out';
    ctx.beginPath();
    ctx.moveTo(mouse_last_x, mouse_last_y);
    ctx.lineTo(mouse_x, mouse_y);
    ctx.stroke();  // 用线段连接前后位置
    ctx.restore();
}
```

### 2.4 压感实现对比

**Pencil - 仅硬件压感：**

```javascript
if (params.pressure == true && this.pressure_supported) {
    new_size = size * this.pointer_pressure * 2;
}
```

**Brush - 硬件压感 + 模拟速度压感：**

```javascript
if (params.pressure == true) {
    if (this.pressure_supported) {
        new_size = size * this.pointer_pressure * 2;
    } else {
        // 模拟压感：速度越快，线条越细
        new_size = size + size / this.max_speed * mouse.speed_average * this.power;
        new_size = Math.max(new_size, size / 4);
        new_size = Math.round(new_size);
    }
}
```

---

## 三、Fill.js 油漆桶实现分析

### 3.1 遍历算法：栈式扫描线（非递归）

```javascript
// fill.js fill_general()
var stack = [];
stack.push([x, y]);
while (stack.length > 0) {
    var curPoint = stack.pop();
    for (var i = 0; i < 4; i++) {  // 四邻域：上、左、右、下
        var nextPointX = curPoint[0] + dx[i];
        var nextPointY = curPoint[1] + dy[i];
        // ... 边界检查、颜色容差检查
        if (/* 颜色匹配 */) {
            // 填充像素
            imgData_tmp[k] = color_to.r;
            imgData_tmp[k + 1] = color_to.g;
            imgData_tmp[k + 2] = color_to.b;
            imgData_tmp[k + 3] = color_to.a;
            stack.push([nextPointX, nextPointY]);
        }
    }
}
```

**算法选择：** 使用显式栈而非递归，避免大区域填充时的栈溢出问题。

### 3.2 容差（Tolerance）比较逻辑

```javascript
sensitivity = sensitivity * 255 / 100;  // 转换为 0-255 区间

// 逐通道绝对值差比较
if (Math.abs(imgData[k + 0] - color_from.r) <= sensitivity &&
    Math.abs(imgData[k + 1] - color_from.g) <= sensitivity &&
    Math.abs(imgData[k + 2] - color_from.b) <= sensitivity &&
    Math.abs(imgData[k + 3] - color_from.a) <= sensitivity) {
    // 匹配，填充
}
```

**注意：** 容差是基于起点颜色 `color_from` 的绝对差值，而非颜色空间距离（如欧几里得距离）。

### 3.3 性能保护机制

| 机制 | 实现 | 说明 |
|------|------|------|
| 工作状态锁 | `this.working` 标志 | 防止重复触发，10ms 延迟解锁 |
| 异步处理 | `async fill()` | 使用 Promise 延迟，避免阻塞主线程 |
| 栈式遍历 | 显式栈代替递归 | 避免调用栈溢出 |
| 临时画布 | `canvasTemp` | 双缓冲，减少直接操作原图 |

**潜在风险：** 对于超大图像（如 4K+），flood fill 仍可能耗时较长，代码中没有设置最大迭代次数或区域面积限制。

### 3.4 两种填充模式

```javascript
if (contiguous == false) {
    // 连续模式：四邻域栈式 flood fill
    // 只填充与起点连通的区域
} else {
    // 全局模式：遍历全图所有像素
    // 填充所有匹配颜色的像素（类似魔棒选择）
    for (var i = 0; i < imgData.length; i += 4) {
        // 逐像素比较
    }
}
```

---

## 四、Erase.js 与 Clone.js 实现分析

### 4.1 橡皮擦的合成方式

Erase.js 使用 **`globalCompositeOperation = 'destination-out'`** 实现擦除：

```javascript
// 矩形擦除
ctx.save();
ctx.globalCompositeOperation = 'destination-out';
ctx.fillStyle = "rgba(255, 255, 255, " + alpha / 255 + ")";
ctx.fillRect(mouse_x - size_half, mouse_y - size_half, size, size);
ctx.restore();

// 圆形柔边擦除
ctx.save();
ctx.globalCompositeOperation = 'destination-out';
if (strict == true)
    ctx.fillStyle = "rgba(255, 255, 255, " + alpha / 255 + ")";
else
    ctx.fillStyle = radgrad;  // 径向渐变实现柔边
ctx.beginPath();
ctx.arc(mouse_x, mouse_y, size / 2, 0, Math.PI * 2, true);
ctx.fill();
ctx.restore();
```

**为什么用 destination-out 而不是直接设透明？**

| 方式 | 优点 | 缺点 |
|------|------|------|
| `destination-out` | 利用 GPU 加速、支持柔边渐变、代码简洁 | 需要临时画布保存状态 |
| 直接修改 pixel data | 精确控制每个像素、可自定义算法 | CPU 密集型、无硬件加速、柔边实现复杂 |

Erase 选择 `destination-out` 是正确且高效的做法。

### 4.2 克隆图章的源点/目标点偏移记录

**源点记录（右键或长按）：**

```javascript
// clone.js mouseRightClick() / mouseLongClick()
this.clone_coords = {
    x: mouse_x,  // 适配原始尺寸后的坐标
    y: mouse_y,
};
```

**偏移计算（绘制时）：**

```javascript
// 计算源采样位置：源点 - (点击位置 - 当前位置)
var x_from = Math.round(this.clone_coords.x - (mouse.click_x - mouse_x));
var y_from = Math.round(this.clone_coords.y - (mouse.click_y - mouse_y));
```

这意味着克隆图章保持**相对偏移不变**：无论你把画笔移动到哪里，它始终从源点平移相同偏移的位置采样。

### 4.3 多图层支持

```javascript
if (params.source_layer.value == 'Previous') {
    var previous_layer = this.Base_layers.find_previous(config.layer.id);
    
    // 调整坐标到前一图层坐标系
    x_from = Math.round(this.clone_coords.x - (mouse.click_x - mouse_x)) 
             - previous_layer.x + config.layer.x;
    y_from = Math.round(this.clone_coords.y - (mouse.click_y - mouse_y)) 
             - previous_layer.y + config.layer.y;
    
    ctx_source.drawImage(previous_layer.link, x_from - half, y_from - half, w, h, 0, 0, w, h);
} else {
    // 当前图层模式
    ctx_source.drawImage(canvas_from, x_from - half, y_from - half, w, h, 0, 0, w, h);
}
```

**多图层处理流程：**
1. 创建临时 `canvas_source` 作为采样缓冲区
2. 根据设置决定从 `previous_layer.link` 还是当前图层采样
3. 应用抗锯齿蒙版（可选）
4. 绘制到目标画布

---

## 五、Gradient.js 渐变实现分析

### 5.1 使用 Canvas 原生渐变 API

**线性渐变：**

```javascript
var grd = ctx.createLinearGradient(
    layer.x, layer.y,
    width, height);

grd.addColorStop(0, color1);
grd.addColorStop(1, "rgba(" + color2_rgb.r + ", " + color2_rgb.g + ", "
    + color2_rgb.b + ", " + alpha / 255 + ")");
ctx.fillStyle = grd;
ctx.fill();
```

**径向渐变：**

```javascript
var radgrad = ctx.createRadialGradient(
    center_x, center_y, distance * power / 100,  // 内圆
    center_x, center_y, distance);                // 外圆

radgrad.addColorStop(0, color1);
radgrad.addColorStop(1, "rgba(" + color2_rgb.r + ", " + color2_rgb.g + ", "
    + color2_rgb.b + ", " + alpha / 255 + ")");
ctx.fillStyle = radgrad;
ctx.fillRect(0, 0, config.WIDTH, config.HEIGHT);
```

### 5.2 颜色来源

```javascript
// 从工具参数读取
var color1 = params.color_1;  // 起点/中心颜色
var color2 = params.color_2;  // 终点/边缘颜色
var alpha = params.alpha / 100 * 255;  // 全局透明度

// color2 需要转换为 RGB 以支持透明度
var color2_rgb = this.Helper.hexToRgb(color2);
```

**注意：** Gradient 工具**不使用** `config.COLOR`（当前选中的前景色），而是使用工具参数中预设的双色。

### 5.3 矢量图层支持

```javascript
// 径向渐变标记为矢量图层
if (params.radial == true) {
    name = 'Radial gradient';
    is_vector = true;  // 可编辑
}
```

线性渐变在创建后不可再编辑（`is_vector = false`），径向渐变支持后续调整。

---

## 六、代码组织评估与重构建议

### 6.1 一致性分析

| 方面 | 一致性评分 | 说明 |
|------|------------|------|
| 继承基类 | ⭐⭐⭐⭐⭐ | 全部继承 Base_tools_class |
| 事件处理 | ⭐⭐⭐ | 3个用 default_events，3个自定义 |
| 图层管理 | ⭐⭐⭐⭐ | 都使用 app.State.do_action 操作 |
| 参数读取 | ⭐⭐⭐⭐⭐ | 统一使用 getParams() |
| 临时画布 | ⭐⭐⭐ | erase/clone 用 tmpCanvas，其他直接渲染 |
| 压感支持 | ⭐⭐ | 只有 brush/pencil 支持 |

### 6.2 可抽取的公共基类

**建议新增 `Base_draw_tool_class` 继承自 `Base_tools_class`：**

```javascript
class Base_draw_tool_class extends Base_tools_class {
    // 通用属性
    this.tmpCanvas = null;
    this.tmpCanvasCtx = null;
    this.started = false;
    
    // 通用方法
    create_tmp_canvas() { ... }
    dispose_tmp_canvas() { ... }
    
    // 压感相关
    pointerdown(e) { ... }
    pointermove(e) { ... }
    
    // 快速移动间隙填充
    fill_gaps(ctx, x1, y1, x2, y2) { ... }
}
```

**可复用代码片段：**

1. **压感检测逻辑**（brush/pencil 几乎相同）
2. **临时画布创建/销毁**（erase/clone 相同）
3. **图层类型检查**（erase/fill/clone 都检查是否为 image 类型）
4. **坐标适配**（`adaptSize` 已提取到基类）

### 6.3 新增纹理笔刷工具的参考模板

如果要新增带纹理的笔刷工具，**建议参考 brush.js**，原因：

1. **数据结构最完善**：支持多笔触分组、压感大小变化
2. **渲染最灵活**：分离了稳定线条模式和压感模式
3. **触控支持**：已处理多指触控的 event_links 映射

**改造点：**

```javascript
// 在 render() 中增加纹理采样
render(ctx, layer) {
    // ... 原有代码 ...
    
    // 加载纹理图案
    var pattern = ctx.createPattern(texture_image, 'repeat');
    ctx.fillStyle = pattern;
    
    // 用纹理绘制每个点
    for (...) {
        ctx.beginPath();
        ctx.arc(x, y, size/2, 0, Math.PI * 2);
        ctx.fill();
    }
}
```

---

## 七、总结

| 工具 | 核心技术 | 性能特点 | 可改进点 |
|------|----------|----------|----------|
| Brush | 三次平滑 + 压感 | 中等（平滑计算开销）| 支持纹理、更多笔刷预设 |
| Pencil | 像素硬边 | 高（无抗锯齿计算）| 增加断线补偿 |
| Erase | destination-out | 高（GPU 加速）| 支持更多笔刷形状 |
| Fill | 栈式 flood fill | 依赖图像大小 | 增加迭代限制防卡死 |
| Clone | 双缓冲采样 | 中等（临时画布开销）| 支持对齐、透明度预览 |
| Gradient | 原生渐变 API | 极高 | 支持更多色标、角度控制 |

整体而言，miniPaint 的工具代码组织较为清晰，基类抽象合理，但仍有部分重复代码可进一步抽取。各工具根据功能需求选择了合适的底层实现方式，在性能和效果之间取得了较好的平衡。
