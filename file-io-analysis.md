# miniPaint 文件读写模块分析报告

## 1. 模块概览：modules/file/ 目录结构

`src/js/modules/file/` 目录包含 6 个文件处理模块：

| 文件 | 功能入口 | 主要职责 |
|------|----------|----------|
| [new.js](src/js/modules/file/new.js) | 创建新文件 | 弹出对话框设置画布尺寸、分辨率、透明度，初始化空白画布 |
| [open.js](src/js/modules/file/open.js) | 打开文件 | 支持本地文件选择、拖拽、URL、Data URL、摄像头、目录等多种方式打开图片和 JSON 工程文件 |
| [save.js](src/js/modules/file/save.js) | 保存/导出文件 | 支持 PNG、JPG、WEBP、GIF、TIFF、BMP、JSON 等多种格式导出 |
| [print.js](src/js/modules/file/print.js) | 打印 | 简单调用 `window.print()` 实现浏览器打印功能 |
| [quickload.js](src/js/modules/file/quickload.js) | 快速加载 | 从 localStorage 恢复之前快速保存的状态（F10 快捷键） |
| [quicksave.js](src/js/modules/file/quicksave.js) | 快速保存 | 将当前工程状态保存到 localStorage（F9 快捷键） |

---

## 2. open.js 深度分析：File API 读取文件到画布的完整链路

### 2.1 文件读取入口

open.js 提供了多种文件打开方式：

```javascript
// 主要入口方法
open_file()      // 点击打开本地文件（创建 input[type=file]）
open_handler(e)  // 统一处理文件选择/拖拽事件
open_url()       // 通过 URL 打开图片
open_data_url()  // 通过 Data URL 打开
open_webcam()    // 从摄像头捕获
open_dir()       // 打开目录
```

### 2.2 浏览器 File API 读取流程

以 `open_file()` → `open_handler()` 为例，完整链路如下：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. 用户交互触发                                                               │
│    open_file() 创建 <input type="file" multiple> 并触发 click               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. 文件选择完成                                                               │
│    change 事件触发 → open_handler(e) 接收 FileList                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. FileReader 读取文件内容                                                    │
│    var FR = new FileReader();                                               │
│    FR.readAsDataURL(f);  // 读取为 base64 Data URL                          │
│    或 FR.readAsText(f);  // JSON 文件用文本方式读取                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. 文件类型判断与处理                                                         │
│    if (f.type.match('image.*')) → 图片处理                                   │
│    else if (f.name.match('.json')) → JSON 工程文件处理                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. 创建图层对象                                                               │
│    var new_layer = {                                                        │
│        name: this.file.name,                                                │
│        type: 'image',                                                       │
│        data: event.target.result,  // base64 Data URL                       │
│        order: order,                                                        │
│        _exif: this.extract_exif(this.file)                                  │
│    };                                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. 执行 Action 插入图层                                                      │
│    app.State.do_action(                                                     │
│        new app.Actions.Bundle_action('open_image', 'Open Image', [          │
│            new app.Actions.Insert_layer_action(new_layer)                   │
│        ])                                                                   │
│    );                                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 关键代码片段

```javascript
// open.js 第 316-340 行
var FR = new FileReader();
FR.file = files[i];

FR.onload = function (event) {
    if (this.file.type.match('image.*')) {
        var order = auto_increment + order_map[this.file.name];
        var new_layer = {
            name: this.file.name,
            type: 'image',
            data: event.target.result,  // Data URL (base64)
            order: order,
            _exif: _this.extract_exif(this.file)
        };
        app.State.do_action(
            new app.Actions.Bundle_action('open_image', 'Open Image', [
                new app.Actions.Insert_layer_action(new_layer)
            ])
        );
    }
};

// 根据文件类型选择读取方式
if (f.type == "text/plain")
    FR.readAsText(f);
else if (f.name.match('.json'))
    FR.readAsText(f);
else
    FR.readAsDataURL(f);  // 图片文件读取为 Data URL
```

---

## 3. save.js 深度分析：不同导出格式的处理方式

### 3.1 支持的导出格式

