# **特性标志**(`Feature Flags`)

**特性标志**(`Feature flags`)是一种系统，允许开发者将一组特性置于某些必需的标志之后，这些标志可以是注册元素、游戏机制、数据包条目或模组特有的其他系统。

常见用例是将实验性特性/元素置于实验标志之后，允许用户在特性最终确定前轻松启用并进行测试。

:::tip
您无需强制添加自己的标志。如果发现原版标志适合您的用例，可以自由使用该标志标记您的方块/物品/实体等。

例如在 `1.21.3` 版本中，若需添加到浅色橡木方块集合中，应仅在 `WINTER_DROP` 标志启用时才显示这些方块。
:::

## 创建特性标志

要创建新的特性标志，需创建 JSON 文件并在 `neoforge.mods.toml` 文件的 `[[mods]]` 块中通过 `featureFlags` 条目引用。指定路径必须相对于 `resources` 目录：

```toml
# 在 neoforge.mods.toml 中：
[[mods]]
    # 文件路径相对于资源的输出目录，或编译后 jar 内的根路径
    # 'resources' 目录表示资源的根输出目录
    featureFlags="META-INF/feature_flags.json"
```

条目定义包含要注册的特性标志名称列表，这些标志将在游戏初始化期间加载并注册。

```json5
{
    "flags": [
        // 要注册的特性标志标识符
        "examplemod:experimental"
    ]
}
```

## 获取特性标志

可通过 `FeatureFlagRegistry.getFlag(ResourceLocation)` 获取已注册的特性标志。这可在模组初始化期间的任何时间完成，建议将标志存储在某处以供后续使用，而非每次需要时都查询注册表。

```java
// 查找 'examplemod:experimental' 特性标志
public static final FeatureFlag EXPERIMENTAL = FeatureFlags.REGISTRY.getFlag(ResourceLocation.fromNamespaceAndPath("examplemod", "experimental"));
```

## 特性元素

`FeatureElement` 是可赋予一组必需标志的注册值。仅当所需标志与关卡中启用的标志匹配时，这些值才对玩家可用。

当特性元素被禁用时，它会完全从玩家视野中隐藏，所有交互将被跳过。请注意，这些禁用元素仍存在于注册表中，仅是功能上不可用。

以下是直接实现 `FeatureElement` 系统的完整注册表列表：

- 物品(`Item`)
- 方块(`Block`)
- 实体类型(`EntityType`)
- 菜单类型(`MenuType`)
- 药水(`Potion`)
- 状态效果(`MobEffect`)

### 标记元素

要将给定 `FeatureElement` 标记为需要您的特性标志，只需将其及任何其他所需标志传递到相应的注册方法中：

- `Item`：`Item.Properties#requiredFeatures`
- `Block`：`BlockBehaviour.Properties#requiredFeatures`
- `EntityType`：`EntityType.Builder#requiredFeatures`
- `MenuType`：`MenuType#new`
- `Potion`：`Potion#requiredFeatures`
- `MobEffect`：`MobEffect#requiredFeatures`

