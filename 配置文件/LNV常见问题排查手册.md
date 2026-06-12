# LNV 常见问题排查手册

## 1. 这篇笔记解决什么问题

这篇记录 LNV 项目里常见问题的根源排查方式，重点是：

- 不优先写兜底。
- 先找到链路断点。
- 从配置、接口、生命周期、UI、平台环境几个方向定位。
- 把你之前遇到的问题整理成可复用的排查手册。

核心原则：

```text
能从根源解决，就不要靠兜底盖住问题。
兜底只能保证不崩，但不能保证数据正确。
```

## 2. 问题一：配置表读不到

### 现象

```text
ConfigManager.DB 为空
ConfigManager.DB.xxx 找不到
GetByIdx 返回 null
DataList 为空
UI 显示空数据
运行时报 ByteBuf 解析错误
```

### 根源链路

```text
源表
-> Luban 生成代码
-> Luban 生成 bytes
-> bytes 打进 cfg 资源
-> ConfigManager.LoadText
-> DB.Manager
-> Tables.xxx
-> Beans.xxx
-> 业务读取
```

### 排查顺序

```text
1. 源配置是否有数据
2. 是否重新导表
3. AutoGen 是否更新
4. bytes 是否更新
5. bytes 是否在 Assets/DynamicRes/Tables/Luban
6. cfg 资源是否被打包
7. ConfigManager.Init 是否执行
8. ConfigManager.DB 是否成功创建
9. 表名和字段名是否一致
10. key 是否存在
```

### 根源处理

不要在 UI 里写：

```csharp
if (cfg == null) cfg = new DefaultCfg();
```

应该先修：

```text
导表
bytes
ConfigManager
DB.Manager
配置 key
```

### 特别注意

当前检查到：

```text
D:\LNV\trunk\client\Hotfix\Core\Config\ConfigManager.cs
```

文件内容结构看起来异常，`LoadText` 方法和类括号位置不完整。

如果项目出现配置初始化失败、编译异常、读表失败，优先检查这个文件是否被误改或合并坏了。

## 3. 问题二：AutoGen 代码改了又没了

### 现象

```text
在 AutoGen 里加的方法消失
字段被覆盖
GetByXxx 方法没了
重新导表后代码变回去了
```

### 根源

`AutoGen` 是 Luban 自动生成目录。

位置：

```text
D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban
```

里面的代码会被生成工具覆盖。

### 正确处理

不要手改：

```text
AutoGen\Luban\Manager.cs
AutoGen\Luban\Tables\xxx.cs
AutoGen\Luban\Beans\xxx.cs
```

人工扩展放这里：

```text
D:\LNV\trunk\client\Hotfix\Database\LubanMaual
```

用 `partial class` 扩展：

```csharp
namespace DB.Tables
{
    public partial class hero
    {
        public Beans.hero GetByType(int type)
        {
            return DataList.Find(x => x.Type == type);
        }
    }
}
```

## 4. 问题三：接口没有返回或按钮没反应

### 现象

```text
点击按钮没反应
接口 response 为 null
UI 没刷新
奖励没到账
红点不消失
```

### 游戏服接口链路

```text
UI 点击
-> Feature/Service 方法
-> CallRequest<Response>(Request)
-> ProtobufHelper.GetProtoId<Response>
-> Session.Send
-> 后端处理
-> MessageDispatch
-> ParseMessageFrom<Response>
-> 更新本地数据
-> UI 刷新
```

### 排查顺序

```text
1. 按钮事件是否绑定
2. 是否调用到 Feature/Service
3. Request 字段是否正确
4. Response 类型是否正确
5. ProtoId 是否注册
6. Session 是否连接
7. 后端是否收到请求
8. 后端是否返回 TipMessage
9. response 是否为空
10. 本地数据是否更新
11. UI 是否刷新
```

### 根源处理

如果 response 为空，不要直接 UI 提示成功。

应该确认：

```text
协议是否一致
后端是否报错
字段是否缺失
是否超时
是否返回 TipsMessage
```

## 5. 问题四：登录卡住

### 现象

```text
卡在登录界面
一直 loading
选服后进不去
创建角色失败
进入主界面前卡住
```

### 登录主链路

```text
VerifyPackage
-> PlatformManager.Login
-> VerifyToken
-> WebServerManager.QueryAll
-> GUIManager.Wait<Login>
-> Features.Login.DoLogin
-> GameMessageManager.Connect
-> LoginRequest
-> Features.Manager.LoginReq
-> BattleManager.AfterLoginReq
-> TranslateToState(Gaming)
```

### 排查顺序

```text
1. VerifyPackage 是否通过
2. 平台 SDK 登录是否成功
3. VerifyToken 是否返回 openId
4. QueryAll 是否返回 ServerInfo
5. server.s_u/server.s_p/ws_url 是否正确
6. GameMessageManager.Connect 是否成功
7. LoginResponse 是否返回
8. LoginResponse.Status 是多少
9. 是否进入创角流程
10. 哪个 Feature.LoginReq 卡住
```