```javascript
this.SAVE_TYPES = {
    PNG: "Portable Network Graphics",
    JPG: "JPG/JPEG Format",
    JSON: "Full layers data",
    WEBP: "Weppy File Format",
    GIF: "Graphics Interchange Format",
    BMP: "Windows Bitmap",
    TIFF: "Tag Image File Format",
};
```

### 3.2 位图格式处理：canvas.toBlob() vs toDataURL()

**PNG 和 JPG 等位图格式统一使用 `canvas.toBlob()` + `file-saver` 库：**

```javascript
// PNG 导出（第 519-528 行）
if (type == 'PNG') {
    if (this.Helper.strpos(fname, '.png') == false)
        fname = fname + ".png";
    
    canvas.toBlob(function (blob) {
        filesaver.saveAs(blob, fname);
    });
}

// JPG 导出（第 529-537 行）
else if (type == 'JPG') {
    if (this.Helper.strpos(fname, '.jpg') == false)
        fname = fname + ".jpg";
    
    canvas.toBlob(function (blob) {
        filesaver.saveAs(blob, fname);
    }, "image/jpeg", quality);  // 传入 quality 参数
}
```

**为什么选择 `toBlob()` 而非 `toDataURL()`？**

1. **内存效率**：Blob 是二进制数据，Data URL 是 base64 字符串（体积增大约 33%）
2. **性能优势**：Blob 可以直接传给 FileSaver，无需再转换
3. **大文件处理**：对于大尺寸图片，Blob 方式内存占用更低

代码中也保留了 `toDataURL()` 的注释示例：
```javascript
//simple save example (已注释)
//var link = document.createElement('a');
//link.download = fname;
//link.href = canvas.toDataURL();
//link.click();
```

### 3.3 GIF 导出：依赖 gif.js.optimized 库

GIF 是动图格式，需要逐帧编码，使用第三方库 `gif.js.optimized`：

```javascript
// GIF 导出（第 575-604 行）
else if (type == 'GIF') {
    var cores = navigator.hardwareConcurrency || 4;
    var gif_settings = {
        workers: cores,                    // 使用多线程加速
        quality: 10,                       // 1-30，越低质量越好
        repeat: 0,                         // 循环播放
        width: config.WIDTH,
        height: config.HEIGHT,
        dither: 'FloydSteinberg-serpentine', // 抖动算法
        workerScript: './src/js/libs/gifjs/gif.worker.js',
    };
    if (config.TRANSPARENCY == true) {
        gif_settings.transparent = 'rgba(0,0,0,0)';
    }
    var gif = new GIF(gif_settings);

    // 逐帧添加图层
    for (var i = 0; i < config.layers.length; i++) {
        if (config.layers[i].visible == false)
            continue;
        
        ctx.clearRect(0, 0, config.WIDTH, config.HEIGHT);
        if (config.TRANSPARENCY == false) {
            this.fillCanvasBackground(ctx, '#ffffff');
        }
        this.Base_layers.convert_layers_to_canvas(ctx, config.layers[i].id, false);
        
        gif.addFrame(ctx, {copy: true, delay: delay});
    }
    
    gif.render();
    gif.on('finished', function (blob) {
        filesaver.saveAs(blob, fname);
    });
}
```

**GIF 导出特点：**
- 每个可见图层作为一帧
- 使用 Web Worker 多线程加速编码
- 支持 Floyd-Steinberg 抖动算法
- 支持透明背景

### 3.4 TIFF 导出：使用 CanvasToTIFF 库

TIFF 格式浏览器原生不支持，使用自定义库 `CanvasToTIFF`：

```javascript
// TIFF 导出（第 564-572 行）
else if (type == 'TIFF') {
    if (this.Helper.strpos(fname, '.tiff') == false)
        fname = fname + ".tiff";
    var data_header = "image/tiff";

    CanvasToTIFF.toBlob(canvas, function(blob) {
        filesaver.saveAs(blob, fname);
    }, data_header);
}
```

`CanvasToTIFF` 库（位于 [src/js/libs/canvastotiff.js](src/js/libs/canvastotiff.js)）实现了：
- 将 Canvas 像素数据编码为 TIFF 格式
- 支持 32-bit RGBA
- 支持设置 DPI
- 支持大端/小端字节序

