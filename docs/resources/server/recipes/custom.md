# **自定义配方**(`Custom Recipes`)

要添加自定义配方，我们至少需要三样东西：一个**配方**(`Recipe`)、一个**配方类型**(`RecipeType`)和一个**配方序列化器**(`RecipeSerializer`)。根据你要实现的内容，如果你无法复用现有的子类，可能还需要一个自定义的**配方输入**(`RecipeInput`)、**配方显示**(`RecipeDisplay`)、**槽位显示**(`SlotDisplay`)、**配方书类别**(`RecipeBookCategory`)和**配方属性集**(`RecipePropertySet`)。

为了举例并突出许多不同的特性，我们将实现一个由配方驱动的机制，该机制要求你在游戏世界中用某个特定物品右键点击一个**方块状态**(`BlockState`)，破坏该**方块状态**并掉落结果物品。

## **配方输入**(`Recipe Input`)

让我们从定义我们想放入配方的内容开始。重要的是要理解，配方输入代表玩家当前正在使用的实际输入。因此，我们在这里不使用标签或成分，而是使用我们可用的实际物品堆和方块状态。

```java
// 我们的输入是一个方块状态和一个物品堆。
public record RightClickBlockInput(BlockState state, ItemStack stack) implements RecipeInput {
    // 从特定槽位获取物品的方法。我们只有一个物品堆，没有槽位的概念，所以我们假设
    // 槽位 0 持有我们的物品，对于其他任何槽位则抛出异常。（取自 SingleRecipeInput#getItem。）
    @Override
    public ItemStack getItem(int slot) {
        if (slot != 0) throw new IllegalArgumentException("No item for index " + slot);
        return this.stack();
    }

    // 我们的输入所需的槽位大小。同样，我们实际上没有槽位的概念，所以我们只返回 1
    // 因为我们只涉及一个物品堆。包含多个物品的输入应该在这里返回实际数量。
    @Override
    public int size() {
        return 1;
    }
}
```

配方输入不需要以任何方式进行注册或序列化，因为它们是按需创建的。并不总是需要创建自己的配方输入，原版的那些（`CraftingInput`、`SingleRecipeInput` 和 `SmithingRecipeInput`）在许多用例中就足够了。

此外，NeoForge 提供了 `RecipeWrapper` 输入，它会根据构造函数中传入的 `IItemHandler` 包装 `#getItem` 和 `#size` 调用。基本上，这意味着任何基于网格的库存，如箱子，都可以通过包装在 `RecipeWrapper` 中用作配方输入。

## **配方类**(`Recipe Class`)

现在我们有了输入，让我们来看看配方本身。这是保存我们配方数据的地方，同时也处理匹配和返回配方结果。因此，对于你的自定义配方，它通常是最长的类。

```java
// Recipe<T> 的泛型参数是我们上面定义的 RightClickBlockInput。
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 我们配方数据的代码表示。这基本上可以是你想要的任何东西。
    // 这里常见的内容是某种处理时间整数，或者经验奖励。
    // 注意，我们现在对输入使用成分而不是物品堆。
    private final BlockState inputState;
    private final Ingredient inputItem;
    private final ItemStack result;

    // 添加一个设置所有属性的构造函数。 
    public RightClickBlockRecipe(BlockState inputState, Ingredient inputItem, ItemStack result) {
        this.inputState = inputState;
        this.inputItem = inputItem;
        this.result = result;
    }

    // 检查给定的输入是否匹配此配方。第一个参数与泛型匹配。
    // 我们检查我们的方块状态和物品堆，只有当两者都匹配时才返回 true。
    // 如果我们需要检查输入的维度，我们也会在这里进行检查。
    @Override
    public boolean matches(RightClickBlockInput input, Level level) {
        return this.inputState == input.state() && this.inputItem.test(input.stack());
    }

    // 根据给定的输入在这里返回配方的结果。第一个参数与泛型匹配。
    // 重要提示：如果你使用现有的结果，一定要调用 .copy()！如果你不这样做，事情可能会出错，
    // 因为结果每个配方只存在一次，但每次制作配方时都会创建组装好的物品堆。
    @Override
    public ItemStack assemble(RightClickBlockInput input, HolderLookup.Provider registries) {
        return this.result.copy();
    }

    // 当为 true 时，将阻止配方在配方书中同步或在使用/解锁时获得奖励。
    // 只有当配方不应该出现在配方书中时，才应该为 true，例如地图扩展配方。
    // 尽管这个配方接受一个输入状态，但它仍然可以使用下面的方法用于自定义配方书。
    @Override
    public boolean isSpecial() {
        return true;
    }

    // 这个例子概述了最重要的方法。还有许多其他方法需要重写。
    // 有些方法将在下面的部分进行解释，因为它们不能在这里简单地压缩和理解。
    // 查看 Recipe 类的定义以查看所有方法。
}
```

## **配方书类别**(`Recipe Book Categories`)

