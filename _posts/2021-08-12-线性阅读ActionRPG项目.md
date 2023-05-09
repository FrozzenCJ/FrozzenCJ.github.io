---
title: UE4 GAS 方案探究
tags: Game Program

---

## 简介
本文是我的一篇UE引擎的学习笔记，在此之前，我看过不少文档和资料，但看过之后仍然觉得UE是个很抽象的东西，不够具体，不足以支撑实践。所以，我整理了这篇笔记，尽量以相对线性的方式，一步不落的从头开始介绍一个比较完整的Demo项目。帮助那些想要入门UE，但是又不知道从何学起的同学。

### 项目

本文介绍的是UE4的官方项目《ActionRPG》，相关信息如下：

1.  项目地址： https://www.unrealengine.com/marketplace/en-US/product/action-rpg
2.  文档说明： https://docs.unrealengine.com/4.26/en-US/Resources/SampleGames/ARPG/
3.  项目升级5.1方法（我试过，但是失败了） https://dev.epicgames.com/community/learning/tutorials/d8Ra/unreal-engine-updating-action-rpg-to-ue5-1  
    虽然这是个4.27的项目，但是仍然极具学习价值。

### 内容

本文的主要内容包括两方面：

1.  游戏基础配置，关卡逻辑，玩家角色，HUD等构成的整体结构
2.  如何定义一个玩家技能，它的运作逻辑，以及最终如何生效。

### 学习方法

关于学习方法：

1.  UE的项目逻辑配置极多，蓝图类型很多，之间会有网状结构，乍一看很乱。但任何杂乱的内容，最会有个头，我们最终可以理出一个线性流程来学习。从而最大程度上降低学习者的心智负担。
2.  偶尔会有环，这需要一点点的忍受未知的能力，以及需要你多做点猜想、假设。偶尔会需要逆向查找和学习一些前置知识，这需要把握好逆向学习的度，从而避免偏离主线太远。
3.  使用Obsidian笔记软件，配合BlueTopaz主题阅读本文，效果更佳。
![](/assets/images/img/Pasted image 20230316175700.png)
## 游戏的全局配置

了解UE项目，需要从配置开始，首先从菜单栏 Edit->Project Settings->Maps&Modes打开游戏的全局配置页面

### Project Settings / Maps & Modes

配置的核心内容如下：

1.  Default Maps（也就是一些地图相关的配置，本文中的level、map、关卡、地图，基本上说的都是一个东西，约等于Messiah的World，或者Unity的Scene）
    1.  Game Default Map为ActionRPG_Main说明点击编辑器Play按钮（你偶尔会听到PIE——Play In Editor也是这个概念，约等于Unity点击编辑器上的绿色三角）启动的会是这个Level（level就是地图）
    2.  Editor Startup Map为ActionRPG_P则说明我们打开编辑器看到的Level
2.  Default Modes
    1.  Default GameMode实际上是一个默认值
        1.  如果某Level没有定义GameMode（也就是值为None），则使用这里的GameMode。
        2.  GameMode包含的是一套玩法逻辑，它本质上是个蓝图或者C++类，可以相应各种事件执行对应逻辑，也可用于提供一些方法给其它类。
        3.  通常每个level也就是地图都会配置一个GameMode，多个level可以应用同一个GameMode
    2.  Selected GameMode下面的列表则定义了每个GameMode下具体运用的Class
        1.  `TODO`：进一步说明每个选项的意义
        2.  每个GameMode比较核心的配置时 创建一个什么样的角色（类似Unity中的Prefab，UE中的Prefab其实就是蓝图），以及用什么样的控制器PlayerController来控制这个角色。
        3.  蓝图在UE中的概念比较多，但无外乎就两个大类，一个是纯粹的逻辑模块，比如各种控制器。一个是类似Unity的Prefab那样的角色预制件，附带一个逻辑模块，比如角色蓝图。
3.  Game Instance Class
    1.  关于GameInstance
        1.  详见： https://zhuanlan.zhihu.com/p/23167068
        2.  概括一下：里面编写了应用于整个游戏范围的逻辑，那些独立于Level的逻辑或数据要在GameInstance中存储。
    2.  这里配置了一个具体的蓝图资源：BP_GameInstance，则也是我们首先要去了解的一个蓝图。

![](/assets/images/img/Pasted image 20230316180501.png)

## 游戏全局控制器：BP_GameInstance和技能资源

作者认为，UE中的蓝图主要有两个大类

1.  “控制器逻辑”，和代码中的各种Manager或Controller等价，控制着游戏某个层级上的逻辑，提供相关的函数接口，处理相关的事件。比如GameInstance蓝图，GameMode蓝图，LevelScript蓝图，PlayerController蓝图。
2.  类似Unity的Prefab预制件，是逻辑和资源的组合，用来定义一个玩家预制件，一个UI界面预制件，一个技能预制件等等。

### BP_GameInstance 的 EventGraph

BP_GameInstance 属于第一类蓝图，是ActionRPG游戏的全局控制器，继承自GameInstance蓝图基类，用来控制游戏的整体逻辑。

可以在资源面板中，Content目录下搜索BP_GameInstance，双击打开蓝图。一个蓝图可以有多个Graph，可以在左边的列表中看到。

这里，首先引入眼帘的就是EventGraph：

1.  鼠标放上去可以看到 BP_GameInstance 继承自 `URPGGameInstanceBase`
    1.  所以，你发现项目中各种cpp之所以叫做xxxxbase，就是因为它们是给蓝图继承用的。
2.  双击打开，其EventGraph中存在6个功能Group
    1.  Event Init事件
        1.  唯一一个系统事件，触发Initialize StoreItems事件（详见下一条）
    2.  Initialize StoreItems事件
        1.  这里的ItemSlotsPerType是啥？
            1.  在基类`URPGGameInstanceBase`中定义了一个PrimaryAssetsType到Int32的字典，本质上是个字典（配置），定义了每种战斗资源的槽位数量，例如一个玩家最多持有三个武器，最多持有三瓶药水等等。在Details面板中可以看到详情，游戏中共有三种：技能，药品和武器。
        2.  `FPrimaryAssetType`的枚举值是哪里来的？
            1.  详见下文中的AssetManager，建议先去看看。
        3.  这里其实就是异步加载了每种类型的资源，然后加入到了StoreItems（一个蓝图定义的RPGItem数组）
        4.  这里还会有存盘逻辑，先略过。
    3.  Connect to Service事件
        1.  登录相关，先略过。
    4.  Load Game Level事件
        1.  一个函数，先清理，读条然后切换Level到战斗场景
    5.  RestartGameLevel
        1.  就是读条，然后触发Load Game Level
        2.  有GameMode蓝图在重启游戏时调用。
        3.  结束UI也会直接调用这个函数。
    6.  Pause Game事件
        1.  就是弹出（创建）/关闭暂停窗口，UMG部分的文档有讲过，有空再回顾。
    7.  GoToMainMenu事件
        1.  切换Level到登录界面。

### 概念：Custom Event

1.  我们可以自行定义任意Event，作为一段逻辑的起点。
2.  如果我们想触发上述逻辑，在需要的地方，右键菜单中查询Call Fuction，然后选择已经定义的Custom Event，从而创建一个触发该事件的节点。  
    https://docs.unrealengine.com/4.27/zh-CN/ProgrammingAndScripting/Blueprints/UserGuide/Events/Custom/

