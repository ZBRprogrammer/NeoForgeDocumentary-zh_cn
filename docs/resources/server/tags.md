# 标签(`Tags`)

简单来说，**标签**(`tag`)是相同类型注册对象的列表。它们从数据文件加载，可用于成员资格检查。例如，制作木棍配方会接受任何组合的木制木板（标记有`minecraft:planks`的物品）。标签通常通过前缀`#`与“常规”对象区分开来（例如`#minecraft:planks`，而`minecraft:oak_planks`）。

任何[**注册表**][registry]都可以有标签文件 - 虽然方块和物品是最常见的用例，但其他注册表如流体、实体类型或伤害类型也经常使用标签。如果需要，您也可以创建自己的标签。

标签位于`data/<标签命名空间>/tags/<注册表路径>/<标签路径>.json`（对于 Minecraft 注册表），或`data/<标签命名空间>/tags/<注册表命名空间>/<注册表路径>/<标签路径>.json`（对于非 Minecraft 注册表）。例如，要修改`minecraft:planks`物品标签，您应将标签文件放在`data/minecraft/tags/item/planks.json`。

:::info
与大多数其他 NeoForge 数据文件不同，NeoForge 添加的标签通常不使用`neoforge`命名空间。相反，它们使用`c`命名空间（例如`c:ingots/gold`）。这是因为根据许多在多加载器上开发的模组作者的要求，标签在 NeoForge 和 Fabric 模组加载器之间是统一的。

此规则有一些例外情况，特别是与 NeoForge 系统紧密集成的标签。例如，许多[伤害类型][damagetype]标签。
:::

覆盖标签文件通常是**添加式**(`additive`)而非替换式。这意味着如果两个数据包指定了相同 ID 的标签文件，则两个文件的内容将被合并（除非另有指定）。这种行为使标签与大多数其他数据文件区分开来，后者通常会替换所有现有值。

## 标签文件格式(`Tag File Format`)

标签文件具有以下语法：

```json5
{
    // 标签的值。
    "values": [
        // 一个值对象。必须指定要添加对象的 ID 以及是否必需。
        // 如果条目是必需的但对象不存在，则标签不会加载。"required"字段
        // 在技术上是可选的，但如果省略，该条目等同于下面的简写形式。
        {
            "id": "examplemod:example_ingot",
            "required": false
        }
        // {"id": "minecraft:gold_ingot", "required": true} 的简写形式，即必需条目。
        "minecraft:gold_ingot",
        // 标签对象。通过前导 # 与常规条目区分。在这种情况下，所有木板
        // 将被视为标签的条目。与常规条目一样，这也可以具有 "id"/"required" 格式。
        // 警告：循环标签依赖将导致数据包无法加载！
        "#minecraft:planks"
    ],
    // 是否在添加自己的条目之前删除所有预先存在的条目（true），或仅添加自己的条目（false）。
    // 通常应为 false，将此选项设置为 true 主要针对包开发者。
    "replace": false,
    // 一种更精细的移除标签中条目的方法（如果存在）。可选，NeoForge 添加。
    // 条目语法与 "values" 数组相同。
    "remove": [
        "minecraft:iron_ingot"
    ]
}
```

## 查找和命名标签(`Finding and Naming Tags`)

当您尝试查找现有标签时，通常建议遵循以下步骤：

- 查看 Minecraft 的标签，看看您要找的标签是否在那里。Minecraft 的标签可以在`BlockTags`、`ItemTags`、`EntityTypeTags`等中找到。
- 如果没有，请查看 NeoForge 的标签，看看您要找的标签是否在那里。NeoForge 的标签可以在`Tags.Blocks`、`Tags.Items`、`Tags.EntityTypes`等中找到。
- 否则，假设该标签未在 Minecraft 或 NeoForge 中指定，因此您需要创建自己的标签。

创建自己的标签时，您应该问自己以下问题：

- 这会修改我的模组的行为吗？如果是，标签应在您的模组命名空间中。（这常见于例如“我的东西可以在此方块上生成”类标签。）
- 其他模组是否也想使用此标签？如果是，标签应在`c`命名空间中。（这常见于例如新金属或宝石。）
- 否则，请使用您的模组命名空间。

