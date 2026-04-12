# Code Review: 可编辑滤镜 (Commit 433d702)

## 概述

**Commit Hash:** 433d702b0b45a74a946c8b08a97f02854a682bf4  
**Author:** viliusle  
**Date:** Sun Jun 13 22:00:34 2021 +0300  
**Message:** editable filters (only if non-destructive)  
**Files Changed:** 13 files (+114, -39)

---

## 1. 核心思路分析

### 1.1 破坏性 vs 非破坏性滤镜的本质区别

**破坏性滤镜（改动前）：**
- 滤镜参数在应用时直接修改图像数据
- 参数一旦确定就无法回溯修改
- 每次调整都需要重新应用滤镜，原图数据可能丢失

**非破坏性滤镜（改动后）：**
- 滤镜参数存储在图层对象的 `filters` 数组中
- 滤镜效果通过 CSS filter 属性实时渲染
- 支持通过 `filter_id` 定位并修改已有滤镜参数
- 原图数据保持不变，滤镜参数可随时调整

### 1.2 改动的核心机制

这次改动引入了 **filter_id** 作为滤镜的唯一标识符，实现了以下核心能力：

1. **滤镜定位**：通过 `filter_id` 在图层的 `filters` 数组中找到目标滤镜
2. **参数回填**：编辑已有滤镜时，从存储的参数中恢复默认值
3. **更新而非新增**：Action 层判断 `filter_id` 存在时执行更新操作，否则执行新增操作
4. **预览隔离**：编辑滤镜时临时禁用当前滤镜效果，避免预览叠加

---

## 2. 抽象层改动分析 (css.js)

### 2.1 改动内容

[css.js](src/js/modules/effects/abstract/css.js) 作为滤镜的抽象基类，做了以下关键改动：

#### 新增方法：`find_filter_by_id(filter_id, filter_name)`

```javascript
find_filter_by_id(filter_id, filter_name) {
    var filter = {};
    for(var i in config.layer.filters){
        if(config.layer.filters[i].name == filter_name && config.layer.filters[i].id == filter_id) {
            return config.layer.filters[i].params;
        }
    }
    return {};
}
```

**作用**：根据 `filter_id` 和 `filter_name` 从当前图层的滤镜列表中查找已存储的参数。

#### 修改方法：`show_dialog(type, params, filter_id)`

- 增加 `filter_id` 参数
- 在显示对话框前后调用 `Base_layers.disable_filter()` 实现预览隔离

```javascript
this.Base_layers.disable_filter(filter_id);
this.POP.show(settings);
this.Base_layers.disable_filter(null);
```

#### 修改方法：`save(params, type, filter_id)`

- 将 `filter_id` 传递给 `Add_layer_filter_action`，由 Action 层决定是新增还是更新

#### 修改方法：`preview(params, type)`

- 修复了 `shadow` 滤镜预览时的名称映射问题（`shadow` → `drop-shadow`）

### 2.2 设计评价

**优点：**
- 抽象层正确承担了"参数查找"和"预览隔离"的通用逻辑
- 通过 `??=` 运算符优雅处理了默认值和已有值的合并

**潜在问题：**
- `find_filter_by_id` 方法名与其返回值（params 对象）不完全匹配，建议命名为 `find_filter_params_by_id`

---

## 3. 具体滤镜改动模式分析

### 3.1 改动模式总结

所有滤镜文件都遵循统一的改动模式：

| 改动项 | 改动前 | 改动后 |
|--------|--------|--------|
| 方法签名 | `effectName()` | `effectName(filter_id)` |
| 参数查找 | 无 | `var filter = this.find_filter_by_id(filter_id, 'effect-name')` |
| 默认值 | `value: 固定值` | `value: filter.value ??= 固定值` |
| 对话框调用 | `this.show_dialog('name', params)` | `this.show_dialog('name', params, filter_id)` |

### 3.2 各滤镜改动详情

#### blur.js
```javascript
// 改动前
blur() {
    var params = [{name: "value", title: "Percentage:", value: 5, range: [0, 50]}];
    this.show_dialog('blur', params);
}

// 改动后
blur(filter_id) {
    var filter = this.find_filter_by_id(filter_id, 'blur');
    var params = [{name: "value", title: "Percentage:", value: filter.value ??= 5, range: [0, 50]}];
    this.show_dialog('blur', params, filter_id);
}
```