### AssetManager

知乎介绍： https://zhuanlan.zhihu.com/p/129712105  
官方文档： https://docs.unrealengine.com/5.1/zh-CN/asset-management-in-unreal-engine/

1.  我们在C++中对AssetType进行声明和定义
    1.  这通常在UAssetManager的子类，比如URPGAssetManager
    2.  例如 SkillItemType 定义了技能资源对象这个类型。
    3.  ActionARPG自定义了一个资源管理器子类`URPGAssetManager`，需要在Engine - General Settings目录下进行设置，让UE项目使用我们自定义的AssetManager，。
2.  在Project Settings / Game / AssetManager中进行资源基本信息的配置
    1.  如果我们想要在蓝图中对资源进行引用的话，还需要在项目设置里面对AssetType进行注册
    2.  每个ITEM会配置：
        1.  Type：也就是代码里面定义的Text
        2.  资产的BaseClass：例如 RPGSkillItem
        3.  资产的搜索的路径：/Game/Items/Skills
3.  去cpp中看下 RPGSkillItem
    1.  继承自URPGItem，override了不同的ItemType
    2.  URPGItem继承自UPrimaryDataAsset
    3.  URPGItem定义了GetPrimaryAssetId方法，从而让Secondary资产变成了Primary资产。
        1.  默认情况下，只有关卡（Map）（/Game/Maps）是PrimaryAsset（可能后边会发现还有一个PrimaryAssetLabel，这个暂时忽略）。 那如何将SecondaryAsset设为PrimaryAsset呢？那第一步就是给资源类实现 **virtual FPrimaryAssetId GetPrimaryAssetId() const override;** 接口方法。
        2.  返回的就是被子类复写的ItemType： `return FPrimaryAssetId(ItemType, GetFName());`
    4.  URPGItem定义了通用Item的各种属性，价格，数量，等级等。
4.  看看资源 /Game/Items/Skills
    1.  里面有很多RPGSkillItem类型的资源。
    2.  可有右键，在Miscellaneous目录下创建Data Asset，这样就有一个资源了。
    3.  里面有个Granted Ability选项，定义了具体技能逻辑模板，是`URPGGameplayAbility`的子类。

### URPGAssetManager

1.  有个关键点，该类在StartInitialLoading函数中，调用了：  
    `UAbilitySystemGlobals::Get().InitGlobalData();`
2.  Should be called once as part of project setup to load global data tables and tags
3.  这个GAS的初始化逻辑，我们暂时不care它，有空再看`TODO`

### 初识技能蓝图 URPGGameplayAbility

1.  别怕，我们只是瞥一眼。。。
2.  skill资源中的GrantedAbility字段，指向的就是这种类型的子类。
3.  URPGGameplayAbility 继承自 UGameplayAbility，后者来自GAS插件的定义。
4.  在player/skills目录下，会有具体的资源蓝图文件
    1.  可以通过创建Blueprint目录下的GamePlayAbility BP来创建一个新的。  

![](/assets/images/img/Pasted image 20230316175803.png)

5.  这里已经深入到了具体的技能实现流程，我们先跳过，暂时这些还不重要。

### SaveSlot

它是UE存档的一个概念，知道就行，等研究存档系统时再细看。

### 总结

上面我们了解了GameInstance蓝图，它是代表的是整个游戏，提供了一些全局的方法，比如加载存档、切换Level、暂停等等。我们也顺着它初步了解了游戏中的资源和GamePlayAbilitySystem后续称为GAS。

接下来，我们从登录界面开始看。从项目Settings中可以了解到，我们加载的第一张地图是ActionRPG_Main，所以直接打开这张地图即可（不知道在哪里的可以搜索一下）。

## 登录地图/界面 ActionRPG_Main

1.  直接双击打开Map：ActionRPG_Main，可以看到一个简单的场景。
2.  选中这个Map，然后打开Wordsettings窗口，可以看到这张地图使用的GameMode是蓝图是`BP_MainMenuGameMode`

### BP_MainMenuGameMode

1.  基类就是GameMode没什么好说
2.  BeginPlay事件  
    1. 创建了`WB Title界面`  
    2. Get/SetGlobalOptions（不重要，不用太关心）

游戏菜单界面，在Blueprint/WidgetBP下面

1.  构造时，基于手机与否决定是否显示退出按钮，然后播动画、音效、设置焦点
2.  点击StartGame按钮：调用LoadGameLevel函数（由GameInstance BP定义，前文有提过）进入战斗场景。
3.  点击OptionsButton，打开WB Options Screen
4.  点击Quit按钮，执行QuitGame函数

### WB Title界面

不重要，先跳过

### 总结

我们已经了解了登录界面是如何运作，以及如何跳转到战斗场景的，接下来直接打开战斗场景进行学习即可。

### WB Options Screen界面

### 战斗场景： ActionRPG_P

1.  打开Map：ActionRPG_P，
2.  同样的方法查看WorldSettings，发现全是None，说明采用的是默认值。
3.  查看ProjectSettings中的配置即可：
    1.  这张地图使用的GameMode为`BP_GameMode`
    2.  默认创建的玩家对象为`BP_PlayerCharacter`
    3.  默认使用的玩家控制器为`BP_PlayerController`
4.  上面这些蓝图就是游戏的核心蓝图了，后续主要就是学习它们。  
5.  选中地图的同时，打开Levels窗口
    1.  它是由一个persistlevel和另外两个level构成的
    2.  查看Persistent Level的LevelBlueprint，这是关卡蓝图。

### Level蓝图：Level BP

先看关卡蓝图，比较简单：

1.  BeginPlay
    1.  会加载场景Level，ActionRPG_Dungeon02_Asset，从节点上看可以选择阻塞式加载
    2.  淡出LoadingScreen，这是Instance的函数。
2.  DebugMode事件
    1.  `TODO` 没有找到触发者，可能是我搜索的方式不对，回头再想想
    2.  过程：
        1.  打印字符串
        2.  GetPlayerCharacterBP可以获取玩家BP，然后调用器方法
        3.  先切换武器DebugAutoSwitch（详见BP_PlayerCharacter）
        4.  再DebugAutoPlay（详见BP_PlayerCharacter）
        5.  再通过命令行指令显示帧率图和fps统计

## 全局函数库： BP_RPGFunctionLibrary蓝图

我们会发现有些函数定义在这个蓝图里，它继承自Blueprint Fuction Library基类，提供了一组静态函数给蓝图系统，主要是方便使用。

### 函数列表

1.  GetPlayerCharacterBP 获取玩家角色BP
    1.  从玩家控制器中获取
2.  GetPlayerControllerBP获取玩家控制器BP
    1.  优先从Game Mode BP中获取（已经转换过类型的BP）
    2.  否则从Player Controller中获取，并进行类型转换
3.  GetGameModeBP
    1.  获取 当前GameMode并转换为BPGameMode类型
4.  GetPlayerAnimBP
    1.  TODO
5.  GetGameInstanceBP
    1.  获取并转换为我们定义的子类
6.  IsRunningOnMobile
    1.  获取字符串并用一个Switch on String节点来控制返回值
7.  ShowMouseCursor
    1.  TODO

## 关卡逻辑控制者：BP_GameMode蓝图

### 问题：关卡蓝图和GameMode蓝图

