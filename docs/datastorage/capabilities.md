---
sidebar_position: 6
---
# **能力系统**(`Capabilities`)

**能力系统**(`Capabilities`)允许以动态灵活的方式暴露功能，无需直接实现多个接口。

简言之，每种能力以接口形式提供一项功能。

NeoForge 为方块、实体和物品堆栈添加了能力支持。下文将详细说明。

## 为何使用能力系统？

能力系统旨在将方块、实体或物品堆栈的**功能**(`what`)与其**实现方式**(`how`)分离。若不确定能力系统是否适用，请思考以下问题：

1. 是否只关心方块/实体/物品堆栈的**功能**，而不关心其**实现方式**？
2. 该功能是否仅适用于部分（而非全部）方块/实体/物品堆栈？
3. 该功能的实现是否依赖于特定方块/实体/物品堆栈？

以下是能力系统的优秀用例：
- *"希望流体容器兼容其他模组，但不了解每个容器的具体实现"* - 是，使用 `IFluidHandler` 能力
- *"想统计实体物品数量，但不知实体如何存储"* - 是，使用 `IItemHandler` 能力
- *"想为物品堆栈充能，但不知其如何存储能量"* - 是，使用 `IEnergyStorage` 能力
- *"想为玩家目标方块上色，但不知方块如何转换"* - 是（NeoForge 未提供此能力，但可自行实现）

以下是应避免的用例：
- *"检查实体是否在机器范围内"* - 否，应改用辅助方法

## NeoForge 提供的能力

NeoForge 为以下三个接口提供能力：`IItemHandler`、`IFluidHandler` 和 `IEnergyStorage`。

`IItemHandler` 暴露处理物品栏位的接口，其能力类型包括：
- `Capabilities.ItemHandler.BLOCK`：方块自动化可访问的物品栏（箱子、机器等）
- `Capabilities.ItemHandler.ENTITY`：实体的物品栏内容（额外玩家栏位、生物背包等）
- `Capabilities.ItemHandler.ENTITY_AUTOMATION`：实体自动化可访问的物品栏（船、矿车等）
- `Capabilities.ItemHandler.ITEM`：物品堆栈内容（便携背包等）

`IFluidHandler` 暴露处理流体容器的接口，其能力类型包括：
- `Capabilities.FluidHandler.BLOCK`：方块自动化可访问的流体容器
- `Capabilities.FluidHandler.ENTITY`：实体的流体容器
- `Capabilities.FluidHandler.ITEM`：物品堆栈的流体容器（因桶装流体特性，此能力为特殊 `IFluidHandlerItem` 类型）

`IEnergyStorage` 基于 TeamCoFH 的 RedstoneFlux API 暴露处理能量容器的接口，其能力类型包括：
- `Capabilities.EnergyStorage.BLOCK`：方块内部能量
- `Capabilities.EnergyStorage.ENTITY`：实体内部能量
- `Capabilities.EnergyStorage.ITEM`：物品堆栈内部能量

## 创建能力

NeoForge 支持为方块、实体和物品堆栈创建能力。

能力系统允许通过调度逻辑查找 API 实现。NeoForge 实现的能力类型包括：
- `BlockCapability`：方块与方块实体的能力（行为取决于具体 `Block`）
- `EntityCapability`：实体的能力（行为取决于具体 `EntityType`）
- `ItemCapability`：物品堆栈的能力（行为取决于具体 `Item`）

:::tip
为兼容其他模组，建议优先使用 `Capabilities` 类中 NeoForge 提供的能力。否则可按本节描述创建自定义能力。
:::

创建能力仅需单次函数调用，结果对象应存储在 `static final` 字段中。需提供以下参数：
- 能力名称（相同名称多次创建将返回同一对象）
- 被查询的行为类型（类型参数 `T`）
- 查询中的额外上下文类型（类型参数 `C`）

例如，声明支持方向感知的方块 `IItemHandler` 能力：

```java
public static final BlockCapability<IItemHandler, @Nullable Direction> ITEM_HANDLER_BLOCK =
    BlockCapability.create(
        // 提供唯一标识能力的名称
        ResourceLocation.fromNamespaceAndPath("mymod", "item_handler"),
        // 提供被查询类型（此处为 IItemHandler）
        IItemHandler.class,
        // 提供上下文类型（允许查询接收额外 Direction side 参数）
        Direction.class);
```

因 `@Nullable Direction` 在方块中很常见，有专用辅助方法：

