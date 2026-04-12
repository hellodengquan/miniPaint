# miniPaint 可编辑滤镜功能 Code Review

## 提交信息
- **Commit Hash**: `433d702`
- **提交者**: viliusle
- **日期**: 2021-06-13
- **提交信息**: editable filters (only if non-destructive)

## 1. 改动概览与核心思路

### 1.1 涉及文件（共 13 个）

| 模块 | 文件路径 | 改动类型 |
|------|----------|----------|
| actions | `src/js/actions/add-layer-filter.js` | 新增/修改 |
| core | `src/js/core/base-layers.js` | 新增方法 |
| core/gui | `src/js/core/gui/gui-layers.js` | 新增交互 |
| effects/abstract | `src/js/modules/effects/abstract/css.js` | 核心抽象层修改 |
| effects/common | `blur.js`, `brightness.js`, `contrast.js`, `grayscale.js` | 具体滤镜适配 |
| effects/common | `hue-rotate.js`, `invert.js`, `saturate.js`, `sepia.js`, `shadow.js` | 具体滤镜适配 |

### 1.2 核心思路

**破坏性滤镜（Destructive）vs 非破坏性滤镜（Non-destructive）**

| 特性 | 破坏性滤镜（改动前） | 非破坏性滤镜（改动后） |
|------|---------------------|----------------------|
| 参数存储 | 不存储，直接应用到像素 | 存储在图层对象的 `filters` 数组中 |
| 可编辑性 | 不可编辑，应用后无法修改 | 可随时双击编辑参数 |
| 实现方式 | 直接修改 canvas 像素 | 通过 CSS filter 在渲染时动态应用 |
| 撤销支持 | 依赖像素级撤销 | 通过 Action 系统管理滤镜的增删改 |

**实现原理**：
1. 每个滤镜的参数以对象形式存储在图层的 `filters` 数组中
2. 渲染时通过 `pre_render_object` 和 `after_render_object` 方法动态应用滤镜
3. 用户双击图层面板中的滤镜名称时，重新打开参数对话框并回填现有参数

---

## 2. 抽象层改动分析：`src/js/modules/effects/abstract/css.js`

### 2.1 `show_dialog` 方法的变化

```javascript
// 改动前
show_dialog(type, params) {
    // ...
    on_finish: function (params) {
        _this.save(params, type);  // 无 filter_id
    },
}

// 改动后
show_dialog(type, params, filter_id) {  // 新增 filter_id 参数
    // ...
    on_finish: function (params) {
        _this.save(params, type, filter_id);  // 传递 filter_id
    },
    // 新增：临时禁用当前滤镜以预览效果
    this.Base_layers.disable_filter(filter_id);
    this.POP.show(settings);
    this.Base_layers.disable_filter(null);
}
```

**关键变化**：
- 新增 `filter_id` 参数用于区分"新增滤镜"和"编辑现有滤镜"
- 新增 `disable_filter` 调用，在编辑时临时禁用该滤镜，避免重复应用

### 2.2 `save` 方法的变化

```javascript
// 改动前
save(params, type) {
    return app.State.do_action(
        new app.Actions.Add_layer_filter_action(null, type, params)
    );
}

// 改动后
save(params, type, filter_id) {  // 新增 filter_id 参数
    return app.State.do_action(
        new app.Actions.Add_layer_filter_action(null, type, params, filter_id)
    );
}
```

### 2.3 新增 `find_filter_by_id` 方法的封装

```javascript
// 在 css.js 中通过 Base_layers 调用
var filter = this.Base_layers.find_filter_by_id(filter_id, 'blur');
```

---

## 3. 具体滤镜文件改动模式分析

### 3.1 标准改动模式（blur、brightness、contrast、grayscale、hue-rotate、invert、saturate、sepia）

这 8 个滤镜的改动模式完全一致：

```javascript
// 改动前
blur() {
    var params = [
        {name: "value", title: "Percentage:", value: 5, range: [0, 50]},
    ];
    this.show_dialog('blur', params);
}

// 改动后
blur(filter_id) {  // 新增参数
    var filter = this.Base_layers.find_filter_by_id(filter_id, 'blur');  // 查找现有滤镜

    var params = [
        {name: "value", title: "Percentage:", value: filter.value ??= 5, range: [0, 50]},  // 使用空值合并运算符
    ];
    this.show_dialog('blur', params, filter_id);  // 传递 filter_id
}
```

**模式总结**：
1. 方法签名增加 `filter_id` 参数
2. 调用 `find_filter_by_id` 获取现有滤镜参数
3. 使用 `??=`（空值合并赋值运算符）设置默认值：如果 `filter.value` 存在则使用，否则使用默认值
4. 将 `filter_id` 传递给 `show_dialog`

### 3.2 特殊处理：shadow.js

`shadow.js` 与其他滤镜有显著不同：

