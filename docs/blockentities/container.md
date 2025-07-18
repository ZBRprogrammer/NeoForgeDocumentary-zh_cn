# **容器**(`Containers`)

[方块实体][blockentity]的一个常见用途是存储某种物品。Minecraft 中一些最基本的[方块][block]（如熔炉或箱子）都使用方块实体实现此目的。要在某物上存储物品，Minecraft 使用 `Container`（容器）。

`Container` 接口定义了 `#getItem`、`#setItem` 和 `#removeItem` 等方法，可用于查询和更新容器。由于是接口，它本身并不包含后备列表或其他数据结构，这取决于实现系统。

因此，`Container` 不仅可在方块实体上实现，也可在任何其他类上实现。值得注意的例子包括实体库存，以及常见的模组化[物品][item]（如背包）。

:::warning
NeoForge 在许多地方提供 `ItemStackHandler` 类作为 `Container` 的替代方案。应尽可能使用它而非 `Container`，因为它允许更清晰地与其他 `Container`/`ItemStackHandler` 交互。

本文存在的主要原因是供原版代码参考，或为多加载器开发模组时使用。在您自己的代码中请始终使用 `ItemStackHandler`！相关文档正在编写中。
:::

## 基本容器实现

容器可以按您喜欢的任何方式实现，只要满足规定的方法（与 Java 中的任何其他接口一样）。然而，通常使用固定长度的 `NonNullList<ItemStack>` 作为后备结构。单槽容器也可直接使用 `ItemStack` 字段。

例如，大小为 27 个槽位（一个箱子）的 `Container` 基本实现可能如下：

```java
public class MyContainer implements Container {
    private final NonNullList<ItemStack> items = NonNullList.withSize(
            // 列表大小，即容器中的槽位数
            27,
            // 用于替代普通列表中 null 的默认值
            ItemStack.EMPTY
    );

    // 容器中的槽位数
    @Override
    public int getContainerSize() {
        return 27;
    }
    
    // 容器是否被视为空
    @Override
    public boolean isEmpty() {
        return this.items.stream().allMatch(ItemStack::isEmpty);
    }
    
    // 返回指定槽位的物品堆
    @Override
    public ItemStack getItem(int slot) {
        return this.items.get(slot);
    }
    
    // 容器更改时调用（如物品堆添加、修改或移除时）
    // 例如，可在此处调用 BlockEntity#setChanged
    @Override
    public void setChanged() {
    
    }
    
    // 从指定槽位移除指定数量的物品，返回被移除的物品堆
    // 此处委托给 ContainerHelper，它按预期处理
    // 但必须手动调用 #setChanged
    @Override
    public ItemStack removeItem(int slot, int amount) {
        ItemStack stack = ContainerHelper.removeItem(this.items, slot, amount);
        this.setChanged();
        return stack;
    }
    
    // 移除指定槽位的所有物品，返回被移除的物品堆
    // 再次委托给 ContainerHelper，仍需手动调用 #setChanged
    @Override
    public ItemStack removeItemNoUpdate(int slot) {
        ItemStack stack = ContainerHelper.takeItem(this.items, slot);
        this.setChanged();
        return stack;
    }
    
    // 在指定槽位设置物品堆。先限制为容器的最大堆叠大小
    @Override
    public void setItem(int slot, ItemStack stack) {
        stack.limitSize(this.getMaxStackSize(stack));
        this.items.set(slot, stack);
        this.setChanged();
    }
    
    // 容器对给定玩家是否仍被视为"有效"。例如，箱子及
    // 类似方块在此检查玩家是否仍在方块一定距离内
    @Override
    public boolean stillValid(Player player) {
        return true;
    }
    
    // 清空内部存储，将所有槽位重置为空
    @Override
    public void clearContent() {
        items.clear();
        this.setChanged();
    }
}
```

### **SimpleContainer**

`SimpleContainer` 类是容器的基础实现，带有额外功能（如添加 `ContainerListener` 的能力）。若需要无特殊要求的容器实现，可使用此类。

### **BaseContainerBlockEntity**

`BaseContainerBlockEntity` 类是 Minecraft 中许多重要方块实体的基类，如箱子及类似方块、各种熔炉类型、漏斗、发射器、投掷器、酿造台等。

除 `Container` 外，它还实现了 `MenuProvider` 和 `Nameable` 接口：

- `Nameable` 定义了与设置（自定义）名称相关的几个方法，除许多方块实体外，`Entity` 等类也实现了此接口。这使用了[`Component` 系统][component]
- `MenuProvider` 定义了 `#createMenu` 方法，允许从容器构建 [`AbstractContainerMenu`][menu]。这意味着若需要无关联 GUI 的容器（如唱片机），此类并不适用