```java
public static final BlockCapability<IItemHandler, @Nullable Direction> ITEM_HANDLER_BLOCK =
    BlockCapability.createSided(
        // 提供唯一标识能力的名称
        ResourceLocation.fromNamespaceAndPath("mymod", "item_handler"),
        // 提供被查询类型
        IItemHandler.class);
```

若无需上下文，应使用 `Void`（也有专用辅助方法）：

```java
public static final BlockCapability<IItemHandler, Void> ITEM_HANDLER_NO_CONTEXT =
    BlockCapability.createVoid(
        // 提供唯一标识能力的名称
        ResourceLocation.fromNamespaceAndPath("mymod", "item_handler_no_context"),
        // 提供被查询类型
        IItemHandler.class);
```

实体与物品堆栈能力分别在 `EntityCapability` 和 `ItemCapability` 中有类似方法。

## 查询能力

将 `BlockCapability`、`EntityCapability` 或 `ItemCapability` 对象存入静态字段后，即可查询能力。

实体和物品堆栈可通过 `getCapability` 查找能力实现（返回 `null` 表示无可用实现）：

```java
var object = entity.getCapability(CAP, context);
if (object != null) {
    // 使用 object
}
```

```java
var object = stack.getCapability(CAP, context);
if (object != null) {
    // 使用 object
}
```

方块能力查询方式不同（因无方块实体的方块也可拥有能力），需在 `level` 上额外提供目标 `pos` 进行查询：

```java
var object = level.getCapability(CAP, pos, context);
if (object != null) {
    // 使用 object
}
```

若已知方块实体和/或方块状态，可传入以节省查询时间：

```java
var object = level.getCapability(CAP, pos, blockState, blockEntity, context);
if (object != null) {
    // 使用 object
}
```

具体示例（从 `Direction.NORTH` 方向查询方块的 `IItemHandler` 能力）：

```java
IItemHandler handler = level.getCapability(Capabilities.ItemHandler.BLOCK, pos, Direction.NORTH);
if (handler != null) {
    // 使用 handler 执行物品相关操作
}
```

## 方块能力缓存

查询能力时，系统底层执行以下步骤：
1. 获取方块实体和方块状态（若未提供）
2. 获取注册的能力提供器（见下文）
3. 遍历提供器询问是否能提供能力
4. 某个提供器返回能力实例（可能分配新对象）

该实现虽高效，但对高频查询（如每游戏刻）仍可能消耗大量服务器时间。`BlockCapabilityCache` 系统可显著加速对特定位置的能力查询。

:::tip
通常，`BlockCapabilityCache` 创建一次后存储在执行高频查询的对象字段中。具体存储位置由您决定。
:::

创建缓存需调用 `BlockCapabilityCache.create`，并传入能力、维度、位置和查询上下文：

```java
// 声明字段：
private BlockCapabilityCache<IItemHandler, @Nullable Direction> capCache;

// 后续使用（如在方块实体的 onLoad 中）：
this.capCache = BlockCapabilityCache.create(
    Capabilities.ItemHandler.BLOCK, // 要缓存的能力
    level, // 维度
    pos, // 目标位置
    Direction.NORTH // 上下文
);
```

随后通过 `getCapability()` 查询缓存：

```java
IItemHandler handler = this.capCache.getCapability();
if (handler != null) {
    // 使用 handler 执行物品相关操作
}
```

**缓存由垃圾回收器自动清理，无需手动注销。**

还可接收能力对象变更通知！包括能力变更（`oldHandler != newHandler`）、不可用（`null`）或重新可用（非 `null`）。

此时需额外参数创建缓存：
- **有效性检查**(`validity check`)：确定缓存是否有效
  - 作为方块实体字段时，`() -> !this.isRemoved()` 即可
- **失效监听器**(`invalidation listener`)：能力变更时调用
  - 可在此响应能力变更、移除或出现

```java
// 含失效监听器：
this.capCache = BlockCapabilityCache.create(
    Capabilities.ItemHandler.BLOCK, // 要缓存的能力
    level, // 维度
    pos, // 目标位置
    Direction.NORTH, // 上下文
    () -> !this.isRemoved(), // 有效性检查（因缓存可能比所属对象更长寿）
    () -> onCapInvalidate() // 失效监听器
);
```

## 方块能力失效

:::info
**失效**(`Invalidation`)是方块能力特有机制。实体和物品堆栈能力无法缓存，故无需失效。
:::

为确保缓存正确更新存储的能力，**模组开发者必须在能力变更、出现或消失时调用 `level.invalidateCapabilities(pos)`**。

