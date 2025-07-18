# **资源**(`Resources`)

资源是游戏使用的外部文件（非代码）。最显著的是**纹理**(`textures`)，但 Minecraft 生态中还存在许多其他类型资源。这些资源需由代码端消费系统使用，因此相关系统也归入本节。

Minecraft 通常有两类资源：
- [**逻辑客户端(`logical client`)**][logicalsides]资源（称为**资产**(`assets`)）
- [**逻辑服务器(`logical server`)**][logicalsides]资源（称为**数据**(`data`)）

资产主要是显示信息（如纹理、显示模型、翻译文本、音效），数据则包含影响游戏性的内容（如战利品表、配方、世界生成信息）。它们分别从**资源包**(`resource packs`)和**数据包**(`data packs`)加载。NeoForge 为每个模组生成内置资源包和数据包。

资源包和数据包通常需 [`pack.mcmeta` 文件][packmcmeta]，但现代 NeoForge 会在运行时生成，因此无需手动创建。

若对格式有疑问，可参考原版资源。NeoForge 开发环境不仅包含原版代码，还包含原版资源。位置如下：
- **IntelliJ**：External Resources 部分
- **Eclipse**：Project Libraries 部分
- 名称：`ng_dummy_ng.net.minecraft:client:client-extra:<minecraft_version>`（Minecraft 资源）或 `ng_dummy_ng.net.neoforged:neoforge:<neoforge_version>`（NeoForge 资源）

## 资产

_另见：[Minecraft Wiki 上的资源包][mcwikiresourcepacks]_

资产（即客户端资源）是仅与[客户端][sides]相关的资源。它们通过资源包（旧称“材质包”）加载。资源包本质上是 `assets` 文件夹，其中包含各**命名空间**(`namespaces`)的子文件夹（每个命名空间对应一个子文件夹）。例如：
- `coolmod` 模组的资源包包含 `coolmod` 命名空间
- 也可包含其他命名空间（如 `minecraft`）

NeoForge 自动将所有模组资源包收集到 `Mod resources` 包中，位于资源包菜单的底层。当前无法禁用此包，但优先级更高的资源包可覆盖底层资源。这使资源包制作者能覆盖模组资源，模组开发者也可覆盖 Minecraft 资源。

资源包可包含以下文件夹：

| 文件夹名称       | 内容                                |
|------------------|-------------------------------------|
| `atlases`        | **纹理图集**(`Texture Atlas`)源文件 |
| `blockstates`    | [方块状态文件][bsfile]              |
| `equipment`      | [装备信息][equipment]               |
| `font`           | 字体定义                            |
| `items`          | [客户端物品][citems]                |
| `lang`           | [翻译文件][translations]            |
| `models`         | [模型][models]                      |
| `particles`      | [粒子定义][particles]               |
| `post_effect`    | 后期处理屏幕效果                    |
| `shaders`        | 元数据、片段和顶点着色器            |
| `sounds`         | [音效文件][sounds]                  |
| `texts`          | 杂项文本文件                        |
| `textures`       | [纹理][textures]                    |
| `waypoint_style` | 路径点图标元数据                    |

## 数据

_另见：[Minecraft Wiki 上的数据包][mcwikidatapacks]_

数据是与[服务器][sides]相关的资源术语。类似资源包，数据通过**数据包**(`data packs`)加载。数据包包含 [`pack.mcmeta` 文件][packmcmeta]和 `data` 根文件夹，其中 `data` 又包含各命名空间的子文件夹。

NeoForge 会在创建新世界时自动应用所有模组数据包。当前无法禁用模组数据包，但优先级更高的数据包可覆盖（或通过空文件移除）大多数数据文件。额外数据包可放入世界的 `datapacks` 子文件夹，并通过 [`/datapack`][datapackcmd] 命令启用/禁用。

:::info
当前无内置方法为所有世界应用自定义数据包集合，但部分模组可实现此功能。
:::

数据包可包含以下文件夹：