前面的level蓝图已经定义了一些关卡相关的内容，这里有两个问题

1.  为什么这么小一个场景，却要定义persistLevel之外的level？分别是个场景和光照
    1.  `TODO` 有空再想，这个也不太重要
2.  level里面的逻辑和gamemode里面的逻辑异同何在？

> **思考：哪些逻辑应该写在GameMode里？哪些应该写在Level Blueprint里？**

-   概念上，Level是表示，World是逻辑，一个World如果有很多个Level拼在一起，那么也就是有了很多个LevelScriptActor，无法想象在那么多个地方写一个完整的游戏逻辑。所以**GameMode应该专注于逻辑的实现，而LevelScriptActor应该专注于本Level的表示逻辑**，比如改变Level内某些Actor的运动轨迹，或者某一个区域的重力，或者触发一段特效或动画。而GameMode应该专注于玩法，比如胜利条件，怪物刷新等。
-   组合上，同Controller应用到Pawn一样道理，因为**GameMode是可以应用在不同的Level的，所以通用的玩法应该放在GameMode里。**
-   GameMode只在Server存在（单机游戏也是Server），对于已经连接上Server的Client来说，因为游戏的状态都是由Sever决定的，Client只是负责展示，所以**Client上是没有GameMode的，但是有LevelScriptActor，所以GameMode里不要写Client特定相关的逻辑，比如操作UI等**。但是LevelScriptActor还是有的，而且支持RPC，即使如此，**LevelScriptActor还是应该只专注于表现，比如网络中触发一个特效火焰**。至于**UI，可以通过PlayerController的RPC，然后转发到GameInstance来操作**。
-   跟下层的PlayerController比较，**GameMode关心的是构建一个游戏本身的玩法，PlayerController关心的玩家的行为**。这两个行为是独立正交可以自由组合的。所以想想哪些逻辑属于游戏，哪些属于玩家，就应该清楚写在哪里了。
-   跟上层的GameInstance比较，**GameInstance关注的是更高层的不同World之间的逻辑，虽然有时候他也把手伸下来做些UI的管理工作，不过严谨来说，在UE里UI是独立于World的一个结构**，所以也还算能理解。因此可以把不同GameMode之间协调的工作交给GameInstance，而GameMode只专注自己的玩法世界。

### EventGraph部分——游戏的开始、计时器和结束

1.  Event BeginPlay
    1.  播放了一段过场动画（PlayDefaultIntroCutScene)
    2.  播放了一段2D音效
2.  PlayDefaultIntroCutScene
    1.  GetAllActorsOfClass 获取所有LevelSequenceActor
    2.  如果存在过场
        1.  获取第0个过场的SequencePlayer
        2.  调用BP_Player的`PlaySkippableCutscene`函数播放这个过场
            1.  详见玩家BP
            2.  `问题` 这里是如何StartGame的呢？过场播完了或者中断了就会由PlayerController蓝图中的逻辑调用过来。
    3.  如果不存在过场
        1.  Start Game函数
3.  Start Game
    1.  系统函数RestartPlayer创建玩家对象
        1.  Tries to spawn the player's pawn, at the location returned by FindPlayerStart
        2.  需要传入PlayerController
        3.  https://docs.unrealengine.com/4.26/en-US/BlueprintAPI/Game/RestartPlayer/
    2.  调用同一个蓝图的事件`StartPlayTimer`
    3.  调用蓝图的function部分开始布乖：`Start Enemy Spawn`
4.  StartPlayTimer
    1.  SetTimerByEvent函数用于设定一个重复定时器，不断触发函数。
    2.  设定好计时器后，记录当前的StartTime（注意，这是一个全局参数，不会被某个具体的蓝图实例所修改）
    3.  当事件触发器触发时，BattleTime每次-1
    4.  更新战斗界面的UI倒计时信息
        1.  从PlayerController中获取`OnScreenControls`（详见BP_PlayerController）
        2.  这里有个UpdateTimer函数用来设置
            1.  具体原理还没看懂，`TODO`
    5.  判断BattleTime倒计时是否到0，如果到0，则调用`GameOver`函数。
        1.  这是GameMode的一个变量，默认值为60
        2.  从EnemyManager的IncreaseNPCKillCount事件可以看到，每次杀怪都可以增加这个值，参数为TimeBonusPerKill
        3.  也就是说无限躲避也是不行，会因为时间不足而输掉游戏。
5.  `OnGameOver`事件
    1.  这个是经过了GameModeC++基类转而调用过来的。
    2.  SetGlobalTimeDilation 0.25放慢全局时间流速
    3.  延迟0.5s
    4.  恢复全局时间流速
    5.  暂停游戏
    6.  显示结束UI WB_inGame_Finish
    7.  切换为UIOnly的操作模式
6.  ShoweDebugUI
    1.  全局搜了似乎没有调用者
    2.  就是在非发布版中创建一个`Debug_UI`
7.  Restart事件和EventDoRestart
    1.  `TODO`EventDoRestart应该是Base基类提供的事件
    2.  没有找到Restart事件的调用者
        1.  inGameFinish的UI蓝图中，是直接调用了GameInstance中的方法。
    3.  调用GameInstance蓝图中的对应事件RestartGameLevel

### EnemyManager——刷怪机制

这是GameMode中除了EventGraph之外的另一个graph，它控制刷怪逻辑。  
插播几个推论：

1.  一个蓝图中的Functions部分是共享的，并不属于某个Graph，不会因为切换Graph而具有不同的Functions
2.  Custom Event的调用方式叫做call fuction

接下来看逻辑：

1.  IncreaseNPCKillCount
    1.  按参数增加剩余可用战斗时间
    2.  更新两个UI
    3.  杀敌数增加NPCKillsCount
2.  `Start Enemy Spawn` 进入战斗，开始生成怪物
    1.  由GameMode的EventGraph在StartGame事件的最后调用
    2.  判断是否有位置生成怪物`Get Random Spawn Point`
    3.  有的话延迟1s，释放下一波怪物`StartNewWave`
3.  `StartNewWave`
    1.  从DataTable（/Game/Blueprints/Progression/WavesProgression）中获取信息，基于wave index计算一个row index，提取数据后，逻辑流一分为二。
        1.  找不到数据就直接`GameOver`
    2.  找到数据则有一个拆解struct的过程，提取了SpawnGroup和Time，然后set为变量`wave`和`BattleTime`
        1.  `wave`本身是个数组，每个元素是一个字典`{'enemies': <list>}`，其中的list是个NPC蓝图列表。
    3.  最后`SpawnNewWave`
4.  `SpawnNewWave`
    1.  实例化UI蓝图：WB_WaveStart蓝图
    2.  `SpawnNextEnimesGroup` 创建下一组怪物
    3.  更新BattleTIme
5.  `SpawnNextEnimesGroup` 放一组怪
    1.  取`wave`[0]这个字典
        1.  从中获取`enemies`，得到一个数组
        2.  ForEachLoop循环这个数组，取出每个NPC，调用`SpawnEnemy`，然后移除数组的第0个元素
            1.  `TODO`为啥要移除第0个元素，难道不会导致跳跃式的创建吗？
    2.  移除wave的第0个元素
    3.  注意，这个wave是数据表中的SpawnGroup