```java
// 能力变更、出现或消失时：
level.invalidateCapabilities(pos);
```

NeoForge 已处理区块加载/卸载和方块实体创建/移除等常见情况，但其他情况需模组开发者显式处理。例如以下情况必须失效能力：
- 先前返回的能力不再有效
- 提供能力的方块（无方块实体）被放置或状态变更时（通过重写 `onPlace`）
- 提供能力的方块（无方块实体）被移除时（通过重写 `onRemove`）

基础方块示例参考 `ComposterBlock.java` 文件。更多信息参阅 [`IBlockCapabilityProvider`][block-cap-provider] 的 Javadoc。

## 注册能力

**能力提供器**(`capability provider`)是最终提供能力的函数。它能返回能力实例或 `null`（表示无法提供）。提供器特定于：
- 其提供的能力类型
- 提供能力的方块实例、方块实体类型、实体类型或物品实例

提供器需在 `RegisterCapabilitiesEvent` 中注册。

方块提供器用 `registerBlock` 注册：

```java
@SubscribeEvent // 在模组事件总线上
public static void registerCapabilities(RegisterCapabilitiesEvent event) {
    event.registerBlock(
        Capabilities.ItemHandler.BLOCK, // 要注册的能力
        (level, pos, state, be, side) -> <返回 IItemHandler>,
        // 要注册的方块列表
        MY_ITEM_HANDLER_BLOCK,
        MY_OTHER_ITEM_HANDLER_BLOCK
    );
}
```

通常注册针对特定方块实体类型，故提供 `registerBlockEntity` 辅助方法：

```java
event.registerBlockEntity(
    Capabilities.ItemHandler.BLOCK, // 要注册的能力
    MY_BLOCK_ENTITY_TYPE, // 要注册的方块实体类型
    (myBlockEntity, side) -> myBlockEntity.myIItemHandlerForTheGivenSide // 返回 IItemHandler
);
```

:::danger
若方块或方块实体提供器先前返回的能力不再有效，**必须通过调用 `level.invalidateCapabilities(pos)` 使缓存失效**。详见上文[失效机制][invalidation]。
:::

实体注册类似（使用 `registerEntity`）：

```java
event.registerEntity(
    Capabilities.ItemHandler.ENTITY, // 要注册的能力
    MY_ENTITY_TYPE, // 要注册的实体类型
    (myEntity, context) -> myEntity.myIItemHandlerForTheGivenContext // 返回 IItemHandler
);
```

物品注册也类似（注意提供器接收物品堆栈）：

```java
event.registerItem(
    Capabilities.ItemHandler.ITEM, // 要注册的能力
    (itemStack, context) -> <返回物品堆栈的 IItemHandler>,
    // 要注册的物品列表
    MY_ITEM,
    MY_OTHER_ITEM
);
```

## 为所有对象注册能力

若需为所有方块/实体/物品注册提供器，需遍历对应注册表并为每个对象注册。

例如，NeoForge 使用此系统为所有 `BucketItem`（不含子类）注册流体处理器能力：

```java
// 参考代码位于 CapabilityHooks 类
for (Item item : BuiltInRegistries.ITEM) {
    if (item.getClass() == BucketItem.class) {
        event.registerItem(Capabilities.FluidHandler.ITEM, (stack, ctx) -> new FluidBucketWrapper(stack), item);
    }
}
```

提供器按注册顺序被询问能力。若需在 NeoForge 之前运行，请以更高优先级注册 `RegisterCapabilitiesEvent` 处理器：

```java
// 使用 HIGH 优先级以在 NeoForge 之前注册！
@SubscribeEvent(priority = EventPriority.HIGH) // 在模组事件总线上
public static void registerCapabilities(RegisterCapabilitiesEvent event) {
    event.registerItem(
        Capabilities.FluidHandler.ITEM,
        (stack, ctx) -> new MyCustomFluidBucketWrapper(stack), // 自定义包装器
        MY_CUSTOM_BUCKET // 要注册的物品
    );
}
```

NeoForge 自身注册的提供器列表参见 [`CapabilityHooks`][capability-hooks]。

[block-cap-provider]: https://github.com/neoforged/NeoForge/blob/1.21.x/src/main/java/net/neoforged/neoforge/capabilities/IBlockCapabilityProvider.java
[capability-hooks]: https://github.com/neoforged/NeoForge/blob/1.21.x/src/main/java/net/neoforged/neoforge/capabilities/CapabilityHooks.java
[invalidation]: #方块能力失效