```javascript
// 改动前
shadow() {
    var params = [
        {name: "x", title: "Offset X:", value: 10, range: [-100, 100]},
        {name: "y", title: "Offset Y:", value: 10, range: [-100, 100]},
        {name: "value", title: "Radius:", value: 5, range: [0, 100]},
        {name: "color", title: "Color:", value: "#000000", type: 'color'},
    ];
    this.show_dialog('drop-shadow', params);  // 注意：类型是 'drop-shadow'
}

// 改动后
shadow(filter_id) {
    var filter = this.Base_layers.find_filter_by_id(filter_id, 'shadow');  // 注意：查找用 'shadow'

    var params = [
        {name: "x", title: "Offset X:", value: filter.x ??= 10, range: [-100, 100]},
        {name: "y", title: "Offset Y:", value: filter.y ??= 10, range: [-100, 100]},
        {name: "value", title: "Radius:", value: filter.value ??= 5, range: [0, 100]},
        {name: "color", title: "Color:", value: filter.color ??= "#000000", type: 'color'},
    ];
    this.show_dialog('shadow', params, filter_id);  // 注意：改为 'shadow'
}
```

**差异点**：

| 项目 | 其他滤镜 | shadow |
|------|---------|--------|
| 参数数量 | 1 个（value） | 4 个（x, y, value, color） |
| show_dialog 类型 | 与滤镜名一致 | 改动前是 `drop-shadow`，改动后改为 `shadow` |
| find_filter_by_id 查找名 | 与滤镜名一致 | 使用 `shadow`（与 show_dialog 改动后一致） |

**潜在问题**：
- `shadow.js` 中类名是 `Effects_brightness_class`，疑似复制粘贴错误，应为 `Effects_shadow_class`
- `convert_value` 方法中处理了 `shadow` 到 `drop-shadow` 的映射：
  ```javascript
  preview(params, type) {
      if(type == 'shadow'){
          type = 'drop-shadow';  // CSS filter 需要 drop-shadow
      }
      // ...
  }
  ```

---

## 4. 图层面板与滤镜存储分析

### 4.1 滤镜存储结构

滤镜数据存储在图层对象的 `filters` 数组中，每个滤镜对象结构如下：

```javascript
{
    id: 123456789,        // 唯一标识符（随机生成）
    name: "blur",         // 滤镜类型名称
    params: {             // 滤镜参数
        value: 5
        // 其他参数...
    }
}
```

### 4.2 `base-layers.js` 新增方法

```javascript
/**
 * 根据 filter_id 查找滤镜参数
 * @param filter_id 滤镜唯一ID
 * @param filter_name 滤镜名称（用于验证）
 * @param layer_id 图层ID（可选，默认当前图层）
 * @returns {object} 滤镜参数对象
 */
find_filter_by_id(filter_id, filter_name, layer_id) {
    if (typeof layer_id == "undefined") {
        var layer = config.layer;
    } else {
        var layer = this.get_layer(layer_id);
    }

    var filter = {};
    for (var i in layer.filters) {
        if (
            layer.filters[i].name == filter_name &&
            layer.filters[i].id == filter_id
        ) {
            return layer.filters[i].params;  // 返回参数对象
        }
    }

    return filter;  // 未找到返回空对象
}
```

### 4.3 `gui-layers.js` 的交互入口

**渲染滤镜列表**（新增部分）：

```javascript
// 显示滤镜
if (layers[i].filters.length > 0) {
    html += '<div class="filters">';
    for (var j in layers[i].filters) {
        var filter = layers[i].filters[j];
        var title = this.Helper.ucfirst(filter.name);
        title = title.replace(/-/g, ' ');

        html += '<div class="filter">';
        html += '  <span class="delete" id="delete_filter" data-pid="' + layers[i].id + '" data-id="' + filter.id + '"></span>';
        html += '  <span class="layer_name" id="filter_name" data-pid="' + layers[i].id + '" data-id="' + filter.id + '" data-filter="' + filter.name + '">' + title + '</span>';
        html += '</div>';
    }
    html += '</div>';
}
```

**编辑滤镜事件处理**：

```javascript
else if (target.id == 'filter_name') {
    // 编辑滤镜
    var effects = _this.Effects_browser.get_effects_list();
    var key = target.dataset.filter.toLowerCase();
    for (var i in effects) {
        if(effects[i].title.toLowerCase() == key){
            _this.Base_layers.select(target.dataset.pid);  // 选中对应图层
            var function_name = _this.Effects_browser.get_function_from_path(key);
            effects[i].object[function_name](target.dataset.id);  // 传入 filter_id
        }
    }
}
```

### 4.4 `add-layer-filter.js` 的增改逻辑