6.  `SpawnEnemy`创建一个怪物
    1.  获取一个地点，然后SpawnActor
    2.  SpawnedEnemies计数器自增
    3.  创建出来的实例，绑定一个`EnemyDestroyed`事件
7.  `EnemyDestroyed`事件
    1.  计数器-1
    2.  然后检查是否一波被消灭完了`CheckCurrentWave`
8.  `CheckCurrentWave`
    1.  检查计数器是否小于等于0
    2.  检查wave是否还有剩余（0这个idnex是是否合法）
    3.  合法：`SpawnNextEnimesGroup` 放一组怪
    4.  空了：延迟2s，`触发OnWaveFinished事件`，CurrentWave++
9.  `OnWaveFinished事件`
    1.  显示Wave结束的UI
    2.  延迟4s，`StartNewWave`放下一波怪
        1.  StartNewWave负责检查，如果没怪可用了，结束游戏。
    3.  `TODO`先延迟2s再延迟4s，靠时差来把控进度，有点虚啊。

`重要`ForEachLoop节点有一个非常重要的特性，LoopBody端口会循环执行，全部执行完毕后会执行Complete端口。  
`重要` 一个Event被触发，如果尚未执行完毕，就再次出发，结果会如何？

#### 函数部分

1.  `Get Random Spawn Point`
    1.  事先在场景中摆了需要占位对象`NPCSpawnBox`
    2.  全部获取到一个数组A
    3.  看A[0]是否合法
        1.  非法则直接返回inValid
        2.  合法，则从0~lastIndex中随机一个index，取A[i]的Transform，设置给SpawnPoint并返回。
2.  `GameOver`
    1.  这是C++GameMode基类提供的一个函数
    2.  它会设定标记，然后调用 K2_OnGameOver
    3.  K2_OnGameOver没有实现，估计是给蓝图重载用的，实际名称为OnGameOver

#### 自动战斗

PlayerController会在按键触发时调用过来，按键的配置来自ProjectSettings。

#### TODO

`TODO`有两个函数没有找到入口点，估计和自动战斗有关，有空补回来看看。

1.  ToggleAutoBattleMode
2.  InitializeAutoBattle

## 玩家控制器：BP_PlayerController蓝图

第一个问题：PlayerController是如何被创建的？  
https://forums.unrealengine.com/t/where-are-instances-of-default-playercontrollers-and-pawns-created-in-gamemode/336551/3  
有两个调用点，一个是Login一个Travel函数，都是由GameModeBase函数负责实现的。

拓展一下：  
Understanding Game Mode, Sessions and game life cycle! - UE4 Advanced Blueprints Tutorial  
https://www.youtube.com/watch?v=CI8flHPwT7Y

### EventGraph

#### 输入处理

1.  暂停：调用GameInstance的暂停函数
2.  打开背包：用了一堆逻辑与或节点，
3.  自动战斗：调用GameMode的函数启用自动战斗

从这里可以理解GameMode和GameInstance的区别的

#### 过场动画的控制

1.  PlaySkippableCutscene
    1.  这个函数会在GameMode启动后发现存在过场动画时被调用
        1.  首先播放过场动画，创建一个跳过过场动画的UI，隐藏HUD
        2.  触发一个事件分发器：OnCutsceneStarted
        3.  但暂时没找到监听者
    2.  当播放完毕或者玩家按下回车
        1.  关闭过场动画播放器
        2.  恢复HUD
        3.  StartGame
            1.  `问题`：这里的做法是否正确？StartGame这种重要逻辑是否应该允许由一个PlayerController来触发？毕竟StartGame还负责了整个游戏的倒计时逻辑。在多人协作尤其是网游中，有某个playerController控制GameMode的逻辑有点不可思议。
            2.  `回答` 本地分屏或者本地多人的话Controller就会有多个，它们更像是作为每个真实玩家的代表和服务端通信的用途，被动接受信息以及推送本人的信息。GameMode就像是单机游戏中的服务端，所以这里这个函数最好不要叫做StartGame，改为TellServerIamReady比较好。我觉得就和谐多了
        4.  移除跳过按钮
        5.  触发一个事件分发器：OnCutsceneEnded

#### 初始化逻辑——Posses

1.  入口是EventOnPossess
    1.  有pawn被绑定过来时就会触发，且通过DoOnce节点保证只执行一次
    2.  把PlayerCharacter缓存起来
    3.  CreateHUD
        1.  `问题` 这里虽然保证了HUD只被创建一次，但是PlayerCharacter经过不同的Possess会变的呀？？
    4.  监听物品（红魂）的变化
    5.  创建所有武器，调用Character的函数：`CreateAllWeapons`
    6.  切换输入模式

#### HUD

1.  CreateHUD自定义事件
    1.  这里会基于PC或者移动端，创建不同的UI界面
    2.  PC的话会创建WB_HUD_PC，保存为On Screen Controls参数
        1.  在GameMode蓝图中就会获取这个变量，然后tick的更新它上面的倒计时。
    3.  UpdateIcons
        1.  更新上面的按钮
2.  ShowHUD
    1.  控制显示与隐藏，过场等过程中需要
3.  UpdateOnScreenControls
    1.  调用iHud定义的接口，更新实际的主界面。

#### 玩家转身

和常规逻辑差不多，基于Touch，按下时记录一个x，拖动时可以转镜头，松开后则不行。拖动时，计算两次之间的x偏移，得出一个数值，叠加到当前方向上，设置为新方向

两个重要接口：

1.  GetControlRotation
    1.  `TODO`
2.  SetControlRotation
    1.  `TODO`

#### 玩家移动

移动的逻辑也在EventGraph当中，但是比较重要，拉出来单独说：

1.  前提条件1：
    1.  非自动战斗模式
    2.  能够GetControlledPawn并且PawnValid
    3.  没有被限制移动，BlockedMovement标记控制
2.  前提条件2：Pawn是Character
3.  AnimInstance没有Montage在播放中，如果有说明在放技能，只允许转身不允许移动。
4.  动作中
    1.  计算相机方向和角色yaw方向的插值，然后让角色Actor旋转这个角度值
5.  非动作中
    1.  使用GetMoveForward的数值让角色沿着CameraForward的方向移动
    2.  使用GetMoveRight的数值让角色沿着CameraRIght方向移动
    3.  Get MoveForward 获取的就是Axis Input的输入值
    4.  AddMovementInput
        1.  文档地址：https://docs.unrealengine.com/5.1/en-US/API/Runtime/Engine/GameFramework/APawn/AddMovementInput/
        2.  简单说Character和DefaultPawn天然支持，有空看看`TODO`

### 背包Inventory

#### EventGraph中监听Soul（就是战神里面的红魂）变化

在PlayerController的OpnPossess过程中，会有BindEvent OnInventoryItemChanged函数，绑定了一个事件，这里其实是个EventDispatcher，从红色Event端口拖出来，选择EventDispatchers就可以了。

1.  Handle Inventory Item Changed回调函数
    1.  这里就是判断一下变化了的物品是否为红魂，如果是的话，就再触发一个OnSoulsUpdated事件

OnInventoryItemChanged事件，是C++层的ControllerBase提供的一个Delegate，类型是FOnInventoryItemChanged