一个**配方书类别**(`RecipeBookCategory`)只是定义了一个在配方书中显示此配方的组。例如，铁镐镐的制作配方会显示在 `RecipeBookCategories#CRAFTING_EQUIPMENT` 中，而熟鳕鳕鱼的配方会显示在 `#FURNANCE_FOOD` 或 `#SMOKER_FOOD` 中。每个配方都有一个关联的**配方书类别**。原版的类别可以在 `RecipeBookCategories` 中找到。

:::note
有两个熟鳕鳕鱼配方，一个是熔炉的，一个是烟熏炉的。熔炉和烟熏炉的配方有不同的配方书类别。
:::

如果你的配方不适合现有的任何类别，通常是因为该配方不使用现有的任何制作站（例如，工作台、熔炉），那么可以创建一个新的**配方书类别**。每个**配方书类别**都必须[注册][registry]到 `BuiltInRegistries#RECIPE_BOOK_CATEGORY`：

```java
/// 对于某个 DeferredRegister<RecipeBookCategory> RECIPE_BOOK_CATEGORIES
public static final Supplier<RecipeBookCategory> RIGHT_CLICK_BLOCK_CATEGORY = RECIPE_BOOK_CATEGORIES.register(
    "right_click_block", RecipeBookCategory::new
);
```

然后，要设置类别，我们必须像这样重写 `#recipeBookCategory`：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 这里是其他内容

    @Override
    public RecipeBookCategory recipeBookCategory() {
        return RIGHT_CLICK_BLOCK_CATEGORY.get();
    }
}
```

### **搜索类别**(`Search Categories`)

所有的**配方书类别**在技术上都是**扩展配方书类别**(`ExtendedRecipeBookCategory`)。还有一种**扩展配方书类别**叫做**搜索配方书类别**(`SearchRecipeBookCategory`)，它用于在查看配方书中的所有配方时聚合**配方书类别**。

NeoForge 允许用户通过模组事件总线上的 `RegisterRecipeBookSearchCategoriesEvent#register` 指定自己的**扩展配方书类别**作为搜索类别。`register` 接受代表搜索类别的**扩展配方书类别**和组成该搜索类别的**配方书类别**。**扩展配方书类别**搜索类别不需要注册到某个静态的原版注册表中。

```java
// 在某个位置
public static final ExtendedRecipeBookCategory RIGHT_CLICK_BLOCK_SEARCH_CATEGORY = new ExtendedRecipeBookCategory() {};

@SubscribeEvent // 在模组事件总线上
public static void registerSearchCategories(RegisterRecipeBookSearchCategoriesEvent event) {
    event.register(
        // 搜索类别
        RIGHT_CLICK_BLOCK_SEARCH_CATEGORY,
        // 搜索类别内的所有配方类别，作为可变参数
        RIGHT_CLICK_BLOCK_CATEGORY.get()
    )
}
```

## **放置信息**(`Placement Info`)

一个**放置信息**(`PlacementInfo`)用于定义配方使用者使用的制作要求，以及它是否可以和如何放置到其关联的制作站（例如，工作台、熔炉）中。**放置信息** 仅用于物品成分，因此如果需要其他类型的成分（例如，流体、方块），则需要从头开始实现周围的逻辑。在这些情况下，配方可以标记为不可放置，并通过 `PlacementInfo#NOT_PLACEABLE` 进行说明。但是，如果你的配方中至少有一个类似物品的对象，你应该创建一个**放置信息**。

可以通过 `create` 创建一个**放置信息**，它接受一个或一个成分列表，或者通过 `createFromOptionals` 创建，它接受一个可选成分列表。如果你的配方包含空槽位的表示，则应该使用 `createFromOptionals`，为空槽位提供一个空的可选对象：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 这里是其他内容
    private PlacementInfo info;

    @Override
    public PlacementInfo placementInfo() {
        // 这个委托是以防成分在这个时间点还没有完全填充
        // 标签和配方是同时加载的，这就是为什么可能会出现这种情况。
        if (this.info == null) {
            // 使用可选成分，因为方块状态可能有物品表示
            List<Optional<Ingredient>> ingredients = new ArrayList<>();
            Item stateItem = this.inputState.getBlock().asItem();
            ingredients.add(stateItem != Items.AIR ? Optional.of(Ingredient.of(stateItem)): Optional.empty());
            ingredients.add(Optional.of(this.inputItem));

            // 创建放置信息
            this.info = PlacementInfo.createFromOptionals(ingredients);
        }

        return this.info;
    }
}
```

## **槽位显示**(`Slot Displays`)

**槽位显示**(`SlotDisplay`)表示当配方使用者（如配方书）查看时，每个槽位应该渲染什么信息。一个**槽位显示**有两个方法。首先是 `resolve`，它接受一个包含可用注册表和燃料值的 **上下文映射**(`ContextMap`)（如 `SlotDisplayContext` 所示）；以及当前的 **显示内容工厂**(`DisplayContentsFactory`)，它接受要为此槽位显示的内容；并返回转换后的内容列表作为要接受的输出。然后是 `type`，它包含用于编码/解码显示的[**映射编解码器**(`MapCodec`)][codec] 和 [**流编解码器**(`StreamCodec`)][streamcodec]。

**槽位显示** 通常在[`Ingredient` 通过 `#display` 实现，或者对于模组成分通过 `ICustomIngredient#display` 实现][ingredients]；然而，在某些情况下，输入可能不是一个成分，这意味着**槽位显示** 将需要使用一个可用的，或者创建一个新的。