命名标签本身也有一些约定要遵循：

- 使用复数形式。例如：`minecraft:planks`、`c:ingots`。
- 对同一类型的多个对象使用文件夹，并为每个文件夹设置一个整体标签。例如：`c:ingots/iron`、`c:ingots/gold`，以及包含两者的`c:ingots`。（注意：这是 NeoForge 约定，Minecraft 对大多数标签不遵循此约定。）

## 使用标签(`Using Tags`)

要在代码中引用标签，您必须创建一个`TagKey<T>`，其中`T`是标签的类型（`Block`、`Item`、`EntityType<?>`等），使用[**注册表键**][regkey]和[**资源位置**][resloc]：

```java
public static final TagKey<Block> MY_TAG = TagKey.create(
        // 注册表键。注册表的类型必须与标签的泛型类型匹配。
        Registries.BLOCK,
        // 标签的位置。此示例将我们的标签放在 data/examplemod/tags/blocks/example_tag.json。
        ResourceLocation.fromNamespaceAndPath("examplemod", "example_tag")
);
```

:::warning
由于`TagKey`是一个记录(`record`)，其构造函数是公共的。但是，不应直接使用构造函数，因为这样做可能导致各种问题，例如在查找标签条目时。
:::

然后我们可以使用标签对其执行各种操作。让我们从最明显的一个开始：检查对象是否在标签中。以下示例将假设方块标签，但功能对于每种类型的标签都是完全相同的（除非另有说明）：

```java
// 检查泥土是否在我们的标签中。
// 假设可以访问 Level level
boolean isInTag = level.registryAccess().lookupOrThrow(BuiltInRegistries.BLOCK).getOrThrow(MY_TAG).stream().anyMatch(holder -> holder.is(Items.DIRT));
```

由于这是一个非常冗长的语句，尤其是在经常使用时，`BlockState`和`ItemStack` - 标签系统的两个最常用者 - 各自定义了一个`#is`辅助方法，使用如下：

```java
// 检查 blockState 的方块是否在我们的标签中。
boolean isInBlockTag = blockState.is(MY_TAG);
// 检查 itemStack 的物品是否在我们的标签中。假设存在作为 TagKey<Item> 的 MY_ITEM_TAG。
boolean isInItemTag = itemStack.is(MY_ITEM_TAG);
```

如果需要，我们也可以获取标签条目的集合进行流式处理，如下所示：

```java
// 假设可以访问 Level level
Stream<Holder<Block>> blocksInTag = level.registryAccess().lookupOrThrow(BuiltInRegistries.BLOCK).getOrThrow(MY_TAG).stream();
```

### 引导期间用于静态注册表的标签(`Tags for Static Registries During Bootstrap`)

有时，您需要在注册过程中访问`HolderSet`。在这种情况下，**仅**对于静态注册表，您可以通过`BuiltInRegistries#acquireBootstrapRegistrationLookup`获取必要的`HolderGetter`：

```java
// 假设可以访问 Level level
HolderSet<Block> blockTag = BuiltInRegistries.acquireBootstrapRegistrationLookup(BuiltInRegistries.BLOCK).getOrThrow(MY_TAG);
```

## 数据生成(`Datagen`)

与许多其他 JSON 文件一样，标签可以进行[**数据生成**][datagen]。每种标签都有自己的数据生成基类 - 一个用于方块标签，一个用于物品标签等 - 因此，我们每种标签也需要一个类。所有这些类都从`TagsProvider<T>`基类扩展而来，其中`T`又是标签的类型（`Block`、`Item`等）。`TagsProvider`进一步分为两类：`IntrinsicHolderTagsProvider<T>`用于通常的静态注册表对象，允许您直接将对象传递给标签；`KeyTagProvider`用于通常的数据包注册表对象，允许您传递对象的`ResourceKey`给标签。

下表显示了不同对象的标签提供器列表：

