# miniPaint Undo/Redo 机制深度分析报告

## 一、核心架构与字段说明

### 1.1 Base_state_class 核心字段解析

| 字段名 | 声明位置 | 实际用途 | 备注 |
|--------|---------|---------|------|
| `layers_archive` | `base-state.js:30` | **已废弃** | 代码中仅声明但未使用，是旧版快照式架构的遗留物 |
| `action_history` | `base-state.js:34` | 命令历史栈 | 存储所有可撤销/重做的 Action 对象数组 |
| `action_history_index` | `base-state.js:35` | 历史游标 | 当前在历史栈中的位置指针，undo 时递减，redo 时递增 |
| `levels` | `base-state.js:31` | **已废弃** | 原快照层数配置，现无实际作用 |
| `action_history_max` | `base-state.js:36` | 历史上限 | 最大记录 50 步，超出后队首出队 |

### 1.2 架构模式：从快照式到命令式的演进

#### 当前实现：命令模式（Command Pattern）
```
┌─────────────────────────────────────────────────────────┐
│                     Base_state_class                    │
├─────────────────────────────────────────────────────────┤
│  action_history: [Action1, Action2, Action3, ...]       │
│                              ↑                          │
│                    action_history_index                 │
└─────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌───────────┐       ┌───────────┐       ┌───────────┐
    │ Update_.. │       │ Insert_.. │       │ Delete_.. │
    ├───────────┤       ├───────────┤       ├───────────┤
    │ do()      │       │ do()      │       │ do()      │
    │ undo()    │       │ undo()    │       │ undo()    │
    │ free()    │       │ free()    │       │ free()    │
    └───────────┘       └───────────┘       └───────────┘
```

#### 快照式 vs 命令式 对比

| 维度 | 快照式 (Snapshot) | 命令式 (Command) |
|------|------------------|------------------|
| **存储内容** | 完整图层数据深拷贝 | 操作本身 + 反向操作所需数据 |
| **内存占用** | 高，每步 ~几 MB | 低，增量数据 / 差异数据 |
| **实现难度** | 简单，无脑复制 | 复杂，每个操作需独立开发 |
| **回退速度** | 快，直接替换 | 需执行反向逻辑 |
| **合并操作** | 困难 | 支持 Bundle 合并 |
| **内存泄漏** | 易残留对象引用 | 需手动 free 资源 |

---

## 二、状态保存机制分析

### 2.1 do_action 调用时机

**核心入口**：`app.State.do_action(new Action())`

**触发场景**（通过代码 grep 确认）：
- 绘图工具：画笔/铅笔/橡皮擦等 **mouseup** 时
- 图层操作：新增/删除/重命名/合并/复制
- 属性变更：透明度/混合模式/可见性
- 图像操作：滤镜/变换/调整
- 选区操作：移动/变形

**关键代码位置** (`brush.js:221-225`)：
```javascript
// 画笔按下时创建 Bundle Action
app.State.do_action(
    new app.Actions.Bundle_action('new_brush_layer', 'New Brush Layer', [
        new app.Actions.Insert_layer_action(this.layer)
    ])
);
```

### 2.2 存档策略：混合式存储

#### 图层属性变更：纯增量存储
例：`Update_layer_action` 只存变更的字段（如 name, opacity）
- 内存占用：~几十字节 / 步

#### 像素数据变更：IndexedDB 持久化存储
例：`Update_layer_image_action` (`update-layer-image.js:69-79`)
```javascript
// 像素数据不存内存，存 IndexedDB
this.old_image_id = await image_store.add(this.reference_layer.link.src);
this.new_image_id = await image_store.add(canvas_data_url);
```

**image-store 存储策略**：
- 优先使用 IndexedDB 持久化（`image-store.js:29-68`）
- 降级方案：内存对象存储（`image-store.js:80-83`）
- 跨 Tab 隔离：通过 `tabUuid` + `sessionStorage`

### 2.3 连续绘画的内存占用估算

| 操作类型 | 单步内存 | 50步上限 | 100步（超出后自动回收） |
|---------|---------|---------|----------------------|
| 矢量画笔（坐标点数据） | ~5-20KB | 0.25-1MB | 稳定在 0.5-1MB |
| 像素笔刷（1000x1000） | ~50-200KB | 2.5-10MB | 稳定在 5-10MB |
| 全屏滤镜（1920x1080） | ~2-4MB | 100-200MB | 稳定在 100-200MB |

> ✅ **自动防溢出**：`base-state.js:108-112` 监控 JS 堆内存，超过 80% 主动释放 20% 历史

---

## 三、Undo/Redo 恢复机制

### 3.1 Undo 执行流程

```
Ctrl+Z 点击按钮
      ↓
Base_state.undo()  [base-state.js:209-211]
      ↓
Base_state.undo_action()  [base-state.js:138-145]
      ├─ 检查 can_undo() → action_history_index > 0
      ├─ action_history_index-- （游标左移）
      └─ 调用 Action.undo()
           ├─ 恢复对象属性 (Update_layer)
           ├─ 从 image_store 读取旧像素数据
           └─ 设置 config.need_render = true
```

**像素恢复关键代码** (`update-layer-image.js:110-117`)：
```javascript
async undo() {
    // 从 IndexedDB 恢复像素数据
    this.reference_layer.link.src = await image_store.get(this.old_image_id);
    this.reference_layer._link_database_id = this.old_link_database_id;
}
```

### 3.2 图层对象引用处理

