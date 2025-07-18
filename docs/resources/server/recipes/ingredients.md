# 材料

**材料**(`Ingredient`) 用于 [配方] 中，以检查给定的 ``ItemStack``([`ItemStack`][itemstack]) 是否为该配方的有效输入。为此，``Ingredient`` 实现了 ``Predicate<ItemStack>``，并且可以调用 ``#test`` 来确认给定的 ``ItemStack`` 是否与该材料匹配。

不幸的是，``Ingredient`` 的许多内部实现一团糟。NeoForge 通过尽可能忽略 ``Ingredient`` 类来解决这个问题，而是为自定义材料引入了 ``ICustomIngredient`` 接口。这并不是对常规 ``Ingredient`` 的直接替代，但我们可以分别使用 ``ICustomIngredient#toVanilla`` 和 ``Ingredient#getCustomIngredient`` 在 ``Ingredient`` 之间进行转换。

## 内置材料类型

获取材料的最简单方法是使用 ``Ingredient#of`` 辅助方法。存在几种变体：

- ``Ingredient.of()`` 返回一个空材料。
- ``Ingredient.of(Blocks.IRON_BLOCK, Items.GOLD_BLOCK)`` 返回一个接受铁块或金块的材料。参数是一个 ``ItemLike``([`ItemLike`s][itemlike]) 的可变参数，这意味着可以使用任意数量的方块和物品。
- ``Ingredient.of(Stream.of(Items.DIAMOND_SWORD))`` 返回一个接受物品的材料。与前一个方法类似，但使用 ``Stream<ItemLike>``，如果你恰好有一个这样的流的话。
- ``Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.WOODEN_SLABS))`` 返回一个接受指定 [标签] 中任何物品的材料，例如任何木质台阶。

此外，NeoForge 添加了一些额外的材料：

- ``BlockTagIngredient.of(BlockTags.CONVERTABLE_TO_MUD)`` 返回一个类似于 ``Ingredient.of()`` 的标签变体的材料，但使用方块标签。当你需要使用物品标签，但只有方块标签可用时（例如 ``minecraft:convertable_to_mud``），应该使用此方法。
- ``CustomDisplayIngredient.of(Ingredient.of(Items.DIRT), SlotDisplay.Empty.INSTANCE)`` 返回一个带有自定义 ``SlotDisplay``([`SlotDisplay`][slotdisplay]) 的材料，你可以提供该显示方式来确定客户端渲染时槽位如何消耗。
- ``CompoundIngredient.of(Ingredient.of(Items.DIRT))`` 返回一个带有子材料的材料，子材料在构造函数（可变参数）中传递。如果其子材料中的任何一个匹配，则该材料匹配。
- ``DataComponentIngredient.of(true, new ItemStack(Items.DIAMOND_SWORD))`` 返回一个除了物品之外还匹配数据组件的材料。布尔参数表示严格匹配（true）或部分匹配（false）。严格匹配意味着数据组件必须完全匹配，而部分匹配意味着数据组件必须匹配，但也可以存在其他数据组件。存在 ``#of`` 的其他重载方法，允许指定多个 ``Item`` 或提供其他选项。
- ``DifferenceIngredient.of(Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.PLANKS)), Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.NON_FLAMMABLE_WOOD)))`` 返回一个匹配第一个材料中所有不匹配第二个材料的物品的材料。给定的示例仅匹配可以燃烧的木板（即除绯红木板、诡异木板和模组化下界木板之外的所有木板）。
- ``IntersectionIngredient.of(Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.PLANKS)), Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.NON_FLAMMABLE_WOOD)))`` 返回一个匹配同时匹配两个子材料的所有物品的材料。给定的示例仅匹配不能燃烧的木板（即绯红木板、诡异木板和模组化下界木板）。

:::note
如果你在使用数据生成时使用的材料需要一个用于标签实例的 ``HolderSet``（即调用 ``Registry#getOrThrow`` 的那些），那么该 ``HolderSet`` 应该通过 ``HolderLookup.Provider`` 获得，使用 ``HolderLookup.Provider#lookupOrThrow`` 来获取物品注册表，并使用 ``HolderGetter#getOrThrow`` 和 ``TagKey`` 来获取持有者集合。
:::

请记住，NeoForge 提供的材料类型是 ``ICustomIngredient``，并且在香草上下文（vanilla contexts）中使用它们之前必须调用 ``#toVanilla``，如本文开头所述。

## 自定义材料类型

模组开发者可以通过 ``ICustomIngredient`` 系统添加他们自己的自定义材料类型。为了举例，让我们创建一个接受物品标签和附魔到最小等级映射的附魔物品材料：

