# miniPaint 手绘/填充类工具底层实现横向分析报告

## 一、整体架构与事件框架

### 1.1 工具基类统一接入

所有 6 个工具都继承自 `Base_tools_class`（位于 `src/js/core/base-tools.js`），形成了统一的继承体系：

| 工具文件 | 继承关系 | 事件注册方式 |
|---------|---------|------------|
| pencil.js | extends Base_tools_class | load() 中注册 pointer 事件 + default_events() |
| brush.js | extends Base_tools_class | load() 中分别注册 mouse/touch 事件 |
| erase.js | extends Base_tools_class | load() 中调用 default_events() |
| fill.js | extends Base_tools_class | load() 中注册 dragStart 事件 |
| clone.js | extends Base_tools_class | load() 中分别注册 mouse/touch 事件 |
| gradient.js | extends Base_tools_class | load() 中调用 default_events() |

### 1.2 事件处理三阶段组织

**基类提供的标准事件处理流程：**

```
mousedown/touchstart → dragStart → default_dragStart → mousedown()
mousemove/touchmove → dragMove → default_dragMove → mousemove()
mouseup/touchend → dragEnd → default_dragEnd → mouseup()
```

**各工具事件处理实现差异：**

1. **pencil.js**：标准三阶段（mousedown/mousemove/mouseup）+ 独立 pointer 事件处理压感
2. **brush.js**：封装为 action 层（mousedown_action/mousemove_action/mouseup_action），支持多指触摸
3. **erase.js**：标准三阶段，move 阶段额外显示鼠标光标
4. **fill.js**：仅处理 mousedown，无拖动过程
5. **clone.js**：标准三阶段 + 右键 + 长按事件
6. **gradient.js**：标准三阶段

---

## 二、Pencil vs Brush：连续画线的实现差异

### 2.1 UI 功能定位

| 维度 | Pencil（铅笔） | Brush（画笔） |
|-----|--------------|-------------|
| 边缘特性 | 硬边、无抗锯齿 | 柔边、圆形笔触 |
| 压感支持 | ✅ 支持 | ✅ 支持 |
| 速度感应 | ❌ 不支持 | ✅ 无压感时用速度模拟 |
| 平滑处理 | ❌ 无平滑 | ✅ 二次贝塞尔曲线平滑 |
| 绘制方式 | 逐像素 fillRect | Canvas lineTo + arc |

### 2.2 鼠标移动时的插值与平滑处理

**Pencil 的实现（`pencil.js:245-257` draw_simple_line）：**

```javascript
// 对两点之间的每个像素进行步进填充
for (var j = 0; j < distance; j++) {
    var x_tmp = Math.round(to_x + Math.cos(radiance) * j) - Math.floor(size / 2) - 1;
    var y_tmp = Math.round(to_y + Math.sin(radiance) * j) - Math.floor(size / 2) - 1;
    ctx.fillRect(x_tmp, y_tmp, size, size);
}
```

**关键点分析：**
- ✅ **不会断开**：使用极坐标步进法，沿直线方向每隔 1 个像素填充一次
- 算法：计算两点距离和角度，沿方向逐点填充
- 效果：即使移动很快，两点间也会被完整的像素填满

**Brush 的实现（`brush.js:421-489` render_stabilized）：**

```javascript
// 三级平滑 + 二次贝塞尔曲线
// 1. 第一轮平均：相邻点取中点
for (var i = 1; i < data.length - 1;  i = i+1) {
    c = (data[i][0] + data[i + 1][0]) / 2;
    d = (data[i][1] + data[i + 1][1]) / 2;
    temp_data1.push([c, d]);
}
// 2. 第二轮、第三轮重复平均...
// 3. 最终用 quadraticCurveTo 绘制贝塞尔曲线
ctx.quadraticCurveTo(temp_data[i][0], temp_data[i][1], c, d);
```

**关键点分析：**
- ✅ **不会断开**：平滑算法本质是对原始点云进行多次插值
- 算法：三级中点平均 + 二次贝塞尔曲线拟合
- 压感模式：退化为简单 lineTo，但由于记录了每一步 size，粗细变化连续

### 2.3 高速移动防断开策略

| 工具 | 策略 | 实现位置 |
|-----|------|---------|
| Pencil | 极坐标步进逐像素填充 | `pencil.js:251-256` |
| Brush | 1. 压感关闭时：三级平滑插值<br>2. 压感开启时：逐点 lineTo 连接 | `brush.js:394` |

---

## 三、Fill 油漆桶：Flood Fill 算法深度分析

### 3.1 遍历算法选择

**Fill 采用 **栈式 4 邻域扫描线** 算法（非递归）：**