### 3.5 JSON 工程文件格式

JSON 格式保存完整的工程数据，支持无损编辑：

```javascript
// export_as_json() 方法（第 636-687 行）
export_as_json() {
    var export_data = {};
    
    // 1. 元信息
    export_data.info = {
        width: config.WIDTH,
        height: config.HEIGHT,
        about: 'Image data with multi-layers...',
        date: today,
        version: VERSION,
        layer_active: config.layer.id,
        guides: config.guides,
    };

    // 2. 用户字体
    export_data.user_fonts = config.user_fonts;

    // 3. 图层列表
    export_data.layers = [];
    for (var i in config.layers) {
        var layer = {};
        for (var j in config.layers[i]) {
            if (j[0] == '_' || j == 'link_canvas') {
                continue;  // 跳过私有数据
            }
            layer[j] = config.layers[i][j];
        }
        export_data.layers.push(layer);
    }

    // 4. 图层图像数据（仅 image 类型图层）
    export_data.data = [];
    for (var i in config.layers) {
        if (config.layers[i].type != 'image')
            continue;
        
        var canvas = document.createElement('canvas');
        canvas.width = config.layers[i].width_original;
        canvas.height = config.layers[i].height_original;
        canvas.getContext('2d').drawImage(config.layers[i].link, 0, 0);
        
        var data_tmp = canvas.toDataURL("image/png");
        export_data.data.push({
            id: config.layers[i].id,
            data: data_tmp,
        });
    }

    return JSON.stringify(export_data, null, "\t");
}
```

**JSON 工程文件结构：**
```json
{
    "info": {
        "width": 800,
        "height": 600,
        "about": "...",
        "date": "<导出时的日期，格式 YYYY-MM-DD>",
        "version": "<当前 miniPaint 版本号>",
        "layer_active": 1,
        "guides": []
    },
    "user_fonts": {},
    "layers": [
        {
            "id": 1,
            "name": "Layer 1",
            "type": "image",
            "x": 0,
            "y": 0,
            "width": 800,
            "height": 600,
            "opacity": 100,
            "visible": true,
            "rotate": 0,
            "composition": "source-over",
            "filters": []
        }
    ],
    "data": [
        {
            "id": 1,
            "data": "data:image/png;base64,..."
        }
    ]
}
```

---

## 4. quickload 与 quicksave 分析

### 4.1 设计目的

这两个功能是为**快速工作流**设计的场景：

- **快速迭代**：用户在编辑过程中需要频繁保存中间状态
- **临时存储**：不需要生成文件，只是暂存当前工作状态
- **快捷键操作**：F9 保存、F10 加载，无需对话框交互

### 4.2 数据存储位置

**使用 `localStorage` 存储：**

```javascript
// quicksave.js
quicksave() {
    var data_json = this.File_save.export_as_json();
    if (data_json.length > 5000000) {
        alertify.error('Sorry, image is too big, max 5 MB.');
        return false;
    }
    localStorage.setItem('quicksave_data', data_json);
}

// quickload.js
quickload() {
    var json = localStorage.getItem('quicksave_data');
    if (json == '' || json == null) {
        return false;
    }
    this.File_open.load_json(json);
}
```

### 4.3 与正常 save/open 的对比

| 特性 | quicksave/load | 正常 save/open |
|------|----------------|----------------|
| 存储位置 | localStorage | 本地文件系统 |
| 文件大小限制 | 5 MB | 无限制 |
| 用户交互 | 无（快捷键触发） | 有（对话框选择格式/位置） |
| 格式选择 | 仅 JSON | PNG/JPG/GIF/TIFF/JSON 等 |
| 持久性 | 浏览器清理后丢失 | 永久保存 |
| 跨设备 | 否 | 是（文件可传输） |
| 版本历史 | 仅最新一份 | 可保存多个版本 |

**quicksave/load 少做的事情：**
1. 无文件名输入对话框
2. 无格式选择
3. 无文件大小预览
4. 无文件下载过程
5. 直接序列化/反序列化 JSON，无需用户确认

---

## 5. "打开图片 → 新图层出现在画布" 完整调用链