#### brightness.js / contrast.js / grayscale.js
改动模式与 blur.js 完全一致，仅默认值不同：
- brightness: 默认 50
- contrast: 默认 40
- grayscale: 默认 100

### 3.3 shadow.js 的特殊处理

**shadow.js 是唯一有额外改动的滤镜文件：**

```javascript
// 改动前
this.show_dialog('drop-shadow', params);

// 改动后
this.show_dialog('shadow', params, filter_id);
```

**差异原因：**
- CSS filter 属性名为 `drop-shadow`，但内部存储名称为 `shadow`
- 改动前直接使用 CSS 名称，改动后统一使用内部名称
- 抽象层 `preview()` 方法中新增了名称映射逻辑来兼容

**其他多参数处理：**
shadow 有 4 个参数（x, y, value, color），改动模式与其他滤镜一致，只是参数更多：

```javascript
{name: "x", title: "Offset X:", value: filter.x ??= 10, range: [-100, 100]},
{name: "y", title: "Offset Y:", value: filter.y ??= 10, range: [-100, 100]},
{name: "value", title: "Radius:", value: filter.value ??= 5, range: [0, 100]},
{name: "color", title: "Color:", value: filter.color ??= "#000000", type: 'color'},
```

### 3.4 一致性评估

| 滤镜 | 改动一致性 | 备注 |
|------|-----------|------|
| blur | ✅ 完全一致 | - |
| brightness | ✅ 完全一致 | - |
| contrast | ✅ 完全一致 | - |
| grayscale | ✅ 完全一致 | - |
| shadow | ⚠️ 有额外改动 | 对话框名称从 `drop-shadow` 改为 `shadow` |

---

## 4. 图层相关改动分析

### 4.1 gui-layers.js 改动

#### 新增滤镜编辑入口

```javascript
else if (target.id == 'filter_name') {
    //edit filter
    var effects = _this.Effects_browser.get_effects_list();
    var key = target.dataset.filter.toLowerCase();
    for (var i in effects) {
        if(effects[i].title.toLowerCase() == key){
            _this.Base_layers.select(target.dataset.pid);
            var function_name = _this.Effects_browser.get_function_from_path(key);
            effects[i].object[function_name](target.dataset.id);
        }
    }
}
```

**触发方式：** 点击滤镜名称（`#filter_name`）进入编辑模式

**处理流程：**
1. 从 `dataset.filter` 获取滤镜名称
2. 在效果列表中匹配对应的滤镜对象
3. 选中所属图层
4. 调用滤镜方法，传入 `filter_id`（来自 `dataset.id`）

#### HTML 模板改动

```javascript
// 改动前
html += '<span class="layer_name" id="filter_name" data-pid="' + layers[i].id + '" data-id="' + filter.id + '">' + title + '</span>';

// 改动后
html += '<span class="layer_name" id="filter_name" data-pid="' + layers[i].id + '" data-id="' + filter.id + '" data-filter="' + filter.name + '">' + title + '</span>';
```

新增 `data-filter` 属性存储滤镜名称，用于编辑时定位滤镜类型。

### 4.2 base-layers.js 改动

#### 新增属性：`disabled_filter_id`

```javascript
this.disabled_filter_id = null;
```

用于存储当前需要禁用的滤镜 ID。

#### 新增方法：`disable_filter(filter_id)`

```javascript
disable_filter(filter_id) {
    this.disabled_filter_id = filter_id;
}
```

#### 渲染时跳过禁用的滤镜

```javascript
for (var i in object.filters) {
    var filter = object.filters[i];
    if(filter.id == this.disabled_filter_id){
        continue;
    }
    // ... 应用滤镜
}
```

**设计目的：** 编辑滤镜时，预览对话框会实时显示滤镜效果。如果不禁用原图层上的滤镜，预览效果会叠加（原图层滤镜 + 预览滤镜），导致预览不准确。

### 4.3 滤镜参数存储位置

滤镜参数存储在 **图层对象** 的 `filters` 数组中，每个滤镜对象结构如下：

```javascript
{
    id: number,        // 滤镜唯一标识符
    name: string,      // 滤镜类型名称（如 'blur', 'shadow'）
    params: object     // 滤镜参数（如 {value: 5} 或 {x: 10, y: 10, value: 5, color: '#000000'}）
}
```

