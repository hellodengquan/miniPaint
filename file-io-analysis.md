# miniPaint 文件 IO 模块分析报告

## 1. 功能入口模块概览

| 文件名 | 核心功能 | 快捷键 | 主要职责 |
|--------|----------|--------|----------|
| **new.js** | 新建文件/画布 | - | 提供新建画布对话框，支持自定义尺寸、预设分辨率、透明背景设置 |
| **open.js** | 打开文件 | O | 本地文件选择、拖放、URL/数据URL打开、剪贴板粘贴、摄像头捕获、目录遍历、JSON工程加载、EXIF提取 |
| **save.js** | 导出/保存 | S (导出) Shift+S (保存工程) | 多格式导出、质量控制、图层导出选项、文件大小预览 |
| **print.js** | 打印功能 | - | 调用浏览器原生打印接口 |
| **quickload.js** | 快速加载 | F10 | 从localStorage恢复工程数据 |
| **quicksave.js** | 快速保存 | F9 | 将工程数据存入localStorage |

---

## 2. open.js - 文件打开链路深度分析

### 2.1 浏览器 File API 读取流程

**完整链路：用户选择文件 → 图像出现在画布**

```
用户点击"打开"或按O键
    ↓
open_file() 动态创建 <input type="file"> 并触发click
    ↓
用户选择本地文件
    ↓
open_handler(e) 接收 change 事件
    ├─ 获取 FileList 对象 (e.target.files)
    ├─ 支持多文件，按文件名排序
    └─ 识别目录拖放（通过 webkitGetAsEntry）
        ↓
针对每个文件：
    ├─ 创建 FileReader 对象 (FR = new FileReader())
    ├─ 根据类型选择读取方式：
    │   ├─ 图片文件 → FR.readAsDataURL(f)
    │   └─ JSON工程 → FR.readAsText(f)
    └─ FR.onload 回调处理数据
        ↓
图片文件处理分支：
    ├─ 构建 new_layer 对象：
    │   ├─ name: 文件名
    │   ├─ type: 'image'
    │   ├─ data: event.target.result (base64 DataURL)
    │   ├─ order: 排序号
    │   └─ _exif: EXIF元数据（通过exif-js库）
    ↓
app.State.do_action() 执行动作
    ↓
Insert_layer_action.do() 处理图层插入
    ├─ 补全图层默认属性
    ├─ layer.data 是 base64 字符串时：
    │   ├─ 创建 new Image()
    │   ├─ 设置 img.src = layer.data
    │   ├─ onload 时：
    │   │   ├─ 更新图层宽高
    │   │   ├─ layer.link 指向 Image 对象
    │   │   ├─ layer.data = null (释放内存)
    │   │   └─ 设置 config.need_render = true
    │   └─ 触发 Autoresize_canvas_action
    ├─ 将 layer 推入 config.layers 数组
    ├─ 设置为当前活动图层
    ├─ app.Layers.render() 触发渲染
    └─ app.GUI.GUI_layers.render_layers() 更新图层列表UI
        ↓
画布显示图像
```

### 2.2 支持的打开方式
- 本地文件选择对话框
- 拖放文件/目录到浏览器窗口
- 打开网络图片URL
- 打开DataURL
- 摄像头拍照
- 剪贴板粘贴
- URL参数自动加载 (`?image=xxx`)
- 搜索媒体库

---

## 3. save.js - 导出格式处理机制

### 3.1 导出格式总览

| 格式 | 导出技术 | 第三方依赖 | 支持透明度 | 质量参数 |
|------|----------|-----------|-----------|----------|
| **PNG** | `canvas.toBlob()` | - | ✅ | ❌ |
| **JPG** | `canvas.toBlob()` | - | ❌ (自动加白底) | ✅ 1-100 |
| **WEBP** | `canvas.toBlob()` | - | ✅ | ✅ 1-100 |
| **BMP** | `canvas.toBlob()` | - | ❌ | ❌ |
| **AVIF** | `canvas.toBlob()` | - | ✅ | ✅ 1-100 |
| **GIF** | gif.js.optimized | gif.js.optimized | ✅ | ✅ |
| **TIFF** | CanvasToTIFF.toBlob() | canvastotiff.js | ✅ | ❌ |
| **JSON** | 自定义序列化 | - | - | - |

### 3.2 位图导出技术选择

**所有位图格式（PNG/JPG/WEBP/BMP/AVIF）统一使用 `canvas.toBlob()`** 而非 `toDataURL()`，优势：
- 异步处理，不阻塞主线程
- 直接生成Blob对象，内存效率更高
- 配合 FileSaver 库触发下载
- 代码示例：
```javascript
canvas.toBlob(function (blob) {
    filesaver.saveAs(blob, fname);
}, "image/jpeg", quality);
```

