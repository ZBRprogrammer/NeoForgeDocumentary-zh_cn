---
sidebar_position: 1
---
# **注册表**(`Registries`)

**注册**(`Registration`)是将模组对象（如[物品][item]、[方块][block]、实体等）告知游戏的过程。注册至关重要，因为未经注册的对象会导致无法解释的行为和崩溃。

**注册表**(`registry`)本质上是一个映射包装器，它将**注册名**(`registry names`)映射到**注册对象**(`registered objects`)，后者常称为**注册项**(`registry entries`)。同一注册表中的注册名必须唯一，但不同注册表可使用相同注册名。最常见的例子是方块（位于 `BLOCKS` 注册表）与其对应物品（位于 `ITEMS` 注册表）共享相同的注册名。

每个注册对象都有唯一的**注册名**(`registry name`)，以 [`ResourceLocation`][resloc] 表示。例如，泥土方块的注册名是 `minecraft:dirt`，僵尸的注册名是 `minecraft:zombie`。模组对象当然不使用 `minecraft` 命名空间，而是使用其模组 ID。

## 原版 vs 模组

为理解 NeoForge 注册表系统的设计决策，我们先看 Minecraft 的实现方式。以方块注册表为例（大多数其他注册表工作方式相同）。

注册表通常注册[单例(`singletons`)][singleton]。这意味着所有注册项仅存在唯一实例。例如，游戏中所有石头方块实际上是同一个实例的多次显示。若需石头方块，可通过引用已注册的方块实例获取。

Minecraft 在 `Blocks` 类中注册所有方块。通过 `register` 方法调用 `Registry#register()`，其中 `BuiltInRegistries.BLOCK` 作为首个参数。注册完成后，Minecraft 基于方块列表执行各种检查（如验证所有方块是否加载了模型的自我检查）。

这一切运作的主要原因是 `Blocks` 类被 Minecraft 提前加载。模组不会被 Minecraft 自动加载，因此需要变通方案。

## 注册方法

NeoForge 提供两种注册方式：`DeferredRegister` 类和 `RegisterEvent`。注意前者是后者的包装器，推荐`DeferredRegister`使用以防止错误。

### `DeferredRegister`

首先创建 `DeferredRegister`：

```java
public static final DeferredRegister<Block> BLOCKS = DeferredRegister.create(
        // 要使用的注册表
        // Minecraft 注册表位于 BuiltInRegistries，NeoForge 注册表位于 NeoForgeRegistries
        // 模组也可添加自定义注册表，请查阅对应模组文档或源代码获取位置
        BuiltInRegistries.BLOCKS,
        // 模组 ID
        ExampleMod.MOD_ID
);
```

然后通过以下任一方法将注册项添加为静态 final 字段（有关 `new Block()` 的参数说明，请参阅[方块][block]文章）：

```java
public static final DeferredHolder<Block, Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
        // 注册名
        "example_block",
        // 要注册对象的供应器
        () -> new Block(...)
);

public static final DeferredHolder<Block, SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
        // 注册名
        "example_block",
        // 接收注册名（ResourceLocation）并创建对象的函数
        registryName -> new SlabBlock(...)
);
```

`DeferredHolder<R, T extends R>` 类持有我们的对象。类型参数 `R` 是注册表类型（此处为 `Block`），`T` 是供应器类型。第一个示例直接注册 `Block`，因此第二个参数为 `Block`。若注册子类对象（如第二个示例的 `SlabBlock`），则此处应提供 `SlabBlock`。

`DeferredHolder<R, T extends R>` 是 `Supplier<T>` 的子类。获取注册对象时可调用 `DeferredHolder#get()`。由于 `DeferredHolder` 继承 `Supplier`，也可使用 `Supplier` 作为字段类型：

```java
public static final Supplier<Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
        // 注册名
        "example_block",
        // 要注册对象的供应器
        () -> new Block(...)
);

public static final Supplier<SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
        // 注册名
        "example_block",
        // 接收注册名（ResourceLocation）并创建对象的函数
        registryName -> new SlabBlock(...)
);
```