| 文件夹名称                                                                                     | 内容                          |
|------------------------------------------------------------------------------------------------|-------------------------------|
| `advancement`                                                                                  | [进度][advancements]          |
| `banner_pattern`                                                                               | 旗帜图案                      |
| `cat_variant`, `chicken_variant`, `cow_variant`, `frog_variant`, `pig_variant`, `wolf_variant` | 实体变种                      |
| `damage_type`                                                                                  | [伤害类型][damagetypes]       |
| `datapacks`                                                                                    | 内置数据包                    |
| `dialog`                                                                                       | 对话框                        |
| `enchantment`, `enchantment_provider`                                                          | [附魔][enchantment]           |
| `instrument`, `jukebox_song`, `wolf_sound_variant`                                             | 音效引用元数据                |
| `painting_variant`                                                                             | 画作                          |
| `loot_table`                                                                                   | [战利品表][loottables]        |
| `recipe`                                                                                       | [配方][recipes]               |
| `tags`                                                                                         | [标签][tags]                  |
| `test_environment`, `test_instance`                                                            | [游戏测试][gmt]               |
| `trial_spawner`                                                                                | 战斗挑战                      |
| `trim_material`, `trim_pattern`                                                                | 盔甲纹饰                      |
| `neoforge/data_maps`                                                                           | [数据映射][datamap]           |
| `neoforge/loot_modifiers`                                                                      | [全局战利品修改器][glm]       |
| `dimension`, `dimension_type`, `structure`, `worldgen`, `neoforge/biome_modifier`              | 世界生成文件                  |

此外，还可能包含与命令集成的系统子文件夹（模组中较少使用）：

| 文件夹名称     | 内容                       |
|----------------|----------------------------|
| `chat_type`    | [聊天类型][chattype]       |
| `function`     | [函数][function]           |
| `item_modifier`| [物品修改器][itemmodifier] |
| `predicate`    | [谓词][predicate]          |

## `pack.mcmeta`

_另见：[Minecraft Wiki 上的资源包 pack.mcmeta][packmcmetaresourcepack] 和数据包 [pack.mcmeta][packmcmetadatapack]_

`pack.mcmeta` 文件存储资源包/数据包的元数据。对模组而言，NeoForge 使此文件过时（因其会合成生成）。完整规范见 Minecraft Wiki 文章。

## **数据生成**(`Data Generation`)

数据生成（俗称 **datagen**）通过编程生成 JSON 资源文件，避免手动编写易出错。名称易误解，因其同时适用于资产和数据。

Datagen 通过 **Data 运行配置**执行（与 Client/Server 配置一同生成）。该配置遵循[模组生命周期][lifecycle]，在**注册事件**(`registry events`)后触发 [`GatherDataEvent`s][event]，可在其中以**数据提供器**(`data providers`)形式注册待生成对象，写入磁盘后结束进程。

有两种操作于[物理端(`physical side`)][physicalside]的子类型：
- `GatherDataEvent.Client`：可包含所有生成提供器
- `GatherDataEvent.Server`：仅包含生成数据包条目的提供器

:::note
注册提供器有两种推荐方式：
1. 全部注册到 `GatherDataEvent.Client`，使用 `runClientData` 任务生成
2. 客户端提供器注册到 `GatherDataEvent.Client`，服务端提供器注册到 `GatherDataEvent.Server`，分别通过 `runClientData` 和 `runServerData` 生成

MDK 默认采用第一种方案（设置 `clientData` 配置），因此示例均注册到 `GatherDataEvent.Client`。
:::

所有数据提供器均继承 `DataProvider` 接口，通常需重写一个方法。以下是 Minecraft 和 NeoForge 提供的重要数据生成器（详情见链接文章）：