以下是原版和 NeoForge 提供的可用槽位显示：

- `SlotDisplay.Empty`：表示空的槽位。
- `SlotDisplay.ItemSlotDisplay`：表示一个物品的槽位。
- `SlotDisplay.ItemStackSlotDisplay`：表示一个物品堆的槽位。
- `SlotDisplay.TagSlotDisplay`：表示一个物品标签的槽位。
- `SlotDisplay.WithRemainder`：表示有制作剩余物的某种输入的槽位。
- `SlotDisplay.AnyFuel`：表示所有燃料物品的槽位。
- `SlotDisplay.Composite`：表示其他槽位显示组合的槽位。
- `SlotDisplay.SmithingTrimDemoSlotDisplay`：表示将随机锻造装饰应用于具有给定材料的某个基础物品的槽位。
- `FluidSlotDisplay`：表示一个流体的槽位。
- `FluidStackSlotDisplay`：表示一个流体堆的槽位。
- `FluidTagSlotDisplay`：表示一个流体标签的槽位。

我们的配方中有三个"槽位"：`BlockState` 输入、`Ingredient` 输入和 `ItemStack` 结果。`Ingredient` 输入已经有一个关联的**槽位显示**，而 `ItemStack` 可以用 `SlotDisplay.ItemStackSlotDisplay` 表示。另一方面，`BlockState` 需要它自己的自定义**槽位显示** 和 **显示内容工厂**，因为现有的只接受物品堆，并且在这个例子中，方块状态的处理方式不同。

从 **显示内容工厂** 开始，它旨在将某种类型转换为所需的内容显示类型。可用的工厂有：

- `DisplayContentsFactory.ForStacks`：一个接受`ItemStack` 的转换器。
- `DisplayContentsFactory.ForRemainders`：一个接受输入对象和剩余物对象列表的转换器。
- `DisplayContentsFactory.ForFluidStacks`：一个接受`FluidStack` 的转换器。

有了这些，就可以实现 **显示内容工厂** 来将提供的对象转换为所需的输出。例如，`SlotDisplay.ItemStackContentsFactory` 接受 `ForStacks` 转换器，并将物品堆转换为`ItemStack`。

对于我们的`BlockState`，我们将创建一个接受方块状态的工厂，以及一个输出方块状态本身的基本实现。

```java
// 方块状态的基本转换器
public interface ForBlockStates<T> extends DisplayContentsFactory<T> {

    // 委托方法
    default forState(Holder<Block> block) {
        return this.forState(block.value());
    }

    default forState(Block block) {
        return this.forState(block.defaultBlockState());
    }

    // 要接受并转换为所需输出的方块状态
    T forState(BlockState state);
}

// 方块状态输出的实现
public class BlockStateContentsFactory implements ForBlockStates<BlockState> {
    // 单例实例
    public static final BlockStateContentsFactory INSTANCE = new BlockStateContentsFactory();

    private BlockStateContentsFactory() {}

    @Override
    public BlockState forState(BlockState state) {
        return state;
    }
}

// 物品堆输出的实现
public class BlockStateStackContentsFactory implements ForBlockStates<ItemStack> {
    // 单例实例
    public static final BlockStateStackContentsFactory INSTANCE = new BlockStateStackContentsFactory();

    private BlockStateStackContentsFactory() {}

    @Override
    public ItemStack forState(BlockState state) {
        return new ItemStack(state.getBlock());
    }
}
```

然后，有了这些，我们可以创建一个新的**槽位显示**。`SlotDisplay.Type` 必须[注册][registry]：

```java
// 一个简单的槽位显示
public record BlockStateSlotDisplay(BlockState state) implements SlotDisplay {
    public static final MapCodec<BlockStateSlotDisplay> CODEC = BlockState.CODEC.fieldOf("state")
        .xmap(BlockStateSlotDisplay::new, BlockStateSlotDisplay::state);
    public static final StreamCodec<RegistryFriendlyByteBuf, BlockStateSlotDisplay> STREAM_CODEC =
        StreamCodec.composite(
            ByteBufCodecs.idMapper(Block.BLOCK_STATE_REGISTRY), BlockStateSlotDisplay::state,
            BlockStateSlotDisplay::new
        );
    
    @Override
    public <极光> Stream<T> resolve(ContextMap context, DisplayContentsFactory<T> factory) {
        return switch (factory) {
            // 检查我们的内容工厂并在必要时进行转换
            case ForBlockStates<T> states -> Stream.of(states.forState(this.state));
            // 如果你想根据内容显示以不同的方式处理内容
            // 那么你可以像这样对其他显示进行 case 操作
            case ForStacks<T> stacks -> Stream.of(stacks.forStack(state.getBlock().asItem()));
            // 如果没有匹配的工厂，则在转换后的流中不返回任何内容
            default -> Stream.empty();
        }
    }

    @Override
    public SlotDisplay.Type<? extends SlotDisplay> type() {
        // 返回下面注册的类型
        return BLOCK_STATE_S极光_DISPLAY.get();
    }
}

// 在某个注册类中
/// 对于某个 DeferredRegister<SlotDisplay.Type<?>> SLOT_DISPLAY_TYPES
public static final Supplier<SlotDisplay.Type<BlockStateSlotDisplay>> BLOCK_STATE_SLOT_DISPLAY = SLOT_DISPLAY_TYPES.register(
    "block_state",
    () -> new SlotDisplay.Type<>(BlockStateSlotDisplay.CODEC, BlockStateSlotDisplay.STREAM_CODEC)
);
```