:::note
注意：少数场景明确要求 `Holder` 或 `DeferredHolder`，不接受普通 `Supplier`。若需使用，请将 `Supplier` 类型改回 `Holder` 或 `DeferredHolder`。
:::

最后，由于整个系统是注册事件的包装器，需告知 `DeferredRegister` 在必要时附加到注册事件：

```java
// 模组构造函数
public ExampleMod(IEventBus modBus) {
    //highlight-next-line
    ExampleBlocksClass.BLOCKS.register(modBus);
    // 其他初始化代码
}
```

:::info
`DeferredRegister` 为方块、物品和**数据组件**(`DataComponents`)提供了专用变体：[`DeferredRegister.Blocks`][defregblocks]、[`DeferredRegister.Items`][defregitems]、[`DeferredRegister.DataComponents`][defregcomp] 和 [`DeferredRegister.Entities`][defregentity]。
:::

### `RegisterEvent`

第二种注册方式是 `RegisterEvent`。此[事件][event]在每个注册表上触发，时机在模组构造函数之后（因 `DeferredRegister` 在其中注册内部事件处理器）和配置加载之前。`RegisterEvent` 在模组事件总线上触发。

```java
@SubscribeEvent // 在模组事件总线上
public static void register(RegisterEvent event) {
    event.register(
            // 注册表的键
            // 原版注册表从 BuiltInRegistries 获取
            // NeoForge 注册表从 NeoForgeRegistries.Keys 获取
            BuiltInRegistries.BLOCKS,
            // 在此注册对象
            registry -> {
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_1"), new Block(...));
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_2"), new Block(...));
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_3"), new Block(...));
            }
    );
}
```

## 查询注册表

有时需要根据给定 ID 获取注册对象，或获取特定对象的 ID。由于注册表本质是 ID（`ResourceLocation`）到唯一对象的映射（即可逆映射），两种操作均可行：

```java
BuiltInRegistries.BLOCKS.getValue(ResourceLocation.fromNamespaceAndPath("minecraft", "dirt")); // 返回泥土方块
BuiltInRegistries.BLOCKS.getKey(Blocks.DIRT); // 返回资源定位符 "minecraft:dirt"

// 假设 ExampleBlocksClass.EXAMPLE_BLOCK.get() 是 ID 为 "yourmodid:example_block" 的 Supplier<Block>
BuiltInRegistries.BLOCKS.getValue(ResourceLocation.fromNamespaceAndPath("yourmodid", "example_block")); // 返回示例方块
BuiltInRegistries.BLOCKS.getKey(ExampleBlocksClass.EXAMPLE_BLOCK.get()); // 返回资源定位符 "yourmodid:example_block"
```

若仅需检查对象是否存在，也可实现（但仅限键查询）：

```java
BuiltInRegistries.BLOCKS.containsKey(ResourceLocation.fromNamespaceAndPath("minecraft", "dirt")); // true
BuiltInRegistries.BLOCKS.containsKey(ResourceLocation.fromNamespaceAndPath("create", "brass_ingot")); // 仅当安装 Create 模组时为 true
```

如最后示例所示，此方式适用于任意模组 ID，是检查其他模组特定物品是否存在的理想方法。

最后，可遍历注册表所有条目（键或键值对）：

```java
for (ResourceLocation id : BuiltInRegistries.BLOCKS.keySet()) {
    // ...
}
for (Map.Entry<ResourceKey<Block>, Block> entry : BuiltInRegistries.BLOCKS.entrySet()) {
    // ...
}
```

:::note
查询操作始终使用原版 `Registry`，而非 `DeferredRegister`（后者仅为注册工具）。
:::

:::danger
**注册进行中切勿查询注册表！** 查询操作仅可在注册完成后安全使用。
:::

## 自定义注册表

**自定义注册表**(`Custom registries`)允许您定义扩展系统，供其他模组接入。例如，若模组添加法术系统，可将法术设为注册表，允许其他模组添加法术而无需额外操作。此外，还能实现自动同步等功能。