| 类                                                    | 方法                           | 生成内容                                                      | 端     | 备注                                                                 |
|-------------------------------------------------------|--------------------------------|-------------------------------------------------------------|--------|----------------------------------------------------------------------|
| [`ModelProvider`][modelprovider]                     | `registerModels()`             | 模型、方块状态文件、客户端物品                                | 客户端 |                                                                      |
| [`LanguageProvider`][langprovider]                   | `addTranslations()`            | 翻译                                                        | 客户端 | 构造器需传入语言                                                     |
| [`ParticleDescriptionProvider`][particleprovider]    | `addDescriptions()`            | 粒子定义                                                    | 客户端 |                                                                      |
| [`SoundDefinitionsProvider`][soundprovider]          | `registerSounds()`             | 音效定义                                                    | 客户端 |                                                                      |
| `SpriteSourceProvider`                               | `gather()`                     | 精灵源/图集                                                  | 客户端 |                                                                      |
| [`AdvancementProvider`][advancementprovider]         | `generate()`                   | 进度                                                        | 服务端 | 需额外类支持（见文章）                                               |
| [`LootTableProvider`][loottableprovider]             | `generate()`                   | 战利品表                                                    | 服务端 | 需额外方法和类支持（见文章）                                         |
| [`RecipeProvider`][recipeprovider]                   | `buildRecipes(RecipeOutput)`   | 配方                                                        | 服务端 |                                                                      |
| [`TagsProvider` 子类][tagsprovider]                  | `addTags(HolderLookup.Provider)` | 标签                                                        | 服务端 | 存在多个专用子类（见文章）                                           |
| [`DataMapProvider`][datamapprovider]                 | `gather()`                     | 数据映射条目                                                | 服务端 |                                                                      |
| [`GlobalLootModifierProvider`][glmprovider]          | `start()`                      | 全局战利品修改器                                            | 服务端 |                                                                      |
| [`DatapackBuiltinEntriesProvider`][datapackprovider] | N/A                            | 数据包内置条目（如世界生成和[伤害类型][damagetypes]）        | 服务端 | 无重写方法，条目在构造器 lambda 中添加（见文章）                     |
| `JsonCodecProvider`（抽象类）                        | `gather()`                     | 含[编解码器(`codec`)][codec]的对象                          | 双端   | 可扩展用于任何含编解码器的对象                                       |

这些提供器遵循相同模式：创建子类添加资源 → 在[事件处理器][eventhandler]中将提供器加入事件。以 `RecipeProvider` 为例：

```java
public class MyRecipeProvider extends RecipeProvider {
    public MyRecipeProvider(HolderLookup.Provider registries, RecipeOutput output) {
        super(registries, output);
    }

    @Override
    protected void buildRecipes() {
        // 在此注册配方
    }

    // 数据提供器类
    public static class Runner extends RecipeProvider.Runner {

        public Runner(PackOutput output, CompletableFuture<HolderLookup.Provider> registries) {
            super(output, registries);
        }

        @Override
        protected abstract RecipeProvider createRecipeProvider(HolderLookup.Provider registries, RecipeOutput output) {
            return new MyRecipeProvider(registries, output);
        }
    }
}

// 在某个事件处理器类中
@SubscribeEvent // 在模组事件总线
public static void gatherData(GatherDataEvent.Client event) {
    // 数据提供器应先调用 event.createDatapackRegistryObjects(...)
    // 注册数据包注册表对象，供其他提供器在生成时使用

    // 通常通过 event.createProvider(...) 注册提供器
    // 该函数提供 PackOutput 和可选的 CompletableFuture<HolderLookup.Provider>

    // 注册提供器
    event.createProvider(MyRecipeProvider.Runner::new);
    // 其他数据提供器

    // 若要在全局包中创建数据包，可调用
    // DataGenerator#getBuiltinDatapack
    // 然后使用 PackGenerator#addProvider 添加提供器
    DataGenerator.PackGenerator examplePack = event.getGenerator().getBuiltinDatapack(
        true, // 应始终为 true
        "examplemod", // 模组 ID
        "example_pack" // 包名称
    );
    
    examplePack.addProvider(output -> ...);
}
```

事件提供以下辅助工具和上下文：
- `event.createDatapackRegistryObjects(...)`：通过 `RegistrySetBuilder` 创建并注册 `DatapackBuiltinEntriesProvider`，强制后续查找提供器包含生成条目
- `event.createProvider(...)`：通过 lambda 提供 `PackOutput` 和 `CompletableFuture<HolderLookup.Provider>` 注册提供器
- `event.createBlockAndItemTags(...)`：通过 `TagsProvider<Block>` 构造 `TagsProvider<Item>` 并注册两者
- `event.getGenerator()`：返回注册提供器的 `DataGenerator`
- `event.getPackOutput()`：返回 `PackOutput`（某些提供器用于确定文件输出位置）
- `event.getResourceManager(PackType)`：返回 `ResourceManager`（用于检查已存在文件）
- `event.getLookupProvider()`：返回 `CompletableFuture<HolderLookup.Provider>`（主要用于标签和数据生成注册表引用其他元素）
- `event.includeDev()` 和 `event.includeReports()`：检查是否启用特定命令行参数