| 类型                       | 标签提供器类                     | 提供器类型                 |
|----------------------------|----------------------------------------|-------------------------------|
| `BannerPattern`            | `BannerPatternTagsProvider`            | `KeyTagProvider`              |
| `Biome`                    | `BiomeTagsProvider`                    | `KeyTagProvider`              |
| `Block`                    | `BlockTagsProvider`\*                  | `IntrinsicHolderTagsProvider` |
| `DamageType`               | `DamageTypeTagsProvider`               | `KeyTagProvider`              |
| `Dialog`                   | `DialogTagsProvider`                   | `KeyTagProvider`              |
| `Enchantment`              | `EnchantmentTagsProvider`              | `KeyTagProvider`              |
| `EntityType`               | `EntityTypeTagsProvider`               | `IntrinsicHolderTagsProvider` |
| `FlatLevelGeneratorPreset` | `FlatLevelGeneratorPresetTagsProvider` | `IntrinsicHolderTagsProvider` |
| `Fluid`                    | `FluidTagsProvider`                    | `KeyTagProvider`              |
| `GameEvent`                | `GameEventTagsProvider`                | `KeyTagProvider`              |
| `Instrument`               | `InstrumentTagsProvider`               | `KeyTagProvider`              |
| `Item`                     | `ItemTagsProvider`\*                   | `IntrinsicHolderTagsProvider` |
| `PaintingVariant`          | `PaintingVariantTagsProvider`          | `KeyTagProvider`              |
| `PoiType`                  | `PoiTypeTagsProvider`                  | `KeyTagProvider`              |
| `Structure`                | `StructureTagsProvider`                | `KeyTagProvider`              |
| `WorldPreset`              | `WorldPresetTagsProvider`              | `KeyTagProvider`              |

\* 这些提供器由 NeoForge 提供。

为了示例，假设我们要生成方块标签（一个**内在持有者**）：

```java
public class MyBlockTagsProvider extends BlockTagsProvider {
    // 从其中一个 `GatherDataEvent` 获取参数。
    public MyBlockTagsProvider(PackOutput output, CompletableFuture<HolderLookup.Provider> lookupProvider) {
        super(output, lookupProvider, ExampleMod.MOD_ID);
    }

    // 在此添加您的标签条目。
    @Override
    protected void addTags(HolderLookup.Provider lookupProvider) {
        // 为我们的标签创建一个注册表对象的 TagAppender。这也可能是例如原版或 NeoForge 标签。
        this.tag(MY_TAG)
            // 添加条目。这是一个可变参数。
            // 键标签提供器必须在此提供 ResourceKey 而不是实际对象。
            .add(Blocks.DIRT, Blocks.COBBLESTONE)
            // 添加如果不存在将被忽略的可选条目。此示例使用植物魔法的 Pure Daisy。
            // 这不是可变参数。
            .add(TagEntry.optionalElement(ResourceLocation.fromNamespaceAndPath("botania", "pure_daisy")))
            // 添加一个标签条目。
            .addTag(BlockTags.PLANKS)
            // 添加多个标签条目。这是一个可变参数。
            // 可能导致可以安全抑制的未检查警告。
            .addTags(BlockTags.LOGS, BlockTags.WOODEN_SLABS)
            // 添加一个如果不存在将被忽略的可选标签条目。
            .addOptionalTag(ItemTags.create(ResourceLocation.fromNamespaceAndPath("c", "ingots/tin")))
            // 添加多个可选标签条目。这是一个可变参数。
            // 可能导致可以安全抑制的未检查警告。
            .addOptionalTags(ItemTags.create(ResourceLocation.fromNamespaceAndPath("c", "nuggets/tin")), ItemTags.create(ResourceLocation.fromNamespaceAndPath("c", "storage_blocks/tin")))
            // 将 replace 属性设置为 true。
            .replace()
            // 将 replace 属性设置回 false。
            .replace(false)
            // 移除条目。这是一个可变参数。
            // 键标签提供器必须在此提供 ResourceKey 而不是实际对象。
            // 可能导致可以安全抑制的未检查警告。
            .remove(Blocks.CRIMSON_SLAB, Blocks.WARPED_SLAB);
    }
}
```

此示例生成以下标签 JSON：

