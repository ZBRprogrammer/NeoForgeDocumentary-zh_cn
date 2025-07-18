---
sidebar_position: 2
---
# **端**(`Sides`)

与许多程序类似，Minecraft 遵循客户端-服务器架构，其中**客户端**(`client`)负责显示数据，而**服务器**(`server`)负责更新数据。使用这些术语时，我们对其含义有相当直观的理解……对吧？

然而事实并非如此。许多混淆源于 Minecraft 根据上下文有两种不同的端概念：**物理端**(`physical side`)和**逻辑端**(`logical side`)。

## 逻辑端 vs 物理端

### 物理端

打开 Minecraft 启动器，选择一个安装版本并点击启动时，您启动的是**物理客户端**(`physical client`)。“物理”在此处的含义是“这是一个客户端程序”。这意味着客户端功能（如所有渲染相关功能）在此可用并可随时调用。相反，**物理服务器**(`physical server`)（也称**专用服务器**(`dedicated server`)）是当您启动 Minecraft 服务器 JAR 时开启的程序。虽然 Minecraft 服务器带有基础 GUI，但它缺少所有仅客户端功能。最显著的是，服务器 JAR 中缺少各种客户端类。在物理服务器上调用这些类将导致类缺失错误（即崩溃），因此需防范此类情况。

### 逻辑端

逻辑端主要关注 Minecraft 的内部程序结构。**逻辑服务器**(`logical server`)是游戏逻辑运行的地方。时间与天气变化、实体刻处理、实体生成等均在服务器上运行。各类数据（如物品栏内容）也由服务器负责。**逻辑客户端**(`logical client`)则负责显示所有需呈现的内容。Minecraft 将所有客户端代码隔离在 `net.minecraft.client` 包中，并在名为**渲染线程**(`Render Thread`)的独立线程中运行，其余代码被视为公共（即客户端与服务器共享）代码。

### 区别何在？

物理端与逻辑端的区别可通过两种场景说明：

- 玩家加入**多人游戏**(`multiplayer`)世界。这相对简单：玩家的物理（及逻辑）客户端连接到其他位置的物理（及逻辑）服务器——玩家不关心服务器位置；只要连接成功，客户端就无需知晓其他信息。
- 玩家进入**单人游戏**(`singleplayer`)世界。此处情况变得有趣。玩家的物理客户端会启动一个逻辑服务器，随后以逻辑客户端的身份连接到同一机器上的该逻辑服务器。若熟悉网络概念，可将其视为连接到 `localhost`（仅为概念类比；实际不涉及套接字等）。

这两种场景也揭示了主要问题：若逻辑服务器能运行您的代码，并不能保证物理服务器也能运行。因此应始终通过专用服务器测试以检查意外行为。由于客户端与服务器分离不当导致的 `NoClassDefFoundError` 和 `ClassNotFoundException` 是模组开发中最常见的错误。另一常见错误是使用静态字段并从逻辑端两侧访问；此问题尤为棘手，因为通常无异常迹象。

:::tip
若需将数据从一端传输到另一端，必须[发送数据包][networking]。
:::

在 NeoForge 代码库中，物理端由 `Dist` 枚举表示，逻辑端由 `LogicalSide` 枚举表示。

:::info
历史上，服务器 JAR 包含客户端没有的类。现代版本中已非如此；物理服务器可视为物理客户端的子集。
:::

## 执行端特定操作

### `Level#isClientSide()`

此布尔检查将是您最常用的端判断方式。在 `Level` 对象上查询此字段可确定该维度所属的**逻辑端**(`logical side`)：若为 `true`，则维度运行于逻辑客户端；若为 `false`，则运行于逻辑服务器。因此物理服务器的此字段恒为 `false`，但无法反推 `false` 即物理服务器，因为物理客户端内的逻辑服务器（即单人世界）此字段也可能为 `false`。

当需确定是否应运行游戏逻辑或其他机制时，请使用此检查。例如，若需在玩家点击方块时造成伤害，或让机器将泥土加工为钻石，应确保 `#isClientSide` 为 `false` 后再执行。在逻辑客户端应用游戏逻辑可能导致不同步（幽灵实体、统计信息不同步等）或崩溃。

:::tip
此检查应作为默认首选。只要存在可用的 `Level`，就使用此检查。
:::

### `FMLEnvironment.dist`

`FMLEnvironment.dist` 是 `Level#isClientSide()` 的**物理端**(`physical`)对应项。若此字段为 `Dist.CLIENT`，表示处于物理客户端；若为 `Dist.DEDICATED_SERVER`，则表示处于物理服务器。

#### `@Mod`

处理仅客户端类时，检查物理环境至关重要。推荐通过单独的 [`@Mod` 注解][mod]分离仅应在单一物理端执行的代码，并将 `dist` 参数设置为模组类应加载的物理端：

```java
@Mod("examplemod")
public class ExampleMod {
    public ExampleMod(IEventBus modBus) {
        // 执行应同时在两端运行的逻辑
    }
}

@Mod(value = "examplemod", dist = Dist.CLIENT) 
public class ExampleModClient {
    public ExampleModClient(IEventBus modBus) {
        // 执行应仅在物理客户端运行的逻辑
    }
}

@Mod(value = "examplemod", dist = Dist.DEDICATED_SERVER) 
public class ExampleModDedicatedServer {
    public ExampleModDedicatedServer(IEventBus modBus) {
        // 执行应仅在物理服务器运行的逻辑
    }
}
```

:::tip
模组通常被期望在两端均可运行。这意味着若开发仅客户端模组，应验证其确实在物理客户端运行，否则执行空操作。
:::

[networking]: ../networking/index.md
[mod]: ../gettingstarted/modfiles.md#javafml-and-mod