### 命令行参数

数据生成器接受以下命令行参数：
- `--mod examplemod`：指定运行该模组的 datagen（NeoGradle 自动添加所属模组 ID，多模组项目需手动添加）
- `--output path/to/folder`：指定输出目录（建议用 Gradle 的 `file(...).getAbsolutePath()` 生成绝对路径）。默认 `file('src/generated/resources').getAbsolutePath()`
- `--existing path/to/folder`：指定检查已存在文件时考虑的目录
- `--existing-mod examplemod`：指定检查已存在文件时考虑的模组 JAR 资源
- 生成模式（布尔参数，无需额外值）：
    - `--includeDev`：是否运行开发工具（模组通常不用）。运行时通过 `GatherDataEvent#includeDev()` 检查
    - `--includeReports`：是否转储已注册对象列表。运行时通过 `GatherDataEvent#includeReports()` 检查
    - `--all`：启用所有生成模式

参数可通过 `build.gradle` 添加到运行配置：

```groovy
runs {
    // 其他运行配置

    clientData {
        arguments.addAll '--arg1', 'value1', '--arg2', 'value2', '--all' // 布尔参数无值
    }
}
```

例如，复制默认参数：

```groovy
runs {
    // 其他运行配置

    clientData {
        arguments.addAll '--mod', 'examplemod', // 替换为你的模组 ID
                '--output', file('src/generated/resources').getAbsolutePath(),
                '--all'
    }
}
```

[advancementprovider]: server/advancements.md#数据生成
[advancements]: server/advancements.md
[bsfile]: client/models/index.md#方块状态文件
[chattype]: https://minecraft.wiki/w/Chat_type
[citems]: client/models/items.md
[codec]: ../datastorage/codecs.md
[damagetypes]: server/damagetypes.md
[datamap]: server/datamaps/index.md
[datamapprovider]: server/datamaps/index.md#数据生成
[datapackcmd]: https://minecraft.wiki/w/Commands/datapack
[datapackprovider]: ../concepts/registries.md#数据包注册表的数据生成
[enchantment]: server/enchantments/index.md
[equipment]: ../items/armor.md#装备模型
[event]: ../concepts/events.md
[eventhandler]: ../concepts/events.md#注册事件处理器
[function]: https://minecraft.wiki/w/Function_(Java_Edition)
[glm]: server/loottables/glm.md
[glmprovider]: server/loottables/glm.md#数据生成
[gmt]: ../misc/gametest.md
[itemmodifier]: https://minecraft.wiki/w/Item_modifier
[langprovider]: client/i18n.md#数据生成
[lifecycle]: ../concepts/events.md#模组生命周期
[logicalsides]: ../concepts/sides.md#逻辑端
[loottableprovider]: server/loottables/index.md#数据生成
[loottables]: server/loottables/index.md
[mcwiki]: https://minecraft.wiki
[mcwikidatapacks]: https://minecraft.wiki/w/Data_pack
[mcwikiresourcepacks]: https://minecraft.wiki/w/Resource_pack
[modelprovider]: client/models/datagen.md
[models]: client/models/index.md
[packmcmeta]: #packmcmeta
[packmcmetadatapack]: https://minecraft.wiki/w/Data_pack#pack.mcmeta
[packmcmetaresourcepack]: https://minecraft.wiki/w/Resource_pack#内容
[particleprovider]: client/particles.md#数据生成
[particles]: client/particles.md
[physicalside]: ../concepts/sides.md#物理端
[predicate]: https://minecraft.wiki/w/Predicate
[recipeprovider]: server/recipes/index.md#数据生成
[recipes]: server/recipes/index.md
[sides]: ../concepts/sides.md
[soundprovider]: client/sounds.md#数据生成
[sounds]: client/sounds.md
[tags]: server/tags.md
[tagsprovider]: server/tags.md#数据生成
[textures]: client/textures.md
[translations]: client/i18n.md#语言文件