# miniPaint Undo/Redo 机制分析报告

## 1. 核心字段与历史栈机制

### 1.1 关键字段解析

在 [base-state.js](src/js/core/base-state.js) 的 `Base_state_class` 类中，定义了以下核心字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| `layers_archive` | Array | **遗留字段**，旧版本用于存储图层快照，现已废弃 |
| `action_history` | Array | 存储 Action 对象的历史记录栈 |
| `action_history_index` | Number | 当前历史位置指针，指向下一个可 redo 的位置 |
| `levels` | Number | **遗留字段**，旧版本快照级别数，值为 3 |
| `action_history_max` | Number | 历史记录上限，默认值 50 |

### 1.2 快照式 vs 命令式

**miniPaint 采用的是命令模式（Command Pattern），而非快照式保存。**

两种方式的对比：

| 特性 | 快照式（Memento） | 命令式（Command） |
|------|------------------|------------------|
| 存储内容 | 完整状态副本 | 操作指令 + 必要的反向数据 |
| 内存占用 | 高（每次全量拷贝） | 低（只存增量） |
| 实现复杂度 | 简单 | 较复杂（需为每种操作定义 Action） |
| 灵活性 | 低 | 高（可组合、可合并） |
| 恢复速度 | 快（直接替换） | 取决于操作复杂度 |

代码中的 `save()` 方法已被标记为废弃：
```javascript
save() {
    const message = 'window.State.save() is removed. Use State.do_action() to manage undo history instead.';
    console.warn(message);
    alertify.error(message);
}
```

---

## 2. save_state 调用时机与存档方式

### 2.1 新的历史管理机制

**注意：`save_state` 方法已不存在，取而代之的是 `do_action` 方法。**

历史存档通过 `app.State.do_action(action, options)` 触发，调用时机包括：

#### 用户操作触发场景

1. **绘图工具操作**（brush, pencil 等）
   - 鼠标按下时创建新图层或更新图层数据
   - 鼠标抬起时更新图层边界

2. **图层操作**
   - 插入图层：`Insert_layer_action`
   - 删除图层：`Delete_layer_action`
   - 更新图层属性：`Update_layer_action`
   - 更新图层图像：`Update_layer_image_action`

3. **图像处理操作**
   - 滤镜效果、旋转、翻转、裁剪等

4. **文件操作**
   - 打开文件、新建画布

### 2.2 存档方式：增量存储

**Action 存储的是反向恢复所需的数据，而非完整快照：**

以 `Update_layer_action` 为例：
```javascript
constructor(layer_id, settings) {
    this.layer_id = layer_id;
    this.settings = settings;        // 新值
    this.old_settings = {};          // 旧值（do 时记录）
}

async do() {
    this.reference_layer = app.Layers.get_layer(this.layer_id);
    for (let i in this.settings) {
        this.old_settings[i] = this.reference_layer[i];  // 记录旧值
        this.reference_layer[i] = this.settings[i];      // 应用新值
    }
}

async undo() {
    for (let i in this.old_settings) {
        this.reference_layer[i] = this.old_settings[i];  // 恢复旧值
    }
}
```

### 2.3 图像数据的存储

对于图像类型图层，使用 **IndexedDB** 存储图像数据：

```javascript
// image-store.js
async add(imageData) {
    let imageId = tabUuid + '-' + (imageIdCounter++);
    // 存储到 IndexedDB
    const image = { id: imageId, tabUuid, data: imageData };
    images.add(image);
    return imageId;
}
```

图层对象只保存图像 ID 引用：
```javascript
this.reference_layer._link_database_id = this.new_image_id;
```

### 2.4 连续 100 笔的内存占用估算

假设画布尺寸 1000x1000 像素：

| 存储方式 | 单笔内存 | 100 笔内存 |
|---------|---------|-----------|
| 快照式（PNG） | ~1-3 MB | 100-300 MB |
| 命令式（矢量数据） | ~1-10 KB | 0.1-1 MB |

**miniPaint 的 brush/pencil 工具存储的是矢量路径数据：**
```javascript
// 每个点存储 [x, y, size]
current_group.push([mouse_x, mouse_y, new_size]);
```

连续 100 笔的内存占用约为 **几百 KB 到几 MB**，远小于快照式方案。

---

## 3. undo/redo 方法实现细节

### 3.1 核心方法

```javascript
async undo_action() {
    if (this.can_undo()) {
        this.action_history_index--;
        await this.action_history[this.action_history_index].undo();
    }
}

async redo_action() {
    if (this.can_redo()) {
        const action = this.action_history[this.action_history_index];
        await action.do();
        this.action_history_index++;
    }
}
```

