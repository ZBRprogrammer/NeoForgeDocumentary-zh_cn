# **菜单**(`Menus`)

菜单是**图形用户界面**（`Graphical User Interfaces, GUIs`）的一种后端类型，处理与所表示数据持有者交互的逻辑。菜单本身非数据持有者，而是允许用户间接修改内部数据持有者状态的视图。因此，数据持有者不应直接耦合到任何菜单，而应传入数据引用以供调用和修改。

## `MenuType`

菜单动态创建和移除，因此非注册对象。需注册另一个工厂对象以轻松创建和引用菜单的*类型*。对菜单而言，此对象为`MenuType`。

`MenuType`必须[**注册**(`registered`)]。

### `MenuSupplier`

通过向`MenuType`构造函数传入`MenuSupplier`和`FeatureFlagSet`创建`MenuType`。`MenuSupplier`表示一个函数，接收容器 ID 和查看菜单的玩家物品栏，返回新建的[`AbstractContainerMenu`][acm]。

```java
// 对于某个 DeferredRegister<MenuType<?>> REGISTER
public static final Supplier<MenuType<MyMenu>> MY_MENU = REGISTER.register("my_menu", () -> new MenuType<>(MyMenu::new, FeatureFlags.DEFAULT_FLAGS));

// 在 AbstractContainerMenu 子类 MyMenu 中
public MyMenu(int containerId, Inventory playerInv) {
    super(MY_MENU.get(), containerId);
    // ...
}
```

:::note
容器标识符对单个玩家唯一。即不同玩家的相同容器 ID 代表不同菜单，即使查看同一数据持有者。
:::

`MenuSupplier`通常负责在客户端创建菜单，使用虚拟数据引用存储服务器数据持有者的同步信息并与之交互。

### `IContainerFactory`

若客户端需额外信息（如数据持有者在世界中的位置），可使用子类`IContainerFactory`。除容器 ID 和玩家物品栏外，还提供`RegistryFriendlyByteBuf`存储从服务器发送的额外信息。可通过`IMenuTypeExtension#create`使用`IContainerFactory`创建`MenuType`。

```java
// 对于某个 DeferredRegister<MenuType<?>> REGISTER
public static final Supplier<MenuType<MyMenuExtra>> MY_MENU_EXTRA = REGISTER.register("my_menu_extra", () -> IMenuTypeExtension.create(MyMenu::new));

// 在 AbstractContainerMenu 子类 MyMenuExtra 中
public MyMenuExtra(int containerId, Inventory playerInv, FriendlyByteBuf extraData) {
    super(MY_MENU_EXTRA.get(), containerId);
    // 从缓冲区存储额外数据
    // ...
}
```

## `AbstractContainerMenu`

所有菜单均继承`AbstractContainerMenu`。菜单接收两个参数：[`MenuType`][mt]（表示菜单自身类型）和容器 ID（表示当前访问者菜单的唯一标识符）。

:::note
菜单标识符在 0-99 间循环，玩家每次打开菜单时递增。
:::

每个菜单应包含两个构造函数：一个在服务器初始化菜单，另一个在客户端初始化。传入`MenuType`的是客户端菜单构造函数。服务器菜单构造函数的字段应在客户端菜单构造函数中有默认值。

```java
// 客户端菜单构造函数
public MyMenu(int containerId, Inventory playerInventory) { // 可选 FriendlyByteBuf 参数（若从服务器读取数据）
    this(containerId, playerInventory, /* 默认参数 */);
}

// 服务器菜单构造函数
public MyMenu(int containerId, Inventory playerInventory, /* 额外参数 */) {
    // ...
}
```

:::note
若菜单无需显示额外数据，则仅需一个构造函数。
:::

每个菜单实现必须实现两个方法：`#stillValid`和[`#quickMoveStack`][qms]。

### `#stillValid` 与 `ContainerLevelAccess`

`#stillValid`决定菜单是否应为给定玩家保持打开。通常调用静态`#stillValid`方法（传入`ContainerLevelAccess`、玩家和菜单关联的`Block`）。客户端菜单此方法必须始终返回`true`（静态`#stillValid`默认如此）。此实现检查玩家是否在数据存储对象的 8 格范围内。