## **配方显示**(`Recipe Display`)

**配方显示**(`RecipeDisplay`)与**槽位显示** 类似，不同之处在于它代表整个配方。默认接口只跟踪配方的`result`(**结果**)和代表应用配方的工作台的`craftingStation`(**制作站**)。**配方显示** 也有一个`type`，它包含用于编码/解码显示的[**映射编解码器**][codec] 和 [**流编解码器**][streamcodec]。然而，**配方显示** 的现有子类型都不包含在客户端正确渲染我们的配方所需的所有信息。因此，我们需要创建自己的**配方显示**。

所有槽位和成分都应该表示为**槽位显示**。任何限制，如网格大小，可以由用户以任何方式提供。

```java
// 一个简单的配方显示
public record RightClickBlockRecipeDisplay(
    SlotDisplay inputState,
    SlotDisplay inputItem,
    SlotDisplay result, // 实现 RecipeDisplay#result
    SlotDisplay craftingStation // 实现 RecipeDisplay#craftingStation
) implements RecipeDisplay {
    public static final MapCodec<RightClickBlockRecipeDisplay> MAP_CODEC = RecordCodecBuilder.mapCodec(
        instance -> instance.group(
                    SlotDisplay.CODEC.fieldOf("inputState").forGetter(RightClickBlockRecipeDisplay::inputState),
                    SlotDisplay.CODEC.fieldOf("inputState").forGetter(RightClickBlockRecipeDisplay::inputItem),
                    SlotDisplay.CODEC.fieldOf("result").forGetter(RightClickBlockRecipeDisplay::result),
                    SlotDisplay.CODEC.fieldOf("crafting_station").forGetter(RightClickBlockRecipeDisplay::craftingStation)
                )
                .apply(instance, RightClickBlockRecipeDisplay::new)
    );
    public static final StreamCodec<RegistryFriendlyByteBuf, RightClickBlockRecipeDisplay> STREAM_CODEC = StreamCodec.composite(
        SlotDisplay.STREAM_CODEC,
        RightClickBlockRecipeDisplay::inputState,
        SlotDisplay.STREAM_CODEC,
        RightClickBlockRecipeDisplay::inputItem,
        SlotDisplay.STREAM_CODEC,
        RightClickBlockRecipeDisplay::result,
        SlotDisplay.STREAM_CODEC,
        RightClick极光RecipeDisplay::craftingStation,
        RightClickBlockRecipeDisplay::new
    );

    @Override
    public RecipeDisplay.Type<? extends RecipeDisplay> type() {
        // 返回下面注册的类型
        return RIGHT_CLICK_BLOCK_RECIPE_DISPLAY.get();
    }
}

// 在某个注册类中
/// 对于某个 DeferredRegister<RecipeDisplay.Type<?>> RECIPE_DISPLAY_TYPES
public static final Supplier<RecipeDisplay.Type<RightClickBlockRecipeDisplay>> RIGHT_CLICK_BLOCK_RECIPE_DISPLAY = RECIPE_DISPLAY_TYPES.register(
    "right_click_block",
    () -> new RecipeDisplay.Type<>(RightClickBlockRecipeDisplay.CODEC, RightClickBlockRecipeDisplay.STREAM_CODEC)
);
```

然后我们可以通过重写 `#display` 为配方创建配方显示，如下所示：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 这里是其他内容

    @Override
    public List<RecipeDisplay> display() {
        // 同一个配方可以有许多不同的显示
        // 但这个例子将像其他配方一样只使用一个。
        return List.of(
            // 添加我们的配方显示，并指定槽位
            new RightClickBlockRecipeDisplay(
                new BlockStateSlotDisplay(this.inputState),
                this.inputItem.display(),
                new SlotDisplay.ItemStackSlotDisplay(this.result),
                new SlotDisplay.ItemSlotDisplay(Items.GRASS_BLOCK)
            )
        )
    }
}
```

## **配方类型**(`Recipe Type`)

接下来是我们的配方类型。这相当简单，因为与配方类型关联的除了一个名称之外没有其他数据。它们是配方系统中两个[注册][registry]的部分之一，所以和所有其他注册表一样，我们创建一个 `DeferredRegister` 并向其注册：

```java
public static final DeferredRegister<RecipeType<?>> RECIPE_TYPES =
        DeferredRegister.create(Registries.RECIPE_TYPE, ExampleMod.MOD_ID);