```javascript
// fill.js:169-197 栈式深度优先遍历
var stack = [];
stack.push([x, y]);
while (stack.length > 0) {
    var curPoint = stack.pop();
    for (var i = 0; i < 4; i++) {  // 上下左右四方向
        var nextPointX = curPoint[0] + dx[i];
        var nextPointY = curPoint[1] + dy[i];
        // 边界检查 + 已访问检查
        // 颜色容差检查
        // 符合条件则入栈
    }
}
```

**算法特点：**
- ✅ **避免栈溢出**：使用数组模拟栈，而非函数递归调用
- ✅ **4 邻域**：只检查上下左右，不包含对角线
- ❌ **非扫描线优化**：是朴素的像素级遍历，非优化的扫描线算法

### 3.2 Tolerance 容差与颜色距离计算

```javascript
// fill.js:183-186 各通道独立比较
if (Math.abs(imgData[k + 0] - color_from.r) <= sensitivity &&
    Math.abs(imgData[k + 1] - color_from.g) <= sensitivity &&
    Math.abs(imgData[k + 2] - color_from.b) <= sensitivity &&
    Math.abs(imgData[k + 3] - color_from.a) <= sensitivity)
```

**颜色距离计算方式：**
- **曼哈顿距离（L1）**：R、G、B、A 四个通道分别计算绝对差
- **独立阈值判断**：每个通道都必须小于 sensitivity，而非整体欧氏距离
- **参数转换**：power 参数（0-100）线性映射到 0-255

### 3.3 大图性能保护机制

| 保护措施 | 实现说明 |
|---------|---------|
| 边界检查 | `nextPointX < 0 || nextPointY < 0 || nextPointX >= W || nextPointY >= H` |
| 已访问标记 | 使用临时画布 alpha 通道标记（imgData_tmp[k + 3] != 0） |
| 非 contiguous 模式 | **O(1) 全局遍历**：直接逐像素比较，不使用栈 |
| 防抖保护 | fill.js:131 `await new Promise(r => setTimeout(r, 10))` 防止触摸屏连击 |

**潜在风险：**
- 超大图（>4000×4000）在 contiguous=true 时，最坏情况 O(n) 全图遍历
- 但由于是纯数组操作，现代浏览器一般不会卡死，只是短暂无响应

---

## 四、Erase 橡皮擦与 Clone 克隆图章

### 4.1 Erase 橡皮擦的实现方式

**采用 globalCompositeOperation 合成方式（推荐做法）：**

```javascript
// erase.js:160-163, 181
ctx.save();
ctx.globalCompositeOperation = 'destination-out';  // 关键！
ctx.fillStyle = "rgba(255, 255, 255, " + alpha / 255 + ")";
ctx.fillRect(mouse_x - size_half, mouse_y - size_half, size, size);
ctx.restore();
```

**两种方式对比：**

| 方式 | 优点 | 缺点 |
|-----|------|------|
| **destination-out 合成** | ✅ 硬件加速<br>✅ 支持半透明擦除<br>✅ 代码简洁 | ❌ 需要 save/restore |
| 直接设透明像素 | ✅ 原理直观 | ❌ 需要 getImageData 读回内存<br>❌ 无法支持柔边<br>❌ 性能差 |

**橡皮擦额外特性：**
- 圆形模式支持径向渐变柔边（strict=false）
- 高速移动时使用 lineTo 补全间隙（erase.js:193-202）
- 支持矩形/圆形两种笔刷形状

### 4.2 Clone 克隆图章实现

**源点-目标点偏移记录机制：**

```javascript
// clone.js:139-142 右键/长按记录源点
this.clone_coords = {
    x: mouse_x,
    y: mouse_y,
};
```

**每次移动时的采样偏移计算：**

```javascript
// clone.js:299-300 相对偏移 = 点击时的偏差
var x_from = Math.round(this.clone_coords.x - (mouse.click_x - mouse_x));
var y_from = Math.round(this.clone_coords.y - (mouse.click_y - mouse_y));
```

**采样绘制流程：**
1. 创建临时 canvas_source（尺寸 = 笔刷大小）
2. 从原图层/上一图层 drawImage 采样（考虑偏移）
3. 可选：圆形裁剪 + 径向渐变抗锯齿
4. 绘制到目标位置

**多图层支持：**

```javascript
// clone.js:305-311 source_layer 参数支持
if (params.source_layer.value == 'Previous') {
    // 从上个图层采样
    ctx_source.drawImage(previous_layer.link, ...);
} else {
    // 从当前图层采样
    ctx_source.drawImage(canvas_from, ...);
}
```

✅ **支持跨图层克隆**：可以从 "Previous" 上一个图层采样，绘制到当前图层

---

## 五、Gradient 渐变工具