### 3.2 状态恢复机制

**恢复过程不涉及 `layers_archive`，而是直接操作 `config.layers` 数组：**

1. **图层引用处理**
   ```javascript
   // Update_layer_action.undo()
   this.reference_layer = app.Layers.get_layer(this.layer_id);
   for (let i in this.old_settings) {
       this.reference_layer[i] = this.old_settings[i];
   }
   ```
   - 通过 `layer_id` 从 `config.layers` 获取图层引用
   - 直接修改图层对象属性

2. **图像恢复**
   ```javascript
   // Update_layer_image_action.undo()
   this.reference_layer.link.src = await image_store.get(this.old_image_id);
   ```
   - 从 IndexedDB 读取旧图像数据
   - 更新 Image 对象的 src 属性

3. **图层增删恢复**
   ```javascript
   // Delete_layer_action.undo()
   config.layers.splice(this.delete_index, 0, this.deleted_layer);
   
   // Insert_layer_action.undo()
   config.layers.pop();
   ```

### 3.3 悬挂引用风险分析

**存在潜在悬挂引用的场景：**

1. **图层删除后恢复**
   - `Delete_layer_action` 在 `free()` 时会删除 `deleted_layer.link`
   - 如果在 undo 后调用 free，可能导致引用失效

2. **跨 Action 引用**
   - 某些 Action 可能持有其他图层的引用
   - 如果被引用图层被删除，会产生悬挂引用

**代码中的防护措施：**
```javascript
// Delete_layer_action.free()
free() {
    if (this.deleted_layer) {
        delete this.deleted_layer.link;
        delete this.deleted_layer.data;
    }
    this.deleted_layer = null;
}
```

---

## 4. 快捷键与菜单按钮调用链

### 4.1 快捷键监听