### 3.3 特殊格式处理

#### GIF 导出
- **依赖库**：`gif.js.optimized` (优化版的GIF编码器)
- **工作方式**：多线程WebWorker编码，支持多核CPU
- **核心配置**：
  ```javascript
  {
    workers: navigator.hardwareConcurrency || 4,  // CPU核心数
    quality: 10,          // 1-30，值越小质量越高
    repeat: 0,            // 循环播放
    dither: 'FloydSteinberg-serpentine', // 抖动算法
    transparent: 'rgba(0,0,0,0)' // 透明色
  }
  ```
- **动画机制**：每个可见图层作为一帧，支持自定义帧延迟

#### TIFF 导出
- **依赖库**：自定义 `CanvasToTIFF` 库 (`./../../libs/canvastotiff.js`)
- 直接调用：`CanvasToTIFF.toBlob(canvas, callback, mimeType)`

### 3.4 JSON 工程文件结构

```typescript
{
  info: {
    width: number,           // 画布宽度
    height: number,          // 画布高度
    about: string,           // 软件信息URL
    date: string,            // 导出日期 YYYY-MM-DD
    version: string,         // 软件版本号
    layer_active: number,    // 活动图层ID
    guides: Guide[]          // 参考线数据
  },
  user_fonts: FontMeta,      // 用户自定义字体
  layers: Layer[],           // 所有图层元数据（排除_开头私有字段）
  data: [                    // 图片图层像素数据（仅image类型）
    {
      id: number,            // 对应图层ID
      data: string           // PNG base64 DataURL
    }
  ]
}
```

**关键设计：**
- 图层元数据与像素数据分离存储
- 非图片图层（形状、文字等）只存储在 `layers` 数组
- 像素数据统一转成PNG格式保存，不保留原始格式
- 支持版本迁移（目前已支持v3→v4, v4.5, v4.8, v4.11等多个版本的兼容）

---

## 4. quickload / quicksave 设计分析

### 4.1 设计场景
- **目标用户**：需要临时保存/恢复工作进度的用户
- **典型场景**：
  - 尝试危险操作前快速存档
  - 浏览器刷新/崩溃后快速恢复
  - 不同操作分支间快速切换
  - 无需下载/上传文件的临时存储

### 4.2 数据存储位置
- **存储引擎**：浏览器 `localStorage`
- **Key名**：`quicksave_data`
- **容量限制**：5MB（超过时提示错误）

### 4.3 与正常 save/open 的差异对比

| 特性 | Quicksave/Quickload | 正常 Save/Open |
|------|---------------------|---------------|
| 文件对话框 | ❌ 无 | ✅ 有 |
| 格式选择 | ❌ 仅JSON | ✅ 8种格式 |
| 用户交互 | ❌ 一键完成 | ✅ 参数配置 |
| 文件名 | ❌ 固定key | ✅ 自定义 |
| 持久化 | ❌ 依赖浏览器存储 | ✅ 本地文件 |
| 容量限制 | ✅ 5MB硬限制 | ❌ 无 |
| 执行速度 | ✅ 极快 | ❌ 相对较慢 |

### 4.4 实现机制
```javascript
// quicksave.js - 33-41行
quicksave() {
  var data_json = this.File_save.export_as_json();  // 复用JSON导出逻辑
  if (data_json.length > 5000000) { /* 容量检查 */ }
  localStorage.setItem('quicksave_data', data_json);
}

// quickload.js - 33-42行
quickload() {
  var json = localStorage.getItem('quicksave_data');
  this.File_open.load_json(json);  // 复用JSON加载逻辑
}
```

**设计亮点**：高度复用 save/open 的 JSON 序列化/反序列化代码，避免重复实现。

---

## 5. 文件打开 → 图层创建的完整调用链

### 5.1 跨模块调用链路