public static final Supplier<RecipeType<RightClickBlockRecipe>> RIGHT_CLICK_BLOCK_TYPE =
        RECIPE_TYPES.register(
                "right_click_block",
                // 创建配方类型，将 `toString` 设置为类型的注册表名称
                RecipeType::simple
        );
```

在我们注册了配方类型之后，我们必须在我们的配方中重写 `#getType`，如下所示：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 这里是其他内容

    @Override
    public RecipeType<? extends Recipe<RightClickBlockInput>> getType() {
        return RIGHT_CLICK_BLOCK_TYPE.get();
    }
}
```

## **配方序列化器**(`Recipe Serializer`)

配方序列化器提供两个编解码器，一个**映射编解码器**和一个**流编解码器**，分别用于从/到 JSON 以及从/到网络进行序列化。本节不会深入介绍编解码器的工作原理，请参阅 [**映射编解码器**][codec] 和 [**流编解码器**][streamcodec] 以获取更多信息。

由于配方序列化器可能会变得相当大，原版将它们移到单独的类中。建议但不是必须遵循这个做法 - 较小的序列化器通常在配方类的字段内的匿名类中定义。为了遵循良好的做法，我们将创建一个单独的类来保存我们的编解码器：

```java
// 泛型参数是我们的配方类。
// 注意：这假设存在简单的 RightClickBlockRecipe#getInputState、#getInputItem 和 #getResult 获取器
// 上面的代码中省略了这些。
public class RightClickBlockRecipeSerializer implements RecipeSerializer<RightClickBlockRecipe> {
    public static final MapCodec<RightClickBlockRecipe> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            BlockState.CODEC.fieldOf("state").forGetter(RightClickBlockRecipe::getInputState),
            Ingredient.CODEC.fieldOf("ingredient").forGetter(RightClickBlockRecipe::getInputItem),
            ItemStack.CODEC.fieldOf("result").forGetter(RightClickBlockRecipe::getResult)
    ).apply(inst, RightClickBlockRecipe::new));
    public static final StreamCodec<RegistryFriendlyByteBuf, RightClickBlockRecipe> STREAM_CODEC =
            StreamCodec.composite(
                    ByteBufCodecs.idMapper(Block.BLOCK_STATE_REGISTRY), RightClickBlockRecipe::getInputState,
                    Ingredient.CONTENTS_STREAM_CODEC, RightClickBlockRecipe::getInputItem,
                    ItemStack.STREAM_CODEC, RightClickBlockRecipe::getResult,
                    RightClickBlockRecipe::new
            );

    // 返回我们的映射编解码器。
    @Override
    public MapCodec<RightClickBlockRecipe> codec() {
        return CODEC;
    }

    // 返回我们的流编解码器。
    @Override
    public StreamCodec<RegistryFriendlyByteBuf, RightClickBlockRecipe> streamCodec() {
        return STREAM_CODEC;
    }
}
```

和类型一样，我们注册我们的序列化器：

```java
public static final DeferredRegister<RecipeType<?>> RECIPE_SERIALIZERS =
        DeferredRegister.create(Registries.RECIPE_SERIALIZER, ExampleMod.MOD_ID);

public static final Supplier<RecipeSerializer<RightClickBlockRecipe>> RIGHT_CLICK_BLOCK =
        RECIPE_SERIALIZERS.register("right_click_block", RightClickBlockRecipeSerializer::new);
```

同样，我们也必须在我们的配方中重写 `#getSerializer`，如下所示：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 这里是其他内容

    @Override
    public RecipeSerializer<? extends Recipe<RightClickBlockInput>> getSerializer() {
        return RIGHT_CLICK_BLOCK.get();
    }
}
```

## **制作机制**(`Crafting Mechanic`)

现在你的配方的所有部分都完成了，你可以创建一些配方 JSON 文件（有关这方面的内容，请参阅 [数据生成](datagen) 部分），然后像上面那样向配方管理器查询你的配方。然后你对配方做什么取决于你自己。一个常见的用例是一个可以处理你的配方的机器，将活动配方作为一个字段存储。

然而，在我们的例子中，我们希望在物品右键点击方块时应用配方。我们将使用一个[事件处理程序][event]来实现。请记住，这是一个示例实现，你可以根据自己的喜好以任何方式修改它（只要你在服务器上运行它）。由于我们希望交互状态在客户端和服务器上都匹配，我们还需要[跨网络同步任何相关的输入状态][networking]。

我们可以设置一个简单的网络实现来同步配方输入，如下所示：

```java
// 一个基本的数据包类，必须进行注册。
public record ClientboundRightClickBlockRecipesPayload(
    Set<BlockState> inputStates, Set<Holder<Item>> inputItems
) implements CustomPacketPayload {

    // ...
}

// 数据包将数据存储在一个实例类中。
// 在服务器和客户端都存在，用于进行初始匹配。
public interface RightClickBlockRecipeInputs {

    Set<BlockState> inputStates();
    Set<Holder<Item>> inputItems();

    default boolean test(BlockState state, ItemStack stack) {
        return this.inputStates().contains(state) && this.inputItems().contains(stack.getItemHolder());
    }
}

