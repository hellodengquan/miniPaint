# Code Review: Bezier Curve Tool (Commit 3df8377)

## 1. 改动概述

### 1.1 涉及文件

| 文件 | 变更类型 | 行数变化 | 说明 |
|------|----------|----------|------|
| `src/js/tools/shapes/bezier_curve.js` | 新增 | +412 | 贝塞尔曲线工具核心实现 |
| `src/js/libs/helpers.js` | 修改 | +29 | 新增 `draw_control_point` 辅助函数 |
| `src/js/config.js` | 修改 | +8 | 注册新工具、添加 `mouse_lock` 配置 |
| `src/js/tools/select.js` | 修改 | +24 | 支持工具自定义选择渲染、添加 mouse_lock 检查 |
| `src/js/core/base-tools.js` | 修改 | +4 | 代码格式化（括号风格统一） |
| `images/test-collection.json` | 修改 | +48 | 添加测试数据 |

### 1.2 整体目的

本次提交实现了一个**三次贝塞尔曲线（Cubic Bezier Curve）**绘图工具，支持：
- 通过两次拖拽交互完成曲线绘制（先设起点+CP1，再设终点+CP2）
- 使用选择工具时，可拖拽 4 个控制点（起点、终点、两个控制点）进行编辑
- 支持吸附（snap）、正交约束（Ctrl/Cmd 键）
- 触摸设备支持

---

## 2. 核心算法分析 (`bezier_curve.js`)

### 2.1 数据结构

```javascript
{
  start: {x, y},  // 曲线起点
  cp1: {x, y},    // 第一个控制点
  cp2: {x, y},    // 第二个控制点
  end: {x, y}     // 曲线终点
}
```

使用 `{x: null, y: null}` 表示未设置的点，通过判断 `end.x === null` 区分绘制阶段。

### 2.2 绘制流程（状态机）

```
第一次 mousedown  → 创建 draft 图层，设置 start
第一次 mousemove  → 实时更新 cp1（第一个控制点跟随鼠标）
第一次 mouseup    → 确定 cp1

第二次 mousedown  → 设置 end（终点）
第二次 mousemove  → 实时更新 cp2（第二个控制点跟随鼠标）
第二次 mouseup    → 确定 cp2，status 设为 null（完成）
```

### 2.3 曲线绘制

使用 Canvas 原生 API `ctx.bezierCurveTo(cp1x, cp1y, cp2x, cp2y, x, y)`：

```javascript
ctx.beginPath();
ctx.moveTo(x + data.start.x, y + data.start.y);
ctx.bezierCurveTo(
  x + data.cp1.x, y + data.cp1.y,
  x + data.cp2.x, y + data.cp2.y,
  x + data.end.x, y + data.end.y
);
ctx.stroke();
```

### 2.4 控制点交互逻辑

**命中检测**：使用 `Path2D` + `ctx.isPointInPath()` 进行精确点击检测

```javascript
// 渲染时保存 Path2D 对象
this.selected_obj_positions.cp1_start = this.Helper.draw_control_point(...);

// 点击检测
if (this.ctx.isPointInPath(position, mouse.x, mouse.y)) {
  // 命中控制点
}
```

**拖拽更新**：计算鼠标位移 delta，直接修改对应控制点坐标

```javascript
var dx = Math.round(mouse.x - mouse.click_x) - config.layer.x;
var dy = Math.round(mouse.y - mouse.click_y) - config.layer.y;
bezier.start.x = mouse.click_x + dx;
bezier.start.y = mouse.click_y + dy;
```

### 2.5 正交约束（Ctrl/Cmd 键）

在移动控制点时，通过比较宽高绝对值决定锁定方向：

```javascript
if (e.ctrlKey == true || e.metaKey) {
  var width = mouse_x - bezier.start.x;
  var height = mouse_y - bezier.start.y;
  if (Math.abs(width) > Math.abs(height))
    bezier.cp1.y = bezier.start.y;  // 锁定 Y
  else
    bezier.cp1.x = bezier.start.x;  // 锁定 X
}
```

---

## 3. 系统集成分析

### 3.1 工具注册 (`config.js`)

```javascript
{
  name: 'bezier_curve',
  visible: false,  // 不在工具栏显示
  attributes: {
    size: 4,       // 线宽
  },
}
```

**评价**：遵循现有工具配置格式，通过 `visible: false` 隐藏（可能通过其他菜单访问）。

### 3.2 鼠标锁定机制 (`config.js` + `select.js`)

新增 `config.mouse_lock` 用于协调多工具间的鼠标事件：

- **问题**：贝塞尔工具需要在选择模式下也能响应控制点拖拽
- **解决方案**：
  1. 贝塞尔工具在 `selected_object_actions` 中检查 `config.TOOL.name != 'select'`
  2. 当拖拽控制点时设置 `config.mouse_lock = true`
  3. `select.js` 在 mousedown/mousemove/mouseup 中检查 `config.mouse_lock === true` 并跳过

**评价**：这是一种合理的协作机制，但增加了全局状态的复杂度。

### 3.3 选择工具扩展 (`select.js`)

新增钩子允许工具自定义选择状态下的渲染：

```javascript
if(config.layer.render_function != null) {
  var render_class = config.layer.render_function[0];
  var render_function = 'select';
  if (typeof this.Base_gui.GUI_tools.tools_modules[render_class].object[render_function] != "undefined") {
    this.Base_gui.GUI_tools.tools_modules[render_class].object[render_function](this.ctx);
  }
}
```

