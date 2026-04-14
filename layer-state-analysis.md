# miniPaint 图层系统与状态管理机制分析报告

## 目录

1. [图层数据结构](#1-图层数据结构)
2. [图层操作实现](#2-图层操作实现)
3. [图层属性与 Canvas 重绘](#3-图层属性与-canvas-重绘)
4. [Action 调用链路](#4-action-调用链路)
5. [状态管理方案评估](#5-状态管理方案评估)
6. [总结](#6-总结)

---

## 1. 图层数据结构

### 1.1 图层对象定义

图层数据结构定义在 [base-layers.js](src/js/core/base-layers.js#L22-L44) 文件头部注释中，每个图层是一个包含以下属性的对象：

```javascript
{
    id: int,              // 唯一标识符，自增生成
    link: Image,          // 图像对象引用（type 为 image 时）
    link_canvas: Canvas,  // 可选的 canvas 引用
    parent_id: int,       // 父图层 ID（用于图层分组，当前未完全实现）
    name: string,         // 图层名称
    type: string,         // 图层类型：'image' | 'text' | null 等
    x: int,               // X 坐标位置
    y: int,               // Y 坐标位置
    width: int,           // 当前宽度
    height: int,          // 当前高度
    width_original: int,  // 原始宽度
    height_original: int, // 原始高度
    visible: bool,        // 可见性
    is_vector: bool,      // 是否为矢量图（SVG）
    hide_selection_if_active: bool, // 激活时是否隐藏选区
    opacity: int,         // 透明度 0-100
    order: int,           // 渲染顺序（数值越大越靠前）
    composition: string,  // 混合模式，如 'source-over', 'multiply' 等
    rotate: int,          // 旋转角度 0-359
    data: any,            // 临时数据（如 base64 字符串）
    params: object,       // 工具特定参数
    color: string,        // 颜色值（hex）
    status: string,       // 状态标识
    filters: array,       // 滤镜数组 [{id, name, params}]
    render_function: [string, string] // 自定义渲染函数 [类名, 方法名]
}
```

### 1.2 图层列表存储方式

图层列表存储在全局配置对象 `config.layers` 中：

```javascript
// config.js
config.layers = [];      // 图层数组
config.layer = null;     // 当前选中的图层引用
```

**存储特点：**

- **数组存储**：图层以数组形式存储，通过 `id` 进行索引查找
- **order 属性排序**：渲染时通过 `order` 属性进行排序，而非数组顺序
- **单例引用**：`config.layer` 始终指向当前活动图层

**图层查找方法**（[base-layers.js:506-517](src/js/core/base-layers.js#L506-L517)）：

```javascript
get_layer(id) {
    if (id == undefined) {
        id = config.layer.id;
    }
    for (var i in config.layers) {
        if (config.layers[i].id == id) {
            return config.layers[i];
        }
    }
    return null;
}
```

**排序渲染方法**（[base-layers.js:589-595](src/js/core/base-layers.js#L589-L595)）：

```javascript
get_sorted_layers() {
    return config.layers.concat().sort(
        (a, b) => b.order - a.order
    );
}
```

---

## 2. 图层操作实现

### 2.1 新建图层

**入口文件**：[src/js/modules/layer/new.js](src/js/modules/layer/new.js)

**核心实现**：

```javascript
new() {
    app.State.do_action(
        new app.Actions.Insert_layer_action()
    );
}
```

**Insert_layer_action 执行流程**（[insert-layer.js](src/js/actions/insert-layer.js)）：

1. 创建默认图层对象，设置初始属性
2. 如果是图像类型，处理图像加载
3. 如果首个图层为空，则更新而非新建
4. 将图层推入 `config.layers` 数组
5. 更新 `config.layer` 引用
6. 触发渲染和 GUI 刷新

**关键代码**（[insert-layer.js:53-83](src/js/actions/insert-layer.js#L53-L83)）：

```javascript
const layer = {
    id: app.Layers.auto_increment,
    parent_id: 0,
    name: config.TOOL.name + ' #' + app.Layers.auto_increment,
    type: null,
    // ... 其他默认属性
    opacity: 100,
    order: app.Layers.auto_increment,
    composition: 'source-over',
    filters: [],
};

config.layers.push(layer);
config.layer = app.Layers.get_layer(layer.id);
app.Layers.auto_increment++;
```

### 2.2 删除图层

**入口文件**：[src/js/modules/layer/delete.js](src/js/modules/layer/delete.js)

**Delete_layer_action 执行流程**（[delete-layer.js](src/js/actions/delete-layer.js)）：

1. 查找目标图层索引
2. 如果只剩一个图层且非空，先创建新的空图层
3. 如果删除的是当前选中图层，自动选择相邻图层
4. 从 `config.layers` 数组中移除
5. 触发渲染和 GUI 刷新

**关键代码**（[delete-layer.js:35-56](src/js/actions/delete-layer.js#L35-L56)）：

```javascript
// 如果只剩一个图层
if (config.layers.length == 1 && (force == undefined || force == false)) {
    if (config.layer.type == null) {
        throw new Error('Aborted - Will not delete last layer');
    } else {
        // 先创建新空图层
        this.insert_layer_action = new app.Actions.Insert_layer_action();
        this.insert_layer_action.do();
    }
}

// 如果删除当前图层，选择相邻图层
if (config.layers.length > 1 && config.layer.id == id) {
    const select_action = new app.Actions.Select_next_layer_action(id);
    await select_action.do();
}

// 移除图层
this.deleted_layer = config.layers.splice(this.delete_index, 1)[0];
```

### 2.3 复制图层

**入口文件**：[src/js/modules/layer/duplicate.js](src/js/modules/layer/duplicate.js)

**实现逻辑**：

```javascript
duplicate() {
    var params = JSON.parse(JSON.stringify(config.layer));
    delete params.id;
    delete params.order;

    // 生成新名称
    var name_number = params.name.match(/^(.*) #([0-9]+)$/);
    if(name_number == null){
        params.name = params.name + " #2";
    } else {
        params.name = name_number[1] + " #" + (parseInt(name_number[2]) + 1);
    }

    // 如果图层不在画布中心，偏移位置
    if(params.x != 0 || params.y != 0){
        params.x += 10;
        params.y += 10;
    }

    // 图像类型需要克隆 DOM 元素
    if (params.type == 'image') {
        params.link = config.layer.link.cloneNode(true);
    }

    app.State.do_action(
        new app.Actions.Bundle_action('duplicate_layer', 'Duplicate Layer', [
            new app.Actions.Insert_layer_action(params)
        ])
    );
}
```

### 2.4 合并图层

**入口文件**：[src/js/modules/layer/merge.js](src/js/modules/layer/merge.js)

**实现逻辑**：

```javascript
merge() {
    // 创建临时 canvas
    var canvas = document.createElement('canvas');
    canvas.width = config.WIDTH;
    canvas.height = config.HEIGHT;
    var ctx = canvas.getContext("2d");

    // 获取前一个图层
    var previous_layer = this.Base_layers.find_previous(config.layer.id);
    
    // 先渲染前一个图层
    ctx.globalAlpha = previous_layer.opacity / 100;
    ctx.globalCompositeOperation = previous_layer.composition;
    this.Base_layers.render_object(ctx, previous_layer);

    // 再渲染当前图层
    ctx.globalAlpha = config.layer.opacity / 100;
    ctx.globalCompositeOperation = config.layer.composition;
    this.Base_layers.render_object(ctx, config.layer);

    // 创建合并后的新图层，删除原来的两个图层
    var params = {
        type: 'image',
        name: config.layer.name + ' + merged',
        order: current_order,
        data: canvas.toDataURL("image/png")
    };
    
    app.State.do_action(
        new app.Actions.Bundle_action('merge_layers', 'Merge Layers', [
            new app.Actions.Insert_layer_action(params),
            new app.Actions.Delete_layer_action(current_id),
            new app.Actions.Delete_layer_action(previous_id)
        ])
    );
}
```

### 2.5 调整图层顺序

**入口文件**：[src/js/modules/layer/move.js](src/js/modules/layer/move.js)

**Reorder_layer_action 实现**（[reorder-layer.js](src/js/actions/reorder-layer.js)）：

```javascript
async do() {
    this.reference_layer = app.Layers.get_layer(this.layer_id);
    
    // 根据方向找到目标图层
    if (this.direction < 0) {
        this.reference_target = app.Layers.find_previous(this.layer_id);
    } else {
        this.reference_target = app.Layers.find_next(this.layer_id);
    }
    
    // 交换 order 属性
    this.old_layer_order = this.reference_layer.order;
    this.old_target_order = this.reference_target.order;
    this.reference_layer.order = this.old_target_order;
    this.reference_target.order = this.old_layer_order;

    app.Layers.render();
    app.GUI.GUI_layers.render_layers();
}
```

**注意**：图层顺序调整是通过交换 `order` 属性实现的，而非改变数组位置。

---

## 3. 图层属性与 Canvas 重绘

### 3.1 透明度设置

**入口方法**（[base-layers.js:543-552](src/js/core/base-layers.js#L543-L552)）：

```javascript
async set_opacity(id, value) {
    value = parseInt(value);
    if (value < 0 || value > 100) {
        value = 100;
    }
    return app.State.do_action(
        new app.Actions.Update_layer_action(id, {
            opacity: value,
        })
    );
}
```

**渲染时应用**（[base-layers.js:279](src/js/core/base-layers.js#L279)）：

```javascript
ctx.globalAlpha = layer.opacity / 100;
```

### 3.2 可见性设置

**入口文件**：[src/js/modules/layer/visibility.js](src/js/modules/layer/visibility.js)

**Toggle_layer_visibility_action**（[toggle-layer-visibility.js](src/js/actions/toggle-layer-visibility.js)）：

```javascript
async do() {
    const layer = app.Layers.get_layer(this.layer_id);
    this.old_visible = layer.visible;
    layer.visible = !layer.visible;
    app.Layers.render();
    app.GUI.GUI_layers.render_layers();
}
```

**渲染时检查**（[base-layers.js:316](src/js/core/base-layers.js#L316)）：

```javascript
render_object(ctx, object, is_preview) {
    if (object.visible == false || object.type == null) return;
    // ... 渲染逻辑
}
```

### 3.3 混合模式设置

**入口文件**：[src/js/modules/layer/composition.js](src/js/modules/layer/composition.js)

支持的混合模式包括：`source-over`（默认）、`multiply`、`screen`、`overlay`、`darken`、`lighten`、`difference` 等 28 种。

**设置流程**：

```javascript
composition() {
    var settings = {
        params: [
            {name: "composition", title: "Composition:", value: config.layer.composition, values: compositions}
        ],
        on_change: function (params) {
            // 实时预览
            config.layer.composition = params.composition;
            config.need_render = true;
        },
        on_finish: function (params) {
            // 确认后通过 Action 记录
            app.State.do_action(
                new app.Actions.Update_layer_action(config.layer.id, {
                    composition: params.composition
                })
            );
        }
    };
}
```

**渲染时应用**（[base-layers.js:280](src/js/core/base-layers.js#L280)）：

```javascript
ctx.globalCompositeOperation = layer.composition;
```

### 3.4 Canvas 重绘机制

**核心渲染流程**（[base-layers.js:82-152](src/js/core/base-layers.js#L82-L152)）：

```
用户操作 → 设置 config.need_render = true → requestAnimationFrame 轮询 → render(true)
```

**渲染流程详解**：

```javascript
render(force) {
    if (force !== true) {
        // 仅标记需要渲染
        config.need_render = true;
        return;
    }

    // 1. 预渲染准备
    this.pre_render();
    
    // 2. 获取排序后的图层
    var layers_sorted = this.get_sorted_layers();
    
    // 3. 创建临时 canvas（用于混合模式隔离）
    const newCanvas = this.create_new_canvas(null, config.WIDTH, config.HEIGHT);
    
    // 4. 渲染所有图层
    this.render_objects(this.ctx, newCanvas, layers_sorted, () => {
        this.ctx.save();
    });
    
    // 5. 绘制网格和参考线
    this.Base_gui.draw_grid(this.ctx);
    this.Base_gui.draw_guides(this.ctx);
    
    // 6. 绘制选区控件
    this.Base_selection.draw_selection();
    
    // 7. 渲染工具覆盖层
    this.render_overlay();
    
    // 8. 渲染预览缩略图
    this.render_preview(layers_sorted);
    
    // 9. 重置状态
    this.after_render();
    
    // 10. 继续轮询
    requestAnimationFrame(() => this.render(force));
}
```

**图层渲染核心方法**（[base-layers.js:207-269](src/js/core/base-layers.js#L207-L269)）：

```javascript
render_objects(ctx, tempCanvas, layers, prepare, shouldSkip) {
    const tempCtx = tempCanvas.getContext("2d");
    
    for (var i = layers.length - 1; i >= 0; i--) {
        var layer = layers[i];
        
        // 处理剪贴蒙版（source-atop 混合模式）
        if (layer.composition === 'source-atop' || 
            (nextLayer && nextLayer.composition === 'source-atop')) {
            // 在隔离的临时 canvas 上渲染
            tempCtx.globalAlpha = layer.opacity / 100;
            tempCtx.globalCompositeOperation = layer.composition;
            this.render_object(tempCtx, layer);
            
            // 合并回主 canvas
            ctx.drawImage(tempCanvas, 0, 0);
        } else {
            // 直接在主 canvas 上渲染
            ctx.globalAlpha = layer.opacity / 100;
            ctx.globalCompositeOperation = layer.composition;
            this.render_object(ctx, layer);
        }
    }
}
```

**渲染单个图层对象**（[base-layers.js:316-355](src/js/core/base-layers.js#L316-L355)）：

```javascript
render_object(ctx, object, is_preview) {
    if (object.visible == false || object.type == null) return;

    // 应用前置滤镜
    this.pre_render_object(ctx, object);

    if (object.type == 'image') {
        // 图像类型
        ctx.save();
        ctx.translate(object.x + object.width / 2, object.y + object.height / 2);
        ctx.rotate((object.rotate * Math.PI) / 180);
        ctx.drawImage(
            object.link_canvas != null ? object.link_canvas : object.link,
            -object.width / 2,
            -object.height / 2,
            object.width,
            object.height
        );
        ctx.restore();
    } else {
        // 其他类型，调用自定义渲染函数
        var render_class = object.render_function[0];
        var render_function = object.render_function[1];
        this.Base_gui.GUI_tools.tools_modules[render_class].object[render_function](ctx, object);
    }

    // 应用后置滤镜
    this.after_render_object(ctx, object);
}
```

---

## 4. Action 调用链路

### 4.1 Action 基类

**文件**：[src/js/actions/base.js](src/js/actions/base.js)

```javascript
export class Base_action {
    constructor(action_id, action_description) {
        this.action_id = action_id;
        this.action_description = action_description;
        this.is_done = false;
        this.memory_estimate = 0;
        this.database_estimate = 0;
    }
    
    do() {
        this.is_done = true;
    }
    
    undo() {
        this.is_done = false;
    }
    
    free() {
        // 释放内存
    }
}
```

### 4.2 状态管理器

**文件**：[src/js/core/base-state.js](src/js/core/base-state.js)

**核心属性**：

```javascript
this.action_history = [];        // Action 历史记录
this.action_history_index = 0;   // 当前历史位置
this.action_history_max = 50;    // 最大历史记录数
```

**执行 Action**（[base-state.js:47-98](src/js/core/base-state.js#L47-L98)）：

```javascript
async do_action(action, options = {}) {
    try {
        await action.do();
    } catch (error) {
        // Action 中止
        return { status: 'aborted', reason: error };
    }
    
    // 清除所有 redo 记录
    if (this.action_history_index < this.action_history.length) {
        const freed_actions = this.action_history.slice(this.action_history_index);
        this.action_history = this.action_history.slice(0, this.action_history_index);
        for (let freed_action of freed_actions) {
            await freed_action.free();
        }
    }
    
    // 添加到历史记录
    this.action_history.push(action);
    if (this.action_history.length > this.action_history_max) {
        let action_to_free = this.action_history.shift();
        await action_to_free.free();
    } else {
        this.action_history_index++;
    }
    
    return { status: 'completed' };
}
```

### 4.3 完整调用链路

以「新建图层」为例：

```
用户点击 '+' 按钮
    ↓
GUI_layers.set_events() 捕获点击
    ↓
app.State.do_action(new app.Actions.Insert_layer_action())
    ↓
Base_state_class.do_action()
    ↓
Insert_layer_action.do()
    ├── 创建图层对象
    ├── config.layers.push(layer)
    ├── config.layer = new_layer
    └── app.Layers.render() / app.GUI.GUI_layers.render_layers()
    ↓
action_history.push(action)
    ↓
渲染循环检测到 config.need_render = true
    ↓
Base_layers_class.render(true)
    ↓
Canvas 重绘完成
```

### 4.4 Bundle Action 组合操作

**文件**：[src/js/actions/bundle.js](src/js/actions/bundle.js)

用于将多个 Action 组合成一个原子操作：

```javascript
export class Bundle_action extends Base_action {
    constructor(bundle_id, bundle_name, actions_to_do) {
        super(bundle_id, bundle_name);
        this.actions_to_do = actions_to_do;
    }

    async do() {
        for (let i = 0; i < this.actions_to_do.length; i++) {
            try {
                await this.actions_to_do[i].do();
            } catch (e) {
                // 任一 Action 失败，回滚所有已执行的 Action
                for (i--; i >= 0; i--) {
                    await this.actions_to_do[i].undo();
                }
                throw e;
            }
        }
    }

    async undo() {
        // 逆序撤销
        for (let i = this.actions_to_do.length - 1; i >= 0; i--) {
            await this.actions_to_do[i].undo();
        }
    }
}
```

### 4.5 图层相关 Action 列表

| Action 文件 | 功能 |
|------------|------|
| [insert-layer.js](src/js/actions/insert-layer.js) | 插入新图层 |
| [delete-layer.js](src/js/actions/delete-layer.js) | 删除图层 |
| [update-layer.js](src/js/actions/update-layer.js) | 更新图层属性 |
| [reorder-layer.js](src/js/actions/reorder-layer.js) | 调整图层顺序 |
| [select-layer.js](src/js/actions/select-layer.js) | 选择图层 |
| [toggle-layer-visibility.js](src/js/actions/toggle-layer-visibility.js) | 切换可见性 |
| [reset-layers.js](src/js/actions/reset-layers.js) | 重置所有图层 |
| [clear-layer.js](src/js/actions/clear-layer.js) | 清空图层内容 |
| [update-layer-image.js](src/js/actions/update-layer-image.js) | 更新图层图像 |
| [add-layer-filter.js](src/js/actions/add-layer-filter.js) | 添加滤镜 |
| [delete-layer-filter.js](src/js/actions/delete-layer-filter.js) | 删除滤镜 |

---

## 5. 状态管理方案评估

### 5.1 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                      config.js                          │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │   layers    │  │    layer    │  │  need_render   │  │
│  │   (Array)   │  │  (current)  │  │    (flag)      │  │
│  └─────────────┘  └─────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ↑
                          │ 直接修改
                          │
┌─────────────────────────┴─────────────────────────────┐
│                    Actions Layer                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ InsertLayer  │  │ DeleteLayer  │  │ UpdateLayer │  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
└───────────────────────────────────────────────────────┘
                          ↑
                          │ do_action()
                          │
┌─────────────────────────┴─────────────────────────────┐
│                   Base_state_class                     │
│  ┌──────────────────────────────────────────────────┐  │
│  │              action_history (Array)              │  │
│  │  [action1, action2, action3, ...]                │  │
│  │              action_history_index = 3            │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 5.2 优点

#### 5.2.1 命令模式实现完整

- 每个 Action 封装了 `do()`、`undo()`、`free()` 方法
- 支持细粒度的撤销/重做操作
- Bundle_action 支持原子性复合操作

#### 5.2.2 内存管理考虑周全

- Action 包含 `memory_estimate` 和 `database_estimate`
- 内存压力大时自动释放历史记录
- 使用 IndexedDB 存储大型图像数据（[image-store.js](src/js/actions/store/image-store.js)）

#### 5.2.3 渲染优化

- 使用 `config.need_render` 标志避免频繁重绘
- `requestAnimationFrame` 实现渲染节流
- 临时 canvas 隔离混合模式效果

#### 5.2.4 代码组织清晰

- 模块化设计，职责分离
- Action 与业务逻辑解耦
- 单例模式管理核心类

### 5.3 缺点与风险

#### 5.3.1 状态可变性问题

**问题**：`config` 对象是全局可变的，任何代码都可以直接修改

```javascript
// 危险：直接修改状态，绕过 Action 系统
config.layer.opacity = 50;
config.layers[0].visible = false;
```

**风险**：
- 状态变更不可追踪
- 无法撤销
- 可能导致状态不一致

#### 5.3.2 竞态条件风险

**问题**：异步 Action 执行可能导致竞态

```javascript
// 用户快速连续操作
await app.State.do_action(new Actions.Delete_layer_action(1));
await app.State.do_action(new Actions.Delete_layer_action(2));
// 如果第一个 Action 还未完成，第二个可能操作错误的状态
```

**当前缓解措施**：
- Action 执行失败时会抛出错误中止
- 但缺乏全局锁机制

#### 5.3.3 引用一致性风险

**问题**：`config.layer` 是引用，可能指向已删除的图层

```javascript
// 删除当前图层后，config.layer 可能短暂无效
await app.State.do_action(new Actions.Delete_layer_action(config.layer.id));
// config.layer 在 Action 完成前仍指向旧对象
```

**当前处理**：Delete_layer_action 会自动选择相邻图层

#### 5.3.4 缺乏状态快照

**问题**：无法保存/恢复完整应用状态

- 没有 `serialize()` / `deserialize()` 方法
- 无法实现「保存项目文件」功能（需要额外实现）

#### 5.3.5 调试困难

**问题**：
- 状态变更分散在各处
- 缺乏统一的状态变更日志
- 难以追踪「谁在何时修改了什么」

### 5.4 与 Redux 对比

| 特性 | miniPaint | Redux |
|-----|-----------|-------|
| 状态存储 | 全局可变对象 `config` | 单一不可变 Store |
| 状态变更 | 直接修改 / Action | 仅通过 Reducer |
| 变更追踪 | Action 历史 | 时间旅行调试 |
| 中间件 | 无 | 丰富的中间件生态 |
| 类型安全 | 无（原生 JS） | 可配合 TypeScript |
| 订阅机制 | 手动调用 render() | 自动订阅更新 |
| 撤销/重做 | 内置 Action 系统 | 需额外实现 |

---

## 6. 总结

### 6.1 架构特点

miniPaint 的图层系统采用了**命令模式 + 全局状态**的混合架构：

1. **状态存储**：使用全局 `config` 对象，包含 `layers` 数组和 `layer` 当前引用
2. **操作封装**：通过 Action 类封装所有状态变更，支持撤销/重做
3. **渲染机制**：基于 `requestAnimationFrame` 的轮询渲染，使用 `need_render` 标志节流
4. **历史管理**：维护 Action 历史数组，支持多级撤销

### 6.2 设计亮点

- 完整的撤销/重做系统
- Bundle_action 支持原子性复合操作
- 内存管理机制（自动释放、IndexedDB 存储）
- 混合模式隔离渲染

### 6.3 潜在风险

- 全局状态可被直接修改
- 异步操作可能存在竞态
- 缺乏状态快照和序列化
- 调试和追踪困难

### 6.4 适用场景

该架构适合：
- 中小型单页应用
- 需要撤销/重做功能的编辑器类应用
- 原生 JavaScript 项目

不适合：
- 大型复杂应用（建议使用 Redux/MobX）
- 需要严格状态追踪的生产环境
- 多人协作场景

---

*报告生成时间：2026-04-12*