```javascript
async do() {
    super.do();
    this.reference_layer = app.Layers.get_layer(this.layer_id);
    
    var filter = {
        id: this.filter_id,
        name: this.name,
        params: this.params,
    };
    
    if(this.filter_id) {
        // 编辑模式：更新现有滤镜
        for(var i in this.reference_layer.filters) {
            if(this.reference_layer.filters[i].id == this.filter_id){
                this.reference_layer.filters[i] = filter;
                break;
            }
        }
    }
    else{
        // 新增模式：生成新 ID 并添加
        filter.id = Math.floor(Math.random() * 999999999) + 1;
        this.reference_layer.filters.push(filter);
    }
    
    config.need_render = true;
    app.GUI.GUI_layers.render_layers();  // 刷新图层面板
}
```

---

## 5. 模块边界与依赖评估

### 5.1 模块职责划分

```
┌─────────────────────────────────────────────────────────────────┐
│                        GUI 模块                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ gui-layers.js                                          │   │
│  │ - 渲染滤镜列表                                          │   │
│  │ - 处理双击编辑事件                                       │   │
│  │ - 调用 Effects_browser 打开滤镜对话框                    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Effects 模块                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ abstract/css.js (抽象层)                                │   │
│  │ - 提供 show_dialog 通用方法                              │   │
│  │ - 管理滤镜预览和保存逻辑                                  │   │
│  │ - 依赖: Base_layers (disable_filter)                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ common/*.js (具体滤镜)                                  │   │
│  │ - 定义滤镜参数结构                                       │   │
│  │ - 调用 find_filter_by_id 获取现有参数                    │   │
│  │ - 依赖: Base_layers                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Core 模块                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ base-layers.js                                         │   │
│  │ - 管理图层和滤镜数据                                     │   │
│  │ - 提供 find_filter_by_id 方法                           │   │
│  │ - 提供 disable_filter 方法                              │   │
│  │ - 渲染时应用滤镜 (pre_render_object/after_render_object) │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Actions 模块                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ add-layer-filter.js                                    │   │
│  │ - 封装滤镜增改逻辑                                       │   │
│  │ - 支持 undo/redo                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 依赖关系评估

**合理的依赖**：
1. **Effects → Core**: 具体滤镜需要调用 `Base_layers.find_filter_by_id()` 获取参数，这是合理的
2. **GUI → Effects**: 图层面板需要调用滤镜对象的方法打开对话框，通过 `Effects_browser` 中转是合理设计
3. **Effects → Actions**: 通过 `app.Actions.Add_layer_filter_action` 保存滤镜，符合 Action 模式

**潜在问题**：

| 问题 | 位置 | 说明 |
|------|------|------|
| 循环依赖风险 | effects/abstract/css.js | `css.js` 导入 `Base_layers`，而 `Base_layers` 在渲染时需要遍历所有 effects 模块。虽然当前通过 `Base_gui.modules` 动态获取避免了直接导入，但架构上略显复杂 |
| 命名不一致 | shadow.js | 类名 `Effects_brightness_class` 应为 `Effects_shadow_class` |
| 类型映射分散 | css.js + shadow.js | `shadow` 到 `drop-shadow` 的映射分别在 `preview()` 和 `pre_render_object`/`after_render_object` 中处理，容易遗漏 |

### 5.3 改进建议

1. **统一类型映射**：建议在 `css.js` 中集中管理滤镜名称到 CSS filter 名称的映射
2. **修复类名**：`shadow.js` 中的类名应修正为 `Effects_shadow_class`
3. **考虑依赖注入**：`find_filter_by_id` 的调用方式导致每个具体滤镜都需要导入 `Base_layers`，可以考虑通过抽象层统一处理

---

## 6. 总结

### 6.1 改动质量评估

| 维度 | 评分 | 说明 |
|------|------|------|
| 功能完整性 | ⭐⭐⭐⭐⭐ | 完整实现了可编辑滤镜功能 |
| 代码一致性 | ⭐⭐⭐⭐ | 8个滤镜改动模式一致，shadow 有细微差异 |
| 架构合理性 | ⭐⭐⭐⭐ | 模块边界基本清晰，依赖关系合理 |
| 可维护性 | ⭐⭐⭐⭐ | Action 模式支持 undo/redo，便于维护 |

### 6.2 核心实现亮点

1. **巧妙使用 `??=` 运算符**：简洁地实现了"有则用之，无则默认值"的逻辑
2. **Action 模式**：通过 `Add_layer_filter_action` 统一处理新增和编辑，天然支持撤销重做
3. **临时禁用机制**：`disable_filter` 方法在编辑时临时禁用滤镜，避免预览时的重复应用

### 6.3 待改进点

1. `shadow.js` 类名错误需要修复
2. 滤镜名称映射逻辑可以进一步统一
3. 随机 ID 生成方式（`Math.random()`）在高并发场景下存在极低的碰撞概率，建议使用 UUID 库
