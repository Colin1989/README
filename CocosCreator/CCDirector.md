# CCDirector

## 前言
_本文基于 Cocos Creator 2.4.8 撰写_

## 正文

`cc.director` 一个管理你的游戏的逻辑流程的单例对象。由于 `cc.director` 是一个单例，你不需要调用任何构造函数或创建函数，使用它的标准方法是通过调用：
- cc.director.methodName();

 它创建和处理主窗口并且管理什么时候执行场景。
 
` cc.director` 还负责：
- 初始化 `OpenGL` 环境。
- 设置`OpenGL`像素格式。(默认是 `RGB565`)
- 设置`OpenGL`缓冲区深度 (默认是 `0-bit`)
- 设置空白场景的颜色 (默认是 黑色)
- 设置投影 (默认是 `3D`)
- 设置方向 (默认是 `Portrait`)
  
`cc.director` 设置了 `OpenGL` 默认环境 
- `GL_TEXTURE_2D`   启用。
- `GL_VERTEX_ARRAY` 启用。
- `GL_COLOR_ARRAY`  启用。
- `GL_TEXTURE_COORD_ARRAY` 启用。

`cc.director` 负责触发和派发事件:
- `EVENT_BEFORE_SCENE_LAUNCH` 运行新场景之前所触发的事件。
- `EVENT_AFTER_SCENE_LAUNCH` 运行新场景之后所触发的事件。
- `EVENT_BEFORE_UPDATE` 每个帧的开始时所触发的事件。
- `EVENT_AFTER_UPDATE` 将在引擎和组件 “update” 逻辑之后所触发的事件。
- `EVENT_BEFORE_DRAW` 渲染过程之前所触发的事件。
- `EVENT_AFTER_DRAW` 渲染过程之后所触发的事件。
- `EVENT_BEFORE_PHYSICS` 物理过程之前所触发的事件。
  
`cc.director` 也同步定时器与显示器的刷新速率,坐标点与webGl/UI的转换。

特点和局限性: 
  - 将计时器 & 渲染与显示器的刷新频率同步。
  - 只支持动画的间隔 1/60 1/30 & 1/15。

  
## 方法


### sharedInit
```
sharedInit: function () {
    // 组件调度管理
    this._compScheduler = new ComponentScheduler();
    // 激活和反激活节点以及节点身上的组件
    this._nodeActivator = new NodeActivator();

    // 启用事件管理器
    if (eventManager) {
        eventManager.setEnabled(true);
    }

    // 动作管理器 Action / Tween,
    if (cc.AnimationManager) {
        this._animationManager = new cc.AnimationManager();
        this._scheduler.scheduleUpdate(this._animationManager, Scheduler.PRIORITY_SYSTEM, false);
    }
    else {
        this._animationManager = null;
    }

    // 碰撞管理器
    if (cc.CollisionManager) {
        this._collisionManager = new cc.CollisionManager();
        this._scheduler.scheduleUpdate(this._collisionManager, Scheduler.PRIORITY_SYSTEM, false);
    }
    else {
        this._collisionManager = null;
    }

    // 物理管理器
    if (cc.PhysicsManager) {
        this._physicsManager = new cc.PhysicsManager();
        this._scheduler.scheduleUpdate(this._physicsManager, Scheduler.PRIORITY_SYSTEM, false);
    }
    else {
        this._physicsManager = null;
    }

    // 3d物理管理器
    if (cc.Physics3DManager && (CC_PHYSICS_BUILTIN || CC_PHYSICS_CANNON)) {
        this._physics3DManager = new cc.Physics3DManager();
        this._scheduler.scheduleUpdate(this._physics3DManager, Scheduler.PRIORITY_SYSTEM, false);
    } else {
        this._physics3DManager = null;
    }

    // WidgetManager
    if (cc._widgetManager) {
        cc._widgetManager.init(this);
    }
},
```

### mainLoop
`cc.director.mainLoop` 函数可能是引擎中最关键的逻辑之一了，包含的内容很多也很关键。
```
mainLoop: function (now) {
    // 计算“全局”增量时间（DeltaTime）
    // 也就是距离上一次调用 mainLoop 的时间间隔
    this.calculateDeltaTime(now);
    // 游戏没有暂停则进行更新
    if (!this._paused) {
        // 发射 'director_before_update' 事件
        this.emit(cc.Director.EVENT_BEFORE_UPDATE);
        // 调用新增的组件（已启用）的 start 函数
        this._compScheduler.startPhase();
        // 调用所有组件（已启用）的 update 函数
        this._compScheduler.updatePhase(this._deltaTime);
        // 更新调度器（cc.Scheduler）
        this._scheduler.update(this._deltaTime);
        // 调用所有组件（已启用）的 lateUpdate 函数
        this._compScheduler.lateUpdatePhase(this._deltaTime);
        // 发射 'director_after_update' 事件
        this.emit(cc.Director.EVENT_AFTER_UPDATE);
        // 销毁最近被移除的实体（节点）
        Obj._deferredDestroy();
    }
    // 发射 'director_before_draw' 事件
    this.emit(cc.Director.EVENT_BEFORE_DRAW);
    // 渲染游戏场景
    renderer.render(this._scene, this._deltaTime);
    // 发射 'director_after_draw' 事件
    this.emit(cc.Director.EVENT_AFTER_DRAW);
    // 更新事件管理器的事件监听（cc.eventManager 已被废弃）
    eventManager.frameUpdateListeners();
    // 累加游戏运行的总帧数
    this._totalFrames++;
}
``` 

`cc.director` 对象中的 `_compScheduler` 属性是 `ComponentScheduler` 类的实例。

**ComponentScheduler 类是用来集中调度（管理）游戏场景中所有组件（cc.Component）的生命周期的。至于生命周期中的 `onLoad` 和 `onEnable` 是由 [NodeActivator](./NodeActivator.md) 类的来管理的：**


## Scheduler
`cc.director` 对象的 `_scheduler` 属性是 `cc.Scheduler` 类的一个实例。

`cc.Scheduler` 是负责触发回调函数的类。