`ContainerLevelAccess`在封闭作用域内提供当前世界和方块位置。在服务器构建菜单时，可通过`ContainerLevelAccess#create`创建新访问器。客户端菜单构造函数可传入`ContainerLevelAccess#NULL`（无操作）。

```java
// 客户端菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory) {
    this(containerId, playerInventory, ContainerLevelAccess.NULL);
}

// 服务器菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory, ContainerLevelAccess access) {
    // ...
}

// 假设此菜单关联到 Supplier<Block> MY_BLOCK
@Override
public boolean stillValid(Player player) {
    return AbstractContainerMenu.stillValid(this.access, player, MY_BLOCK.get());
}
```

### **数据同步**(`Data Synchronization`)

部分数据需同时在服务器和客户端存在以展示给玩家。为此，菜单实现基础数据同步层，确保当前数据与上次同步到客户端的数据匹配。对玩家而言，此检查每刻执行。

Minecraft 默认支持两种数据同步形式：通过`Slot`同步`ItemStack`，通过`DataSlot`同步整数。`Slot`和`DataSlot`是持有数据存储引用的视图，玩家可在屏幕中修改（假设操作有效）。可通过构造函数中的`#addSlot`和`#addDataSlot`添加到菜单。

:::note
因 NeoForge 弃用`Slot`使用的`Container`（改用[`IItemHandler`能力][cap]），后续说明围绕能力变体`SlotItemHandler`展开。
:::

`SlotItemHandler`包含四个参数：表示物品堆所在物品栏的`IItemHandler`、此槽位特定表示的物品堆索引，以及槽位在屏幕上渲染的左上角位置（相对于`AbstractContainerScreen#leftPos`和`#topPos`）。客户端菜单构造函数应始终提供同尺寸的空物品栏实例。

多数情况下，先添加菜单包含的所有槽位，再添加玩家物品栏，最后添加玩家快捷栏。要从菜单访问单个`Slot`，必须根据槽位添加顺序计算索引。

`DataSlot`是抽象类，应实现 getter 和 setter 引用数据存储对象中的数据。客户端菜单构造函数应始终通过`DataSlot#standalone`提供新实例。

这些（连同槽位）应在每次初始化新菜单时重建。

:::note
尽管`DataSlot`存储整数，但因网络传输方式限制，实际范围为**短整型**（-32768 至 32767）。整数的高 16 位被忽略。

NeoForge 修补数据包以向客户端提供完整整数。
:::

```java
// 假设数据对象有尺寸为 5 的物品栏
// 假设每次初始化服务器菜单时构建 DataSlot

// 客户端菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory) {
    this(containerId, playerInventory, new ItemStackHandler(5), DataSlot.standalone());
}

// 服务器菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory, IItemHandler dataInventory, DataSlot dataSingle) {
    // 检查数据物品栏尺寸是否为固定值
    // 然后为数据物品栏添加槽位
    this.addSlot(new SlotItemHandler(dataInventory, /*...*/));

    // 为玩家物品栏添加槽位
    this.addSlot(new Slot(playerInventory, /*...*/));

    // 为处理的整数添加数据槽位
    this.addDataSlot(dataSingle);

    // ...
}
```

#### `ContainerData`

若需同步多个整数到客户端，可用`ContainerData`引用这些整数。此接口作为索引查找表（每个索引代表不同整数）。若通过`#addDataSlots`将`ContainerData`添加到菜单，也可在数据对象自身构建。此方法为接口指定的数据量创建新`DataSlot`。客户端菜单构造函数应始终通过`SimpleContainerData`提供新实例。

```java
// 假设有尺寸为 3 的 ContainerData

// 客户端菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory) {
    this(containerId, playerInventory, new SimpleContainerData(3));
}

// 服务器菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory, ContainerData dataMultiple) {
    // 检查 ContainerData 尺寸是否为固定值
    checkContainerDataCount(dataMultiple, 3);

    // 为处理的整数添加数据槽位
    this.addDataSlots(dataMultiple);

    // ...
}
```

