# LNV 活动模块接入实战笔记

## 1. 这篇笔记解决什么问题

这篇记录 LNV 活动系统的接入方式，重点是看懂：

- 活动为什么不是普通 UI。
- `Activity Feature` 如何自动构建活动。
- `Promotion -> Service -> Module -> UI` 的关系。
- 新活动怎么接入。
- 活动接口、配置、红点、刷新事件怎么串起来。
- 以 `P695` 老玩家登录奖励为例看完整链路。

核心代码位置：

```text
D:\LNV\trunk\client\Hotfix\Features\Activity.cs
D:\LNV\trunk\client\Hotfix\Features\Activities\Promotion.cs
D:\LNV\trunk\client\Hotfix\Features\Activities\PromotionAttribute.cs
D:\LNV\trunk\client\Hotfix\Features\Activities\Service.cs
D:\LNV\trunk\client\Hotfix\Features\Activities\Module.cs
D:\LNV\trunk\client\Hotfix\Features\Activities\Services\Fixtures\P695.cs
D:\LNV\trunk\client\Hotfix\Features\Activities\Modules\Fixtures\P695.cs
```

## 2. 活动系统整体结构

活动系统不是一个 UI Controller 直接接接口，而是分成：

```text
Activity Feature
活动总管理器。

Promotion
单个活动实例，绑定活动配置、Service、Module、Duty。

Service
负责请求后端接口、监听事件、更新 Module。

Module
负责保存活动状态、计算红点、整理 UI 数据。

Duty
负责活动时间状态，比如未开始、进行中、结束。

UI Controller
负责展示 Module 数据、调用 Service 方法。
```

整体链路：

```text
[PromotionAttribute]
-> Activity.BuildPromotions()
-> Promotion.Build()
-> Module.Build()
-> Service.Build()
-> Duty.Build()
-> Promotion.Init()
-> Activity.LoginReq()
-> Promotion.LoginQuery()
-> Service.DoFuncUnlock()
-> QueryData()
-> Module.SortData()
-> UI 展示
```

## 3. Activity Feature 做什么

入口：

```text
D:\LNV\trunk\client\Hotfix\Features\Activity.cs
```

`Activity` 是一个 Feature：

```csharp
[Feature(100)]
public class Activity : Core.Feature
```

它的 `Init()` 主要做：

```text
BuildPromotions()
SubscribeRedDot()
绑定功能开启事件
绑定跨天事件
绑定充值事件
绑定角色升级事件
绑定资源变化事件
注册服务端推送协议
```

它的 `LoginReq()` 会请求活动总信息：

```csharp
var response = await GameManager.GameMessageManager.CallRequest<OperatePromotionInfoResponse>(
    new OperatePromotionInfoRequest());
```

然后遍历所有 promotion：

```csharp
await promotion.LoginQuery();
```

所以活动数据是在登录后统一拉取的，不依赖你有没有打开活动 UI。

## 4. PromotionAttribute 是活动注册入口

活动 Service 上会写特性：

```csharp
[Promotion(EPromotion.ID695, typeof(Modules.Fixtures.P695), EObserverFlags.CHANGE_DAY)]
public class P695 : Service
```

这行很关键，它告诉系统：

```text
这个 Service 对应哪个活动 ID
这个活动用哪个 Module
这个活动监听哪些刷新事件
```

如果没有这个 Attribute，`Activity.BuildPromotions()` 扫描不到活动，这个活动就不会被创建。

常见参数：

```text
EPromotion.IDxxx
活动 ID，通常要和 residentactivities 配置对应。

typeof(Module)
活动数据模块类型。

EObserverFlags
监听事件，比如跨天、充值、角色升级、资源变化。

DutyType
活动时间类型，默认 Disposable，也可以是 Periodic 等。
```

## 5. Promotion 是单个活动实例

入口：

```text
D:\LNV\trunk\client\Hotfix\Features\Activities\Promotion.cs
```

`Promotion.Build()` 会做：

```text
根据 EPromotion 从 ConfigManager.DB.residentactivities 取配置
创建 Module
创建 Service
创建 Duty
保存 AliasIds
```

关键代码逻辑：

```csharp
var formula = ConfigManager.DB.residentactivities.GetByIdx((int)id);
promotion.Module = Module.Build(attr.ModuleType, promotion);
promotion.Service = Service.Build(serviceType, promotion);
promotion.Duty = Duty.Build(attr, promotion);
```

所以活动能不能显示，至少依赖：

```text
residentactivities 配置存在
EPromotion 枚举正确
PromotionAttribute 正确
Module 能创建
Service 能创建
Duty 时间状态正确
功能开启 OpenStatus 正确
```