```
┌─────────────────────────────────────────────────────────────────┐
│  用户交互层 (open.js)                                            │
│  File_open_class.open_handler()                                 │
│    - FileReader 读取文件 → DataURL                               │
│    - 构建 new_layer 初始数据                                      │
│    - 提取 EXIF 元数据                                            │
└───────────────────────────┬─────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  状态管理层 (app.js → State)                                     │
│  app.State.do_action()                                          │
│    - 支持 Bundle_action 批量执行                                  │
│    - 支持撤销/重做历史记录                                        │
└───────────────────────────┬─────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  动作执行层 (actions/insert-layer.js)                            │
│  Insert_layer_action.do()                                       │
│    - 补全图层默认属性（id, name, opacity等）                      │
│    - Image 对象异步加载和onload处理                               │
│    - 处理空图层覆盖逻辑                                          │
│    - 自动调整画布大小 Autoresize_canvas_action                   │
│    - 更新 config.layers 数组                                     │
└───────────────────────────┬─────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  渲染层 (core/base-layers.js)                                    │
│  app.Layers.render()                                            │
│    - 图层按order排序                                             │
│    - 调用 convert_layers_to_canvas()                             │
│    - 绘制到主画布                                                │
└───────────────────────────┬─────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  UI层 (core/gui/gui-layers.js)                                   │
│  app.GUI.GUI_layers.render_layers()                             │
│    - 更新图层列表DOM                                             │
│    - 显示图层缩略图                                              │
│    - 高亮当前活动图层                                            │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 关键数据流动
1. **File → DataURL**：FileReader API 转换
2. **DataURL → Image对象**：new Image() + onload 回调
3. **Image对象 → 图层属性**：宽高自动赋值
4. **图层对象 → config.layers**：全局状态数组
5. **图层数组 → Canvas像素**：图层合成渲染

---

## 6. 可维护性评估与架构建议

### 6.1 新增 BMP 格式需要修改的位置

基于现有代码，新增 BMP 格式需要修改 **仅 3 处**：

1. **`save.js:37-46`** - 在 `SAVE_TYPES` 字典添加条目：
   ```javascript
   BMP: "Windows Bitmap",
   ```

2. **`save.js:402-415`** - save_dialog_onchange() 添加文件大小计算：
   ```javascript
   else if (type == 'BMP') {
       var data_header = "image/bmp";
       if (this.check_format_support(canvas, data_header, false) == false) {
           this.update_file_size('-');
           return;
       }
       canvas.toBlob(function (blob) {
           _this.update_file_size(blob.size);
       }, data_header);
   }
   ```

3. **`save.js:556-569`** - save_action() 添加导出分支：
   ```javascript
   else if (type == 'BMP') {
       if (this.Helper.strpos(fname, '.bmp') == false)
           fname = fname + ".bmp";
       var data_header = "image/bmp";
       if (this.check_format_support(canvas, data_header) == false)
           return false;
       canvas.toBlob(function (blob) {
           filesaver.saveAs(blob, fname);
       }, data_header);
   }
   ```

> ✅ **实际观察**：代码中 BMP 格式已经完整实现了！

### 6.2 耦合度分析

**当前耦合度：中等偏高**

| 问题 | 表现 |
|------|------|
| **格式分支散列** | 每种格式的处理逻辑重复出现在 `save_dialog_onchange()` 和 `save_action()` 两个函数中，形成镜像分支 |
| **硬编码判断** | `type == 'PNG'` 这类字符串比较散落在 10+ 处 |
| **UI与业务逻辑混合** | DOM操作（`style.display`）与导出计算逻辑在同一函数 |

**代码重复示例：**
- `save_dialog_onchange()`: 260-435行，8种格式分支
- `save_action()`: 503-625行，8种格式分支
- **两处的分支判断完全对应，逻辑高度重复**

### 6.3 可抽取的公共流程

**建议重构为格式处理器注册表模式：**

```javascript
// 理想架构
const FormatHandlers = {
  PNG: {
    extension: '.png',
    supportTransparency: true,
    supportQuality: false,
    calculateSize: (canvas, quality) => {...},
    export: (canvas, filename, quality) => {...}
  },
  JPG: { /* ... */ },
  // ...
}

// 好处：
// 1. 新增格式只需注册一个新对象
// 2. 消除重复的 switch/if-else
// 3. 支持动态注册/卸载格式
// 4. 易于单元测试每个格式
```

### 6.4 现有架构优点

1. **Action 模式优秀**：文件操作都通过 State + Action 执行，支持完整的撤销/重做
2. **Quicksave 高度复用**：没有重复实现序列化逻辑
3. **JSON 版本兼容机制完善**：多版本迁移策略清晰
4. **依赖注入清晰**：各模块通过 app 单例互相访问，没有硬耦合
5. **异常处理到位**：图片加载失败、格式不支持、存储超限等都有用户提示

---

## 分析总结

miniPaint 的文件 IO 模块是一套**设计成熟、功能完整**的浏览器端图像处理解决方案：

- ✅ 覆盖了桌面级图像编辑器的主要文件操作
- ✅ 针对浏览器环境做了大量优化（File API、localStorage、WebWorker）
- ✅ 多版本兼容机制确保工程文件的向后兼容
- ✅ Action 模式的使用保证了状态一致性

主要的改进空间在于**格式处理器的抽象统一**，将目前的多分支重复逻辑重构为注册表模式，可进一步提升可维护性和扩展性。