// 服务器资源监听器，以便在配方重新加载时可以重新加载。
public class ServerRightClickBlockRecipeInputs implements ResourceManagerReloadListener, RightClickBlockRecipeInputs {

    private final RecipeManager recipeManager;

    private Set<BlockState> inputStates;
    private Set<Holder<Item>> inputItems;

    public RightClickBlockRecipeInputs(RecipeManager recipeManager) {
        this.recipeManager = recipeManager;
    }

    // 在这里设置输入，因为 #apply 是根据监听器注册顺序同步触发的。
    // 配方总是首先应用。
    @Override
    public void onResourceManagerReload(ResourceManager manager) {
        MinecraftServer server = ServerLifecycleHooks.getCurrentServer();
        if (server == null) return; // 应该永远不会为 null

        // 填充输入
        Set<BlockState> inputStates = new HashSet<>();
        Set<Holder<Item>> inputItems = new HashSet<>();

        this.recipeManager.recipeMap().byType(RIGHT_CLICK_BLOCK_TYPE.get())
            .forEach(holder -> {
                var recipe = holder.value();
                inputStates.add(recipe.getInputState());
                inputItems.addAll(recipe.getInputItem().items());
            });
        
        this.inputStates = Set.copyOf(inputStates);
        this.inputItems = Set.copyOf(inputItems);
    }

    public void syncToClient(Stream<ServerPlayer> players) {
        ClientboundRightClickBlockRecipesPayload payload =
            new ClientboundRightClickBlockRecipesPayload(this.inputStates, this.inputItems);
        players.forEach(player -> PacketDistributor.sendToPlayer(player, payload));
    }

    @Override
    public Set<BlockState> inputStates() {
        return this.inputStates;
    }

    @Override
    public Set<Holder<Item>> inputItems() {
        return this.inputItems;
    }
}

// 客户端实现，用于保存输入。
public record ClientRightClickBlockRecipeInputs(
    Set<BlockState> inputStates, Set<Holder<Item>> inputItems
) implements RightClickBlockRecipeInputs {

    public ClientRightClickBlockRecipeInputs(Set<BlockState> inputStates, Set<Holder<Item>> inputItems) {
        this.inputStates = Set.copyOf(inputStates);
        this.inputItems = Set.copyOf(inputItems);
    }
}

// 根据不同的端处理配方实例。
public class ServerRightClickBlockRecipes {

    private static ServerRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ServerRightClickBlockRecipes.inputs;
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void addListener(AddReloadListenerEvent event) {
        // 注册服务器重新加载监听器
        ServerRightClickBlockRecipes.inputs = new ServerRightClickBlockRecipeInputs(
            event.getServerResources().getRecipeManager()
        );
        event.addListener(ServerRightClickBlockRecipes.inputs);
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void datapackSync(OnDatapackSyncEvent event) {
        // 发送到客户端
        ServerRightClickBlockRecipes.inputs.syncToClient(event.getRelevantPlayers());
    }
}

public class ClientRightClickBlockRecipes {

    private static ClientRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ClientRightClickBlockRecipes.inputs;
    }

    // 处理发送的数据包
    public static void handle(final ClientboundRightClickBlockRecipesPayload data, final IPayloadContext context) {
        // 在主线程上处理数据
        ClientRightClickBlockRecipes.inputs = new ClientRightClickBlockRecipeInputs(
            data.inputStates(), data.inputItems()
        );
    }

    @SubscribeEvent // 仅在物理客户端的游戏事件总线上
    public static void clientLogOut(ClientPlayerNetworkEvent.LoggingOut event) {
        // 在退出世界时清除存储的输入
        ClientRightClickBlockRecipes.inputs = null;
    }
}

public class RightClickBlockRecipes {
    // 创建代理方法以正确访问
    public static RightClickBlockRecipeInputs inputs(Level level) {
        return level.isClientSide
            ? ClientRightClickBlockRecipes.inputs()
            : ServerRightClickBlockRecipes.inputs();
    }
}
```

或者，你可以[将完整的配方同步到客户端][clientrecipes]：

```java
// 在服务器和客户端都存在，用于进行初始匹配。
public interface RightClickBlockRecipeInputs {

    Set<BlockState> inputStates();
    Set<Holder<Item>> inputItems();

    default boolean test(BlockState state, ItemStack stack) {
        return this.inputStates().contains(state) && this.inputItems().contains(stack.getItemHolder());
    }
}

// 服务器资源监听器，以便在配方重新加载时可以重新加载。
public class ServerRightClickBlockRecipeInputs implements ResourceManagerReloadListener, RightClickBlockRecipeInputs {

    private final RecipeManager recipeManager;

    private Set<BlockState> inputStates;
    private Set<Holder<Item>> inputItems;

    public RightClickBlockRecipeInputs(RecipeManager recipeManager) {
        this.recipeManager = recipeManager;
    }

