# Desktop_Butler_VMC 场景手册

这份场景把 **本地互动、智能体对话、角色表情、肢体表现和桌面活动** 接在一起。它既是一个可继续修改的陪伴场景，也是一份较完整的 WarudoDC 使用实例。

本文对应当前目录里的两份文件：

| 文件 | 内容 |
| --- | --- |
| [Desktop_Butler_VMC.json](Desktop_Butler_VMC.json) | Warudo 场景：资源、角色配置、蓝图和接线 |
| [warudodc-adapter-config.json](warudodc-adapter-config.json) | AstrBot 侧 Adapter 业务配置：API、上下文规则、钩子、工具、剧本与首段预设 |

**阅读基线：2026-11-02。** 场景保存版本为 Warudo **0.15.1**，记录的 Link／Runtime 版本为 **0.2.41／0.10.6**。共 112 张蓝图、39 项资源、4298 个节点、3124 条数据线、2226 条流程线和49个原生表情键。配套配置有99项API、33个LLM钩子、9个工具注册（8个启用）、2条上下文规则、54个导演行为和26个首段预设。

这些数量用来识别这份文件，不代表每项行为的全部组合都已实机验收。本文讲当前文件怎样工作，也保留影响后续维护的设计理由与失败经验；早期日志和图内注释若与当前接线冲突，以实际资源配置、节点和连线为准。

## 目录