### 5.1 调用链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 用户操作：点击"打开文件"或拖拽图片到浏览器                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ File_open_class.open_file() / open_handler(e)                               │
│ [src/js/modules/file/open.js]                                               │
│ - 创建 FileReader 读取文件                                                   │
│ - 将文件读取为 Data URL (base64)                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 创建 new_layer 对象                                                          │
│ {                                                                           │
│   name: filename,                                                           │
│   type: 'image',                                                            │
│   data: "data:image/png;base64,..."                                         │
│ }                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ app.State.do_action()                                                       │
│ [src/js/core/state.js] (状态管理器)                                          │
│ - 管理撤销/重做历史                                                          │
│ - 执行 Action                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ Bundle_action.do()                                                          │
│ [src/js/actions/bundle.js]                                                  │
│ - 组合多个 Action 原子操作                                                   │
│ - 按顺序执行子 Action                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ Insert_layer_action.do()                                                    │
│ [src/js/actions/insert-layer.js]                                            │
│ - 创建 Image 对象加载图片                                                    │
│ - 设置图层属性 (width, height, link 等)                                      │
│ - 将图层添加到 config.layers 数组                                            │
│ - 触发 Autoresize_canvas_action (自动调整画布大小)                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ Autoresize_canvas_action.do()                                               │
│ [src/js/actions/autoresize-canvas.js]                                       │
│ - 根据图片尺寸调整画布大小                                                    │
│ - 更新 config.WIDTH / config.HEIGHT                                         │
│ - 调用 app.GUI.prepare_canvas()                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ Base_layers_class.render()                                                  │
│ [src/js/core/base-layers.js]                                                │
│ - 设置 config.need_render = true                                            │
│ - 使用 requestAnimationFrame 循环渲染                                        │
│ - 调用 render_objects() 绘制所有图层                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ render_object()                                                             │
│ [src/js/core/base-layers.js]                                                │
│ - 对于 type='image' 的图层：                                                 │
│   ctx.drawImage(object.link, x, y, width, height)                           │
│ - 对于其他类型图层：调用对应的 render_function                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ GUI_layers.render_layers()                                                  │
│ [src/js/core/gui/gui-layers.js]                                             │
│ - 更新图层面板 UI                                                           │
│ - 显示新图层列表项                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键代码流程

**Insert_layer_action.do() 核心逻辑：**

```javascript
// src/js/actions/insert-layer.js
async do() {
    // 1. 构建图层默认数据
    const layer = {
        id: app.Layers.auto_increment,
        name: 'Layer #' + app.Layers.auto_increment,
        type: null,
        link: null,
        x: 0, y: 0,
        width: null, height: null,
        visible: true,
        opacity: 100,
        // ... 其他属性
    };

    // 2. 合并用户传入的设置
    for (let i in this.settings) {
        layer[i] = this.settings[i];
    }

    // 3. 处理图片类型图层
    if (layer.type == 'image') {
        if (layer.link == null && typeof layer.data == 'string') {
            // 从 Data URL 加载图片
            layer.link = new Image();
            layer.link.onload = () => {
                layer.width = layer.link.width;
                layer.height = layer.link.height;
                config.need_render = true;
            };
            layer.link.src = layer.data;
        }
    }

    // 4. 添加到全局图层列表
    config.layers.push(layer);
    config.layer = app.Layers.get_layer(layer.id);
    app.Layers.auto_increment++;

    // 5. 触发渲染
    app.Layers.render();
    app.GUI.GUI_layers.render_layers();
}
```

### 5.3 跨模块依赖关系

```
modules/file/open.js
    ├── core/base-layers.js (图层管理)
    ├── actions/insert-layer.js (插入图层 Action)
    ├── actions/autoresize-canvas.js (调整画布 Action)
    ├── actions/bundle.js (Action 组合器)
    ├── core/state.js (状态管理)
    └── libs/helpers.js (工具函数)
```

---

## 6. 可维护性评估

### 6.1 新增 BMP 导出格式需要改动的地方

**好消息：BMP 格式已经在代码中实现！** 以下是分析其实现方式：