    // 在这里设置输入，因为 #apply 是根据监听极光注册顺序同步触发的。
    // 配方总是首先应用。
    @Override
    public void onResourceManagerReload(ResourceManager manager) {
        MinecraftServer server = ServerLifecycleHooks.getCurrentServer();
        if (server != null) { // 应该永远不会为 null
            // 填充输入
            Set<BlockState> inputStates = new HashSet<>();
            Set<Holder<Item>> inputItems = new HashSet<>();

            this.recipeManager.recipeMap().byType(RIGHT_CLICK_BLOCK_TYPE.get())
                .forEach(holder -> {
                    var recipe = holder.value();
                    inputStates.add(recipe.getInputState());
                    inputItems.addAll(recipe.getInputItem().items());
                });
            
            this.inputStates = Set.copyOf(inputStates);
            this.inputItems = Set.copyOf(inputItems);
        }
    }

    @Override
    public Set<BlockState> inputStates() {
        return this.inputStates;
    }

    @Override
    public Set<Holder<Item>> inputItems() {
        return this.inputItems;
    }
}

// 客户端实现，用于保存输入。
public record ClientRightClickBlockRecipeInputs(
    Set<BlockState> inputStates, Set<Holder<Item>> inputItems
) implements RightClickBlockRecipeInputs {

    public ClientRightClickBlockRecipeInputs(Set<BlockState> inputStates, Set<Holder<Item>> inputItems) {
        this.inputStates = Set.copyOf(inputStates);
        this.inputItems = Set.copyOf(inputItems);
    }
}

// 根据不同的端处理配方实例。
public class ServerRightClickBlockRecipes {

    private static ServerRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ServerRightClickBlockRecipes.inputs;
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void addListener(AddReloadListenerEvent event) {
        // 注册服务器重新加载监听器
        ServerRightClickBlockRecipes.inputs = new ServerRightClickBlockRecipeInputs(
            event.getServerResources().getRecipeManager()
        );
        event.addListener(ServerRightClickBlockRecipes.inputs);
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void datapackSync(OnDatapackSyncEvent event) {
        // 指定要同步到客户端的配方类型
        event.sendRecipes(RIGHT_CLICK_BLOCK_TYPE.get());
    }
}

public class ClientRightClickBlockRecipes {

    private static ClientRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ClientRightClickBlockRecipes.inputs;
    }

    @SubscribeEvent // 仅在物理客户端的游戏事件总线上
    public static void recipesReceived(RecipesReceivedEvent event) {
        // 存储配方
        Set<BlockState> inputStates = new HashSet<>();
        Set<Holder<Item>> inputItems = new HashSet<>();

        event.getRecipeMap().byType(RIGHT_CLICK_BLOCK_TYPE.get())
            .forEach(holder -> {
                var recipe = holder.value();
                inputStates.add(recipe.getInputState());
                inputItems.addAll(recipe.getInputItem().items());
            });
        
        ClientRightClickBlockRecipes.inputs = new ClientRightClickBlockRecipeInputs(
            inputStates, inputItems
        );
    }

    @SubscribeEvent // 仅在物理客户端的游戏事件总线上
    public static void clientLogOut(ClientPlayerNetworkEvent.LoggingOut event) {
        // 在退出世界时清除存储的输入
        ClientRightClickBlockRecipes.inputs = null;
    }
}

public class RightClickBlockRecipes {
    // 创建代理方法以正确访问
    public static RightClickBlockRecipeInputs inputs(Level level) {
        return level.isClientSide
            ? ClientRightClickBlockRecipes.inputs()
            : ServerRightClickBlockRecipes.inputs();
    }
}
```

然后，使用同步的输入，我们可以在游戏中检查使用的输入：

```java
@SubscribeEvent // 在游戏事件总线上
public static void useItemOnBlock(UseItemOnBlockEvent event) {
    // 如果我们不在事件的方块指定阶段，则跳过。有关详细信息，请参阅事件的 javadoc。
    if (event.getUsePhase() != UseItemOnBlockEvent.UsePhase.BLOCK) return;
    // 获取参数以首先检查输入
    Level level = event.getLevel();
    BlockPos pos = event.getPos();
    BlockState blockState = level.getBlockState(pos);
    ItemStack itemStack = event.getItemStack();

    // 检查输入在两端是否都能匹配配方
    if (!RightClickBlockRecipes.inputs(level).test(blockState, itemStack)) return;

    // 如果是这样，在检查配方之前确保在服务器端
    if (!level.isClientSide() && level instanceof ServerLevel serverLevel) {
        // 创建一个输入并查询配方。
        RightClickBlockInput input = new RightClickBlockInput(blockState, itemStack);
        Optional<RecipeHolder<? extends Recipe<CraftingInput>>> optional = serverLevel.recipeAccess().getRecipeFor(
            // 配方类型。
            RIGHT_CLICK_BLOCK_TYPE.get(),
            input,
            level
        );
        ItemStack result = optional
            .map(RecipeHolder::value)
            .map(e -> e.assemble(input, level.registryAccess()))
            .orElse(ItemStack.EMPTY);
        
        // 如果有结果，破坏方块并在世界中掉落结果。
        if (!result.isEmpty()) {
            level.removeBlock(pos, false);
            ItemEntity entity = new ItemEntity(level,
                    // 方块位置的中心。
                    pos.getX() + 0.5, pos.getY() + 0.5, pos.getZ() + 0.5,
                    result);
            level.addFreshEntity(entity);
        }
    }

    // 取消事件以停止交互管道，无论在哪一端。
    // 已经确保可能有结果。
    event.cancelWithResult(InteractionResult.SUCCESS_SERVER);
}
```

## **数据生成**(`Data Generation`)

要为你自己的配方序列化器创建一个配方构建器，你需要实现 `RecipeBuilder` 及其方法。一个常见的实现，部分复制自原版，如下所示：

```java
// 这个类是抽象的，因为每个配方序列化器有很多特定的逻辑。
// 它的目的是展示所有（原版）配方构建器的通用部分。
public abstract class SimpleRecipeBuilder implements RecipeBuilder {
    // 将字段设为受保护，以便我们的子类可以使用它们。
    protected final ItemStack result;
    protected final Map<String, Criterion<?>> criteria = new LinkedHashMap<>();
    @Nullable
    protected final String group;