**评价**：良好的扩展点设计，允许矢量工具在选择模式下显示自定义控制点。

---

## 4. 辅助函数评估 (`helpers.js`)

### 4.1 `draw_control_point`

```javascript
draw_control_point(ctx, x, y) {
  var dx = 0;  // 未使用的变量
  var dy = 0;  // 未使用的变量
  var block_size = 12 / config.ZOOM;
  const wholeLineWidth = 2 / config.ZOOM;

  ctx.strokeStyle = "#000000";
  ctx.fillStyle = "#ffffff";
  ctx.lineWidth = wholeLineWidth;

  const circle = new Path2D();
  circle.arc(x + dx * block_size, y + dy * block_size, block_size / 2, 0, 2 * Math.PI);

  ctx.fill(circle);
  ctx.stroke(circle);

  return circle;  // 返回 Path2D 用于点击检测
}
```

**评价**：
- ✅ 命名清晰，功能明确
- ✅ 考虑缩放（`config.ZOOM`）
- ✅ 黑白配色确保在任何背景下可见
- ⚠️ `dx`、`dy` 是死代码（始终为 0）
- ⚠️ 魔法数字 `12`、`2` 建议提取为常量

---

## 5. 代码质量评估

### 5.1 优点

| 方面 | 说明 |
|------|------|
| 架构一致性 | 继承 `Base_tools_class`，遵循现有工具模式 |
| 撤销/重做支持 | 正确使用 `app.State.do_action` 和 `Bundle_action` |
| 矢量图形支持 | 设置 `is_vector: true`，支持无限缩放 |
| 交互完整性 | 支持鼠标、触摸、吸附、正交约束 |
| 状态管理 | 使用 `status: 'draft'` 区分绘制中和已完成状态 |

### 5.2 问题与改进建议

#### 5.2.1 魔法数字

```javascript
// 当前代码
var block_size = 12 / config.ZOOM;
const wholeLineWidth = 2 / config.ZOOM;

// 建议
const CONTROL_POINT_SIZE = 12;
const CONTROL_POINT_BORDER_WIDTH = 2;
```

#### 5.2.2 重复代码

`mousedown`/`mousemove`/`mouseup` 中有大量重复的坐标计算逻辑，建议提取为共用函数：

```javascript
// 建议提取
_getMousePositionWithSnap(e, mouse, layerId) {
  var mouse_x = Math.round(mouse.x);
  var mouse_y = Math.round(mouse.y);
  
  var snap_info = this.calc_snap_position(e, mouse_x, mouse_y, layerId);
  if(snap_info != null) {
    if(snap_info.x != null) mouse_x = snap_info.x;
    if(snap_info.y != null) mouse_y = snap_info.y;
  }
  return {x: mouse_x, y: mouse_y};
}
```

#### 5.2.3 状态检查逻辑复杂

```javascript
// 当前：多层嵌套条件
if (config.layer.type != this.name || params_hash != this.params_hash
  || (data_clone != null && data_clone.cp2.x !== null)) {
  // ...
}

// 建议：提取为有意义的变量
const isDifferentTool = config.layer.type != this.name;
const paramsChanged = params_hash != this.params_hash;
const isComplete = data_clone != null && data_clone.cp2.x !== null;

if (isDifferentTool || paramsChanged || isComplete) {
  // ...
}
```

#### 5.2.4 缺少边界检查

- 未处理所有控制点重合的情况（会导致曲线不可见）
- 未限制控制点数量（理论上可以无限创建新曲线）

#### 5.2.5 事件监听作用域

```javascript
// 当前：监听整个 document
document.addEventListener('mousedown', (e) => {
  this.selected_object_actions(e);
});

// 潜在问题：即使工具未激活也会触发
// 建议：在工具切换时动态添加/移除，或在回调中尽早 return
```

#### 5.2.6 代码风格不一致

与现有工具对比发现：
- `line.js` 使用 `this.mouse_click` 保存初始位置
- `bezier_curve.js` 直接使用 `mouse.click_x`（来自 `get_mouse_info`）

两种风格都可以，但建议保持一致。

#### 5.2.7 缺少注释

核心算法（如控制点拖拽的坐标计算）缺少注释说明计算逻辑：

```javascript
// 建议添加注释
// Calculate delta from initial click position, accounting for layer offset
var dx = Math.round(mouse.x - mouse.click_x) - config.layer.x;
var dy = Math.round(mouse.y - mouse.click_y) - config.layer.y;
```

---

## 6. 总结

### 6.1 整体评价

本次提交实现了一个功能完整的贝塞尔曲线工具，**架构设计合理**，与现有系统**集成良好**，**代码质量中等偏上**。

### 6.2 风险等级

| 类别 | 等级 | 说明 |
|------|------|------|
| 功能风险 | 低 | 功能完整，边界情况处理尚可 |
| 维护风险 | 中 | 部分重复代码和魔法数字 |
| 性能风险 | 低 | 无复杂计算，渲染效率高 |

### 6.3 建议优先级

1. **高优先级**：移除 `draw_control_point` 中的死代码（`dx`, `dy`）
2. **中优先级**：提取魔法数字为常量
3. **中优先级**：简化 `mousedown` 中的复杂条件判断
4. **低优先级**：提取重复的坐标计算逻辑为共用函数
5. **低优先级**：补充核心算法的注释
