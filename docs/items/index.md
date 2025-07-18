# **物品**(`Items`)

与**方块**(`Blocks`)类似，**物品**(`Items`)是Minecraft的核心组成部分。方块构成周围世界，而物品存在于**物品栏**(`inventories`)中。

## 物品是什么？(`What Even Is an Item?`)

深入创建物品前，理解物品本质及其与[方块](`block`)的区别至关重要。以下示例说明：

- 世界中遇到泥土方块并挖掘时，它是**方块**(`block`)，因其放置在世界中（实际是**方块状态**(`blockstate`)，详见[方块状态文章](`blockstates`)）。
    - 并非所有方块被破坏时掉落自身（如树叶），详见[战利品表文章](`loottables`)。
- [破坏方块](`breaking`)后，方块被移除（替换为空气方块），泥土作为**物品实体**(`item entity`)掉落。这意味着它可被水推动或火/熔岩燃烧（类似猪、僵尸、箭等实体）。
- 拾取泥土物品实体后，它成为物品栏中的**物品堆**(`item stack`)。物品堆本质上是物品实例及其额外信息（如堆叠数量）。
- 物品堆由其对应的**物品**(`item`)（我们将创建）支持。物品持有[数据组件](`datacomponents`)，包含所有物品堆初始化的默认信息（如所有铁剑耐久度为250）。物品堆可修改这些数据组件，使相同物品的不同堆拥有不同信息（如一把铁剑剩余100耐久，另一把剩余200）。更多信息详见后续内容。
    - 物品与物品堆的关系类似[方块](`block`)与[方块状态](`blockstates`)（方块状态始终由方块支持）。此比喻虽不精确（物品堆非单例），但有助于理解核心概念。

## 创建物品(`Creating an Item`)

理解物品后，开始创建！

对于无需特殊功能的物品（如木棍、糖），可直接使用`Item`类。注册时用`Item.Properties`参数实例化`Item`。该参数通过`Item.Properties#of`创建，可通过其方法定制：

- `setId` - 设置物品的资源键（**必须设置**，否则抛出异常）。
- `overrideDescription` - 设置物品翻译键（生成的`Component`存储在`DataComponents#ITEM_NAME`）。
- `useBlockDescriptionPrefix` - 便捷方法，用`block.<modid>.<registry_name>`调用`overrideDescription`（应在所有`BlockItem`上调用）。
- `requiredFeatures` - 设置物品所需的**特性标志**(`feature flags`)。主要用于原版小版本的特性锁定系统，除非集成原版特性标志系统，否则不建议使用。
- `stacksTo` - 设置最大堆叠数量（通过`DataComponents#MAX_STACK_SIZE`）。默认为64（如末影珍珠堆叠上限为16）。
- `durability` - 设置耐久度（通过`DataComponents#MAX_DAMAGE`）和初始伤害值0（通过`DataComponents#DAMAGE`）。默认为0（无耐久度），铁工具设为250。**注意**：设置耐久度会强制最大堆叠数为1。
- `fireResistant` - 使使用此物品的物品实体免疫火/熔岩（通过`DataComponents#FIRE_RESISTANT`），下界合金物品使用此属性。
- `rarity` - 设置物品稀有度（通过`DataComponents#RARITY`）。目前仅改变物品颜色：`COMMON`（白色，默认）、`UNCOMMON`（黄色）、`RARE`（青色）、`EPIC`（浅紫色）。注意模组可能添加更多稀有度类型。
- `setNoCombineRepair` - 禁用砂轮和合成格修复（原版未使用）。
- `jukeboxPlayable` - 设置插入唱片机时播放的数据包`JukeboxSong`资源键。
- `food` - 设置物品的[`FoodProperties`](`food`)（通过`DataComponents#FOOD`）。

示例或原版值参考`Items`类。

### 残留物与冷却时间(`Remainders and Cooldowns`)

物品在使用时可能有额外属性或阻止使用一段时间：

- `craftRemainder` - 设置合成残留物。原版用于填充桶（合成后留下空桶）。
- `usingConvertsTo` - 设置物品通过`Item#use`、`IItemExtension#finishUsingItem`或`Item#releaseUsing`使用后返回的物品（`ItemStack`存储在`DataComponents#USE_REMAINDER`）。
- `useCooldown` - 设置再次使用前的冷却秒数（通过`DataComponents#USE_COOLDOWN`）。

### 工具与护甲(`Tools and Armor`)

部分物品作为[工具](`tools`)和[护甲](`armor`)使用。通过一系列物品属性构建，部分功能委托给关联类：