首先创建[注册表键][resourcekey]和注册表本身：

```java
// 以法术注册表为例（忽略法术具体实现）
// 所有 "spell" 均可替换为实际注册内容
public static final ResourceKey<Registry<Spell>> SPELL_REGISTRY_KEY = ResourceKey.createRegistryKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "spells"));
public static final Registry<YourRegistryContents> SPELL_REGISTRY = new RegistryBuilder<>(SPELL_REGISTRY_KEY)
        // 启用整型 ID 同步（用于网络）
        // 仅应在网络上下文（如数据包或纯网络相关 NBT 数据）中使用
        .sync(true)
        // 默认键（类似 minecraft:air）。可选
        .defaultKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "empty"))
        // 限制最大数量（通常不推荐，但在网络等场景可能有意义）
        .maxId(256)
        // 构建注册表
        .create();
```

接着，在 `NewRegistryEvent` 中将注册表注册到根注册表以告知游戏其存在：

```java
@SubscribeEvent // 在模组事件总线上
public static void registerRegistries(NewRegistryEvent event) {
    event.register(SPELL_REGISTRY);
}
```

现在可通过 `DeferredRegister` 或 `RegisterEvent` 注册新内容：

```java
public static final DeferredRegister<Spell> SPELLS = DeferredRegister.create(SPELL_REGISTRY, "yourmodid");
public static final Supplier<Spell> EXAMPLE_SPELL = SPELLS.register("example_spell", () -> new Spell(...));

// 或：
@SubscribeEvent // 在模组事件总线上
public static void register(RegisterEvent event) {
    event.register(SPELL_REGISTRY_KEY, registry -> {
        registry.register(ResourceLocation.fromNamespaceAndPath("yourmodid", "example_spell"), () -> new Spell(...));
    });
}
```

## **数据包注册表**(`Datapack Registries`)

**数据包注册表**(`datapack registry`)（也称**动态注册表**(`dynamic registry`)或**世界生成注册表**(`worldgen registry`)是一种特殊注册表，它在世界加载时从[数据包][datapack] JSON 文件加载数据（因此得名），而非游戏启动时加载。默认数据包注册表主要包括大多数世界生成注册表等。

数据包注册表允许通过 JSON 文件指定内容。这意味着无需编写代码（除非使用[数据生成][datagen]且不愿手动编写 JSON）。每个数据包注册表都有关联的 [`Codec`][codec] 用于序列化，其 ID 决定数据包路径：

- Minecraft 数据包注册表路径格式：`data/yourmodid/registrypath`（如 `data/yourmodid/worldgen/biome`，其中 `worldgen/biome` 是注册表路径）
- 其他（NeoForge 或模组）数据包注册表路径格式：`data/yourmodid/registrynamespace/registrypath`（如 `data/yourmodid/neoforge/biome_modifier`，其中 `neoforge` 是注册表命名空间，`biome_modifier` 是注册表路径）

数据包注册表可通过 `RegistryAccess` 获取。服务端调用 `ServerLevel#registryAccess()`，客户端调用 `Minecraft.getInstance().getConnection()#registryAccess()`（后者仅在连接世界时有效）。获取后可像普通注册表一样查询特定元素或遍历内容。

### 自定义数据包注册表

自定义数据包注册表无需构建 `Registry`，仅需注册表键和至少一个用于（反）序列化的 [`Codec`][codec]。沿用之前的法术示例，将法术注册表注册为数据包注册表如下：

```java
public static final ResourceKey<Registry<Spell>> SPELL_REGISTRY_KEY = ResourceKey.createRegistryKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "spells"));

@SubscribeEvent // 在模组事件总线上
public static void registerDatapackRegistries(DataPackRegistryEvent.NewRegistry event) {
    event.dataPackRegistry(
            // 注册表键
            SPELL_REGISTRY_KEY,
            // 注册表内容的编解码器
            Spell.CODEC,
            // 注册表内容的网络编解码器（通常与普通编解码器相同）
            // 可为普通编解码器的精简版（省略客户端无需的数据）
            // 可为 null（若为 null，则不同步到客户端）
            // 可省略（功能等同于传递 null）
            Spell.CODEC,
            // 通过 RegistryBuilder 配置注册表的消费者
            // 可省略（功能等同于 builder -> {}）
            builder -> builder.maxId(256)
    );
}
```