| 场景 | 引用策略 | 悬挂引用风险 |
|------|---------|------------|
| Action.do 执行时 | 通过 `layer_id` 查找实时对象 `app.Layers.get_layer(id)` | ⚠️ 中风险 |
| Action 生命周期 | 存储 `reference_layer` 引用，undo 后置 null | ✅ 主动释放 |
| 图层被删除后 undo | 检查 `if (!this.reference_layer) throw Error` | ✅ 有防护 |
| Bundle 嵌套 Action | 独立维护各自引用 | 正常 |

**潜在风险点**：
`update-layer-image.js:31` 通过 layer_id 查找图层，如果图层已被其他操作删除，直接抛出异常中止，不会产生悬挂引用。

### 3.3 Redo 执行流程

```
Ctrl+Y 点击按钮
      ↓
Base_state.redo()  [base-state.js:216-218]
      ↓
Base_state.redo_action()  [base-state.js:128-136]
      ├─ 检查 can_redo()
      ├─ 调用 Action.do()
      └─ action_history_index++ （游标右移）
```

---

## 四、调用链与快捷键分析

### 4.1 三条调用链的交汇点

#### ① 键盘快捷键 (`base-state.js:41-58`)
```javascript
document.addEventListener('keydown', (event) => {
    if (key == "z" && (ctrlKey || metaKey)) {
        this.undo();    // 直接调用
        event.preventDefault();
    }
    if (key == "y" && (ctrlKey || metaKey)) {
        this.redo();    // 直接调用
        event.preventDefault();
    }
});
```

#### ② 工具栏按钮 (`undo.js:18-24`)
```javascript
document.querySelector('#undo_button').addEventListener('click', () => {
    this.Base_state.undo();  // 间接调用
});
```

#### ③ 菜单栏 Edit → Undo
菜单点击事件 → 触发自定义事件 → 最终调用 `Base_state.undo()`

### 4.2 modules/edit/undo.js 的作用

| 职责 | 具体内容 |
|------|---------|
| 按钮绑定 | `#undo_button` click 事件监听 |
| 桥接层 | 作为 UI 层与 core 层的适配器 |
| 单例管理 | 维护 Base_state_class 单例引用 |

> ❗ **注意**：redo.js 实际上**没有绑定按钮事件**，只有空壳的 redo() 桥接方法。

---

## 五、工程化评估

### 5.1 vs 经典命令模式的优缺点对比

| 维度 | miniPaint 实现 | 经典 GoF 命令模式 |
|------|--------------|----------------|
| ✅ **优点1** | 历史上限 + 内存主动回收 | 通常无界 |
| ✅ **优点2** | Action.free() 资源释放接口 | 通常无释放概念 |
| ✅ **优点3** | Bundle_action 支持操作合并 | 需独立实现 Composite |
| ✅ **优点4** | IndexedDB 像素离线存储 | 全部内存存储 |
| ❌ **缺点1** | do_action 自动截断 redo 分支 | 保留分支能力 |
| ❌ **缺点2** | 异步 Action 错误处理薄弱 | 通常同步执行 |
| ❌ **缺点3** | 缺少事务回滚机制 | 部分实现支持 |

### 5.2 历史上限回收机制 (`action_history_max = 50`)

**回收触发点**：`base-state.js:95-104`
```javascript
if (this.action_history.length > this.action_history_max) {
    let action_to_free = this.action_history.shift();  // 队首出队
    await action_to_free.free();                       // 释放资源
    // ⚠️ 注意：此处 action_history_index 不会递减！
} else {
    this.action_history_index++;
}
```

**Bug 风险**：达到 50 步上限后，`action_history_index` 停止增长，历史游标与数组开始错位！

### 5.3 潜在问题分析

#### ① 状态不一致风险
**场景**：用户在第 50 步执行新操作，第 1 步被回收但游标仍为 50。连续 undo 49 次后到达索引 1，第 0 步已无法访问。

#### ② 旧引用残留
**高风险点**：`update-layer-image.js:31` 存储 `reference_layer` 强引用
- Action 被 free 后置 null → ✅ 安全
- Action 仍在历史中 → ⚠️ 图层对象无法 GC
- 影响：已删除的图层对象可能长期驻留内存

#### ③ 内存泄漏路径
1. **image_store 泄漏**：多 Tab 场景下未正常 delete → 浏览器重启后自动清理
2. **Action 引用循环**：Bundle 嵌套引用 → 可达但无用
3. **事件监听**：无 off 对应 → 页面刷新解除

---

## 六、总结与改进建议

### 架构评级：⭐⭐⭐⭐

| 评价项 | 得分 | 说明 |
|-------|------|------|
| 内存效率 | 4/5 | IndexedDB 优化很到位 |
| 鲁棒性 | 3/5 | 边界 case 处理需加强 |
| 可扩展性 | 4/5 | Action 抽象良好 |
| 性能 | 4/5 | 矢量操作几乎零开销 |

### 核心改进建议

1. **游标错位修复**：超限时 `action_history_index` 同步递减
2. **引用弱化**：图层只存 id，不存对象引用
3. **redo 分支保留**：不截断历史，支持多分支时间线
4. **free 错误重试**：单次失败不放弃整个清理流程

---

**文件位置说明**：
- 核心状态机：`src/js/core/base-state.js`
- Action 基类：`src/js/actions/base.js`
- 像素存储：`src/js/actions/store/image-store.js`
- 按钮入口：`src/js/modules/edit/undo.js` / `redo.js`
