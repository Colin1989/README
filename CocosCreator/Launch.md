# 启动流程

## 前言
_本文基于 Cocos Creator 2.4.8 撰写_

## Web
web平台的程序启动入口在`index.html`
在默认的 `index.html` 文件中，定义了游戏启动页面的布局，加载了 `main.js` 文件，并且还有一段立即执行的代码。

```
// 加载引擎脚本
loadScript(debug ? 'cocos2d-js.js' : 'cocos2d-js-min.js', function () {
  // 是否开启了物理系统？
  if (CC_PHYSICS_BUILTIN || CC_PHYSICS_CANNON) {
    // 加载物理系统脚本并启动引擎
    loadScript(debug ? 'physics.js' : 'physics-min.js', window.boot);
  } else {
    // 启动引擎
    window.boot();
  }
});
```
上面这段代码主要用于加载引擎脚本和物理系统脚本，脚本加载完成之后就会调用 `main.js` 中定义的 `window.boot` 函数。
>原生平台<br/>
对于原生平台，会在 `{项目目录}build\jsb-link\frameworks\runtime-src\Classes\AppDelegate.cpp` 文件的 `applicationDidFinishLaunching` 函数中加载 `main.js` 文件。


## Native
Native平台的程序启动入口在`AppDelegate.cpp`
```
bool AppDelegate::applicationDidFinishLaunching()
{
    se::ScriptEngine *se = se::ScriptEngine::getInstance();
    
    // 解密密钥Key
    jsb_set_xxtea_key("de1fe079-c7dc-4d");
    // 初始化js binding中的文件操作委托函数
    jsb_init_file_operation_delegate();
    
    // 开启http debug 调试
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    // Enable debugger here
    jsb_enable_debugger("0.0.0.0", 6086, false);
#endif
    
    // 异常捕捉
    se->setExceptionCallback([](const char *location, const char *message, const char *stack) {
        // Send exception information to server like Tencent Bugly.
        cocos2d::log("\nUncaught Exception:\n - location :  %s\n - msg : %s\n - detail : \n      %s\n", location, message, stack);
    });
    
    jsb_register_all_modules();
    
    se->start();
    
    se::AutoHandleScope hs;
    // 加载启动脚本
    jsb_run_script("jsb-adapter/jsb-builtin.js");
    jsb_run_script("main.js");
    
    se->addAfterCleanupHook([]() {
        JSBClassType::destroy();
    });
    
    return true;
}
```

## main.js

### boot
>对于不同平台 `main.js` 的内容也有些许差异，这里我们忽略差异部分，只关注其中关键的共同行为。
关于 `main.js` 文件的内容基本上就是定义了 `window.boot` 函数。

对于非 Web 平台，会在定义完之后直接就调用 `window.boot` 函数，所以 `main.js` 就是他们的起点。

而 `window.boot` 函数内部有以下关键行为：

1. 定义 `onStart` 函数：主要用于加载启动场景
2. `cc.assetManager.init(...)`：初始化 AssetManager
3. `cc.assetManager.loadScript(...)`：加载 src 目录下的插件脚本
4. `cc.assetManager.loadBundle(...)`：加载项目中的 `INTERNAL` , `RESOURCES`
5. `cc.game.run(...)`：启动引擎
6. `bundle.loadScene` 加载首场景
   