```java
// 这些元素仅在 'EXPERIMENTAL' 标志启用后可用

// 物品
DeferredRegister.Items ITEMS = DeferredRegister.createItems("examplemod");
DeferredItem<Item> EXPERIMENTAL_ITEM = ITEMS.registerSimpleItem("experimental", new Item.Properties()
    .requiredFeatures(EXPERIMENTAL) // 标记为需要 'EXPERIMENTAL' 标志
);

// 方块
DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("examplemod");
// 注意 BlockBehaviour.Properties#ofFullCopy 和 BlockBehaviour.Properties#ofLegacyCopy 会复制必需特性
// 这意味着在 1.21.3 中，使用 BlockBehaviour.Properties.ofFullCopy(Blocks.PALE_OAK_WOOD) 会使您的方块需要 'WINTER_DROP' 标志
DeferredBlock<Block> EXPERIMENTAL_BLOCK = BLOCKS.registerSimpleBlock("experimental", BlockBehaviour.Properties.of()
    .requiredFeatures(EXPERIMENTAL) // 标记为需要 'EXPERIMENTAL' 标志
);

// 方块物品特殊之处在于其必需特性继承自对应方块
// 生成蛋同理继承自对应实体类型
DeferredItem<BlockItem> EXPERIMENTAL_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(EXPERIMENTAL_BLOCK);

// 实体类型
DeferredRegister<EntityType<?>> ENTITY_TYPES = DeferredRegister.create(Registries.ENTITY_TYPE, "examplemod");
DeferredHolder<EntityType<?>, EntityType<ExperimentalEntity>> EXPERIMENTAL_ENTITY = ENTITY_TYPES.register("experimental", registryName -> EntityType.Builder.of(ExperimentalEntity::new, MobCategory.AMBIENT)
    .requiredFeatures(EXPERIMENTAL) // 标记为需要 'EXPERIMENTAL' 标志
    .build(ResourceKey.create(Registries.ENTITY_TYPE, registryName))
);

// 菜单类型
DeferredRegister<MenuType<?>> MENU_TYPES = DeferredRegister.create(Registries.MENU, "examplemod");
DeferredHolder<MenuType<?>, MenuType<ExperimentalMenu>> EXPERIMENTAL_MENU = MENU_TYPES.register("experimental", () -> new MenuType<>(
    // 使用原版 MenuSupplier：
    // 当菜单在 `player.openMenu` 期间不编码复杂数据时使用。示例：
    // (windowId, inventory) -> new ExperimentalMenu(windowId, inventory),

    // 使用 NeoForge 的 IContainerFactory：
    // 当您希望读取 `player.openMenu` 期间编码的复杂数据时使用
    // 此处强制转换很重要，因为 `MenuType` 明确期望 `MenuSupplier`
    (IContainerFactory<ExperimentalMenu>) (windowId, inventory, buffer) -> new ExperimentalMenu(windowId, inventory, buffer),
    
    FeatureFlagSet.of(EXPERIMENTAL) // 标记为需要 'EXPERIMENTAL' 标志
));

// 状态效果
DeferredRegister<MobEffect> MOB_EFFECTS = DeferredRegister.create(Registries.MOB_EFFECT, "examplemod");
DeferredHolder<MobEffect, ExperimentalMobEffect> EXPERIMENTAL_MOB_EEFECT = MOB_EFFECTS.register("experimental", registryName -> new ExperimentalMobEffect(MobEffectCategory.NEUTRAL, CommonColors.WHITE)
    .requiredFeatures(EXPERIMENTAL) // 标记为需要 'EXPERIMENTAL' 标志
);

// 药水
DeferredRegister<Potion> POTIONS = DeferredRegister.create(Registries.POTION, "examplemod");
DeferredHolder<Potion, ExperimentalPotion> EXPERIMENTAL_POTION = POTIONS.register("experimental", registryName -> new ExperimentalPotion(registryName.toString(), new MobEffectInstance(EXPERIMENTAL_MOB_EEFECT))
    .requiredFeatures(EXPERIMENTAL) // 标记为需要 'EXPERIMENTAL' 标志
);
```

### 验证启用状态

要验证特性是否应启用，必须先获取已启用的特性集合。可通过多种方式完成，但常用且推荐的方法是使用 `LevelReader#enabledFeatures`。

```java
level.enabledFeatures(); // 通过 'LevelReader' 实例
entity.level().enabledFeatures(); // 通过 'Entity' 实例

// 客户端
minecraft.getConnection().enabledFeatures();

// 服务端
server.getWorldData().enabledFeatures();
```

要验证任何 `FeatureFlagSet` 是否启用，可将已启用特性传递给 `FeatureFlagSet#isSubsetOf`。要验证特定 `FeatureElement` 是否启用，可调用 `FeatureElement#isEnabled`。

:::note
`ItemStack` 有特殊的 `isItemEnabled(FeatureFlagSet)` 方法。这是为了确保即使支持物品的必需特性与已启用特性不匹配，空堆栈也被视为启用。建议尽可能优先使用此方法而非 `Item#isEnabled`。
:::

```java
requiredFeatures.isSubsetOf(enabledFeatures);
featureElement.isEnabled(enabledFeatures);
itemStack.isItemEnabled(enabledFeatures);
```

## 特性包

