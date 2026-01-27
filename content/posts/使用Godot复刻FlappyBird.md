---
title: "使用Godot复刻FlappyBird"
date: 2026-01-20T11:09:53+08:00
lastmod: 2026-01-20T11:09:53+08:00
draft: false
tags: [Godot，GameDev]
categories: [Godot]
summary: "学习使用Godot引擎，通过复刻Flappy Bird游戏快速上手"

# PaperMod主题配置
showToc: false
tocOpen: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true

# 封面图片（可选）
cover:
    image: "" # 图片路径/URL
    alt: "" # 替代文本
    caption: "" # 封面下的文字说明
    relative: false # 使用页面捆绑包时设为true
    hidden: true # 仅在当前单页隐藏
---
## 为什么学习Godot

**免费，开源，未来可期**

### 项目需求匹配度

站在个人项目的角度。完善的2d开发工具链，发布多平台一条龙，一体化轻量的特点都是我决定尝试的重要考量。借助AI也可以极大地控制学习成本。

### 其他

可能与自己的生活经历有关，或者被称为应试教育受害者。开发中会去追求最优解，对于有缺陷，不够优雅的解决方案会心存芥蒂。这在工作上表现出来的负面影响有限，或者说有一定的积极作用。毕竟在现实的高压强度下，精神洁癖是可以妥协的。但是在个人项目中就会出现频繁造轮子，过分追求架构设计与细节，虎头蛇尾，最后无疾而终。最主要的原因还是自己技术能力有限却又眼高手低，技术上的不自信也会放大这种做事心态。在使用新的技术时就可以减少这种困扰。在不熟悉的领域，更多地会去追求实现而不会追求如何实现。其次，接触不同事物，以不同的视角看待问题有助于摆脱盲目与局限。

我相信通过对Godot的学习能加深我对游戏开发的理解。

## Godot实战

### 游戏需求

- 创建小鸟与关卡(背景，管道)
- 游戏流程控制，游戏开始，游戏结束判定，游戏重开
- 计分与UI

### 技术细节

1. 创建Sprite角色，添加序列帧动画
2. 通过物理引擎控制角色运动，给小鸟添加刚体，应用重力
3. 添加脚本，在点击或者按键输入时给他一个y方向的速度
4. 添加碰撞检测
5. 创建管道，添加脚本实现向左的持续位移
6. 创建一个管道生成控制器，使用定时器创建管道
7. `Game Manager`，我们需要管理这些游戏对象并控制游戏流程
8. 创建UI显示数据

 能看到实现思路和在其他引擎中没什么不同，当然这里还只是一个较为简陋的原型。接下来会讲讲在具体场景中上有哪些不一样的地方：

#### 获取节点(对象与组件)

在Game Manager中控制管道Spawner。在unity中，我们可以拖拽可序列化对象达到目的。Godot中也有类似的方式，通过`@export`将引用暴露在编辑器中实现拖拽引用。

此外，除了常规的路径查找，Godot引入了`%`访问，可以对子节点选择`Access as Unique Name`(作为唯一名称访问)

``` GDScript
extends Node2D
func update_score():
    # 无论 ScoreLabel 藏得多深，直接用 % 获取，不依赖手写路径
    %ScoreLabel.text = "100"
```

同类对象组的操作。

``` GDScript
func game_over():
    # 呼叫 "pipes" 组里的所有节点，执行它们的 "stop_moving" 方法
    get_tree().call_group("pipes"， "stop_moving")
```

看上去和unity为对象添加标签然后根据标签查找对象组比较像。实际上`Godot` `Group`的功能更加强大，除了有数量管理上的优势，一个 `Node` 可以属于 无数个 `Group`.相比于Unity先查找，再遍历的低效方式

``` c#
void KillAllEnemies() {
    // 1. 查找所有带标签的对象 (性能消耗大)
    GameObject[] enemies = GameObject.FindGameObjectsWithTag("Enemy");
    // 2. 遍历并获取组件
    foreach (GameObject enemy in enemies) {
        EnemyScript script = enemy.GetComponent<EnemyScript>();
        if (script != null) {
            script.Die();
        }
    }
}
```

Godot的`call_group`提供了一个 `广播 (Broadcast)`机制。

#### 节点间的通信

如何在我的`Bird`脚本中处理子节点碰撞检测的事件。

![signal](/img/img_2026-01-21.00.09.20.png)

节点信号栏中可以看到节点有哪些信号，其中`body_entered`就是我们所需要的.Godot的`Signal`可以像`Unity Event`一样支持拖拽连接，也可以在代码里像`c# Event`一样高性能连接。更方便的是，它会跟随节点的生命周期在销毁时自动解绑。

使用信号，我们还可以为`PipeSpawner`添加一个`Timer`节点，将`Timer` `_on_timer_timeout`信号与我们的定时创建管道逻辑方法绑定。

如果是两个在结构上毫无关联的节点呢? 比如得分后如何通知UI进行刷新? 出于解耦，我们肯定不会向上面一样获取节点绑定信号。在 Godot 中，我们可以使用 Autoload（自动加载/单例）实现“信号总线 (Event Bus)”模式。
这相当于 Unity 中的 public static event Action OnDeath;但更可视化、更易于调试。

``` GDScript
# GameEvents.gd
extends Node

# 定义整个游戏通用的事件
signal game_started
signal game_over
signal bird_died
signal score_added(current_score) # 甚至可以带参数
```

**配置 Autoload** 后`GameEvents` 这个节点会在游戏启动时自动加载，并且你可以在任何脚本里直接通过 `GameEvents` 这个全局变量访问它。同理，我还创建了`GameData`存储我们的score。

![flappybird](/img/image_2026-01-21.10.50.37.png)
项目地址 <https://github.com/kinzee/FlappyBird_GoDot/tree/master>

## Godot初体验

在AI的帮助下，我差不多花了1天的时间就完成了demo。这其中包括了寻找美术资产，上班摸鱼...

实践下来还是比较满意的。项目结构清晰，编辑器操作符合直接，GDScript简洁便捷。

### 关于Node

既是结构也是功能

既是对象也是组件

有时间再写写和unity组件，cocos2dx中的Node有什么区别。