---

## 5. 模块边界与跨层依赖评估

### 5.1 模块结构

```
src/js/
├── actions/           # Action 层 - 操作封装与撤销/重做
│   └── add-layer-filter.js
├── core/              # Core 层 - 核心业务逻辑
│   ├── base-layers.js
│   └── gui/
│       └── gui-layers.js
└── modules/           # Modules 层 - 功能模块
    └── effects/
        ├── abstract/
        │   └── css.js
        └── common/
            ├── blur.js
            ├── brightness.js
            └── ...
```

### 5.2 依赖关系分析

```
┌─────────────────────────────────────────────────────────────┐
│                      GUI Layer (gui-layers.js)              │
│  - 引用 Effects_browser_class (modules/effects/browser.js) │
│  - 引用 Base_layers_class (core/base-layers.js)            │
└──────────────────────────┬──────────────────────────────────┘
                           │ 调用
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   Effects Module (css.js + filters)         │
│  - 引用 Base_layers_class (core/base-layers.js)            │
│  - 引用 config (全局配置)                                   │
└──────────────────────────┬──────────────────────────────────┘
                           │ 调用
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   Action Layer (add-layer-filter.js)        │
│  - 引用 config (全局配置)                                   │
│  - 引用 app.GUI (全局 GUI 引用)                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 边界清晰度评估

| 评估项 | 评分 | 说明 |
|--------|------|------|
| 职责分离 | ⭐⭐⭐⭐ | Action 负责状态变更，Effects 负责参数处理，GUI 负责交互 |
| 依赖方向 | ⭐⭐⭐⭐ | 依赖方向基本合理，上层依赖下层 |
| 跨层调用 | ⭐⭐⭐ | 存在少量直接依赖全局对象的情况 |

### 5.4 潜在问题

#### 问题 1：全局对象依赖

```javascript
// css.js 中直接访问全局 config
for(var i in config.layer.filters){...}

// add-layer-filter.js 中直接访问全局 app.GUI
app.GUI.GUI_layers.render_layers();
```

**风险：** 增加了测试难度，降低了模块可复用性。

**建议：** 考虑通过依赖注入或构造函数参数传入这些依赖。

#### 问题 2：GUI 层直接操作 Effects

```javascript
// gui-layers.js
var effects = _this.Effects_browser.get_effects_list();
var function_name = _this.Effects_browser.get_function_from_path(key);
effects[i].object[function_name](target.dataset.id);
```

**风险：** GUI 层需要了解 Effects 模块的内部结构。

**建议：** 可以在 Effects 模块中提供一个统一的 `edit_filter(filter_id, filter_name)` 方法。

#### 问题 3：Base_layers 的 disable_filter 状态管理

```javascript
// css.js
this.Base_layers.disable_filter(filter_id);
this.POP.show(settings);
this.Base_layers.disable_filter(null);
```

**风险：** 这是一个临时状态修改，如果 `POP.show()` 抛出异常，状态可能无法恢复。

**建议：** 使用 try-finally 或封装为更高层的抽象。

### 5.5 总体评价

模块边界整体清晰，改动遵循了现有架构风格。跨层依赖虽然存在，但都在合理范围内，没有引入循环依赖或严重的架构违规。

---

## 6. 总结

### 6.1 改动亮点

1. **最小改动原则**：改动集中在必要的地方，没有过度设计
2. **向后兼容**：新增的 `filter_id` 参数可选，不影响现有调用
3. **统一模式**：所有滤镜文件遵循相同的改动模式，易于维护
4. **预览隔离**：巧妙地通过 `disable_filter` 解决了预览叠加问题

### 6.2 改进建议

| 优先级 | 建议 |
|--------|------|
| 中 | 将 `find_filter_by_id` 重命名为 `find_filter_params_by_id` |
| 低 | 考虑使用 try-finally 保护 `disable_filter` 状态恢复 |
| 低 | 减少对全局 `config` 和 `app` 对象的直接依赖 |

### 6.3 风险评估

- **回归风险**：低。改动模式一致，逻辑清晰
- **性能影响**：无。滤镜参数查找是 O(n) 操作，滤镜数量通常很少
- **兼容性**：良好。使用 `??=` 运算符需要现代浏览器支持

---

*Review Date: 2026-04-12*
