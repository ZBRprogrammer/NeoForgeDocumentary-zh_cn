# 配方

**配方**(`Recipes`)是一种在Minecraft世界中将一组对象转换为其他对象的方式。尽管Minecraft仅将此系统用于物品转换，但该系统的设计允许对任何类型的对象（如方块、实体等）进行转换。几乎所有配方都使用配方数据文件；除非另有明确说明，本文中提到的“配方”均指数据驱动的配方。

配方数据文件位于`data/<namespace>/recipe/<path>.json`。例如，配方`minecraft:diamond_block`位于`data/minecraft/recipe/diamond_block.json`。

## 术语

- **配方JSON**(`recipe JSON`)或**配方文件**(`recipe file`)是由`RecipeManager`加载和存储的JSON文件。它包含配方类型、输入和输出以及其他附加信息（如处理时间）等信息。
- **`Recipe`** 保存了所有JSON字段的代码表示，以及匹配逻辑（“此输入是否与配方匹配？”）和其他一些属性。
- **`RecipeInput`** 是一种为配方提供输入的类型。它有几个子类，例如`CraftingInput`或`SingleRecipeInput`（用于熔炉等）。
- **配方材料**(`recipe ingredient`)或简称**材料**(`ingredient`)是配方的单个输入（而`RecipeInput`通常表示用于与配方材料进行匹配检查的一组输入）。材料是一个非常强大的系统，因此在[单独的文章][ingredients]中进行了详细介绍。
- **`PlacementInfo`** 定义了配方中包含的物品以及它们应填充的索引。如果根据提供的物品在某种程度上无法捕获配方（例如，仅更改数据组件），则使用`PlacementInfo#NOT_PLACEABLE`。
- **`SlotDisplay`** 定义了在配方查看器（如配方书）中单个插槽应如何显示。
- **`RecipeDisplay`** 定义了配方的`SlotDisplay`，供配方查看器（如配方书）使用。虽然该接口仅包含用于获取配方结果和配方执行所在工作台的方法，但子类型可以捕获材料或网格大小等信息。
- **`RecipeManager`** 是服务器上的一个单例字段，用于保存所有已加载的配方。
- **`RecipeSerializer`** 基本上是`[`MapCodec`][codec]`和`[`StreamCodec`][streamcodec]`的包装器，两者都用于序列化。
- **`RecipeType`** 是`Recipe`的注册类型等效物。它主要用于按类型查找配方。一般来说，不同的 crafting 容器应该使用不同的`RecipeType`。例如，`minecraft:crafting`配方类型涵盖了`minecraft:crafting_shaped`和`minecraft:crafting_shapeless`配方序列化器，以及特殊的 crafting 序列化器。
- **`RecipeBookCategory`** 是在通过配方书查看配方时代表某些配方的组。
- **配方 [进度]**(`recipe [advancement]`)是负责在配方书中解锁配方的进度。它们不是必需的，玩家通常会忽略它们而选择使用配方查看器模组，但是[配方数据提供器][datagen]会为你生成这些进度，因此建议使用它们。
- **`RecipePropertySet`** 定义了菜单中定义的输入插槽可以接受的可用材料列表。
- **`RecipeBuilder`** 在数据生成期间用于创建JSON配方。
- **配方工厂**(`recipe factory`)是一个方法引用，用于从`RecipeBuilder`创建`Recipe`。它可以是构造函数的引用、静态构建方法或专门为此目的创建的函数式接口（通常命名为`Factory`）。

## JSON规范

配方文件的内容根据所选类型的不同而有很大差异。所有配方文件共有的属性是`type`和`[`neoforge:conditions`][conditions]`：

```json5
{
    // 配方类型。这对应于配方序列化器注册表中的一个条目。
    "type": "minecraft:crafting_shaped",
    // 数据加载条件列表。可选，由NeoForge添加。有关更多信息，请参阅上面链接的文章。
    "neoforge:conditions": [ /*...*/ ]
}
```

