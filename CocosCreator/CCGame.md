# CCGame

## 前言
_本文基于 Cocos Creator 2.4.8 撰写_

## 正文

`cc.game` 对象是 `cc.Game` 类的一个实例，`cc.game` 包含了游戏主体信息并负责驱动游戏。

>`cc.game` 对象就是管理引擎生命周期的模块，启动、暂停、重启和帧率控制等操作都需要用到它。
>
>继承了`cc.EventTarget`,可触发事件:`EVENT_HIDE`/`EVENT_SHOW`/`EVENT_RESTART`/`EVENT_GAME_INITED`/`EVENT_ENGINE_INITED`/~~`EVENT_RENDERER_INITED`~~.
>
>记录了渲染器类型`renderType`:`RENDER_TYPE_CANVAS`/`RENDER_TYPE_WEBGL`/`RENDER_TYPE_OPENGL`
>
>管理持久化节点`_persistRootNodes`,在切换场景的时候保留持久化节点,在重启的时候销毁持久化节点

### run

`cc.game.run` 函数内指定了引擎配置和 `onStart` 回调并触发 `cc.game.prepare()` 函数。
```
run: function (config, onStart) {
    // 指定引擎配置
    this._initConfig(config);
    this.onStart = onStart;
    this.prepare(game.onStart && game.onStart.bind(game));
}
```

### prepare
`cc.game.prepare` 函数内主要在项目预览时快速编译项目代码并调用 `_prepareFinished` 函数。
```
prepare(cb) {
    // 已经准备过则跳过
    if (this._prepared) {
        if (cb) cb();
        return;
    }
    // 加载预览项目代码
    this._loadPreviewScript(() => {
        this._prepareFinished(cb);
    });
    }
```
>快速编译<br/>
对于快速编译的细节，可以在项目预览时打开浏览器的开发者工具，在 Sources 栏中搜索（Ctrl + P） `__quick_compile_project__` 即可找到 `__quick_compile_project__.js` 文件。

### _prepareFinished
`cc.game._prepareFinished()` 函数的作用主要为初始化引擎、设置帧率计时器和初始化内建资源（`effect` 资源和 `material` 资源）。

当内建资源加载完成后就会调用 `cc.game._runMainLoop()` 启动引擎主循环。
```
_prepareFinished(cb) {
    // 初始化引擎
    this._initEngine();
    // 设置帧率计时器
    this._setAnimFrame();
    // 初始化内建资源（加载内置的 effect 和 material 资源）
    cc.assetManager.builtins.init(() => {
        // 打印引擎版本到控制台
        console.log('Cocos Creator v' + cc.ENGINE_VERSION);
        this._prepared = true;
        // 启动 mainLoop
        this._runMainLoop();
        // 发射 ‘game_inited’ 事件（代表引擎已初始化完成）
        this.emit(this.EVENT_GAME_INITED);
        // 调用 main.js 中定义的 onStart 函数
        if (cb) cb();
    });
}
```

### _initRenderer
```

    if (CC_JSB || CC_RUNTIME) {
        this.container = localContainer = document.createElement("DIV");
        this.frame = localContainer.parentNode === document.body ? document.documentElement : localContainer.parentNode;
        localCanvas = window.__canvas;
        this.canvas = localCanvas;
    }
    else {
        // 处理Canvas,DIV
        var element = (el instanceof HTMLElement) ? el : (document.querySelector(el) || document.querySelector('#' + el));
        ...

        function addClass (element, name) {
            ...
        }
        addClass(localCanvas, "gameCanvas");
        localCanvas.setAttribute("width", width || 480);
        localCanvas.setAttribute("height", height || 320);
        localCanvas.setAttribute("tabindex", 99);
    }
    // 确认渲染 webGL/Canvas
    this._determineRenderType();
    // WebGL context created successfully
    if (this.renderType === this.RENDER_TYPE_WEBGL) {
        var opts = {
            'stencil': true,
            // MSAA is causing serious performance dropdown on some browsers.
            'antialias': cc.macro.ENABLE_WEBGL_ANTIALIAS,
            'alpha': cc.macro.ENABLE_TRANSPARENT_CANVAS
        };
        renderer.initWebGL(localCanvas, opts);
        this._renderContext = renderer.device._gl;

        // Enable dynamic atlas manager by default
        if (!cc.macro.CLEANUP_IMAGE_CACHE && dynamicAtlasManager) {
            dynamicAtlasManager.enabled = true;
        }
    }
    if (!this._renderContext) {
        this.renderType = this.RENDER_TYPE_CANVAS;
        // Could be ignored by module settings
        renderer.initCanvas(localCanvas);
        this._renderContext = renderer.device._ctx;
    }
```

### _initEvents
兼容性封装监听Hide/Show事件,触发时派发事件`cc.game.EVENT_HIDE`/`cc.game.EVENT_SHOW`

### _setAnimFrame

对于 `_prepareFinished` 函数内调用的 `_setAnimFrame` 函数这里必须提一下。

 函数内部对不同的游戏帧率做了适配。

另外还对 `window.requestAnimationFrame` 函数做了兼容性封装，用于兼容不同的浏览器环境，具体的我们下面再说。

这里就不贴 `_setAnimFrame` 函数的代码了，有需要的小伙伴可自行查阅。

### _runMainLoop
`cc.game._runMainLoop` 函数的名字取得很简单直接，摊牌了它就是用来运行 `mainLoop` 函数的。
```
_runMainLoop: function () {
    if (CC_EDITOR) return;
    if (!this._prepared) return;
    // 定义局部变量
    var self = this, callback, config = self.config,
    director = cc.director,
    skip = true, frameRate = config.frameRate;
    // 展示或隐藏性能统计
    debug.setDisplayStats(config.showFPS);
    // 设置帧回调
    callback = function (now) {
    if (!self._paused) {
        // 循环调用回调
        self._intervalId = window.requestAnimFrame(callback);
        if (!CC_JSB && !CC_RUNTIME && frameRate === 30) {
            if (skip = !skip) return;
        }
        // 调用 mainLoop
        director.mainLoop(now);
    }
    };
    // 将在下一帧开始循环回调
    self._intervalId = window.requestAnimFrame(callback);
    self._paused = false;
}
```

### requestAnimFrame
`window.requestAnimFrame` 函数就是我们上面说到的 `_setAnimFrame` 函数内部对于 `window.requestAnimationFrame` 函数的兼容性封装。

对前端不太熟悉的小伙伴可能会有疑问，`window.requestAnimationFrame` 又是啥，是用来干嘛的，又是如何运行的？

### requestAnimationFrame
简单来说，`window.requestAnimationFrame` 函数用于向浏览器请求进行一次重绘（repaint），并在重绘之前调用指定的回调函数。

`window.requestAnimationFrame` 函数接收一个回调作为参数并返回一个整数作为唯一标识，浏览器将会在下一个重绘之前执行这个回调；并且执行回调时会传入一个参数，参数的值与 `performance.now()` 返回的值相等。

>Performance.now()<br/>
performance.now() 的返回值可以简单理解为浏览器窗口的运行时长，即从打开窗口到当前时刻的时间差。

[MDN] [Performance.now()](https://developer.mozilla.org/zh-CN/docs/Web/API/Performance/now): 
回调函数的执行次数通常与浏览器屏幕刷新次数相匹配，也就是说，对于刷新率为 60Hz 的显示器，浏览器会在一秒内执行 60 次回调函数。

对于 `window.requestAnimationFrame` 函数的说明到此为止，如果想要了解更多信息请自行检索。

[MDN] [window.requestAnimationFrame](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame)