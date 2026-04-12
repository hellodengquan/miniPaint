# miniPaint 文件读写模块分析报告

## 1. 功能入口概览

### 1.1 六个核心文件的功能定位

| 文件 | 功能入口 | 主要用途 |
|------|----------|----------|
| [new.js](src/js/modules/file/new.js) | `File → New` | 创建新画布，支持自定义尺寸、预设分辨率、透明背景选项 |
| [open.js](src/js/modules/file/open.js) | `File → Open` / `Ctrl+O` | 打开本地图片文件（PNG/JPG/GIF/WEBP等）、JSON工程文件、支持拖放、URL打开、摄像头捕获 |
| [save.js](src/js/modules/file/save.js) | `File → Save` / `Ctrl+S` / `Ctrl+Shift+S` | 导出为多种图片格式（PNG/JPG/WEBP/GIF/TIFF/BMP）或保存JSON工程文件 |
| [print.js](src/js/modules/file/print.js) | `File → Print` | 调用浏览器打印功能 |
| [quickload.js](src/js/modules/file/quickload.js) | `F10` 快捷键 | 从 localStorage 快速恢复上次保存的状态 |
| [quicksave.js](src/js/modules/file/quicksave.js) | `F9` 快捷键 | 将当前工程快速保存到 localStorage |

---

## 2. open.js 文件读取链路分析

### 2.1 浏览器 File API 读取流程

用户选择文件后的完整处理链路：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         用户触发文件选择                                       │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                       │
│  │  点击打开按钮 │ or │  拖放文件   │ or │  粘贴图片   │                       │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                       │
└─────────┼──────────────────┼──────────────────┼───────────────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         open_handler(e) 入口                                 │
│                                                                             │
│  1. 获取 FileList: e.target.files 或 e.dataTransfer.files                   │
│  2. 文件排序: 按文件名排序确保加载顺序一致                                      │
│  3. 类型检查: f.type.match('image.*') 或 .json                               │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FileReader 读取文件内容                               │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  图片文件 (PNG/JPG/GIF/WEBP等):                                      │   │
│  │  FR.readAsDataURL(f) → 返回 data:image/png;base64,xxx...            │   │
│  │                                                                     │   │
│  │  JSON 工程文件:                                                      │   │
│  │  FR.readAsText(f) → 返回 JSON 字符串                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FR.onload 回调处理                                    │
│                                                                             │
│  图片处理分支:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  var new_layer = {                                                  │   │
│  │      name: file.name,        // 文件名                              │   │
│  │      type: 'image',          // 图层类型                            │   │
│  │      data: event.target.result,  // base64 data URL                 │   │
│  │      order: order,           // 排序                                │   │
│  │      _exif: extract_exif(file)  // EXIF 元数据                      │   │
│  │  };                                                                 │   │
│  │                                                                     │   │
│  │  app.State.do_action(                                               │   │
│  │      new app.Actions.Insert_layer_action(new_layer)                 │   │
│  │  );                                                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  JSON 处理分支:                                                              │
│  └─→ load_json(event.target.result) → 解析并恢复完整工程状态                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 关键代码位置

