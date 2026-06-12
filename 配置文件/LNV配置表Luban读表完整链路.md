# LNV 配置表 Luban 读表完整链路

## 1. 这篇笔记解决什么问题

这篇记录 LNV 项目里前端读取配置表的完整链路，重点是看懂：

- 配置表源文件在哪里。
- Luban 生成的 `Tables` 和 `Beans` 分别是干嘛的。
- `.bytes` 配置如何被客户端加载。
- 业务代码怎么通过 `ConfigManager.DB.xxx` 读表。
- 新增字段或新增表后应该怎么接。
- 为什么 `AutoGen` 目录不要手改。

核心代码位置：

```text
D:\LNV\trunk\client\Hotfix\Core\Config\ConfigManager.cs
D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Manager.cs
D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Tables
D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Beans
D:\LNV\trunk\client\Hotfix\Database\LubanMaual
D:\LNV\trunk\client\Assets\DynamicRes\Tables\Luban
D:\LNV\trunk\server_bin\config\src
```

## 2. 配置表整体链路

完整链路可以理解为：

```text
CSV/Excel 源配置
-> Luban 生成 C# 代码
-> Luban 生成 bytes 数据
-> Unity/Tuanjie 打包到 cfg 资源
-> ConfigManager 加载 cfg
-> DB.Manager 构造所有表对象
-> Tables.xxx 反序列化 bytes
-> Beans.xxx 表示单行数据
-> 业务/UI 通过 ConfigManager.DB.xxx 读取
```

最常见的业务读取方式：

```csharp
var hero = ConfigManager.DB.hero.GetByIdx(heroId);
var allHeroes = ConfigManager.DB.hero.DataList;
```

## 3. 关键目录说明

```text
D:\LNV\trunk\server_bin\config\src
服务端或导表源配置目录，通常是 CSV 源数据。

D:\LNV\trunk\client\Assets\DynamicRes\Tables\Luban
客户端实际运行时加载的 bytes 配置文件。

D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Manager.cs
Luban 自动生成的总表管理类，里面有每张表的属性。

D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Tables
每张表的容器类，负责读取整张表、构建索引、提供 GetByXxx 方法。

D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Beans
每一行数据的数据结构，字段都在这里。

D:\LNV\trunk\client\Hotfix\Database\LubanMaual
人工扩展 partial 逻辑的位置，适合写额外索引、额外方法。
```

## 4. ConfigManager 做什么

`ConfigManager` 是客户端配置表入口。

它的职责是：

```text
1. 加载 cfg 资源里的所有 TextAsset/bytes
2. 把 bytes 包装成 Bright.Serialization.ByteBuf
3. 创建 DB.Manager
4. DB.Manager 再去构造所有 Luban 表对象
5. 最后业务通过 ConfigManager.DB 访问所有表
```

核心代码逻辑：

```csharp
public static DB.Manager DB { get; private set; }

public async Task Init()
{
    var assets = await LoadText();
    assetByteBufs = new Dictionary<string, ByteBuf>();

    foreach (var kv in assets)
    {
        assetByteBufs.Add(kv.Key, new ByteBuf(kv.Value));
    }

    DB = new DB.Manager(LoadByteBufForLuban);
}
```

这里的关键点是：

```text
LoadText()
负责把所有配置 bytes 加载进内存。

LoadByteBufForLuban(name)
按表名返回对应 ByteBuf。

new DB.Manager(...)
开始真正初始化所有 Luban 表。
```

## 5. DB.Manager 是什么

`DB.Manager` 是 Luban 自动生成的总入口。

文件：

```text
D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Manager.cs
```

它里面会有很多属性：

```csharp
public Tables.Monster Monster { get; }
public Tables.Language Language { get; }
public Tables.Skill Skill { get; }
public Tables.hero hero { get; }
```

这些属性就是业务里 `ConfigManager.DB.xxx` 的来源。

例如：

```csharp
ConfigManager.DB.hero
```

实际就是 `DB.Manager` 里的 `Tables.hero` 实例。

## 6. Tables 是干嘛的

以：

```text
D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Tables\hero.cs
```

为例。

`Tables.hero` 是整张 hero 表的容器。

它主要做三件事：

```text
1. 从 ByteBuf 里循环反序列化每一行 Beans.hero
2. 放进 DataList
3. 建立索引字典，比如 _dataMap_idx
```

典型结构：

```csharp
private readonly List<Beans.hero> _dataList;
private Dictionary<int, Beans.hero> _dataMap_idx;

public List<Beans.hero> DataList => _dataList;

public Beans.hero GetByIdx(int key)
    => _dataMap_idx.TryGetValue(key, out Beans.hero __v) ? __v : null;
```

所以：

```csharp
ConfigManager.DB.hero.DataList
```

表示整张表。

```csharp
ConfigManager.DB.hero.GetByIdx(1001)
```

表示按主键 `Idx` 查一行。

## 7. Beans 是干嘛的

以：

```text
D:\LNV\trunk\client\Hotfix\Database\AutoGen\Luban\Beans\hero.cs
```

为例。

`Beans.hero` 表示 hero 表中的一行数据。

里面会生成字段属性：

```csharp
public int Idx { get; private set; }
public string Comment { get; private set; }
public int Type { get; private set; }
public int Occupation { get; private set; }
public string HeroName { get; private set; }
public Dictionary<string, int> SkillC { get; private set; }
public List<Database.Reward> RewardC { get; private set; }
```

字段来源就是配置表表头。

如果配置表新增字段，重新生成后这里会多出对应属性。

如果业务读字段编译不过，说明生成代码里没有这个字段，要回头检查：

```text
源表是否真的加了字段
字段名是否一致
导表是否成功
客户端 AutoGen 是否更新
bytes 是否更新
```