`BaseContainerBlockEntity` 通过 `#getItems` 和 `#setItems` 两个方法捆绑了所有本应对 `NonNullList<ItemStack>` 的调用，大幅减少了需编写的样板代码。`BaseContainerBlockEntity` 的示例实现可能如下：

```java
public class MyBlockEntity extends BaseContainerBlockEntity {
    // 容器大小。当然可以是任意值
    public static final int SIZE = 9;
    // 物品堆列表。因 #setItems 存在而非 final
    private NonNullList<ItemStack> items = NonNullList.withSize(SIZE, ItemStack.EMPTY);

    // 构造函数，如前所述
    public MyBlockEntity(BlockPos pos, BlockState blockState) {
        super(MY_BLOCK_ENTITY.get(), pos, blockState);
    }
    
    // 容器大小，如前所述
    @Override
    public int getContainerSize() {
        return SIZE;
    }
    
    // 获取物品堆列表
    @Override
    protected NonNullList<ItemStack> getItems() {
        return items;
    }
    
    // 设置物品堆列表
    @Override
    protected void setItems(NonNullList<ItemStack> items) {
        this.items = items;
    }
    
    // 菜单的显示名称。别忘了添加翻译！
    @Override
    protected Component getDefaultName() {
        return Component.translatable("container.examplemod.myblockentity");
    }
    
    // 从此容器创建的菜单。下文说明应返回的内容
    @Override
    protected AbstractContainerMenu createMenu(int containerId, Inventory inventory) {
        return null;
    }
}
```

请记住，此类同时是 `BlockEntity` 和 `Container`。这意味着可将此类用作方块实体的超类型，以获得具有预实现容器的功能方块实体。

:::note
实现 `Container` 的 `BlockEntity` 默认会处理内容物掉落。若选择不实现 `Container`，则需自行处理[移除逻辑][beremove]。
:::

### **WorldlyContainer**

`WorldlyContainer` 是 `Container` 的子接口，允许按 `Direction` 访问给定 `Container` 的槽位。主要用于仅向特定面公开部分容器的方块实体。例如，可用于从一面输出并从其他面输入的机器。该接口的简单实现可能如下：

```java
// 参见上文的 BaseContainerBlockEntity 方法。当然也可直接扩展 BlockEntity
// 并在需要时自行实现 Container
public class MyBlockEntity extends BaseContainerBlockEntity implements WorldlyContainer {
    // 其他内容
    
    // 假设槽位 0 是输出槽，槽位 1-8 是输入槽
    // 进一步假设我们向顶部输出并从其他所有面输入
    private static final int[] OUTPUTS = new int[]{0};
    private static final int[] INPUTS = new int[]{1, 2, 3, 4, 5, 6, 7, 8};
    
    // 根据传入的 Direction 返回公开槽位索引数组
    @Override
    public int[] getSlotsForFace(Direction side) {
        return side == Direction.UP ? OUTPUTS : INPUTS;
    }
    
    // 是否可通过给定面在指定槽位放置物品
    // 本例中，仅当不是从上方输入且在索引范围 [1,8] 内时返回 true
    @Override
    public boolean canPlaceItemThroughFace(int index, ItemStack itemStack, @Nullable Direction direction) {
        return direction != Direction.UP && index > 0 && index < 9;
    }
    
    // 是否可从给定面和指定槽位取出物品
    // 本例中，仅当从上方拉取且槽位索引为 0 时返回 true
    @Override
    public boolean canTakeItemThroughFace(int index, ItemStack stack, Direction direction) {
        return direction == Direction.UP && index == 0;
    }
}
```

## 使用容器

现在我们已经创建了容器，来使用它们吧！

由于 `Container` 和 `BlockEntity` 有相当大的重叠，最佳做法是在可能时将方块实体强制转换为 `Container`：

```java
if (blockEntity instanceof Container container) {
    // 对容器进行操作
}
```

然后可使用之前提到的方法操作容器，例如：

```java
// 获取容器中的第一个物品
ItemStack stack = container.getItem(0);

// 将容器中的第一个物品设置为泥土
container.setItem(0, new ItemStack(Items.DIRT));

// 从第三个槽位移除最多 16 个物品
container.removeItem(2, 16);
```

:::warning
若尝试访问超出容器大小的槽位，容器可能抛出异常。或者，它们可能返回 `ItemStack.EMPTY`（例如 `SimpleContainer` 就是如此）。
:::

## `ItemStack` 上的容器

到目前为止，我们主要讨论了 `BlockEntity` 上的 `Container`。但它们也可通过 `minecraft:container` [数据组件][datacomponent]应用于 [`ItemStack`][itemstack]：