#### `#quickMoveStack`

`#quickMoveStack`是菜单必须实现的第二个方法。当物品堆被 Shift 点击或快速移出当前槽位时调用（直至物品堆完全移出原槽位或无其他可移动位置）。方法返回被快速移动槽位中物品堆的副本。

通常使用`#moveItemStackTo`在槽位间移动物品堆（将物品堆移到首个可用槽位）。接收待移动物品堆、尝试移动的起始槽位索引（含）、结束槽位索引（不含），以及是否从前向后检查槽位（`false`时）或从后向前（`true`时）。

Minecraft 实现中此方法逻辑高度一致：

```java
// 假设数据物品栏尺寸为 5
// 物品栏有 4 个输入槽（索引 1-4）和 1 个输出槽（索引 0）
// 另有 27 个玩家物品栏槽位和 9 个快捷栏槽位
// 因此实际槽位索引如下：
//   - 数据物品栏：输出槽（0）、输入槽（1-4）
//   - 玩家物品栏（5-31）
//   - 玩家快捷栏（32-40）
@Override
public ItemStack quickMoveStack(Player player, int quickMovedSlotIndex) {
    // 被快速移动槽位的物品堆
    ItemStack quickMovedStack = ItemStack.EMPTY;
    // 被快速移动的槽位
    Slot quickMovedSlot = this.slots.get(quickMovedSlotIndex);
  
    // 若槽位在有效范围内且非空
    if (quickMovedSlot != null && quickMovedSlot.hasItem()) {
        // 获取待移动原始物品堆
        ItemStack rawStack = quickMovedSlot.getItem(); 
        // 设置槽位物品堆为原始物品堆的副本
        quickMovedStack = rawStack.copy();

    /*
    以下快速移动逻辑可简化为：若在数据物品栏中，
    尝试移到玩家物品栏/快捷栏；反之亦然（适用于无法转换数据的容器，如箱子）。
    */

        // 若在数据物品栏输出槽执行快速移动
        if (quickMovedSlotIndex == 0) {
            // 尝试将输出槽移到玩家物品栏/快捷栏
            if (!this.moveItemStackTo(rawStack, 5, 41, true)) {
                // 若无法移动，停止快速移动
                return ItemStack.EMPTY;
            }

            // 在输出槽快速移动后执行逻辑
            quickMovedSlot.onQuickCraft(rawStack, quickMovedStack);
        }
        // 若在玩家物品栏或快捷栏槽位执行快速移动
        else if (quickMovedSlotIndex >= 5 && quickMovedSlotIndex < 41) {
            // 尝试将物品栏/快捷栏槽位移到数据物品栏输入槽
            if (!this.moveItemStackTo(rawStack, 1, 5, false)) {
                // 若无法移动且在玩家物品栏槽位，尝试移到快捷栏
                if (quickMovedSlotIndex < 32) {
                    if (!this.moveItemStackTo(rawStack, 32, 41, false)) {
                        // 若无法移动，停止快速移动
                        return ItemStack.EMPTY;
                    }
                }
                // 否则尝试将快捷栏移到玩家物品栏槽位
                else if (!this.moveItemStackTo(rawStack, 5, 32, false)) {
                    // 若无法移动，停止快速移动
                    return ItemStack.EMPTY;
                }
            }
        }
        // 若在数据物品栏输入槽执行快速移动，尝试移到玩家物品栏/快捷栏
        else if (!this.moveItemStackTo(rawStack, 5, 41, false)) {
            // 若无法移动，停止快速移动
            return ItemStack.EMPTY;
        }

        if (rawStack.isEmpty()) {
            // 若原始物品堆已完全移出槽位，将槽位设为空物品堆
            quickMovedSlot.setByPlayer(ItemStack.EMPTY);
        } else {
            // 否则通知槽位物品堆数量已变更
            quickMovedSlot.setChanged();
        }

        /*
        若菜单不代表可转换物品堆的容器（如箱子），
        可移除以下 if 语句和 Slot#onTake 调用。
        */
        if (rawStack.getCount() == quickMovedStack.getCount()) {
            // 若原始物品堆无法移到其他槽位，停止快速移动
            return ItemStack.EMPTY;
        }
        // 在移动后对剩余物品堆执行逻辑
        quickMovedSlot.onTake(player, rawStack);
    }

    return quickMovedStack; // 返回槽位物品堆
}
```