- **文件选择对话框创建**: [open.js:95-110](src/js/modules/file/open.js#L95-L110)
- **拖放处理**: [open.js:52-60](src/js/modules/file/open.js#L52-L60)
- **FileReader 读取**: [open.js:290-330](src/js/modules/file/open.js#L290-L330)
- **EXIF 数据提取**: [open.js:680-708](src/js/modules/file/open.js#L680-L708)

---

## 3. save.js 导出格式处理分析

### 3.1 支持的导出格式

```javascript
// SAVE_TYPES 配置 (save.js:35-46)
this.SAVE_TYPES = {
    PNG:  "Portable Network Graphics",
    JPG:  "JPG/JPEG Format",
    JSON: "Full layers data",
    WEBP: "Weppy File Format",
    GIF:  "Graphics Interchange Format",
    BMP:  "Windows Bitmap",
    TIFF: "Tag Image File Format",
};
```

### 3.2 各格式的处理方式

| 格式 | 导出方法 | 质量参数 | 第三方库 |
|------|----------|----------|----------|
| **PNG** | `canvas.toBlob(callback)` | ❌ 不支持 | 原生 API |
| **JPG** | `canvas.toBlob(callback, "image/jpeg", quality)` | ✅ 0-100 | 原生 API |
| **WEBP** | `canvas.toBlob(callback, "image/webp", quality)` | ✅ 0-100 | 原生 API |
| **BMP** | `canvas.toBlob(callback, "image/bmp")` | ❌ 不支持 | 原生 API |
| **GIF** | `gif.js.optimized` 库 | ❌ 固定 | ✅ gif.js.optimized |
| **TIFF** | `CanvasToTIFF.toBlob()` | ❌ 不支持 | ✅ canvas-to-tiff |
| **JSON** | `JSON.stringify()` + `Blob` | - | 原生 API |

### 3.3 PNG/JPG/WEBP/BMP 位图导出流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         导出流程（PNG/JPG/WEBP/BMP）                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 创建临时 Canvas                                                          │
│     var canvas = document.createElement('canvas');                          │
│     canvas.width = config.WIDTH;                                            │
│     canvas.height = config.HEIGHT;                                          │
│                                                                             │
│  2. 渲染图层到 Canvas                                                        │
│     Base_layers.convert_layers_to_canvas(ctx, layer_id, is_preview);        │
│                                                                             │
│  3. 处理透明背景（JPG/BMP需要白色背景）                                       │
│     if (type == 'JPG' || config.TRANSPARENCY == false) {                    │
│         ctx.globalCompositeOperation = 'destination-over';                  │
│         fillCanvasBackground(ctx, '#ffffff');                               │
│     }                                                                       │
│                                                                             │
│  4. 导出为 Blob                                                              │
│     ┌─────────────────┬─────────────────────────────────────────────────┐  │
│     │ PNG             │ canvas.toBlob(function(blob) { ... })           │  │
│     │ JPG             │ canvas.toBlob(..., "image/jpeg", quality)       │  │
│     │ WEBP            │ canvas.toBlob(..., "image/webp", quality)       │  │
│     │ BMP             │ canvas.toBlob(..., "image/bmp")                 │  │
│     └─────────────────┴─────────────────────────────────────────────────┘  │
│                                                                             │
│  5. 使用 FileSaver 保存文件                                                  │
│     filesaver.saveAs(blob, filename);                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 GIF 导出（使用 gif.js.optimized）

```javascript
// save.js:640-660
var cores = navigator.hardwareConcurrency || 4;
var gif_settings = {
    workers: cores,                    // 使用多核 Worker
    quality: 10,                       // 1-30, 越低质量越好
    repeat: 0,                         // 循环播放
    width: config.WIDTH,
    height: config.HEIGHT,
    dither: 'FloydSteinberg-serpentine',  // 抖动算法
    workerScript: './src/js/libs/gifjs/gif.worker.js',
};
if (config.TRANSPARENCY == true) {
    gif_settings.transparent = 'rgba(0,0,0,0)';
}
var gif = new GIF(gif_settings);

// 添加每一帧
for (var i = 0; i < config.layers.length; i++) {
    if (config.layers[i].visible == false) continue;
    
    ctx.clearRect(0, 0, config.WIDTH, config.HEIGHT);
    Base_layers.convert_layers_to_canvas(ctx, config.layers[i].id, false);
    gif.addFrame(ctx, {copy: true, delay: delay});
}
gif.render();
gif.on('finished', function (blob) {
    filesaver.saveAs(blob, fname);
});
```

### 3.5 TIFF 导出（使用 canvas-to-tiff）

```javascript
// save.js:615-620
CanvasToTIFF.toBlob(canvas, function(blob) {
    filesaver.saveAs(blob, fname);
}, "image/tiff");
```

**CanvasToTIFF 库位置**: [src/js/libs/canvastotiff.js](src/js/libs/canvastotiff.js)

### 3.6 JSON 工程文件结构

```javascript
// save.js:670-720 export_as_json() 方法
{
    "info": {
        "width": 800,           // 画布宽度
        "height": 600,          // 画布高度
        "about": "...",         // 关于信息
        "date": "2024-01-15",   // 保存日期
        "version": "4.x.x",     // miniPaint 版本
        "layer_active": 5,      // 当前选中图层ID
        "guides": [...]         // 辅助线数据
    },
    "user_fonts": {...},        // 用户自定义字体
    "layers": [                 // 图层元数据数组
        {
            "id": 1,
            "name": "Layer 1",
            "type": "image",    // image/text/shape/gradient等
            "x": 0, "y": 0,
            "width": 800,
            "height": 600,
            "visible": true,
            "opacity": 100,
            "order": 1,
            "filters": [],
            "params": {...},    // 类型特定参数
            "render_function": ["tool_name", "render"]
        }
    ],
    "data": [                   // 图片图层实际像素数据
        {
            "id": 1,
            "data": "data:image/png;base64,iVBORw0KGgo..."
        }
    ]
}
```

---

## 4. QuickLoad / QuickSave 分析

### 4.1 设计场景

| 特性 | QuickSave (F9) | QuickLoad (F10) |
|------|----------------|-----------------|
| **用途** | 快速保存当前工作状态 | 快速恢复到上次保存的状态 |
| **触发方式** | F9 快捷键 | F10 快捷键 |
| **存储位置** | `localStorage.getItem('quicksave_data')` | 从 localStorage 读取 |
| **数据格式** | JSON 字符串（与 Save JSON 相同） | JSON 字符串 |
| **大小限制** | 5MB（代码中硬编码检查） | 无限制 |

### 4.2 与正常 Save/Open 的区别

| 对比项 | QuickSave/QuickLoad | 正常 Save/Open |
|--------|---------------------|----------------|
| **交互方式** | 无对话框，一键操作 | 弹出对话框，需要用户确认 |
| **存储位置** | 浏览器 localStorage | 用户选择的本地文件 |
| **数据持久性** | 浏览器清理后丢失 | 永久保存为文件 |
| **文件命名** | 自动处理 | 用户指定 |
| **格式选择** | 固定为 JSON | 支持多种格式选择 |
| **图层选项** | 保存所有图层 | 可选择 All/Selected/Separated |
| **适用场景** | 临时备份、防止意外关闭 | 长期存档、跨设备传输 |

### 4.3 关键代码

```javascript
// quicksave.js:35-40
quicksave() {
    var data_json = this.File_save.export_as_json();
    if (data_json.length > 5000000) {
        alertify.error('Sorry, image is too big, max 5 MB.');
        return false;
    }
    localStorage.setItem('quicksave_data', data_json);
}

// quickload.js:32-38
quickload() {
    var json = localStorage.getItem('quicksave_data');
    if (json == '' || json == null) {
        return false;
    }
    this.File_open.load_json(json);  // 复用 open.js 的 JSON 加载逻辑
}
```

---

## 5. "打开文件 → 新图层出现" 完整调用链

### 5.1 调用链路图

```
用户选择图片文件
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  File_open_class.open_handler(e)                                           │
│  src/js/modules/file/open.js:252                                           │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ 创建 FileReader
    │ FR.readAsDataURL(file)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  FileReader.onload 回调                                                     │
│  src/js/modules/file/open.js:290-310                                        │
│                                                                             │
│  var new_layer = {                                                          │
│      name: this.file.name,                                                  │
│      type: 'image',                                                         │
│      data: event.target.result,  // base64 Data URL                         │
│      order: order,                                                          │
│      _exif: _this.extract_exif(this.file)                                   │
│  };                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ app.State.do_action(new app.Actions.Insert_layer_action(new_layer))
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Insert_layer_action.do()                                                  │
│  src/js/actions/insert-layer.js:45-150                                      │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ 创建 layer 对象，合并默认配置和用户传入的 settings
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  图片加载处理 (insert-layer.js:72-120)                                      │
│                                                                             │
│  if (layer.type == 'image') {                                               │
│      if (layer.link == null) {                                              │
│          // 从 data URL 创建 Image 对象                                      │
│          layer.link = new Image();                                          │
│          layer.link.onload = () => {                                        │
│              layer.width = layer.link.width;                                │
│              layer.height = layer.link.height;                              │
│              layer.width_original = layer.width;                            │
│              layer.height_original = layer.height;                          │
│              layer.data = null;  // 释放 data URL                            │
│              config.need_render = true;                                     │
│          };                                                                 │
│          layer.link.src = layer.data;                                       │
│      }                                                                      │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ config.layers.push(layer)
    │ app.Layers.auto_increment++
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Base_layers_class.render()                                                │
│  src/js/core/base-layers.js:120-180                                         │
│                                                                             │
│  - 清空画布                                                                  │
│  - 遍历所有图层（按 order 排序）                                              │
│  - 调用 render_object() 绘制每个图层                                          │
│  - 对于 image 类型: ctx.drawImage(layer.link, ...)                          │
│  - 更新预览窗口                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  GUI_layers.render_layers()                                                │
│  更新图层列表面板 UI                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 跨模块调用关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              模块调用关系                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   modules/file/open.js                                                      │
│        │                                                                    │
│        ├──► actions/insert-layer.js  ──► core/base-layers.js               │
│        │              │                          │                          │
│        │              │                          ├──► canvas 渲染           │
│        │              │                          └──► GUI 更新              │
│        │              │                                                     │
│        │              └──► config.layers (全局状态)                          │
│        │                                                                    │
│        └──► libs/popup.js (对话框)                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 代码可维护性评估

### 6.1 新增 BMP 导出格式所需改动

实际上 **BMP 格式已经支持**（原生 canvas.toBlob），但如果要新增一个类似的新格式（如 AVIF），需要修改：

| 文件 | 位置 | 改动内容 |
|------|------|----------|
| [save.js](src/js/modules/file/save.js) | SAVE_TYPES 对象 (L35) | 添加新格式标识和描述 |
| [save.js](src/js/modules/file/save.js) | save_dialog_onchange() | 添加文件大小计算逻辑 |
| [save.js](src/js/modules/file/save.js) | save_action() | 添加导出实现代码 |
| [save.js](src/js/modules/file/save.js) | 质量参数控制 | 决定是否需要显示质量滑块 |

### 6.2 格式分支逻辑耦合度分析

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         当前 save_action 结构                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  if (type == 'PNG') { ... }      ◄── 每个格式一个 if-else 分支               │
│  else if (type == 'JPG') { ... }                                           │
│  else if (type == 'WEBP') { ... }                                          │
│  else if (type == 'GIF') { ... }    ◄── GIF 逻辑最复杂，独立处理             │
│  else if (type == 'TIFF') { ... }   ◄── 依赖外部库                           │
│  else if (type == 'JSON') { ... }   ◄── 完全不同的处理逻辑                   │
│                                                                             │
│  耦合度: 中等偏高                                                            │
│  问题: 新增格式需要修改多处，容易遗漏                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 可抽取的公共流程

建议重构为策略模式：

```javascript
// 建议的格式处理器接口
const FormatHandlers = {
    PNG: {
        extension: '.png',
        supportsQuality: false,
        export: (canvas, filename, quality) => {
            canvas.toBlob(blob => filesaver.saveAs(blob, filename));
        }
    },
    JPG: {
        extension: '.jpg',
        supportsQuality: true,
        export: (canvas, filename, quality) => {
            canvas.toBlob(blob => filesaver.saveAs(blob, filename), 
                "image/jpeg", quality);
        }
    },
    GIF: {
        extension: '.gif',
        supportsQuality: false,
        export: (canvas, filename, quality, delay) => {
            // GIF 特殊处理
        }
    },
    // ... 其他格式
};

// 统一调用
const handler = FormatHandlers[type];
handler.export(canvas, filename, quality, delay);
```

### 6.4 可维护性评分

| 维度 | 评分 | 说明 |
|------|------|------|
| **代码组织** | ⭐⭐⭐☆☆ | 功能集中但分支较多 |
| **扩展性** | ⭐⭐⭐☆☆ | 新增格式需要改多处 |
| **可读性** | ⭐⭐⭐⭐☆ | 流程清晰，注释充分 |
| **错误处理** | ⭐⭐⭐☆☆ | 基本覆盖，但可更完善 |
| **测试友好性** | ⭐⭐☆☆☆ | 与 DOM 和全局状态耦合 |

### 6.5 改进建议

1. **格式处理器注册机制**: 使用插件式注册，各格式独立成模块
2. **统一导出接口**: 抽象 `Exporter` 基类，各格式实现具体导出逻辑
3. **分离 UI 逻辑**: 将对话框逻辑与导出逻辑分离
4. **支持导出配置**: 允许各格式自定义导出参数

---

## 7. 总结

miniPaint 的文件读写模块设计清晰，功能完整：

- **Open 模块**: 支持多种输入方式（文件选择、拖放、URL、粘贴、摄像头），File API 使用规范
- **Save 模块**: 支持丰富的导出格式，对 GIF/TIFF 使用成熟的第三方库
- **Quick 功能**: 基于 localStorage 的快速存取，适合临时备份场景
- **图层集成**: 文件打开后通过 Action 机制无缝集成到图层系统

主要改进空间在于导出格式的分支逻辑可以进一步解耦，采用更灵活的插件化设计。