```javascript
// save.js 第 555-563 行
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

**如果要新增一个全新格式（如 AVIF），需要改动的地方：**

| 序号 | 文件 | 改动内容 |
|------|------|----------|
| 1 | save.js | 在 `SAVE_TYPES` 对象中添加新格式定义 |
| 2 | save.js | 在 `save_dialog_onchange()` 中添加格式特定 UI 逻辑 |
| 3 | save.js | 在 `save_action()` 中添加导出逻辑分支 |
| 4 | save.js | 如需第三方库，添加 import 语句 |
| 5 | package.json | 如需第三方库，添加依赖 |
| 6 | open.js | 如需特殊打开逻辑，添加处理分支 |

### 6.2 格式分支逻辑耦合度分析

**当前代码的格式分支使用 if-else 链：**

```javascript
if (type == 'PNG') {
    // PNG 处理
}
else if (type == 'JPG') {
    // JPG 处理
}
else if (type == 'WEBP') {
    // WEBP 处理
}
// ... 更多格式
```

**问题：**
- 分支逻辑分散在 `save_dialog_onchange()` 和 `save_action()` 两个方法中
- 新增格式需要同时修改多处
- 格式特定的 UI 逻辑（quality、delay）硬编码

### 6.3 可抽取的公共流程

**建议重构方案：策略模式**

```javascript
// 建议的格式处理器接口
const FormatHandlers = {
    PNG: {
        extension: 'png',
        mimeType: 'image/png',
        supportsQuality: false,
        export(canvas, options) {
            return new Promise(resolve => {
                canvas.toBlob(resolve, this.mimeType);
            });
        }
    },
    JPG: {
        extension: 'jpg',
        mimeType: 'image/jpeg',
        supportsQuality: true,
        export(canvas, options) {
            return new Promise(resolve => {
                canvas.toBlob(resolve, this.mimeType, options.quality);
            });
        }
    },
    GIF: {
        extension: 'gif',
        mimeType: 'image/gif',
        supportsDelay: true,
        async export(canvas, options) {
            // GIF 特殊处理
        }
    }
    // ... 其他格式
};

// 统一导出方法
async export(canvas, type, options) {
    const handler = FormatHandlers[type];
    const blob = await handler.export(canvas, options);
    filesaver.saveAs(blob, options.filename);
}
```

**公共流程可抽取为：**

1. **文件名处理**：统一添加扩展名逻辑
2. **Canvas 准备**：统一创建临时 Canvas、处理透明背景
3. **格式支持检测**：`check_format_support()` 已存在
4. **文件大小计算**：`update_file_size()` 已存在
5. **Blob 保存**：统一使用 file-saver

### 6.4 代码质量评估

| 维度 | 评分 | 说明 |
|------|------|------|
| 可读性 | ★★★★☆ | 代码结构清晰，注释充分 |
| 可扩展性 | ★★★☆☆ | if-else 分支较多，可改用策略模式 |
| 可测试性 | ★★☆☆☆ | 缺少单元测试，依赖 DOM API |
| 错误处理 | ★★★☆☆ | 有基本错误提示，但缺少详细错误信息 |
| 文档完整性 | ★★★★☆ | JSDoc 注释较完善 |

---

## 7. 总结

### 7.1 架构特点

1. **Action 模式**：所有状态变更通过 Action 类封装，支持撤销/重做
2. **单例模式**：核心类使用单例，全局访问
3. **事件驱动**：文件选择、拖拽等通过事件触发
4. **模块化**：文件操作、图层管理、状态管理分离

### 7.2 技术选型

| 功能 | 技术方案 |
|------|----------|
| 文件读取 | FileReader API |
| 图片编码 | Canvas toBlob() / toDataURL() |
| 文件保存 | file-saver 库 |
| GIF 编码 | gif.js.optimized 库 |
| TIFF 编码 | 自定义 CanvasToTIFF 库 |
| 临时存储 | localStorage |

### 7.3 改进建议

1. **格式处理器重构**：使用策略模式替代 if-else 链
2. **添加单元测试**：特别是格式转换逻辑
3. **错误处理增强**：提供更详细的错误信息和恢复建议
4. **大文件支持**：考虑使用 IndexedDB 替代 localStorage 存储快速保存数据
5. **Worker 线程**：将耗时的编码操作移到 Web Worker