- `enchantable` - 设置堆叠的最大[附魔](`enchantment`)值（通过`DataComponents#ENCHANTABLE`），允许物品附魔。
- `repairable` - 设置可修复物品耐久的物品或标签（通过`DataComponents#REPAIRABLE`）。必须有耐久组件且非`DataComponents#UNBREAKABLE`。
- `equippable` - 设置可装备的槽位（通过`DataComponents#EQUIPPABLE`）。
- `equippableUnswappable` - 同`equippable`，但禁用使用物品按钮（默认右键）快速交换。

更多信息见相关页面。

### 更多功能(`More Functionality`)

直接使用`Item`仅支持基础物品。如需添加功能（如右键交互），需扩展`Item`的自定义类。`Item`类有许多可重写方法，详见`Item`和`IItemExtension`类。

物品最常见的两种用例是左键和右键点击。因其复杂性和跨系统特性，在单独的[交互文章](`interactions`)中说明。

### `DeferredRegister.Items`

所有注册表使用`DeferredRegister`注册内容，物品也不例外。因添加新物品是模组核心功能，NeoForge提供`DeferredRegister.Items`助手类（扩展`DeferredRegister<Item>`），提供物品专用方法：

```java
public static final DeferredRegister.Items ITEMS = DeferredRegister.createItems(ExampleMod.MOD_ID);

public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerItem(
    "example_item",
    Item::new, // 属性将传入的工厂
    new Item.Properties() // 使用的属性
);
```

内部将调用`ITEMS.register("example_item", registryName -> new Item(new Item.Properties().setId(ResourceKey.create(Registries.ITEM, registryName))))`（将属性参数应用于提供的物品工厂，通常为构造函数）。属性中设置ID。

若使用`Item::new`，可省略工厂直接使用`simple`方法变体：

```java
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerSimpleItem(
    "example_item",
    new Item.Properties() // 使用的属性
);
```

此操作与上例相同但更简洁。当然，若使用`Item`子类而非`Item`自身，需用前一种方法。

两种方法均有省略`new Item.Properties()`参数的重载：

```java
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerItem("example_item", Item::new);

// 省略Item::new参数的重载
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerSimpleItem("example_item");
```

最后，还有方块物品的快捷方式。除`setId`外，还调用`useBlockDescriptionPrefix`设置翻译键（与方块相同）：

```java
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    "example_block",
    ExampleBlocksClass.EXAMPLE_BLOCK,
    new Item.Properties()
);

// 省略属性参数的重载：
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    "example_block",
    ExampleBlocksClass.EXAMPLE_BLOCK
);

// 省略名称参数的重载（使用方块的注册名）：
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    // 必须是`Holder<Block>`实例
    // DeferredBlock<T>也可用
    ExampleBlocksClass.EXAMPLE_BLOCK,
    new Item.Properties()
);

// 省略名称和属性的重载：
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    // 必须是`Holder<Block>`实例
    // DeferredBlock<T>也可用
    ExampleBlocksClass.EXAMPLE_BLOCK
);
```

:::note
若方块注册在独立类中，应在物品类前加载方块类。
:::

### **资源**(`Resources`)

注册物品后通过`/give`或[创造模式标签页](`creativetabs`)获取，会发现缺少模型和纹理。因纹理和模型由Minecraft资源系统处理。

为物品应用简单纹理需创建客户端物品、模型JSON和纹理PNG。详见[客户端物品](`citems`)部分。

## **物品堆**(`ItemStack`s)

与方块和方块状态类似，预期使用`Item`的场合实际多用`ItemStack`。`ItemStack`代表容器中一个或多个物品的堆叠（如物品栏）。类似方块和方块状态，方法应由`Item`重写并在`ItemStack`上调用，`Item`中许多方法传入`ItemStack`实例。

`ItemStack`包含三部分：
- 代表的`Item`（通过`ItemStack#getItem`获取，或`getItemHolder`获取`Holder<Item>`）。
- 堆叠数量（通常1-64），通过`getCount`获取，`setCount`或`shrink`修改。
- [数据组件](`datacomponents`)映射，存储堆叠特定数据（通过`getComponents`获取）。通常通过`has`、`get`、`set`、`update`和`remove`访问和修改组件值。

创建新`ItemStack`调用`new ItemStack(Item)`（传入支撑物品）。默认数量1且无NBT数据；有重载构造函数接受数量和NBT数据。

`ItemStack`是**可变对象**(`mutable objects`)（见下文），但有时需视为不可变。若需修改被视为不可变的`ItemStack`，可用`#copy`克隆堆栈，或`#copyWithCount`克隆指定数量。

表示空堆栈用`ItemStack.EMPTY`。检查`ItemStack`是否空调用`#isEmpty`。

### **物品堆的可变性**(`Mutability of ItemStack`s)