### 关键文件

```text
D:\LNV\trunk\client\Hotfix\Core\LifeCycle\LifeCycleState\LifeCycleLogin.cs
D:\LNV\trunk\client\Hotfix\Features\Login.cs
D:\LNV\trunk\client\Hotfix\Core\WebService\Manager.cs
D:\LNV\trunk\client\Hotfix\Managers\Network\NetMessageManager.cs
```

## 6. 问题五：活动入口不显示

### 现象

```text
活动按钮没有
活动列表没有
活动 UI 打不开
活动显示未开启
```

### 活动链路

```text
residentactivities 配置
-> EPromotion
-> PromotionAttribute
-> Activity.BuildPromotions
-> Promotion.Build
-> Duty 时间
-> OpenStatus 功能开启
-> Activity.LoginReq
-> Service.QueryData
-> Module.IsShow
-> UI 活动入口
```

### 排查顺序

```text
1. residentactivities 是否有这条活动
2. EPromotion 是否有对应 ID
3. Service 是否写了 PromotionAttribute
4. Module 类型是否能创建
5. Activity.BuildPromotions 是否扫到
6. OperatePromotionInfoResponse 是否有活动数据
7. Duty 时间是否有效
8. OpenStatus 是否开启
9. QueryData 是否成功
10. Module.IsShow 是否为 true
11. UI 是否绑定了 PromotionViewAttribute
```

### 根源处理

不要直接在 UI 强制显示按钮。

如果活动没显示，必须先确认：

```text
配置存在
活动实例存在
时间有效
后端开启
Module 状态正确
```

## 7. 问题六：活动红点不对

### 现象

```text
可领取但没有红点
已领取但红点还在
重新打开 UI 红点状态变了
跨天后红点不刷新
```

### 红点链路

```text
Service.QueryData
-> 更新 Module 数据
-> UpdateModule
-> Module.SortData
-> CalcArchived
-> CalcFinished
-> CalcRedDot
-> module.HasRedDot
-> GRedPointManager.MarkRedPoint
-> UI 红点组件
```

### 排查顺序

```text
1. 后端返回的任务/领取状态是否正确
2. Module 保存的数据是否正确
3. CalcArchived 是否正确
4. CalcFinished 是否正确
5. HasRedDot 是否变化
6. MarkRedPoint 是否执行
7. UI 红点 key 是否一致
8. 跨天/充值/资源变化事件是否监听
```

### 根源处理

红点逻辑应该在 Module/Service 中统一计算。

不要在 UI 单独写：

```csharp
redDot.SetActive(true);
```

否则 UI 和真实业务状态会不一致。

## 8. 问题七：备案号怎么移除

### 现象

登录界面显示备案号或版号信息，需要隐藏或移除。

### 排查方向

优先查：

```text
D:\LNV\trunk\client\Hotfix\UI\Controllers\Login.cs
```

之前定位到备案号主要是登录 UI 里硬编码或 UI 绑定显示，不是单纯配置表兜底。

### 建议处理

如果这个包体确定不需要备案号：

```text
建议直接从 UI 逻辑或 prefab 绑定源头移除。
```

如果只是某些渠道不显示：

```text
应该按渠道配置或平台条件控制显示。
```

不建议只用透明度隐藏，因为：

```text
节点还在
可能挡点击
后续维护容易误判
自动化检查仍可能扫到文本
```

根源处理是确认显示来源，然后删掉或做渠道条件控制。

## 9. 问题八：MaskableParticle 报错

### 现象

UI 粒子或 MaskableParticle 相关对象报组件缺失、渲染异常。

### 关键文件

```text
D:\LNV\trunk\client\Hotfix\Utils\MaskableParticle.cs
```

### 根源

`MaskableParticle` 依赖：

```text
ParticleSystem
ParticleSystemRenderer
CanvasRenderer
```

如果对象缺少组件，就会出现运行时问题。

### 根源处理

使用组件依赖声明：

```csharp
[RequireComponent(typeof(ParticleSystem), typeof(ParticleSystemRenderer), typeof(CanvasRenderer))]
```

这样从源头保证挂这个脚本时需要的组件存在。

这比在运行时到处判空更稳。

## 10. 问题九：Tuanjie/小游戏扩展安装后怎么办

### 现象

安装了 minigame 扩展，不确定为什么要单独装，或者切平台后项目状态异常。

### 理解方式

小游戏扩展属于 Unity/Tuanjie 的平台构建能力，不一定默认包含在编辑器核心里。

单独安装通常是因为：

```text
小游戏平台构建工具链
平台 API
导出模板
构建面板
平台适配脚本
```

不一定会自动进入项目业务代码。

### 安装后要做什么