    // 构造函数通常接受结果物品堆。
    // 另外，静态构建器方法也是可能的。
    public SimpleRecipeBuilder(ItemStack result) {
        this.result = result;
    }

    // 此方法为配方进度添加一个条件。
    @Override
    public SimpleRecipeBuilder unlockedBy(String name, Criterion<?> criterion) {
        this.criteria.put(name, criterion);
        return this;
    }

    // 此方法添加一个配方书组。如果你不想使用配方书组，
    // 请移除此 group 字段，并让此方法成为空操作（即返回 this）。
    @Override
    public SimpleRecipeBuilder group(@Nullable String group) {
        this.group = group;
        return this;
    }

    // 原版需要一个 Item 而不是 ItemStack。你仍然可以也应该使用 ItemStack
    // 来序列化配方。
    @Override
    public Item getResult() {
        return this.result.getItem();
    }
}
```

因此，我们为配方构建器奠定了基础。现在，在继续处理依赖于配方序列化器的部分之前，我们应该首先考虑如何制作配方工厂。在我们的情况下，直接使用构造函数是有意义的。在其他情况下，使用静态助手或小型函数接口是可行的方法。如果你为一个构建器对应多个配方类，这一点尤其重要。

利用 `RightClickBlockRecipe::new` 作为我们的配方工厂，并重用上面的 `SimpleRecipeBuilder` 类，我们可以为 `RightClickBlockRecipe` 创建以下配方构建器：

```java
public class RightClickBlockRecipeBuilder extends SimpleRecipeBuilder {
    private final BlockState inputState;
    private final Ingredient inputItem;

    // 由于我们每种输入正好有一个，因此将它们传递给构造函数。
    // 对于具有某种成分列表的配方序列化器的构建器，
    // 通常会初始化一个空列表，并有 #addIngredient 或类似方法。
    public RightClickBlockRecipeBuilder(ItemStack result, BlockState inputState, Ingredient inputItem) {
        super(result);
        this.inputState = inputState;
        this.inputItem = inputItem;
    }

    // 使用给定的 RecipeOutput 和键保存配方。此方法在 RecipeBuilder 接口中定义。
    @Override
    public void save(RecipeOutput output, ResourceKey<Recipe<?>> key) {
        // 构建进度。
        Advancement.Builder advancement = output.advancement()
                .addCriterion("has_the_recipe", RecipeUnlockedTrigger.unlocked(key))
                .rewards(AdvancementRewards.Builder.recipe(key))
                .requirements(AdvancementRequirements.Strategy.OR);
        this.criteria.forEach(advancement::addCriterion);
        // 我们的工厂参数是结果、方块状态和成分。
        RightClickBlockRecipe recipe = new RightClickBlockRecipe(this.inputState, this.inputItem, this.result);
        // 将 id、配方和配方进度传递给 RecipeOutput。
        output.accept(key, recipe, advancement.build(key.location().withPrefix("recipes/")));
    }
}
```

现在，在[数据生成][recipedatagen]期间，你可以像其他配方构建器一样调用你的配方构建器：

```java
@Override
protected void buildRecipes(RecipeOutput output) {
    new RightClickRecipeBuilder(
            // 我们的构造函数参数。这个例子添加了备受欢迎的泥土 -> 钻石转换。
            new ItemStack(Items.DIAMOND),
            Blocks.DIRT.defaultBlockState(),
            Ingredient.of(Items.APPLE)
    )
            .unlockedBy("has_apple", this.has(Items.APPLE))
            .save(output);
    // 这里还有其他配方构建器
}
```

:::note
也可以将 `SimpleRecipeBuilder` 合并到 `RightClickBlockRecipeBuilder`（或你自己的配方构建器）中，特别是如果你只有一个或两个配方构建器。这里的抽象是为了展示构建器的哪些部分依赖于配方，哪些不依赖。
:::

[clientrecipes]: index.md#client-side-recipes
[codec]: ../../../datastorage/codecs.md
[datagen]: #data-generation
[event]: ../../../concepts/events.md
[ingredients]: ingredients.md
[networking]: ../../../networking/payload.md
[recipedatagen]: index.md#data-generation
[registry]: ../../../concepts/registries.md#methods-for-registering
[streamcodec]: ../../../networking/streamcodecs.md