Minecraft提供的完整类型列表可以在[内置配方类型文章][builtin]中找到。模组也可以[定义自己的配方类型][customrecipes]。

## 使用配方

配方通过`RecipeManager`类加载、存储和获取，而`RecipeManager`类又可以通过`ServerLevel#recipeAccess`获取，或者如果你没有可用的`ServerLevel`，可以通过`ServerLifecycleHooks.getCurrentServer()#getRecipeManager`获取。默认情况下，服务器不会将配方同步到客户端，而是只发送`RecipePropertySet`以限制菜单插槽上的输入。此外，每当在配方书中解锁一个配方时，其`RecipeDisplay`和相应的`RecipeDisplayEntry`会被发送到客户端（不包括`Recipe#isSpecial`返回true的所有配方）。因此，配方逻辑应该始终在服务器上运行。

获取配方最简单的方法是通过其资源键：

```java
RecipeManager recipes = serverLevel.recipeAccess();
// RecipeHolder<?> 是资源键和配方本身的记录。
Optional<RecipeHolder<?>> optional = recipes.byKey(
    ResourceKey.create(Registries.RECIPE, ResourceLocation.withDefaultNamespace("diamond_block"))
);
optional.map(RecipeHolder::value).ifPresent(recipe -> {
    // 在这里对配方执行你想要的操作。请注意，配方可能是任何类型。
});
```

一种更实际适用的方法是构造一个`RecipeInput`并尝试获取匹配的配方。在这个例子中，我们将使用`CraftingInput#of`创建一个包含一个钻石块的`CraftingInput`。这将创建一个无形状的输入，如果是有形状的输入则使用`CraftingInput#ofPositioned`，其他输入则使用其他`RecipeInput`（例如，熔炉配方通常使用`new SingleRecipeInput`）。

```java
RecipeManager recipes = serverLevel.recipeAccess();
// 根据配方要求构造一个 RecipeInput。例如，为 crafting 配方构造一个 CraftingInput。
// 参数分别是宽度、高度和物品。
CraftingInput input = CraftingInput.of(1, 1, List.of(new ItemStack(Items.DIAMOND_BLOCK)));
// 配方持有者上的泛型通配符应该扩展 CraftingRecipe。
// 这在以后可以提供更多的类型安全。
Optional<RecipeHolder<? extends CraftingRecipe>> optional = recipes.getRecipeFor(
        // 要获取配方的配方类型。在我们的例子中，我们使用 crafting 类型。
        RecipeType.CRAFTING,
        // 我们的配方输入。
        input,
        // 我们的世界上下文。
        serverLevel
);
// 这将返回钻石块 -> 9 个钻石的配方（除非数据包更改了该配方）。
optional.map(RecipeHolder::value).ifPresent(recipe -> {
    // 在这里执行你想要的操作。请注意，现在配方是 CraftingRecipe 而不是 Recipe<?>。
});
```

或者，你也可以获取一个可能为空的与你的输入匹配的配方列表，这在可以合理假设多个配方匹配的情况下特别有用：

```java
RecipeManager recipes = serverLevel.recipeAccess();
CraftingInput input = CraftingInput.of(1, 1, List.of(new ItemStack(Items.DIAMOND_BLOCK)));
// 这些不是 Optionals，可以直接使用。但是，列表可能为空，表示没有匹配的配方。
Stream<RecipeHolder<? extends Recipe<CraftingInput>>> list = recipes.recipeMap().getRecipesFor(
    // 与上面相同的参数。
    RecipeType.CRAFTING, input, serverLevel
);
```

一旦我们有了正确的配方输入，我们还想获取配方输出。这可以通过调用`Recipe#assemble`来完成：

```java
RecipeManager recipes = serverLevel.recipeAccess();
CraftingInput input = CraftingInput.of(...);
Optional<RecipeHolder<? extends CraftingRecipe>> optional = recipes.getRecipeFor(...);
// 使用 ItemStack.EMPTY 作为后备选项。
ItemStack result = optional
        .map(RecipeHolder::value)
        .map(recipe -> recipe.assemble(input, serverLevel.registryAccess()))
        .orElse(ItemStack.EMPTY);
```