_另见：[资源包](../resources/index.md#assets)、[数据包](../resources/index.md#data) 和 [Pack.mcmeta](../resources/index.md#packmcmeta)_

特性包是一种不仅能加载资源和/或数据，还能切换给定特性标志集合的包类型。这些标志定义在包根目录的 `pack.mcmeta` JSON 文件中，格式如下：

:::note
此文件与模组 `resources/` 目录中的文件不同。此文件定义全新的特性包，因此必须位于自己的文件夹中。
:::

```json5
{
    "features": {
        "enabled": [
            // 要启用的特性标志标识符
            // 必须是有效的已注册标志
            "examplemod:experimental"
        ]
    },
    "pack": { /*...*/ }
}
```

用户获取特性包的方式主要有两种：从外部来源作为数据包安装，或下载带有内置特性包的模组。这两种方式都需要根据[物理端](../concepts/sides.md)进行不同安装。

### 内置包

内置包与您的模组捆绑，使用 `AddPackFindersEvent` 事件向游戏提供。

```java
@SubscribeEvent // 在模组事件总线上
public static void addFeaturePacks(final AddPackFindersEvent event) {
    event.addPackFinders(
            // 相对于模组 'resources' 的路径，指向此包
            // 注意这会使用以下格式定义包的 ID
            // mod/<命名空间>:<路径>`，例如 `mod/examplemod:data/examplemod/datapacks/experimental`
            ResourceLocation.fromNamespaceAndPath("examplemod", "data/examplemod/datapacks/experimental"),
            
            // 此包包含的资源类型
            // 'CLIENT_RESOURCES' 表示包含客户端资源（资源包）
            // 'SERVER_DATA' 表示包含服务端数据（数据包）
            PackType.SERVER_DATA,
            
            // 在实验屏幕上显示的展示名称
            Component.literal("ExampleMod：实验特性"),
            
            // 为使此包能加载并启用特性标志，此处必须为 'FEATURE'
            // 任何其他 PackSource 类型在此均无效
            PackSource.FEATURE,
            
            // 若为 true，则包始终激活且无法禁用，特性包应始终为 false
            false,
            
            // 从此包加载资源的优先级
            // 'TOP' 表示此包优先于其他包
            // 'BOTTOM' 表示其他包优先于此包
            Pack.Position.TOP
    );
}
```

#### 在单人游戏中启用

1. 创建新世界
2. 导航至实验屏幕
3. 切换所需包
4. 点击 `完成` 确认更改

#### 在多人游戏中启用

1. 打开服务器的 `server.properties` 文件
2. 将特性包 ID 添加到 `initial-enabled-packs` 中，各包用 `,` 分隔（包 ID 在注册包查找器时定义，如上所示）

### 外部包

外部包以数据包形式提供给用户。

#### 在单人游戏中安装

1. 创建新世界
2. 导航至数据包选择屏幕
3. 将数据包 zip 文件拖放到游戏窗口
4. 将新出现的数据包移至 `已选择` 列表
5. 点击 `完成` 确认更改

游戏将警告您任何新选择的实验特性、潜在错误、问题和崩溃。可点击 `继续` 确认更改，或点击 `详情` 查看所有选定包及其将启用的特性的详细列表。

:::note
外部特性包不会显示在实验屏幕上。实验屏幕仅显示内置特性包。

启用后要禁用外部特性包，需返回数据包屏幕，将外部包从 `已选择` 移回 `可用`。
:::

#### 在多人游戏中安装

特性包只能在初始创建世界时启用，一旦启用便无法禁用。

1. 创建目录 `./world/datapacks`
2. 将数据包 zip 文件上传至新创建的目录
3. 打开服务器的 `server.properties` 文件
4. 将数据包 zip 文件名（不含 `.zip`）添加到 `initial-enabled-packs`（各包用 `,` 分隔）
   - 示例：zip 文件 `examplemod-experimental.zip` 应添加为 `initial-enabled-packs=vanilla,examplemod-experimental`

### 数据生成

_另见：[数据生成](../resources/index.md#data-generation)_

特性包可在常规模组数据生成期间生成。这最好与内置包结合使用，但也可将生成结果打包为外部包共享。请选择一种方式，即不要同时提供外部包和内置包。

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(final GatherDataEvent.Client event) {
    DataGenerator generator = event.getGenerator();
    
    // 要生成特性包，必须先获取所需包的包生成器实例
    // generator.getBuiltinDatapack(<是否生成>, <命名空间>, <路径>);
    // 这将生成特性包到以下路径：
    // ./data/<命名空间>/datapacks/<路径>
    PackGenerator featurePack = generator.getBuiltinDatapack(true, "examplemod", "experimental");
        
    // 注册提供器以生成 `pack.mcmeta` 文件
    featurePack.addProvider(output -> PackMetadataGenerator.forFeaturePack(
            output,
            
            // 实验屏幕上显示的描述
            Component.literal("为 ExampleMod 启用实验特性"),
            
            // 此包应启用的特性标志集合
            FeatureFlagSet.of(EXPERIMENTAL)
    ));
    
    // 将其他提供器（配方、战利品表）注册到 `featurePack`，将生成的资源写入此包而非根包
}
```