- [阅读方式与场景定位](#阅读方式与场景定位)
- [首次加载与接入](#首次加载与接入)
- [资源总览与变换链](#资源总览与变换链)
- [自主活动、明确命令与运动回执](#自主活动明确命令与运动回执)
- [原生追踪、口型与最终骨骼](#原生追踪口型与最终骨骼)
- [全局变量、鼠标与本地交互](#全局变量鼠标与本地交互)
- [表情、注视、注意力与精力](#表情注视注意力与精力)
- [身体表现、待机与说话动画](#身体表现待机与说话动画)
- [配套 Adapter 配置怎样参与场景](#配套-adapter-配置怎样参与场景)
- [99项API完整目录](#99项api完整目录)
- [全部蓝图索引](#全部蓝图索引)
- [如何修改和扩展这份场景](#如何修改和扩展这份场景)
- [已知边界与当前文件中的说明差异](#已知边界与当前文件中的说明差异)
- [保存、重载与维护检查](#保存重载与维护检查)


## 阅读方式与场景定位

第一次使用，先读“首次加载与接入”和“资源总览”。想修改行为，按蓝图原名寻找对应小节；想改智能体怎样理解交互或怎样编排动作，读“配套 Adapter 配置”。图名保留英文，是为了能在 Warudo 里原样找到，不代表需要先理解全部英文术语。

几种频繁使用的概念：

- **Controller（控制器）**：对某类状态拥有最终决定权的蓝图，例如交互是否占用、身体动作由谁持有。其他图通过函数提出申请，不各自修改一套互斥规则。
- **来源与代次**：某一轮表现属于谁、是否仍是当前这一轮。用于避免旧定时结束、旧回执或晚到流程清掉新状态。
- **API／控制事件**：有名字、带类型化字段的一次通知。对端受理只说明接纳投递，动作完成要通过独立结果事件说明。
- **Timeline**：Link内部的回复调度设施，维护一轮回复的片段、语音及表现提示。它不是场景里的另一张控制器图。
- **Director（导演）**：Adapter侧为回复片段选择语调和行为的处理阶段。选择了动作不等于本地一定允许执行。
- **REQUESTED／INTERACTION／BASE**：外部请求表情、本地交互表情、基础表情三个贡献层；本场景采用原生分层合成，而不是整张脸只选一个赢家。

### 谁决定什么

| 部分 | 本场景中的职责 |
| --- | --- |
| AstrBot／Adapter | 主回复、输入收集、语音识别／合成、导演决定、工具调用和业务关联 |
| Link | 对话／控制传输、本地展示记录、录音提交、Timeline和内置音频播放 |
| Runtime | 交互感受、注意力、精力、移动／限位、注视计算、通用计时 |
| 场景蓝图 | 接触识别、交互许可、动作互斥、表情组合、原生动画、原生口型和最终资源映射 |

Runtime不会因精力低就自行拒绝命令，也不根据舒适值直接强制角色“开心”。这些值经现有交互事件提供给智能体参考，最终意愿仍由AstrBot侧决定。本地交互和动作许可负责守住执行边界。

### 四条主流程

```text
鼠标 → Cursor坐标/射线 → Contact或拖拽图 → Interaction Controller许可
     → 本地表情/身体申请 + Runtime模型输入 → Link事件 → Adapter钩子

文字/录音 → Link对话流 → Adapter收集与LLM → 文本/语音/表现计划
        → Timeline释放语调或控制事件 → Emotion/Expression/Performance/Speak

主LLM工具 → Control入站事件 → Autonomy准入 → Runtime运动
         → 命令状态终结 → 对应.result事件 → Adapter工具结果

原生追踪/音频 → 面部与姿态追踪图 → 启动眼睑合成/眼神求解
            → Warudo原生角色表现
```

## 首次加载与接入

### 需要哪些东西

1. 可运行此场景的Warudo环境；保存基线是0.15.1，不把这个版本号解释成全部新旧版本兼容保证。
2. WarudoDC Link与Runtime。场景同时使用两者，尽管两个插件本身可以独立运行。
3. 场景实际使用的原生交互、音频、程序动画及VMC等节点。文件的`plugins`保存了当时加载的37项元数据，**不是37个插件全部必需的依赖清单**。
4. Hanekit的鼠标位置节点与数据变化检测器。保存的插件版本分别为1.1.0和1.1.0；实际使用了屏幕相对鼠标位置、数据变化检测和Boolean变化检测。
5. 与场景引用匹配的角色、动画和音素配置。仓库中的两份JSON不会携带模型、动画库和所有外部素材；缺失资源需在Warudo中重新选择。
6. 需要智能体对话时，另行运行与Link匹配的Adapter、配置LLM及语音服务。仅本地交互不需要每次都调用LLM。

没有同样的角色时，特别需要重新检查接触区域、骨骼、手部IK、49个表情键对应的BlendShape、笑的组合以及音素映射。资源同名不代表模型结构相同。

### 加载场景

关闭Warudo后，将场景JSON放到安装目录的`Warudo_Data/StreamingAssets/Scenes/`，再在Warudo里打开。文件名与场景内部名称都应保留为`Desktop_Butler_VMC`；只改其中一个，曾出现首次能打开、之后无法重新加载／切换的情况。

第一轮先确认角色、相机、原生动画和鼠标交互，再接入网络。透明相机已开启Spout输出；Spout负责把渲染画面交给外部桌面展示，不等于Link的网络通道，也不会替你提供桌面窗口点击穿透或完整输入焦点系统。

场景保存了一些运行后的端口值，例如最近事件ID、上次音频或运动结果。**这些不是应手填的初始参数**：已连接的数据口会被实时数据覆盖，初始化图也会重置各自持有状态。不要把一串保存过的`reply_to`当成永久配置。

### 连接Adapter

Link全局设置不包含在这份场景中，需另行配置。默认分工为：

| 通道 | 示例默认值 | 用途 |
| --- | --- | --- |
| Link Control | `ws://127.0.0.1:39550/` | 控制事件、受理确认与关联 |
| Link对话Stream | `ws://127.0.0.1:39551/` | 对话正文、录音、语音和表现声明 |
| 场景原生VMC接收 | UDP `39539` | 既有标准追踪，不与Control共用生命周期 |
| 可选VBAN | `127.0.0.1:6980` | 发声器明确开启时的旧输出路线，当前关闭 |

若Adapter在Docker中，监听和映射需使Windows上的Link实际可访问。不要把容器内部的`127.0.0.1`和Windows本机地址混为一谈。这里不需要另外启动Audio Host进程，也没有必须打开的39552播放器端口。

### 导入配套业务配置

在Adapter的Pages中选定目标WarudoDC平台，进入“JSON配置转换”，读取同层的`warudodc-adapter-config.json`，勾选需要导入的上下文规则、钩子、工具和剧本，再导入编辑区。先核对并保存API与上下文规则，再保存依赖它们的其他模块。

这个文件是 **`warudodc.adapter-config.experimental` v1业务包**，里面嵌套v2共用API表；不能整包粘进Link的“共用API JSON”框。场景已经带有匹配的API注册表。两端99项API的方向、名字、字段和类型在本次文档核对中一致。

导入合并API／钩子／工具／规则，不自动删除目标平台额外旧条目；剧本则整体替换所选平台的剧本草稿。导入不等于保存，也不会顺便执行模型请求。已有个人配置应先导出备份，再确认差异。

业务包不含Provider凭据、插件全局设置、平台连接与调度覆写、对话历史，也不包含真实音频。导入后仍需选择自己的LLM、ASR、TTS、Director Provider，检查平台端口与输出发声器目标；等待时间、并发和分段策略也以实际平台运行配置为准。

### 上手前应检查的场景值

| 位置 | 当前保存值与调整建议 |
| --- | --- |
| 对话注册表 | 保存的是作者的一段具名对话ID，自动获取历史开启；请按自己要继续的会话选择，不把资源里的旧`SessionId=local`当路由 |
| 麦克风 | 采集与自动提交均开启，设备为作者的UGREEN USB麦克风；先改成自己的设备并按意愿确认这两个开关 |
| 录音阈值 | RMS 0.01、最短起音0.1秒、静音结束0.8秒、单段30秒、暂存4段；不保证适合全部环境 |
| 对话面板 | 输入框与记录均显示，绑定Camera 1，发送`user.input`、接收`assistant.reply` |
| 音频发声器 | `audio.main`、程序音频、3D开启、距离1/30、音量100、非静音、VBAN关闭；绑定角色 |
| 舞台 | 目标范围`(-1.25,-0.25,-0.5)`到`(1.25,1.25,0.5)`；活动外框`(-2,-1,-1)`到`(2,2,1)` |
| 模型 | 感受、注意力、精力与注视都已启用；参数是本场景方案，不等于插件新资源默认值 |

## 资源总览与变换链

### 为什么不是只有一个角色资源

根位移、拖拽偏移、摆动和实际角色跟随分别有持有者。当前固定层级是：

```text
Character Node 0                    ← Runtime移动设施控制的运动根
└─ Character Node 1                ← 背部拖拽的抬起/局部偏移
   └─ Back Drag Swing Anchor       ← 摆动角度及支点补偿
      └─ Character Root           ← 最终角色跟随目标

角色1通过Character Transform跟随Character Root
音频发声器绑定角色1
```

`Character Root`不会因为摆动暂时停止就换父对象。把摆动锚点的父子关系和是否计算摆动捆在一起，会使显示链反复重组。真正负责世界位移的是`Character Node 0`；不要再给角色本身接一套旧位移补间与它抢写。

注视另有独立链：`Look-at Input Anchor`表示已经选中的输入，`Look-at IK Anchor`接平滑结果，子级`Look-at Motion Anchor`接眼神角度投影。`Raw Cursor`位于相机坐标下，不能直接当作角色空间的最终注视点。

### 39项资源逐项索引

| 资源原名 | 作用与关键绑定 |
| --- | --- |
| 新手向导 | 原生向导；不持有本场景的交互或动作权威。 |
| Camera 1 | 主相机、透明背景与Spout输出；面板、鼠标坐标、射线和看镜头目标共享其空间。 |
| 环境 | 保存为禁用且Source为空，不应因旧环境扩展错误强行启用。 |
| 平行光源 | 角色/道具照明；不参与交互和网络。 |
| Character Node 0 | 唯一Runtime运动根，世界位移/朝向入口。 |
| VMC 接收器 | 原生UDP39539标准追踪输入，和Link连接独立。 |
| Character Node 1 | 背部抬起和局部姿态偏移层；父级Node 0。 |
| Back Drag Swing Anchor | 固定在Node 1下的摆动层，Character Root始终在其下。 |
| WarudoDC Runtime 计时器 · 交互阶段 | timer.interaction.phases；持续计时，阶段来自各交互Integer毫秒数组。 |
| Raw Cursor | 原始光标位置，父级Camera 1。 |
| WarudoDC Runtime 计时器 · 肢体表现 | timer.performance；普通/限时身体表现期限。 |
| Character Root | 显示链最终根，Character Transform让角色跟随。 |
| 角色 1 | 模型、49表情键、原生动画/追踪/骨骼配置主体；需本机素材。 |
| WarudoDC Link 对话注册表 | 选择会话，3个文本通道，最多100展示条，加载自动同步历史开启。 |
| WarudoDC Runtime 计时器 · Speak阶段 | timer.performance.speak；原生说话Start/End阶段时间。 |
| Look-at Cursor Anchor | 把原始光标位置变成注视候选，父级Raw Cursor。 |
| WarudoDC Runtime 交互感受模型 | interaction.main：8个Contact部位及各方式偏好。 |
| WarudoDC Runtime 计时器 · 场景启动 | timer.presentation.startup；醒来分阶段。 |
| WarudoDC Link API 注册表 | 61入站/38出站，应用后的目录决定动态端口。 |
| Mouse Click Probe | 左键探针，父级相机；只在允许状态进入接触区域。 |
| Look-at Camera Anchor | 镜头目标，父级Camera 1，不等于所有注视都直接写相机。 |
| WarudoDC Runtime 计时器 · REQUESTED 表情 | timer.expression.requested；外部表情来源期限。 |
| Look-at IK Anchor | 注视设施平滑后的主目标，头/身体与眼睛父目标。 |
| WarudoDC Link 音频发声器 | audio.main，绑定角色，3D开启，程序音频，VBAN关闭，音量100。 |
| Cursor Track | 可见星形光标效果，与是否追视/接触分开。 |
| WarudoDC Runtime 拖拽与旋转感受 | interaction.motion：back_drag/halo_rotate/hand_drag三个逻辑交互。 |
| WarudoDC Link 麦克风 | microphone.main，user.input；设备需覆写，当前采集/自动提交开启。 |
| Look-at Input Anchor | Controller选好的世界目标，Runtime只读跟随。 |
| WarudoDC Runtime 注意力模型 | attention.main；user/cursor逻辑关注，供许可与短看判断。 |
| WarudoDC Runtime 计时器 · 短触后效 | timer.expression.aftereffect；松手后的独立短触表情，不持有交互锁。 |
| WarudoDC Link 预置对话面板 | Camera 1内显示输入/记录，通道user.input和assistant.reply。 |
| WarudoDC Runtime 计时器 · 短看与冷却 | timer.gaze.glance；短看窗口及冷却，本轮节点覆写时长。 |
| WarudoDC Runtime 注视设施 | gaze.main；0.2秒正常/0.35秒回正、0.01秒眼神过渡。 |
| WarudoDC Runtime 精力与休息模型 | energy.main；初始100、低值20/25、静息5秒、恢复2/分钟。 |
| WarudoDC Runtime 计时器 · 点击注视 | timer.gaze.pointer；按住/松开目标尾随，本轮节点覆写时长。 |
| Look-at Motion Anchor | IK目标下的眼神子锚点，接角度投影，而非另一个根运动。 |
| WarudoDC Runtime Locomotion | 角色1、Node 0、共享舞台；根运动和结果，不自动播放动画。 |
| Hand Drag IK Anchor | 牵手的独立手部IK目标，不替代Character Node 0的运动权威。 |
| WarudoDC Runtime 限位舞台 | 共享世界坐标目标盒/活动外框，只约束运动根点。 |

### Runtime计时器为什么保存为0或60秒

八个计时器分别供交互阶段、普通肢体、Speak、启动、REQUESTED、短触后效、短看冷却、点击注视使用。多数资源默认0持续、无阶段；后两份保存60秒模板值。**实际本轮期限和阶段经开始节点覆写**，例如交互`[600,2000]`毫秒、短看和松开停留，而不是看资源模板就断言它们实际等60秒。

计时器只维护时间和本轮ID，Controller仍负责业务来源、占用和结束权限。不要把一个结束通知接到“清空一切”，也不要用“时间到了”代替原生动画确实结束。

## 自主活动、明确命令与运动回执

Autonomy组共7图。名字叫Autonomy，不代表所有输入都来自角色自发意愿；它还承接用户经主LLM发出的明确工具命令。

### 两种模式

| 模式 | 触碰交互 | 明确移动/转向命令 | 自主活动倾向 | 模式刚切换后的姿势 |
| --- | --- | --- | --- | --- |
| 待命，`ActMode=0` | 允许，仍受占用守卫 | 允许 | 低，不由Runtime替智能体做最终决定 | 返回完成后Idle-1 |
| 自由，`ActMode=1` | 不允许 | 允许 | 高；切模式本身不随机下发移动 | 切入时Idle-0 |

待命返回是“从自由状态回归位置与朝向后再完成模式切换”，不是按键一按就伪造已到达。单独`idle.return_normal_pose`只恢复身体姿态，不等于切回待命模式。

<a id="graph-autonomy-controller"></a>

### Autonomy Controller

本图统一业务准入与目标交接，提供`WarudoDC.Autonomy.CanSubmit`和`WarudoDC.Autonomy.ResolveTarget`。它要求本地Autonomy启用、模式属于待命或自由、无交互占用、未在待命返回中，且未被启动流程阻止。**没有“只能自由模式执行工具”的条件。**

`RequestedTarget`经许可交接为`ResolvedTarget`，并用本次请求序号确认交接。舞台实际夹取和运动积分属于Runtime，不在Controller中重复实现。

本图还根据交互与受保护身体占用停止冲突的Runtime运动，并协调是否允许界外恢复。禁用图时撤回自己的恢复许可。它不会自行决定下一步去哪里，也不在没有用户/LLM命令时凭空生成路径。

<a id="graph-autonomy-state-tools"></a>

### Autonomy State Tools

监听`autonomy.mode.free`与`autonomy.mode.standby`，经过统一`WarudoDC.Mode.Request`入口切模式，并通过`autonomy.state.result`返回业务状态。`PendingStandbyEventId`保留需要等待归位完成的那次工具身份；另一个在途待命请求不能覆盖它。

自由切换可在本地状态受理后确认，但不冒充移动完成。待命请求如果当前已待命可以直接确认；需要归位则等待`WarudoDC.Mode.StandbyCompleted`再结算。失败、拒绝、禁用和重新加载要清理等待者，不在重启时补发旧工具结果。

这里还提供`WarudoDC.Thinking.IsActive`，以及集中处理模式姿势的`WarudoDC.Mode.ResetPresentation`：自由用Idle-0、返回待命完成用Idle-1。思考的真正身体与表情在Thinking Pose中，不是状态工具图直接播放。

<a id="graph-return-to-standby"></a>

### Return To Standby

`FreeExitPending`进入返回流程后，冻结本次关联，先向原点提交移动，必要时选重力路径，移动完成后再提交零朝向转向。移动和转向各保存自己的Runtime命令ID，仅匹配本轮的正常完成才能进入下一步。

当前选择路径时使用高度差约0.02的判定。正常返回不再经过旧Locomotion Controller；原生补间退出，由当前Runtime移动设施执行。移动或转向取消／被替代不能当作返回成功，失败会清理Playing和两个命令ID，不把模式强行标为待命。

当移动与转向都完成后，清除返回中状态、提交`ActMode=0`并通知状态工具图结算。图启用／禁用时撤销自己持有的返回状态。不要把观察器的命令ID筛选接成永远不更新的旧缓存。

<a id="graph-move-to"></a>

### Move To

监听`autonomy.move_to`的`target: Vector3`。目标是世界坐标，先保存当前事件ID，再调用统一准入和目标交接。根据实际高度差选择直线或重力请求，交给Runtime后保存命令ID。

受理出口不会立刻发完成回执；观察命令终态后，先冻结结果对应的原事件ID、位置与状态，清理活动关联，再发送`autonomy.move_to.result`，其关联事件ID指向原请求。这样同一入口新命令不会被旧回调覆盖。

本图每个入口只维护最近一项活动关联，不建立独立离线队列。`LastResult/LastReason/LastError`供诊断，不应作为新的命令来源。目标超出舞台会由设施夹取；工具提示中的参考范围不是实际舞台的替代。

<a id="graph-move-by"></a>

### Move By

监听`autonomy.move_by`的`delta: Vector3`。它先采样当前运动根位置，再加位移增量，形成明确世界目标后走同一目标交接和运动流程；不是直接把delta当作绝对目的地，也不是持续追逐一个会动的目标资源。

结果发往`autonomy.move_by.result`，字段为实际位置与终态。其身份、来源失效清理及失败处理与Move To一致。中途换目标可由Runtime保留速度连续性，但原生动画仍由Locomotion Animation按实际状态选择。

<a id="graph-rotate-to"></a>

### Rotate To

监听`autonomy.rotate_to`的`yaw: Float`，经许可后提交绝对朝向，位置不改变。结果为`autonomy.rotate_to.result`中的实际根旋转与状态，并关联原事件。

**当前存在范围说明不一致：** Adapter工具说明写0～360度，但本图实际在提交前把yaw夹到-180～180度，而且夹取上下界没有其他输入覆盖。因此大于180度的请求当前会被截到180，不能据工具说明声称它可以原样转到270度。本手册记录事实，不顺便改场景。

<a id="graph-rotate-by"></a>

### Rotate By

监听`autonomy.rotate_by`的`delta_yaw: Float`，当前也夹到-180～180度，再触发相对转向入口。与Rotate To的区别是基于受理时朝向增加角度，而不是绝对朝向。

结果为`autonomy.rotate_by.result`。两个转向图都从运动根的原生旋转读取最终值；JSON里结果口偶然保存的旧向量不能当作未来回执内容，因为它们有实际数据线。

### 运动执行与动画分开

Runtime资源绑定角色1、运动根Character Node 0、共享限位舞台；轴映射XYZ、世界+Y为上方向，当前转向速度240度/秒、重力9.81、跳高0.3、加减速度4/6、落地停留0.3秒。

移到目标、落地停稳、动画播完和工具回执送达是四件事。原地／短跳与远距离跨步由Locomotion Animation用预计腾空水平峰值速度选择，不能只看请求速度或把所有重力路径都选跨步。

工具进入后立即收到Control受理不代表运动结束；业务结果在实际本地终态时返回。Adapter如果超时，并不能据此推断角色没有移动。禁用入口图会取消它持有的命令，结果不能跨连接误认成另一轮工作。

### 运动改造中的教训

**双执行者比参数不自然更应先排查。** 初接Runtime时旧补间和角色跟随链仍能写位移，导致抖动与朝向异常。后来让移动设施只写运动根、行动图直接接命令节点，清掉旧Locomotion执行组；动画消费状态，而不是另造运动。

**取消的零值必须真的到达原生端口。** 旧版object选择器中直接保存的Float字面量没有按预期恢复，null又不会覆盖下游数值，所谓取消反而以正时长补间回旧位置。修正使用有类型的原生输入；当前不要从旧试验复活那套取消选择器。常数应直接用原生自变量节点，不用无意义运算制造。

**朝向不能依赖任意反向向量的完整欧拉角。** 曾用FromToRotation写整套欧拉角，沿反向Z移动时带来180度滚转。固定世界Up并明确只改变水平朝向后改善；离线模型若把原节点偷换成理想atan2，会掩盖真实问题。

**正确过滤也会被缓存挡住。** 待命返回曾因观察器读到旧命令ID而始终过不了完成守卫；插件修复原生惰性输入的求值后，场景也需在移动、转向各自的完成分支明确匹配。不能删掉关联校验来“让它能跑”。

## 原生追踪、口型与最终骨骼

动作捕捉组的两张图沿用标准Warudo追踪能力。它们不属于新的自定义VMC控制协议；即使没有外部VMC数据，面部图内的原生程序动画、音频与眼神执行仍可能参与角色表现。

<a id="graph-面部追踪---vmc"></a>

### 面部追踪 - VMC

读取VMC接收器的面部／骨骼数据，根据开关选择有效输入，再合并原生头眼生成和Lip Sync输出。BlendShape经过平滑与最终设置写入角色，骨骼分支合并默认姿态与眼睛求解结果。

本图中的原生唇形节点当前选择`loopback`系统音频、立体声、阈值25，使用所选音素配置及A/I/U/E/O映射。这不是Link专属播放器PCM入口：其他系统声音也可能影响嘴型。换模型必须检查这五个嘴部BlendShape名字，不能只复制音素列表。

眼神执行通过原生头眼节点，头部权重设为0、眼睛权重交由当前Gaze管线管理；角色普通头部／身体注视仍看父目标。停用时恢复接管前的原生眼睛通道，避免两套系统同时抢眼骨。

启动眼睑的`WarudoDC.Presentation.ProcessStartupEyes`必须插在最终BlendShape管线中：原生函数接收字典、做闭眼／睁眼叠加、再返回继续写入。只在另一张图单独改眼睑会被这条每帧追踪输出马上覆盖。这也是“启动时数值正确但眼睛不闭”的真正组合问题。

<a id="graph-姿态追踪---vmc"></a>

### 姿态追踪 - VMC

读取标准VMC骨骼旋转、位置和根变换，经原生权重选择、平滑、合并与覆盖节点进入角色。它与Runtime根位移的边界是：追踪处理姿态，实际场景根运动只有明确选定的执行者。

切换或停用时走原生重置骨骼分支。不要因为有VMC字样就把对话Control端点、长文本、音频或嘴型也塞进这个UDP接收器；该接收器端口独立保存为39539。

### 原生音频与骨骼实验的边界

原生Lip Sync已经能从系统回录工作，但这不等于播放器专属入口完成。实验曾发现空但非null的BlendShape字典可以进入原生链，null则可能令生成／应用无效，因此本图保留实际输入合并。

没有使用反射、私有API或自制嘴型替代原生系统。为摆动增加的Runtime骨骼偏移读取实验也已经撤回；当前场景使用原生骨骼节点。动作、追踪、表情和空间根变换可能同时影响视觉，不能用其中一条数值在变证明最终画面正确。


## 全局变量、鼠标与本地交互

这一组蓝图把桌面鼠标转成场景中的位置与接触，再由唯一的交互控制器决定谁可以开始、何时进入下一阶段、谁可以结束。
鼠标命中不等于交互获准，交互获准不等于网络消息已送达，网络受理也不等于角色已经完成表现。
以下说明以这份保存场景的变量、参数和连线为准；保存下来的鼠标坐标、序号、事件关联字符串只是运行预览，不是需要手动复制的配置。

### 先认清三套状态

| 状态 | 保存位置 | 回答的问题 |
| --- | --- | --- |
| `IsMouseHeld`、`IsDragging` | Mouse Drag State | 左键是否按住，移动是否已越过拖拽门槛？ |
| `DragTarget` | Drag Target State | 这次鼠标捕获的是背部、光环，还是哪只手？ |
| `InteractionOwner`、`InteractionPhase` | Interaction Controller | 哪一种交互已经得到本地许可，目前进行到哪一阶段？ |

`DragTarget` 与 `InteractionOwner` 分开存在，是因为“鼠标抓住了什么”和“谁拥有整个交互生命周期”不是同一件事。
左右手有两个拖拽目标，却共用同一种 Hand Drag 交互身份；Contact 触碰则通常不需要写拖拽目标。
具体交互图不能通过直接改 `InteractionOwner` 来绕过许可；当前权威入口是 Controller 的原生函数。
Runtime 感受资源只维护接触与数值，不会替 Blueprint 抢占交互、选择表情或操纵角色。

<a id="graph-constants"></a>

### Constants：全场景共用的符号字典

Constants 当前启用，图内只有说明节点，真正被使用的是图变量。
它没有逐帧入口，也不主动发消息；其他图通过变量名读取身份、模式和配置基准。
修改一个常量会影响所有读取它的图，所以应先确认消费者，而不是仅改正在看的那个分支。

与本章直接相关的身份如下。

| 常量族 | 当前含义 |
| --- | --- |
| `ACT_MODE_STANDBY=0`、`ACT_MODE_FREE=1` | 待命与自由活动；模式总数为 2 |
| `INTERACTION_NONE=0` | 当前没有交互拥有者 |
| `INTERACTION_BACK_DRAG=1`、`INTERACTION_HALO_ROTATE=2` | 背部拖拽、光环区域旋转 |
| `INTERACTION_CONTACT_NOSE=3`、`INTERACTION_CONTACT_FOREHEAD=4` | 鼻子、额头触碰 |
| `INTERACTION_HAND_DRAG=5` | 左右手共用的牵手交互 |
| `INTERACTION_CONTACT_BREAST=6`、`…BELLY=7` | 胸部、腹部触碰 |
| `INTERACTION_CONTACT_THIGH=8`、`…LEG=9`、`…BUTT=10` | 大腿、小腿、臀部触碰 |
| `INTERACTION_CONTACT_HEAD_TOP=11` | 独立于额头的头顶触碰 |
| `DRAG_TARGET_NONE=0`、`…BACK=1`、`…HALO=2`、`…LEFT_HAND=3`、`…RIGHT_HAND=4` | 鼠标捕获目标 |
| `GAZE_MODE_INHERIT=-1`、`…ANIMATION=0`、`…CAMERA=1`、`…CURSOR=2`、`…GLANCE=3`、`…CLICK=4` | 继承、动画、看镜头、跟鼠标、短看、点击目标 |

Contact 的默认阶段表为 `INTERACTION_PHASE_POINTS_DEFAULT=[600,2000]`，单位是毫秒。
实际开始交互时，具体图仍会把自己的阶段表作为参数交给 Controller；不是所有交互都共用这两个时间点。
屏幕换算使用 `SCREEN_WIDTH=2560`、`SCREEN_HEIGHT=1440`；分辨率变化首先检查这里。
表情、基础动画与叠加表现也有各自的整数常量族；这些身份属于不同用途，不能因数字相同就混作一个来源。
例如交互来源 3 是鼻子，基础动画来源或表现来源中的 3 不应据此被解释成鼻子。

<a id="graph-variables"></a>

### Variables：活动模式、开关与手动入口

Variables 当前启用，集中保存 `ActMode`、`EnableCursorTrack`、`EnableLookCursor` 与 `FreeExitPending`。
保存值是待命模式、光标轨迹开启、鼠标注视开关关闭、没有待处理的自由模式退出。
这里的保存值说明本文件保存时的状态，不保证打开后每一时刻都保持该值。

三个快捷键是本地手动入口。

| 快捷键 | 处理过程 | 下游影响 |
| --- | --- | --- |
| Ctrl+Alt+C | 翻转 `EnableCursorTrack` | Cursor Track Switch 开关星形轨迹资源 |
| Ctrl+Alt+S | 翻转 `EnableLookCursor` | 注视体系决定是否采用鼠标跟随偏好 |
| Ctrl+Alt+X | 待命时清退出标记并进入自由；自由时设置一次退出请求 | Autonomy 体系安排返回待命 |

切回待命采用 `FreeExitPending` 请求，是为了让正在进行的自由活动按自己的收尾流程退出。
重复请求不会持续叠加同一退出意图；不能把设置退出标记理解为角色已经回到待命位置。
正式的同步入口为 `WarudoDC.Mode.Request(Mode:int)->bool`，它检查启动阻塞、有效模式及退出中的状态。
请求自由模式时清退出标记再写模式；请求待命时，已待命可直接成功，否则询问 `WarudoDC.Autonomy.CanSubmit` 后才设置退出请求。
`WarudoDC.Mode.InitializeStandby()` 用于初始化，依次清 `FreeExitPending` 并写 `ActMode=0`。
手动快捷键保留自己的直接处理链，不能把正式函数的全部启动检查自动归到快捷键上。
日常调整主要是两个布尔开关；修改模式值、退出流程或函数类型属于行为变更。

### 从屏幕位置到可触碰的射线

这一段按数据流阅读最容易：Mouse Pos to Cursor Pos → Cursor Movement → Mouse Click Ray Probe → 各接触图。
Mouse Drag State 同时观察鼠标，Drag Target State 保存已捕获目标；它们不负责选择触碰事件或发送 API。
Cursor Track Switch 只控制可见轨迹，关闭轨迹不会自动关闭鼠标输入或交互检测。

<a id="graph-mouse-pos-to-cursor-pos"></a>

#### Mouse Pos to Cursor Pos：屏幕坐标换算

此图当前启用，入口为每帧 `OnUpdate`。
它读取相对屏幕的鼠标位置，当前偏移为 X=0、Y=13，再除以 Constants 的屏幕宽高。
归一化后的 X、Y 都限制到 0～1，鼠标跑出屏幕边界时不会把游标无限推远。
X 再减 0.5；Y 使用 0.5 减归一化坐标，完成屏幕坐标到场景朝向的转换。
当前水平跨度乘数为 `0.936195552`，垂直跨度乘数为 `0.52661`，构成 Z=2 的 `RowPos`。
`RowPos` 是场景保留的实际变量名，虽然拼写近似 Raw，也不要自行更名后留下失效读取。
另一条数据支路读取 Raw Cursor 的世界位置，按 `LookAtOffsetAmplifier=1` 放大 X、Y，生成 Z=0 的 `LookAtPos`。
每帧流程先写 `RowPos`，再写 `LookAtPos`，供位置同步图与注视体系消费。
可调项是屏幕宽高、屏幕偏移、两个跨度乘数及注视放大量；它们需要与当前摄像机和桌面布局一起校准。
这不是通用相机反投影，也没有从显示器自动推导所有摄像机参数。

<a id="graph-cursor-movement"></a>

#### Cursor Movement：把变量放到资源上

此图当前启用，入口是 `RowPos` 的数据变化检测。
它从上一图读取 `RowPos`、`LookAtPos`，先设置 Raw Cursor，再设置 Look-at Cursor Anchor。
两个位置写入目前都没有过渡时长，所以这里不增加拖尾或注视平滑。
Raw Cursor 是 Camera 1 的子资源；它既给射线方向，也给后续拖拽坐标使用。
Look-at Cursor Anchor 是注视目标辅助点；最终头眼动作由注视链路执行。
此图不计算屏幕坐标、不发交互，也不取得占用权。
可调的是资源绑定和必要的位置过渡；更改过渡会改变输入与可见反馈的时序，应单独观察。
流程由 `RowPos` 变化触发，而不是独立监听 `LookAtPos`；调整注视放大量后可移动鼠标检查目标实际更新。

<a id="graph-cursor-track-switch"></a>

#### Cursor Track Switch：可见轨迹开关

此图当前启用，读取 Variables 的 `EnableCursorTrack`。
布尔变化检测的真、假出口分别把 Cursor Track 的 `Enabled` 写成 true、false。
当前 Cursor Track 是挂在 Raw Cursor 下的星形粒子资源，所以只需切换启用即可随游标显示。
这里不移动 Raw Cursor，也不改变探针、点击、牵手或 Contact 的许可。
可调项是被控制资源，以及是否采用这组视觉轨迹；资源开关的过渡当前为 0 秒。
排查“没有星星但仍能碰到角色”时，先区分视觉开关与输入链，不必因此改交互控制器。

<a id="graph-mouse-click-ray-probe"></a>

#### Mouse Click Ray Probe：把左键变成接触探针

此图当前启用，每帧从 Raw Cursor 的世界位置求“局部前方指向该位置”的旋转，再写到 Mouse Click Probe。
Mouse Click Probe 是 Camera 1 的子资源；左键按下时检查 `ActMode`，待命分支把探针移到本地原点。
左键松开直接把探针移到 `(0,0,-10)`，从角色附近撤走，令接触图收到退出。
接触图把探针解释为沿 Z 轴的长胶囊：中心 `(0,0,4)`、半径 0.018、总高度 8。
因此它是一条有厚度的鼠标接触射线，不是摄像机画面上的某个小球。
本地原点与沿相机前方的方向配合，让鼠标位置能覆盖角色不同深度的接触区域。
可调项是探针长度、半径、资源父级及撤离位置；半径增大会更容易同时穿过相邻部位。
当前保存结构有一项需注意：模式变化检测之后的第二个整数 Switch 没有连接实时 `ActMode`，输入保存为 0。
因此不能把“切到自由模式会自动撤回仍按住的探针”当作已成立行为；左键松开的直接撤回连线仍然存在。
这里只报告该结构边界，没有修复场景；Drag Target State 自己的模式清理是另一条完整链路。

<a id="graph-mouse-drag-state"></a>

#### Mouse Drag State：按住与拖动分开判断

此图当前启用，左键按下依次写 `IsMouseHeld=true`、`IsDragging=false`，并记录当时的 `RowPos` 为 `DragStartPos`。
左键松开依次清除这两个布尔状态。
鼠标位置变化时，只有仍按住且尚未成为拖拽，才计算当前位置与起点之差的向量长度。
长度达到 `DragStartThreshold=0.003` 后，才把 `IsDragging` 升为 true。
成为拖拽后，即使又移回起点，也不会在本次按住中自动降回“未拖拽”；松开才重置。
这样一次按住只跨越一次启动门槛，不会在门槛附近反复启动和停止。
门槛单位来自换算后的场景游标坐标，不是 0.003 像素，也不是真实力或拉力。
背部拖拽、牵手与旋转读取它；Contact 的长短触则由时间阶段判定，不能用它代替。
可调项主要是拖拽门槛，需同时考虑屏幕到场景跨度，避免更换分辨率后手感突然变化。

<a id="graph-drag-target-state"></a>

#### Drag Target State：保留本次抓取对象

此图当前启用，保存 `DragTarget`，初始/空闲值为 0。
背部、光环和左右手图只有获准后才写入对应目标。
本图在左键松开时清为 `DRAG_TARGET_NONE`；`ActMode` 改成自由或其他未识别值时也清空。
与探针图不同，这里的模式 Switch 输入确实连接了实时 `ActMode`。
目标清空的变化会被具体拖拽图观察，具体图再向 Controller 请求结束自己拥有的交互。
它不直接写 `InteractionOwner`，也不直接停动画、关闭 Runtime 模型或发取消消息。
可调的是 Constants 中的目标身份及消费者映射；不要把当前保存的目标值当成开机时应保留的抓取。

<a id="graph-interaction-controller"></a>

### Interaction Controller：唯一的交互许可和阶段权威

Controller 当前启用，所有 Contact、背部拖拽、牵手及光环旋转都通过它取得许可。
它独占本地交互的 Owner、阶段表、阶段推进、交互代次、交互视线模式和阶段通知去重。
具体图可以检测鼠标、执行自己负责的动作、发送自己的 API，却不能同时再维护一份占用许可。
Runtime 计时器提供时间事实；Controller 将计时阶段解释为业务阶段，这两层仍各自有明确职责。

#### 同步函数的用途与顺序

| 函数 | 调用者传入 | Controller 做什么 |
| --- | --- | --- |
| `WarudoDC.Interaction.TryBegin` | `Source:int, GazeMode:int, PhasePoints:int[]` | 空闲、来源有效且启动未阻塞时先申请本次计时；成功后提交视线、阶段表、Owner并返回 true |
| `WarudoDC.Interaction.IsOwner` | `Source:int` | 仅当非 0 来源等于当前 Owner 时返回 true |
| `WarudoDC.Interaction.CanUpdate` | `Source:int, FirstTarget:int, SecondTarget:int` | 同时检查所有权、正在拖拽以及目标匹配 |
| `WarudoDC.Interaction.End` | `Source:int` | 只允许当前非 0 拥有者清视线和 Owner，再执行统一提交 |
| `WarudoDC.Interaction.OwnershipCommitted` | 无参数 | 统一处理表情归属、递增代次、初始化或清理阶段 |
| `WarudoDC.Interaction.Phase.Notify` | 无参数 | 进入新的函数求值边界，读取提交后的状态，去重并广播 |

这些是 Warudo 原生同步函数，不是网络请求，也不是“写序号等下一帧发现”的异步信箱。
多张具体图在调用前把 bool 结果默认清为 false；若权威函数没有执行，就不会沿用上次 true 继续获取。
`TryBegin` 会在获得 Runtime 计时器的 Accepted 后才设置交互状态；计时申请拒绝则返回 false。
这防止“已经占住交互，但没有任何有效阶段时基”的半启动状态。
它不排队，也不抢走已有 Owner；同时命中多个区域时，后来的申请会被拒绝。
所有权结束以后会递增交互代次，因此不要把当前保存序号当成固定协议版本或用户要填的值。

#### 阶段如何推进

本图绑定的专用资源是 `timer.interaction.phases`，持续模式、关闭循环。
每次开始覆写本轮毫秒阶段数组，并保存返回的 `PhaseTimerId`；计时器资源面板中的空阶段表不是本轮实际阶段表。
获得 Owner 后，Controller 进入 `Phase=1`，先把本轮前一阈值设为 0，再发阶段通知。
观察到 Runtime 的 StageChanged，且活动交互代次仍与当前代次一致，才提交下一业务阶段。
业务 `Phase` 等于计时阶段加一；`InteractionPhasePreviousPointMs` 取刚跨过的实际阈值。
Contact 因而是 0 毫秒 Phase1、600 毫秒 Phase2、2000 毫秒 Phase3。
三类持续拖拽则是 0 毫秒 Phase1、3000 毫秒 Phase2、15000 毫秒 Phase3、30000 毫秒 Phase4。
`DurationMs` 是阶段阈值参数，不是网络送达耗时，也不保证等于松手时实际经过的毫秒数。
结束交互时取消对应计时、清计时 ID、恢复默认阶段表，设 Phase0，再通知下游。
图禁用也有按 ID 取消路径；每帧还检查计时是否存在/活动，丢失时请求结束当前 Owner。
观察事件受本轮计时 ID 与交互代次约束，旧一轮的计时通知不能借此推进新一轮。

阶段广播名为 `WarudoDC.Interaction.PhaseChanged(Source:int, Phase:int, Serial:int, DurationMs:int)`。
各接收图用自己的整数常量筛选 Source，匹配才做自身业务；Controller 不枚举每一张具体 Contact 图。
中央先记下最近通知的 serial+phase，再广播；同一代次同一阶段重复通知不重复发业务事件。
同名 void 定义在这里有意充当广播接收者；bool 权限函数则不能照搬成多定义投票。
接收图禁用时不会执行，也不会在重新启用后补发曾经错过的阶段。
可调的业务阈值放在具体图 TryBegin 参数；应保持递增、合法，并同步核对阶段分支含义。
Controller 内部版本变量用于识别迁移状态，不能拿来当作调整手感的旋钮。

<a id="graph-interaction-context"></a>

### Interaction Context：给已有交互消息补充上下文

此图当前启用，提供三个无参数数据函数，供 Contact 和持续拖拽发送节点读取。
`WarudoDC.Context.Energy()` 从精力资源的 `current.energy` 读取数值。
`WarudoDC.Context.EnergyState()` 只有精力和低精力字段都可用才返回 normal 或 low，否则返回 unavailable。
`WarudoDC.Context.ActivityMode()` 把 ActMode 的 0、1 转成 standby、free，其他值返回 unknown。
启用和禁用时会依次初始化函数默认结果：能量 0、状态 unavailable、活动模式 unknown。
所以缺少有效模型时，发送方仍能得到有含义的不可用标记，而不是把预览中的旧精力值当成实时事实。
它不取得交互权、不修改精力、不决定休息，也不另发一条“上下文快照”网络消息。
输出由既有交互事件携带；同一次交互在不同阶段可读到不同的精力和模式。
维护时可调整资源绑定或显式字符串映射；不要将这些上下文函数改成强制表现指令。

<a id="graph-interaction-expression-bridge"></a>

### Interaction Expression Bridge：把许可后的表情提交写入表情体系

此图当前启用，定义 `Apply(Source,Expression)`、`ClearApplied()`、`ClearSource()` 三个薄桥接函数。
上游 Controller 的 `WarudoDC.Expression.Interaction.Submit` 先检查来源非 0 且等于当前 Owner，再调用 Apply。
Apply 依次写来源、交互表情代码、active=true，最后递增交互表情 serial，供 Expression Controller 消费。
ClearApplied 先清 active 与来源，再提交新的 serial；ClearSource 只清过时来源，不额外假造一次表情变化。
桥接图禁用时也有清 active/source/serial 的收尾链。
这份拆分保留“Controller 许可，Bridge 写槽，Expression Controller 合成”的职责方向。
它不直接选 BlendShape、不控制嘴型，也不以自己的一套优先级替换场景的原生表情合成。
普通交互表情仍属于 INTERACTION 槽；短触后效由各 Touch Reaction 图通过表情体系请求。
可调内容是表情代码映射及资源体系中的表情配置，不应绕过来源检查直接调用内部 Apply 取得权限。

### Contact 公共流程：短触、长触与取消

本场景有八张 Contact 图，当前全部启用，均为 `ContactSlotVersion=4` 的同类流程。
它们分别检测部位、保存自己的方式与本次长触事件关联、发送各自 API；不同部位不共用这一私有缓存。
它们都以 Mouse Click Probe 的长胶囊为发送端，以人体骨骼附近的自定义接收体为接收端。
接收配置当前面向任意场景角色，所以更换或增加角色时要核对命中范围，不能默认只有某个角色会响应。

#### 按下、分类、发送的实际顺序

1. 接触进入时先默认拒绝结果，再调用 Controller.TryBegin，传本部位身份、视线模式与 `[600,2000]`。
2. 仅获准后清空本图 `LongEventId`，设初始短触方式，并对 Runtime 调用 `BeginPending`。
3. 同一获准链用 `WarudoDC.Performance.Contact.Signal(Source,1)` 请求即时接触表现，不等两秒后才反馈。
4. 到 600 毫秒，PhaseChanged 的本部位 Phase2 分支把方式切成长触，并显式 `UpdateTouch`；此处不发 stroke 特殊事件。
5. 到 2000 毫秒，Phase3 再确认长触方式，先更新 Runtime，再发送本部位 `.stroke`。
6. 本地 OnSubmitted 捕获本次 stroke 的 EventId 到 `LongEventId`，供本次后续取消关联。
7. 接触退出先询问 IsOwner，再按当前 Phase 结算；非拥有者不会补造本次业务事件。

| 松开时的阶段 | 实际含义 | 业务发送与表现 |
| --- | --- | --- |
| Phase1，未满 600 毫秒 | 短触 | 确认短触方式、更新感受，发 Signal2 短触收尾，再发 `.touch` |
| Phase2，600～2000 毫秒 | 已经是长触，特殊事件尚未产生 | 发 `.stroke.cancel`，不补发 touch；本轮关联通常为空 |
| Phase3，2000 毫秒起 | 长触特殊事件已到触发点 | 发 `.stroke.cancel`，关联本次已提交的 stroke；不补发 touch |
| 非拥有者或其他阶段 | 不属于本图的有效结算 | 只走安全结束，不制造短触或特殊事件 |

因此“600 毫秒后才算有效短触”和“2 秒才开始长触”都不是当前行为。
事件名 `.stroke` 是沿用的业务命名，鼻子的实际长触方式是 press；不能仅凭名称推断动作测量方法。
具体分类仍由 Blueprint 的时长规则给出，Runtime 不根据部位名、鼠标速度或持续时间自行猜方式。

取消后，发送节点的本地提交或本地拒绝都会继续 Signal3 与 Controller.End，再释放 Runtime 接触。
这里不等待网络 ACK、LLM 回复或 TTS 播完，避免断线时把角色永远锁在交互中。
2 秒前没有本次 stroke，空关联是正常分支；不能拿上一轮 EventId 来“补齐”关联。
2 秒后也只有本地提交成功才捕获新 ID；网络受理不参与这次缓存写入。
目标事件若已结算，取消不代表能够撤回既成回复；取消事件自身还可按 Adapter 的独立钩子策略处理。
图禁用先发 Signal4 取消表现，再检查是否仍拥有交互；长触阶段仍走取消链，短触阶段不伪造一次普通松手事件。
重新启用不会重放上次的接触或阶段，重复松开也不应重新结算同一 Owner。

#### Runtime 感受如何进入已有 API

八个 Contact 绑定 `interaction.main` 的独立条目；背部、光环、牵手绑定另一资源 `interaction.motion`。
`BeginPending` 让接触先存在，负荷与适应可持续累积，待 Blueprint 确认方式后再归因相应舒适倾向。
`UpdateTouch` 明确提交当前方式；仅修改节点下拉框不等于调用了一次 Update。
读取节点只读本条目的 `.load`、`.adaptation`、`.comfort`，不会推进时间、开始接触或修改模型。
发送字段为 `interaction_serial`、`feeling_available`、`feeling_load`、`feeling_adaptation`、`feeling_comfort`，再附能量与活动模式上下文。
三个感受数值分别表达累积负荷、适应程度和舒适倾向；舒适值不是 Character Emotion，也不是必须执行的表情命令。
`feeling_available` 当前从 load 读取节点的 HasValue 接出；不可用时应忽略默认 0，不能把 0 解读成“确定毫无感受”。
Sequence 保证先更新 Runtime 再读数发送；感受节点拒绝不会阻断后面的交互消息与原有清理。
所有这些发送都复用已有 `.touch/.stroke/.stroke.cancel` 或阶段 API，没有逐帧发送感受，也没有新增感受快照 API。
当前节点均 `OverrideStrength=false`，所以采用资源条目 BaseStrength；端口旁看到的 Strength=0.5 不是一律覆盖配置的值。

#### 八张 Contact 的身份、方式与主要差异

下表的偏好是当前示例预设，表示方式偏好，不保证持续接触时舒适读数永远为该值。
过量、适应、残留恢复仍会影响实际结果；这些是角色模拟参数，不是实际生理测量。

| 图 | Source / TouchKey | 短触 → 长触方式（偏好） | GazeMode |
| --- | --- | --- | --- |
| Contact Head Top | 11 / `head_top` | poke(-0.35) → stroke(+0.8) | 2，跟鼠标 |
| Contact Forehead | 4 / `forehead` | poke(-0.6) → stroke(+0.7) | 2，跟鼠标 |
| Contact Nose | 3 / `nose` | poke(-0.7) → press(-0.8) | 2，跟鼠标 |
| Contact Breast | 6 / `breast` | short(-0.5) → long(0) | -1，继承 |
| Contact Belly | 7 / `belly` | short(-0.5) → long(0) | -1，继承 |
| Contact Thigh | 8 / `thigh` | short(-0.5) → long(+0.3) | 2，跟鼠标 |
| Contact Leg | 9 / `leg` | short(-0.3) → long(+0.3) | -1，继承 |
| Contact Butt | 10 / `butt` | short(-0.5) → long(0) | -1，继承 |

除鼻子外，当前各 Contact 条目敏感度 1、基础强度 0.5、积累 30 秒、适应 10 秒、恢复 15 秒、耐受阈值 0.8。
鼻子敏感度为 1.2、积累时间为 5 秒，其余这组时间与耐受参数相同，因此它会更快累积负荷。
Contact 图里的可调阈值、资源中的感受参数、Touch Reaction 图中的视觉后效时长分别属于三个不同层次。
本章的 Contact 不保存 ShortReactionSeconds；需要调短触可见停留时间时，到对应 Touch Reaction/后效体系查找。

<a id="graph-contact-head-top"></a>

#### Contact Head Top：头顶区域独立于额头

入口为探针接触头骨附近的盒体，偏移 `(0,0.135,-0.005)`，尺寸 `(0.18,0.07,0.18)`。
它用 Source11 申请 Controller，走公共 600/2000 毫秒流程，读取 `head_top.*`。
输出前缀为 `interaction.contact_head_top`，对应 touch、stroke、stroke.cancel 三类请求。
表现信号仍通过公共函数传入 Source11，由对应头顶表现图筛选，不借用额头图的私有状态。
头顶更高、略靠后的范围与额头前方胶囊分开，是增加此图而不扩张额头碰撞体的原因。
可调项是盒体位置/尺寸、头顶 poke/stroke 偏好、视线模式与本图阶段数组。
换角色时先看头顶盒体与光环盒体的间隔；同时命中时仍由 Controller 首次受理决定唯一 Owner。

<a id="graph-contact-forehead"></a>

#### Contact Forehead：额头短戳与抚摸

接收体绑定头骨，偏移 `(0,0.075,0.05)`，为 X 向胶囊，半径 0.03、高度 0.18。
入口命中后用 Source4、鼠标注视模式申请；获准即时表现，随后按公共阶段切换 poke/stroke。
输出为 `interaction.contact_forehead.touch/.stroke/.stroke.cancel`，感受只读 `forehead.*`。
额头的短戳与抚摸偏好一负一正，适合观察“同一部位共享负荷和适应，但交互方式可不同”。
短触收尾交给 Touch Forehead Reaction，持续接触清理由 Controller 与表情体系完成。
可调项为额头胶囊、两种方式偏好与资源积累时间；不要通过修改事件名来改变方式。
旧实机证据覆盖了短触、特殊事件前松手、特殊事件后松手三类交付，未测得每次按下/松开的精确物理时刻。

<a id="graph-contact-nose"></a>

#### Contact Nose：事件叫 stroke，方式实际是 press

接收体同样绑定头骨，偏移 `(0,0.02,0.08)`，使用半径 0.02 的球，覆盖鼻尖附近。
它以 Source3 申请，当前跟随鼠标注视；短触为 poke，600 毫秒起为 press。
输出保留 `interaction.contact_nose.touch/.stroke/.stroke.cancel`，没有另造 `.press` API。
方式字段连到 Runtime，因此持续按鼻子不会误用额头的正向 stroke 偏好。
读取 `nose.*` 并转交 Touch Nose Reaction；鼻子数值不会因为额头接触而同时新增刺激。
可调项是小球位置/半径、鼻子敏感度、5 秒积累时间及两个负向方式偏好。
更换模型时鼻尖偏移通常比阶段逻辑更需要校准；扩大球体会增加与额头同时命中的机会。

<a id="graph-contact-breast"></a>

#### Contact Breast：左右接收体共用同一部位状态

入口有胸骨上的两个球，偏移 X 为 -0.055/+0.055，Y=0.055、Z=0.06，半径分别 0.05/0.052。
两侧共用 Source6 与 `breast` 条目，不是两套独立占用或两条独立感受记忆。
阶段流程使用 short→long，当前继承原有视线，不主动强制看鼠标。
输出为 `interaction.contact_breast.touch/.stroke/.stroke.cancel`，表现交给 Touch Breast Reaction。
长触方式偏好当前为 0，表示该方式即时偏好中性，不代表高负荷时永远不会不适。
可调项包括左右球体范围和 short/long 偏好；不要把两个接收体误写成两个不同 Source。
已有胸部短触的实际 API、感受字段和钩子结算证据；不能据此扩张为持续触碰全流程或全部视觉效果已验收。

<a id="graph-contact-belly"></a>

#### Contact Belly：腹部盒体

接收体绑定腰椎，偏移 `(0,0.04,0)`，盒体尺寸 `(0.15,0.05,0.15)`。
入口用 Source7 获取许可，读取 `belly.*`，短触/长触方式分别为 short/long。
输出前缀为 `interaction.contact_belly`，三类事件及取消关联遵循公共模板。
本图继承视线，由 Touch Belly Reaction 处理对应表情请求。
可调的是腹部盒体、short=-0.5/long=0 的方式偏好及通用感受参数。
更换角色身材或腰部骨骼位置时，需要看盒体实际覆盖；不能仅凭图名断言它与胸部、臀部完全无重叠。

<a id="graph-contact-thigh"></a>

#### Contact Thigh：双侧大腿，共用一个 Source

左右大腿各有 Y 向胶囊，沿各自骨骼下移 0.18；Z 旋转为 +5/-5 度。
高度均 0.2，左右半径分别 0.06/0.065；双侧都归 Source8、`thigh` 条目。
它当前 GazeMode=2，获准后跟鼠标；方式为 short(-0.5) 与 long(+0.3)。
输出为 `interaction.contact_thigh.touch/.stroke/.stroke.cancel`，下游 Touch Thigh Reaction。
与牵手的左右目标区分不同，这里没有新增左右大腿 DragTarget，也没有左右独立网络 API。
可调的是两胶囊的骨骼、位置、倾角和半径，以及资源方式偏好；保持同一条目和同一生命周期。

<a id="graph-contact-leg"></a>

#### Contact Leg：左右小腿

左右小腿各有 Y 向胶囊，偏移 `(0,-0.15,-0.025)`，半径 0.04、高度 0.3。
两边共用 Source9 与 `leg`，当前继承视线；方式为 short(-0.3) 与 long(+0.3)。
输出前缀 `interaction.contact_leg`，由 Touch Leg Reaction 消费本部位表现信号。
开始函数端口旁若显示旧预览 0，不代表实际使用 NONE：Source 数据线来自 Constants 的 `INTERACTION_CONTACT_LEG=9`。
这是本手册以接线为准的例子；不能抄有线端口的保存预览来重建逻辑。
可调项为左右小腿胶囊和资源条目参数；维持公共短长触分流与取消缓存规则。

<a id="graph-contact-butt"></a>

#### Contact Butt：骨盆附近双侧胶囊

两个 Y 向胶囊绑定骨盆，X 偏移 -0.075/+0.075，Y=-0.065，半径 0.08、高度 0.2。
两边都以 Source10 申请，读取 `butt.*`，方式为 short(-0.5) 与 long(0)，视线继承。
输出前缀 `interaction.contact_butt`，表现信号下游为 Touch Butt Reaction。
球/胶囊覆盖和角色当前朝向一起决定实际可点击位置；图名本身不构成“只在背面可碰”的几何遮挡保证。
可调的是骨盆附近两胶囊的范围、方式偏好与通用感受参数。
保持双侧同 Source、同条目；为其中一侧单独复制事件或计时会改变当前的共享生命周期。

### 三类持续鼠标操作

Back Drag、Hand Drag 与 Halo Rotate 都先在未拖拽、尚无 DragTarget 时尝试捕获，再等鼠标位移跨过门槛。
它们用 `[3000,15000,30000]` 申请 Controller，在 Phase2/3/4 各发送一次对应阶段事件。
发送字段除交互代次与 `duration_ms`，还包含本条目感受和共同精力/模式上下文。
这些阶段事件没有 Contact 的 touch/stroke 分类，也没有在这里凭空新增“每帧拖动进度”API。

<a id="graph-back-drag"></a>

#### Back Drag：抓住背部移动角色根辅助点

当前启用；接收体位于颈椎附近，X 向胶囊半径 0.04、高度 0.22。
探针进入且尚未开始拖动、没有捕获目标时，以 Source1、GazeMode1（看镜头）申请交互。
获准后写 DragTarget1，并开始 `interaction.motion/back_drag` 感受条目。
跨过拖拽门槛时，记录“Character Node 0 世界位置 - Raw Cursor 世界位置×DepthScale”为 `DragOffset`。
随后游标变化只有通过 Controller.CanUpdate，才写 `Raw Cursor×DepthScale+DragOffset` 到 Character Node 0。
保留偏移使抓取不会把角色突然吸到鼠标中心；`DepthScale=2` 决定这一平面拖拽的放大比例。
它调用 `WarudoDC.BackDrag.Presentation.Start/Stop`，把背部拖拽的肢体摆动等表现留给独立表现图。
目标丢失、图启停清理会先确认 IsOwner，再释放 Controller 与 Runtime，Stop 也有独立收尾路径。
阶段输出为 `interaction.back_drag.phase_2/.phase_3/.phase_4`，时长分别 3000、15000、30000 毫秒。
感受预设为基础强度 0.6、偏好 -0.6、积累15秒、适应20秒、恢复20秒、耐受0.65。
可调项为背部接收体、DepthScale、阶段阈值和感受条目；摆动参数应去表现图调整。
这里移动的是角色根辅助资源，既不是 Runtime Locomotion 命令，也不是给同一角色增加第二套自动移动执行器。

<a id="graph-hand-drag"></a>

#### Hand Drag：左右手捕获，共享牵手生命周期

当前启用；左右手各有半径 0.055 的球形接收体。
两个入口分别尝试同一个 Source5，获准后写 DragTarget3 或 4，视线模式继承。
入口用接触点减 `Raw Cursor×DepthScale` 保存偏移；当前 `DepthScale=2`。
随后只有 CanUpdate 同时确认 Source5、IsDragging 和左右手目标之一，才更新 Hand Drag IK Anchor。
捕获左手时先把锚点放到目标位置，再关闭两手 IK、启用左手；右手分支对称执行。
目标离开、拖拽停止、图禁用时关闭两手 IK；若本图仍为 Owner，再清 DragTarget 并结束交互。
因此左右手视觉执行可区分，但没有两套可同时占用的 Hand Drag 会话。
两入口获准后都开始 `interaction.motion/hand_drag`，结束后都释放同一条目。
阶段输出为 `interaction.hand_drag.phase_2/.phase_3/.phase_4`，沿用 3/15/30 秒。
温和牵手预设为强度0.4、偏好+0.5、积累30秒、适应20秒、恢复20秒、耐受0.8，无单独方式覆写。
这些参数说明模型建议，不把鼠标距离、持续时间或 IK 偏移当成真实拉力。
可调项是两手球体、DepthScale、角色原生 IK 配置与资源预设；不能给左右手各写一份 InteractionOwner。
既有实机证据覆盖牵手首个 3 秒阶段的单次 API/感受/钩子交付；不因此承诺之后整轮语音必定完成。

<a id="graph-halo-rotate"></a>

#### Halo Rotate：抓住光环区域旋转角色

当前启用；头骨上方偏移 `(0,0.24,0)` 的盒体为入口，尺寸 `(0.2,0.1,0.2)`。
获准使用 Source2、DragTarget2、继承视线，并开始 `interaction.motion/halo_rotate` 感受条目。
真正开始拖拽时记录 `RotateStartPos` 与 Character Node 0 的起始世界旋转。
更新使用“起点 X - 当前 X”乘 `RotateSensitivity=150`，作为绕 Y 轴的增量加到起始旋转。
因此鼠标横移改变角色朝向，纵向位移不参与该角度计算；图名不表示只旋转光环模型本身。
每次写旋转前仍通过 Controller.CanUpdate，检查 Source2、IsDragging 与目标2。
松开导致目标变化，或图启停清理时检查归属后结束；Runtime 接触随 Controller.End 后释放。
阶段输出为 `interaction.halo_rotate.phase_2/.phase_3/.phase_4`，仍为 3/15/30 秒。
感受预设为强度0.5、偏好-0.5、积累12秒、适应20秒、恢复18秒、耐受0.7。
可调项为光环盒体、RotateSensitivity、阶段阈值和感受参数；此模型没有测量角速度、离心力或眩晕。

<a id="graph-touch-feelings"></a>

### Touch Feelings：默认关闭的数值观察图

Touch Feelings 位于 Debug，当前禁用；它不是正式交互入口，也不是自动向 Adapter 发消息的模块。
启用后 OnUpdate 先经过 `Throttle Flow`，间隔 0.25 秒，再依次把额头、鼻子的读取结果写到本图变量。
每个部位显示可用、错误、接触中、当前刺激、累积负荷、适应程度、舒适程度七项。
对应 Runtime 字段为 `.contact/.stimulus/.load/.adaptation/.comfort`；可用和错误来自 load 读取节点。
图中没有 Touch Begin/Update/End，不会因为展开观察变量而制造一次接触。
它只观察 `interaction.main` 的额头与鼻子，不代表已经覆盖头顶、所有身体部位或 motion 三条目。
可调项是采样间隔与所读条目；不需要为了看瞬时读数把采样结果逐帧送网络。
禁用时蓝图变量可能保留上次显示，不能把停用观察图中的数字当成当前实时模型状态。
排查模型本身可优先使用资源自带、默认关闭的诊断日志；日志与观察变量仍各有采样和时点边界。

### 这套交互为什么形成现在的结构

以下记录解释当前选择，也指出旧方案哪些内容已经被替代；不应据早期文档或注释回退当前接线。

| 具体问题 | 曾经的处理/误区 | 当前办法与证据边界 |
| --- | --- | --- |
| 多图各自读取占用，难以保证同一帧只有一个获取者 | 依赖分散的状态观察与本地缓存 | 统一同步 TryBegin/IsOwner/CanUpdate/End，Owner 与阶段只由 Controller 提交；接线/模型可查，极端启停仍需实机 |
| 写完阶段再读同一个 getter，广播落后一档 | 旧检查直接设阶段，没有模拟原生端口缓存 | 零参数 Phase.Notify 跨入新的求值边界，再读取新状态填充四参数广播；保留专门缓存模型，不能恢复旧式同流程 getter 取参 |
| 每个交互都靠阶段变化检测器发现通知 | 改成中央逐图调用，又使中央知道全部具体图 | 改成同名 PhaseChanged 广播，各图只按整数 Source 筛选；禁用接收图不会补播 |
| 七张 Contact 重复写交互表情槽 | 每图都写来源、表情、active、serial | 公共同步 Submit 做归属防御，Bridge 统一写入，serial 最后提交；视觉仍由原生表情体系负责 |
| 视觉反馈也被绑在两秒业务点，短按没有反馈 | 阈值检查通过，却没有证明用户看起来及时 | 获准即发接触表现 Signal1，短触松开发 Signal2；用户曾明确拒绝延迟反馈，不能用测试数量覆盖该反馈 |
| 普通点按有表现却没 API | 过渡版只修视觉，Phase1 仍绕过 touch | 当前 Phase1 完成方式确认、更新、短触表现与 touch；不再把小于600毫秒当作无效触碰 |
| 一度把 Phase1/2 都归 touch | 误读了600/2000毫秒的业务含义 | 版本4为600毫秒分短长、2000毫秒触发特殊事件；早期“600毫秒才有效”的文字作废 |
| 2秒前取消带了上一轮 stroke 的 ID | 取消直接读发送节点保留的 EventId | 每次获准清 LongEventId，仅本次本地提交后捕获；空关联、已结算目标都与交付失败分开判断 |
| 短戳和抚摸用同一舒适偏好，感受含义混淆 | 根据长按或部位名臆测模型方式 | Blueprint 显式选择 poke/stroke/press 或 short/long，先 Update 再读数；Runtime 不接管交互分类 |
| 旧阶段 Clock 和业务控制混在一起 | Blueprint 维护时间计算与阶段推进 | 当前以 Runtime 专用计时器提供时基，Controller 保留准入与业务通知；资源化不改变唯一权威 |
| 观察图读到停止变化，或存档里留下旧值 | 把持久化数值、getter结果当作模型一直在推进的证据 | 资源自行推进，读取只读；区分活跃样本、禁用图旧显示与保存预览 |

历史验证中，额头实际展示过接触刺激约0.5、持续时负荷/适应增长、松开后残留恢复；鼻子有独立约0.6刺激的观察。
这与当前敏感度/基础强度相符，但不是所有参数、所有角色和所有持续时长的自动验收。
额头三类事件、胸部短触、背拖/旋转首阶段、牵手首阶段有各自真实交付记录；其他部位共享分流已获用户接受，不要求机械重跑全部同构路径。
最新保存场景已经扩展到独立头顶等更多内容，不能把旧“七处 Contact”的验证数量自动扩成当前八处逐项实机通过。
15/30秒阶段、长帧一次跨多阈值、计时资源消失、旧代次隔离、快速重入和保存禁用状态重载，仍主要依靠模型/结构证据。
本次手册只读保存文件与既有记录，没有新运行 Unity、导出 Mod、加载 Warudo、发 LLM 请求或实施设备体验测试。

### 调整与排查时按链路定位

鼠标位置不准，先查屏幕常量、偏移、跨度和 Camera 1 下的 Raw Cursor；不要先扩大所有身体碰撞体。
星形轨迹不显示，先查 EnableCursorTrack 与 Cursor Track 资源；它不代表交互输入已经关闭。
点击没有接触，检查探针是否在待命时激活、方向是否跟鼠标、是否松开撤回，再看具体部位接收体。
命中却没开始，查看 Controller 是否已有 Owner、启动是否仍阻塞、本次 Runtime 计时申请是否成功。
拖拽已经捕获却不移动，分别查 IsDragging 门槛、DragTarget 与 Controller.CanUpdate，不要直接写 Owner 强行放行。
事件分类不对，核对实际 Phase 和 `[600,2000]`，再看 Contact 的两组 Switch；不要依据旧说明节点判断。
感受为0，先看 feeling_available，再查 Resource/TouchKey/方式和模型启用状态；0不是足够的诊断结论。
取消没有撤回回复，分清是否尚无本次 stroke、目标已结算、网络未交付，或 Adapter 本身的取消/钩子策略。
表现没有及时结束，核对本地图是否仍拥有 Source、Signal 是否到对应 Reaction、Bridge 与表情控制器是否启用。
需要实时诊断时先读有界日志，或读取用户明确保存的场景文件；不要通过完整 live scene 序列化来补充这一手册。


## 表情、注视、注意力与精力

这一组让角色在已有动作、交互和回复之上，具有可持续的脸部状态、明确的注视目标和本地状态读数。
本节覆盖 Affect 的 1 张图、Expression 的 25 张图、Character 的 8 张图，共 34 张。
以下数值来自随本说明提供的保存场景与 Adapter 配置；保存的序号、指针位置、运行标记不是此刻的实时读数。
图中的英文名是查找入口，中文解释说明它在这套场景中的实际用途。

### 先分清五件事

| 名称 | 回答的问题 | 当前权威 |
| --- | --- | --- |
| Timeline Tone | 这一段回复以什么语调表现 | Adapter 提供，Link 按 Timeline 时机释放 |
| Emotion | 当前基础表情选哪个本地编号 | Emotion Controller |
| REQUESTED | 当前外部请求或特定动作希望叠哪张脸 | Expression Request Controller 管生命周期 |
| INTERACTION | 本地交互当下该显示哪张脸 | Interaction Controller 与具体交互图 |
| Attention／Energy | 当前关注什么、活动后还剩多少精力 | 各自 Runtime 场景资源 |

这些状态可以相关，但不互相替代。
疲惫表情不会自动扣精力，精力低也不会自动换疲惫脸；注意力指向 cursor 不代表最终一定看鼠标。
Runtime 的感受、注意力、精力是本地模拟与上下文参考，当前 Emotion Controller 也不是独立的情绪心理引擎。
最终表情、注视许可与原生头眼表现仍由本场景蓝图组合；Warudo Native Lip Sync 继续拥有口型表现。

<a id="graph-emotion-controller"></a>

### Affect：Emotion Controller（25 节点，启用）

职责：把 Link Timeline 真正释放的分段语调，转成场景的基础表情编号。
入口是“Timeline 语调已释放”节点；它消费的是到达本地表现时机的段信息，不是每个文字 token。
当前只有四个明确映射，字符串比较忽略大小写并去除首尾空白。

| 分段 Tone | Emotion | 对应基础脸 |
| --- | ---: | --- |
| neutral | 0 | BASE 的空槽 |
| happy | 1 | EXPRESSION_JOY |
| angry | 2 | EXPRESSION_ANGRY |
| sad | 3 | EXPRESSION_SORROW |

未知或自定义 Tone 保留当前 Emotion，不擅自归零，也不临时创造一张新脸。
这条入口还递增 EmotionSerial，表示收到更新；Serial 本身不要求表情重播。
EmotionIntensity 当前保留为字段，分段强度还没有用于调整原生表情权重。
Constants 里还保留 Emotion=4 的本地身份；BASE 映射会得到 FUN，但当前四 Tone 入口不会产生这个值。
`WarudoDC.Emotion.ResetPresentation()` 把本地 Emotion 与强度归零，供开场／重载初始化使用。
它不修改 AstrBot 的角色设定或情绪，也不会把本地数字反向当成 LLM 必须服从的表现命令。
调整时应先决定 Tone 到本地表现的政策，再改这四条分支；别把基础脸映射和独立 REQUESTED 入口混在这里。

### Expression：脸部如何共同呈现

普通情况下，三份来源同时进入 Warudo 的三个原生表情层。
这是当前场景经过确认的组合策略，不能把“层号更高”直接解释成整脸覆盖优先级。

| 层 | 来源 | 当前普通内容 | 没有贡献时进入 |
| --- | --- | --- | --- |
| 0 | BASE | Emotion 映射出的基础脸 | EXPRESSION_BASE_NEUTRAL |
| 1 | REQUESTED | 外部表情、思考或笑容等请求 | EXPRESSION_REQUESTED_NEUTRAL |
| 2 | INTERACTION | 当前接触表情，或短触松开后的后效 | EXPRESSION_INTERACTION_NEUTRAL |
| 3 | 独立组合覆写 | 明确规则命中时的一张组合脸 | EXPRESSION_OVERRIDE_NEUTRAL |

例如 happy 基础脸与 worried 请求可以同时存在，第 2 层仍可加入当前触碰反应。
共享形变可能被强化，组合是否自然取决于角色表情与原生约束，不能仅靠层号保证。
只有独立组合覆写规则明确命中时，才把普通三层归空，临时交给第 3 层整张组合脸。
覆写结束后按“当时仍有效”的三份来源恢复，不恢复进入覆写前的旧快照。

<a id="graph-expression-controller"></a>

#### Expression Controller（149 节点，启用）

职责：唯一决定这四个呈现槽最终进入哪一个原生表情键。
入口包括每帧更新、图启停、组合规则候选，以及 `WarudoDC.Expression.ResetPresentation()`。
它先快照 Emotion、有效 REQUESTED、有效交互／后效，再调用 Expression Mapping 解析原生名称。
InteractionExpressionActive 为真时采用实际交互；否则才考虑 ContactAftereffectActive。
因此新交互可以遮住旧短触后效，短触后效不需要继续占着交互锁来维持脸部反馈。
三个来源的 payload、Active、Source 与提交 Serial 由各自控制者维护，本图不解析网络请求。

普通层的切换通过 `WarudoDC.Expression.Presentation.Switch(Next, Neutral)` 共用函数完成。
Next 非空就启用目标键，空值就启用该层 NEUTRAL；旧表情按 Warudo 原生同层互斥退出。
当前版每帧幂等确保目标键激活，处理模型重载后“逻辑已空、原生还留旧脸”的失步。
幂等确保不代表按 Serial 重播；Resolved 名称变化才记录 CompositionChanged 和 ExpressionSerial。
所以请求相同脸时可以刷新期限，而不要求每次都重新播放淡入。
启用／禁用时使用内联四层归空，避免图已禁用后再调用本图函数而清理失效。

组合规则通过同步 Evaluate 广播提交候选，Controller 不持有具体规则图的引用。
只有收集窗口内、正 RuleId、当前有交互、且名称在 AllowedOverrideNames 中的候选才可受理。
Priority 较大者优先；相同 Priority 取较小的正 RuleId，不能靠广播顺序抢占。
收集结束后再调用 Compose.Apply，跨过原生数据缓存边界，保证读取本轮写入的候选。
当前 AllowedOverrideNames 只有 EXPRESSION_COMBINATION_JOY_FOREHEAD_CONTACT。
本图可观察 ResolvedBaseName、ResolvedRequestedName、ResolvedInteractionName、ResolvedOverrideName 与 SelectedOverrideRuleId。
这些是诊断值；不要把它们当作外部 API，或绕过来源控制直接写入。

<a id="graph-expression-mapping"></a>

#### Expression Mapping（27 节点，启用）

职责：把本地逻辑代码转换成原生资源键，并按层检查允许目录。
唯一函数入口是 `WarudoDC.Expression.Mapping.Resolve(Layer, Expression)`，输出字符串结果。
BASE、REQUESTED、INTERACTION 各有索引列表和 Allowed 名称列表；未知或不允许的值归空。
中央 Controller 在每次调用前先清函数返回槽，避免缺失定义等情况下沿用上一轮结果。
这里不计时、不受理请求、不启用表情，也不会根据名称猜测某张脸应该在哪一层。

| 映射位置 | 当前有效逻辑代码 | 用途 |
| --- | --- | --- |
| BASE | 1–4 | JOY、ANGRY、SORROW、FUN |
| REQUESTED | 12–30 | 十九种独立表情入口 |
| REQUESTED | 42–45 | Laugh 的四种独立笑脸 |
| REQUESTED | 46 | Ojou Laugh 的临时专属笑脸 |
| INTERACTION | 5–11 | 额头、鼻子、胸、腹、大腿、小腿、臀部持续接触 |
| INTERACTION | 33–39 | 上述七部位短触表情 |
| INTERACTION | 40／41 | 头顶持续／短触表情 |

代码只属于本场景内部映射，Adapter 使用具名 API，不发送原生键或这些整数身份。
新增一张原生脸时，要同时检查角色目录、对应索引、Allowed 列表和真实原生层号。
空索引槽有意保留，不能压缩数组，否则后续代码会整体错位。

<a id="graph-expression-request-controller"></a>

#### Expression Request Controller（68 节点，启用）

职责：接管所有 REQUESTED 来源的替换、持续／限时和来源安全释放。
提交入口为 `WarudoDC.Expression.Submit(EmotionVariants, DurationSeconds, SourceId)`。
来源 ID 必须为正、时长不得为负，公共控制图必须可用；随后先申请 REQUESTED 计时器。
只有计时开始受理后，才保存 OwnedTimerId 并提交表情状态，失败不先抹掉现有脸。
提交时同步读取当前 Emotion，从五项 EmotionVariants 选出本次代码；请求存活期间不会随 Tone 变化重新选差分。
写入顺序是 payload／duration／active／私有来源，最后提交 RequestedExpressionSerial。
这使 Serial 成为完整状态的提交点，读取者不会把半写入状态当成新请求。

| DurationSeconds | 当前语义 |
| --- | --- |
| 0 | 持续，直到替换、明确释放或自身生命周期退出 |
| 正数 | 从本地受理开始按秒计时，到期尝试释放 |
| 负数 | 拒绝提交，保留当前状态 |

现在使用独立的“Runtime 计时器 · REQUESTED 表情”，不再使用历史位置 Tween 时钟。
计时器只报告这一段时间的状态，不拥有表情优先级或原生资源。
HasOwnedRequest、ActiveSourceId、TimedRequestSerial 是本图私有归属；不能把“存在计时器”当成“仍有表情所有权”。
到期前先比较本图记录的代次与中央当前 RequestedExpressionSerial，再检查仍有归属且 Active。
`WarudoDC.Expression.Release(SourceId)` 还必须匹配当前来源，旧图启停不能清掉后来替换它的新脸。
每帧发现中央 Serial 已被别的写入替换，会丢弃旧私有归属并取消旧计时。
计时资源不可用时，也按现有所有权守卫清理，避免把旧表情无限留在脸上。
`WarudoDC.Expression.ResetRequested()` 用于明确全局初始化：取消自己的计时、清来源、清 Active／payload／duration。
这类初始化与普通来源 Release 不同，不能拿它当某个旧请求的专属撤销入口。

<a id="graph-expression-aftereffect-controller"></a>

#### Expression Aftereffect Controller（64 节点，启用）

职责：管理短触松开后的短暂表情残留，让交互锁可以先正常释放。
入口为 `WarudoDC.Expression.ContactAftereffect.Submit(Source, Expression, DurationSeconds)`。
来源和表情代码都须为正，本图须可用；时长限制在 0.05–5 秒。
各部位 Reaction 图决定要提交哪张短触脸与时长，常用配置为松开后 0.65 秒。
计时先由独立“Runtime 计时器 · 短触后效”受理，再写 ContactAftereffectExpression／Active，最后提交 Serial。
输出仍只是一份逻辑交互层候选，实际原生启用统一交给 Expression Controller。
有真实交互表情时，后效暂居其后，不抢走新交互的显示权。

新短触会替换上一份后效，ArmedSerial 与 ActiveSource 防止旧结束清理新后效。
`ContactAftereffect.Release(Source)` 只释放匹配来源且代次仍相符的状态。
图启停清理、计时失效清理、中央 Serial 被替换时的退役均有独立路径。
`WarudoDC.Expression.ResetAftereffect()` 是开场重置入口，取消计时并清 payload／active／来源。
本图不决定 600／2000 毫秒业务分流，不发送 touch／stroke 事件，也不延长 InteractionOwner。
快速点按有视觉后效，不等于凭空追加了业务事件；二者的时机分别由交互流程维护。

<a id="graph-expression-combination-override---1"></a>

#### Expression Combination Override - 1（23 节点，当前停用）

职责：保留一份可编辑的独立组合脸规则样例，只提交提案。
入口是 Controller 广播的 Override.Evaluate(Base, Requested, Interaction)。
现有条件是 BASE=JOY、REQUESTED=0、INTERACTION=FOREHEAD_CONTACT。
命中时提交 RuleId=1、Priority=100、EXPRESSION_COMBINATION_JOY_FOREHEAD_CONTACT。
图内 ExpressionOverrideEnabled 保存为 true，但整张图 Enabled=false，所以当前不会接收广播或参与呈现。
旧注释里“图1启用”描述的是历史配置，不代表本次保存状态。
条件支持 -1 任意、0 无贡献；BASE 与 INTERACTION 当前由 Constants 连入，不是可随意删掉的常量预览。
若以后启用，应先确认规则图与内部开关都开启、目标存在第 3 层且进入中央允许目录。
本图不切原生表情、不改请求来源、不改交互占用；规则退出后由中央在下一轮恢复当前组合。

<a id="graph-expression-combination-override---2"></a>

#### Expression Combination Override - 2（23 节点，当前停用）

职责：第二份复制用模板，不是当前生效的第二套覆写方案。
它接收同名 Evaluate 广播，同样检查输入三元组，再走 Propose。
当前 RuleId=2、Priority=100、MatchRequested=-1，BASE／INTERACTION 仍接 JOY／FOREHEAD_CONTACT。
ExpressionOverrideEnabled=false，OverrideExpressionName 为空，整图 Enabled=false。
因此无论读到何种普通表情组合，当前都不会输出有效覆写。
准备新规则时，要填真实独立资源、唯一正 RuleId、适当 Priority，并同步 AllowedOverrideNames。
不能保留空目标，把它解释成“抑制某一层”；旧 ReplaceBase／ReplaceRequested 等语义已废除。
按某个 BlendShape 抑制其他贡献应使用原生表情约束，不能让规则图直接争写网格或口型。

### 十九张 REQUESTED 入口的共同用法

下列十九图都已启用，每张 9 节点，保留独立图是为了分别编辑意图和本地差分。
每张用 Link 的请求观察节点订阅自己一个 expression.* Key，不解析 WebSocket 或 Timeline 消息正文。
收到请求后，读取 DurationSeconds、五项 EmotionVariants 及 Constants 来源，调用公共 Submit。
图启用／停用走 Release，只清理仍属于自己的当前请求，不扫描和清空整个 REQUESTED 层。
真正选中原生键、处理到期和恢复三层组合的工作由公共控制器完成。
Adapter 里十九项行为都无业务字段，当前允许的 Director 时机为 media_started。
接口没有“传一个任意原生表情名”的字段，默认时长是在场景图里调整。
多数请求现在是持续表情；不能照早期 Adapter 描述把它们全部理解成几秒后自动恢复。

| 蓝图 | 中文用途 | API | 代码／来源 | 当前时长 |
| --- | --- | --- | ---: | ---: |
| Surprised | 惊讶 | expression.surprised | 12 | 1.5 秒 |
| Confused | 困惑 | expression.confused | 13 | 0，持续 |
| Speechless | 无语 | expression.speechless | 14 | 0，持续 |
| Embarrassed | 尴尬 | expression.embarrassed | 15 | 0，持续 |
| Expectant | 期待 | expression.expectant | 16 | 2.5 秒 |
| Smug | 得意 | expression.smug | 17 | 0，持续 |
| Skeptical | 怀疑 | expression.skeptical | 18 | 0，持续 |
| Displeased | 不满 | expression.displeased | 19 | 0，持续 |
| Disgusted | 嫌弃 | expression.disgusted | 20 | 0，持续 |
| Worried | 担忧 | expression.worried | 21 | 0，持续 |
| Fearful | 害怕 | expression.fearful | 22 | 1.8 秒 |
| Aggrieved | 委屈 | expression.aggrieved | 23 | 0，持续 |
| Shy | 害羞 | expression.shy | 24 | 0，持续 |
| Tender | 温柔 | expression.tender | 25 | 0，持续 |
| Relieved | 释然 | expression.relieved | 26 | 0，持续 |
| Playful | 俏皮 | expression.playful | 27 | 0，持续 |
| Tired | 疲惫 | expression.tired | 28 | 0，持续 |
| Bored | 无聊 | expression.bored | 29 | 0，持续 |
| Thinking | 思考 | expression.thinking | 30 | 0，持续 |

当前各图 EmotionVariants 都是自身代码重复五次，因此不同 Emotion 暂时选择同一张请求脸。
要做情绪差分，可按 0 中性、1 高兴、2 生气、3 悲伤、4 本地尴尬身份配置五项；保留完整列表。
这里只改“请求选哪张现有逻辑脸”，不修改 Tone 映射，也不直接修改原生形变。
相同请求再次到达会刷新当前归属／期限；持续请求则保持，直到新的请求或明确清理发生。

<a id="graph-surprised"></a>

#### Surprised：惊讶

入口 expression.surprised，提交代码 12，映射 EXPRESSION_SURPRISED。
用途是突然意外、惊喜或转折；当前形变以抬眉、睁大眼为主。
DurationSeconds=1.5，是少数明确限时的请求之一。
重复触发刷新这段本地期限，最终是否仍显示同时受三层与组合规则影响。
想表现身体受惊，可另配 performance.surprise；本图本身不会播放身体动作。

<a id="graph-confused"></a>

#### Confused：困惑

入口 expression.confused，提交代码 13，映射 EXPRESSION_CONFUSED。
用于没听懂、需要解释；当前脸含漫画式眼神与小三角嘴的用户配置。
DurationSeconds=0，持续至替换／释放，不会因一段音频播放完自动消失。
与 Skeptical 的区别是理解尚未完成，而非已经听懂但不相信。
如果困惑跨多个音频片段保持，无需每片段重新发一次请求。

<a id="graph-speechless"></a>

#### Speechless：无语

入口 expression.speechless，提交代码 14，映射 EXPRESSION_SPEECHLESS。
用于克制吐槽、无奈、暂时难以评价，当前以平眉与半睁眼表现。
DurationSeconds=0；它是一张持续的请求脸，不是停止回复或切断声音。
本图不改变嘴部跟踪开关，也不控制正文是否继续生成。
需要换回其他脸时，由后续请求或已有生命周期明确接管。

<a id="graph-embarrassed"></a>

#### Embarrassed：尴尬

入口 expression.embarrassed，提交代码 15，映射 EXPRESSION_EMBARRASSED。
用于出糗、被拆穿或社交窘迫，当前脸带眉眼收敛与红晕。
DurationSeconds=0；旧记录曾把这个入口称为害羞，当前与 Shy 已明确区分。
EMOTION_EMBARRASSED=4 是另一份本地基础状态编号，不等于此请求代码 15。
改说明时应保留 API 绑定，不能因中文近义就把历史入口暗换到 Shy。

<a id="graph-expectant"></a>

#### Expectant：期待

入口 expression.expectant，提交代码 16，映射 EXPRESSION_EXPECTANT。
用于期待答案、礼物或下一步互动，当前有扬起眉眼、亮眼装饰与微张嘴。
DurationSeconds=2.5，到期释放后按届时仍有效的其他层呈现。
它不是“等待 LLM 中”的系统状态，也不会自动跟踪请求是否返回。
时长要更长时调整本图配置；不要用相邻片段机械重发来模拟等待状态。

<a id="graph-smug"></a>

#### Smug：得意

入口 expression.smug，提交代码 17，映射 EXPRESSION_SMUG。
用于小小炫耀、受到夸奖后的得意，当前有扬眉、略眯眼与猫式微笑。
DurationSeconds=0，只改变 REQUESTED，不改基础 Emotion。
普通得意脸与 Laugh 的坏笑脸是不同键，后者有自己的动作来源与结束守卫。
不应为了共用相似外观，把两种生命周期合并。

<a id="graph-skeptical"></a>

#### Skeptical：怀疑

入口 expression.skeptical，提交代码 18，映射 EXPRESSION_SKEPTICAL。
用于听懂后仍质疑真实性或动机，当前是略皱眉、收眼与偏侧嘴形。
DurationSeconds=0；当前用户已调过形变，不按初始目录重新生成覆盖。
它不触发对方输入重新解释，也不是协议错误或拒绝执行的状态。
可与冷静说话等普通动作分开选择，由 Director 组合语境决定。

<a id="graph-displeased"></a>

#### Displeased：不满

入口 expression.displeased，提交代码 19，映射 EXPRESSION_DISPLEASED。
用于抗议、责怪或闹别扭；当前是压眉、略眯眼与抿嘴鼓气的样式。
DurationSeconds=0，与 angry Tone 所选 BASE 生气不是同一来源。
两者可能同时叠加，过强时应检查原生表情和约束，而非偷改外部消息。
这张图不取消交互，也不把不满转换成负舒适值。

<a id="graph-disgusted"></a>

#### Disgusted：嫌弃

入口 expression.disgusted，提交代码 20，映射 EXPRESSION_DISGUSTED。
用于排斥某个对象或行为，当前有皱眉、眯眼与面部压暗装饰。
DurationSeconds=0；装饰来自角色现有原生表情配置。
图里不新增材质切换、网格写入或身体撤退动作。
它与第 3 层“开心＋戳额头”的故作嫌弃样例是两张不同的脸。

<a id="graph-worried"></a>

#### Worried：担忧

入口 expression.worried，提交代码 21，映射 EXPRESSION_WORRIED。
用于关切、犹豫或预感不妙，当前以眉尾下垂、双眼略睁大表现。
DurationSeconds=0，适合一段关心语境持续保持。
这只是表现意图，不代表 Runtime 已检测到危险或生理异常。
随后释然可以提交 Relieved，交接仍由同一个公共请求槽处理。

<a id="graph-fearful"></a>

#### Fearful：害怕

入口 expression.fearful，提交代码 22，映射 EXPRESSION_FEARFUL。
用于明确受惊或恐惧，当前为高挑而下垂的眉、睁大眼与下弯嘴角。
DurationSeconds=1.8，是当前第三个正时长表情入口。
只请求脸，不自动播放颤抖、移动或取消动作；这些需要独立行为准入。
较强惊讶也不必自动选害怕，Director 说明保留两者的语境差别。

<a id="graph-aggrieved"></a>

#### Aggrieved：委屈

入口 expression.aggrieved，提交代码 23，映射 EXPRESSION_AGGRIEVED。
用于被误解、冷落或想得到安慰，当前是低垂眉尾与委屈的眼嘴。
DurationSeconds=0；它不写感受模型，也不累计“好感度”。
要调脸直接在角色对应原生键中改，保持代码与层号不变。
同一来源图关闭时仅释放自己的当前请求，不清除后来新来的温柔脸等请求。

<a id="graph-shy"></a>

#### Shy：害羞

入口 expression.shy，提交代码 24，映射 EXPRESSION_SHY。
用于亲近、夸奖后的正向羞涩，当前眉眼柔和并带羞涩装饰。
DurationSeconds=0，与 Embarrassed 的社交窘迫分别表达。
低头羞涩笑也有独立 Laugh 键，本图不额外压低头部或播放笑声。
是否搭配肢体由表现编排选择，脸部入口自身不申请身体占用。

<a id="graph-tender"></a>

#### Tender：温柔

入口 expression.tender，提交代码 25，映射 EXPRESSION_TENDER。
用于安慰、关爱或亲昵回应，当前有轻柔眉眼和温和嘴角。
DurationSeconds=0，适合保持一段平缓关心语境。
它可以与解释型说话等独立动作搭配，不自动把所有礼貌回答都切成温柔。
当前形变是用户调配版，初始模板只说明来源，不能覆盖现有脸。

<a id="graph-relieved"></a>

#### Relieved：释然

入口 expression.relieved，提交代码 26，映射 EXPRESSION_RELIEVED。
用于担忧解除、确认安全或松一口气，当前为舒展眉头与放松眼神。
DurationSeconds=0，释然不是一段固定自动恢复计时。
它是担忧之后可选择的表演转折，不根据精力恢复量自动触发。
REQUESTED 替换后旧 Worried 的释放不能再清除这张新脸。

<a id="graph-playful"></a>

#### Playful：俏皮

入口 expression.playful，提交代码 27，映射 EXPRESSION_PLAYFUL。
用于善意调侃、逗弄或小小使坏，当前有扬眉和明显笑眼。
DurationSeconds=0；与坏笑肢体组合并非同一来源。
严肃、道歉或对方不适时不宜因“笑脸”标签机械选用，语境选择仍在 Adapter／LLM。
蓝图只处理被提交后的本地生命周期，不实现这种对话判断。

<a id="graph-tired"></a>

#### Tired：疲惫

入口 expression.tired，提交代码 28，映射 EXPRESSION_TIRED。
用于困倦、疲劳或勉强打精神，当前眼皮明显降低。
DurationSeconds=0；这个请求不证明精力模型已经 low。
Energy and Rest 的读数也不会反向直接触发本图。
想把精力作为表达参考，应由现有上下文和对话政策选择，不能在此图硬绑自动换脸。

<a id="graph-bored"></a>

#### Bored：无聊

入口 expression.bored，提交代码 29，映射 EXPRESSION_BORED。
用于兴致不足、敷衍或等待太久，当前是平眉、半睁眼并略眯起。
DurationSeconds=0，与 Tired 的低精力外观虽相近，语义分别保留。
没有“等了多少秒就无聊”的自动计时，也没有新增自主意愿模型。
它只参与 REQUESTED 层，不改变待命／自由模式。

<a id="graph-thinking"></a>

#### Thinking：思考

入口 expression.thinking，提交代码 30，映射 EXPRESSION_THINKING。
用于回忆、斟酌、权衡时的轻微眉眼表现，不表示展示模型内部推理过程。
DurationSeconds=0；单独调用只请求脸，不自动进入 Thinking Pose。
直接表情来源与 Performance 的思考状态来源分开，不能用同一个旧结束流程互相清除。
要同时改变姿态应走对应表现入口；统一回归普通姿势也不能随意释放其他来源的脸。

### 当前 49 个原生表情键怎么维护

角色的原生 Expressions 中共有 49 项：BASE 5、REQUESTED 25、INTERACTION 17、OVERRIDE 2。
下面是完整键目录；表情键属于本地角色呈现，不能作为网络 API 参数暴露。

| 层／类别 | 当前原生键 |
| --- | --- |
| BASE 空槽 | EXPRESSION_BASE_NEUTRAL |
| BASE 基础 | EXPRESSION_JOY、EXPRESSION_ANGRY、EXPRESSION_SORROW、EXPRESSION_FUN |
| REQUESTED 空槽 | EXPRESSION_REQUESTED_NEUTRAL |
| REQUESTED 12–15 | EXPRESSION_SURPRISED、EXPRESSION_CONFUSED、EXPRESSION_SPEECHLESS、EXPRESSION_EMBARRASSED |
| REQUESTED 16–19 | EXPRESSION_EXPECTANT、EXPRESSION_SMUG、EXPRESSION_SKEPTICAL、EXPRESSION_DISPLEASED |
| REQUESTED 20–23 | EXPRESSION_DISGUSTED、EXPRESSION_WORRIED、EXPRESSION_FEARFUL、EXPRESSION_AGGRIEVED |
| REQUESTED 24–27 | EXPRESSION_SHY、EXPRESSION_TENDER、EXPRESSION_RELIEVED、EXPRESSION_PLAYFUL |
| REQUESTED 28–30 | EXPRESSION_TIRED、EXPRESSION_BORED、EXPRESSION_THINKING |
| REQUESTED 四种笑 | EXPRESSION_LAUGH_SHY_DOWN、EXPRESSION_LAUGH_JOY_UP、EXPRESSION_LAUGH_SMUG_DOWN、EXPRESSION_LAUGH_SMUG_UP |
| REQUESTED 大小姐笑 | EXPRESSION_OJOU_LAUGH |
| INTERACTION 空槽 | EXPRESSION_INTERACTION_NEUTRAL |
| 头顶 | EXPRESSION_HEAD_TOP_CONTACT、EXPRESSION_HEAD_TOP_CONTACT_SHORT |
| 额头 | EXPRESSION_FOREHEAD_CONTACT、EXPRESSION_FOREHEAD_CONTACT_SHORT |
| 鼻子 | EXPRESSION_NOSE_CONTACT、EXPRESSION_NOSE_CONTACT_SHORT |
| 胸部 | EXPRESSION_BREAST_CONTACT、EXPRESSION_BREAST_CONTACT_SHORT |
| 腹部 | EXPRESSION_BELLY_CONTACT、EXPRESSION_BELLY_CONTACT_SHORT |
| 大腿 | EXPRESSION_THIGH_CONTACT、EXPRESSION_THIGH_CONTACT_SHORT |
| 小腿 | EXPRESSION_LEG_CONTACT、EXPRESSION_LEG_CONTACT_SHORT |
| 臀部 | EXPRESSION_BUTT_CONTACT、EXPRESSION_BUTT_CONTACT_SHORT |
| OVERRIDE 空槽 | EXPRESSION_OVERRIDE_NEUTRAL |
| OVERRIDE 样例 | EXPRESSION_COMBINATION_JOY_FOREHEAD_CONTACT |

四个 NEUTRAL 均为空表情，保留同层互斥能力，不代表一项新业务情绪。
它们没有形变或物体副作用，不能拿空槽去关闭原生口型、眨眼或眉毛跟踪。
现有脸中保留用户自己制作的嘴部造型；这不授权蓝图另写口型、下巴或音素系统。
当前 49 项 DisableMouthTracking 均为 false，原生 Lip Sync 的职责保持。
四种 Laugh 独立脸最初从 FUN 复制初值，现已由用户分别调配，不再与 BASE 的 FUN 自动同步。
Ojou Laugh 只有 laugh_once=true 时临时申请专属脸，头部动作结束后按来源／代次释放；持续肢体仍可保留。
laugh_once=false 不请求或刷新表情；不要按较早记录把这张脸绑定整段持续姿态。

### Character：注视链先选目标，再执行

Character 组八张图分为五张注视图、一张注意力、一张精力、一张最终根变换同步。
下列两条链分别处理“看哪里”和“角色整体在哪里”，不应互相争写。

```text
镜头／光标／交互／手势候选
  → Gaze Controller 选择获准目标
  → Look-at Input Anchor
  → Runtime 注视设施（跟随、平滑、小眼神角度）
  → Look-at IK Anchor → 原生头／身体注视
                         └→ Look-at Motion Anchor → 原生眼睛注视

Character Node 0 → Character Node 1 → Back Drag Swing Anchor
  → Character Root → Character Transform → 角色 1 的世界位置／旋转
```

<a id="graph-gaze-controller"></a>

#### Gaze Controller（92 节点，启用）

职责：集中仲裁注视许可、模式与最终目标，唯一写入主注视输出锚点。
每帧先 Tick 更新手势／短看状态，再 Render 选择目标；原生函数调用也用于跨缓存边界读取新状态。
Ctrl+Alt+S 控制 EnableLookCursor：开启是“常追视”，关闭是“一般”。
这个偏好与 ActMode 的待命／自由相互独立，TrackingStatus 显示一般或常追视。
明确 GAZE_MODE_ANIMATION 的动画接管始终受保护，此时停止驱动注视锚点并关闭角色 LookAtEnabled。

| 一般模式中的顺序 | 如何表现 |
| --- | --- |
| 明确交互注视 | 使用 Interaction Controller 提交的明确模式 |
| 当前回复表现 | 看用户／镜头，优先于鼠标手势和短看 |
| 左键手势 | 按住跟光标，松开看固定点 |
| 光标短看 | 新注意力刺激满足窗口与冷却后短跟随 |
| 空闲回退 | 沿原模式与 FreeLookAtCamera 的回退策略 |

常追视覆盖一般交互和回复的目标选择，仍不会越过明确动画接管。
继承型交互不产生新的注视要求；注意力 user 可作为看镜头的建议，cursor 不直接等同于连续跟随。
Link“读取回复表现状态”选择 assistant.reply，文字成功结束后当前保留 2 秒阅读保持。
它兼顾文字与整轮 Timeline，分段间隙或单段音频结束不等于整轮回复结束。
这些是本地表现活动，不能解释为 LLM 生成状态或物理声卡已经播放结束。

选定位置先零过渡写 Look-at Input Anchor，Runtime 注视设施以同一来源跟随这个资源。
资源有有效 HasTarget 时，唯一原生位置节点把平滑 Position 零过渡写 Look-at IK Anchor。
正常重复提交同一跟随资源不创建新命令、不清平滑速度，连续换目标不绕经世界原点或镜头。
回到镜头时 Returning=true，使用资源的独立 ReturnSeconds；其余转向用 TurnSeconds。
动画接管、图启停会按当前 CommandId 结束自己的命令，并清手势／短看运行态。
可调项 FreeLookAtCamera 位于本图，具体平滑时间在 Runtime 注视资源，不要恢复第二套位置 Tween。

<a id="graph-gaze-pointer-gesture"></a>

#### Gaze Pointer Gesture（98 节点，启用）

职责：实现一般模式下的左键按住跟随、松开采样与尾段停留。
Press／Release 由 Gaze Controller 的鼠标入口调用；本图不另加一套全局鼠标监听政策。
按下经中央许可后快照 InteractionSerial，进入按住跟随，CandidatePoint 持续采用当前光标锚点。
长按不会消耗松开后的停留期限，也不在每帧重启一次手势。
松开时采样当帧位置，切为固定点，从松开开始计 ClickLookSeconds=1.2 秒。
ClickConfirmSeconds=0.08 秒用于确认整次手势未被交互接管，不重启这 1.2 秒。
只要期间 InteractionSerial 变过，即使那个短交互已经结束，也不会误把这次触碰当空白点击。

连续再次按下直接切回实时光标，保留主目标平滑，不插入一次回正镜头。
交互、回复、常追视抢占后清掉候选／活动，松开时不补发旧尾段，也不在结束后恢复旧手势。
CandidatePoint 是给中央使用的输出，ClickCandidate／ClickReleased／ClickLookActive 是内部状态。
当前使用“Runtime 计时器 · 点击注视”，TimerId 属于运行态，保存值不应抄成配置。
初始化、禁用、明确 Reset 都清理计时与活动，避免保存时按住的状态在重载后自行复活。
这个手势不依赖注意力模型可用，也不发 Control 事件或调用 LLM。

<a id="graph-gaze-cursor-glance"></a>

#### Gaze Cursor Glance（54 节点，启用）

职责：在一般模式允许时，由新的鼠标刺激发起一个短跟随窗口。
Attention 图在模型真正受理 cursor 瞬时刺激之后调用 CursorActivity，中央再转到 Glance.TryStart。
要求注意力样本可用且目标为 cursor、无交互／回复／手势占用、不是常追视、没有活动窗口且冷却已结束。
ShortLookSeconds=0.8 秒，期间每帧跟随当前光标位置，不是冻结刺激发生时的点。
ShortLookCooldownSeconds=4 秒，新活动不会续期或刷新正在运行的窗口。
从一次启动起，至少窗口时长再加冷却后才可重新开始，持续读到 cursor 不能重放短看。
被更高优先级打断、模型不可用、图启停会清窗口，旧窗口不恢复。
计时使用“Runtime 计时器 · 短看与冷却”，GlanceClockIdleSeconds=60 是内部闲置计时范围，不是一次短看时长。
可调窗口／冷却在本图；注意力灵敏度在注意力资源，两种参数分别影响触发与节奏。

<a id="graph-gaze-motion"></a>

#### Gaze Motion（32 节点，启用）

职责：把 Runtime 已计算的小眼神角度接入原生眼睛通道，并负责接管／恢复。
资源有效、中央处于可注视模式、投影可用且 EnableMotion=true 时才 Running。
写子锚点前同步调用 Gaze Aim Projection，再把 LocalOffset 零过渡写入 Look-at Motion Anchor 局部位置。
头和身体继续看父 Look-at IK Anchor；独立原生眼睛节点看子锚点，避免小眼神把整个头也带走。
管线接管时角色普通 LookAtEyesWeight 设为 0，独立眼睛通道使用 EyesWeight=1。
管线关闭／目标失效时恢复普通眼睛权重，并把子锚点偏移归零。
所以保存中角色眼睛权重为 0 本身不表示眼睛坏了，先检查独立管线是否正在接管。

EnableMotion 控制本图的原生适配；资源 EnableEyeMotion 只控制随机角度微动，关闭后普通目标跟随仍可继续。
MotionSmoothTime=0.1 保留接给原生眼睛求解，不能把它直接说成最终扫视耗时 0.1 秒。
随机节奏和角度阻尼已移入 Runtime，本图不再每帧自行抽签或建立第二套眼神时钟。
当前原生眼神限幅为水平 15°、竖直 10°；头／身体权重继续沿角色用户配置。
本图不改变眨眼、Lip Sync、Emotion，也不自动向外部报告“看完了”。

<a id="graph-gaze-aim-projection"></a>

#### Gaze Aim Projection（30 节点，启用）

职责：把角度变成与目标距离相适应的眼睛落点，是纯粹的原生坐标换算图。
入口 `WarudoDC.Gaze.Motion.Project()` 由 Gaze Motion 每次写子锚点之前同步调用。
先取得头骨世界位置与父锚点世界位置，建立“头看向主目标”的方向与距离。
把 Runtime EyeAngles 旋转到这个观察坐标中，保留原距离得到新的世界落点。
再减父锚点位置，用父锚点逆旋转换成子锚点 LocalOffset。
EyeAngles=0 时自然回到主目标，头到目标不足 0.05 米时输出零，避免近零方向求解。
启停清 LocalOffset 并维护 Available，中央策略与计时不在这里重复实现。
父锚点必须保持单位缩放；当前无父绑定、单位缩放的布置是这份换算的使用条件。
改变角色大小、父级或锚点缩放后应复查投影，不要靠盲目扩大米制偏移掩盖坐标错误。

### Runtime 注视资源的当前可调值

“WarudoDC Runtime 注视设施”是一个独立场景资源，ModelKey=gaze.main。
它只读跟随资源的位置，输出结果，不直接写角色、锚点、骨骼或原生注视开关。
InitialReference 选 Look-at IK Anchor，用现有输出初始化第一次接管，避免从世界原点缓慢追入。

| 配置 | 当前值 | 影响 |
| --- | --- | --- |
| TurnSeconds | 0.20 秒 | 正常转向与跟随的阻尼参数 |
| ReturnSeconds | 0.35 秒 | 回镜头的独立阻尼参数 |
| FocusMinSeconds／Max | 2／4 秒 | 看主目标的随机保持区间 |
| AwayMinSeconds／Max | 0.5／1.1 秒 | 小眼神移开后的保持区间 |
| EyeSeconds | 0.01 秒 | 角度微动的阻尼参数 |
| GlanceAngles | 六个落点 | 左右约 10°、斜向约 8° 的偏移 |

六项角度为 (0,10,0)、(0,-10,0)、(-5,-8,0)、(4,8,0)、(-4,8,0)、(4,-8,0)。
采用 Unity 欧拉角：X 俯仰、Y 左右、Z 滚转；当前 Z 都为 0。
阻尼参数不是严格到达时长，目标持续移动时也不具有一次“播放完成”的意义。
正常替换保留当前位置和速度；固定点在受理时采样，跟随资源持续只读采样。
资源按本地帧推进一次，多个读取者不会多走时钟；读取状态也不会重新随机。
资源禁用、目标丢失、来源图禁用／删除或改绑会使旧命令失效；重新启用需要重新提交获准目标。
手势窗口和短看冷却仍由对应蓝图及 Runtime 计时器管理，不放入注视资源当交互政策。

<a id="graph-attention"></a>

#### Attention（34 节点，启用）

职责：把已经发生的用户交互和鼠标活动交给本地注意力模型，并提供低频观察变量。
它订阅 `WarudoDC.Interaction.OwnershipCommitted`；Owner 非零时开始 user 持续刺激 0.8，清零后结束。
这是占用提交后的观察者，不参与 TryBegin，也不能通过停用此图来阻止原交互获准。
鼠标读取 Mouse Pos to Cursor Pos 的 RowPos，每 0.25 秒采样一次。
任一平面轴变化至少 0.02 场景单位才提交 cursor 瞬时刺激 0.25，微小抖动不构成刺激。
启用时先保存基线；每次采样先比较再更新 PreviousCursor，独立于鼠标追视偏好开关。
只有刺激实际被模型受理后才通知 CursorActivity，持续的目标读数不是短看触发器。

模型资源当前有 user 与 cursor 两个目标，Sensitivity 均为 1。
衰减 3 秒、适应 8 秒、恢复 8 秒、最短保持 0.75 秒、切换强度差 0.15。
瞬时刺激会留下衰减残留，重复／持续刺激会习惯化；同目标多来源取最大有效强度，不直接叠成双倍。
AttentionCurrentTarget／Strength／Has_Target 与 UserStrength／CursorStrength 每 4Hz 更新供观察。
current.has_target=false 是有效的无目标状态；模型不可用则应检查 Bound／HasValue，不能冒充强度为零。
资源回答“关注谁、多少”，Gaze Controller 回答“当前允许看谁”，注视资源回答“如何平滑转过去”。
它不持有角色、骨骼或原生表情，也不把不适刺激直接换算成负注意力。

<a id="graph-energy-and-rest"></a>

#### Energy and Rest（29 节点，启用）

职责：按实际运动／交互登记活动，把精力、低精力和恢复读数交给场景观察与上下文。
当前移动输入读取 Runtime Locomotion 的 motion.active，变化检测器发 LocomotionChanged(true／false)。
这反映实际执行状态，不根据命令是否收到、LLM 是否请求或旧 IsMoving 快照开始扣精力。
交互输入同样订阅 OwnershipCommitted，已获准 Owner 非零才开始 interaction，释放后结束。
两类活动强度当前均为 1；不是“自由模式一直扣精力”，也不是“待命就不会消耗”。
图中六个观察变量每 0.25 秒从资源读取，观察频率不改变模型计算时钟。

| 精力资源配置 | 当前值 |
| --- | ---: |
| 初始精力 | 100，范围 0–100 |
| 进入低精力／解除低精力 | 20／25 |
| 无活动后的恢复等待 | 5 秒 |
| 安静恢复 | 每分钟 2 点 |
| locomotion 满强度消耗 | 每分钟 3 点 |
| interaction 满强度消耗 | 每分钟 0.5 点 |

同活动多来源取最大强度，不因重复登记扣两份；不同活动消耗相加。
强度大于零且消耗为零仍算活动，会推迟休息；“不费精力”与“没有活动”分别处理。
Energy／LowEnergy／Active／Recovering／DrainPerMinute／RecoveryPerMinute 是输出，不是建议直接手写的控制变量。
修改初始值不立刻补满，显式重置才使用初值并结束旧活动；禁用冻结、不补算离线休息。
低精力不会自动拒绝明确命令、切模式、强制睡眠或切 Tired 表情。
Adapter 的 runtime.activity 规则把精力与模式解释成参考，最终回复／意愿仍结合对话语境决定。
当前没有默认接入 speaking 活动；整轮回复活动也不能简单等同持续发声。

<a id="graph-character-transform"></a>

#### Character Transform（6 节点，启用）

职责：把辅助层级合成后的根位置与朝向统一同步到最终角色资源。
每帧读取 Character Root 的世界位置与世界旋转，顺序零过渡写入“角色 1”。
它没有本地变量、命令受理、插值计时或动画播放，只做最后一步同步。
Character Node 0／1、Back Drag Swing Anchor、Character Root 是沿父子关系合成的辅助链。
移动、背部拖拽／摆动等应写各自获准拥有的辅助层级，避免多个行为直接争写角色根变换。
这条同步链不代替 Performance 的肢体准入，也不替代原生骨骼动画、注视或表情。
Runtime Locomotion 的根运动例外仍要求明确绑定唯一执行者；不能让旧移动图与新执行者同时驱动同一层。
更换角色资源时，要一起核对本图输出、Character Root 绑定以及头眼／表情引用，而非只改可见模型。

### 为什么最后形成现在这套分工

| 具体问题 | 曾尝试或查明的路径 | 当前方案与保留限制 |
| --- | --- | --- |
| 临时表情一启用就消失 | 早期位置时钟复位写本地，但同帧到期判断读到旧世界缓存 | 曾修正读法，现迁为 Runtime 计时器；来源和 Serial 守卫继续保留，离线计时通过不等于所有实机时序都通过 |
| 三层谁覆盖谁含混 | 旧整脸胜出者不适合用户希望的并行组合 | 普通 0／1／2 层共同呈现；共享形变的视觉关系仍由原生约束与用户调脸决定 |
| 组合覆写被误解成局部抑制 | 曾安装“开心＋惊讶时抑制开心”的 Replace* 方案，用户否定其用途 | 改为第 3 层独立组合脸，规则同步提案、中央选择；当前两个规则图均关闭 |
| 清理原生脸需要逐项枚举 | 用户确认空表情也具有同层互斥 | 四个 NEUTRAL 作为归空目标；不把空槽增加为对话 API 或业务情绪 |
| 重载后逻辑已中性但脸仍生气 | 只在 Resolved 名称改变时调用原生切换，可能漏掉原生加载失步 | 每帧幂等确保当前目标激活；用户已确认常用重载修复，任意异步加载／外部改脸边界未全部覆盖 |
| 短触没有即时反馈、长触等到 2 秒才变脸 | 初版把视觉许可误绑业务阶段门槛 | 获准接触立即有交互脸，松开可有 0.65 秒独立后效；600／2000 毫秒业务分流仍分别维护 |
| 注视目标切换抖动 | 不同 SetAssetPosition 节点的 Tween 可以继续争写同一锚点 | 合并唯一输出写入口，后来平滑迁入 Runtime；不据此断言所有视觉抖动来源已经排除 |
| 空白点击几乎看不到停留 | 旧多节点时钟 Tween 竞争会读到已超期起点 | 统一计时入口后迁 Runtime；现按住跟随、松开固定 1.2 秒，0.08 秒确认不续期 |
| 连续点击先回正 | 旧按下流程先清活动，再等待新候选 | 直接交接实时光标和松开点，保留平滑状态；被交互／回复抢占后仍不恢复旧手势 |
| 小眼神不明显 | 固定米制偏移随距离变化，曾只产生约 1–2° 转动 | 换六个角度落点与距离投影，保留原生眼睛求解；父锚点单位缩放仍是使用条件 |
| 误以为眼睛没动 | 保存值只能证明许可，临时有限日志才显示原生输出与最终眼骨确有变化 | 角度版与快速过渡获用户基本观感认可；不把日志角度称为眼球网格可见幅度的全面验收 |
| 注意力直接驱动自由模式光标 | 早期把模型 target=cursor 直接映射到连续追视，含混了模式与追视偏好 | 常追视／一般独立，模型只触发获准短看或提供建议；0.8 秒／4 秒的间歇节奏仍可后续调优 |
| 运动结束仍扣精力 | 早期只补中央部分写口，漏掉正常转向收尾通知 | 已补释放并获正常路径反馈；当前保存改为观察 motion.active，仍应核对实际开始与结束 |

当前常规注视、资源开关与重载恢复已有用户反馈；按住跟随／松开停留／连续交接已做基本观感确认。
精力的实际活动消耗、停止后恢复与初值／重置分离也已有正常使用反馈。
这不扩张为所有资源删除、来源改绑、禁用保存加载、快速抢占或长期随机边界都逐项验过。
调试优先看安全共享读取的 Player.log 与用户主动保存的 JSON，不能读取或轮询完整实时场景来追几秒内的状态。
需要新状态证据时先让用户保存，再区分“保存值”“日志事件”“当前未验证值”，不把快照当直播。

### 需要改哪里时的查找顺序

| 想改的体验 | 先改哪里 | 还应核对 |
| --- | --- | --- |
| 某张脸的外观 | 角色 Expressions 对应键 | 原生层号、约束、口型跟踪与现有用户调配 |
| 某类请求保持多久 | 对应九节点表情图的 DurationSeconds | 0 持续、正数限时；Adapter 说明不能覆盖实际场景值 |
| Tone 对应基础脸 | Emotion Controller 与 BASE 映射 | 四 Tone 范围、未知保留、强度尚未消费 |
| 新增逻辑表情 | Constants、Expression Mapping、角色目录与入口 | 索引空槽、Allowed 目录、来源身份、API／Adapter 同步 |
| 戳后脸保留时间 | 对应 Touch … Reaction 的后效配置 | 独立后效计时、锁立即释放、真实交互优先 |
| 转头／回正快慢 | Runtime 注视资源 TurnSeconds／ReturnSeconds | 不再叠位置 Tween，独立眼睛求解速度分别看 |
| 按住／松开停留 | Gaze Pointer Gesture | 交互 Serial、0.08 秒确认、抢占不恢复 |
| 自发短看频率 | Gaze Cursor Glance 与注意力资源 | 窗口与冷却分开，不用持续目标读数重放 |
| 小眼神幅度／节奏 | Runtime 注视资源 | Gaze Aim Projection、原生限幅、父锚点缩放 |
| 精力下降／恢复速度 | Runtime 精力与休息资源 | 活动是否实际登记／释放，初值与显式重置分开 |


## 身体表现、待机与说话动画

这一部分覆盖 Performance 组35张图、Idle 组1张图、Speak 组10张图，共46张。
它们把已经受理的意图变成角色动画，同时保留移动、拖拽、表情和原生口型各自的控制权。
以下参数按本次提供的保存场景核对；保存下来的运行变量不代表此刻正在执行的状态。
本节中的 `WarudoDC.*` 函数是场景内部接线入口；对接 Adapter 时使用列出的业务 API。
动画素材由场景本地选择，不需要把素材地址、图节点身份或角色表情资源名发送到协议里。

### 先分清三种身体表现

| 类别 | 典型图 | 对已有身体动作的影响 | 怎么结束 |
| --- | --- | --- | --- |
| 普通身体表现 | Thinking、各姿势、Wave、Speak、Ojou的底姿 | 申请同一个身体通道，获准的新动作可以接替旧普通动作 | 回归Idle、期限到达、新动作接管或生命周期清理 |
| 短附加动作 | Nod、Shake Head、Surprise、Swing Body、Laugh、Ojou抬头 | 叠在已有Idle／Speak／普通姿势上，不重启底姿 | 原生单次动画结束信号，随后一帧释放逻辑槽 |
| 持续附加 | 歪头、低头望向一边、颤抖 | 保留底姿，在三个固定原生层中选择一个目标 | 显式清除、另一持续项接替、短附加获准、保护状态撤销 |

Locomotion和Back Drag属于受保护身体来源。
普通姿势、回归Idle或说话动作不能把它们抢走；它们也不是普通姿势列表中的另一种差分。
普通身体、短附加和持续附加是不同槽位，但有共同的准入条件，不代表可以任意叠加三层意图。
身体表现不自动等于表情；只有Thinking、Laugh和Ojou的一次抬头笑明确提交配套表情。
Warudo Native Lip Sync始终负责口型，本节没有自定义嘴形、下颌、音素或viseme控制。

<a id="graph-performance-controller"></a>

### Performance Controller：统一许可、占用与释放

**用途。** 这是身体占用的唯一权威，统一管理普通表现、受保护来源和两个附加槽。
不要直接改它的 `BaseAnimationOwner`、`Source`、`OverlaySource` 或 `SustainedSource` 来触发动作。
这些变量描述控制器正在管理谁，不是给外部调用者准备的命令端口。

普通动作调用 `WarudoDC.Performance.TryStart(Source, RequiredInteraction, AutoEnd, DurationSeconds)`。
返回true后才播放动画、抬起角色或提交该动作的临时原生开关政策。
校验或身体许可失败就保持当前动作；没有等待队列，也不会等保护状态结束后补播旧请求。
普通姿势通常用 `RequiredInteraction=0`，Back Drag使用自己的交互来源作为持续条件。

这里区分“开始时允许”和“开始后必须一直满足”。
`RequiredInteraction=0` 的动作开始时要求没有交互；获准后，单纯进入接触并不自动取消它。
新的交互如果没有申请身体动作，原姿势继续；真正获准的新身体来源才执行交接。
绑定特定交互的动作失去所属交互后则必须结束；不能继续占着已结束的拖拽身体状态。

控制器内部还提供这些入口：

| 函数 | 使用场合 |
| --- | --- |
| `WarudoDC.Performance.Stop(Source)` | 结束仍属于该来源的普通表现 |
| `WarudoDC.Performance.Terminate(Reason)` | 控制器统一执行当前表现的结束流程 |
| `WarudoDC.Performance.Ended(Source, Reason)` | 通知各动作图清理自己拥有的附属状态 |
| `WarudoDC.Performance.ResetPresentation()` | 场景生命周期复位普通表现 |
| `WarudoDC.Animation.TryAcquire(...)` | Locomotion等来源申请身体占用 |
| `WarudoDC.Animation.IsOwner(Source)` | 播放或阶段推进前核对身体仍属于自己 |
| `WarudoDC.Animation.Release(Source)` | 仅释放仍匹配的来源，旧来源不能释放新占用 |
| `WarudoDC.Animation.ResetPresentation()` | 生命周期重置身体占用 |

`BaseAnimationOwner=0` 表示空闲，Locomotion和Back Drag各有自己的内部来源。
这些内部整数用于本场景仲裁，不要求Adapter知道，也不能作为公开协议里的动作标识。
原来独立的Base Animation Owner图已经合并到本图，不需要寻找或恢复那张旧图。
Idle Animation观察统一占用变化，在确实空闲后才补回所选站姿。

**期限。** 普通限时表现使用专用“肢体表现”Runtime计时器，持续表现也由同一权威管理。
控制器保存本次TimerId并校验归属；旧计时结束不能结束后来接管的动作。
计时器缺失或归属失效时释放本图拥有的表现，不取消其他来源的时钟。
初始化不重放保存时的姿势、旧期限或旧结束通知。
期限到达是本地生命周期结束；它不是逐帧核验某个动画动作刚好完整播放的回执。

**呼吸与晃动。** `WarudoDC.Performance.SetAnimationPolicy` 接收两个独立开关。
各姿势图中的“禁用呼吸动画”“禁用晃动动画”是在本次动作获准后使用的临时政策。
控制器首次接管某开关时记录原值，结束时只恢复自己实际控制过的通道。
原来关闭的通道结束后仍关闭；没有被本动作接管的通道不会被统一强制打开。
同来源刷新也不会把临时关闭后的false误记为原值。
只改图变量不会让当前已开始的动作立即重新应用政策，需下一次受理或刷新。

**短附加。** `TryStartOverlay(Gesture)` 不修改身体占用、不结束Speak，也不重播Idle。
启动阻挡、交互、保护移动／拖拽、身体交接临界区，以及附加槽已忙或待释放时会拒绝。
获准后调用原生单次动画；结束回调只设待释放，控制器后续Update再放开槽位，避免回调内重入。
已开始的短动作不能按来源即时物理取消；后来出现的交互或运动仍可开始，不被旧短动作阻塞。
如果素材未加载或原生结束回调丢失，逻辑槽可能保持忙；它不会用猜测片长提前宣布完成。
少见的停图、异步加载、快速重入边界未全部实机验证，必要时以场景重载复位。

**持续附加。** `TryStartSustainedOverlay(Kind)` 复用上述共同准入，还检查渲染图是否Ready。
控制器先复位函数结果、调用呈现、确认成功，之后才登记持续来源。
普通身体／Speak切换产生的Busy本身不会清持续槽；交互和保护移动／拖拽会在下一次Update撤销它。
短附加只有自身准入成功后才清持续槽；被拒绝的点头等请求不会让已有歪头消失。
`StopSustainedOverlay(Immediate)` 控制普通淡出或保护撤销所需的立即清理。

<a id="graph-presentation-lifecycle"></a>

### Presentation Lifecycle：加载后的开场与复位

**入口。** 图启用后先记录初始化任务，再由Update完成复位和开场阶段，不是每次更新都重新开场。
它调用Performance、Expression、Emotion、Mode等各自的Reset入口，清掉属于上一轮呈现的状态。
开场顺序为：NEUTRAL／待命／Idle0 → 闭眼停留 → 睁眼 → Idle1过渡 → 开放交互。
这里的Idle0是一般站姿，Idle1是双手前放，最后仍由Idle Animation负责原生播放。

本次保存的“闭眼停留（秒）”为1，“睁眼过渡（秒）”为0.5。
旧图注里仍有1.5秒的描述，实际调整应看这两个图变量；当前执行以变量及其数据线为准。
睁眼用Runtime启动计时器逐帧计算闭眼权重，不依赖一次性网格Tween留下的临时状态。
姿势阶段还读取Idle Animation的过渡时间，当前为0.5秒。

`WarudoDC.Presentation.ProcessStartupEyes(BlendShapes)` 接在面部追踪末端。
开场期间它修改选定的眼睑项，再交还原有唯一的原生Tracking写入器。
其他追踪键及口型键保留，不清空全局BlendShape，也不取得Native Lip Sync的呈现所有权。
本场景相应面部追踪图需要启用，才能经过这条眼睑合成路径。

开场的 `StartupBlocked` 由中央准入读取，阻挡交互、移动、切模式和身体表现。
这些请求不会排队，开场结束也不会重新播放它们。
阶段推进校验本代TimerId，旧流程不能推进或释放新一轮开场。
取消、结束或停图后眼睑函数立即透传原数据，不继续残留闭眼覆写。
`StartupPhase` 等状态是诊断用读数，不应靠手改它们来跳过初始化。

<a id="graph-idle-animation"></a>

### Idle Animation：五种基础站姿与统一回归

**用途。** 身体通道空闲时播放所选的基础待机，不按Emotion自动随机换姿势。
图启用、身体占用变化、内部 `WarudoDC.Animation.ReturnIdle()` 都走被动回归。
被动入口只在已经空闲时播放，不能借“回归”取消正在执行的普通表现。
主动入口是 `WarudoDC.Animation.TryReturnIdle(Variant)`，业务API为 `idle.return_normal_pose`。

| `variant` | 站姿 | 适用含义 |
| --- | --- | --- |
| -1 | 沿用当前选择 | 普通收尾，避免每次cleanup都重选站姿 |
| 0 | 一般 | 自然、日常的默认身体姿态 |
| 1 | 双手前放 | 平静待命；本次保存的选择也是1 |
| 2 | 双手后放 | 另一种平静站姿 |
| 3 | 单手撑腰 | 稳定日常站姿，和较强的双手叉腰表现分开 |
| 4 | 侧站抱臂 | 较收敛的侧身抱臂站姿 |

字段为必填Integer；旧的无字段请求不能继续原样发送。
非法值在终止当前表现之前拒绝，不改变动作、站姿偏好或计时。
0～4只在本地受理后写入“姿势差分”，以后普通动作结束继续回这一项。
-1只回归，不把偏好强改成某个默认值。
手工把本地图变量改成非法值时，视觉选择被动回退一般站姿，原值留下供修正。

主动回归先请求结束当前普通表现，再向统一身体权威确认能回Idle。
Speak活跃时会先走正常End，所选姿势留待End之后播放；受理成功不代表End已经完成。
Locomotion和Back Drag仍受保护，拒绝时公开入口不会改变所选站姿。
回归不切换自由／待命模式，不改Character Emotion，也不清无关的REQUESTED表情。

五个原生待机播放器共用 `TransitionTime=0.5`。
它们关闭“已播放时重启／重置”式切回，不复制素材预览样本中的自动复位逻辑。
不要为普通站姿配置循环触发器；持续播放由原生动画设施保持。

本图还集中观察Variables的正式 `ActMode` 变化：自由模式选Idle0，待命模式选Idle1。
待命归位途中模式尚未成功变为待命，不会提前切成双手前放。
启用时只捕获当前模式作为比较基准，同模式重复命令不再强制重选姿势。
启动阻挡期间的模式变化只记下，不应用，也不会解除阻挡后补播。
内部 `ApplyModeIdle` 可先改变保护占用释放后的站姿偏好，这与公开回归“拒绝则偏好不变”不同。

<a id="graph-locomotion-animation"></a>

### Locomotion Animation：把运动状态翻译成原生动画

**入口。** 每次Update只读已选择Runtime Locomotion资源的状态，再调用内部 `Locomotion.Refresh(Key)`。
本图不受理移动命令，不算角色轨迹，不执行根位移；这些归运动设施与业务准入负责。
先向Performance申请受保护Locomotion来源，获准后才播放对应的身体动画。
`LastAnimationKey` 阻止同阶段每帧重播，停用时按来源释放占用。

水平运动按步态选择Walk／Run；只有转向而没有水平步态时也沿现有步行表现分支。
无活动运动时释放该来源，由Idle的占用变化观察负责补回所选站姿。
不能给普通姿势发送更高“优先级”来强行夺走这个身体来源。

跳跃按Runtime物理阶段选择Start、Loop、End，而非由动画片长反过来驱动物理。
上升用Start，下落用Loop，落地收尾用End；直接观察到下落时从Loop开始，不补播未看到的阶段。
Jump01和Jump02两套样式共用Jump01_End，因为现有素材没有独立Jump02_End。
七个原生播放器共用 `TransitionTime=0.5`，不额外添加起跳前等待。

本次保存的 `Jump02MinPlanarSpeed=1.15` 米／秒。
它与只读 `PlannedPlanarSpeed` 比较，达到门槛才选跨步Jump02，较短／较慢或纯垂直方向用Jump01。
历史说明曾使用0.75，当前场景已调到1.15，应保留用户现值。
这个速度是接受运动命令时估算的腾空水平峰值参考，不是起步第一帧的实际速度。
短距离即使速度上限高，也可能因制动空间不足仍选Jump01。

一次起落开始后把样式锁在 `ActiveJumpVariant`，半空改目标或修改门槛不突然换套。
新一轮起落才重新选择；取消不会伪造落地，也不保证Start／End完整播放。
原生 `OnTransitionEnd` 只表示混合完成，既不是片段播完，也不是物理触地。
跳跃阶段和根运动可用，不代表所有素材衔接已自然；落地与跳跃观感仍应单独检查。

### 八张 Touch Reaction：接触表现与短触后效

这些图是按部位拆开的表现接收器，当前保存实际有八张，头顶和额头分别独立。
共同入口为 `WarudoDC.Performance.Contact.Signal(Source, Signal)`，先核对Source属于自己才执行。
它们不直接接原始鼠标或协议消息；对应Contact图和交互权威已经决定这一轮是否获准。
当前八图仅提交表情，不申请身体通道，不移动角色，也不取消已获准的普通姿势。

| Signal | 当前流程 |
| --- | --- |
| 1：获准接触 | 向Expression Controller提交本部位INTERACTION表情 |
| 2：短触收尾 | 向ContactAftereffect提交本部位短触表情及保持时间 |
| 3：持续结束 | 本模板没有额外流程输出，不制造新的身体动作或后效 |
| 4：禁用取消 | 按来源撤销本部位仍持有的短触后效 |

INTERACTION层的最终释放与接触生命周期仍由交互／表情权威处理，不能因为Signal3未接线就认定永不结束。
每图的 `ShortReactionSeconds` 当前均为0.65秒，按部位独立可调。
短触后效是表情层的保持，不是TTS时长、动画片长或接触长按阈值。

| 图名 | 部位与差异 | 本图提交的配置 |
| --- | --- | --- |
| <a id="graph-touch-head-top-reaction"></a>Touch Head Top Reaction | 头顶，与Forehead分离 | 头顶接触／头顶短触两套表情键 |
| <a id="graph-touch-forehead-reaction"></a>Touch Forehead Reaction | 额头 | 额头接触／额头短触两套表情键 |
| <a id="graph-touch-nose-reaction"></a>Touch Nose Reaction | 鼻子 | 鼻部接触及对应短触后效 |
| <a id="graph-touch-breast-reaction"></a>Touch Breast Reaction | 胸部 | 胸部接触及对应短触后效 |
| <a id="graph-touch-belly-reaction"></a>Touch Belly Reaction | 腹部 | 腹部接触及对应短触后效 |
| <a id="graph-touch-thigh-reaction"></a>Touch Thigh Reaction | 大腿 | 大腿接触及对应短触后效 |
| <a id="graph-touch-leg-reaction"></a>Touch Leg Reaction | 腿部，与大腿分离 | 腿部接触及对应短触后效 |
| <a id="graph-touch-butt-reaction"></a>Touch Butt Reaction | 臀部 | 臀部接触／臀部短触两套表情键 |

各表情选择从Constants的本部位键读取，不应把未接线缓存里的0误当作实际无表情。
修改某部位的ShortReactionSeconds只影响该部位后效；修改脸部形变应在角色原生表情资源完成。
若以后为某部位增加身体反应，仍须请求Performance许可，不能从Contact直接播放身体动画绕过保护。
这些图也不把外部REQUESTED来源升级为INTERACTION，来源所有权仍按本地交互边界保留。

<a id="graph-back-drag-swing"></a>

### Back Drag Swing：受保护拖拽的悬空姿态、升降和摆动

**入口。** Back Drag图调用 `WarudoDC.BackDrag.Presentation.Start()`／`Stop()`。
接触准入、鼠标拖动根位置、阶段业务事件仍在Back Drag；本图处理身体呈现。
Start先申请与Back Drag交互绑定的受保护身体来源，失败时不播放、不抬起，也不改呼吸／晃动开关。
获准后播放悬空姿态、控制Character Node 1升降，并在允许期间计算摆动。

固定层级为Character Node 0 → Character Node 1 → Back Drag Swing Anchor → Character Root →角色。
Node 0提供未受摆动污染的拖动输入；Swing Anchor只承担本图的局部位置和旋转补偿。
Character Transform仍按原方式复制Root世界姿态，没有恢复旧试验中直接写角色的旁路。
资源菜单的父级优先顺序也是运行结构的一部分，不能只改JSON数组而忽略原生资源顺序。

本图读取Node 0世界X作为横向驱动，经平滑后送入原生Float摆锤。
输入与摆锤输出的滞后换算成世界Z倾角，再转成父级局部旋转，换朝向仍保持世界左右摆动。
肩部支点来自原生骨骼世界位置与角色根变换的换算；骨骼局部位置不能直接当作角色根空间偏移。
倾斜同时使用位置补偿，减少支点因旋转漂移。
原生世界读取使用帧末缓存，不能把同一数据流中的数值解释成原子一致的实时Transform采样。

当前保留抬起约0.6秒后按交互计时器ID记录一次肩部偏移的策略。
记录的是抬起姿势后的根空间偏移，不是抓起前站姿点，也不是固定世界位置。
新一轮交互必须重新采样；旧交互标记不同就不能复用旧捕获。
“已记录肩部偏移”“肩部偏移所属交互”“本轮已捕获拖拽”是运行记录，不是建议手工调整的配置。

| 本图配置 | 当前保存值 | 影响 |
| --- | --- | --- |
| 启用摆动 | true | 撤销或启用本图的视觉摆动层 |
| 每单位滞后倾角 | 55 | 同等横向滞后产生的倾斜强度 |
| 最大倾角 | 16度 | 平滑后的最终倾角上限 |
| 进入退出平滑秒 | 0.22 | 抓起／松开时的摆动权重过渡 |
| 摆角平滑秒 | 0.10 | 缓和摆锤固定步长与画面更新差异 |
| 驱动平滑秒 | 0.06 | 摆锤输入位置的平滑 |
| TransitionTime | 0.25秒 | 悬空姿态的原生动画混合 |
| 禁用呼吸动画／禁用晃动动画 | false／true | 保留呼吸，临时停用原生晃动 |

升起0.3秒、普通回落0.4秒沿现有原生位置过渡设置。
升降和启停复位共用一个位置节点，后一次输入会取消这个节点自己的旧Tween，避免两个节点对拉。
松手保留正常回落和摆动淡出；其他交互接管、配置关闭或停图会清理本图自己的变换。
重置／禁用的特殊结束立即归零，不保留悬空位置。

**已经排除的旧用法。** 固定倾角开关、窗口日志、节流和诊断版本变量已从当前图撤除。
Runtime试验骨骼查询也已撤回，不能按旧记录重新要求安装那项控件。
早期固定高度像节拍器；按整轮锁住抓起前支点也被用户否定，因此保留抬起后记录。
父级更新顺序模型改善过一类延迟，但实际仍有抖动，不能把模型残差接近零当成渲染修复。
当前结构比初版流畅的反馈成立，残余抖动尚未解决；本手册不承诺拖动完全平稳。

### 普通姿势图的共同用法

下面14张普通姿势使用同一基本流程：收到具名API → 验证差分／时长 → Performance准入 → 原生待机播放。
获准后才应用本图的呼吸／晃动政策；没有额外的动画队列、逐帧姿势重发或根运动控制。
绝大多数姿势默认保持到明确回归Idle或另一普通身体动作接管。
持续姿势不是“单次动作卡住”，其选用的就是原生待机播放方式。

带“保持时长（秒，0为持续）”的图当前均为0。
设正数时交给既有Performance计时器到期，负数或无效配置在播放前拒绝。
Hands on Hips没有这个本地图时长字段，本来就是持续姿势。
公开API不再提供配对的 `.start`／`.end`；不要复制历史配置里的旧入口。
统一恢复使用 `idle.return_normal_pose(variant=-1)`，明确换基础站姿时才传0～4。

同来源重复请求经控制器刷新，原生播放器通常不因“已经在播放”而重启素材片头。
旧来源的Stop或停图清理不能释放后来接管的来源，结束后也不会恢复被替换的姿势快照。
普通接触若未申请新身体动作不会取消姿势；移动、Back Drag等获准接管时仍服从统一身体权威。
以下图均不自动配表情；Director可另选合适的REQUESTED表情，但应避免每个语音段都换脸。

| 图名／业务API | 用途与可调差分 | 当前政策与结束 |
| --- | --- | --- |
| Hands on Hips／`performance.hands_on_hips` | 身体前倾的双手叉腰，适合抗议、质问或刻意强调；不是Idle3的单手撑腰 | 过渡0.5秒，呼吸保留／晃动关闭；持续到回归或接管 |
| Invite Hug／`performance.invite_hug` | 双臂张开邀请拥抱或亲近，表达邀请本身，不代表已发生实际接触 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Shrug／`performance.shrug` | 摊手、耸肩式疑问或无奈，不强绑某一种情绪 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Protest Pose／`performance.protest` | 双拳置前的抗议姿势，强度比一般站姿更明显 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Hand on Chest／`performance.hand_on_chest` | 手放胸前，用于认真、自我表明或郑重回应 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Concern Pose／`performance.concern` | 手向身前的担心、关切姿势；脸部含义仍由表情决定 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Cute Pose／`performance.cute` | 必填variant：0双手置于身前，1身体向前倾；只接受0／1 | 过渡0.5秒，呼吸／晃动均保留；时长0持续 |
| Refuse Pose／`performance.refuse` | 必填variant：0普通拒绝，1双臂比叉的明确拒绝；只接受0／1 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Flex Muscles／`performance.flex` | 展示肌肉、精神十足地表现力量感 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Slouch／`performance.slouch` | 无精打采或泄气的垮肩姿态，不改真实生理模型值 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Facepalm／`performance.facepalm` | 右手扶额／掩面，适合无语、尴尬或头疼的表现 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Hand Behind Head／`performance.hand_behind_head` | 右手放脑后，适合不好意思、苦笑或自嘲的身体态度 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Reserved Pose／`performance.reserved` | 保留、拘谨的稳定姿态，适合收敛的回应 | 过渡0.5秒，呼吸保留／晃动关闭；时长0持续 |
| Show Back／`performance.show_back` | 面向屏幕翘起臀部展示背部的特定姿势，不负责根朝向或位移 | 过渡0.5秒，呼吸／晃动均保留；时长0持续 |

这些“当前政策”来自本次保存值，早期模板的“默认不关闭晃动”不再代表用户调配后的现状。
替换素材时先保留动作的准入和释放路径，再观察新素材是否需要不同的混合时间。
只换脸不会自动释放身体姿势；只回身体Idle也不会清无关来源后来提交的脸。

**Hands on Hips的区别。** 平时稳定单手撑腰请选Idle3；本图的双手叉腰有明显前倾和强调感。
本图不自动生气，不以素材名为依据改Character Emotion，也不额外增加回归后的站姿偏好。

**Cute Pose与Refuse Pose的区别。** variant是一次明确选择，不是程度插值或随机种子。
非法variant必须在打断旧表现之前拒绝，不能先清场再回退到0。
前倾卖萌只改变身体动画，不把角色根向用户移动；比叉拒绝也不是移动或交互拒绝命令。

**Show Back的调用位置。** 当前Adapter把它放在主LLM工具 `performance_show_back`，未列入Director动作目录。
不要因为其他普通姿势多由Director选择，就自行把这项重新加入Director或当作通用cleanup。
其受理结果与其他本地状态工具一样，不表示身体混合已完成。

<a id="graph-thinking-pose"></a>

### Thinking Pose：持续的思考组合

业务API为 `performance.thinking`，必填Integer `variant`；内部入口为 `WarudoDC.Thinking.Start(Variant)`。
0使用原思考姿势，1使用双腿交叉的思考姿势；非法值在任何占用或表情变更前拒绝。
函数显式接收本次参数，不读API观察器上一次保留的字段缓存。

流程先申请普通身体来源，再提交本图拥有的持续思考REQUESTED表情并播放所选素材。
这是一组可见的思考姿态和表情，不是宣布LLM正在内部推理，也不更改Character Emotion事实。
当前 `TransitionTime=0.5`，呼吸和晃动均保留。
它没有固定保持时长，直到回归Idle、新身体动作接管、所属生命周期结束或停图清理。

本图接收Performance.Ended并按思考表情来源释放，不恢复旧表情快照。
未申请身体动作的交互不会单独打断已获准思考姿态；移动／Back Drag仍受保护规则约束。
通过 `autonomy.state.result` 返回的是本地获准或拒绝，不等待“思考完成”。
普通cleanup使用 `idle.return_normal_pose(-1)`，不需要已退役的成对思考结束API。

<a id="graph-wave"></a>

### Wave：原生待机播放，按样式估算挥手次数

业务API为 `performance.wave`，必填 `variant` 与 `count` 两个Integer字段。
内部 `WarudoDC.Wave.Begin(Variant, Count)` 在准入前检查全部参数。
count为1～20；API传0时先换成本图 `DefaultWaveCount`，当前默认3，默认值本身也必须有效。
不再使用旧名 `performance.wave_once`，也没有旧message字段的兼容含义。

| variant | 样式 | 当前一次挥手估算 | count=3的期限 |
| --- | --- | --- | --- |
| 0 | 举臂挥动 | 约0.600秒 | 约1.800秒 |
| 1 | 双手挥动 | 约0.733秒素材内含三次，按0.244秒／次 | 约0.733秒 |
| 2 | 单手挥动 | 约0.267秒 | 约0.800秒 |

本图保留三个 `播放角色待机动画`，不是原生单次串播，也不是持久叠加层方案。
播放器按原速运行， `RestartIfPlaying=true`、`ResetIfPlaying=false`，共用 `TransitionTime=0.5`。
经Performance获准后，用“单次估算×次数”作为统一表现期限，不在每轮结束时重播。
计时从受理／开始播放时算，包含淡入；退出沿原生回Idle混合，不另加淡出计时器。
因此次数是估算目标，可见动作受过渡、帧边界、子周期不均匀及淡出影响，不能视为逐轮完成确认。

Wave占普通身体通道，会接替其他普通姿势，但不能抢Locomotion或Back Drag。
期限、回归Idle、新身体接管和停图都按原来源释放；旧计时不能结束后来动作。
当前呼吸／晃动均保留，不自动变脸，也没有独立速度控件。
单手原速较快；延长期限只增加动作次数，不能假装把挥手减速。

**保留的失败经验。** 把循环素材改成原生单次并在结束回调续播，用户看到了突跳、残影和更差的接缝。
把回调延迟到下一Update避免了重入，却没有保留上一轮动画的混合状态，不能证明视觉连续。
早期主场景持久层版也曾实测不通过；后来简化的独立持久层样本反而较流畅，不能概括为持久层机制都失败。
用户最后明确选择当前原生待机播放加计时计次，当前生产图不需要迁回任何一种试验设施。
双手素材最初被当成“一份素材等于一次”，导致约3N次；数值测试通过也没有验证这层视觉语义。
修正为素材长度除以3后，用户认为双手／单手三次频率接近，可作为当前阶段基线。
这个认可不等于全部次数、所有打断和快速重入都逐项实机通过。

<a id="graph-nod"></a>

<a id="graph-shake-head"></a>

<a id="graph-surprise"></a>

<a id="graph-swing-body"></a>

### Nod、Shake Head、Surprise、Swing Body：短附加动作

四张图共同调用Performance的独立短附加准入。
获准才运行原生 `播放角色单次动画`，从真正的 `OnAnimationEnd` 通知 `OverlayEnded`。
不申请普通身体、不主动回Idle，不用时长计时器猜测动画结束。
当前均为 `Additive=true`、`Masked=false`、`Speed=1`、`Weight=1`， `TransitionTime=0.1`。
节点虽然保存了Head遮罩列表，Masked=false时列表不生效；当前不是“只准头部骨骼变化”。

| 图／API | 入口和差异 | 持续与结束 |
| --- | --- | --- |
| Nod／`performance.nod` | 必填count仅1或2；内部 `WarudoDC.HeadGesture.Nod(Count)` 选择一次／两次的两份完整素材 | 不是重复调用一次素材拼次数；所选原生素材自然结束 |
| Shake Head／`performance.shake_head` | 无字段；内部 `WarudoDC.HeadGesture.shake_head()` | 一次摇头，自然结束，不自动表示业务拒绝 |
| Surprise／`performance.surprise` | 无字段；内部 `WarudoDC.AdditiveGesture.surprise()` | 一次惊讶身体颤动，不自动换惊讶脸 |
| Swing Body／`performance.swing_body` | 无字段；内部 `WarudoDC.AdditiveGesture.swing_body()` | 一次摇摆身体，和Back Drag的物理摆动不是同一设施 |

它们共享一个互斥槽，忙时拒绝，不排队，也不让同片段多个附加动作挤在一起。
Idle、Speak或普通姿势可继续作为底层；动作获准不会重播底层Start或刷新Idle。
持续歪头等若存在，只有这次短附加成功获准才先撤销；短附加结束后不自动恢复旧持续项。
动作开始之后，后来的交互或保护移动不承诺即时撤回原生单次动画。
本地拒绝与逻辑释放和实际骨骼混合是不同事实，不能凭槽位空闲证明画面完全静止。

**原生设置的区别。** 单次播放决定瞬时条目的生命周期；Additive决定附加混合方式；遮罩决定哪些部位参与。
关闭Additive不会把持久层变成单次；打开遮罩也不等于已经选对附加混合。
早期试验把遮罩列表误读为排除列表，已纠正：勾选项是允许参与的部位。
用户后来更喜欢附加开启、无遮罩时全身轻微带动的自然感，故当前四图保留这个方向。
常用点头／摇头获认可，不代表所有素材与每一套底姿、Gaze及保护状态都完成视觉验收。

<a id="graph-laugh"></a>

### Laugh：四种笑容与身体组合

业务API为 `performance.laugh`，必填Integer `variant=0..3`；内部入口为 `WarudoDC.Laugh.Begin(Variant)`。
一次请求选择一整组，不需要分别发笑脸和身体，也没有配套结束API。

| variant | 身体素材方向 | 专属REQUESTED表情 |
| --- | --- | --- |
| 0 | 低头笑 | 低头羞涩笑 |
| 1 | 抬头笑 | 仰头大笑 |
| 2 | 低头笑 | 低头坏笑 |
| 3 | 抬头笑 | 仰头坏笑 |

四张脸是角色原生第1层中的四个独立REQUESTED项，不是四个新原生层。
它们最初从FUN复制，但用户已经分别精调；当前应保留保存的形变、约束和追踪设置，不重置回初值。
BASE层的原FUN仍保持自己的身份，旧版借用SHY／PLAYFUL／SMUG的表格已经不是现行映射。
内部表情代码和资源身份无需进入API参数，Adapter仍只发variant。

流程先校验差分及表情请求控制器可用，再申请共享短附加槽。
身体准入后提交持续到本次动画结束的笑容请求，确认新代次／来源／Active确实受理后才播放原生单次。
若脸的申请未受理，就释放逻辑预留并不播放，不能假装一次笑已完成。
低头和抬头各用一份完整非循环素材，不通过重复拼接增加次数。
当前原生设置仍为附加开启、无遮罩、速度／权重1、过渡0.1秒。

动画自然结束时仅在表情来源和捕获代次仍匹配时撤销自己这次笑脸。
中途被新表情接管，甚至同来源有更新代次，旧回调也不能清掉新脸。
不恢复旧REQUESTED快照，BASE／INTERACTION及组合覆盖仍由Expression Controller决定。
清脸之后按原短附加的待释放／下一Update路径放开槽位，不重播底姿。
停图也只清理自己仍持有的脸，不伪造原生结束；少见丢回调和异步加载边界仍待观察。

<a id="graph-ojou-laugh"></a>

### Ojou Laugh：持续大小姐姿态，可选一次临时笑

业务API为 `performance.ojou_laugh`，必填Boolean `laugh_once`。
内部入口 `WarudoDC.OjouLaugh.Begin(LaughOnce)` 同时协调普通身体与可选短附加预留。
底姿使用原生待机循环，当前 `TransitionTime=0.5`，保留呼吸和晃动。

| 参数 | 身体与表情的实际行为 |
| --- | --- |
| false | 进入／保持大小姐肢体姿态；不申请、刷新、清理或重新夺取表情 |
| true | 保持底姿，同时申请短附加槽，临时切专属笑容并抬头笑一次 |

false不依赖表情请求控制器可用；即使抬头已在播，重复false也不刷新那次临时脸。
重复请求已有底姿不重播其片头。
true若短附加槽忙就拒绝本次请求，不排队，不承诺笑完后再补一次。
只有确认短附加与临时脸均可用时才播放抬头， `LaughTransitionTime=0.1`。
其原生单次仍是附加开启、无遮罩、速度／权重1。

**关键结束区别。** 抬头笑结束，释放本次仍拥有的临时脸和短附加槽，持续的身体底姿继续。
这覆盖旧版“专属表情跟着姿态一直保持”的说明，当前不是那种生命周期。
临时脸按来源和序列号清理，后来接管的新脸不受旧结束回调影响，也不恢复旧脸快照。
用 `idle.return_normal_pose(-1)` 或其他获准身体动作结束底姿。
底姿提前结束／停图也清理本次仍持有的脸，但不假装原生头部动画已物理取消。
如果失败发生在旧身体已被替换之后，不承诺把旧姿势恢复回来。
常用搭配与快速重复、停图、素材异步加载等边界仍应分开验证。

<a id="graph-sustained-overlay"></a>

### Sustained Overlay：三种持续附加与统一清除

业务API与内部函数一一对应，四项均无业务字段。

| API／内部入口 | 目标表现 | 何时自然结束 |
| --- | --- | --- |
| `performance.tilt_neck`／`WarudoDC.SustainedOverlay.tilt_neck()` | 歪头，非循环素材末态保持 | 不自然结束，等待槽位被清除或接管 |
| `performance.look_away`／`WarudoDC.SustainedOverlay.look_away()` | 低头望向一边，非循环素材末态保持 | 不自然结束，等待槽位被清除或接管 |
| `performance.tremble`／`WarudoDC.SustainedOverlay.tremble()` | 持续颤抖，原生循环 | 不自然结束，等待槽位被清除或接管 |
| `performance.sustained.clear`／`WarudoDC.SustainedOverlay.clear()` | 清执行时当前持续项 | 按普通权重过渡淡出 |

图内呈现入口为 `WarudoDC.Performance.SustainedOverlay.Apply(Kind, Immediate)`，由Controller调用。
Controller唯一维护持续来源，本图只实施固定层权重并报告成功，不能绕过它直接登记所有权。
当前 `TransitionTime=0.5`，实际使用限在0～2秒。
三层原生附加开启、无遮罩、速度1；每层共用一个原生权重节点，不为每个入口制造并行Tween。
同项重复不重播、不刷新权重过渡；换项时目标层到1，其他层到0，过渡中允许短暂混合。

隐藏层仍推进素材，重新显现不保证从片头重新播。
歪头／低头保持当前末态，颤抖继续本来循环；不额外冻结、换Source或增加逐帧权重包络。
这些是持久原生层，和前面短附加的原生单次条目不能相互混称。

持续附加的主要结束路径有三类：

1. **明确结束或换项。** clear淡出当前槽；另一持续项获准后正常淡变到新项。
2. **短附加接管。** 点头、摇头、笑等真正准入成功后先清持续槽，结束后不恢复旧项。
3. **保护或生命周期撤销。** 交互／保护移动／拖拽在下一Controller Update立即撤销；停图或重载也清三层。

普通身体／Speak切换本身不会清掉持续附加，因此可在底姿和说话动作之间继续保持。
clear只结束持续槽，不释放身体、不重播Idle、不停Speak，也不清表情。
clear作用于执行时的当前槽，不是按某个旧事件或旧代次撤销原请求。
若新持续项已接管，迟到的cleanup仍可能清当前项，编排时必须避免机械逐段发送clear。
开始项可用immediate／media_started；统一结束还可用media_completed和有明确必要的cleanup。

图启用先清保存权重，不重放旧来源；停图通知Controller并直接归零自己的三个层。
本次保存已删除四个无接线手动预览入口及其注释，API入口、四个内部行为函数和清理路径仍完整。
不要按旧试验截图寻找那些按钮，或把缺少预览控件误判成缺少正式能力。
歪头隔离试验及主场景常用兼容已有正面反馈，部分底姿组合仍不自然。
正式Controller／API流程与所有加载异常边界尚不能由结构测试推断全部实机通过。

<a id="graph-speak-controller"></a>

### Speak Controller：整轮说话阶段和来源关联

Speak是说话时的身体动作，不负责TTS、音频队列、口型或台词文本。
九张风格图分别提供Start／Loop／End，本控制器管理何时调用它们。
普通身体的唯一权威仍是Performance Controller，Speak使用一个普通身体来源，不另建第二个占用系统。

共同申请入口为 `WarudoDC.Speak.Request(Style, PlanId, StartSeconds, EndSeconds)`。
风格图把本次来源表现计划ID和计算后的阶段等待时间传入；不能用上一次观察器缓存重新绑定当前轮。
自动Speak只接受与当前Timeline音频表现计划匹配的来源；直接手动Control的PlanId为空。
内部 `StyleReady(Style)` 确认对应风格图可用， `Play(Style, Phase)` 广播给匹配图播放片段。

**正常一轮。** 申请身体成功 → Start → 本图专用阶段计时器到期 → Loop原生保持。
同一表现计划中再次选择同一风格，Start／Loop期间保持，不因每段TTS开始而重播Start。
不同风格获准时直接进入新风格Start，不先补旧风格End。
Loop不按素材的五秒周期重发；原生循环与业务“还在说话”是两回事。

只有关联的整轮Timeline以Consumed正常结束，才请求当前Speak进入End。
End的阶段等待完成后释放Speak身体来源，再被动回到已经选择的Idle。
这不是每段TTS结束都收尾，也不是LLM停止生成时立刻收尾。
传输提交、设备消费和物理出声不是同一完成事实；这里依据的是本地Timeline的整轮消费事件。
无音频文本轮不承诺自动Speak生命周期。

取消、替换、失败、截断，或更高优先级身体接管时清旧阶段，不补旧End，不在后来抢回Idle。
计时器TimerId不匹配时仅清自己的状态，不能取消他人的计时器。
图停用时内联清理自己的阶段，再调用外部身体权威，避免依赖已停用图自己的函数。
`Generation`、`Phase`、`OwnedTimerId` 等为内部守卫，不能手改来控制动画。

主动 `idle.return_normal_pose` 遇到Speak会走正常End，选定的站姿在End后播放。
返回受理并不代表End、Idle过渡或实体动作完成。
无Timeline来源的手动 `speak.*` 不被不相关批次结束，测试后用回归Idle收尾。
各风格图停用调用 `WarudoDC.Speak.StopStyle(Style)`，只中断仍属于该风格的当前Speak。

### 九张Speak风格图：同一阶段模板，不同身体态度

每张图入口均是同名 `speak.*` API，无业务字段；来源PlanId由本地Link传递，不让业务字段伪造。
图收到请求先计算阶段等待，再交Speak Controller许可；本图不自行夺取身体占用。
获准后只响应匹配Style／Phase，播放本风格的三个原生待机片段。
Start和End允许重新开始，Loop保持原生循环，避免语音切段时重复小动作。
这些图没有自动绑定表情或Character Emotion，身体态度和面部可由编排合理组合。

| 图名 | API／Style | 用途和相邻风格区别 |
| --- | --- | --- |
| <a id="graph-speak-02解释型说明"></a>Speak 02（解释型说明） | `speak.explaining`／2 | 用手势解释、说明或梳理事情，适合有长度的讲解 |
| <a id="graph-speak-03激动地说"></a>Speak 03（激动地说） | `speak.excited`／3 | 活跃、热切地分享，身体表达兴奋，不自动切兴奋脸 |
| <a id="graph-speak-04冷静分析"></a>Speak 04（冷静分析） | `speak.calm`／4 | 克制、沉稳的分析或认真说明，比丰富手势交谈收敛 |
| <a id="graph-speak-05手指镜头"></a>Speak 05（手指镜头） | `speak.scolding`／5 | 指向屏幕并强调、责备或告诫，态度较强 |
| <a id="graph-speak-06手指镜头轻佻"></a>Speak 06（手指镜头，轻佻） | `speak.pointing`／6 | 轻松、打趣地指向，与scolding的责备含义区分 |
| <a id="graph-speak-07手部动作丰富地"></a>Speak 07（手部动作丰富地） | `speak.chatty`／7 | 活泼聊天、叙述，持续丰富手部动作 |
| <a id="graph-speak-08双手交叉"></a>Speak 08（双手交叉） | `speak.pouting`／8 | 抱臂赌气、抗议或不满地说，区别于稳定Idle4 |
| <a id="graph-speak-09双手交叉冷静"></a>Speak 09（双手交叉，冷静） | `speak.composed`／9 | 抱臂、从容、保留意见地说，区别于pouting的抗议态度 |
| <a id="graph-speak-10害羞地说"></a>Speak 10（害羞地说） | `speak.shy`／10 | 拘谨、羞怯或含蓄回应，身体不自动决定脸部表情 |

九图均用自己的本地图变量，当前数值相同：

| 参数 | 当前值 | 实际作用 |
| --- | --- | --- |
| TransitionTime | 0.5秒 | 进入该Speak的Start混合 |
| StartSeconds | 2秒 | Start完整素材片长，计算交接时间的依据 |
| EndSeconds | 约2.666667秒 | End完整素材片长，计算退出时间的依据 |
| LoopTransitionTime | 0.12秒 | Start→Loop混合 |
| EndTransitionTime | 0.25秒 | Loop→End混合 |
| SeamSafetySeconds | 0.05秒 | 避免旧循环片段在仍有权重时绕回片头的提前量 |

Start的等待为 `StartSeconds - LoopTransitionTime - SeamSafetySeconds`，当前1.83秒。
End的等待为 `EndSeconds - Idle的实际TransitionTime - SeamSafetySeconds`，当前约2.116667秒。
这里保留完整片长配置，不应把StartSeconds直接改成已经减过混合的1.83。
原生End进入混合和End释放后的Idle混合仍在继续，本地阶段结束不等于视觉过渡瞬间结束。
替换素材时要同时更新片长与各混合参数，确保阶段等待为正；不合格配置会被申请校验拒绝。
一轮开始后应保持这些参数稳定，不在Loop中途靠改时长追赶正在播放的音频。

**曾经发生的切段问题。** 原Start／Loop／End素材自身都带循环属性。
旧流程等满片长才交接，再淡出0.5秒，旧Start或End可能在仍有权重时绕回开头，用户看到了手部抽动。
现在将进入、Loop交接、End交接分开，并为下一段混合提前留出余量。
用户已反馈修订后衔接改善，长回复中Director也能触发Speak。
这个反馈不覆盖任意卡顿、全部快速重入、无音频和禁用时序；0.05秒余量也不是任意掉帧保证。

Director宜在media_started选定整轮身体风格，跨分段保持，不机械地逐段换风格或追加Idle。
现有首段预设里，讲解与分析可分别选explaining／calm，愉快、期待等也有对应Speak组合。
首段选择与语义搭配仍需长期观察，不要把“已安装预设”当成每次LLM都会正确选用。
中性默认可以完全不选Speak，短促点头或笑也不要求先进入某一说话风格。

### 调整与检查时的实用顺序

先确认动作所属的是普通身体、短附加还是持续附加，再决定用回归Idle、原生自然结束还是持续clear收尾。
预览原生播放节点会绕过正式准入，只能验证素材，不能据此证明移动／拖拽保护或API生命周期正确。
先单独看一轮，再看与Idle／Speak和Gaze共存，最后看换项、重复、打断与重新启用。
动作计数、期限和函数连线正确是结构证据，渲染是否自然仍需实际观察。
保留用户已经调好的表情、晃动政策和跳跃阈值，不用旧生成初值覆盖当前保存。
发现回调缺失或未知加载异常时先留存有界日志，不通过完整实时场景快照探查，也不自动重试危险查询。
本节依据已保存场景与既有观察记录整理，没有在本轮重新执行Unity编译、Mod导出、Warudo加载或设备验收。


## 配套 Adapter 配置怎样参与场景

这份业务包不是把99个API都变成同一种“请求”。需要分清三种用途：

| 配置 | 信息流向 | 作用 |
| --- | --- | --- |
| LLM钩子 | 场景交互 → Link → Adapter → 主LLM输入 | 解释发生了什么，附上本地读数，等待与其他输入合并 |
| LLM工具 | 主LLM → Adapter → Link → 业务蓝图，必要时回.result | 主动选择移动、模式、思考或展示等有业务意义的操作 |
| 剧本导演行为 | 回复片段 → Director → Stream → Timeline → 本地控制事件 | 选择该段怎样说、做什么表现；不阻塞主LLM等一个动画播完 |

API只是双方能识别的事件与字段目录。钩子、工具和导演可以引用同一项API，但生命周期和目的不同。Director决定过某动作，不等于收到过本地执行成功回执；当前专用Director回执路径尚未接入，不能把前文动作当作角色实际做完的报告。

### 两条上下文规则：一次说明含义，事件只带事实

所有33条交互钩子都引用`runtime.feelings`和`runtime.activity`，目标均为主LLM系统提示。固定解释按本轮实际参与输入的引用去重加入，不在每条交互正文里重复长篇规则。

- `runtime.feelings`：负荷与适应为0～1、舒适为-1～1；适应不是好感，舒适不是Character Emotion；可用标记为false时忽略相应数值。它们是事件发生时的本地参考，不要求智能体朗读或机械表现。
- `runtime.activity`：精力0～100；`normal/low/unavailable`说明本地精力状态；`standby/free/unknown`说明活动模式。低精力不自动拒绝命令，模式也不等于关闭明确命令的执行器。

每条事件的提示词模板负责具体语义，如“戳鼻子”“摸额头”“牵手持续较久”。它们并非仅按API名字自动生成，腹部等条目含作者个人场景描述；换角色设定时应改钩子文案，而不是认为插件已经知道这类行为的意义。

格式化示例`{{feeling_load:.2f}}`表示保留两位小数，`{{energy:.1f}}`表示一位。它仅简化提示词显示，不把模型内部数据截断到同样精度，也不改变Control中的Float值。

### 33条钩子的共同收集规则

当前全部启用，目标为当前对话，采用等待后提交、按顺序追加、冷却0秒、最大驻留30秒、到期作废；没有逐条等待策略覆写。

这里有两种不同时间：平台输入收集窗口决定何时把积累输入交给LLM；钩子最大驻留决定这条输入能留多久。平台的等待／最长收集时间不在这个业务包里，不能根据30秒驻留断言每次交互都等30秒才发。

钩子内容本身是一条普通用户输入。收集时保留独立身份，在正式向LLM提交时才合并；所属生成取得结果后才结算，不能请求刚发出就把原输入丢掉。若收到匹配的取消事件，未结算钩子会被取消；若它已经参加在途生成，则按原输入策略重新生成不含被取消项的请求。

八类持续触碰钩子各绑定对应`.stroke.cancel`作为取消API。取消事件自身又有独立钩子，用“已经松开”的新信息进入当前对话；“取消A”与“B自己也产生输入”是两个独立配置事实。取消目标已结算时不撤回过去回复，但B自己的合法输入仍可登记。

| 钩子名称 | 事件Key | 取消该钩子的事件 | 具体语义（模板首句） |
| --- | --- | --- | --- |
| 鼻子 · 触碰 | interaction.contact_nose.touch | 未绑定 | (用户戳了一下你的鼻子。) |
| 鼻子 · 持续触碰 | interaction.contact_nose.stroke | interaction.contact_nose.stroke.cancel | (用户按着你的鼻子。) |
| 鼻子 · 松开 | interaction.contact_nose.stroke.cancel | 未绑定 | (用户按了一会你的鼻子，松开了。) |
| 额头 · 触碰 | interaction.contact_forehead.touch | 未绑定 | （用户轻触了一下你的额头。） |
| 额头 · 持续抚摸 | interaction.contact_forehead.stroke | interaction.contact_forehead.stroke.cancel | （用户正在持续抚摸你的额头。） |
| 额头 · 松开 | interaction.contact_forehead.stroke.cancel | 未绑定 | （用户摸了一会你的额头，松开了。） |
| 胸部 · 触碰 | interaction.contact_breast.touch | 未绑定 | （用户轻触了一下你的胸部。） |
| 胸部 · 持续抚摸 | interaction.contact_breast.stroke | interaction.contact_breast.stroke.cancel | （用户正在持续抚摸你的胸部。） |
| 胸部 · 松开 | interaction.contact_breast.stroke.cancel | 未绑定 | （用户摸了一会你的胸部，松开了。） |
| 腹部 · 触碰 | interaction.contact_belly.touch | 未绑定 | （用户轻触了一下你的腹部，好像想扒你的衣服。） |
| 腹部 · 持续抚摸 | interaction.contact_belly.stroke | interaction.contact_belly.stroke.cancel | （用户正在持续抚摸你的腹部。） |
| 腹部 · 松开 | interaction.contact_belly.stroke.cancel | 未绑定 | （用户摸了一会你的腹部，松开了。） |
| 大腿 · 触碰 | interaction.contact_thigh.touch | 未绑定 | （用户轻触了一下你的大腿。） |
| 大腿 · 持续抚摸 | interaction.contact_thigh.stroke | interaction.contact_thigh.stroke.cancel | （用户正在持续抚摸你的大腿。） |
| 大腿 · 松开 | interaction.contact_thigh.stroke.cancel | 未绑定 | （用户摸了一会你的大腿，松开了。） |
| 小腿 · 触碰 | interaction.contact_leg.touch | 未绑定 | （用户轻触了一下你的小腿。） |
| 小腿 · 持续抚摸 | interaction.contact_leg.stroke | interaction.contact_leg.stroke.cancel | （用户正在持续抚摸你的小腿。） |
| 小腿 · 松开 | interaction.contact_leg.stroke.cancel | 未绑定 | （用户摸了一会你的小腿，松开了。） |
| 臀部 · 触碰 | interaction.contact_butt.touch | 未绑定 | （用户轻触了一下你的臀部。） |
| 臀部 · 持续抚摸 | interaction.contact_butt.stroke | interaction.contact_butt.stroke.cancel | （用户正在持续抚摸你的臀部。） |
| 臀部 · 松开 | interaction.contact_butt.stroke.cancel | 未绑定 | （用户摸了一会你的臀部，松开了。） |
| 背部拖拽 · 持续一会儿 | interaction.back_drag.phase_2 | 未绑定 | （用户正从你的背部拖拽你，这次交互已达到 {{duration_ms}} 毫秒的持续阈值。） |
| 背部拖拽 · 持续较久 | interaction.back_drag.phase_3 | 未绑定 | （用户还没有松开你的背部，拖拽已持续较久，达到 {{duration_ms}} 毫秒的阈值。） |
| 背部拖拽 · 长时间持续 | interaction.back_drag.phase_4 | 未绑定 | （用户依然没有结束背部拖拽，这次交互已经持续很久，达到 {{duration_ms}} 毫秒的阈值。） |
| 光环旋转 · 持续一会儿 | interaction.halo_rotate.phase_2 | 未绑定 | （用户抓住你的光环进行拖动，让你转动朝向；交互已达到 {{duration_ms}} 毫秒的持续阈值。） |
| 光环旋转 · 持续较久 | interaction.halo_rotate.phase_3 | 未绑定 | （用户仍在拖动你的光环，让你转动朝向；这次交互持续较久，已达到 {{duration_ms}} 毫秒的阈值。） |
| 光环旋转 · 长时间持续 | interaction.halo_rotate.phase_4 | 未绑定 | （用户还没有松开你的光环，旋转交互已经持续很久，达到 {{duration_ms}} 毫秒的阈值。） |
| 牵手 · 持续一会儿 | interaction.hand_drag.phase_2 | 未绑定 | （用户自然地牵着你的手，已经牵了一小会儿；这次牵手达到 {{duration_ms}} 毫秒的持续阈值。） |
| 牵手 · 持续较久 | interaction.hand_drag.phase_3 | 未绑定 | （用户仍然牵着你的手，还没有松开；你们已经牵手一段时间，达到 {{duration_ms}} 毫秒的持续阈值。） |
| 牵手 · 长时间持续 | interaction.hand_drag.phase_4 | 未绑定 | （用户一直自然地牵着你的手，没有松开；这次牵手已经持续很久，达到 {{duration_ms}} 毫秒的持续阈值。） |
| 头顶 · 触碰 | interaction.contact_head_top.touch | 未绑定 | （用户轻触了一下你的头顶，不是额头。） |
| 头顶 · 持续抚摸 | interaction.contact_head_top.stroke | interaction.contact_head_top.stroke.cancel | （用户正在温柔抚摸你的头顶，不是额头。） |
| 头顶 · 松开 | interaction.contact_head_top.stroke.cancel | 未绑定 | （用户结束了对你头顶的持续抚摸。） |

### 9个工具：动作受理和业务完成分开

移动、转向工具等待各自`.result`，默认结果超时30秒；待命返回最多等120秒，自由／思考／回归姿势类状态结果为10秒。展示背部只确认投递受理，不等待动作结束。

`autonomy.state.result`共用状态字段`status/detail/mode/thinking`。移动和转向使用较小的状态词`completed`等，而状态工具可能使用`Succeeded/Rejected`等；不要仅凭两端都叫status就忽略具体API的大小写与合同。

结果关联使用原控制事件ID作为反向事件的关联ID；Runtime运动命令ID是另一层身份，只在本地匹配完成。让两者共用一个字段，或在动画结束时临时读取“最近收到的事件”，都会把并发或替代后的结果发错对象。

| 工具名 / 当前启用 | Control API / 输入字段 | 结果API / 超时 | 实际用途 |
| --- | --- | --- | --- |
| autonomy_move_to · 开 | autonomy.move_to；target: Vector3 | autonomy.move_to.result；30.0秒 | 用途：需要角色前往指定位置时调用，目标为世界坐标；水平方向参考范围为 x∈[-1,1]。实际移动受场景限位和占用规则约束。<br>结果：以业务结果 status=completed 为完成；拒绝、超时或未知不视为完成。 |
| autonomy_move_by · 开 | autonomy.move_by；delta: Vector3 | autonomy.move_by.result；30.0秒 | 用途：需要角色从当前位置移动指定距离时调用，参数为位移增量，不是目标坐标；移动后世界坐标的水平方向参考范围为 x∈[-1,1]。实际移动受场景限位和占用规则约束。<br>结果：以业务结果 status=completed 为完成；拒绝、超时或未知不视为完成。 |
| autonomy_rotate_to · 开 | autonomy.rotate_to；yaw: Float | autonomy.rotate_to.result；30.0秒 | 用途：需要角色转向指定朝向时调用，参数为绕世界 Y 轴的绝对角度，范围[0,360]；0°面朝屏幕，不移动位置。<br>结果：以业务结果 status=completed 为完成；拒绝、超时或未知不视为完成。 |
| autonomy_rotate_by · 开 | autonomy.rotate_by；delta_yaw: Float | autonomy.rotate_by.result；30.0秒 | 用途：需要角色相对当前朝向转动时调用，参数为绕世界 Y 轴的角度增量，范围[-180,180]；按最短路径转动，不移动位置。<br>结果：以业务结果 status=completed 为完成；拒绝、超时或未知不视为完成。 |
| autonomy_standby · 开 | autonomy.mode.standby；无业务字段 | autonomy.state.result；120.0秒 | 用途：需要角色回到待命模式时调用，会沿本地返回流程归位。待命时可交互、可接受行动指令，自主活动意图较低；返回过程可能需要一段时间。<br>结果：status=Succeeded 且 mode=0 表示切换完成；拒绝时不重复调用。 |
| autonomy_free · 开 | autonomy.mode.free；无业务字段 | autonomy.state.result；10.0秒 | 用途：需要角色进入自由模式时调用。自由时不接受触碰交互，可接受行动指令，自主活动意图较高；切换模式本身不会自动发起移动。<br>结果：status=Succeeded 且 mode=1 表示切换完成；拒绝时不重复调用。 |
| performance_thinking · 开 | performance.thinking；variant: Integer | autonomy.state.result；10.0秒 | 用途：即将查资料或执行耗时工具时，先进入思考姿态及配套表情。variant=0为原思考姿势，1为双腿交叉思考；持续时间由场景决定，设为0时保持到结束或被打断。不再需要时调用 performance_return_idle，variant=-1。<br>结果：返回本地受理结果，不等待思考姿态结束。 |
| performance_return_idle · 关 | idle.return_normal_pose；variant: Integer | autonomy.state.result；10.0秒 | 用途：结束当前普通肢体表现，回归所选待命站姿；也可明确选择新的待命站姿。不切换待命／自由模式，不打断移动或背部拖拽，不清除独立表情；思考动作自带的表情随所属表现结束。<br>姿势选择：<br>- -1：沿用当前选择，仅结束动作时优先使用。<br>- 0·一般：自然、中性，适合日常聊天或没有明显表现意图时。<br>- 1·双手前放：端正、礼貌，适合待命、倾听和认真回应。<br>- 2·双手后放：拘谨、含蓄，可表达羞涩期待，也可表达低落克制。<br>- 3·单手撑腰：放松、自信或俏皮，适合熟稔聊天与打趣，不等同于生气。<br>- 4·侧站抱臂：保留、怀疑、不满或自我保护，不必是强烈愤怒。<br>选择0..4后会保持为后续待机基础；结合语境选择，不因每段语调略变而频繁切换。<br>结果：返回本地受理结果，不代表动画过渡已经结束。 |
| performance_show_back · 开 | performance.show_back；无业务字段 | 不等业务结果；30.0秒 | 用途：用户提出要求或你主动决定时，朝屏幕翘起臀部，展示背部与臀部。这是主动展示身体的姿态，不是普通转身，也不自动改变表情。展示结束后调用 performance_return_idle，variant=-1。<br>结果：仅确认事件受理，不提供动作完成回执。 |

**当前一项值得注意的配置：** `performance_return_idle`工具注册还在，但已禁用。思考／展示背部的用途说明仍推荐调用它，因此主LLM不能仅凭说明就真的获得这个工具。对应API和导演行为仍可用；是否重新启用工具，应由使用者在Adapter配置中明确决定，而不是手册悄悄改动。

### 四种语调与54个导演行为

Director开启。语调目录为`neutral/happy/sad/angry`，neutral默认；四项都声明支持强度，默认0.5。本场景Emotion Controller只消费标签映射，未消费强度。支持某一TTS后端按语调变声也不意味着每个Provider都理解强度参数。

导演模板先放本场景的动作、Speak、表情和持续附加规则，再注入七块固定信息：有限前文、前文语调、前文动作、当前片段正文、可用语调目录、有效行为目录、结构化输出合同。

固定信息由Adapter填写，用户可以重新包装、调整位置和增加说明；不应该手写一个假的前文执行回执。Director串行保持决策链，TTS可以并发；音频生产并发不意味着动作导演也可以互不知情地同时给相邻片段定调。

本场景模板的几个关键决定是：

1. 行为不是单选菜单。可将独立肢体与独立表情组合，但不要同一时刻选择两个互相替换的身体姿势。
2. 自带表情的思考、Laugh、一次抬头大小姐笑，不再叠加另一条REQUESTED请求争抢同一层；纯大小姐站姿不自带持续笑脸。
3. 普通姿势可以跨片段保持，不需要每段开始／结束重发；Speak同风格重复选择不重播Start。
4. `cleanup_cues`不是每段必填的“回正”。只有确有收尾必要才选已注册行为，不能清理另一段后来建立的新状态。
5. 持续附加、普通身体和原生短附加是不同槽；停止持续附加不应把身体、Speak或表情一起清空。

| 导演行为 | 入站API | 允许时机 | 参数约束与搭配 |
| --- | --- | --- | --- |
| 挥手 | performance.wave | media_started | 挥动手臂向对方示意。适用于打招呼、告别、主动吸引注意力，以及兴奋地赞同或庆祝的片段。普通陈述、严肃或低落语境，以及相邻片段已经使用过相同动作时不要选择。 姿势差分必须指定variant：0举臂挥动（原动作）、1双手挥动、2单手挥动；按count指定次数，以各样式单次挥手估算时长×次数计时，不附带表情。 count必填；通常填0沿用本地默认次数，需要时明确指定1–20轮。不对每轮重复发起行为。<br>搭配参考：轻松招呼可配expression.tender，兴奋示意可配expression.expectant；礼貌告别也可保持已有表情。<br>参数：variant=挥手差分：0举臂、1双手、2单手；仅接受0、1、2。不自动改变表情。；count=挥手次数：1–20；0使用Wave的DefaultWaveCount（默认3）。字段必填。按各样式单次挥手估算时长计时：双手按素材长度/3换算，其他按一个素材周期。含淡入，帧边界与淡出存在误差；单手原速较快。 |
| 回归平时待命动作 | idle.return_normal_pose | immediate、media_started、media_completed | 根据当前语境、心境和与对方的亲近程度选择待命姿势。姿势只表达身体态度，不替代表情；同一种情绪可以搭配不同姿势。<br>- 0·一般：自然、中性的放松站姿。适合日常聊天和不紧张的情境；没有明显的表现意图时优先选择；<br>- 1·双手前放：端正、礼貌，带有正式待命或认真倾听的感觉。适合接待、等待指示、认真回应；<br>- 2·双手后放：略显拘谨、含蓄，像是在收敛自己的情绪。既适合高兴时的羞涩与期待，也适合难过时的低落与克制；<br>- 3·单手撑腰：放松而有主见，带有熟稔、自信或轻微俏皮的感觉。适合自在聊天、打趣或从容表态，不等同于生气；<br>- 4·侧站抱臂：带有距离感和自我保护意味。适合保留、怀疑、不满或暂时不愿亲近的情境，不必表现为强烈愤怒。（从容或有距离感）；站姿不等于固定情绪。选择0..4后该站姿继续作为待机基础，不必每段重发。不切活动模式、不打断移动或背部拖拽，不清理独立表情；仍按执行时当前普通表现处理，不是旧来源的专属撤销。<br>- 仅结束当前动作、没有明确换姿势意图时，使用 variant=-1 沿用当前选择。不要因为每段措辞或语调略有变化就频繁切换站姿。<br>参数：variant=待机姿势差分：-1=沿用当前所选；0=一般，1=双手前放，2=双手后放，3=单手撑腰，4=侧站抱臂。0..4仅在本地受理后更新所选姿势，之后回归待机继续使用；不自动随机切换。 |
| 惊讶 | expression.surprised | media_started | 眉毛明显抬起、双眼睁大；用于意外、惊喜或突然的转折。 |
| 疑惑 | expression.confused | media_started | 眉头一高一低，眼神变成困惑的漫画式样，嘴呈小三角；用于没听懂或需要解释。 |
| 无语 | expression.speechless | media_started | 平眉、半睁眼，显得淡漠又无奈；用于克制的吐槽或一时不知如何评价。 |
| 尴尬 | expression.embarrassed | media_started | 眉尾略垂、眼神略弯并泛起红晕；用于出糗、被拆穿或社交窘迫。 |
| 期待 | expression.expectant | media_started | 眉眼扬起、眼睛亮晶晶，小嘴微张；用于兴奋地期待答案、礼物或下一步互动。 |
| 得意 | expression.smug | media_started | 眉毛扬起、眼睛略眯，嘴呈猫式微笑；用于小小炫耀或受到夸奖后的得意。 |
| 怀疑 | expression.skeptical | media_started | 略皱眉、略眯眼，嘴呈小三角；用于听懂了却不太相信、质疑真实性或动机。 |
| 不满 | expression.displeased | media_started | 眉头压低、眼睛略眯，抿嘴鼓气般闹别扭；用于抗议、责怪或表达不满。 |
| 嫌弃 | expression.disgusted | media_started | 皱眉眯眼，面部带压暗的阴影；用于嫌弃、排斥某个对象或行为。 |
| 担忧 | expression.worried | media_started | 眉尾下垂、双眼略睁大；用于关切对方、犹豫或预感不妙。 |
| 害怕 | expression.fearful | media_started | 眉毛高挑又下垂，眼睛明显睁大、嘴角下弯；用于直接受惊或害怕。 |
| 委屈 | expression.aggrieved | media_started | 眉尾低垂、眼神带委屈，小嘴向下撇；用于被误解、冷落或想得到安慰。 |
| 害羞 | expression.shy | media_started | 眉眼柔和，眼睛微弯并显露羞涩；用于亲近或夸奖后的正向害羞。 |
| 温柔 | expression.tender | media_started | 眉眼轻柔、眼睛微弯，嘴角带温和笑意；用于安慰、关爱或亲昵回应。 |
| 释然 | expression.relieved | media_started | 眉头舒展、眼睛半睁，显得放松；用于解除担忧、确认安全或松一口气。 |
| 俏皮 | expression.playful | media_started | 眉毛轻扬，双眼明显弯成笑眼；用于善意逗弄、调侃或小小使坏。 |
| 疲惫 | expression.tired | media_started | 眉尾略垂、眼皮明显低垂；用于困倦、疲劳或勉强打起精神。 |
| 无聊 | expression.bored | media_started | 平眉、半睁眼并略微眯起，显得兴致缺缺；用于敷衍或等待得无聊。 |
| 思考 | expression.thinking | media_started | 眉头轻微一高一低、眼睛略收拢，显得在斟酌；用于回忆或权衡。 |
| 进入思考状态 | performance.thinking | media_started | 请求思考表情和姿态，本地占用可能拒绝；场景保持时长0时持续，正数限时。需要结束时统一选择“回归平时待命动作”；不要相邻每段重复。 姿势差分必须指定variant：0原思考姿势、1双腿交叉；两者均保留原思考表情组合，不表示模型内部推理。<br>参数：variant=姿势差分：0=原思考姿势，1=双腿交叉思考；仅接受0或1。仍为持续思考姿态及所属思考表情组合，统一回归待机结束。 |
| 双手叉腰 | performance.hands_on_hips | media_started | 身体明显前倾并双手叉腰，是较强烈的肢体表现，适合刻意强调、抗议、质问或催促。平时稳定的单手撑腰站姿选择idle.return_normal_pose的variant=3。本行为默认持续，表情由expression.*按语境独立搭配；不要相邻片段重复触发。需要回归所选待机时使用回归API的variant=-1。明确新表现获准后可替换，无新肢体要求的交互保持当前姿势。<br>搭配参考：认真抗议可配expression.displeased，熟稔打趣可配expression.playful；不因叉腰就强制生气。 |
| 邀请拥抱 | performance.invite_hug | media_started | 张开双臂朝向对方，表达邀请拥抱、求安慰或希望靠近；只是姿态，不代表已发生接触。<br>搭配参考：亲昵邀请可配expression.tender，求安慰可配expression.aggrieved。 |
| 耸肩 | performance.shrug | media_started | 耸肩摊手，表示不知道、无所谓或无奈。<br>搭配参考：无奈可配expression.speechless，不解可配expression.confused。 |
| 赌气抗议 | performance.protest | media_started | 双拳置于身前，以身体姿态表达赌气、不满或抗议，不自动设置生气表情。<br>搭配参考：赌气可配expression.displeased，委屈抗议可配expression.aggrieved。 |
| 认真表态 | performance.hand_on_chest | media_started | 右手放在胸口，适合认真保证、承诺或坚定表态。<br>搭配参考：安慰或真诚承诺可配expression.tender；郑重表态也可保持已有表情。 |
| 担心关切 | performance.concern | media_started | 右手放在身前，身体略显关切，适合担心、询问或安慰。<br>搭配参考：担心可配expression.worried，确认安全后的安慰可配expression.tender。 |
| 卖萌 | performance.cute | media_started | 以双手前放或身体向前倾的可爱姿态表达撒娇、卖萌或主动关注。<br>搭配参考：撒娇可配expression.shy，善意逗弄可配expression.playful。<br>参数：variant=姿势差分：0=双手置于身前，1=身体向前倾；选择一种，不移动角色根位置。 |
| 拒绝 | performance.refuse | media_started | 身体表达拒绝；普通姿势较含蓄，双臂比叉更明确，可配严肃、羞怯或玩笑表情。<br>搭配参考：明确抗议可配expression.displeased，害羞婉拒可配expression.shy；按拒绝缘由选择。<br>参数：variant=姿势差分：0=普通拒绝姿态，1=双臂比叉的明确拒绝；仅接受0或1。 |
| 展示力量 | performance.flex | media_started | 展示肌肉，以肢体表达炫耀力量、得意或搞怪。<br>搭配参考：炫耀可配expression.smug，搞怪可配expression.playful。 |
| 垂头丧气 | performance.slouch | media_started | 垂头、身体松垮的低落姿态，适合沮丧、失望或泄气。<br>搭配参考：失望可配expression.aggrieved，困乏可配expression.tired；不把失落一律当作疲劳。 |
| 扶脸懊恼 | performance.facepalm | media_started | 右手扶住脸，表达无奈、懊恼或对眼前情况难以评价。<br>搭配参考：吐槽可配expression.speechless，出糗可配expression.embarrassed。 |
| 扶后脑勺 | performance.hand_behind_head | media_started | 右手放在后脑勺，适合尴尬、不好意思或有些困惑；不是持续挠头动作。<br>搭配参考：不好意思可配expression.embarrassed，不解可配expression.confused。 |
| 拘谨 | performance.reserved | media_started | 双手握拳收在胸前，呈现拘谨、紧张或羞怯的身体姿态。<br>搭配参考：正向羞怯可配expression.shy，不安可配expression.worried。 |
| 说话 · 解释型说明 | speak.explaining | media_started | 以手势解释、说明或梳理一件事，适合有一定长度的讲解。<br>搭配参考：耐心说明可配expression.tender，疑问分析可配expression.thinking；普通讲解可保持已有表情。 |
| 说话 · 激动地说 | speak.excited | media_started | 以较活跃的肢体姿态兴奋地讲述，适合热切分享或激动回应。<br>搭配参考：热切分享可配expression.expectant，带点炫耀可配expression.smug。 |
| 说话 · 冷静分析 | speak.calm | media_started | 以克制、沉稳的手势分析或说明，适合认真讨论。<br>搭配参考：权衡可配expression.thinking，关切讨论可配expression.worried；平稳陈述可保持已有表情。 |
| 说话 · 指向并强调 | speak.scolding | media_started | 手指朝向屏幕，带有强调、责备或告诫的身体态度。<br>搭配参考：责怪可配expression.displeased，质疑可配expression.skeptical。 |
| 说话 · 轻松指向 | speak.pointing | media_started | 以较轻松的指向手势说话，适合打趣或轻佻地强调；不是指责动作。<br>搭配参考：善意打趣可配expression.playful，轻松得意可配expression.smug。 |
| 说话 · 丰富手势交谈 | speak.chatty | media_started | 说话时手部动作丰富，适合较活泼的叙述和聊天。<br>搭配参考：亲切聊天可配expression.tender，热切叙述可配expression.expectant。 |
| 说话 · 抱臂赌气地说 | speak.pouting | media_started | 抱臂并带有赌气、抗议或不满的身体态度说话。<br>搭配参考：闹别扭可配expression.displeased，受委屈可配expression.aggrieved。 |
| 说话 · 抱臂冷静地说 | speak.composed | media_started | 抱臂、克制而冷静地说话，适合保留意见或从容分析。<br>搭配参考：保留意见可配expression.skeptical，冷静思量可配expression.thinking。 |
| 说话 · 害羞地说 | speak.shy | media_started | 以拘谨、羞怯的身体姿态交谈，适合不好意思或含蓄回应。<br>搭配参考：正向含蓄可配expression.shy，社交出糗可配expression.embarrassed。 |
| 点头 | performance.nod | immediate、media_started、media_completed | 点头表达确认、认可或理解。count必填：1为轻点一次，2为较积极的连续两次；仅接受1或2。一次调用播放对应完整素材，不通过反复请求拼次数。原生附加动作，与Idle/Speak及表情并存；自然结束，不放入cleanup_cues。同片段至多选择一种点头/摇头，避免相邻片段反复使用；本地忙或受保护状态可能拒绝。<br>搭配参考：安慰式认同可配expression.tender，松口气后的确认可配expression.relieved；普通确认不必新增表情。<br>参数：count=点头次数，仅接受1或2；1为轻点一次，2为较积极的连续两次。字段必填，手动入口初值1；不通过重复播放拼接次数。 |
| 摇头 | performance.shake_head | immediate、media_started、media_completed | 摇头表达否定、不赞同、不愿意或对某个判断的纠正；不等同全身拒绝姿态，不自动决定角色表情。 原生附加短动作，可与当前Idle/Speak及表情并存。一次调用播完整素材，自动自然结束，不放入cleanup_cues。每片段至多选择一种点头/摇头，避免相邻片段反复使用；本地忙或受保护状态可能拒绝。<br>搭配参考：无奈否认可配expression.speechless，抗议可配expression.displeased；普通纠正不必新增表情。 |
| 惊讶身体反应 | performance.surprise | immediate、media_started、media_completed | 身体短促颤动一下，表达突然惊讶、被吓一跳或出乎意料的反应；不是持续颤抖。仅控制身体，不自动切换惊讶表情，表情可由独立行为搭配。 原生附加动作，与当前Idle/Speak及表情并存，一次播完整素材后自然结束，不放入cleanup_cues。同片段至多选择一个原生附加动作，不与点头/摇头同时请求；避免相邻片段反复触发。本地槽忙或受保护占用时可能拒绝。<br>搭配参考：意外可配expression.surprised，明确受惊害怕才选expression.fearful。 |
| 摇摆身体 | performance.swing_body | immediate、media_started、media_completed | 身体轻轻摇摆一段，适合轻松、愉快、俏皮或略带撒娇的表达；不是移动、转向或持续站姿。不用于严肃或受惊的语境，不自动改变表情。 原生附加动作，与当前Idle/Speak及表情并存，一次播完整素材后自然结束，不放入cleanup_cues。同片段至多选择一个原生附加动作，不与点头/摇头同时请求；避免相邻片段反复触发。本地槽忙或受保护占用时可能拒绝。<br>搭配参考：轻松撒娇可配expression.shy，俏皮逗弄可配expression.playful。 |
| 笑（四种组合） | performance.laugh | immediate、media_started、media_completed | 结合语境选择笑的表情和肢体动作：variant=0低头羞涩笑（害羞、被夸或含蓄地笑），1仰头大笑（开怀、有趣、明显的喜悦），2低头坏笑（低头调皮、打趣或暗自得意），3仰头坏笑（较张扬地调皮或得意）。这是笑容与原生附加动作的整体请求，无需额外请求相同笑容。一次完整素材自然结束，不放入cleanup_cues，不需要结束请求。仅在确实适合笑时选择；同片段不与其他原生附加动作堆叠，不在相邻片段反复笑。共享附加槽忙或保护占用时可能拒绝；角色Emotion仍由主对话决定，本行为不改其Emotion。<br>参数：variant=笑的组合：0=低头羞涩笑；1=仰头大笑；2=低头坏笑；3=仰头坏笑。必填Integer，仅接受0–3。无需另发笑容请求或结束请求。 |
| 大小姐笑 | performance.ojou_laugh | immediate、media_started、media_completed | 切换为大小姐式持续站姿，适合高傲、自信、得意地调侃或“哦呵呵”式笑。laugh_once=false仅进入/保持肢体站姿，完全不请求或刷新表情；true临时切专属笑容并仰头笑一次，头部动作结束只释放本次仍持有的笑容，肢体站姿继续保持。重复请求不重播已持有的底姿。它是完整身体姿态，不是普通laugh的附加差分。不要仅因片段音频结束就自动恢复站姿；明确需要恢复时使用idle.return_normal_pose或其他身体姿势接管。无需额外请求同一笑容，表情不改变Character Emotion。附加槽忙时带true的本次请求可能被拒绝，不排队。移动、拖拽及交互保护保持，不可强制越过本地权限。<br>参数：laugh_once=是否抬头笑一次：true临时请求专属笑容并播放一次附加抬头笑，结束只释放本次仍持有的表情，肢体姿态继续；false只进入／保持肢体姿态，不碰当前表情。必填Boolean。 |
| 歪头 | performance.tilt_neck | immediate、media_started | 持续轻轻歪头，适合疑惑、好奇或亲昵倾听。不是反复点头。可配expression.confused或expression.shy，不自动切表情。 持续附加，与其他持续附加互相替换，重复同项不刷新。仅在有明确语义时选择；通过performance.sustained.clear结束，不用idle.return_normal_pose收尾。短附加成功接管或交互／保护移动会撤销，恢复后不自动补播。 |
| 低头望向一边 | performance.look_away | immediate、media_started | 持续低头望向一边，适合害羞回避、尴尬或低落克制。可配expression.shy、expression.embarrassed或expression.aggrieved，不自动切表情。 持续附加，与其他持续附加互相替换，重复同项不刷新。仅在有明确语义时选择；通过performance.sustained.clear结束，不用idle.return_normal_pose收尾。短附加成功接管或交互／保护移动会撤销，恢复后不自动补播。 |
| 持续颤抖 | performance.tremble | immediate、media_started | 持续颤抖，适合害怕、紧张或明显不安；不用于普通开心或一般叙述。可配expression.fearful或expression.worried，不自动切表情。 持续附加，与其他持续附加互相替换，重复同项不刷新。仅在有明确语义时选择；通过performance.sustained.clear结束，不用idle.return_normal_pose收尾。短附加成功接管或交互／保护移动会撤销，恢复后不自动补播。 |
| 结束持续附加 | performance.sustained.clear | immediate、media_started、media_completed | 明确结束执行时当前持续附加，不结束身体姿势或Speak、不清表情。不是按原请求身份的专属撤销。确需在片段末尾结束辅助表现时可作为cleanup_cues；若要跨片段保持或已经编排新持续附加，不要机械清理。 |

### 26个首段预设：跳过首段导演的可选捷径

首段优化开启，植入方式为`hybrid`，同时使用系统与临时提示。主LLM可在开头给出约定的emoji组合，Adapter匹配后使用该预设的语调与表现，避免首段必须再等一次Director。

它不是让主LLM随便输出emoji当动作，也不是将emoji原样展示在字幕中。预设标记、动作ID、参数和允许时机需要保持有效。未命中时走既有兜底/导演流程，不能把解析失败当成某个随机表情。

当前neutral预设是默认，中性开场没有强行加动作。其他预设按语义选择Speak、表情、挥手等。下表中的行为名已经由配置ID解析为实际API，修改名称时仍应通过Pages保存对应注册，不要直接把API Key冒充`action_id`。

| 名称 / 标记 | 语调 / 强度 | 普通cues | cleanup_cues | 选用语境 |
| --- | --- | --- | --- | --- |
| 中性开场（默认） 🎭😐 | neutral / 0.5 | 无 | 无 | 平稳、自然地开始回复。 |
| 讲解开场 🎭📖 | neutral / 0.5 | speak.explaining@media_started | 无 | 开场即以解释型手势介绍、说明或梳理一件事，适合有一定长度的讲解；不用于普通短答或仅表示没听懂。 |
| 分析开场 🎭🧐 | neutral / 0.5 | speak.calm@media_started | 无 | 开场即以冷静、克制的手势认真分析、权衡或提出判断；不同于困惑追问，也不必带有不信任。 |
| 愉快交谈开场 🎭✨ | happy / 0.7 | speak.chatty@media_started | 无 | 以轻快、积极的状态开始回复。<br>肢体表现：以丰富手势活泼地交谈，适合分享和叙述；普通简短肯定不必选择。 |
| 悲伤开场 🎭😢 | sad / 0.6 | 无 | 无 | 以低落、克制的状态开始回复。 |
| 生气强调开场 🎭💢 | angry / 0.7 | speak.scolding@media_started | 无 | 以坚定、有张力的状态开始回复。<br>肢体表现：手指朝向对方，适合明确责备、告诫或强调边界；不因语气略显生气就选择。 |
| 惊讶开场 🎭😮 | neutral / 0.65 | expression.surprised@media_started | 无 | 开场即对意外消息或出乎意料的转折表示惊讶，短暂睁大眼睛抬眉；普通强调不选。 |
| 疑惑开场 🎭🤔 | neutral / 0.5 | expression.confused@media_started | 无 | 开场即表示不理解、不确定或准备追问，短暂露出疑惑表情。 |
| 无语开场 🎭😑 | neutral / 0.4 | expression.speechless@media_started | 无 | 开场即表示克制的无奈、冷场或轻微吐槽，短暂露出无语表情；不是愤怒或悲伤。 |
| 尴尬开场 🎭😳 | happy / 0.55 | expression.embarrassed@media_started；speak.shy@media_started | 无 | 开场即因出糗、被拆穿或社交窘迫而尴尬；亲近和夸奖后的正向羞涩选择害羞开场。<br>肢体表现：以不好意思、拘谨的说话姿态面对出糗或社交窘迫，仍保留尴尬表情。 |
| 愉快挥手开场 🎭👋 | happy / 0.75 | performance.wave@media_started {"variant":0,"count":0} | 无 | 愉快地打招呼、告别或主动示意时，开场挥手一次；普通回答不必挥手。 |
| 热切期待开场 🎭🤩 | happy / 0.7 | expression.expectant@media_started；speak.excited@media_started | 无 | 开场即表现期待：期待答案、礼物或接下来的互动，积极地等待；不是已经得到结果后的得意。<br>肢体表现：以激动的肢体姿态谈论期待的事情，适合热切分享；仅安静等待时不选。 |
| 得意开场 🎭😏 | happy / 0.65 | expression.smug@media_started；speak.pointing@media_started | 无 | 开场即表现得意：被夸奖、完成事情或小小炫耀后的自得；不是所有开心片段都需要得意。<br>肢体表现：以轻松指向的手势自信地打趣或小小炫耀，不是责备对方。 |
| 怀疑开场 🎭🤨 | neutral / 0.55 | expression.skeptical@media_started；speak.composed@media_started | 无 | 开场即表现怀疑：已经听懂但不太相信、质疑真实性或动机；不理解含义时使用困惑。<br>肢体表现：抱臂、冷静地质疑或保留意见，带一点距离感；不是没听懂。 |
| 不满开场 🎭😒 | angry / 0.55 | expression.displeased@media_started；speak.pouting@media_started | 无 | 开场即表现不满：抗议、责怪或表达个人边界的即时反应；不是改变角色长期基础情绪。<br>肢体表现：抱臂赌气地抗议、闹别扭或表达不愿接受；不是强烈威胁或恐惧。 |
| 嫌弃开场 🎭🤢 | neutral / 0.65 | expression.disgusted@media_started | 无 | 开场即表现嫌弃：对具体对象或行为表示排斥、嫌弃或不想靠近；不用于单纯没听懂。 |
| 担忧开场 🎭😟 | sad / 0.55 | expression.worried@media_started | 无 | 开场即表现担忧：担心对方、犹豫或预感不妙；是关切，不等于自己已经受到直接威胁。 |
| 害怕开场 🎭😨 | neutral / 0.75 | expression.fearful@media_started | 无 | 开场即表现害怕：受到威胁、恐惧或受惊；普通意外消息使用惊讶，不要把惊喜解释为恐惧。 |
| 委屈开场 🎭🥺 | sad / 0.65 | expression.aggrieved@media_started | 无 | 开场即表现委屈：被误解、受到冷落或希望得到安慰；区别于一般悲伤、责怪和愤怒。 |
| 害羞开场 🎭🙈 | happy / 0.55 | expression.shy@media_started；speak.shy@media_started | 无 | 开场即表现害羞：被亲近或夸奖后的正向羞涩、欲言又止；出糗和社交窘迫使用尴尬。<br>肢体表现：以拘谨、含蓄的说话姿态回应亲近或夸奖。 |
| 温柔开场 🎭🥰 | happy / 0.5 | expression.tender@media_started | 无 | 开场即表现温柔：安慰、关爱或温柔亲昵地回应；无需每次礼貌回答都添加此表情。 |
| 释然开场 🎭😌 | neutral / 0.4 | expression.relieved@media_started | 无 | 开场即表现释然：放下顾虑、确认安全或松了一口气；关注担忧解除的转折，而非所有普通开心。 |
| 俏皮开场 🎭😜 | happy / 0.7 | expression.playful@media_started；speak.pointing@media_started | 无 | 开场即表现俏皮：调侃、逗弄或善意使坏时的俏皮反应；不要用于严肃关切或对方明显不适时。<br>肢体表现：以轻松指向的手势善意逗弄或调侃，不是告诫或指责。 |
| 疲惫开场 🎭😪 | neutral / 0.35 | expression.tired@media_started | 无 | 开场即表现疲惫：对话中展示疲劳、困倦或勉强打起精神；仅是表现请求，不证明本地生理状态已产生。 |
| 无聊开场 🎭🥱 | neutral / 0.35 | expression.bored@media_started | 无 | 开场即表现无聊：兴致缺缺、敷衍或等待太久；不同于身体疲倦、愤怒或受到冒犯。 |
| 思考开场 🎭💭 | neutral / 0.5 | expression.thinking@media_started；performance.thinking@media_started {"variant":0} | idle.return_normal_pose@media_completed（收尾） {"variant":-1} | 开场即表现思考：斟酌答案、回忆或权衡时的轻微思考表现；只是表演，不表示展示模型内部推理。 |

首段预设只省掉其中一个阶段，不省输入收集、ASR、主LLM首段正文、TTS及设备输出。不能因预设命中，就把“首音仍慢”统归为Link或Director。调优时分别记录请求进入、输入提交、首段稳定、导演/预设完成、TTS/规范化、候选获选、PCM交付及设备消费时间。

## 99项API完整目录

以下方向相对于Link：**入站**是Adapter发给场景，**出站**是场景发给Adapter。表内字段只有业务字段，事件ID和关联事件ID在公共信封里，不需要再手工给每项添加`reply_to`字段。

“接线位置”列由当前场景的事件收发节点核对；它不是要求每项都给主LLM或导演开放。API目录、工具授权和导演行为目录是三套不同配置。

| 方向 | 事件Key | 字段 | 接线位置 |
| --- | --- | --- | --- |
| 入站 | autonomy.mode.free | 无业务字段 | Autonomy State Tools |
| 入站 | autonomy.mode.standby | 无业务字段 | Autonomy State Tools |
| 入站 | autonomy.move_by | delta: Vector3 | Move By |
| 入站 | autonomy.move_to | target: Vector3 | Move To |
| 入站 | autonomy.rotate_by | delta_yaw: Float | Rotate By |
| 入站 | autonomy.rotate_to | yaw: Float | Rotate To |
| 入站 | expression.aggrieved | 无业务字段 | Aggrieved |
| 入站 | expression.bored | 无业务字段 | Bored |
| 入站 | expression.confused | 无业务字段 | Confused |
| 入站 | expression.disgusted | 无业务字段 | Disgusted |
| 入站 | expression.displeased | 无业务字段 | Displeased |
| 入站 | expression.embarrassed | 无业务字段 | Embarrassed |
| 入站 | expression.expectant | 无业务字段 | Expectant |
| 入站 | expression.fearful | 无业务字段 | Fearful |
| 入站 | expression.playful | 无业务字段 | Playful |
| 入站 | expression.relieved | 无业务字段 | Relieved |
| 入站 | expression.shy | 无业务字段 | Shy |
| 入站 | expression.skeptical | 无业务字段 | Skeptical |
| 入站 | expression.smug | 无业务字段 | Smug |
| 入站 | expression.speechless | 无业务字段 | Speechless |
| 入站 | expression.surprised | 无业务字段 | Surprised |
| 入站 | expression.tender | 无业务字段 | Tender |
| 入站 | expression.thinking | 无业务字段 | Thinking |
| 入站 | expression.tired | 无业务字段 | Tired |
| 入站 | expression.worried | 无业务字段 | Worried |
| 入站 | idle.return_normal_pose | variant: Integer | Idle Animation |
| 入站 | performance.concern | 无业务字段 | <a id="graph-concern-pose"></a>Concern Pose |
| 入站 | performance.cute | variant: Integer | <a id="graph-cute-pose"></a>Cute Pose |
| 入站 | performance.facepalm | 无业务字段 | <a id="graph-facepalm"></a>Facepalm |
| 入站 | performance.flex | 无业务字段 | <a id="graph-flex-muscles"></a>Flex Muscles |
| 入站 | performance.hand_behind_head | 无业务字段 | <a id="graph-hand-behind-head"></a>Hand Behind Head |
| 入站 | performance.hand_on_chest | 无业务字段 | <a id="graph-hand-on-chest"></a>Hand on Chest |
| 入站 | performance.hands_on_hips | 无业务字段 | <a id="graph-hands-on-hips"></a>Hands on Hips |
| 入站 | performance.invite_hug | 无业务字段 | <a id="graph-invite-hug"></a>Invite Hug |
| 入站 | performance.laugh | variant: Integer | Laugh |
| 入站 | performance.look_away | 无业务字段 | Sustained Overlay |
| 入站 | performance.nod | count: Integer | Nod |
| 入站 | performance.ojou_laugh | laugh_once: Boolean | Ojou Laugh |
| 入站 | performance.protest | 无业务字段 | <a id="graph-protest-pose"></a>Protest Pose |
| 入站 | performance.refuse | variant: Integer | <a id="graph-refuse-pose"></a>Refuse Pose |
| 入站 | performance.reserved | 无业务字段 | <a id="graph-reserved-pose"></a>Reserved Pose |
| 入站 | performance.shake_head | 无业务字段 | Shake Head |
| 入站 | performance.show_back | 无业务字段 | <a id="graph-show-back"></a>Show Back |
| 入站 | performance.shrug | 无业务字段 | <a id="graph-shrug"></a>Shrug |
| 入站 | performance.slouch | 无业务字段 | <a id="graph-slouch"></a>Slouch |
| 入站 | performance.surprise | 无业务字段 | Surprise |
| 入站 | performance.sustained.clear | 无业务字段 | Sustained Overlay |
| 入站 | performance.swing_body | 无业务字段 | Swing Body |
| 入站 | performance.thinking | variant: Integer | Thinking Pose |
| 入站 | performance.tilt_neck | 无业务字段 | Sustained Overlay |
| 入站 | performance.tremble | 无业务字段 | Sustained Overlay |
| 入站 | performance.wave | count: Integer；variant: Integer | Wave |
| 入站 | speak.calm | 无业务字段 | Speak 04（冷静分析） |
| 入站 | speak.chatty | 无业务字段 | Speak 07（手部动作丰富地） |
| 入站 | speak.composed | 无业务字段 | Speak 09（双手交叉，冷静） |
| 入站 | speak.excited | 无业务字段 | Speak 03（激动地说） |
| 入站 | speak.explaining | 无业务字段 | Speak 02（解释型说明） |
| 入站 | speak.pointing | 无业务字段 | Speak 06（手指镜头，轻佻） |
| 入站 | speak.pouting | 无业务字段 | Speak 08（双手交叉） |
| 入站 | speak.scolding | 无业务字段 | Speak 05（手指镜头） |
| 入站 | speak.shy | 无业务字段 | Speak 10（害羞地说） |
| 出站 | autonomy.move_by.result | position: Vector3；status: String | Move By |
| 出站 | autonomy.move_to.result | position: Vector3；status: String | Move To |
| 出站 | autonomy.rotate_by.result | rotation: Vector3；status: String | Rotate By |
| 出站 | autonomy.rotate_to.result | rotation: Vector3；status: String | Rotate To |
| 出站 | autonomy.state.result | detail: String；mode: Integer；status: String；thinking: Boolean | Autonomy State Tools |
| 出站 | interaction.back_drag.phase_2 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Back Drag |
| 出站 | interaction.back_drag.phase_3 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Back Drag |
| 出站 | interaction.back_drag.phase_4 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Back Drag |
| 出站 | interaction.contact_belly.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Belly |
| 出站 | interaction.contact_belly.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Belly |
| 出站 | interaction.contact_belly.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Belly |
| 出站 | interaction.contact_breast.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Breast |
| 出站 | interaction.contact_breast.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Breast |
| 出站 | interaction.contact_breast.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Breast |
| 出站 | interaction.contact_butt.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Butt |
| 出站 | interaction.contact_butt.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Butt |
| 出站 | interaction.contact_butt.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Butt |
| 出站 | interaction.contact_forehead.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Forehead |
| 出站 | interaction.contact_forehead.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Forehead |
| 出站 | interaction.contact_forehead.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Forehead |
| 出站 | interaction.contact_head_top.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Head Top |
| 出站 | interaction.contact_head_top.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Head Top |
| 出站 | interaction.contact_head_top.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Head Top |
| 出站 | interaction.contact_leg.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Leg |
| 出站 | interaction.contact_leg.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Leg |
| 出站 | interaction.contact_leg.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Leg |
| 出站 | interaction.contact_nose.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Nose |
| 出站 | interaction.contact_nose.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Nose |
| 出站 | interaction.contact_nose.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Nose |
| 出站 | interaction.contact_thigh.stroke | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Thigh |
| 出站 | interaction.contact_thigh.stroke.cancel | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Thigh |
| 出站 | interaction.contact_thigh.touch | activity_mode: String；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Contact Thigh |
| 出站 | interaction.halo_rotate.phase_2 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Halo Rotate |
| 出站 | interaction.halo_rotate.phase_3 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Halo Rotate |
| 出站 | interaction.halo_rotate.phase_4 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Halo Rotate |
| 出站 | interaction.hand_drag.phase_2 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Hand Drag |
| 出站 | interaction.hand_drag.phase_3 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Hand Drag |
| 出站 | interaction.hand_drag.phase_4 | activity_mode: String；duration_ms: Integer；energy: Float；energy_state: String；feeling_adaptation: Float；feeling_available: Boolean；feeling_comfort: Float；feeling_load: Float；interaction_serial: Integer | Hand Drag |

## 全部蓝图索引

下表保留场景菜单的分组和顺序。正文按职责重新编排；同构小图在所属章节共用模板并逐项说明差异。当前停用的图照样列出，不能按图内旧注释理解为启用。

| 分组 | 蓝图 | 当前启用 | 节点 / 数据线 / 流程线 |
| --- | --- | --- | --- |
| 动作捕捉 | [面部追踪 - VMC](#graph-面部追踪---vmc) | 是 | 37 / 31 / 11 |
| 动作捕捉 | [姿态追踪 - VMC](#graph-姿态追踪---vmc) | 是 | 14 / 19 / 4 |
| Global | [Constants](#graph-constants) | 是 | 1 / 0 / 0 |
| Global | [Variables](#graph-variables) | 是 | 41 / 21 / 27 |
| Affect | [Emotion Controller](#graph-emotion-controller) | 是 | 25 / 14 / 15 |
| Expression | [Expression Controller](#graph-expression-controller) | 是 | 149 / 76 / 101 |
| Performance | [Performance Controller](#graph-performance-controller) | 是 | 344 / 251 / 168 |
| Expression | [Expression Mapping](#graph-expression-mapping) | 是 | 27 / 24 / 10 |
| Idle | [Idle Animation](#graph-idle-animation) | 是 | 84 / 64 / 48 |
| Expression | [Expression Request Controller](#graph-expression-request-controller) | 是 | 68 / 39 / 42 |
| Performance | [Presentation Lifecycle](#graph-presentation-lifecycle) | 是 | 87 / 56 / 58 |
| Expression | [Expression Aftereffect Controller](#graph-expression-aftereffect-controller) | 是 | 64 / 34 / 43 |
| Speak | [Speak Controller](#graph-speak-controller) | 是 | 153 / 98 / 91 |
| Performance | [Locomotion Animation](#graph-locomotion-animation) | 是 | 72 / 71 / 29 |
| Speak | [Speak 02（解释型说明）](#graph-speak-02解释型说明) | 是 | 29 / 20 / 11 |
| Expression | [Expression Combination Override - 1](#graph-expression-combination-override---1) | 否 | 23 / 25 / 2 |
| Character | [Gaze Controller](#graph-gaze-controller) | 是 | 92 / 73 / 47 |
| Performance | [Touch Head Top Reaction](#graph-touch-head-top-reaction) | 是 | 12 / 10 / 5 |
| Performance | [Touch Forehead Reaction](#graph-touch-forehead-reaction) | 是 | 12 / 10 / 5 |
| Speak | [Speak 03（激动地说）](#graph-speak-03激动地说) | 是 | 29 / 20 / 11 |
| Character | [Gaze Cursor Glance](#graph-gaze-cursor-glance) | 是 | 54 / 47 / 16 |
| Expression | [Expression Combination Override - 2](#graph-expression-combination-override---2) | 否 | 23 / 25 / 2 |
| Cursor | [Cursor Track Switch](#graph-cursor-track-switch) | 是 | 5 / 1 / 2 |
| Character | [Gaze Pointer Gesture](#graph-gaze-pointer-gesture) | 是 | 98 / 58 / 54 |
| Speak | [Speak 04（冷静分析）](#graph-speak-04冷静分析) | 是 | 29 / 20 / 11 |
| Expression | [Surprised](#graph-surprised) | 是 | 9 / 4 / 3 |
| Performance | [Touch Nose Reaction](#graph-touch-nose-reaction) | 是 | 12 / 10 / 5 |
| Cursor | [Cursor Movement](#graph-cursor-movement) | 是 | 7 / 5 / 2 |
| Interaction | [Interaction Controller](#graph-interaction-controller) | 是 | 128 / 99 / 62 |
| Expression | [Confused](#graph-confused) | 是 | 9 / 4 / 3 |
| Speak | [Speak 05（手指镜头）](#graph-speak-05手指镜头) | 是 | 29 / 20 / 11 |
| Interaction | [Interaction Context](#graph-interaction-context) | 是 | 27 / 19 / 7 |
| Cursor | [Mouse Pos to Cursor Pos](#graph-mouse-pos-to-cursor-pos) | 是 | 22 / 21 / 2 |
| Autonomy | [Autonomy Controller](#graph-autonomy-controller) | 是 | 52 / 39 / 13 |
| Performance | [Touch Breast Reaction](#graph-touch-breast-reaction) | 是 | 12 / 10 / 5 |
| Character | [Gaze Motion](#graph-gaze-motion) | 是 | 32 / 22 / 9 |
| Character | [Gaze Aim Projection](#graph-gaze-aim-projection) | 是 | 30 / 28 / 7 |
| Speak | [Speak 06（手指镜头，轻佻）](#graph-speak-06手指镜头轻佻) | 是 | 29 / 20 / 11 |
| Autonomy | [Autonomy State Tools](#graph-autonomy-state-tools) | 是 | 59 / 42 / 25 |
| Debug | [Touch Feelings](#graph-touch-feelings) | 否 | 27 / 14 / 15 |
| Expression | [Speechless](#graph-speechless) | 是 | 9 / 4 / 3 |
| Cursor | [Mouse Click Ray Probe](#graph-mouse-click-ray-probe) | 是 | 14 / 5 / 7 |
| Interaction | [Interaction Expression Bridge](#graph-interaction-expression-bridge) | 是 | 15 / 5 / 9 |
| Performance | [Touch Belly Reaction](#graph-touch-belly-reaction) | 是 | 12 / 10 / 5 |
| Expression | [Embarrassed](#graph-embarrassed) | 是 | 9 / 4 / 3 |
| Cursor | [Mouse Drag State](#graph-mouse-drag-state) | 是 | 22 / 10 / 9 |
| Interaction | [Back Drag](#graph-back-drag) | 是 | 69 / 69 / 36 |
| Performance | [Touch Thigh Reaction](#graph-touch-thigh-reaction) | 是 | 12 / 10 / 5 |
| Speak | [Speak 07（手部动作丰富地）](#graph-speak-07手部动作丰富地) | 是 | 29 / 20 / 11 |
| Autonomy | [Return To Standby](#graph-return-to-standby) | 是 | 75 / 41 / 45 |
| Character | [Attention](#graph-attention) | 是 | 34 / 22 / 17 |
| Interaction | [Hand Drag](#graph-hand-drag) | 是 | 83 / 80 / 53 |
| Performance | [Touch Leg Reaction](#graph-touch-leg-reaction) | 是 | 12 / 10 / 5 |
| Cursor | [Drag Target State](#graph-drag-target-state) | 是 | 8 / 4 / 4 |
| Speak | [Speak 08（双手交叉）](#graph-speak-08双手交叉) | 是 | 29 / 20 / 11 |
| Expression | [Expectant](#graph-expectant) | 是 | 9 / 4 / 3 |
| Autonomy | [Move To](#graph-move-to) | 是 | 79 / 47 / 50 |
| Character | [Energy and Rest](#graph-energy-and-rest) | 是 | 29 / 12 / 17 |
| Interaction | [Halo Rotate](#graph-halo-rotate) | 是 | 68 / 70 / 32 |
| Performance | [Touch Butt Reaction](#graph-touch-butt-reaction) | 是 | 12 / 10 / 5 |
| Speak | [Speak 09（双手交叉，冷静）](#graph-speak-09双手交叉冷静) | 是 | 29 / 20 / 11 |
| Expression | [Smug](#graph-smug) | 是 | 9 / 4 / 3 |
| Autonomy | [Move By](#graph-move-by) | 是 | 81 / 49 / 50 |
| Character | [Character Transform](#graph-character-transform) | 是 | 6 / 2 / 2 |
| Autonomy | [Rotate To](#graph-rotate-to) | 是 | 56 / 29 / 36 |
| Expression | [Skeptical](#graph-skeptical) | 是 | 9 / 4 / 3 |
| Performance | [Back Drag Swing](#graph-back-drag-swing) | 是 | 148 / 148 / 41 |
| Speak | [Speak 10（害羞地说）](#graph-speak-10害羞地说) | 是 | 29 / 20 / 11 |
| Interaction | [Contact Head Top](#graph-contact-head-top) | 是 | 54 / 49 / 45 |
| Expression | [Displeased](#graph-displeased) | 是 | 9 / 4 / 3 |
| Autonomy | [Rotate By](#graph-rotate-by) | 是 | 56 / 29 / 36 |
| Performance | [Thinking Pose](#graph-thinking-pose) | 是 | 36 / 25 / 24 |
| Interaction | [Contact Forehead](#graph-contact-forehead) | 是 | 54 / 49 / 45 |
| Expression | [Disgusted](#graph-disgusted) | 是 | 9 / 4 / 3 |
| Interaction | [Contact Nose](#graph-contact-nose) | 是 | 54 / 49 / 45 |
| Performance | [Hands on Hips](#graph-hands-on-hips) | 是 | 16 / 10 / 7 |
| Expression | [Worried](#graph-worried) | 是 | 9 / 4 / 3 |
| Interaction | [Contact Breast](#graph-contact-breast) | 是 | 54 / 49 / 45 |
| Performance | [Invite Hug](#graph-invite-hug) | 是 | 18 / 13 / 7 |
| Interaction | [Contact Belly](#graph-contact-belly) | 是 | 54 / 49 / 45 |
| Performance | [Shrug](#graph-shrug) | 是 | 18 / 13 / 7 |
| Expression | [Fearful](#graph-fearful) | 是 | 9 / 4 / 3 |
| Interaction | [Contact Thigh](#graph-contact-thigh) | 是 | 54 / 49 / 45 |
| Expression | [Aggrieved](#graph-aggrieved) | 是 | 9 / 4 / 3 |
| Performance | [Protest Pose](#graph-protest-pose) | 是 | 18 / 13 / 7 |
| Interaction | [Contact Leg](#graph-contact-leg) | 是 | 54 / 49 / 45 |
| Expression | [Shy](#graph-shy) | 是 | 9 / 4 / 3 |
| Performance | [Hand on Chest](#graph-hand-on-chest) | 是 | 18 / 13 / 7 |
| Expression | [Tender](#graph-tender) | 是 | 9 / 4 / 3 |
| Interaction | [Contact Butt](#graph-contact-butt) | 是 | 54 / 49 / 45 |
| Performance | [Concern Pose](#graph-concern-pose) | 是 | 18 / 13 / 7 |
| Expression | [Relieved](#graph-relieved) | 是 | 9 / 4 / 3 |
| Performance | [Cute Pose](#graph-cute-pose) | 是 | 24 / 20 / 10 |
| Expression | [Playful](#graph-playful) | 是 | 9 / 4 / 3 |
| Performance | [Refuse Pose](#graph-refuse-pose) | 是 | 24 / 20 / 10 |
| Expression | [Tired](#graph-tired) | 是 | 9 / 4 / 3 |
| Performance | [Flex Muscles](#graph-flex-muscles) | 是 | 18 / 13 / 7 |
| Performance | [Slouch](#graph-slouch) | 是 | 18 / 13 / 7 |
| Expression | [Bored](#graph-bored) | 是 | 9 / 4 / 3 |
| Expression | [Thinking](#graph-thinking) | 是 | 9 / 4 / 3 |
| Performance | [Facepalm](#graph-facepalm) | 是 | 18 / 13 / 7 |
| Performance | [Hand Behind Head](#graph-hand-behind-head) | 是 | 18 / 13 / 7 |
| Performance | [Reserved Pose](#graph-reserved-pose) | 是 | 18 / 13 / 7 |
| Performance | [Wave](#graph-wave) | 是 | 40 / 39 / 16 |
| Performance | [Nod](#graph-nod) | 是 | 18 / 11 / 10 |
| Performance | [Shake Head](#graph-shake-head) | 是 | 11 / 4 / 6 |
| Performance | [Surprise](#graph-surprise) | 是 | 11 / 4 / 6 |
| Performance | [Swing Body](#graph-swing-body) | 是 | 11 / 4 / 6 |
| Performance | [Laugh](#graph-laugh) | 是 | 69 / 58 / 35 |
| Performance | [Show Back](#graph-show-back) | 是 | 18 / 13 / 7 |
| Performance | [Ojou Laugh](#graph-ojou-laugh) | 是 | 136 / 96 / 73 |
| Performance | [Sustained Overlay](#graph-sustained-overlay) | 是 | 56 / 24 / 34 |

### 场景函数索引

这是场景内函数，不是要写进网络协议的接口。表内调用方来自当前显式函数节点；读写共享状态和原生流程求值另受各Controller规则约束。

| 函数名 | 定义图 | 签名 | 显式调用方 |
| --- | --- | --- | --- |
| WarudoDC.AdditiveGesture.surprise | Surprise | 无参数 → 仅流程通知 | Surprise |
| WarudoDC.AdditiveGesture.swing_body | Swing Body | 无参数 → 仅流程通知 | Swing Body |
| WarudoDC.Animation.ApplyModeIdle | Idle Animation | Variant: int → 仅流程通知 | Idle Animation |
| WarudoDC.Animation.IsOwner | Performance Controller | Source: int → bool | Idle Animation、Speak Controller |
| WarudoDC.Animation.Locomotion.Refresh | Locomotion Animation | Key: int → 仅流程通知 | Locomotion Animation |
| WarudoDC.Animation.Release | Performance Controller | Source: int → 仅流程通知 | Locomotion Animation |
| WarudoDC.Animation.ResetPresentation | Performance Controller | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Animation.ReturnIdle | Idle Animation | 无参数 → 仅流程通知 | Idle Animation、Speak Controller |
| WarudoDC.Animation.TryAcquire | Performance Controller | Source: int，RequiredInteraction: int，RequireIdle: bool → bool | Locomotion Animation、Performance Controller |
| WarudoDC.Animation.TryReturnIdle | Idle Animation | Variant: int → bool | Idle Animation、Presentation Lifecycle |
| WarudoDC.Autonomy.CanSubmit | Autonomy Controller | 无参数 → bool | Autonomy Controller、Move By、Move To、Rotate By、Rotate To、Variables |
| WarudoDC.Autonomy.ResolveTarget | Autonomy Controller | 无参数 → bool | Move By、Move To |
| WarudoDC.Autonomy.StateResult | Autonomy State Tools | EventId: string，Status: string，Detail: string → 仅流程通知 | Autonomy State Tools、Idle Animation、Return To Standby、Thinking Pose |
| WarudoDC.BackDrag.Presentation.Start | Back Drag Swing | 无参数 → 仅流程通知 | Back Drag |
| WarudoDC.BackDrag.Presentation.Stop | Back Drag Swing | 无参数 → 仅流程通知 | Back Drag |
| WarudoDC.BackDrag.Swing.Apply | Back Drag Swing | 无参数 → 仅流程通知 | Back Drag Swing |
| WarudoDC.Context.ActivityMode | Interaction Context | 无参数 → string | Back Drag、Contact Belly、Contact Breast、Contact Butt、Contact Forehead、Contact Head Top、Contact Leg、Contact Nose、Contact Thigh、Halo Rotate、Hand Drag |
| WarudoDC.Context.Energy | Interaction Context | 无参数 → float | Back Drag、Contact Belly、Contact Breast、Contact Butt、Contact Forehead、Contact Head Top、Contact Leg、Contact Nose、Contact Thigh、Halo Rotate、Hand Drag |
| WarudoDC.Context.EnergyState | Interaction Context | 无参数 → string | Back Drag、Contact Belly、Contact Breast、Contact Butt、Contact Forehead、Contact Head Top、Contact Leg、Contact Nose、Contact Thigh、Halo Rotate、Hand Drag |
| WarudoDC.Emotion.ResetPresentation | Emotion Controller | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Energy.LocomotionChanged | Energy and Rest | Active: bool → 仅流程通知 | Energy and Rest |
| WarudoDC.Expression.Compose.Apply | Expression Controller | 无参数 → 仅流程通知 | Expression Controller |
| WarudoDC.Expression.ContactAftereffect.Release | Expression Aftereffect Controller | Source: int → 仅流程通知 | Touch Belly Reaction、Touch Breast Reaction、Touch Butt Reaction、Touch Forehead Reaction、Touch Head Top Reaction、Touch Leg Reaction、Touch Nose Reaction、Touch Thigh Reaction |
| WarudoDC.Expression.ContactAftereffect.Submit | Expression Aftereffect Controller | Source: int，Expression: int，DurationSeconds: float → 仅流程通知 | Touch Belly Reaction、Touch Breast Reaction、Touch Butt Reaction、Touch Forehead Reaction、Touch Head Top Reaction、Touch Leg Reaction、Touch Nose Reaction、Touch Thigh Reaction |
| WarudoDC.Expression.Interaction.Apply | Interaction Expression Bridge | Source: int，Expression: int → 仅流程通知 | Interaction Controller |
| WarudoDC.Expression.Interaction.Clear | Interaction Controller | 无参数 → 仅流程通知 | Interaction Controller |
| WarudoDC.Expression.Interaction.ClearApplied | Interaction Expression Bridge | 无参数 → 仅流程通知 | Interaction Controller |
| WarudoDC.Expression.Interaction.ClearSource | Interaction Expression Bridge | 无参数 → 仅流程通知 | Interaction Controller |
| WarudoDC.Expression.Interaction.Submit | Interaction Controller | Source: int，Expression: int → 仅流程通知 | Touch Belly Reaction、Touch Breast Reaction、Touch Butt Reaction、Touch Forehead Reaction、Touch Head Top Reaction、Touch Leg Reaction、Touch Nose Reaction、Touch Thigh Reaction |
| WarudoDC.Expression.Mapping.Resolve | Expression Mapping | Layer: int，Expression: int → string | Expression Controller |
| WarudoDC.Expression.Override.Evaluate | Expression Combination Override - 2 | Base: int，Requested: int，Interaction: int → 仅流程通知 | Expression Controller |
| WarudoDC.Expression.Override.Propose | Expression Controller | RuleId: int，Priority: int，ExpressionName: string → 仅流程通知 | Expression Combination Override - 1、Expression Combination Override - 2 |
| WarudoDC.Expression.Presentation.Switch | Expression Controller | Next: string，Neutral: string → 仅流程通知 | Expression Controller |
| WarudoDC.Expression.Release | Expression Request Controller | SourceId: int → 仅流程通知 | Aggrieved、Bored、Confused、Disgusted、Displeased、Embarrassed、Expectant、Fearful、Laugh、Ojou Laugh、Playful、Relieved、Shy、Skeptical、Smug、Speechless、Surprised、Tender、Thinking、Thinking Pose、Tired、Worried |
| WarudoDC.Expression.ResetAftereffect | Expression Aftereffect Controller | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Expression.ResetPresentation | Expression Controller | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Expression.ResetRequested | Expression Request Controller | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Expression.Submit | Expression Request Controller | EmotionVariants: int[]，DurationSeconds: float，SourceId: int → 仅流程通知 | Aggrieved、Bored、Confused、Disgusted、Displeased、Embarrassed、Expectant、Fearful、Laugh、Ojou Laugh、Playful、Relieved、Shy、Skeptical、Smug、Speechless、Surprised、Tender、Thinking、Thinking Pose、Tired、Worried |
| WarudoDC.Gaze.Click.Clock | Gaze Pointer Gesture | PositionX: float，Seconds: float → 仅流程通知 | Gaze Pointer Gesture |
| WarudoDC.Gaze.Click.Commit | Gaze Pointer Gesture | 无参数 → 仅流程通知 | Gaze Pointer Gesture |
| WarudoDC.Gaze.Click.Reset | Gaze Pointer Gesture | 无参数 → 仅流程通知 | Gaze Controller、Gaze Pointer Gesture |
| WarudoDC.Gaze.Click.Tick | Gaze Pointer Gesture | 无参数 → 仅流程通知 | Gaze Controller |
| WarudoDC.Gaze.CursorActivity | Gaze Controller | 无参数 → 仅流程通知 | Attention |
| WarudoDC.Gaze.Glance.Initialize | Gaze Cursor Glance | 无参数 → 仅流程通知 | Gaze Controller、Gaze Cursor Glance |
| WarudoDC.Gaze.Glance.Reset | Gaze Cursor Glance | 无参数 → 仅流程通知 | Gaze Controller、Gaze Pointer Gesture |
| WarudoDC.Gaze.Glance.Tick | Gaze Cursor Glance | 无参数 → 仅流程通知 | Gaze Controller |
| WarudoDC.Gaze.Glance.TryStart | Gaze Cursor Glance | 无参数 → 仅流程通知 | Gaze Controller |
| WarudoDC.Gaze.Motion.Project | Gaze Aim Projection | 无参数 → 仅流程通知 | Gaze Motion |
| WarudoDC.Gaze.Motion.Render | Gaze Motion | 无参数 → 仅流程通知 | 由公共入口/其他数据引用消费 |
| WarudoDC.Gaze.Motion.Reset | Gaze Motion | 无参数 → 仅流程通知 | 面部追踪 - VMC |
| WarudoDC.Gaze.Pointer.Press | Gaze Pointer Gesture | 无参数 → 仅流程通知 | Gaze Controller |
| WarudoDC.Gaze.Pointer.Publish | Gaze Pointer Gesture | 无参数 → 仅流程通知 | Gaze Pointer Gesture |
| WarudoDC.Gaze.Pointer.Release | Gaze Pointer Gesture | 无参数 → 仅流程通知 | Gaze Controller |
| WarudoDC.Gaze.Render | Gaze Controller | 无参数 → 仅流程通知 | Gaze Controller |
| WarudoDC.Gaze.Tick | Gaze Controller | 无参数 → 仅流程通知 | Gaze Controller |
| WarudoDC.HeadGesture.Nod | Nod | Count: int → 仅流程通知 | Nod |
| WarudoDC.HeadGesture.shake_head | Shake Head | 无参数 → 仅流程通知 | Shake Head |
| WarudoDC.Interaction.CanUpdate | Interaction Controller | Source: int，FirstTarget: int，SecondTarget: int → bool | Back Drag、Halo Rotate、Hand Drag |
| WarudoDC.Interaction.End | Interaction Controller | Source: int → 仅流程通知 | Back Drag、Contact Belly、Contact Breast、Contact Butt、Contact Forehead、Contact Head Top、Contact Leg、Contact Nose、Contact Thigh、Halo Rotate、Hand Drag、Interaction Controller |
| WarudoDC.Interaction.IsOwner | Interaction Controller | Source: int → bool | Back Drag、Contact Belly、Contact Breast、Contact Butt、Contact Forehead、Contact Head Top、Contact Leg、Contact Nose、Contact Thigh、Halo Rotate、Hand Drag |
| WarudoDC.Interaction.OwnershipCommitted | Energy and Rest | 无参数 → 仅流程通知 | Interaction Controller |
| WarudoDC.Interaction.Phase.Notify | Interaction Controller | 无参数 → 仅流程通知 | Interaction Controller |
| WarudoDC.Interaction.PhaseChanged | Contact Butt | Source: int，Phase: int，Serial: int，DurationMs: int → 仅流程通知 | Interaction Controller |
| WarudoDC.Interaction.TryBegin | Interaction Controller | Source: int，GazeMode: int，PhasePoints: int[] → bool | Back Drag、Contact Belly、Contact Breast、Contact Butt、Contact Forehead、Contact Head Top、Contact Leg、Contact Nose、Contact Thigh、Halo Rotate、Hand Drag |
| WarudoDC.Laugh.Begin | Laugh | Variant: int → 仅流程通知 | Laugh |
| WarudoDC.Mode.InitializeStandby | Variables | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Mode.Request | Variables | Mode: int → bool | Autonomy State Tools |
| WarudoDC.Mode.ResetPresentation | Autonomy State Tools | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Mode.SettleStandby | Autonomy State Tools | EventId: string → 仅流程通知 | Autonomy State Tools |
| WarudoDC.Mode.StandbyCompleted | Autonomy State Tools | 无参数 → 仅流程通知 | Return To Standby |
| WarudoDC.OjouLaugh.Begin | Ojou Laugh | LaughOnce: bool → bool | Ojou Laugh |
| WarudoDC.Performance.Contact.Signal | Touch Butt Reaction | Source: int，Signal: int → 仅流程通知 | Contact Belly、Contact Breast、Contact Butt、Contact Forehead、Contact Head Top、Contact Leg、Contact Nose、Contact Thigh |
| WarudoDC.Performance.Ended | Ojou Laugh | Source: int，Reason: int → 仅流程通知 | Performance Controller |
| WarudoDC.Performance.OverlayEnded | Performance Controller | Gesture: int → 仅流程通知 | Laugh、Nod、Ojou Laugh、Shake Head、Surprise、Swing Body |
| WarudoDC.Performance.ResetPresentation | Performance Controller | 无参数 → 仅流程通知 | Presentation Lifecycle |
| WarudoDC.Performance.SetAnimationPolicy | Performance Controller | Source: int，DisableBreathing: bool，DisableSwaying: bool → 仅流程通知 | Back Drag Swing、Concern Pose、Cute Pose、Facepalm、Flex Muscles、Hand Behind Head、Hand on Chest、Hands on Hips、Invite Hug、Ojou Laugh、Protest Pose、Refuse Pose、Reserved Pose、Show Back、Shrug、Slouch、Thinking Pose、Wave |
| WarudoDC.Performance.Stop | Performance Controller | Source: int → 仅流程通知 | Back Drag Swing、Concern Pose、Cute Pose、Facepalm、Flex Muscles、Hand Behind Head、Hand on Chest、Hands on Hips、Idle Animation、Invite Hug、Ojou Laugh、Protest Pose、Refuse Pose、Reserved Pose、Show Back、Shrug、Slouch、Speak Controller、Thinking Pose、Wave |
| WarudoDC.Performance.StopSustainedOverlay | Performance Controller | Immediate: bool → 仅流程通知 | Performance Controller、Sustained Overlay |
| WarudoDC.Performance.SustainedOverlay.Apply | Sustained Overlay | Kind: int，Immediate: bool → bool | Performance Controller |
| WarudoDC.Performance.Terminate | Performance Controller | Reason: int → 仅流程通知 | Performance Controller |
| WarudoDC.Performance.TryFinishCurrent | Speak Controller | 无参数 → bool | Idle Animation、Speak Controller |
| WarudoDC.Performance.TryStart | Performance Controller | Source: int，RequiredInteraction: int，AutoEnd: bool，DurationSeconds: float → bool | Back Drag Swing、Concern Pose、Cute Pose、Facepalm、Flex Muscles、Hand Behind Head、Hand on Chest、Hands on Hips、Invite Hug、Ojou Laugh、Protest Pose、Refuse Pose、Reserved Pose、Show Back、Shrug、Slouch、Speak Controller、Thinking Pose、Wave |
| WarudoDC.Performance.TryStartOverlay | Performance Controller | Gesture: int → bool | Laugh、Nod、Ojou Laugh、Shake Head、Surprise、Swing Body |
| WarudoDC.Performance.TryStartSustainedOverlay | Performance Controller | Kind: int → bool | Sustained Overlay |
| WarudoDC.Presentation.ProcessStartupEyes | Presentation Lifecycle | BlendShapes: System.Collections.Generic.Dictionary&lt;string, float&gt; → System.Collections.Generic.Dictionary&lt;string, float&gt; | 面部追踪 - VMC |
| WarudoDC.Speak.BeginStage | Speak Controller | Seconds: float → bool | Speak Controller |
| WarudoDC.Speak.Clear | Speak Controller | 无参数 → 仅流程通知 | Speak Controller |
| WarudoDC.Speak.Interrupt | Speak Controller | 无参数 → 仅流程通知 | Speak Controller |
| WarudoDC.Speak.Play | Speak 10（害羞地说） | Style: int，Phase: int → 仅流程通知 | Speak Controller |
| WarudoDC.Speak.Request | Speak Controller | Style: int，PlanId: string，StartSeconds: float，EndSeconds: float → bool | Speak 02（解释型说明）、Speak 03（激动地说）、Speak 04（冷静分析）、Speak 05（手指镜头）、Speak 06（手指镜头，轻佻）、Speak 07（手部动作丰富地）、Speak 08（双手交叉）、Speak 09（双手交叉，冷静）、Speak 10（害羞地说） |
| WarudoDC.Speak.StopStyle | Speak Controller | Style: int → 仅流程通知 | Speak 02（解释型说明）、Speak 03（激动地说）、Speak 04（冷静分析）、Speak 05（手指镜头）、Speak 06（手指镜头，轻佻）、Speak 07（手部动作丰富地）、Speak 08（双手交叉）、Speak 09（双手交叉，冷静）、Speak 10（害羞地说） |
| WarudoDC.Speak.StyleReady | Speak 10（害羞地说） | Style: int → bool | Speak Controller |
| WarudoDC.SustainedOverlay.clear | Sustained Overlay | 无参数 → 仅流程通知 | Sustained Overlay |
| WarudoDC.SustainedOverlay.look_away | Sustained Overlay | 无参数 → 仅流程通知 | Sustained Overlay |
| WarudoDC.SustainedOverlay.tilt_neck | Sustained Overlay | 无参数 → 仅流程通知 | Sustained Overlay |
| WarudoDC.SustainedOverlay.tremble | Sustained Overlay | 无参数 → 仅流程通知 | Sustained Overlay |
| WarudoDC.Thinking.IsActive | Autonomy State Tools | 无参数 → bool | 由公共入口/其他数据引用消费 |
| WarudoDC.Thinking.Start | Thinking Pose | Variant: int → bool | Thinking Pose |
| WarudoDC.Wave.Begin | Wave | Variant: int，Count: int → 仅流程通知 | Wave |

## 如何修改和扩展这份场景

### 改现有素材或表现

只换姿势素材，先到对应Performance／Idle／Speak图修改原生动画节点，不动Control字段和来源ID。保持Start、Loop、End的生命周期；同一个素材的长度不一定等于一个语义动作周期，挥手尤其如此。

只改脸，先修改角色原生表情键中相应层的BlendShape和约束，再观察组合效果。不能用“调整优先级”笼统替代每个BlendShape的冲突约束，也不要为改变表情外观重写Controller路由。

过渡不自然时分别检查原生播放方式、TransitionTime、实际状态切换、重复开始、跟踪输入与素材自身，不先加多一套时钟或全局禁用追踪。算对时长或节点数更少，不能证明观感更自然。

### 新增一个接触部位

1. 在角色模型上确定接触骨骼、位置、形状和半径；以邻近Contact图为模板建立输入，但不要复用另一部位的常量ID。
2. 在Constants中加入稳定交互标识，调用Controller的参数化许可／阶段函数；本地图不另建一套占用锁。
3. 明确短触、持续开始、长时间特殊与取消何时发生，给阶段阈值Integer数组；方式进入模型要先于事件读数。
4. 在交互感受资源添加该部位及短／长方式偏好，接入输入和只读数据。
5. 给短触后效和持续INTERACTION分别选择表情；需要身体反应时由独立Reaction图向Performance申请，不在Contact中绕过身体权威。
6. 两侧注册同名同字段API，添加钩子与取消绑定；确认对应提示词真的在描述该部位。

### 新增一个表情请求

按现有REQUESTED模板新增原生键、Constants标识、API与小图；申请进入Expression Request Controller，传入时长和来源。0表示持续，正数表示期限。再在Adapter剧本/首段预设选择开放方式，补外观说明与搭配建议。

不能只新增一个原生键就假定Director可用了，也不能为省小图让网络直接写`InteractionExpression`。若使用组合覆写，规则通过提案函数申请；Controller不依赖具体规则图。当前两个示例默认停用，先按自己角色验证再启用。

### 新增普通肢体、短附加或持续附加

先判断它属于哪个执行槽：持续站姿／多段Speak使用身体权威，原生一次性附加使用短附加入口，歪头等持久附加使用持续槽。分类依据是生命周期与能否组合，不只是素材名字或是否勾了Additive。

复制相同类别图，给唯一来源，先经过Controller许可，再播放；结束只释放自身仍持有的状态。需要计时就保存本轮Timer ID，旧到期不能结束新的拥有者。将API加入两端目录，再让导演注册选择；是否绑脸按语义决定，不能为每个动作都盲目加表情参数。

### 新增一个有结果的LLM工具

先实现业务入口与结果事件：接收原事件，冻结参数和原事件ID，提交本地操作，等匹配的终态，发新事件并关联原ID。再在Adapter配置工具名、用途、输入与结果API、等待时间和字段。

不要把“对端受理”当操作完成；被拒绝、被替代、取消、超时和正常完成需要分别说明。纯表现若不需要模型等待，可以继续走导演/单向事件，不必为了形式统一给每个动作加完整回执。

### 修改共享函数与常量

函数参数必须遵循原生类型候选，`int/float/string/int[]`与`System.Int32`这类可解析CLR名字不是一回事。曾发生生成器和替身测试一起认错，真实加载保存后退为Object。修改后需核对实际端口、参数类型、返回值、所有调用者以及保存重载后的结果。

常量值是跨图协议：调整字典序不等于重新编号，不能因排序而改变既有数值。原生流程函数适合明确通知，数据函数适合按需读取；不能把数据查询误接成每帧执行的流程函数。

## 已知边界与当前文件中的说明差异

以下是文档静态核对发现的边界，不是本轮新跑实机后宣布的故障修复：

| 项目 | 当前事实 | 使用时怎样理解 |
| --- | --- | --- |
| Rotate To角度范围 | 工具说明0～360，但蓝图夹取-180～180 | >180不会原样传入，先使用实际范围；修订需同步双方，不在手册中偷偷换算 |
| 点击探针模式清理 | 第二模式Switch缺少实时ActMode连线，保存输入0 | 松开直接撤回仍有线；不能保证切Free时该分支按旧注释撤回。交互Controller的许可仍独立存在 |
| 组合覆写示例 | 两张图的蓝图启用状态都是关闭 | 图内开关/旧“启用”注释不能令停用图生效 |
| 回归Idle工具 | 工具注册存在但禁用，部分用途说明仍引用它 | API/导演入口可用不等于主LLM拿到了工具；按需显式启用或调整说明 |
| API注册表的ProfileDescription | 留有已退休样例名称等旧描述 | 以99项实际目录和消费图为准，不按描述恢复旧API |
| 背部摆动 | 有实际表现和多轮调优，仍有残留拖动抖动历史 | 不把诊断撤下或支点正确包装成彻底消除抖动 |
| 模型与精力 | 暂定角色模拟，当前参数可用 | 不对应真实力/疼痛/生理量，也不独立决定角色意愿 |
| 多发声器/多智能端 | 当前本场景只有audio.main；Link播放器仍共享 | 不能靠再复制资源就获得并行双角色聊天架构 |
| 口型 | 当前系统回录，可能吃进其他软件声音 | 尚不是Link专属音源；不使用私有接口或自制嘴型绕过 |
| 表情与动作搭配 | 大量常用组合可用，仍有不自然组合与导演节奏问题 | 功能兼容与自然度分开评价，需要具体动作序列再调 |

这些条目不自动成为本轮改场景任务。图中保存值、静态接线、历史用户反馈与当前实时状态是不同证据，不能互相替代。

## 保存、重载与维护检查

保存的场景是配置与结构快照，很多正在运行的中间值也可能被序列化；重载时初始化会重新建立合法状态，并不续播上次的动作或录音。Timer ID、运动命令ID、最近Control事件和上次播放结果不应被当永久业务配置。

文件变小可能是原生保存器省略了默认界面属性，文件变大可能是节点端口、函数参数元数据或注释增加。核对逻辑至少比较图/资源是否仍在、节点类型与端口、数据与流程连接、函数定义与调用、原生表情内容及关键参数；只看KB变化或节点总数不能说明丢没丢逻辑。

### 建议验证顺序

1. **离线结构**：112张图和39资源引用可解析，函数/端口/类型正确，两侧API一致；确认没有意外新增第二位移执行者。
2. **加载与初始化**：角色和素材可用，表情NEUTRAL、启动眼睑、待命姿势和交互放行顺序正确。
3. **本地输入**：八处短触、600ms后松开、2秒特殊、拖拽/牵手/光环；确认许可/阶段/取消和正常结束。
4. **表情与身体**：REQUESTED持续/限时、短触后效、普通姿势保持、移动和BackDrag保护、Speak/短附加/持续附加各自结束。
5. **视线**：一般/常追、按住跟随/松开停留、连续目标过渡、交互注视、回复时优先与眼神微动。
6. **智能端互通**：文字/语音、钩子及取消、运动工具回执、语调/动作随Timeline释放、候选与新旧回复隔离。
7. **改动相关生命周期**：保存重载、禁用重新启用、快速重入、场景切换；不要求每改一次文案就重新穷举全部设备组合。

### 安全地观察

先看Player.log的相关前缀和保存文件；读日志时保留文件共享并避免输出令牌。需要新的保存值，请由用户正常保存，不调用完整场景API来代替。

**不要调用`GET /api/openedScene`，也不要使用等效命令完整序列化运行中的主场景。** 过去这种“只读”观察曾伴随严重卡顿与崩溃，原因尚未唯一确认。出现卡顿/连接重置/崩溃就停用该观察路径，不自动重试或启动Warudo。

### 证据怎样保留

本文依据当前两份JSON、已保存场景的连接和变量、历次设计/实施记录，以及此前用户实测反馈整理。场景基础布置已形成可使用基线，但不代表所有终态交错、长时间运行、缺失资源和全动作组合都通过验收。

历史上否决的单次挥手串播、未稳定的动态肩部摆动、曾误判的函数类型与旧时钟方案都保留为教训，而不是当前安装步骤。更换模型/素材/Provider后需要重新看对应表现，不能用旧模型或替身测试通过代替新的视觉和听感。

当前这份手册覆盖全部112张蓝图及配套配置。它不把原始日志逐行重印：当前有效职责与接口进入正文，旧方案解释其被替代的原因；哈希、构建流水、故障原始文件和少见未验边界继续作为研发证据保留。维护时以新的实际保存结构更新手册，不反过来按旧文字恢复已退役逻辑。