如果需要，还可以遍历某种类型的所有配方。具体做法如下：

```java
RecipeManager recipes = serverLevel.recipeAccess();
// 像之前一样，传递所需的配方类型。
Collection<RecipeHolder<?>> list = recipes.recipeMap().byType(RecipeType.CRAFTING);
```

## 其他配方机制

原版中的一些机制通常被视为配方，但在代码中的实现方式不同。这通常是由于遗留原因，或者因为“配方”是由其他数据（例如[标签]）构建而成的。

:::warning
配方查看器模组通常不会识别这些配方。必须手动添加对这些模组的支持，请参阅相应模组的文档以获取更多信息。
:::

### 铁砧配方

铁砧有两个输入槽和一个输出槽。原版中唯一的用例是工具修复、合并和重命名，由于每个用例都需要特殊处理，因此没有提供配方文件。但是，可以使用`AnvilUpdateEvent`来扩展这个系统。这个[事件]允许获取输入（左输入槽）和材料（右输入槽），并允许设置输出物品栈、经验成本和要消耗的材料数量。整个过程也可以通过[取消][cancel]事件来阻止。

```java
// 这个例子允许用一整组泥土修复一把石镐，消耗半组泥土，花费3级经验。
@SubscribeEvent // 在游戏事件总线上
public static void onAnvilUpdate(AnvilUpdateEvent event) {
    ItemStack left = event.getLeft();
    ItemStack right = event.getRight();
    if (left.is(Items.STONE_PICKAXE) && right.is(Items.DIRT) && right.getCount() >= 64) {
        event.setOutput(new ItemStack(Items.STONE_PICKAXE));
        event.setMaterialCost(32);
        event.setXpCost(3);
    }
}
```

### 酿造

请参阅[药水与效果文章中的酿造章节][brewing]。

### 扩展 crafting 网格大小

`ShapedRecipePattern`类负责保存有形状的 crafting 配方的内存表示，它对插槽大小有一个硬编码的限制，为3x3，这阻碍了想要添加更大 crafting 台同时重用原版有形状的 crafting 配方类型的模组。为了解决这个问题，NeoForge 插入了一个名为`ShapedRecipePattern#setCraftingSize(int width, int height)`的静态方法，允许增加这个限制。该方法应该在`FMLCommonSetupEvent`期间调用。这里取最大值，例如，如果一个模组添加了一个4x6的 crafting 台，另一个模组添加了一个6x5的 crafting 台，那么最终的值将是6x6。

:::danger
`ShapedRecipePattern#setCraftingSize`不是线程安全的。它必须包装在`event#enqueueWork`调用中。
:::

### 客户端配方

默认情况下，原版不会将任何配方发送到[逻辑客户端][logicalside]。相反，会同步`RecipePropertySet` / `SelectableRecipe.SingleInputSet`以处理用户交互期间的正确客户端行为。此外，当在配方书中解锁一个配方时，会同步其`RecipeDisplay`。然而，这两种情况的范围有限，特别是当需要从配方本身获取更多数据时。在这些情况下，NeoForge 提供了一种将给定`RecipeType`的完整配方发送到客户端的方法。

在[游戏事件总线][events]上必须监听两个事件：`OnDatapackSyncEvent`和`RecipesReceivedEvent`。首先，通过调用`OnDatapackSyncEvent#sendRecipes`指定要同步到客户端的`RecipeType`。然后，可以通过`RecipesReceivedEvent#getRecipeMap`从提供的`RecipeMap`中访问配方。此外，玩家退出世界时，应该通过`ClientPlayerNetworkEvent.LoggingOut`清除客户端上存储的所有配方。