### 5.1 实现方式：原生 Canvas API

**100% 使用 Canvas 原生 API：**

```javascript
// gradient.js:152-162 线性渐变
var grd = ctx.createLinearGradient(layer.x, layer.y, width, height);
grd.addColorStop(0, color1);
grd.addColorStop(1, color2_with_alpha);
ctx.fillStyle = grd;
ctx.fill();

// gradient.js:171-179 径向渐变
var radgrad = ctx.createRadialGradient(
    center_x, center_y, distance * power / 100,  // 内圆半径可控制
    center_x, center_y, distance);
```

| 渐变类型 | 实现方式 | 自定义计算 |
|---------|---------|----------|
| 线性渐变 | createLinearGradient | ❌ 无 |
| 径向渐变 | createRadialGradient | ❌ 无 |

### 5.2 颜色与色标来源

```javascript
// 色标来源：工具参数 params
var color1 = params.color_1;      // 起始色：#HEX 格式
var color2 = params.color_2;      // 终止色：#HEX 格式（转 rgba 应用 alpha）
var alpha = params.alpha / 100 * 255;  // 全局透明度
```

**特点：**
- ✅ 仅支持 **双色** 渐变，不支持多色标
- ✅ 径向渐变支持内圆半径（radial_power 参数）
- ❌ 不支持自定义色标位置，始终是 0 和 1 两端

---

## 六、代码组织一致性评估与重构建议

### 6.1 一致性对比

| 维度 | pencil | brush | erase | fill | clone | gradient | 一致性 |
|-----|--------|-------|-------|------|-------|----------|-------|
| 继承 Base_tools_class | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | **100%** |
| 标准 mousedown/mousemove/mouseup | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | **100%** |
| tmpCanvas 预渲染模式 | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | **50%** |
| render() 函数分离 | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | **50%** |
| 向量图层存储 | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | **50%** |
| State action 历史记录 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | **100%** |

### 6.2 重复代码抽取建议

**可抽取到基类的公共逻辑：**

1. **栅格化工具基类（RasterToolBase）**：
   - 受益工具：erase、fill、clone（都是直接修改像素）
   - 公共代码：
     - tmpCanvas 创建与销毁
     - 图层类型检查（必须是 image 不能是 vector）
     - 旋转图层禁用提示
     - mouseup 时统一 Update_layer_image_action
     - 拖动过程中的 link_canvas 预览机制

2. **笔画工具基类（StrokeToolBase）**：
   - 受益工具：pencil、brush
   - 公共代码：
     - pointer 压感检测与处理（两段几乎完全一样）
     - 图层 params_hash 检测
     - check_dimensions() 边界重计算（90% 相同）
     - 历史记录合并

3. **通用辅助方法：**
   - 高速移动间隙补全（erase 与 pencil 思路类似）
   - 圆形笔刷径向渐变柔边（erase 与 clone 重复）

### 6.3 新增纹理笔刷参考模板

**推荐抄改模板：`brush.js`**

**理由：**

| 特性 | brush.js | erase.js |
|-----|----------|----------|
| 压感支持 | ✅ 完整实现 | ❌ |
| 平滑处理 | ✅ 三级平滑 | ❌ |
| 多指触摸 | ✅ event_links 索引机制 | ❌ |
| 速度感应 | ✅ 模拟粗细变化 | ❌ |
| 向量存储 + 重绘 | ✅ 可二次编辑 | ❌ 像素直接修改 |
| 历史记录合并 | ✅ | ✅ |

**抄改步骤：**
1. 复制 brush.js 结构，保留：
   - pointerdown/pointermove 压感处理
   - event_links 多指机制
   - render_stabilized 平滑算法
2. 修改 render() 函数：
   - 不使用纯色 fillStyle，改为纹理图案 createPattern
   - 沿笔画方向旋转纹理 UV
   - 增加纹理采样参数
3. 在 mousemove_action 中增加纹理采样偏移累积

---

## 总结

**miniPaint 这 6 个工具在架构上呈现明显的代际分层：**

1. **第一代（像素直接修改）**：fill、erase、clone
   - 特点：简单直接，不可二次编辑，适合破坏性操作
   - 代码重复度较高，可抽取公共基类

2. **第二代（向量存储+光栅渲染）**：pencil、brush、gradient
   - 特点：数据与渲染分离，支持 undo/redo 细粒度合并
   - 架构更先进，代码复用性更好

**整体设计评分：**
- ✅ 基类抽象：8/10（事件框架统一很好）
- ✅ 代码复用：5/10（栅格化工具间重复较多）
- ✅ 性能保护：7/10（fill 无递归，eraser 用合成）
- ✅ 扩展性：6/10（新增纹理笔刷需复制大量代码）