`ItemStack`是可变对象。调用如`#setCount`或任何数据组件映射方法会修改`ItemStack`自身。原版广泛使用此特性，许多方法依赖它。如`#split`从调用堆栈拆分指定数量，同时修改调用者并返回新`ItemStack`。

但处理多个`ItemStack`时可能导致问题。最常见于处理物品栏槽位时，需同时考虑光标当前选中的`ItemStack`及尝试插入/提取的`ItemStack`。

:::tip
不确定时，安全起见使用`#copy`复制堆栈。
:::

### **JSON表示**(`JSON Representation`)

许多场合（如[配方](`recipes`)）需将物品堆表示为JSON对象：

```json5
{
    // 物品ID（必需）
    "id": "minecraft:dirt",
    // 堆叠数量[1,99]（可选，默认1）
    "count": 4,
    // 数据组件映射（可选，默认空映射）
    "components": {
        "minecraft:enchantment_glint_override": true
    }
}
```

## **创造模式标签页**(`Creative Tabs`)

默认物品仅通过`/give`获取，不出现在创造物品栏。让我们改变这一点！

物品加入创造模式菜单的方式取决于目标标签页。

### 现有创造模式标签页(`Existing Creative Tabs`)

:::note
此方法用于将物品添加到Minecraft或其他模组的标签页。添加到自己标签页见下文。
:::

通过[模组事件总线](`modbus`)（仅[逻辑客户端](`sides`)）的`BuildCreativeModeTabContentsEvent`将物品添加到现有`CreativeModeTab`。调用`event#accept`添加物品。

```java
// MyItemsClass.MY_ITEM是Supplier<? extends Item>，MyBlocksClass.MY_BLOCK是Supplier<? extends Block>
@SubscribeEvent // 在模组事件总线
public static void buildContents(BuildCreativeModeTabContentsEvent event) {
    // 检查目标标签页
    if (event.getTabKey() == CreativeModeTabs.INGREDIENTS) {
        event.accept(MyItemsClass.MY_ITEM.get());
        // 接受ItemLike（假设MY_BLOCK有对应物品）
        event.accept(MyBlocksClass.MY_BLOCK.get());
    }
}
```

事件还提供额外信息，如`getFlags`获取启用的特性标志列表，或`hasPermissions`检查玩家是否有权限查看操作员物品标签页。

### 自定义创造模式标签页(`Custom Creative Tabs`)

`CreativeModeTab`是注册表，自定义`CreativeModeTab`必须[注册](`registering`)。创建创造标签页使用构建器系统（通过`CreativeModeTab#builder`获取）。构建器提供设置标题、图标、默认物品及其他属性的选项。NeoForge额外提供方法定制标签页图像、标签和槽位颜色、排序位置等。

```java
// CREATIVE_MODE_TABS是DeferredRegister<CreativeModeTab>
public static final Supplier<CreativeModeTab> EXAMPLE_TAB = CREATIVE_MODE_TABS.register("example", () -> CreativeModeTab.builder()
    // 设置标签页标题（勿忘添加翻译！）
    .title(Component.translatable("itemGroup." + MOD_ID + ".example"))
    // 设置标签页图标
    .icon(() -> new ItemStack(MyItemsClass.EXAMPLE_ITEM.get()))
    // 向标签页添加物品
    .displayItems((params, output) -> {
        output.accept(MyItemsClass.MY_ITEM.get());
        // 接受ItemLike（假设MY_BLOCK有对应物品）
        output.accept(MyBlocksClass.MY_BLOCK.get());
    })
    .build()
);
```

## `ItemLike`

`ItemLike`是`Item`和原版[方块](`block`)实现的接口。定义`#asItem`方法，返回对象的物品表示：`Item`返回自身，`Block`返回关联的`BlockItem`（若有），否则返回`Blocks.AIR`。`ItemLike`用于物品"来源"不重要的场合（如许多[数据生成器](`datagen`)）。

也可在自定义对象上实现`ItemLike`，重写`#asItem`即可。

[armor]: armor.md
[block]: ../blocks/index.md
[blockstates]: ../blocks/states.md
[breaking]: ../blocks/index.md#breaking-a-block
[citems]: ../resources/client/models/items.md
[creativetabs]: #creative-tabs
[datacomponents]: datacomponents.md
[datagen]: ../resources/index.md#data-generation
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[entity]: ../entities/index.md
[food]: consumables.md#food
[hunger]: https://minecraft.wiki/w/Hunger#Mechanics
[interactions]: interactions.md
[loottables]: ../resources/server/loottables/index.md
[modbus]: ../concepts/events.md#event-buses
[recipes]: ../resources/server/recipes/index.md
[registering]: ../concepts/registries.md#methods-for-registering
[sides]: ../concepts/sides.md
[tools]: tools.md