```java
// 此处使用 SimpleContainer 作为超类，避免自行重新实现物品处理逻辑
// 由于 SimpleContainer 的实现细节，若多方同时访问容器可能导致竞态条件
// 因此我们假设模组不允许这种情况
// 当然也可使用不同的 Container 实现（或自行实现 Container）
public class MyBackpackContainer extends SimpleContainer {
    // 此容器对应的物品堆。在构造函数中传入并设置
    private final ItemStack stack;
    
    public MyBackpackContainer(ItemStack stack) {
        // 调用超类构造函数并传入期望的容器大小
        super(27);
        // 设置物品堆字段
        this.stack = stack;
        // 从数据组件加载容器内容（若存在），由 ItemContainerContents 类表示
        // 若不存在，则使用 ItemContainerContents.EMPTY
        ItemContainerContents contents = stack.getOrDefault(DataComponents.CONTAINER, ItemContainerContents.EMPTY);
        // 将数据组件内容复制到物品堆列表中
        contents.copyInto(this.getItems());
    }
    
    // 内容更改时，将数据组件保存到物品堆
    @Override
    public void setChanged() {
        super.setChanged();
        this.stack.set(DataComponents.CONTAINER, ItemContainerContents.fromItems(this.getItems()));
    }
}
```

这样您就创建了基于物品的容器！调用 `new MyBackpackContainer(stack)` 可为菜单或其他用例创建容器。

:::warning
请注意，直接与 `Container` 交互的菜单在修改 `ItemStack` 时必须 `#copy()` 它们，否则会破坏数据组件的不可变性契约。为此，NeoForge 提供了 `StackCopySlot` 类。
:::

## `Entity` 上的容器

[`Entity`][entity] 上的 `Container` 较为复杂：实体是否有容器无法统一确定。这完全取决于处理的实体类型，因此可能需要大量特殊处理。

若您自己创建实体，可直接在其上实现 `Container`，但请注意无法使用 `SimpleContainer` 等超类（因为 `Entity` 是超类）。

### **Mob** 上的容器

`Mob` 不实现 `Container`，但实现了 `EquipmentUser` 接口（以及其他接口）。该接口定义了 `#setItemSlot(EquipmentSlot, ItemStack)`、`#getItemBySlot(EquipmentSlot)` 和 `#setDropChance(EquipmentSlot, float)` 等方法。虽然代码层面与 `Container` 无关，但功能相当相似：我们将槽位（此处为装备槽位）与 `ItemStack` 关联。

与 `Container` 最显著的区别在于没有类似列表的顺序（尽管 `Mob` 在后台使用 `NonNullList<ItemStack>`）。访问不通过槽位索引，而是通过七个 `EquipmentSlot` 枚举值：`MAINHAND`、`OFFHAND`、`FEET`、`LEGS`、`CHEST`、`HEAD` 和 `BODY`（`BODY` 用于马和狗的护甲）。

与生物"槽位"交互的示例如下：

```java
// 获取 HEAD（头盔）槽位的物品堆
ItemStack helmet = mob.getItemBySlot(EquipmentSlot.HEAD);

// 将基岩放入生物的 FEET（靴子）槽位
mob.setItemSlot(EquipmentSlot.FEET, new ItemStack(Items.BEDROCK));

// 设置该基岩在生物被杀死时总是掉落
mob.setDropChance(EquipmentSlot.FEET, 1f);
```

### **InventoryCarrier**

`InventoryCarrier` 是由某些生物（如村民）实现的接口。它声明了 `#getInventory` 方法，返回 `SimpleContainer`。此接口用于需要实际库存（而非仅由 `EquipmentUser` 提供的装备槽位）的非玩家实体。

### **Player** 上的容器（玩家库存）

玩家的库存通过 `Inventory` 类实现，该类实现了 `Container` 以及前述的 `Nameable` 接口。该 `Inventory` 实例作为名为 `inventory` 的字段存储在 `Player` 上，可通过 `Player#getInventory` 访问。可像任何其他容器一样与此库存交互。

库存内容存储在两个位置：

- `NonNullList<ItemStack> items` 列表覆盖 36 个主库存槽位，包括 9 个快捷栏槽位（索引 0-8）
- `EntityEquipment equipment` 映射存储 `EquipmentSlot` 堆栈：护甲槽位（`FEET`、`LEGS`、`CHEST`、`HEAD`）、`OFFHAND`、`BODY` 和 `SADDLE`（按此顺序）

遍历库存内容时，建议先遍历 `items`，然后使用 `Inventory#EQUIPMENT_SLOT_MAPPING` 遍历 `equipment` 的索引。

[beremove]: index.md#removing-block-entities
[block]: ../blocks/index.md
[blockentity]: index.md
[component]: ../resources/client/i18n.md#components
[datacomponent]: ../items/datacomponents.md
[entity]: ../entities/index.md
[item]: ../items/index.md
[itemstack]: ../items/index.md#itemstacks
[menu]: ../gui/menus.md