```json5
{
    "values": [
        "minecraft:dirt",
        "minecraft:cobblestone",
        {
            "id": "botania:pure_daisy",
            "required": false
        },
        "#minecraft:planks",
        "#minecraft:logs",
        "#minecraft:wooden_slabs",
        {
            "id": "c:ingots/tin",
            "required": false
        },
        {
            "id": "c:nuggets/tin",
            "required": false
        },
        {
            "id": "c:storage_blocks/tin",
            "required": false
        }
    ],
    "remove": [
        "minecraft:crimson_slab",
        "minecraft:warped_slab"
    ]
}
```

与所有数据提供器一样，将每个标签提供器添加到`GatherDataEvent`：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    // 如果添加数据包对象，首先调用 event.createDatapackRegistryObjects(...)

    event.createProvider(MyBlockTagsProvider::new);
}
```

### 自定义标签提供器(`Custom Tag Providers`)

可以通过简单地扩展`TagsProvider<T>`来创建自定义标签提供器，无论是用于现有还是自定义[**注册表**][registry]，其中`T`是您为其生成标签的注册表对象。

```java
public class MyRecipeTypeTagsProvider extends TagsProvider<RecipeType<?>> {
    // 从 `GatherDataEvent` 获取参数。
    public MyRecipeTypeTagsProvider(PackOutput output, CompletableFuture<HolderLookup.Provider> lookupProvider) {
        // 第二个参数是我们为其生成标签的注册表键。
        super(output, Registries.RECIPE_TYPE, lookupProvider, ExampleMod.MOD_ID);
    }
    
    @Override
    protected void addTags(HolderLookup.Provider lookupProvider) { /*...*/ }
}
```

从这里开始，通过创建`TagBuilder`（通过`getOrCreateRawBuilder`）从提供器生成标签。构建器包含通过`ResourceLocation`添加或移除元素和标签的方法。此外，构建器可以指定`replace`属性：

```java
public class MyRecipeTypeTagsProvider extends TagsProvider<RecipeType<?>> {
    
    // ...

    // 假设有以下 TagKey<RecipeType<?>>：SMELTERS, CRAFTERS, SMITHERS
    @Override
    protected void addTags(HolderLookup.Provider lookupProvider) {
        // 为 `ResourceLocation` 创建一个 TagBuilder。
        this.getOrCreateRawBuilder(MY_TAG)
            // 添加条目。
            .addElement(ResourceLocation.fromNamespaceAndPath("minecraft", "crafting"))
            .addElement(ResourceLocation.fromNamespaceAndPath("minecraft", "smelting"))
            // 添加如果不存在将被忽略的可选条目。
            .addOptionalElement(ResourceLocation.fromNamespaceAndPath("minecraft", "blasting"))
            // 添加一个标签条目。
            .addTag(SMELTERS.location())
            // 添加一个如果不存在将被忽略的可选标签条目。
            .addOptionalTag(CRAFTERS.location())
            // 将 replace 属性设置为 true。
            .replace()
            // 将 replace 属性设置回 false。
            .replace(false)
            // 移除条目。
            .removeElement(ResourceLocation.fromNamespaceAndPath("minecraft", "campfire_cooking"))
            // 移除一个标签条目。
            .removeTag(SMITHERS.location());
    }
}
```

目前，整个标签是从`ResourceLocation`构造的。然而，每次指定原始标识符可能变得繁琐，特别是当`ResourceKey`或直接对象可用时。这就是`TagAppender`的用武之地。`TagAppender<E, T>`在功能上是`TagBuilder`的包装器，它接受某个任意条目对象`E`并将其转换为注册表对象`T`的`TagBuilder`调用。`TagAppender`可以通过`map`重新映射到任何任意对象，前提是有办法将新对象类型转换为先前的条目对象`E`。这基本上是`KeyTagProvider`和`IntrinsicHolderTagsProvider`所做的。它们提供了一个方法`tag`，创建一个`TagAppender`，分别将`ResourceKey`映射到`ResourceLocation`或将直接对象映射到`ResourceLocation`：

```java

public class MyRecipeTypeTagsProvider extends TagsProvider<RecipeType<?>> {
    
    // ...