在 [base-state.js#L38-51](src/js/core/base-state.js#L38-51)：

```javascript
set_events() {
    document.addEventListener('keydown', (event) => {
        const key = (event.key || '').toLowerCase();
        if (this.Helper.is_input(event.target))
            return;

        if (key == "z" && (event.ctrlKey == true || event.metaKey)) {
            this.undo();
            event.preventDefault();
        }
        if (key == "y" && (event.ctrlKey == true || event.metaKey)) {
            this.redo();
            event.preventDefault();
        }
    }, false);
}
```

### 4.2 菜单按钮调用链

**Undo 按钮：**

[modules/edit/undo.js](src/js/modules/edit/undo.js)：
```javascript
document.querySelector('#undo_button').addEventListener('click', function (event) {
    _this.Base_state.undo();
});
```

**Redo 按钮：**

[modules/edit/redo.js](src/js/modules/edit/redo.js)：
```javascript
redo() {
    this.Base_state.redo();
}
```

### 4.3 调用链对比

| 触发方式 | 调用链 |
|---------|--------|
| Ctrl+Z | `keydown` → `Base_state.undo()` → `undo_action()` |
| Undo 按钮 | `click` → `Edit_undo.undo()` → `Base_state.undo()` → `undo_action()` |
| Ctrl+Y | `keydown` → `Base_state.redo()` → `redo_action()` |
| Redo 按钮 | `click` → `Edit_redo.redo()` → `Base_state.redo()` → `redo_action()` |

**结论：快捷键和菜单按钮最终走的是同一条调用链，都汇聚到 `Base_state_class` 的 `undo()`/`redo()` 方法。**

---

## 5. 工程评估与潜在问题

### 5.1 与经典命令模式的对比

| 方面 | miniPaint 实现 | 经典命令模式 |
|------|---------------|-------------|
| Action 定义 | 每个操作独立类 | 接口统一，execute/undo |
| 命令组合 | `Bundle_action` 支持组合 | 通常支持宏命令 |
| 历史管理 | 内置于 State 类 | 可分离为 HistoryManager |
| 内存管理 | 显式 `free()` 方法 | 通常依赖 GC |

### 5.2 action_history_max=50 的回收机制

```javascript
this.action_history.push(action);
if (this.action_history.length > this.action_history_max) {
    let action_to_free = this.action_history.shift();  // 移除最旧记录
    await action_to_free.free();                       // 释放资源
}
```

**回收策略：FIFO（先进先出）**

### 5.3 内存压力检测

```javascript
if (window.performance && window.performance.memory) {
    if (window.performance.memory.usedJSHeapSize > window.performance.memory.jsHeapSizeLimit * 0.8) {
        this.free(window.performance.memory.jsHeapSizeLimit * 0.2);
    }
}
```

当内存使用超过 80% 时，主动释放 20% 的堆空间对应的历史记录。

### 5.4 潜在问题

#### 5.4.1 状态不一致风险

1. **Action 执行失败**
   ```javascript
   try {
       await action.do();
   } catch (error) {
       return { status: 'aborted', reason: error };
   }
   ```
   - 如果 Action 执行中途失败，可能导致部分状态已变更
   - `Bundle_action` 有回滚机制，但单个 Action 需自行处理

2. **异步操作竞态**
   - 多个异步 Action 同时执行可能导致状态混乱
   - 当前实现未加锁保护

#### 5.4.2 旧引用残留

1. **图层引用未清理**
   ```javascript
   // Update_layer_action.free()
   free() {
       this.settings = null;
       this.old_settings = null;
       this.reference_layer = null;  // 只是置空，不影响原图层
   }
   ```

2. **IndexedDB 数据残留**
   - 如果 `free()` 调用失败，IndexedDB 中的图像数据可能残留
   - 代码中有错误提示，但未强制清理

#### 5.4.3 内存泄漏风险

1. **闭包引用**
   - Action 对象可能通过闭包持有大量数据
   - `free()` 方法需要显式清理

2. **事件监听器**
   - 部分工具类可能注册了事件监听器
   - Action 释放时未自动移除

### 5.5 优点总结

1. **内存效率高**：增量存储，只保存必要数据
2. **可扩展性好**：新增操作只需定义新 Action 类
3. **支持操作合并**：`merge_with_history` 选项可合并连续操作
4. **持久化支持**：IndexedDB 存储大图像数据
5. **内存压力感知**：主动检测并释放内存

### 5.6 改进建议

1. **添加操作锁**：防止异步操作竞态
2. **事务机制**：确保 Action 原子性
3. **引用计数**：追踪图层引用，防止悬挂引用
4. **定期清理**：定时清理 IndexedDB 中的孤立数据
5. **压缩存储**：对矢量数据使用二进制格式压缩

---

## 6. 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      User Interface                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Ctrl+Z/Y    │  │ Menu Button │  │ Tool Operations     │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
└─────────┼────────────────┼────────────────────┼─────────────┘
          │                │                    │
          ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   Base_state_class (State)                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ action_history: [Action, Action, ..., Action]       │    │
│  │ action_history_index: 3                              │    │
│  │ action_history_max: 50                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Methods:                                                    │
│  - do_action(action) → push to history                      │
│  - undo_action() → action_history[index--].undo()           │
│  - redo_action() → action_history[index++].do()             │
│  - free() → release old actions                             │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      Action Classes                          │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ Base_action      │  │ Bundle_action    │                 │
│  │ - do()           │  │ - actions[]      │                 │
│  │ - undo()         │  │ - do() all       │                 │
│  │ - free()         │  │ - undo() reverse │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ Update_layer     │  │ Update_layer_img │                 │
│  │ - old_settings   │  │ - old_image_id   │───┐             │
│  │ - new_settings   │  │ - new_image_id   │   │             │
│  └──────────────────┘  └──────────────────┘   │             │
│                                                  │             │
│  ┌──────────────────┐  ┌──────────────────┐   │             │
│  │ Insert_layer     │  │ Delete_layer     │   │             │
│  │ - layer_data     │  │ - deleted_layer  │   │             │
│  └──────────────────┘  └──────────────────┘   │             │
└─────────────────────────────────────────────────┼───────────┘
                                                  │
                           ┌──────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    IndexedDB (image-store)                   │
│  Database: undoHistoryImageStore                            │
│  ObjectStore: images                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ { id: "uuid-0", tabUuid: "...", data: "dataURL" }   │    │
│  │ { id: "uuid-1", tabUuid: "...", data: "dataURL" }   │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 关键代码位置索引

| 功能 | 文件路径 | 关键行号 |
|------|---------|---------|
| 状态管理核心 | [src/js/core/base-state.js](src/js/core/base-state.js) | 全文 |
| Action 基类 | [src/js/actions/base.js](src/js/actions/base.js) | L1-19 |
| Action 组合 | [src/js/actions/bundle.js](src/js/actions/bundle.js) | L1-59 |
| 图像存储 | [src/js/actions/store/image-store.js](src/js/actions/store/image-store.js) | 全文 |
| 图层更新 Action | [src/js/actions/update-layer.js](src/js/actions/update-layer.js) | L1-67 |
| 图像更新 Action | [src/js/actions/update-layer-image.js](src/js/actions/update-layer-image.js) | L1-149 |
| Undo 按钮模块 | [src/js/modules/edit/undo.js](src/js/modules/edit/undo.js) | L1-31 |
| Redo 按钮模块 | [src/js/modules/edit/redo.js](src/js/modules/edit/redo.js) | L1-14 |
