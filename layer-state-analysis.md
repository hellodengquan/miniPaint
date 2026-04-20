# miniPaint 图层系统与状态管理分析

## 1. 图层数据结构定义

### 1.1 图层对象属性

根据 [base-layers.js](src/js/core/base-layers.js#L8-L35) 的注释说明，每个图层是一个包含以下属性的对象：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `id` | int | 图层唯一标识符 |
| `parent_id` | int | 父图层ID（用于图层分组） |
| `name` | string | 图层名称 |
| `type` | string | 图层类型（如 'image', 'text' 等） |
| `link` | Image/Canvas | 图像数据引用 |
| `x`, `y` | int | 图层位置坐标 |
| `width`, `height` | int | 图层当前尺寸 |
| `width_original`, `height_original` | int | 图层原始尺寸 |
| `visible` | bool | 是否可见 |
| `is_vector` | bool | 是否为矢量图 |
| `hide_selection_if_active` | bool | 激活时是否隐藏选区 |
| `opacity` | int (0-100) | 不透明度 |
| `order` | int | 图层顺序（用于排序） |
| `composition` | string | 混合模式（如 'source-over'） |
| `rotate` | int (0-359) | 旋转角度 |
| `data` | any | 额外数据存储 |
| `params` | object | 参数对象 |
| `color` | hex | 颜色值 |
| `status` | string | 状态标识 |
| `filters` | array | 滤镜数组 |
| `render_function` | [string, string] | 渲染函数引用 [类名, 方法名] |

### 1.2 图层列表存储方式

图层列表存储在全局配置对象中：

```javascript
// config.js
config.layers = [];  // 所有图层的数组
config.layer = null; // 当前选中的图层
```

图层管理采用 **数组存储 + order 字段排序** 的方式：

```javascript
// base-layers.js
get_sorted_layers() {
    return config.layers.concat().sort(
        (a, b) => b.order - a.order
    );
}
```

- 图层在数组中按插入顺序存储
- 通过 `order` 字段控制显示层级（值越大越在上层）
- 渲染时通过 `get_sorted_layers()` 获取排序后的图层列表

---

## 2. 图层操作实现分析

### 2.1 新建图层

**实现文件**: [src/js/modules/layer/new.js](src/js/modules/layer/new.js)

```javascript
new() {
    app.State.do_action(
        new app.Actions.Insert_layer_action()
    );
}
```

**核心逻辑** ([insert-layer.js](src/js/actions/insert-layer.js)):

1. 创建默认图层对象，包含所有必要属性
2. 如果是图片类型，处理图片加载（支持 data URL 和 Image 对象）
3. 如果是第一个空图层，直接更新现有图层而非创建新图层
4. 将图层推入 `config.layers` 数组
5. 更新 `config.layer` 为当前图层
6. 递增 `auto_increment` 计数器

```javascript
const layer = {
    id: app.Layers.auto_increment,
    parent_id: 0,
    name: 'Layer #' + app.Layers.auto_increment,
    type: null,
    // ... 其他默认属性
};
config.layers.push(layer);
config.layer = app.Layers.get_layer(layer.id);
app.Layers.auto_increment++;
```

### 2.2 删除图层

**实现文件**: [src/js/modules/layer/delete.js](src/js/modules/layer/delete.js)

```javascript
delete() {
    app.State.do_action(
        new app.Actions.Delete_layer_action(config.layer.id)
    );
}
```

**核心逻辑** ([delete-layer.js](src/js/actions/delete-layer.js)):

1. 查找要删除的图层索引
2. 如果只剩一个图层且非强制删除，则创建新空图层后再删除
3. 如果被删除的是当前选中图层，先切换到相邻图层
4. 使用 `Array.splice()` 从数组中移除图层
5. 保存被删除图层引用以便撤销

```javascript
// 删除并保存引用
this.deleted_layer = config.layers.splice(this.delete_index, 1)[0];

// 撤销时恢复
config.layers.splice(this.delete_index, 0, this.deleted_layer);
```

### 2.3 复制图层

**实现文件**: [src/js/modules/layer/duplicate.js](src/js/modules/layer/duplicate.js)

```javascript
duplicate() {
    var params = JSON.parse(JSON.stringify(config.layer));
    delete params.id;
    delete params.order;
    
    // 生成新名称，如 "Layer #2" -> "Layer #3"
    var name_number = params.name.match(/^(.*) #([0-9]+)$/);
    if(name_number == null){
        params.name = params.name + " #2";
    } else {
        params.name = name_number[1] + " #" + (parseInt(name_number[2]) + 1)
    }
    
    // 偏移位置
    params.x += 10;
    params.y += 10;
    
    // 克隆图片数据
    if (params.type == 'image') {
        params.link = config.layer.link.cloneNode(true);
    }
    
    app.State.do_action(
        new app.Actions.Insert_layer_action(params)
    );
}
```

**关键点**:
- 使用 `JSON.parse(JSON.stringify())` 深拷贝图层属性
- 删除 `id` 和 `order` 让新图层获得新值
- 图片数据通过 `cloneNode(true)` 克隆

### 2.4 合并图层

**实现文件**: [src/js/modules/layer/merge.js](src/js/modules/layer/merge.js)

```javascript
merge() {
    // 创建临时 canvas
    var canvas = document.createElement('canvas');
    canvas.width = config.WIDTH;
    canvas.height = config.HEIGHT;
    var ctx = canvas.getContext("2d");

    // 渲染下层图层
    var previous_layer = this.Base_layers.find_previous(config.layer.id);
    ctx.globalAlpha = previous_layer.opacity / 100;
    ctx.globalCompositeOperation = previous_layer.composition;
    this.Base_layers.render_object(ctx, previous_layer);

    // 渲染当前图层
    ctx.globalAlpha = config.layer.opacity / 100;
    ctx.globalCompositeOperation = config.layer.composition;
    this.Base_layers.render_object(ctx, config.layer);

    // 创建合并后的新图层
    var params = [];
    params.type = 'image';
    params.name = config.layer.name + ' + merged';
    params.data = canvas.toDataURL("image/png");
    
    // Bundle Action：插入新图层 + 删除原图层
    app.State.do_action(
        new app.Actions.Bundle_action('merge_layers', 'Merge Layers', [
            new app.Actions.Insert_layer_action(params),
            new app.Actions.Delete_layer_action(current_id),
            new app.Actions.Delete_layer_action(previous_id)
        ])
    );
}
```

**实现特点**:
- 使用 Canvas 2D API 的 `globalCompositeOperation` 实现混合
- 通过 `Bundle_action` 将多个操作打包为原子操作
- 合并后生成新的图片图层

### 2.5 调整图层顺序

**实现文件**: [src/js/modules/layer/move.js](src/js/modules/layer/move.js)

```javascript
up() {
    app.State.do_action(
        new app.Actions.Reorder_layer_action(config.layer.id, 1)
    );
}

down() {
    app.State.do_action(
        new app.Actions.Reorder_layer_action(config.layer.id, -1)
    );
}
```

**核心逻辑** ([reorder-layer.js](src/js/actions/reorder-layer.js)):

```javascript
async do() {
    this.reference_layer = app.Layers.get_layer(this.layer_id);
    // 找到相邻图层
    if (this.direction < 0) {
        this.reference_target = app.Layers.find_previous(this.layer_id);
    } else {
        this.reference_target = app.Layers.find_next(this.layer_id);
    }
    
    // 交换 order 值
    this.old_layer_order = this.reference_layer.order;
    this.old_target_order = this.reference_target.order;
    this.reference_layer.order = this.old_target_order;
    this.reference_target.order = this.old_layer_order;
}
```

**特点**:
- 不移动数组元素，仅交换 `order` 字段值
- 通过 `get_sorted_layers()` 在渲染时重新排序

---

## 3. 图层属性与渲染机制

### 3.1 属性设置位置

| 属性 | 设置位置 | 实现文件 |
|------|----------|----------|
| **透明度 (opacity)** | `Base_layers.set_opacity()` | [base-layers.js](src/js/core/base-layers.js#L537-L550) |
| **可见性 (visible)** | `Toggle_layer_visibility_action` | [toggle-layer-visibility.js](src/js/actions/toggle-layer-visibility.js) |
| **混合模式 (composition)** | 对话框选择 | [composition.js](src/js/modules/layer/composition.js) |
| **名称 (name)** | `Layer_rename_class` | [rename.js](src/js/modules/layer/rename.js) |

### 3.2 透明度设置

```javascript
// base-layers.js
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

### 3.3 可见性切换

```javascript
// toggle-layer-visibility.js
async do() {
    const layer = app.Layers.get_layer(this.layer_id);
    this.old_visible = layer.visible;
    layer.visible = !layer.visible;  // 直接修改状态
    app.Layers.render();
    app.GUI.GUI_layers.render_layers();
}
```

### 3.4 混合模式设置

**实现文件**: [src/js/modules/layer/composition.js](src/js/modules/layer/composition.js)

混合模式通过 Canvas 2D API 的 `globalCompositeOperation` 实现，支持 28 种混合模式：

```javascript
composition() {
    var compositions = [
        "-- Default --",
        "color", "color-burn", "color-dodge", "copy", "darken", "darker",
        "destination-atop", "destination-in", "destination-out", "destination-over",
        "difference", "exclusion", "hard-light", "hue", "lighten", "lighter",
        "luminosity", "multiply", "overlay", "saturation", "screen", "soft-light",
        "source-atop", "source-in", "source-out", "source-over", "xor"
    ];
    
    // 弹出对话框选择混合模式
    var settings = {
        title: 'Composition',
        params: [
            {name: "composition", title: "Composition:", value: config.layer.composition, values: compositions},
        ],
        on_change: function (params, canvas_preview, w, h) {
            // 实时预览：直接修改状态并触发渲染
            config.layer.composition = params.composition;
            config.need_render = true;
        },
        on_finish: function (params) {
            // 确认后通过 Action 正式修改（支持撤销）
            app.State.do_action(
                new app.Actions.Bundle_action('change_composition', 'Change Composition', [
                    new app.Actions.Update_layer_action(config.layer.id, {
                        composition: params.composition
                    })
                ])
            );
        },
        on_cancel: function (params) {
            // 取消时恢复原值
            config.layer.composition = initial_composition;
            config.need_render = true;
        }
    };
}
```

**渲染时的应用** ([base-layers.js](src/js/core/base-layers.js#L250-L260)):

```javascript
render_objects(ctx, tempCanvas, layers, prepare, shouldSkip) {
    for (var i = layers.length - 1; i >= 0; i--) {
        var layer = layers[i];
        
        // 设置透明度和混合模式
        ctx.globalAlpha = layer.opacity / 100;
        ctx.globalCompositeOperation = layer.composition;
        
        this.render_object(ctx, layer);
    }
}
```

**特点**:
- 支持实时预览：对话框中修改时直接作用于 `config.layer.composition`
- 确认后才创建 Action，支持撤销
- 取消时恢复原值
- 使用 `source-atop` 可实现剪贴蒙版效果

### 3.5 Canvas 重新绘制机制

**渲染触发** ([base-layers.js](src/js/core/base-layers.js#L95-L105)):

```javascript
render(force) {
    if (force !== true) {
        // 请求渲染并退出
        config.need_render = true;
        return;
    }
    // ... 实际渲染逻辑
}
```

**渲染流程**:

1. **检查渲染标志**: `config.need_render` 控制是否需要重绘
2. **获取排序后的图层**: `get_sorted_layers()`
3. **遍历渲染每个图层** ([render_objects](src/js/core/base-layers.js#L213-L279)):

```javascript
render_objects(ctx, tempCanvas, layers, prepare, shouldSkip) {
    const tempCtx = tempCanvas.getContext("2d");
    
    for (var i = layers.length - 1; i >= 0; i--) {
        var layer = layers[i];
        
        // 设置透明度和混合模式
        ctx.globalAlpha = layer.opacity / 100;
        ctx.globalCompositeOperation = layer.composition;
        
        // 渲染图层对象
        this.render_object(ctx, layer);
    }
}
```

4. **单个图层渲染** ([render_object](src/js/core/base-layers.js#L297-L340)):

```javascript
render_object(ctx, object, is_preview) {
    if (object.visible == false || object.type == null) return;
    
    this.pre_render_object(ctx, object);  // 应用前置滤镜
    
    if (object.type == "image") {
        // 图片类型：处理旋转和绘制
        ctx.save();
        ctx.translate(object.x + object.width / 2, object.y + object.height / 2);
        ctx.rotate((object.rotate * Math.PI) / 180);
        ctx.drawImage(
            object.link_canvas != null ? object.link_canvas : object.link,
            -object.width / 2, -object.height / 2,
            object.width, object.height
        );
        ctx.restore();
    } else {
        // 其他类型：调用对应的渲染函数
        var render_class = object.render_function[0];
        var render_function = object.render_function[1];
        this.Base_gui.GUI_tools.tools_modules[render_class].object[render_function](ctx, object, is_preview);
    }
    
    this.after_render_object(ctx, object);  // 应用后置滤镜
}
```

5. **渲染循环**: 使用 `requestAnimationFrame` 实现持续渲染

```javascript
requestAnimationFrame(function () {
    _this.render(force);
});
```

### 3.6 属性修改后的更新流程

以更新图层属性为例 ([update-layer.js](src/js/actions/update-layer.js)):

```javascript
async do() {
    this.reference_layer = app.Layers.get_layer(this.layer_id);
    for (let i in this.settings) {
        this.old_settings[i] = this.reference_layer[i];
        this.reference_layer[i] = this.settings[i];  // 直接修改状态
    }
    
    // 标记需要渲染
    if (this.settings.params || this.settings.width || this.settings.height) {
        config.need_render_changed_params = true;
    }
    config.need_render = true;  // 触发重绘
}
```

---

## 4. Action 调用链路分析

### 4.1 整体架构

```
用户操作 → Module 方法 → Action 类 → Base_state.do_action() → 状态变更 → 渲染更新
```

### 4.2 调用链路示例

以**新建图层**为例：

```
1. 用户点击 "+" 按钮
   ↓
2. GUI_layers.set_events() 监听点击
   ↓
3. 调用 app.State.do_action(new app.Actions.Insert_layer_action())
   ↓
4. Base_state.do_action() 执行：
   - 调用 action.do() 执行实际操作
   - 将 action 加入 action_history
   - 更新 action_history_index
   ↓
5. Insert_layer_action.do()：
   - 创建图层对象
   - config.layers.push(layer)
   - config.layer = layer
   - app.Layers.render()
   - app.GUI.GUI_layers.render_layers()
```

### 4.3 Action 基类

```javascript
// base.js
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

### 4.4 Bundle Action（复合操作）

用于将多个操作打包为原子操作：

```javascript
// bundle.js
export class Bundle_action extends Base_action {
    constructor(bundle_id, bundle_name, actions_to_do) {
        super(bundle_id, bundle_name);
        this.actions_to_do = actions_to_do;
    }

    async do() {
        super.do();
        for (let action of this.actions_to_do) {
            await action.do();
        }
        config.need_render = true;
    }

    async undo() {
        super.undo();
        // 逆序撤销
        for (let i = this.actions_to_do.length - 1; i >= 0; i--) {
            await this.actions_to_do[i].undo();
        }
        config.need_render = true;
    }
}
```

---

## 5. 状态管理方案评估

### 5.1 架构特点

miniPaint 采用 **命令模式 (Command Pattern)** 结合 **全局状态** 的管理方案：

1. **全局状态存储**: 使用 `config` 对象作为单一数据源
2. **命令模式**: 每个操作封装为 Action 类，包含 `do()` 和 `undo()` 方法
3. **历史记录**: 数组存储操作历史，支持多级撤销/重做
4. **直接修改**: Action 直接修改 `config` 中的状态

### 5.2 优点

| 优点 | 说明 |
|------|------|
| **Undo/Redo 支持完善** | 命令模式天然支持撤销重做，实现清晰 |
| **内存管理** | 通过 `free()` 方法主动释放内存，控制历史记录数量 |
| **复合操作** | `Bundle_action` 支持将多个操作打包为原子操作 |
| **简单直观** | 直接修改状态，无需额外的状态转换层 |
| **渲染优化** | `config.need_render` 标志避免不必要的重绘 |

### 5.3 缺点与风险

| 缺点 | 说明 |
|------|------|
| **全局状态污染** | 所有状态集中在 `config` 对象，缺乏命名空间隔离 |
| **直接修改状态** | Action 直接修改状态对象，缺乏不可变性保证 |
| **无状态快照** | 撤销通过反向操作实现，而非状态快照 |
| **竞态风险** | 异步操作（如图片加载）可能导致状态不一致 |
| **调试困难** | 缺乏状态变更追踪机制 |

### 5.4 潜在问题分析

#### 5.4.1 竞态问题

在 [insert-layer.js](src/js/actions/insert-layer.js#L72-L95) 中，图片加载是异步的：

```javascript
image_load_promise = new Promise((resolve, reject) => {
    layer.link = new Image();
    layer.link.onload = () => {
        // 异步回调中修改状态
        config.need_render = true;
        resolve();
    };
    layer.link.src = layer.data;
});
```

**风险**: 如果在图片加载过程中执行撤销操作，可能导致状态不一致。

#### 5.4.2 状态引用问题

Action 中直接保存状态引用：

```javascript
// update-layer.js
async do() {
    this.reference_layer = app.Layers.get_layer(this.layer_id);
    this.old_settings[i] = this.reference_layer[i];  // 引用赋值
}
```

如果 `old_settings` 中保存的是对象引用，修改图层属性时可能导致 `old_settings` 也被修改。

#### 5.4.3 渲染时序问题

```javascript
// 多个地方直接调用 render()
app.Layers.render();
app.GUI.GUI_layers.render_layers();
```

如果 Action 执行过程中出错，可能导致 GUI 和 Canvas 状态不一致。

### 5.5 与现代状态管理对比

| 特性 | miniPaint | Redux/MobX 等现代方案 |
|------|-----------|----------------------|
| **数据流** | 双向/直接修改 | 单向数据流 |
| **不可变性** | ❌ 直接修改 | ✅ 状态不可变 |
| **状态追踪** | ❌ 无 | ✅ DevTools 支持 |
| **选择器** | ❌ 手动查找 | ✅ 派生状态 |
| **中间件** | ❌ 无 | ✅ Redux Middleware |
| **性能优化** | 手动控制渲染 | 自动依赖追踪 |
| **学习成本** | 低 | 较高 |

### 5.6 改进建议

1. **引入不可变性**: 使用 immer 等库保证状态不可变
2. **状态归一化**: 使用 `{ byId, allIds }` 结构存储图层
3. **选择器模式**: 封装 `getLayerById`, `getVisibleLayers` 等选择器
4. **异步 Action 管理**: 统一处理异步操作的取消和竞态
5. **状态订阅**: 使用发布订阅模式替代直接调用渲染

---

## 6. 核心文件索引

| 文件路径 | 职责 |
|----------|------|
| [src/js/core/base-layers.js](src/js/core/base-layers.js) | 图层管理核心类，渲染逻辑 |
| [src/js/core/base-state.js](src/js/core/base-state.js) | 状态管理，Undo/Redo 实现 |
| [src/js/config.js](src/js/config.js) | 全局配置对象 |
| [src/js/actions/base.js](src/js/actions/base.js) | Action 基类 |
| [src/js/actions/bundle.js](src/js/actions/bundle.js) | 复合操作实现 |
| [src/js/actions/insert-layer.js](src/js/actions/insert-layer.js) | 新建图层 Action |
| [src/js/actions/delete-layer.js](src/js/actions/delete-layer.js) | 删除图层 Action |
| [src/js/actions/update-layer.js](src/js/actions/update-layer.js) | 更新图层 Action |
| [src/js/actions/reorder-layer.js](src/js/actions/reorder-layer.js) | 调整顺序 Action |
| [src/js/modules/layer/new.js](src/js/modules/layer/new.js) | 新建图层模块 |
| [src/js/modules/layer/duplicate.js](src/js/modules/layer/duplicate.js) | 复制图层模块 |
| [src/js/modules/layer/merge.js](src/js/modules/layer/merge.js) | 合并图层模块 |
| [src/js/modules/layer/move.js](src/js/modules/layer/move.js) | 移动图层模块 |
| [src/js/core/gui/gui-layers.js](src/js/core/gui/gui-layers.js) | 图层面板 GUI |

---

## 7. 总结

miniPaint 的图层系统采用 **命令模式** 管理状态变更，通过全局 `config` 对象存储图层数据，使用 `order` 字段控制图层层级顺序。状态变更通过 Action 类封装，支持完善的 Undo/Redo 功能。

这套方案的优点是实现简单、Undo/Redo 支持完善；缺点是缺乏不可变性保证、存在潜在的竞态风险。与现代 Redux 等方案相比，它缺少单向数据流、状态不可变、选择器等现代特性，但对于一个原生 JavaScript 的图像编辑器来说，这套方案足够实用且易于维护。
