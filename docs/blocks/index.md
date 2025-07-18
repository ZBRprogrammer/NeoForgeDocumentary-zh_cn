# **方块**(`Blocks`)

**方块**(`Blocks`)是 Minecraft 世界的核心。它们构成了所有地形、结构和机器。如果您有兴趣制作模组，很可能会添加一些方块。本页将指导您创建方块及其相关操作。

## 万方归一

在开始之前，重要的是理解游戏中每种方块只有一个实例。世界由对该方块在不同位置的数千个引用组成。换句话说，同一个方块只是被多次显示。

因此，方块应仅实例化一次，即在[注册][registration]期间。方块注册后，即可按需使用注册引用。

与其他大多数注册表不同，方块可使用 `DeferredRegister` 的专门版本 `DeferredRegister.Blocks`。`DeferredRegister.Blocks` 基本类似于 `DeferredRegister<Block>`，但有些细微差异：

- 通过 `DeferredRegister.createBlocks("yourmodid")` 创建，而非常规的 `DeferredRegister.create(...)` 方法
- `#register` 返回 `DeferredBlock<T extends Block>`（扩展了 `DeferredHolder<Block, T>`）。`T` 是注册方块的类类型
- 提供了一些注册方块的辅助方法。更多详情见[下文](#deferredregisterblocks-helpers)

现在注册方块：

```java
//BLOCKS 是 DeferredRegister.Blocks
public static final DeferredBlock<Block> MY_BLOCK = BLOCKS.register("my_block", registryName -> new Block(...));
```

注册方块后，所有对新方块 `my_block` 的引用都应使用此常量。例如，要检查给定位置的方块是否为 `my_block`，代码如下：

```java
level.getBlockState(position) // 返回给定世界位置上的方块状态
    //highlight-next-line
    .is(MyBlockRegistrationClass.MY_BLOCK);
```

此方法还带来便利效果：这使得`block1 == block2` 可以使用且可替代 Java 的 `equals` 方法（当然 `equals` 仍有效，但因按引用比较而显得多余）。

:::danger
切勿在注册外调用 `new Block()`！否则会导致问题：

- 方块必须在注册表解冻时创建。NeoForge 会为您解冻注册表并在之后冻结，因此注册期是创建方块的窗口
- 若在注册表重新冻结后尝试创建和/或注册方块，游戏将崩溃并报告 `null` 方块，这非常令人困惑
- 若仍存在悬空方块实例，游戏在同步和保存时将无法识别它，并将其替换为空气
:::

## 创建方块

如前所述，我们首先创建 `DeferredRegister.Blocks`：

```java
public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("yourmodid");
```

### 基础方块

对于无需特殊功能的简单方块（如圆石、木板等），可直接使用 `Block` 类。在注册时，使用 `BlockBehaviour.Properties` 参数实例化 `Block`。该参数可通过 `BlockBehaviour.Properties#of` 创建，并通过调用其方法自定义。最重要的方法包括：

- `setId` - 设置方块的资源键
    - 每个方块**必须**设置此项，否则会抛出异常
- `destroyTime` - 决定方块破坏所需时间
    - 石头破坏时间为 1.5，泥土为 0.5，黑曜石为 50，基岩为 -1（不可破坏）
- `explosionResistance` - 决定方块的爆炸抗性
    - 石头爆炸抗性为 6.0，泥土为 0.5，黑曜石为 1200，基岩为 3600000
- `sound` - 设置方块被击打、破坏或放置时的声音
    - 默认值为 `SoundType.STONE`。更多详情见[声音页面][sounds]
- `lightLevel` - 设置方块的光照等级。接受带 `BlockState` 参数的函数，返回 0 到 15 的值
    - 例如，荧石使用 `state -> 15`，火把使用 `state -> 14`
- `friction` - 设置方块的摩擦系数（光滑度）
    - 默认值为 0.6。冰使用 0.98

例如，简单实现如下：

```java
//BLOCKS 是 DeferredRegister.Blocks
public static final DeferredBlock<Block> MY_BETTER_BLOCK = BLOCKS.register(
    "my_better_block", 
    registryName -> new Block(BlockBehaviour.Properties.of()
        //highlight-start
        .setId(ResourceKey.create(Registries.BLOCK, registryName))
        .destroyTime(2.0f)
        .explosionResistance(10.0f)
        .sound(SoundType.GRAVEL)
        .lightLevel(state -> 7)
        //highlight-end
    ));
```

更多文档请参阅 `BlockBehaviour.Properties` 的源代码。更多示例请查看 Minecraft 使用的值，或参考 `Blocks` 类。

:::note
需理解世界中的方块与物品栏中的方块不同。物品栏中看似方块的实际上是 `BlockItem`（一种特殊的[物品][item]类型，使用时放置方块）。这也意味着创造模式标签页或最大堆叠大小等由对应的 `BlockItem` 处理。

`BlockItem` 必须与方块分开注册。因为方块不一定需要物品，例如不可收集的方块（如火）。
:::

### 更多功能

直接使用 `Block` 仅允许非常基础的方块。若要添加功能（如玩家交互或不同碰撞箱），需要扩展 `Block` 的自定义类。`Block` 类有许多可重写的方法以用于不同操作；更多信息请参阅 `Block`、`BlockBehaviour` 和 `IBlockExtension` 类。另见下文[使用方块][usingblocks]了解方块最常见用例。

若要制作具有不同变体的方块（如底部、顶部和双层变体的台阶），应使用[方块状态][blockstates]。最后，若需存储额外数据的方块（如存储库存的箱子），应使用[方块实体][blockentities]。经验法则是：若状态数量有限且合理（最多几百种），使用方块状态；若状态数量无限或接近无限，使用方块实体。

#### 方块类型

方块类型是用于序列化和反序列化方块对象的[`MapCodec`s][codec]。此 `MapCodec` 通过 `BlockBehaviour#codec` 设置并[注册][registration]到方块类型注册表。目前其唯一用途是生成方块列表报告。每个 `Block` 的子类应创建一次方块类型。例如，`FlowerBlock#CODEC` 表示大多数花的方块类型，而其子类 `WitherRoseBlock` 有单独的方块类型。

若方块子类仅接受 `BlockBehaviour.Properties`，则可用 `BlockBehaviour#simpleCodec` 创建 `MapCodec`。

```java
// 某方块子类
public class SimpleBlock extends Block {
    public SimpleBlock(BlockBehavior.Properties properties) {
        // ...
    }

    @Override
    public MapCodec<SimpleBlock> codec() {
        return SIMPLE_CODEC.get();
    }
}

// 某注册类
public static final DeferredRegister<MapCodec<? extends Block>> REGISTRAR = DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");

public static final Supplier<MapCodec<SimpleBlock>> SIMPLE_CODEC = REGISTRAR.register(
    "simple",
    () -> BlockBehaviour.simpleCodec(SimpleBlock::new)
);
```

若方块子类包含更多参数，则应使用 [`RecordCodecBuilder#mapCodec`][codec] 创建 `MapCodec`，为 `BlockBehaviour.Properties` 参数传入 `BlockBehaviour#propertiesCodec`。

```java
// 某方块子类
public class ComplexBlock extends Block {
    public ComplexBlock(int value, BlockBehavior.Properties properties) {
        // ...
    }

    @Override
    public MapCodec<ComplexBlock> codec() {
        return COMPLEX_CODEC.get();
    }

    public int getValue() {
        return this.value;
    }
}

// 某注册类
public static final DeferredRegister<MapCodec<? extends Block>> REGISTRAR = DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");

public static final Supplier<MapCodec<ComplexBlock>> COMPLEX_CODEC = REGISTRAR.register(
    "simple",
    () -> RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            Codec.INT.fieldOf("value").forGetter(ComplexBlock::getValue),
            BlockBehaviour.propertiesCodec() // 表示 BlockBehavior.Properties 参数
        ).apply(instance, ComplexBlock::new)
    )
);
```

:::note
尽管方块类型目前基本未使用，但随着 Mojang 继续向以编解码器为中心的结构发展，预计未来会变得更加重要。
:::

### `DeferredRegister.Blocks` 辅助方法

我们已讨论过如何创建 `DeferredRegister.Blocks`（[上文](#one-block-to-rule-them-all)）及其返回 `DeferredBlock`。现在看看专门的 `DeferredRegister` 提供的其他实用工具。从 `#registerBlock` 开始：

```java
public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("yourmodid");

public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.register(
    "example_block", registryName -> new Block(
        BlockBehaviour.Properties.of()
            // 必须在方块上设置 ID
            .setId(ResourceKey.create(Registries.BLOCK, registryName))
    )
);

// 同上，但方块属性的构造是潦草的
// setId 也在属性对象内部调用
public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerBlock(
    "example_block",
    Block::new, // 属性将传入的工厂
    BlockBehaviour.Properties.of() // 要使用的属性
);
```

若想使用 `Block::new`，可完全省略工厂：

```java
public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerSimpleBlock(
    "example_block",
    BlockBehaviour.Properties.of() // 要使用的属性
);
```

这与上例完全相同，但更简洁。当然，若想使用 `Block` 的子类而非 `Block` 本身，必须改用之前的方法。

### 资源

注册方块并放置到世界后，会发现缺少纹理等资源。这是因为[纹理][textures]等由 Minecraft 的资源系统处理。要为方块应用纹理，必须提供[模型][model]和[方块状态文件][bsfile]，将方块与纹理和形状关联。更多信息请阅读链接文章。

## 使用方块

方块很少直接用于操作。实际上，Minecraft 中最常见的两种操作（获取位置的方块和设置位置的方块）使用方块状态而非方块。通用设计方法是让方块定义行为，但行为实际通过方块状态运行。因此，`BlockState`s 常作为参数传递给 `Block` 的方法。关于方块状态的使用及如何从方块获取，请参阅[使用方块状态][usingblockstates]。

多种情况下，`Block` 的多个方法在不同时间使用。以下小节列出最常见的方块相关流程。除非另有说明，所有方法均在逻辑两端调用，且应在两端返回相同结果。

### 放置方块

方块放置逻辑从 `BlockItem#useOn` 调用（或其子类实现，如 `PlaceOnWaterBlockItem` 用于睡莲）。关于游戏如何到达此处的更多信息，请参阅[右键点击物品][rightclick]。实践中，这意味着一旦右键点击 `BlockItem`（如圆石物品），就会调用此行为。

- 检查若干前提条件（如非旁观者模式、方块所有必需特性标志已启用或目标位置未超出世界边界）。若任一检查失败，流程终止
- 为尝试放置方块的位置当前方块调用 `BlockBehaviour#canBeReplaced`。若返回 `false`，流程终止。此处返回 `true` 的典型情况是高草或雪层
- 调用 `Block#getStateForPlacement`。根据上下文（包括位置、旋转和放置面等信息）可返回不同方块状态。例如，这对于可朝不同方向放置的方块非常有用
- 使用上一步获得的方块状态调用 `BlockBehaviour#canSurvive`。若返回 `false`，流程终止
- 通过 `Level#setBlock` 调用将方块状态设置到世界中
    - 在此 `Level#setBlock` 调用中，调用 `BlockBehaviour#onPlace`
- 调用 `Block#setPlacedBy`

### 破坏方块

破坏方块更复杂，因为它需要时间。过程大致分为三个阶段："启动"("initiating")、"挖掘"("mining")和"实际破坏"("actually breaking")。

- 点击鼠标左键时进入"启动"阶段
- 按住鼠标左键时进入"挖掘"阶段。**此阶段的方法每刻调用**
- 若"持续"阶段未被中断（松开鼠标左键）且方块被破坏，则进入"实际破坏"阶段

伪代码如下：

```java
leftClick();
initiatingStage();
while (leftClickIsBeingHeld()) {
    miningStage();
    if (blockIsBroken()) {
        actuallyBreakingStage();
        break;
    }
}
```

以下小节将这些阶段分解为实际方法调用。关于游戏如何从点击左键进入此流程的信息，请参阅[左键点击物品][leftclick]。

#### "启动"阶段

- 检查若干前提条件（如非旁观者模式、主手物品堆(`ItemStack`)所有必需特性标志已启用或目标方块未超出世界边界）。若任一检查失败，流程终止
- 触发 `PlayerInteractEvent.LeftClickBlock`。若事件取消，流程终止
    - 注意：客户端取消事件时，不会向服务端发送数据包，因此服务端不运行逻辑
    - 但服务端取消此事件仍会导致客户端代码运行，可能导致不同步！
- 调用 `BlockBehaviour#attack`

#### "挖掘"阶段

- 触发 `PlayerInteractEvent.LeftClickBlock`。若事件取消，流程进入"完成"阶段
    - 注意：客户端取消事件时，不会向服务端发送数据包，因此服务端不运行逻辑
    - 但服务端取消此事件仍会导致客户端代码运行，可能导致不同步！
- 调用 `BlockBehaviour#getDestroyProgress` 并添加到内部破坏进度计数器
    - `BlockBehaviour#getDestroyProgress` 返回 0 到 1 的浮点值，表示每刻破坏进度计数器的增量
- 相应更新进度覆盖（裂纹纹理）
- 若破坏进度大于 1.0（即完成，即方块应被破坏），则退出"挖掘"阶段并进入"实际破坏"阶段

#### "实际破坏"阶段

- 调用 `Item#canDestroyBlock`。若返回 `false`（确定方块不应被破坏），流程进入"完成"阶段
- 若方块是 `GameMasterBlock` 实例，则调用 `Player#canUseGameMasterBlocks`。这决定玩家是否有能力破坏仅创造模式方块。若 `false`，流程进入"完成"阶段
- 仅服务端：调用 `Player#blockActionRestricted`。这决定当前玩家是否无法破坏方块。若 `true`，流程进入"完成"阶段
- 仅服务端：触发 `BlockEvent.BreakEvent`。若取消，流程进入"完成"阶段。初始取消状态由上述三个方法决定
- 调用 `Block#playerWillDestroy`
- 仅服务端：调用 `IBlockExtension#canHarvestBlock`。这决定方块是否可收获（即破坏时掉落）。若 `Player#preventsBlockDrops` 返回 `true` 则忽略
    - 仅服务端：若未重写 `IBlockExtension#canHarvestBlock` 或其未调用超类方法，则触发 `PlayerEvent.HarvestCheck`。若 `canHarvest` 返回 `false`，则不调用 `Block#playerDestroy`，阻止任何资源或经验掉落
- 调用 `IBlockExtension#onDestroyedByPlayer`。若返回 `false`，流程进入"完成"阶段
    - 通过 `Level#setBlock` 调用（以 `Blocks.AIR.defaultBlockState()` 或当前记录的流体为方块状态参数）从世界中移除方块状态
        - 在此 `Level#setBlock` 调用中，调用 `Block#onRemove`
    - 若 `IBlockExtension#onDestroyedByPlayer` 返回 `true`，则调用 `Block#destroy`
- 仅服务端：若先前 `IBlockExtension#canHarvestBlock` 和 `IBlockExtension#onDestroyedByPlayer` 返回 `true`，则调用 `Block#playerDestroy`
    - 仅服务端：调用 `Block#dropResources`。这决定挖掘时方块的掉落物，包括经验
        - 仅服务端：触发 `BlockDropsEvent`。若事件取消，则方块破坏时不掉落任何物品。否则，将 `BlockDropsEvent#getDrops` 中的每个 `ItemEntity` 添加到当前世界。此外，若 `getDroppedExperience` 大于 0，则调用 `Block#popExperience`
            - 仅服务端：调用 `IBlockExtension#getExpDrop`（由 `EnchantmentHelper#processBlockExperience` 增强）。这是在可能被修改前为 `BlockDropsEvent#getDroppedExperience` 设置的初始值
- 仅服务端：若挖掘方块的物品在上述过程中损坏，则触发 `PlayerDestroyItemEvent`

#### 挖掘速度

挖掘速度根据方块的硬度、使用的[工具][tool]速度和若干实体[属性][attributes]计算，规则如下：

```java
// 返回工具的挖掘速度，若手持物品为空、非工具或不适用于被破坏的方块则返回 1
float destroySpeed = item.getDestroySpeed(blockState);
// 若有适用工具，将 minecraft:mining_efficiency 属性作为加法修饰符添加
if (destroySpeed > 1) {
    destroySpeed += player.getAttributeValue(Attributes.MINING_EFFICIENCY);
}
// 应用急迫或潮涌能量效果
if (player.hasEffect(MobEffects.HASTE) || player.hasEffect(MobEffects.CONDUIT_POWER)) {
    int haste = player.hasEffect(MobEffects.HASTE)
        ? player.getEffect(MobEffects.HASTE).getAmplifier()
        : 0;
    int conduitPower = player.hasEffect(MobEffects.CONDUIT_POWER)
        ? player.getEffect(MobEffects.CONDUIT_POWER).getAmplifier()
        : 0;
    int amplifier = Math.max(haste, conduitPower);
    destroySpeed *= 1 + (amplifier + 1) * 0.2f;
}
// 应用挖掘疲劳效果
if (player.hasEffect(MobEffects.MINING_FATIGUE)) {
    destroySpeed *= switch (player.getEffect(MobEffects.MINING_FATIGUE).getAmplifier()) {
        case 0 -> 0.3F;
        case 1 -> 0.09F;
        case 2 -> 0.0027F;
        default -> 8.1E-4F;
    };
}
// 将 minecraft:block_break_speed 属性作为乘法修饰符添加
destroySpeed *= player.getAttributeValue(Attributes.BLOCK_BREAK_SPEED);
// 若玩家在水中，乘法应用水下挖掘速度惩罚
if (player.isEyeInFluid(FluidTags.WATER)) {
    destroySpeed *= player.getAttributeValue(Attributes.SUBMERGED_MINING_SPEED);
}
// 若玩家试图在半空中破坏方块，使玩家挖掘速度减慢 5 倍
if (!player.onGround()) {
    destroySpeed /= 5;
}
destroySpeed = /* 在此触发 PlayerEvent.BreakSpeed 事件，允许模组进一步修改此值 */;
return destroySpeed;
```

具体代码可参考 `Player#getDestroySpeed`。

### 刻处理

刻处理机制每 1/20 秒（50 毫秒，"一刻"）更新（刻处理）游戏部分。方块提供在不同方式下调用的不同刻处理方法。

#### 服务端刻处理与刻调度

`BlockBehaviour#tick` 在两种情况下调用：通过默认[随机刻][randomtick]（见下文）或通过计划刻。计划刻通过 `Level#scheduleTick(BlockPos, Block, int)` 创建，其中 `int` 表示延迟。原版在多个地方使用此机制，例如大型垂滴叶的倾斜机制严重依赖此系统。其他著名用户是各种红石元件。

#### 客户端刻处理

`Block#animateTick` 仅在客户端每帧调用。这是客户端专属行为（如火把粒子生成）发生的地方。

#### 天气刻处理

天气刻处理由 `Block#handlePrecipitation` 处理，独立于常规刻处理运行。仅在服务端调用，且仅在某种形式下雨时有 1/16 的几率调用。例如，雨水或雪填满炼药锅时使用此机制。

#### 随机刻处理

随机刻系统独立于常规刻处理运行。随机刻必须通过方块的 `BlockBehaviour.Properties` 调用 `BlockBehaviour.Properties#randomTicks()` 方法启用。这使方块能成为随机刻机制的一部分。

随机刻每刻在区块中为设定数量的方块发生。该数量由 `randomTickSpeed` 游戏规则定义。默认值为 3 时，每刻从区块中选择 3 个随机方块。若这些方块启用了随机刻，则调用其对应的 `BlockBehaviour#randomTick` 方法。

随机刻在 Minecraft 中用于多种机制，如植物生长、冰和雪融化或铜氧化。

[above]: #one-block-to-rule-them-all
[attributes]: ../entities/attributes.md
[below]: #deferredregisterblocks-helpers
[blockentities]: ../blockentities/index.md
[blockstates]: states.md
[bsfile]: ../resources/client/models/index.md#blockstate-files
[codec]: ../datastorage/codecs.md#records
[item]: ../items/index.md
[leftclick]: ../items/interactions.md#left-clicking-an-item
[model]: ../resources/client/models/index.md
[randomtick]: #random-ticking
[registration]: ../concepts/registries.md#methods-for-registering
[resources]: ../resources/index.md#assets
[rightclick]: ../items/interactions.md#right-clicking-an-item
[sounds]: ../resources/client/sounds.md
[textures]: ../resources/client/textures.md
[tool]: ../items/tools.md
[usingblocks]: #using-blocks
[usingblockstates]: states.md#using-blockstates