```text
1. 确认 Package/Extension 安装成功
2. 确认 Build Settings 里平台可选
3. 确认 Player Settings 平台参数
4. 确认项目依赖没有丢
5. 确认 Addressables/资源构建配置
6. 切到目标平台后重新编译
7. 处理平台宏导致的编译错误
8. 再构建小游戏包
```

### 不小心点了 Switch Platform

如果误点切平台：

```text
先不要乱删 Library
先确认当前目标平台
等 Unity/Tuanjie 编译完成
看 Console 编译错误
确认是否需要切回原平台
检查平台宏导致的代码分支
```

切平台本身会触发资源和脚本重新导入，时间长是正常的。

## 11. 问题十：UI 有数据但显示不刷新

### 现象

```text
接口返回了
Module 数据也变了
但是界面还是旧的
关闭再打开才正常
```

### 排查方向

```text
1. UI 是否监听了 RefreshEvent
2. Service 是否调用 UpdateModuleAndRefresh
3. UI 刷新函数是否真的执行
4. UI 是否使用了旧缓存列表
5. List/Scroll 是否重新 SetData
6. 异步回来时 UI 是否已经关闭
```

### 根源处理

接口成功后要形成闭环：

```text
后端返回
-> 更新 Module
-> 调 RefreshEvent
-> UI 重新读取 Module
-> 刷新节点
```

不要只更新 UI 局部文本，因为重新打开后还是会从 Module 取数据。

## 12. 问题十一：Feature 登录请求卡住

### 现象

```text
登录游戏服成功
但进入主界面前卡住
日志停在 Feature Start LoginReq
```

### 根源

`Features.Manager.Instance.LoginReq(token)` 会按 Feature Sort 分组执行。

如果某个 Feature 的 `LoginReq()` await 不返回，整体登录流程就会卡住。

### 排查方式

日志里看：

```text
Feature Start LoginReq xxx
Feature Finish LoginReq xxx
```

哪个只有 Start 没有 Finish，就查哪个 Feature。

### 根源处理

```text
检查接口是否返回
检查 await 是否异常吞掉
检查 token 是否取消
检查 response null 后是否继续等待
检查是否有 TaskCompletionSource 没 SetResult
```

## 13. 问题十二：协议类型或 ProtoId 对不上

### 现象

```text
GetProtoId error
CallRequest 一直 null
后端说返回了，前端收不到
收到 TipMessage
```

### 排查方向

```text
1. PBStruct 是否重新生成
2. MessageAttribute 是否存在
3. Opcode 是否正确
4. Request/Response 类型是否对应
5. 前后端 proto 文件是否同版本
6. 是否拿 Response 类型取 ProtoId
7. 后端是否实际返回另一个 Response
```

### 根源处理

协议问题必须前后端对齐，不建议前端写类型转换兜底。

如果协议不一致，应该重新生成协议并确认 Opcode。

## 14. 最常用的排查链路

遇到问题时，先判断是哪一类：

```text
启动问题
查 GameManager 和 LifeCycle。

配置问题
查 ConfigManager、DB.Manager、Tables、Beans、bytes。

平台登录问题
查 WebService.Manager 和 PlatformManager。

游戏服接口问题
查 NetMessageManager、PBStruct、Feature/Service。

活动问题
查 Activity、Promotion、Service、Module、Duty、residentactivities。

UI 问题
查 Controller、View、RefreshEvent、AutoGen 绑定。

平台构建问题
查 Tuanjie 扩展、Build Settings、Player Settings、平台宏。
```

## 15. 根源解决优先级

建议按这个顺序处理问题：

```text
1. 先复现问题
2. 找到具体链路
3. 定位断在哪一层
4. 修断点
5. 删除临时兜底
6. 验证正常路径
7. 验证异常路径
8. 写成笔记
```

兜底只适合：

```text
用户网络异常提示
服务端返回合法空数据
配置允许为空
平台能力不存在
```

不适合：

```text
协议不一致
配置没导表
AutoGen 被手改
Manager 没初始化
活动没注册
UI 绑定错误
```

## 16. 大白话总结

这个项目很多问题看起来是界面问题，其实不是。

比如：

```text
活动没显示
不一定是 UI 没写，可能是配置没有、活动没注册、后端没开、时间没到。

按钮没反应
不一定是按钮坏了，可能是接口没返回、协议没注册、本地数据没更新。

配置为空
不一定是代码没判空，可能是导表、bytes、ConfigManager 这一整条链断了。

登录卡住
不一定是登录 UI 问题，可能是平台服、游戏服、某个 Feature.LoginReq 卡住。
```

排查时记住一句话：

```text
不要在最后一层 UI 上补洞，要回到数据来的地方修。
```

真正稳定的解决方式是：

```text
配置从表来
接口从后端来
状态存在 Feature/Module
UI 只负责显示
```

只要这条链路清楚，问题就不会越改越乱。