```
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnInventoryItemChanged, bool, bAdded, URPGItem*, Item);

// 触发点
void ARPGPlayerControllerBase::NotifyInventoryItemChanged(bool bAdded, URPGItem* Item)
{
	// Notify native before blueprint
	OnInventoryItemChangedNative.Broadcast(bAdded, Item);
	OnInventoryItemChanged.Broadcast(bAdded, Item);

	// Call BP update event
	InventoryItemChanged(bAdded, Item);
}

// 触发点
bool ARPGPlayerControllerBase::RemoveInventoryItem(URPGItem* RemovedItem, int32 RemoveCount)
bool ARPGPlayerControllerBase::AddInventoryItem(URPGItem* NewItem, int32 ItemCount, int32 ItemLevel, bool bAutoSlot)

// 暴露给蓝图用于增删物品
UFUNCTION(BlueprintCallable, Category = Inventory)
bool AddInventoryItem(URPGItem* NewItem, int32 ItemCount = 1, int32 ItemLevel = 1, bool bAutoSlot = true);
```

#### 背包InventoryGraph

不重要，有空再细看，总之记住：

1.  玩家使用SoulItem购买道具的相关逻辑，都在这里足矣。
2.  物品操作的具体函数，大部分都在C++层实现。

### 接下来去哪儿？

此时，显然有两个问题选在心头：

1.  玩家是如何发动攻击的？
2.  玩家发动攻击用的武器是哪里来的？

对于第一个问题，我们可以从Input配置入手，发现一个NormalAttack操作，直接全局搜索这个事件即可，答案也是显然的，下一站在BP_PlayerCharacter蓝图。

## 玩家对象：BP_PlayerCharacter蓝图

到这里，发现内容越来越多了，不过东西越多越不用害怕，内容多而杂的地方往往是简单逻辑平铺大杂烩的。就像暴风城一样，那么多建筑和商店，看似杂乱，但实际上一来有分区，二来你常去的地方也就是拍卖行、银行、训练师和鸟点等少数几个地方。所以，这里一方面需要更多的大局观，泛泛的总结归纳。另一方面，需要明确目标，更多的思考你想了解什么问题。

从这里开始我们也不会再去过所有的细节，而是更有目的性的来看代码。

### Actor蓝图中的组件列表

UE的蓝图不是纯逻辑内容，它更像是Unity的Prefab，包含了预设的逻辑和表现。  
界面左上角查看组件：