```java
// 假设我们有一些自定义的 RecipeType<ExampleRecipe> EXAMPLE_RECIPE_TYPE

@SubscribeEvent // 在游戏事件总线上
public static void datapackSync(OnDatapackSyncEvent event) {
    // 指定要同步到客户端的配方类型
    event.sendRecipes(EXAMPLE_RECIPE_TYPE);
}

// 在仅存在于物理客户端的某个类中

private static final List<RecipeHolder<ExampleRecipe>> EXAMPLE_RECIPES = new ArrayList<>();

@SubscribeEvent // 仅在物理客户端的游戏事件总线上
public static void recipesReceived(RecipesReceivedEvent event) {
    // 首先移除之前的配方
    EXAMPLE_RECIPES.clear();

    // 然后存储你想要的配方
    EXAMPLE_RECIPES.addAll(event.getRecipeMap().byType(EXAMPLE_RECIPE_TYPE));
}

@SubscribeEvent // 仅在物理客户端的游戏事件总线上
public static void clientLogOut(ClientPlayerNetworkEvent.LoggingOut event) {
    // 在退出世界时清除存储的配方
    EXAMPLE_RECIPES.clear();
}
```

:::warning
如果你打算同步你的配方类型的配方，`OnDatapackSyncEvent`应该在两个物理端都调用。所有世界，包括单人游戏，都有服务器和客户端的划分，这意味着在客户端引用服务器上的数据包注册表条目可能会导致游戏崩溃。
:::

## 数据生成

与大多数其他JSON文件一样，配方可以通过数据生成来创建。对于配方，我们需要扩展`RecipeProvider`类并覆盖`#buildRecipes`，并扩展`RecipeProvider.Runner`类以传递给数据生成器：

```java
public class MyRecipeProvider extends RecipeProvider {

    // 构造要运行的提供器
    protected MyRecipeProvider(HolderLookup.Provider provider, RecipeOutput output) {
        super(provider, output);
    }
 
    @Override
    protected void buildRecipes() {
        // 在这里添加你的配方。
    }

    // 要添加到数据生成器的运行器
    public static class Runner extends RecipeProvider.Runner {
        // 从 `GatherDataEvent` 中获取参数。
        public Runner(PackOutput output, CompletableFuture<HolderLookup.Provider> lookupProvider) {
            super(output, lookupProvider);
        }

        @Override
        protected RecipeProvider createRecipeProvider(HolderLookup.Provider provider, RecipeOutput output) {
            return new MyRecipeProvider(provider, output);
        }
    }
}
```

需要注意的是`RecipeOutput`参数。Minecraft使用这个对象自动为你生成配方进度。此外，NeoForge将[条件]支持注入到`RecipeOutput`中，可以通过`#withConditions`调用。

配方本身通常通过`RecipeBuilder`的子类添加。列出所有原版配方构建器超出了本文的范围（它们在[内置配方类型文章][builtin]中进行了解释），但是创建自己的构建器在[自定义配方页面][customdatagen]中有解释。

与所有其他数据提供器一样，配方提供器必须像这样注册到`GatherDataEvent`中：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    // 如果要添加数据包对象，请先调用 event.createDatapackRegistryObjects(...)

    event.createProvider(MyRecipeProvider.Runner::new);
}
```

配方提供器还为常见场景添加了辅助方法，例如`twoByTwoPacker`（用于2x2方块配方）、`threeByThreePacker`（用于3x3方块配方）或`nineBlockStorageRecipes`（用于3x3方块配方和1个方块到9个物品的配方）。

[advancement]: ../advancements.md
[brewing]: ../../../items/mobeffects.md#brewing
[builtin]: builtin.md
[cancel]: ../../../concepts/events.md#cancellable-events
[codec]: ../../../datastorage/codecs.md
[conditions]: ../conditions.md
[customdatagen]: custom.md#data-generation
[customrecipes]: custom.md
[datagen]: #data-generation
[event]: ../../../concepts/events.md
[ingredients]: ingredients.md
[logicalside]: ../../../concepts/sides.md#the-logical-side
[streamcodec]: ../../../networking/streamcodecs.md
[tags]: ../tags.md