## 打开菜单

注册菜单类型、完成菜单并附加[**屏幕**(`screen`)][screen]后，玩家即可打开菜单。在**逻辑服务器**（`logical server`）调用`IPlayerExtension#openMenu`打开菜单。方法接收服务器菜单的`MenuProvider`，若需同步额外数据到客户端，可选传入`Consumer<RegistryFriendlyByteBuf>`。

:::note
仅当菜单类型使用[`IContainerFactory`][icf]创建时，才应使用带`Consumer<RegistryFriendlyByteBuf>`参数的`IPlayerExtension#openMenu`。
:::

#### `MenuProvider`

`MenuProvider`是包含两个方法的接口：`#createMenu`（创建菜单的服务器实例）和`#getDisplayName`（返回传递给[**屏幕**(`screen`)][screen]的菜单标题组件）。`#createMenu`方法接收三个参数：菜单容器 ID、打开菜单玩家的物品栏和打开菜单的玩家。

使用`SimpleMenuProvider`可轻松创建`MenuProvider`，其接收创建服务器菜单的方法引用和菜单标题。

```java
// 在可访问逻辑服务器上 Player 的某实现中（如 ServerPlayer 实例）
// 假设有 ServerPlayer serverPlayer
serverPlayer.openMenu(new SimpleMenuProvider(
    (containerId, playerInventory, player) -> new MyMenu(containerId, playerInventory, /* 服务器参数 */),
    Component.translatable("menu.title.examplemod.mymenu")
));
```

### 常见实现

菜单通常在某种玩家交互时打开（如右键点击方块或实体）。

#### 方块实现

方块通常通过重写`BlockBehaviour#useWithoutItem`实现菜单，为[**交互**(`interaction`)][interaction]返回`InteractionResult#SUCCESS`。

应通过重写`BlockBehaviour#getMenuProvider`实现`MenuProvider`。原版方法使用此功能在旁观模式查看菜单。

```java
// 在某个 Block 子类中
@Override
public MenuProvider getMenuProvider(BlockState state, Level level, BlockPos pos) {
    return new SimpleMenuProvider(/* ... */);
}

@Override
public InteractionResult useWithoutItem(BlockState state, Level level, BlockPos pos, Player player, InteractionHand hand, BlockHitResult result) {
    if (!level.isClientSide && player instanceof ServerPlayer serverPlayer) {
        serverPlayer.openMenu(state.getMenuProvider(level, pos));
    }

    return InteractionResult.SUCCESS;
}
```

:::note
此为实现逻辑的最简方式，非唯一方式。若需方块仅在特定条件下打开菜单，则需预先同步数据到客户端，条件不满足时返回`InteractionResult#PASS`或`#FAIL`。
:::

#### 生物实现

生物通常通过重写`Mob#mobInteract`实现菜单。与方块实现类似，区别在于`Mob`自身应实现`MenuProvider`以支持旁观模式查看。

```java
public class MyMob extends Mob implements MenuProvider {
    // ...

    @Override
    public InteractionResult mobInteract(Player player, InteractionHand hand) {
        if (!this.level.isClientSide && player instanceof ServerPlayer serverPlayer) {
            serverPlayer.openMenu(this);
        }

        return InteractionResult.SUCCESS;
    }
}
```

:::note
此再次为实现逻辑的最简方式，非唯一方式。
:::

[registered]: ../concepts/registries.md#注册方法
[acm]: #abstractcontainermenu
[mt]: #menutype
[qms]: #quickmovestack
[cap]: ../datastorage/capabilities.md#neoforge-提供的能力
[screen]: screens.md
[icf]: #icontainerfactory
[side]: ../concepts/sides.md#逻辑端
[interaction]: ../items/interactions.md#右键点击物品