1.  胶囊体组件是Root
    1.  两个箭头组件(`TODO`
    2.  Mesh
    3.  SpringArm+TPSCamera（相机
    4.  Inventory灯光和相机（`TODO`
2.  CharacterMovement组件
3.  技能系统组件：AbilitySystemComponent

首先是一堆输入的处理的节点，在细看之前，先去ProjectSettings中看一下Input配置：

-   普攻：左键
-   特攻：F按键
-   翻滚：空格和右键
-   切换武器：滚轮
-   背包：Tab
-   自动玩耍：退格
-   暂停：Esc
-   使用道具：R
-   跑步：左边ctrl

### EventGraph

EventGraph内容

1.  玩家操作
    1.  输入处理
    2.  翻滚
    3.  使用血瓶
    4.  切换武器
2.  BeginPlay是开启Tickable，相机Fade效果
3.  监听生命值和法力变化，并更新UI
4.  受击效果
    1.  播放了一个Montage
5.  Debug相关功能

### 功能细节 - 普通攻击

#### 普通攻击（1）

这段比较有趣，从一个问题触发，慢慢看

1.  按下普通攻击按钮时发生了什么？
    1.  执行DoMelleAttack函数触发普通攻击
    2.  子类和基类都实现了DoMeleeAttack函数，子类支持连招，基类就只是普通攻击
2.  玩家（子类）`DoMelleAttack`函数做了什么？  
    1. 先判断CanUseAnyAbility，要求游戏非暂停、玩家没有在使用技能，玩家或者。  
    1. 这里判断是否使用技能，访问了GetActiveAbilitiesWithTag接口，用了一个Ability.Skill的Tag进行筛选。  
    2. 猜测：普通攻击要么不是一个技能，要么不是Skill的Tag。  
    3. 这个函数定义在BP_Character这个基类当中。  
    2. 判断是否在近战攻击  
    1. 这里印证了前面的猜想，它是一个技能但Tag不同。  
    3. 近战中：执行下一段combo：`JumpSectionForCombo`  
    4. 非近战中：激活技能`ActivateAbilitiesWithItemSlot`使用`CurrentWeaponSlot`

到此为止有很多疑惑摆在我们面前：

-   `CurrentWeaponSlot`和技能时什么关系？
-   `ActivateAbilitiesWithItemSlot`为什么可以释放技能？
-   `GetActiveAbilitiesWithTag` 是如何工作的？
-   `JumpSectionForCombo`如何触发连招？

但首先我们需要解决的是`普通攻击的技能从何而来?`，然后剩下三个问题也就迎刃而解了。

#### 玩家的当前武器

`CurrentWeaponSlot`和技能时什么关系？

1.  顺手搜索一下，可以知道，武器的初始化过程源自PlayerController，在Initialize的部分会调用`CreateAllWeapons`函数。當然，这个函数也会在处理背包时被调用。

`CreateAllWeapons`干了啥？

1.  首先清空了当前武器数组`EquippedWeapons`，其中存的是`WeaponActor`也是个蓝图预制件。
2.  获取了一波`GetSlotedItems`-Weapon（武器槽位中的对象信息，详见下个问题），设置到`Equipped RPGItems`，循环处理：
    1.  将RPGItem转换RPGWeaponItem、获取WeaponAcotrClass、然后创建WeaponAcotor实例
3.  循环完毕后，`AttachNextWeapon`来挂接武器，包括循环切换武器。

`SlotedItems`存的是什么信息？

1.  在CreateAllWeapons中，我们获取了这个字典中，类型为Weapon的Item。
2.  这里的类型指的是：`FPrimaryAssetType ItemType`，它是把武器物品定义成了一种UE的资源，由AssetManager管理，前文在BP_GameInstance中已经介绍了这一概念。
3.  这数据结构是由 bool ARPGPlayerControllerBase::LoadInventory() 进行初始化的。
    1.  调用链：GameInstance-EventInit- `HandleSaveGameLoaded`-LoadInventory
    2.  调用链：ARPGPlayerControllerBase::BeginPlay()-LoadInventory
    3.  初始化：它会用GameInstance->ItemSlotsPerType这个结构来初始化，该结构key是`FRPGItemSlot`（key, SlotNumber）
        1.  ItemSlotsPerType的结构key是PrimaryAssetType，value则是数量
    4.  具体槽位中的物品会在加载游戏存档时恢复，或者通过函数接口添加。
        1.  比如捡东西的时候就会调用`SetSlottedItem`

#### 玩家登录时的第一把武器是哪里来的？

1.  单步调试一下，断点放在LoadInventory，发现是来自CurrentSaveGame存档信息。
2.  看下代码，有个AddDefaultInventory函数，把DefaultInventory添加到存档里面了，这个配置则来自于蓝图GamneInstance
3.  打开蓝图GamneInstance，点击上方的ClassDefaults，可以看到Details面板中给玩家添加了Weapon_Axe
4.  改掉然后清档就可以试试了，锤子好玩的。

#### 玩家怎么切换武器？

切换武器有几个关键函数，都定义在了PlayerCharacter的蓝图当中

1.  `WeaponAttachMethod`：通过`DetachFromActor`和`AttachActorToComponent`来挂接和移除武器实体。
2.  `CreateAllWeapons`会在我们获取武器时，自动切换到下一把（大概率就是新武器，还没确认）
3.  `AttachNextWeapon` 计算下一个武器index，然后调用WeaponAttachMethod
4.  `UpdateCurWeaponSlot` 刷新一下手上武器模型。

`AttachActorToComponent` 这是游戏中常见的概念，在UE中是把一个Actor的RootComponent挂接在另一个Actor的组件上，这里就是玩家的Mesh组件。（Attaches the RootComponent of this Actor to the supplied component, optionally at a named socket. It is not valid to call this on components that are not Registered.）  
https://docs.unrealengine.com/5.0/en-US/BlueprintAPI/Transformation/AttachActorToComponent/

#### 普通攻击（2）—— 释放技能的函数

`ActivateAbilitiesWithItemSlot`为什么可以释放技能？

1.  DoMeleeAttack函数会以CurrentWeaponSlot为参数，调用上述函数。
    1.  参数类型为 FRPGItemSlot，本质上是用来做主键的，ItemType+SlotNumber
    2.  CurrentWeaponSlot本质上是个CurrentWeapon的Slot属性
    3.  CurrentWeapon是Weapon Actor
2.  这里先获取技能，然后通过函数释放

```
/**
 * Attempts to activate any ability in the specified item slot. Will return false if no activatable ability found or activation fails
 * Returns true if it thinks it activated, but it may return false positives due to failure later in activation.
 * If bAllowRemoteActivation is true, it will remotely activate local/server abilities, if false it will only try to locally activate the ability
 */
UFUNCTION(BlueprintCallable, Category = "Abilities")
bool ARPGCharacterBase::ActivateAbilitiesWithItemSlot(FRPGItemSlot ItemSlot, bool bAllowRemoteActivation)
{
	FGameplayAbilitySpecHandle* FoundHandle = SlottedAbilities.Find(ItemSlot);

	if (FoundHandle && AbilitySystemComponent)
	{
		return AbilitySystemComponent->TryActivateAbility(*FoundHandle, bAllowRemoteActivation);
	}

	return false;
}
```

#### 玩家的武器为和会赋予普通攻击技能？

SlottedAbilities 是如何被填充的？

1.  在玩家被PossessedBy控制器时，会逐步调用到 FillSlottedAbilitySpecs

```
UE4Editor-ActionRPG.dll!ARPGCharacterBase::FillSlottedAbilitySpecs(TMap<FRPGItemSlot,FGameplayAbilitySpec,FDefaultSetAllocator,TDefaultMapHashableKeyFuncs<FRPGItemSlot,FGameplayAbilitySpec,0>> & SlottedAbilitySpecs) Line 121	C++
 	UE4Editor-ActionRPG.dll!ARPGCharacterBase::AddSlottedGameplayAbilities() Line 149	C++
 	UE4Editor-ActionRPG.dll!ARPGCharacterBase::AddStartupGameplayAbilities() Line 53	C++
 	UE4Editor-ActionRPG.dll!ARPGCharacterBase::PossessedBy(AController * NewController) Line 219	C++

```

2.  `FillSlottedAbilitySpecs` 这个函数会从URPGItem类型的道具实例中提取技能和技能等级，还会判断是否为武器，武器技能的等级会和武器配置信息挂钩。

```
	if (InventorySource)
	{
		const TMap<FRPGItemSlot, URPGItem*>& SlottedItemMap = InventorySource->GetSlottedItemMap();

		for (const TPair<FRPGItemSlot, URPGItem*>& ItemPair : SlottedItemMap)
		{
			URPGItem* SlottedItem = ItemPair.Value;

			// Use the character level as default
			int32 AbilityLevel = GetCharacterLevel();

			if (SlottedItem && SlottedItem->ItemType.GetName() == FName(TEXT("Weapon")))
			{
				// Override the ability level to use the data from the slotted item
				AbilityLevel = SlottedItem->AbilityLevel;
			}

			if (SlottedItem && SlottedItem->GrantedAbility)
			{
				// This will override anything from default
				SlottedAbilitySpecs.Add(ItemPair.Key, FGameplayAbilitySpec(SlottedItem->GrantedAbility, AbilityLevel, INDEX_NONE, SlottedItem));
			}
		}
	}
```

3.  填充完毕后，会执行下面的函数来填充`SlottedAbilities`

```
void ARPGCharacterBase::AddSlottedGameplayAbilities()
{
	TMap<FRPGItemSlot, FGameplayAbilitySpec> SlottedAbilitySpecs;
	FillSlottedAbilitySpecs(SlottedAbilitySpecs);
	
	// Now add abilities if needed
	for (const TPair<FRPGItemSlot, FGameplayAbilitySpec>& SpecPair : SlottedAbilitySpecs)
	{
		FGameplayAbilitySpecHandle& SpecHandle = SlottedAbilities.FindOrAdd(SpecPair.Key);

		if (!SpecHandle.IsValid())
		{
			SpecHandle = AbilitySystemComponent->GiveAbility(SpecPair.Value);
		}
	}
}

```

总结一下，玩家身上挂了两个属性

1.  SlottedAbilitySpecs 保存了一种映射：（FRPGItemSlot, FGameplayAbilitySpec）
2.  SlottedAbilities 保存了一种映射：（FRPGItemSlot, FGameplayAbilitySpecHandle）
3.  二者的key在这里做了一次对齐，handle是从前者生成出来的。`SpecHandle = AbilitySystemComponent->GiveAbility(SpecPair.Value);`
4.  简单来说，我们通过Spec描述技能，然后用这个描述来告诉GAS应该赋予玩家什么样的技能，然后GAS会在成功赋予技能后，返回给我们一个handle，句柄？

## 玩家对象基类：BP_Character

这个是BP_PlayerCharacter的基类，有空再看，定义了一些基本玩家逻辑。

1.  IsUsingSkill判断玩家是否在放技能，Function分类为Animation

## UE的技能系统插件：Gameplay Ability System

这里我们就需要去看看实际技能系统的文档了： https://docs.unrealengine.com/5.1/zh-CN/using-gameplay-abilities-in-unreal-engine/

### 技能的给予和移除

#### 给予

-   `GiveAbility`：使用 `FGameplayAbilitySpec` 指定要添加的技能，并返回 `FGameplayAbilitySpecHandle`。
-   `GiveAbilityAndActivateOnce`：使用 `FGameplayAbilitySpec` 指定要添加的技能，并返回 `FGameplayAbilitySpecHandle`。技能必须实例化，并且必须能够在服务器上运行。尝试在服务器上运行技能后，将返回 `FGameplayAbilitySpecHandle`。如果技能没有满足所需条件，或者无法执行，返回值将无效，并且技能系统组件将不会被授予该技能。

#### 移除

-   `ClearAbility`: 从技能系统组件中移除指定技能。
-   `SetRemoveAbilityOnEnd`：当该技能执行完毕时，将该技能从技能系统组件中移除。如果未执行该技能，将立即移除它。如果正在执行该技能，将立即清除其输入，这样玩家就无法重新激活它或与它互动。
-   `ClearAllAbilities`：从技能系统组件中移除所有技能。此函数是唯一不需要 `FGameplayAbilitySpecHandle` 的函数。

#### 释放

1.  `CanActivateAbility`也可以让调用者知道是否可执行该技能。
2.  `CallActivateAbility` 执行技能相关的游戏代码，但不会检查该技能是否可用。通常在`CanActivateAbility`检查及执行技能之间需要某些逻辑时才会调用该函数。
3.  `TryActivateAbility` 是执行技能的典型方式。该函数调用 `CanActivateAbility` 来确定技能是否可以立即运行，如果可以，则继续调用 `CallActivateAbility`。
4.  `EndAbility` （C++）或End Ability节点（蓝图）会在技能执行完毕后将其关闭。如果技能被取消，`UGameplayAbility` 类会将其作为取消流程的一部分自动处理，但其他情况下，开发者都必须调用C++函数或在技能的蓝图图表中添加节点。

### 属性系统：URPGAttributeSet

#### 定义和赋予

1.  Extend the base Attribute Set class, `UAttributeSet`, and add your Gameplay Attributes as `FGameplayAttributeData` UProperties.
2.  Store the Attribute Set on the Actor, and expose it to the engine. Use the `const` keyword to ensure that code cannot modify the Attribute Set directly. Add this to your Actor's class definition
3.  Register the Attribute Set with the appropriate Ability System Component. (定义的属性需要注册给ASC)
    1.  This happens **automatically** when you instantiate the Attribute Set, which you can do in the Actor's constructor, or during `BeginPlay`, as long as the Actor's `GetAbilitySystemComponent` function returns a valid Ability System Component at the moment of instantiation.
        1.  `AttributeSet = CreateDefaultSubobject<URPGAttributeSet>(TEXT("AttributeSet"));`
    2.  You can also edit the Actor's Blueprint and add the Attribute Set type to the Ability System Component's Default Starting Data. A third method is to instruct the Ability System Component to instantiate the Attribute Set, which will then register it automatically, as in this example:

这块，英文的文档更加完善： https://docs.unrealengine.com/5.1/en-US/gameplay-attributes-and-attribute-sets-for-the-gameplay-ability-system-in-unreal-engine/

可以方便的添加一些Getter和Setter方法，在ActionRPG中就定义了这样的宏，对GAS系统的宏进行了二次封装。

```

// Uses macros from AttributeSet.h
# define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```

### 游戏效果

#### 玩家属性的初始化

下面的内容帮助我们为玩家初始化属性系统：

1.  在PlayerCharacter蓝图中，打开Classes Defaults，在Details/Ability 引用（赋予）了玩家一一个被动技能效果
2.  这个效果定义在文件中
    1.  ActionRPG/Content/Abilities/Player/GE_PlayerStats.uasset
    2.  继承自 GE_StatsBase
    3.  继承自 GamePlayEffect
    4.  可以添加Modifier来用各种方式修改玩家属性
    5.  `TODO` 这里非常值得借鉴

#### GamePlayEffect

这个东西基本就是Buff的概念了（当然，比Buff要更通用一点）

这里非常值得借鉴：

1.  周期属性
2.  Modifier
    1.  Op 可以是覆盖，乘法，除法，加法，减法等等
    2.  Magnitude来源：类型、系数、资源curve文件，列名，预览值等等。

### 技能

#### 普通攻击（3）普通攻击技能的运作流程

有了上面的知识，我们就可以继续了解玩家的普通攻击技能是如何运作的了。

##### 相关资源

1.  再次回到普通攻击的问题上，普通攻击技能来自武器，之前也提到，查看玩家默认武器斧头：
    1.  `ActionRPG/Content/Items/Weapons/Weapon_Axe.uasset`
2.  斧头其中有个属性为GrantedAbility（SlottedItem->GrantedAbility），代表了其上挂的武器技能：
    1.  `ActionRPG/Content/Abilities/Player/Axe/GA_PlayerAxeMelee.uasset`
3.  这个技能资源的基类是：`GA_MeleeBase`
4.  使用的攻击动画（蒙太奇）资源是：`AM_Attack_Axe`
    1.  这里我们需要对Anim Montage有一定了解才行，另起一篇文档： [UE文档学习：动画系统和动画蒙太奇](https://km.netease.com/article/466037)

##### 攻击逻辑流

1.  技能对象EventGraph，在ActivateAbility事件中，会播放动画蒙太奇
    1.  通过封装过的接口节点`PlayMontageAndWaitForEvent`
2.  动画蒙太奇会触发WeaponAttackNS这样一个NotifyState
3.  NotifyState蓝图中
    1.  ReceivedNotifyBegins事件
        1.  从参数Mesh对象获取Owner，转换为PlayerCharacter，再获取其CurrentWeapon对象
        2.  触发CurrentWeapon对象的BeginWeaponAttack，也就是WeaponActor开启其物理碰撞检测。
    2.  ReceivedNotifyEnd事件
        1.  同上，本质上是关闭物理检测。
4.  WeaponActor的物理BeginOverlap事件会
    1.  判断攻击者Instigator和受击者是否为同一人，避免打到自己。
    2.  在overlap接触过程中只碰撞一次，脱离之前都不会再次进入后续逻辑。
    3.  构建攻击事件并发送，SendGamePlayEventToActor，Actor是攻击者。
    4.  消耗武器次数（火焰锤）
    5.  开启顿帧效果
5.  技能蓝图响应SendGamePlayEventToActor
    1.  在技能对象的EventGraph中，PlayMontage节点会在SendGamePlayEventToActor被触发时返回，EventReceived端口后续的节点会被执行。
    2.  后续的`ApplyEffectContainer`会执行C++函数，应用技能效果。
    3.  这个过程中通过Tag对其逻辑，Tag的内容详见扩展阅读部分。

##### 普通攻击的效果

1.  能力图或者说技能蓝图 GA_PlayerAxeMelee 和它的基类，都会定义GamePlayEffects信息，在EffectContainerMap中，保存了多种攻击效果GE。
    1.  Tag表示效果的分类，是人为定义的，需要攻击动作中的关键帧进行匹配。
    2.  TargetType是用于执行相应效果的对象类型，类似于对`URPGTargetType_UseEventData`，这些都是定义在cpp当中的，通过逻辑定义了我们该如何基于一次攻击参数，为效果结算生成结算目标。
    3.  TargetGameplayEffectClasses数组中则包含了实际的效果类，比如GE_PlayerAxeMelee。
2.  效果包括：
    1.  伤害
    2.  音效
    3.  特效等等。
3.  映射关系：
    1.  蒙太奇的WeaponAttackNS会带上Tag，借助Weapon的物理碰撞，将消息传递给GAS，从而让Ability技能对象蓝图接受到事件（附带Tag），然后处理函数`ApplyEffectContainer` 通过这个Tag拿到攻击对应的效果对象GE，并使其生效。

#### 连招的原理

1.  通常连击技能会有一个窗口
    1.  普通攻击的动作蒙太奇有一个JumpSectionNS区间，定义了玩家何时可以通过操作来打出combo：
        1.  在这个NotifyState的蓝图中，会在开始和结束区间开启BP_PlayerCharacter的EnableComboPeriod，从而打开连招区间。
        2.  Tick循环会判断是否开启了自动攻击，如果开启，则会调用JumpSectionForCombo函数跳转到下一个连招阶段。
        3.  这个NS关键帧有一个列表JumpSections，表示当前连招下一段的名字，指向montage的下一段动画
        4.  在Begin事件中，会把这个ns对象设置为玩家的JumpSectionNotify属性，从而让玩家蓝图知道连招的下一段动作是啥。
2.  打出Combo
    1.  玩家在一段攻击时按下攻击键，可以出发连击。
    2.  查看DoMeleeAttack函数，可以知道连击执行的是 BP_PlayerCharacter的JumpSectionForCombo函数
    3.  执行该函数时，首先需要EnableComboPeriod
    4.  然后会获取当前动作蒙太奇的当前section，以及读取JumpSectionNS定义的下一段section的name。
    5.  最终，通过MontageSetNextSection节点，告诉蒙太奇播放完当前动作后，跳转到下一段连击动作。
    6.  此时记得取消EnableComboPeriod。
3.  `问题` 显然，下一段连招动作通常可以触发多段攻击的技能回掉，有空看下细节。
4.  `问题` 显然，多段攻击的效果看上去似乎是一样的，如果不一样，该怎么做呢？
    1.  `答案` 显然是GA定义多个tag分别对应不同段数的攻击，同时NS区间使用不同的Tag名称。

## 怪物是怎么死的？

1.  GameEffect会更新血量
2.  AttributeAsset会调用Actor的血量变化接口`HandleHealthChanged`
3.  `HandleHealthChanged` 会抛出事件给蓝图。
4.  蓝图逻辑更新UI，触发OnEventDeath
5.  OnEventDeath 会销毁怪物，更新统计信息

## 游戏是如何结束的？

1.  GameMode脚本负责创建怪物，同时会绑定监听怪物销毁事件 EnemyDestroyed，从而统计怪物数量。
2.  怪物数量减到0，会触发下一波怪物的刷新，或者直接结束游戏。

## 扩展内容

### 动画系统

[UE文档学习：动画系统和动画蒙太奇](https://km.netease.com/article/466037)

### UE文档学习：GamePlayTag

https://www.unrealengine.com/zh-CN/tech-blog/using-gameplay-tags-to-label-and-organize-your-content-in-ue4

层次化标记可以成为整理概念和数据的非常实用的方法，游戏性标记系统是 UE4 用于声明和查询层次化标记的方法。

1.  Tag是UE提供的分类标签系统，基于Path的，x.y的形式存在。
2.  可以在ProjectSettings中进行Tag的定义，有一套基于Path的tag编辑界面。
3.  管理器是`UGameplayTagsManager`
4.  项目中可以任意定义类型和和进行使用。

### HUD：iHUD接口、PC和移动端实现

1.  iHUD定义了一系列函数，用于描述一个完备的HUD实现类需要实现哪些接口。
2.  这里有UpdateTimer、AddBonusTime, UpdateIcons, SetHP, SetMana, SetAutoPlayEnabled
3.  WB_HUD_PC和WB_HUD_MOBILE都继承了这个接口，所以虽然实现类不同，但是调用比较无痛。
    1.  GameMode的EnemyEvent检测到有敌人被杀，会更新倒计时信息，双击那里的函数，会看到跳跃到iHUD，所以不太方便观看。

### 输入按钮 WB_InputButton

HUD上的战斗用按钮都是WB_InputButton的实例

### WB_WaveStart蓝图

1.  由GameMode的EnemyGraph在释放新的一波小怪时创建
2.  从GameMode中获取波束，然后显示并且播放动画

## TODO

### 列表

1.  `GetActiveAbilitiesWithTag` 是如何工作的？
2.  玩家图标上的技能时哪里来的？
3.  Debug相关功能
4.  使用血瓶
5.  火球术: 攒够魂了可以购买技能，顺着看看如何获得一个技能。
6.  DebugAutoSwitch
7.  DebugAutoPlay
8.  蓝图Variable的Category是什么用途？
9.  Asset的ID是个重要的概念似乎和全局引用有关系

### 杂记

#### 逻辑流和数据流耦合的情况？

这是一个有趣的结构，数据流依赖链上的前置节点，是某个前序逻辑流中的节点的输出。如果这个前序逻辑流中的节点没有被执行，那么会run出来什么效果？  


#### 蓝图的Event触发机制?

一次Event触发后还没有执行完，下一次就进来了怎么办？（同步逻辑还好，如果存在异步逻辑会怎样？)

## 随手记录

技能数据，如果还是类似AL的话，也应当支持版本管理

> ARPG通过原生结构使用 **URPGSaveGame** ARPG 类将玩家的物品栏（包括灵魂/经验）存储在磁盘上。通常，任何关键信息都应使用原生结构存储，以便可以使用原生版本控制和修正代码。对于 **URPGSaveGame**，这使用 **ERPGSaveGameVersion** 枚举值和Serialize函数中的修正代码来实现。这是因为在任何时候，用户定义的结构体都可能被意外修改。如果开发者试图重命名或删除字段，那么玩家保存的游戏就会丢失数据，可能导致玩家保存的数据被完全破坏。通常，应使用具有版本控制功能的原生结构来实现关键数据。

虚拟路径而非实际物理路径来引用资源（之前项目已经是这样了，我们用技能ID，而非mth文件路径

> ARPG保存游戏使用存储为字符串的 **PrimaryAssetIds**（版块类型 **ItemType:ItemName**）来存储物品栏。通过这种方法存储项目引用比资源路径（如 **/Game/Items/ItemName.ItemName**）更安全，因为即使资源移动了，引用也不会破坏。如果更改资源名称，可以使用 **PrimaryAssetIdRedirects** 或原生代码处理这些修复。**ForceLoadItem** 可以同步加载尚未在内存中的项目以从 **PrimaryAssetId** 转换为 **URPGItem**（由于上面提到的存储预载，在ARPG中通常需要这样操作）。

不同范围定义下的属性集合

> 因为ARPG相对简单，所以只有一个AttributeSet，但是对于某些游戏，更合理的做法可能是设置一个玩家和敌人共用的"核心"集以及一个继承自 **核心** 且包含仅可由玩家使用的额外属性的"玩家"集。

Commit节点存在的必要性

> 当启动某项能力时，调用 **ActivateAbility**，并使用 **CommitAbility** 应用能力的 **消耗（Cost）**（通常是ARPG中的法力）和 **冷却效果（Cooldown）**。调用 **EndAbility** 通知系统该能力已执行完毕。另外两个节点专用于ARPG，通常每个游戏都会根据需要添加新的函数和节点。

原则：不应该将任何配置放在代码中

> 配置的归配置，代码的归代码，把显然是配置的东西写死在代码里是有问题的。要想清楚一个“东西”到底是配置还是代码。

GamePlayTag系统得用起来

> 基于Path的路径标记

有一个笔记，有空学习一下

> https://www.zhihu.com/column/uelearn

蓝图比AL好在哪儿？

> 1.  白线逻辑流正向执行，其它颜色线数据流反向计算依赖按需执行，这一点很值得借鉴  
>     2. Event机制  
>     3. out pin解决分支问题  
>     4. foreach节点解决循环问题（配合Event和Function就很赞）  
>     5. 这是“定义清晰而明确”带来的副作用，它会要求策划去cast，去了解每个节点inPin所需的详细参数类型  
>     6. 蓝图没有上下文的，任何东西，用之前要定义，用的时候要Get（唯一的上下文可能就是self了。。）