```java
public class MinEnchantedIngredient implements ICustomIngredient {
    private final TagKey<Item> tag;
    private final Map<Holder<Enchantment>, Integer> enchantments;
    // 用于序列化材料的编解码器。
    public static final MapCodec<MinEnchantedIngredient> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            TagKey.codec(Registries.ITEM).fieldOf("tag").forGetter(e -> e.tag),
            Codec.unboundedMap(Enchantment.CODEC, Codec.INT)
                    .optionalFieldOf("enchantments", Map.of())
                    .forGetter(e -> e.enchantments)
    ).apply(inst, MinEnchantedIngredient::new));
    // 从常规编解码器创建一个流编解码器。在某些情况下，从头定义一个新的流编解码器可能是有意义的。
    public static final StreamCodec<RegistryFriendlyByteBuf, MinEnchantedIngredient> STREAM_CODEC =
            ByteBufCodecs.fromCodecWithRegistries(CODEC.codec());

    // 允许传入一个预先存在的附魔到等级的映射。
    public MinEnchantedIngredient(TagKey<Item> tag, Map<Holder<Enchantment>, Integer> enchantments) {
        this.tag = tag;
        this.enchantments = enchantments;
    }

    // 通过验证物品是否在标签中以及是否存在所有所需等级的附魔来检查传入的 ItemStack 是否匹配我们的材料。
    @Override
    public boolean test(ItemStack stack) {
        return stack.is(tag) && enchantments.keySet()
                .stream()
                .allMatch(ench -> EnchantmentHelper.getEnchantmentsForCrafting(stack).getLevel(ench) >= enchantments.get(ench));
    }

    // 确定此材料是否执行 NBT 或数据组件匹配（false）或不匹配（true）。
    // 还确定是否使用流编解码器进行同步，稍后会详细介绍。
    // 我们查询栈上的附魔，因此我们的材料不是简单的。
    @Override
    public boolean isSimple() {
        return false;
    }

    // 返回匹配此材料的物品流。主要用于显示目的。
    // 这里有一些好的实践需要遵循：
    // - 始终至少包含一个物品，以防止意外识别为空。
    // - 每个接受的 Item 至少包含一次。
    // - 如果 #isSimple 为 true，则此流应精确包含每个匹配的物品。
    //   如果不是，则应尽可能精确，但不需要非常准确。
    // 在我们的例子中，我们使用标签中的所有物品。
    @Override
    public Stream<Holder<Item>> items() {
        return BuiltInRegistries.ITEM.getOrThrow(tag).stream();
    }
}
```

自定义材料是一个 [注册表]，因此我们必须注册我们的材料。我们使用 NeoForge 提供的 ``IngredientType`` 类来完成此操作，该类基本上是一个围绕 ``MapCodec``([`MapCodec`][codec]) 并且可选地围绕 ``StreamCodec``([`StreamCodec`][streamcodec]) 的包装器。

```java
public static final DeferredRegister<IngredientType<?>> INGREDIENT_TYPES =
        DeferredRegister.create(NeoForgeRegistries.Keys.INGREDIENT_TYPE, ExampleMod.MOD_ID);

public static final Supplier<IngredientType<MinEnchantedIngredient>> MIN_ENCHANTED =
        INGREDIENT_TYPES.register("min_enchanted",
                // 流编解码器参数是可选的，如果未指定流编解码器，将使用 ByteBufCodecs#fromCodec 或 #fromCodecWithRegistries 从编解码器创建一个流编解码器。
                () -> new IngredientType<>(MinEnchantedIngredient.CODEC, MinEnchantedIngredient.STREAM_CODEC));
```

完成上述操作后，我们还需要在我们的材料类中重写 ``#getType``：

```java
public class MinEnchantedIngredient implements ICustomIngredient {
    // 这里是其他内容

    @Override    
    public IngredientType<?> getType() {
        return MIN_ENCHANTED.get();
    }
}
```

就这样！我们的材料类型可以使用了。

## JSON 表示

由于香草材料（vanilla ingredients）相当有限，并且 NeoForge 为它们引入了一个全新的注册表，因此查看内置材料和我们自己的材料在 JSON 中的样子也是值得的。

指定了 ``neoforge:ingredient_type`` 的对象材料通常被认为是非香草材料。例如：

```json5
{
    "neoforge:ingredient_type": "neoforge:block_tag",
    "tag": "minecraft:convertable_to_mud"
}
```

或者使用我们自己的材料的另一个示例：

```json5
{
    "neoforge:ingredient_type": "examplemod:min_enchanted",
    "tag": "c:swords",
    "enchantments": {
        "minecraft:sharpness": 4
    }
}
```

如果材料是一个字符串，即未指定 ``neoforge:ingredient_type``，那么我们有一个香草材料。香草材料是表示物品的字符串，或者当以 ``#`` 为前缀时表示标签。

香草物品材料的示例：

```json5
"minecraft:dirt"
```

香草标签材料的示例：

```json5
"#c:ingots"
```

[codec]: ../../../datastorage/codecs.md
[itemlike]: ../../../items/index.md#itemlike
[itemstack]: ../../../items/index.md#itemstacks
[recipes]: index.md
[registry]: ../../../concepts/registries.md
[slotdisplay]: index.md#slot-displays
[streamcodec]: ../../../networking/streamcodecs.md
[tag]: ../tags.md