## 6. Service 负责请求接口

入口：

```text
D:\LNV\trunk\client\Hotfix\Features\Activities\Service.cs
```

Service 的职责：

```text
请求后端数据
处理后端返回
更新 Module 字段
计算活动时间
触发 Module 刷新
触发 UI RefreshEvent
注册服务端推送协议
更新红点
```

Service 必须实现：

```csharp
protected abstract Task<bool> QueryData();
protected abstract void OnUpdateData();
```

常用刷新方法：

```csharp
UpdateModule();
UpdateModuleAndRefresh();
```

`UpdateModule()` 内部会：

```text
OnUpdateData()
-> owner.UpdateFormula()
-> module.SortData()
```

`UpdateModuleAndRefresh()` 额外会：

```text
owner.RefreshEvent?.Invoke()
```

## 7. Module 负责保存和计算状态

入口：

```text
D:\LNV\trunk\client\Hotfix\Features\Activities\Module.cs
```

Module 的职责：

```text
保存后端返回的活动数据
保存配置表整理后的数据
计算 Archived
计算 Finished
计算 IsShow
计算 HasRedDot
给 UI 提供展示数据
```

核心状态：

```text
HasRedDot
红点状态。

IsShow
活动是否显示。

Finished
活动是否完成。

Archived
是否存在已完成但未领取的任务。

EPromotionForShow
当前用于展示的活动 ID。
```

Module 需要实现：

```csharp
protected abstract void CalcFinished();
protected abstract void CalcArchived();
```

也就是说，活动是否完成、是否有可领取奖励，应该在 Module 里算，不应该散落在 UI 里。

## 8. P695 活动实战链路

Service 文件：

```text
D:\LNV\trunk\client\Hotfix\Features\Activities\Services\Fixtures\P695.cs
```

Module 文件：

```text
D:\LNV\trunk\client\Hotfix\Features\Activities\Modules\Fixtures\P695.cs
```

### 8.1 P695 注册

```csharp
[Promotion(EPromotion.ID695, typeof(Modules.Fixtures.P695), EObserverFlags.CHANGE_DAY)]
public class P695 : Service
```

表示：

```text
活动 ID 是 ID695
使用 Modules.Fixtures.P695 保存数据
跨天时刷新
```

### 8.2 P695 登录拉数据

`QueryData()`：

```csharp
var response = await GameManager.GameMessageManager.CallRequest<XinFuDengLuJiangLiResponse>(
    new XinFuDengLuJiangLiRequest());
```

返回后更新 Module：

```csharp
realModule.LoginRewards.Clear();
realModule.LoginRewards.AddRange(response.ReceiveIds);
realModule.LoginDays = response.Day;
owner.UpdateDutyTime(0, response.EndTime, response.EndTime);
```

含义：

```text
ReceiveIds
已经领取过的天数。

Day
当前累计登录天数。

EndTime
活动结束时间。
```

### 8.3 P695 领取奖励

`RequestReward(int day)`：

```csharp
if (realModule.GetState(day) != Module.State.Archive)
{
    return;
}

var response = await GameManager.GameMessageManager.CallRequest<XinFuDengLuJiangLiReceiveResponse>(
    new XinFuDengLuJiangLiReceiveRequest());
```

领取成功后：

```text
更新已领取列表
发放资源变化
UpdateModuleAndRefresh()
```

### 8.4 P695 Module 状态

Module 保存：

```csharp
public List<int> LoginRewards = new List<int>();
public int LoginDays;
```

状态枚举：

```csharp
public enum State
{
    Archive,
    Lock,
    Finish,
}
```

`GetState(day)` 逻辑：

```text
day > LoginDays
-> Lock，未解锁

已经领取
-> Finish，已完成

未领取且已达到天数
-> Archive，可领取
```

奖励配置读取：

```csharp
var costs = ConfigManager.DB.laowanjiadenglu.GetByIdx(day);
return costs != null && costs.RewardsC.Count > 0 ? costs.RewardsC[0] : new Cost();
```

红点逻辑：

```text
从第 1 天到 LoginDays
只要有一天没领取
就显示红点
```

## 9. 新活动接入步骤

标准步骤：

```text
1. 确认活动需求和后端协议
2. 配置 residentactivities
3. 增加或确认 EPromotion 枚举
4. 准备活动专属配置表
5. 生成 Luban 配置代码和 bytes
6. 生成 PBStruct 协议代码
7. 写 Module
8. 写 Service
9. 在 Service 上加 PromotionAttribute
10. 写 UI Controller
11. UI 上绑定 PromotionViewAttribute
12. 验证登录后 Activity.LoginReq 拉数据
13. 验证活动入口、红点、领奖、跨天刷新
```