## 8. LubanMaual 是干嘛的

`AutoGen` 是自动生成代码，不应该手改。

如果需要给某张表加人工逻辑，应该放到：

```text
D:\LNV\trunk\client\Hotfix\Database\LubanMaual
```

因为 Luban 生成的类一般是 `partial class`，可以用 partial 扩展。

适合放在 `LubanMaual` 的逻辑：

```text
额外查询方法
额外索引
特殊数据转换
PostInit
PostResolve
业务辅助方法
```

不适合放在 AutoGen：

```text
手写字段
手写 GetByXxx
手写业务判断
临时兜底数据
```

原因很简单：下一次导表生成会覆盖 AutoGen，你手改的东西会丢。

## 9. 前端读表标准流程

以前端读取 hero 表为例：

```text
1. 确认配置表里有 hero 数据
2. 确认 Luban 已生成 tables_hero.bytes
3. 确认 AutoGen 里有 Tables.hero 和 Beans.hero
4. 确认 ConfigManager.Init 已执行
5. 业务代码用 ConfigManager.DB.hero 读取
6. UI 使用读到的数据刷新显示
```

代码示例：

```csharp
var heroCfg = ConfigManager.DB.hero.GetByIdx(heroId);
if (heroCfg == null)
{
    LOG.Game.E("hero config not found: {0}", heroId);
    return;
}

var heroName = LanguageHelper.GetLanguageByID(heroCfg.HeroName);
```

如果需要遍历：

```csharp
foreach (var cfg in ConfigManager.DB.hero.DataList)
{
    if (cfg.Show == 0) continue;
    // 刷新 UI 数据
}
```

## 10. 新增配置字段步骤

新增字段不要只改前端。

标准步骤：

```text
1. 在源配置表加字段
2. 确认字段名、类型、客户端/服务端导出标记
3. 执行 Luban 导表
4. 确认 AutoGen\Beans\xxx.cs 里生成了字段
5. 确认 bytes 文件更新
6. 前端业务代码读取新字段
7. Unity/Tuanjie 运行验证
```

验证点：

```text
Beans.xxx 里能看到新属性
Tables.xxx 能正常构造
ConfigManager.DB.xxx.GetByIdx 能读到数据
运行时没有 ByteBuf 解析错误
UI 显示符合预期
```

## 11. 新增配置表步骤

新增一张表的完整流程：

```text
1. 添加源表
2. 配置 Luban 导出规则
3. 生成 AutoGen Tables/Beans
4. 生成对应 tables_xxx.bytes
5. 确认 DB.Manager 里出现新表属性
6. 前端通过 ConfigManager.DB.xxx 读取
7. 如果需要特殊方法，在 LubanMaual 写 partial 扩展
```

判断是否接成功：

```csharp
var list = ConfigManager.DB.xxx.DataList;
LOG.Game.I("xxx config count: {0}", list.Count);
```

如果 `ConfigManager.DB.xxx` 编译都找不到，说明 `DB.Manager` 没生成这张表。

如果能编译但运行为空，优先查 bytes 是否打进客户端。

## 12. 读表失败排查

读表失败按这个顺序查：

```text
1. 源表有没有数据
2. 是否重新导表
3. AutoGen 是否更新
4. bytes 是否更新
5. bytes 是否在 Assets/DynamicRes/Tables/Luban
6. cfg 资源是否被 Addressables/AssetBundle 打包
7. ConfigManager.Init 有没有执行
8. ConfigManager.DB 是否为空
9. 表名大小写是否一致
10. key 是否真的存在
```

常见现象：

```text
编译报字段不存在
说明 AutoGen 没更新，或者字段名不一致。

运行时报 bytes 解析异常
说明代码和 bytes 不是同一版。

GetByIdx 返回 null
说明 key 不存在，或者配置没导进去。

DataList 为空
说明 bytes 没数据，或者没加载到这张表。

ConfigManager.DB 为空
说明 ConfigManager.Init 没成功执行。
```

## 13. 当前需要注意的源码问题

当前检查到：

```text
D:\LNV\trunk\client\Hotfix\Core\Config\ConfigManager.cs
```

文件内容看起来存在结构异常，`LoadText` 方法和类括号位置不完整。

如果你遇到读表、编译或配置初始化相关问题，优先检查这个文件是否被误改、合并冲突、复制粘贴破坏。

根源性处理方式：

```text
1. 不要在业务层写默认假数据兜底。
2. 先恢复 ConfigManager.cs 正确结构。
3. 确认 LoadText 能完整返回 cfg bytes。
4. 再确认 DB.Manager 能正常创建。
```

读表是底层链路，如果这里坏了，上层 UI、活动、接口都会出现连锁问题。

## 14. 难点总结

配置表难点主要有四个：

```text
代码和 bytes 必须同版本
AutoGen 不能手改
表名、字段名、索引名要完全对上
配置加载发生在异步初始化阶段
```

很多问题表面是 UI 不显示，其实是配置没读到。

很多问题表面是字段为空，其实是导表没更新。

很多问题表面是接口异常，其实是前端根据配置拼请求参数时读到了错误值。

## 15. 大白话总结

配置表链路可以理解成：

```text
策划填表
-> Luban 把表变成 C# 类和 bytes 文件
-> 客户端启动时把 bytes 读进来
-> DB.Manager 把每张表变成对象
-> 业务代码用 ConfigManager.DB.xxx 查数据
-> UI 再把查到的数据显示出来
```

`Tables` 是整张表。

`Beans` 是表里的一行。

`Manager` 是所有表的总入口。

`AutoGen` 是机器生成的，不要手改。

如果配置读不到，不要先写兜底。先查源表、导表、bytes、AutoGen、ConfigManager 初始化这条根链路，根链路通了，上层业务才会稳定。