    // 假设我们有 TagKey<RecipeType<?>>：SMELTERS, CRAFTERS, SMITHERS
    @Override
    protected void addTags(HolderLookup.Provider lookupProvider) {
        // 为 `ResourceLocation` 创建 TagAppender。
        this.tag(MY_TAG)
            // 替换属性信息
            .replace()
            // 处理可能不存在的任何可选元素
            .addOptional(ResourceLocation.fromNamespaceAndPath("examplemod", "example_type"))
            // 可以接受 TagKey
            .addOptionalTag(CRAFTERS)

            // 映射到 ResourceKey (KeyTagProvider)
            .map((Function<ResourceKey<RecipeType<?>>, ResourceLocation>) ResourceKey::location)
            .add(BuiltInRegistries.RECIPE_TYPE.getResourceKey(RecipeType.CRAFTING).orElseThrow())

            // 映射到直接对象 (IntrinsicHolderTagsProvider)
            .map((Function<RecipeType<?>, ResourceKey<RecipeType<?>>) type -> BuiltInRegistries.RECIPE_TYPE.getResourceKey(type).orElseThrow())
            .add(RecipeType.SMELTING)
            .addTag(SMELTERS)
            .remove(RecipeType.CAMPFIRE_COOKING)
            .remove(SMITHERS);
    }

    private TagAppender<ResourceLocation, RecipeType<?>> tag(TagKey<RecipeType<?>> tag) {
        // 创建构建器
        TagBuilder builder = this.getOrCreateRawBuilder(tag);

        // 生成 appender（可以使用 TagAppender#forBuilder 代替）
        return new TagAppender<ResourceLocation, T>() {

            @Override
            public TagAppender<ResourceLocation, T> add(ResourceLocation element) {
                builder.addElement(element);
                return this;
            }

            @Override
            public TagAppender<ResourceLocation, T> addOptional(ResourceLocation element) {
                builder.addOptionalElement(element);
                return this;
            }

            @Override
            public TagAppender<ResourceLocation, T> addTag(TagKey<T> tag) {
                builder.addTag(tag.location());
                return this;
            }

            @Override
            public TagAppender<ResourceLocation, T> addOptionalTag(TagKey<T> tag) {
                builder.addOptionalTag(tag.location());
                return this;
            }

            // 对于无法访问当前条目对象的情况
            @Override
            public TagAppender<ResourceLocation, T> add(TagEntry entry) {
                builder.add(entry);
                return this;
            }

            @Override
            public TagAppender<ResourceLocation, T> replace(boolean value) {
                builder.replace(value);
                return this;
            }

            @Override
            public TagAppender<ResourceLocation, T> remove(ResourceLocation element) {
                builder.removeElement(element);
                return this;
            }

            @Override
            public TagAppender<ResourceKey<T>, T> remove(TagKey<T> tag) {
                builder.removeTag(tag.location());
                return this;
            }
        };
    }
}
```

#### 复制标签内容(`Copying Tag Contents`)

NeoForge 提供了一种特殊类型的`IntrinsicHolderTagsProvider`，称为`BlockTagCopyingItemTagProvider`，用于镜像其关联方块标签内容的物品标签。不使用`TagAppender`，而是调用`copy`，将方块标签复制到物品标签。

```java
public class ExampleBlockTagCopyingItemTagProvider extends BlockTagCopyingItemTagProvider {

    public ExampleBlockTagCopyingItemTagProvider(
        PackOutput output,
        CompletableFuture<HolderLookup.Provider> lookupProvider,
        CompletableFuture<TagLookup<Block>> blockTags // 从 BlockTagsProvider#contentsGetter 获取
    ) {
        super(output, lookupProvider, blockTags, ExampleMod.MOD_ID);
    }

    @Override
    protected void addTags(HolderLookup.Provider lookupProvider) {
        // 假设两个参数的类型为 TagKey<Block> 和 TagKey<Item>
        this.copy(EXAMPLE_BLOCK_TAG, EXAMPLE_ITEM_TAG);
    }

}
```

[damagetype]: damagetypes.md
[datagen]: ../index.md#data-generation
[registry]: ../../concepts/registries.md
[regkey]: ../../misc/resourcelocation.md#resourcekeys
[resloc]: ../../misc/resourcelocation.md