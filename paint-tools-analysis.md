# miniPaint 绘图工具横向分析报告

## 目录

1. [UI 功能与事件框架概述](#1-ui-功能与事件框架概述)
2. [Pencil vs Brush 深度对比](#2-pencil-vs-brush-深度对比)
3. [Fill 油漆桶算法分析](#3-fill-油漆桶算法分析)
4. [Erase 与 Clone 实现分析](#4-erase-与-clone-实现分析)
5. [Gradient 渐变实现分析](#5-gradient-渐变实现分析)
6. [代码组织评估与扩展建议](#6-代码组织评估与扩展建议)

---

## 1. UI 功能与事件框架概述

### 1.1 各工具 UI 功能说明

| 工具 | 文件 | UI 功能描述 |
|------|------|-------------|
| **Pencil（铅笔）** | `pencil.js` | 硬边画笔，绘制无抗锯齿的锐利线条，适合像素艺术或需要清晰边缘的场景 |
| **Brush（画笔）** | `brush.js` | 柔边画笔，支持压感和速度感应，绘制带有平滑过渡的柔和线条 |
| **Erase（橡皮擦）** | `erase.js` | 擦除像素，支持圆形/矩形两种模式，圆形模式可选柔边效果 |
| **Fill（油漆桶）** | `fill.js` | 区域填充，基于颜色相似度进行 flood fill，支持容差和抗锯齿 |
| **Clone（克隆图章）** | `clone.js` | 从源位置复制像素到目标位置，支持跨图层克隆 |
| **Gradient（渐变）** | `gradient.js` | 创建线性或径向渐变填充，支持双色渐变和透明度控制 |

### 1.2 统一基类：Base_tools_class

所有 6 个工具都继承自 [base-tools.js](src/js/core/base-tools.js) 的 `Base_tools_class`：

```javascript
class Brush_class extends Base_tools_class { ... }
class Pencil_class extends Base_tools_class { ... }
class Erase_class extends Base_tools_class { ... }
class Fill_class extends Base_tools_class { ... }
class Clone_class extends Base_tools_class { ... }
class Gradient_class extends Base_tools_class { ... }
```

### 1.3 事件处理框架

基类提供了两套事件绑定机制：

#### 方式一：`default_events()` 方法

由基类提供标准的三阶段事件绑定：

```javascript
// 在基类中定义
default_events() {
    document.addEventListener('mousedown', (e) => this.default_dragStart(e));
    document.addEventListener('mousemove', (e) => this.default_dragMove(e));
    document.addEventListener('mouseup', (e) => this.default_dragEnd(e));
    // 同时绑定 touch 事件...
}

default_dragStart(event) {
    if (config.TOOL.name != this.name) return;
    this.mousedown(event);
}
// dragMove 和 dragEnd 类似
```

**使用此方式的工具**：Pencil、Erase、Gradient

#### 方式二：自定义事件绑定

部分工具需要更复杂的事件处理（如多点触控、长按等），自行绑定事件：

**Brush** - 支持多点触控绘制：
```javascript
load() {
    // pointer events 用于压感检测
    document.addEventListener('pointerdown', (e) => this.pointerdown(e));
    document.addEventListener('pointermove', (e) => this.pointermove(e));
    
    // mouse + touch 事件分离处理
    document.addEventListener('mousedown', (e) => { if(!is_touch) this.dragStart(e); });
    document.addEventListener('touchstart', (e) => { is_touch = true; this.dragStart(e); });
    // ...
}
```

**Clone** - 需要右键设置源点：
```javascript
load() {
    document.addEventListener('mousedown', (e) => this.dragStart(e));
    document.addEventListener('contextmenu', (e) => this.mouseRightClick(e));
    // ...
}
```

**Fill** - 只需单击触发：
```javascript
load() {
    document.addEventListener('mousedown', (e) => this.dragStart(e));
    document.addEventListener('touchstart', (e) => this.dragStart(e));
}
```

### 1.4 三阶段处理函数组织

| 工具 | mousedown | mousemove | mouseup |
|------|-----------|-----------|---------|
| **Pencil** | 创建新图层或添加断点，记录起点 | 追加数据点到 `config.layer.data` | 计算边界并完成绘制 |
| **Brush** | 创建新图层/笔画组，记录起点和尺寸 | 追加带尺寸的点数据 | 计算边界并完成绘制 |
| **Erase** | 创建临时 Canvas，执行首次擦除 | 持续擦除并更新预览 | 提交历史记录更新图层 |
| **Fill** | 直接执行 flood fill 算法 | 无（单击操作） | 无 |
| **Clone** | 验证源点，创建临时 Canvas，首次克隆 | 持续克隆并更新预览 | 提交历史记录更新图层 |
| **Gradient** | 创建渐变图层，记录起点 | 实时更新渐变范围 | 确定渐变终点并完成 |

---

## 2. Pencil vs Brush 深度对比

### 2.1 核心差异概述

| 特性 | Pencil | Brush |
|------|--------|-------|
| 边缘效果 | 硬边（无抗锯齿） | 柔边（抗锯齿） |
| 压感支持 | 仅支持硬件压感 | 硬件压感 + 速度模拟 |
| 线条平滑 | 无平滑，逐点连线 | 三次平滑曲线（stabilized lines） |
| 数据格式 | `[x, y, size]` 或 `null`（断点） | `[[x, y, size], ...]` 二维数组（笔画组） |

### 2.2 数据结构差异

**Pencil** - 一维数组，`null` 表示笔画断开：
```javascript
config.layer.data = [
    [x1, y1, size1],
    [x2, y2, size2],
    null,  // 断点
    [x3, y3, size3],
    ...
];
```

**Brush** - 二维数组，每个子数组是一次完整笔画：
```javascript
config.layer.data = [
    [[x1, y1, size1], [x2, y2, size2], ...],  // 第一笔
    [[x3, y3, size3], [x4, y4, size4], ...],  // 第二笔
];
```

### 2.3 连线方式与插值处理

#### Pencil：逐像素连线（无插值）

```javascript
draw_simple_line(ctx, from_x, from_y, to_x, to_y, size) {
    var dist_x = from_x - to_x;
    var dist_y = from_y - to_y;
    var distance = Math.sqrt((dist_x * dist_x) + (dist_y * dist_y));
    var radiance = Math.atan2(dist_y, dist_x);

    for (var j = 0; j < distance; j++) {
        var x_tmp = Math.round(to_x + Math.cos(radiance) * j) - Math.floor(size / 2) - 1;
        var y_tmp = Math.round(to_y + Math.sin(radiance) * j) - Math.floor(size / 2) - 1;
        ctx.fillRect(x_tmp, y_tmp, size, size);  // 逐像素填充矩形
    }
}
```

**特点**：
- 使用 Bresenham 风格的逐点绘制
- 每个点用 `fillRect` 绘制方形像素
- **无平滑处理**，保持锐利边缘

#### Brush：三次贝塞尔曲线平滑

```javascript
render_stabilized(ctx, queue) {
    // 点数不足时直接连线
    if (data.length <= 5) {
        for (var i = 1; i < n; i++) {
            ctx.beginPath();
            ctx.moveTo(data[i - 1][0], data[i - 1][1]);
            ctx.lineTo(data[i][0], data[i][1]);
            ctx.stroke();
        }
        return;
    }

    // 三次平滑处理
    var temp_data1 = [data[0]];
    for (var i = 1; i < data.length - 1; i++) {
        c = (data[i][0] + data[i + 1][0]) / 2;  // 中点
        d = (data[i][1] + data[i + 1][1]) / 2;
        temp_data1.push([c, d]);
    }
    // 再进行两轮平滑...
    
    // 使用 quadraticCurveTo 绘制平滑曲线
    ctx.quadraticCurveTo(temp_data[i][0], temp_data[i][1], c, d);
}
```

**平滑算法来源**：[Stack Overflow - Drawing smooth lines with canvas](https://stackoverflow.com/questions/7891740/drawing-smooth-lines-with-canvas/44810470#44810470)

**特点**：
- 对原始点进行三次中点平均
- 使用 `quadraticCurveTo` 绘制贝塞尔曲线
- **自动平滑**用户手绘的抖动

### 2.4 快速移动时的断线处理

#### Pencil：无特殊处理

Pencil 直接连接相邻两点，如果用户快速移动导致 `mousemove` 事件间隔较大：

```javascript
// mousemove 中直接追加点
config.layer.data.push([
    Math.ceil(mouse.x - config.layer.x),
    Math.ceil(mouse.y - config.layer.y),
    new_size
]);
```

**问题**：当两点距离超过笔刷大小时，会出现明显的锯齿或断线。但由于是逐像素绘制，视觉效果相对可控。

#### Brush：无额外插值，但曲线平滑有补偿

Brush 同样只在 `mousemove` 时记录点：

```javascript
current_group.push([mouse_x - config.layer.x, mouse_y - config.layer.y, new_size]);
```

**但有两个补偿机制**：

1. **曲线平滑**：`render_stabilized` 会自动在点之间生成平滑曲线，视觉上填补空隙
2. **圆形线帽**：`ctx.lineCap = 'round'` 和 `ctx.lineJoin = 'round'` 使线条连接处圆滑

**结论**：两者都没有在事件层面做插值（如采样插值），但 Brush 通过曲线平滑和圆形线帽在渲染层面补偿了断线问题。

### 2.5 压感实现差异

#### Pencil：仅硬件压感

```javascript
if (params.pressure == true && this.pressure_supported) {
    new_size = size * this.pointer_pressure * 2;
}
```

#### Brush：硬件压感 + 速度模拟

```javascript
if (params.pressure == true) {
    if (this.pressure_supported) {
        // 硬件压感
        new_size = size * this.pointer_pressure * 2;
    } else {
        // 速度模拟：速度越快，笔触越细
        new_size = size + size / this.max_speed * mouse.speed_average * this.power;
        new_size = Math.max(new_size, size / 4);  // 最小为 1/4
    }
}
```

速度计算在基类中：
```javascript
calc_average_mouse_speed(event) {
    var dx = Math.abs(mouse.x - mouse.last_x);
    var dy = Math.abs(mouse.y - mouse.last_y);
    var delta = Math.sqrt(dx * dx + dy * dy);
    // 速度累积逻辑...
}
```

---

## 3. Fill 油漆桶算法分析

### 3.1 遍历算法：栈式 Flood Fill

Fill 工具使用**基于栈的迭代式 flood fill**，而非递归：

```javascript
fill_general(context, W, H, x, y, color_to, sensitivity, anti_aliasing, contiguous) {
    var stack = [];
    stack.push([x, y]);
    
    while (stack.length > 0) {
        var curPoint = stack.pop();
        for (var i = 0; i < 4; i++) {
            var nextPointX = curPoint[0] + dx[i];
            var nextPointY = curPoint[1] + dy[i];
            // 边界检查...
            // 颜色匹配检查...
            stack.push([nextPointX, nextPointY]);
        }
    }
}
```

**四方向数组**：
```javascript
var dx = [0, -1, +1, 0];
var dy = [-1, 0, 0, +1];
```

**为什么不用递归**：
- JavaScript 递归深度有限制（通常几千到几万）
- 大图填充会触发栈溢出
- 栈式迭代更可控，不会崩溃

### 3.2 容差（Tolerance）颜色比较

```javascript
// sensitivity 即 tolerance，从 0-100 转换为 0-255
sensitivity = sensitivity * 255 / 100;

// 颜色距离判断：各通道独立比较
if (Math.abs(imgData[k + 0] - color_from.r) <= sensitivity &&
    Math.abs(imgData[k + 1] - color_from.g) <= sensitivity &&
    Math.abs(imgData[k + 2] - color_from.b) <= sensitivity &&
    Math.abs(imgData[k + 3] - color_from.a) <= sensitivity) {
    // 匹配成功
}
```

**特点**：
- 使用**曼哈顿距离**（各通道绝对差之和），非欧氏距离
- 每个通道独立判断，必须全部满足
- 包含 Alpha 通道比较

### 3.3 两种填充模式

#### 模式一：连续填充（contiguous = false）

```javascript
if (contiguous == false) {
    var stack = [];
    stack.push([x, y]);
    while (stack.length > 0) {
        // 只填充与起点连通的相似颜色区域
    }
}
```

#### 模式二：全局填充（contiguous = true）

```javascript
else {
    // 遍历整张图，填充所有相似颜色
    for (var i = 0; i < imgData.length; i += 4) {
        if (颜色匹配) {
            imgData_tmp[k] = color_to.r;
            // ...
        }
    }
}
```

### 3.4 性能保护措施

**问题**：大图填充可能卡死浏览器吗？

**现有保护**：

1. **防重复执行**：
```javascript
if (this.working == true) {
    return;  // 正在执行时不响应新操作
}
this.working = true;
// ... 执行填充 ...
this.working = false;
```

2. **异步释放**：
```javascript
await new Promise(r => setTimeout(r, 10));  // 让出主线程 10ms
this.working = false;
```

**缺失的保护**：
- 没有像素数量限制
- 没有执行时间限制
- 没有进度提示

**潜在风险**：在超大画布（如 4000x4000）上填充大面积区域，仍可能导致明显卡顿。

### 3.5 抗锯齿实现

```javascript
if (anti_aliasing == true) {
    context.filter = 'blur(1px)';  // 使用 CSS filter 模糊
}
context.drawImage(canvasTemp, 0, 0);
```

**原理**：先在临时 Canvas 上绘制锐利边缘，然后通过 `blur(1px)` 滤镜柔化边缘。

---

## 4. Erase 与 Clone 实现分析

### 4.1 Erase：globalCompositeOperation 方案

#### 核心实现

```javascript
erase_general(ctx, type, mouse, size, strict, is_circle) {
    ctx.save();
    ctx.globalCompositeOperation = 'destination-out';  // 关键！
    
    if (is_circle == false) {
        // 矩形擦除
        ctx.fillStyle = "rgba(255, 255, 255, " + alpha / 255 + ")";
        ctx.fillRect(mouse_x - size_half, mouse_y - size_half, size, size);
    } else {
        // 圆形擦除
        if (strict == false) {
            // 柔边：径向渐变
            var radgrad = ctx.createRadialGradient(mouse_x, mouse_y, size / 8, mouse_x, mouse_y, size / 2);
            radgrad.addColorStop(0, "rgba(255, 255, 255, " + alpha / 255 + ")");
            radgrad.addColorStop(1, "rgba(255, 255, 255, 0)");
            ctx.fillStyle = radgrad;
        } else {
            ctx.fillStyle = "rgba(255, 255, 255, " + alpha / 255 + ")";
        }
        ctx.arc(mouse_x, mouse_y, size / 2, 0, Math.PI * 2, true);
        ctx.fill();
    }
    ctx.restore();
}
```

#### 为什么用 `destination-out` 而非直接设透明？

| 方案 | 实现方式 | 优缺点 |
|------|----------|--------|
| **直接设透明** | `clearRect()` 或修改 `ImageData` | 简单直接，但无法实现柔边效果 |
| **globalCompositeOperation** | `destination-out` | 支持渐变透明，可实现柔边擦除 |

**`destination-out` 原理**：
- 合成公式：`结果 = 目标 * (1 - 源)`
- 源像素越不透明，目标越透明
- 配合渐变可实现羽化边缘

#### 快速移动时的断线处理

```javascript
// 额外工作：如果鼠标移动太快，填充间隙
if (type == 'move' && is_circle == true && mouse_last_x != false) {
    ctx.save();
    ctx.globalCompositeOperation = 'destination-out';
    ctx.beginPath();
    ctx.moveTo(mouse_last_x, mouse_last_y);
    ctx.lineTo(mouse_x, mouse_y);
    ctx.stroke();  // 连接上一位置和当前位置
    ctx.restore();
}
```

**原理**：在两次 `mousemove` 位置之间画一条线，确保擦除连续。

### 4.2 Clone：源点偏移与跨图层支持

#### 源点记录

```javascript
// 右键或长按设置源点
mouseRightClick(e) {
    if (e.which == 3 && mouse.valid == true) {
        this.clone_coords = {
            x: mouse_x,
            y: mouse_y,
        };
        alertify.success('Source coordinates saved.');
    }
}

// 长按（2秒）
mouseLongClick() {
    this.clone_coords = { x: mouse_x, y: mouse_y };
    alertify.success('Source coordinates saved.');
}
```

#### 偏移计算

```javascript
clone_general(canvas_from, canvas_to, type, mouse) {
    // 源点位置 = 记录的源点 - (当前点击位置 - 当前鼠标位置)
    var x_from = Math.round(this.clone_coords.x - (mouse.click_x - mouse_x));
    var y_from = Math.round(this.clone_coords.y - (mouse.click_y - mouse_y));
    
    // 从源位置采样
    ctx_source.drawImage(canvas_from, x_from - half, y_from - half, w, h, 0, 0, w, h);
    
    // 绘制到目标位置
    canvas_to.getContext("2d").drawImage(canvas_source, mouse_x - half, mouse_y - half);
}
```

**偏移公式**：
```
源位置 = clone_coords - (click_position - current_position)
       = clone_coords - click_position + current_position
```

这意味着：鼠标从点击位置移动多少，源点就相应偏移多少。

#### 跨图层克隆

```javascript
if (params.source_layer.value == 'Previous') {
    var previous_layer = this.Base_layers.find_previous(config.layer.id);
    
    // 偏移需要考虑图层位置差异
    x_from = Math.round(this.clone_coords.x - (mouse.click_x - mouse_x)) 
             - previous_layer.x + config.layer.x;
    y_from = Math.round(this.clone_coords.y - (mouse.click_y - mouse_y)) 
             - previous_layer.y + config.layer.y;
    
    // 从上一个图层采样
    ctx_source.drawImage(previous_layer.link, x_from - half, y_from - half, w, h, 0, 0, w, h);
}
```

**跨图层偏移修正**：
- 源图层和目标图层可能有不同的 `x, y` 偏移
- 需要减去源图层偏移，加上目标图层偏移

#### 抗锯齿实现

```javascript
if (params.anti_aliasing == true) {
    var gradient = ctx_source.createRadialGradient(half, half, 0, half, half, half + 1);
    gradient.addColorStop(0, 'white');
    gradient.addColorStop(0.3, 'white');
    gradient.addColorStop(1, 'transparent');
    
    ctx_source.globalCompositeOperation = 'destination-in';
    ctx_source.fillRect(0, 0, params.size, params.size);
}
```

**原理**：使用径向渐变作为蒙版，通过 `destination-in` 合成操作实现边缘羽化。

---

## 5. Gradient 渐变实现分析

### 5.1 使用 Canvas 原生渐变 API

**Gradient 工具完全使用 Canvas 原生的渐变 API**，而非逐像素计算：

#### 线性渐变

```javascript
if (radial == false) {
    var grd = ctx.createLinearGradient(
        layer.x, layer.y,
        layer.x + layer.width - 1, layer.y + layer.height - 1
    );
    grd.addColorStop(0, color1);
    grd.addColorStop(1, "rgba(" + color2_rgb.r + ", " + color2_rgb.g + ", " + color2_rgb.b + ", " + alpha / 255 + ")");
    ctx.fillStyle = grd;
    ctx.fill();
}
```

#### 径向渐变

```javascript
else {
    var center_x = layer.x + Math.round(layer.width / 2);
    var center_y = layer.y + Math.round(layer.height / 2);
    var distance = Math.sqrt((layer.width * layer.width) + (layer.height * layer.height));
    
    var radgrad = ctx.createRadialGradient(
        center_x, center_y, distance * power / 100,  // 内圆
        center_x, center_y, distance                 // 外圆
    );
    radgrad.addColorStop(0, color1);
    radgrad.addColorStop(1, "rgba(" + color2_rgb.r + ", " + color2_rgb.b + ", " + color2_rgb.b + ", " + alpha / 255 + ")");
    ctx.fillStyle = radgrad;
    ctx.fillRect(0, 0, config.WIDTH, config.HEIGHT);
}
```

### 5.2 颜色参数来源

| 参数 | 来源 | 说明 |
|------|------|------|
| `color_1` | `params.color_1` | 渐变起始色（十六进制） |
| `color_2` | `params.color_2` | 渐变结束色（十六进制） |
| `alpha` | `params.alpha` | 结束色透明度（0-100 转换为 0-255） |
| `radial_power` | `params.radial_power` | 径向渐变内圆半径比例 |

### 5.3 色标（Color Stop）设置

目前只支持**双色渐变**，只有两个色标：

```javascript
grd.addColorStop(0, color1);      // 起点颜色
grd.addColorStop(1, color2_rgba); // 终点颜色（带透明度）
```

**不支持**：中间色标、多色渐变

### 5.4 交互方式

```javascript
mousedown(e) {
    // 记录起点
    this.layer = {
        x: mouse.x,
        y: mouse.y,
        data: { center_x: mouse.x, center_y: mouse.y }
    };
}

mousemove(e) {
    // 实时更新渐变范围
    var width = mouse.x - this.layer.x;
    var height = mouse.y - this.layer.y;
    
    if (params.radial == true) {
        config.layer.x = this.layer.data.center_x - width;
        config.layer.y = this.layer.data.center_y - height;
        config.layer.width = width * 2;
        config.layer.height = height * 2;
    } else {
        config.layer.width = width;
        config.layer.height = height;
    }
}
```

**交互特点**：
- 线性渐变：从起点拖拽到终点
- 径向渐变：从中心拖拽到边缘

---

## 6. 代码组织评估与扩展建议

### 6.1 代码一致性分析

#### 一致性较好的方面

| 方面 | 说明 |
|------|------|
| 基类继承 | 所有工具统一继承 `Base_tools_class` |
| 参数获取 | 统一使用 `this.getParams()` |
| 鼠标信息 | 统一使用 `this.get_mouse_info(event)` |
| 历史记录 | 统一使用 `app.State.do_action()` + `Actions` 模式 |
| 图层渲染 | 统一使用 `this.Base_layers.render()` |

#### 差异较大的方面

| 方面 | 差异说明 |
|------|----------|
| 事件绑定 | 3 种方式：`default_events()`、自定义绑定、仅 mousedown |
| 数据存储 | Brush/Pencil 存储点数据，Erase/Clone 直接修改像素，Gradient 存储几何参数 |
| 渲染方式 | Brush/Pencil/Gradient 有独立 `render()` 方法，Erase/Clone 直接操作 Canvas |
| 多点触控 | 仅 Brush 完整支持，其他工具基本忽略 |

### 6.2 重复代码分析

#### 可抽取的公共代码

**1. 压感检测逻辑**（Brush 和 Pencil 重复）：

```javascript
// 重复代码
pointerdown(e) {
    if (e.pressure && e.pressure !== 0 && e.pressure !== 0.5 && e.pressure <= 1) {
        this.pressure_supported = true;
        this.pointer_pressure = e.pressure;
    } else {
        this.pressure_supported = false;
    }
}
```

**建议**：抽取到 `Base_tools_class` 或创建 `PressureMixin`。

**2. 临时 Canvas 创建**（Erase 和 Clone 重复）：

```javascript
// 重复代码
this.tmpCanvas = document.createElement('canvas');
this.tmpCanvasCtx = this.tmpCanvas.getContext("2d");
this.tmpCanvas.width = config.layer.width_original;
this.tmpCanvas.height = config.layer.height_original;
this.tmpCanvasCtx.drawImage(config.layer.link, 0, 0);
```

**建议**：抽取为 `create_temp_canvas_from_layer()` 方法。

**3. 图层类型验证**（Erase、Clone、Fill 重复）：

```javascript
// 重复代码
if (config.layer.type != 'image') {
    alertify.error('This layer must contain an image...');
    return;
}
if (config.layer.is_vector == true) {
    alertify.error('Layer is vector...');
    return;
}
if (config.layer.rotate || 0 > 0) {
    alertify.error('...disabled. Please rasterize first.');
    return;
}
```

**建议**：抽取为 `validate_raster_layer()` 方法。

**4. 尺寸适配**（Clone 和 Fill 重复）：

```javascript
// 基类已有，但使用方式不一致
adaptSize(value, type = "width") { ... }
```

**建议**：统一调用方式。

### 6.3 建议的公共基类方法

```javascript
// 建议添加到 Base_tools_class

class Base_tools_class {
    // 压感支持
    init_pressure_support() { ... }
    get_pressure_size(base_size, params) { ... }
    
    // 临时 Canvas 管理
    create_temp_canvas() { ... }
    commit_temp_canvas_to_layer(tmpCanvas) { ... }
    
    // 图层验证
    validate_raster_layer() { ... }
    
    // 柔边绘制
    draw_soft_circle(ctx, x, y, size, alpha) { ... }
    
    // 断线填充
    fill_line_gap(ctx, from_x, from_y, to_x, to_y, size) { ... }
}
```

### 6.4 新增纹理笔刷的建议模板

如果要新增一个带纹理的笔刷工具，建议参考 **Brush** 进行抄改：

**原因**：

1. **数据结构成熟**：二维数组支持多笔画，每个点可携带额外属性
2. **平滑算法完善**：`render_stabilized` 已实现曲线平滑
3. **压感支持完整**：硬件压感 + 速度模拟
4. **渲染分离**：数据采集与渲染分离，便于扩展

**抄改步骤**：

```javascript
class TextureBrush_class extends Base_tools_class {
    constructor(ctx) {
        super();
        this.name = 'texture_brush';
        // 复用 Brush 的数据结构
    }
    
    load() {
        // 复用 Brush 的事件绑定方式
        // 添加 pointer events 用于压感
    }
    
    mousedown_action(e, index, event_identifier) {
        // 复用 Brush 的图层创建逻辑
        // 添加纹理参数
    }
    
    mousemove_action(e, index) {
        // 复用 Brush 的数据采集逻辑
        // 可添加纹理采样
    }
    
    render(ctx, layer) {
        // 修改渲染逻辑：使用纹理图案填充
        var pattern = ctx.createPattern(textureImage, 'repeat');
        ctx.strokeStyle = pattern;
        
        // 复用 render_stabilized 或自定义纹理渲染
    }
}
```

**关键修改点**：

| 功能 | Brush 实现 | 纹理笔刷修改 |
|------|-----------|-------------|
| 颜色 | `ctx.fillStyle = layer.color` | `ctx.fillStyle = ctx.createPattern(texture, 'repeat')` |
| 尺寸 | 固定或压感控制 | 可添加纹理缩放 |
| 边缘 | `lineCap = 'round'` | 可添加纹理边缘处理 |

---

## 总结

miniPaint 的 6 个绘图工具在架构上保持了较好的一致性，都基于 `Base_tools_class` 构建，但在具体实现上存在差异：

1. **Pencil vs Brush**：硬边 vs 柔边，逐像素连线 vs 贝塞尔平滑
2. **Fill**：栈式 flood fill，曼哈顿距离容差，有基础性能保护
3. **Erase**：`destination-out` 合成，支持柔边和断线填充
4. **Clone**：偏移公式 + 跨图层支持，径向渐变羽化
5. **Gradient**：原生 Canvas API，双色渐变

**改进建议**：
- 抽取压感、临时 Canvas、图层验证等公共代码
- 统一事件绑定方式
- 为 Fill 添加更完善的性能保护
- 新增纹理笔刷可参考 Brush 模板