### 数据包注册表的数据生成

由于手动编写所有 JSON 文件繁琐且易错，NeoForge 提供[数据提供器][datagenindex]自动生成 JSON 文件（适用于内置和自定义数据包注册表）。

首先创建 `RegistrySetBuilder` 并添加条目（一个 `RegistrySetBuilder` 可容纳多个注册表的条目）：

```java
new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        // 通过引导上下文注册已配置的特性（见下文）
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        // 通过引导上下文注册放置的特性（见下文）
    });
```

`bootstrap` lambda 参数用于注册对象，其类型为 `BootstrapContext`。注册对象需调用其 `#register` 方法：

```java
// 对象的资源键
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_CONFIGURED_FEATURE = ResourceKey.create(
    Registries.CONFIGURED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_configured_feature")
);

new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        bootstrap.register(
            // 已配置特性的资源键
            EXAMPLE_CONFIGURED_FEATURE,
            // 实际的已配置特性
            new ConfiguredFeature<>(Feature.ORE, new OreConfiguration(...))
        );
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        // ...
    });
```

若需从其他注册表查询条目，也可使用 `BootstrapContext`：

```java
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_CONFIGURED_FEATURE = ResourceKey.create(
    Registries.CONFIGURED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_configured_feature")
);
public static final ResourceKey<PlacedFeature> EXAMPLE_PLACED_FEATURE = ResourceKey.create(
    Registries.PLACED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_placed_feature")
);

new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        bootstrap.register(EXAMPLE_CONFIGURED_FEATURE, ...);
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        HolderGetter<ConfiguredFeature<?, ?>> otherRegistry = bootstrap.lookup(Registries.CONFIGURED_FEATURE);
        bootstrap.register(EXAMPLE_PLACED_FEATURE, new PlacedFeature(
            otherRegistry.getOrThrow(EXAMPLE_CONFIGURED_FEATURE), // 获取已配置特性
            List.of() // 放置时的无操作参数 - 替换为实际放置参数
        ));
    });
```

最后，在数据提供器中使用 `RegistrySetBuilder`，并将其注册到事件：

```java
@SubscribeEvent // 在模组事件总线上
public static void onGatherData(GatherDataEvent.Client event) {
    // 将生成的注册表对象添加到当前查询提供器以供其他数据生成使用
    this.createDatapackRegistryObjects(
        // 用于生成数据的 RegistrySetBuilder
        new RegistrySetBuilder().add(...),
        // （可选）接收与资源键关联对象的加载条件
        conditions -> {
            conditions.accept(resourceKey, condition);
        },
        // （可选）为其生成条目的模组 ID 集合
        // 默认提供当前模组容器的模组 ID
        Set.of("yourmodid")
    );

    // 可通过 `#create*` 方法或 `#getLookupProvider` 获取实际查询器使用生成条目
    // ...
}
```

[block]: ../blocks/index.md
[blockentity]: ../blockentities/index.md
[codec]: ../datastorage/codecs.md
[datagen]: #数据包注册表的数据生成
[datagenindex]: ../resources/index.md#数据生成
[datapack]: ../resources/index.md#数据
[defregblocks]: ../blocks/index.md#deferredregisterblocks-辅助方法
[defregcomp]: ../items/datacomponents.md#创建自定义数据组件
[defregentity]: ../entities/index.md#entitytype
[defregitems]: ../items/index.md#deferredregisteritems
[event]: events.md
[item]: ../items/index.md
[resloc]: ../misc/resourcelocation.md
[resourcekey]: ../misc/resourcelocation.md#resourcekeys
[singleton]: https://en.wikipedia.org/wiki/Singleton_pattern