## 10. Module 写法模板

```csharp
public class Pxxx : Module
{
    public List<int> FinishedIds = new List<int>();
    public List<TaskData> Datas = new List<TaskData>();

    protected override void CalcFinished()
    {
        Finished.Value = Datas.Count > 0 && Datas.TrueForAll(x => x.Finished);
    }

    protected override void CalcArchived()
    {
        Archived = Datas.Exists(x => x.Archived);
    }

    public override void SortData()
    {
        Datas.Sort(SortATaskData);
        base.SortData();
    }
}
```

## 11. Service 写法模板

```csharp
[Promotion(EPromotion.IDxxx, typeof(Modules.Fixtures.Pxxx), EObserverFlags.CHANGE_DAY)]
public class Pxxx : Service
{
    private Modules.Fixtures.Pxxx realModule;

    public override void OnInit()
    {
        base.OnInit();
        realModule = module as Modules.Fixtures.Pxxx;
    }

    protected override async Task<bool> QueryData()
    {
        var response = await GameManager.GameMessageManager.CallRequest<XxxInfoResponse>(
            new XxxInfoRequest());

        if (response == null)
        {
            SetDisEnable();
            return false;
        }

        realModule.FinishedIds.Clear();
        realModule.FinishedIds.AddRange(response.FinishedIds);
        return true;
    }

    protected override void OnUpdateData()
    {
        // 把配置表 + 后端状态整理成 UI 需要的数据
    }
}
```

## 12. UI 接入方式

活动 UI 一般不直接自己决定活动是否开放，而是从 Activity Feature 拿 Promotion：

```csharp
var promotion = Features.Manager.Instance.Get<Features.Activity>().GetPromotion(EPromotion.IDxxx);
var service = promotion.Service as XxxService;
var module = promotion.Module as XxxModule;
```

UI 负责：

```text
OnOpen 时拿 Module 数据刷新
按钮点击时调用 Service.RequestXxx
监听 promotion.RefreshEvent 或重新刷新
不要自己保存核心活动状态
不要自己判断红点总逻辑
```

## 13. 活动不显示怎么查

按这个顺序查：

```text
1. residentactivities 是否配置了活动 ID
2. EPromotion 是否有这个 ID
3. Service 是否加了 PromotionAttribute
4. Module 类型是否正确
5. Activity.BuildPromotions 是否扫描到了
6. OpenStatus 功能是否开启
7. Duty 时间是否在活动期内
8. OperatePromotionInfoResponse 是否有这条活动
9. QueryData 是否返回 null
10. Module.IsShow 是否为 true
11. UI 是否绑定了 PromotionViewAttribute
```

## 14. 红点不对怎么查

红点链路：

```text
Service.QueryData()
-> 更新 Module 数据
-> UpdateModule()
-> Module.SortData()
-> CalcArchived()
-> CalcFinished()
-> CalcRedDot()
-> Service.OnInit 里订阅 HasRedDot
-> GRedPointManager.MarkRedPoint()
```

如果红点不显示：

```text
先查 Module.Archived 是否正确
再查 HasRedDot 是否变化
再查 MarkRedPoint 是否执行
最后查 UI 红点绑定
```

不要直接在 UI 上写一个红点兜底，那样数据根源还是错的。

## 15. 难点总结

活动系统难点是它同时依赖：

```text
配置表
活动总开关
功能开启
后端协议
活动时间
Module 状态
红点系统
UI 绑定
跨天/充值/资源变化事件
```

所以活动问题很少是单点问题。

正确排查方式是沿链路查：

```text
配置有没有
实例有没有创建
后端有没有返回
Module 有没有更新
状态有没有计算
UI 有没有刷新
```

## 16. 大白话总结

活动系统可以理解成：

```text
Activity 是活动总管
Promotion 是某一个具体活动
Service 是这个活动派出去和后端沟通的人
Module 是这个活动的本地数据本子
Duty 是这个活动的时间闹钟
UI 是最后把数据画出来的界面
```

你接一个新活动，不是只写一个界面。

你要把这条线接通：

```text
配置表有活动
-> 代码里有活动 ID
-> Service 用 PromotionAttribute 注册
-> Activity 扫描并创建 Promotion
-> 登录后 Service 请求后端
-> Module 保存和计算状态
-> UI 读取 Module 展示
```

如果活动没出来，不要只看 UI。先查活动有没有被创建，再查后端数据，再查 Module 状态，最后再查界面。
