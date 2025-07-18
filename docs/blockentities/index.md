# **方块实体**(`Block Entities`)

**方块实体**(`Block entities`)允许在[方块][block]上存储[方块状态][blockstate]不适用时的数据。这对于具有无限选项的数据（如库存）尤其重要。方块实体是静止的且与方块绑定，但在其他方面与[实体(`entities`)][entities]有许多相似之处，因此得名。

:::note
如果您的方块可能状态数量有限且合理（最多几百种），可考虑改用[方块状态][blockstate]。
:::

## 创建与注册方块实体

与实体类似但不同于方块，`BlockEntity` 类表示方块实体实例，而非[注册][registration]的单例对象。单例通过 `BlockEntityType<?>` 类表示。我们需要两者来创建新的方块实体。

首先创建方块实体类：

```java
public class MyBlockEntity extends BlockEntity {
    public MyBlockEntity(BlockPos pos, BlockState state) {
        super(type, pos, state);
    }
}
```

如您所见，我们将未定义的变量 `type` 传递给超类构造函数。暂时保留该变量，先进行注册。

[注册][registration]过程与实体类似。我们创建关联的单例类 `BlockEntityType<?>` 的实例，并将其注册到方块实体类型注册表：

```java
public static final DeferredRegister<BlockEntityType<?>> BLOCK_ENTITY_TYPES =
        DeferredRegister.create(Registries.BLOCK_ENTITY_TYPE, ExampleMod.MOD_ID);

public static final Supplier<BlockEntityType<MyBlockEntity>> MY_BLOCK_ENTITY = BLOCK_ENTITY_TYPES.register(
        "my_block_entity",
        // 方块实体类型
        () -> new BlockEntityType<>(
                // 用于构造方块实体实例的提供器
                MyBlockEntity::new,
                // 可选值，为 true 时仅允许具有 OP 权限的玩家
                // 加载 NBT 数据（如放置方块物品）
                false,
                // 可拥有此方块实体的方块可变参数
                // 假设引用的方块作为 DeferredBlock<Block> 存在
                MyBlocks.MY_BLOCK_1.get(), MyBlocks.MY_BLOCK_2.get()
        )
);
```

:::note
请记住 `DeferredRegister` 必须注册到[模组事件总线][modbus]！
:::

现在我们有了方块实体类型，可用它替换之前保留的 `type` 变量：

```java
public class MyBlockEntity extends BlockEntity {
    public MyBlockEntity(BlockPos pos, BlockState state) {
        super(MY_BLOCK_ENTITY.get(), pos, state);
    }
}
```

:::info
这种看似混乱的设置过程的原因是 `BlockEntityType` 期望 `BlockEntityType.BlockEntitySupplier<T extends BlockEntity>`（本质上是 `BiFunction<BlockPos, BlockState, T extends BlockEntity>`）。因此，使用 `::new` 直接引用的构造函数非常有益。但我们也需要将构造的方块实体类型传递给 `BlockEntity` 的默认且唯一的构造函数，因此需要传递引用。
:::

最后，我们需要修改与方块实体关联的方块类。这意味着无法将方块实体附加到简单的 `Block` 实例，而需要子类：

```java
// 关键部分是实现 EntityBlock 接口并重写 #newBlockEntity 方法
public class MyEntityBlock extends Block implements EntityBlock {
    // 构造函数委托给超类
    public MyEntityBlock(BlockBehaviour.Properties properties) {
        super(properties);
    }

    // 在此返回方块实体的新实例
    @Override
    public BlockEntity newBlockEntity(BlockPos pos, BlockState state) {
        return new MyBlockEntity(pos, state);
    }
}
```

然后在[方块注册][blockreg]中使用此类作为类型：

```java
public static final DeferredBlock<MyEntityBlock> MY_BLOCK_1 =
        BLOCKS.register("my_block_1", () -> new MyEntityBlock( /* ... */ ));
public static final DeferredBlock<MyEntityBlock> MY_BLOCK_2 =
        BLOCKS.register("my_block_2", () -> new MyEntityBlock( /* ... */ ));
```

## 存储数据

`BlockEntity` 的主要目的之一是存储数据。方块实体上的数据存储可通过两种方式实现：读写[值 I/O][valueio]或使用[数据附加][dataattachments]。本节将介绍读写值 I/O；数据附加请参阅链接文章。

:::info
数据附加的主要目的是将数据附加到现有方块实体（如原版或其他模组提供的方块实体）。对于您自己模组的方块实体，建议直接通过值 I/O 保存和加载。
:::

分别使用 `#loadAdditional` 和 `#saveAdditional` 方法从[值 I/O][valueio]读取和写入数据。这些方法在方块实体同步到磁盘或网络时调用。

```java
public class MyBlockEntity extends BlockEntity {
    // 可以是您想要的任何类型的值，只要可以序列化到值 I/O
    // 为示例使用整数
    private int value;

    public MyBlockEntity(BlockPos pos, BlockState state) {
        super(MY_BLOCK_ENTITY.get(), pos, state);
    }

    // 从传入的 ValueInput 读取值
    @Override
    public void loadAdditional(ValueInput input) {
        super.loadAdditional(input);
        // 若不存在则默认为 0。更多信息参见值 I/O 文章
        this.value = input.getIntOr("value", 0);
    }

    // 将值保存到传入的 ValueOutput
    @Override
    public void saveAdditional(ValueOutput output) {
        super.saveAdditional(output);
        output.putInt("value", this.value);
    }
}
```

在这两个方法中，调用超类方法很重要，因为它添加了位置等基本信息。标签名 `id`、`x`、`y`、`z`、`NeoForgeData` 和 `neoforge:attachments` 被超类方法保留，因此不应自行使用。

当然，您会希望设置其他值而不仅使用默认值。可像任何其他字段一样自由操作。但如果希望游戏保存更改，必须在之后调用 `#setChanged()`，这将标记方块实体的区块为脏（需要保存）。若不调用此方法，方块实体可能在保存过程中被跳过，因为 Minecraft 的保存系统仅保存标记为脏的区块。

### 移除方块实体

有时可能需要在移除时导出方块实体存储的数据（例如在玩家破坏时掉落库存）。此类逻辑应在 `BlockEntity#preRemoveSideEffects` 内处理。默认情况下，若方块实体实现了 [`Container`][container]，则会掉落存储的内容。

```java
public class MyBlockEntity extends BlockEntity {

    @Override
    public void preRemoveSideEffects(BlockPos pos, BlockState state) {
        super.preRemoveSideEffects(pos, state);
        // 在此执行移除时的剩余导出逻辑
    }
}
```

:::warning
使用 `Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS` 标志移除的方块不会调用此方法。通常在使用克隆命令或以严格模式放置结构时发生。
:::

若相邻方块需要知晓方块实体被破坏（例如通过比较器输出红石信号的库存），则方块应重写 `BlockBehaviour#affectNeighborsAfterRemoval`。输出红石信号的方块实体通常在此处调用 `Containers#updateNeighboursAfterDestroy`。

```java
public class MyEntityBlock extends Block implements EntityBlock {

    @Override
    protected void affectNeighborsAfterRemoval(BlockState state, ServerLevel level, BlockPos pos, boolean movedByPiston) {
        // 处理希望在周围邻居上执行的任何逻辑
        Containers.updateNeighboursAfterDestroy(state, level, pos);
    }
}
```

## 刻处理器(`Tickers`)

方块实体的另一常见用途（常与存储数据结合）是刻处理(`ticking`)。刻处理意味着每游戏刻执行某些代码。通过重写 `EntityBlock#getTicker` 并返回 `BlockEntityTicker`（本质上是具有四个参数（世界、位置、方块状态和方块实体）的消费者）实现：

```java
// 注意：刻处理器在方块中定义，而非方块实体。但良好实践是
// 以某种方式将刻处理逻辑保留在方块实体中，例如定义静态 #tick 方法
public class MyEntityBlock extends Block implements EntityBlock {
    // 其他内容

    @SuppressWarnings("unchecked") // 由于泛型，此处需要未检查转换
    @Override
    public <T extends BlockEntity> BlockEntityTicker<T> getTicker(Level level, BlockState state, BlockEntityType<T> type) {
        // 可根据需要返回不同的刻处理器。常见用例是
        // 在客户端或服务端返回不同的刻处理器，仅在一端刻处理，
        // 或仅对某些方块状态返回刻处理器（例如使用"我的机器正在工作"方块状态属性时）
        return type == MY_BLOCK_ENTITY.get() ? (BlockEntityTicker<T>) MyBlockEntity::tick : null;
    }
}

public class MyBlockEntity extends BlockEntity {
    // 其他内容

    // 此方法签名与 BlockEntityTicker 函数接口匹配
    public static void tick(Level level, BlockPos pos, BlockState state, MyBlockEntity blockEntity) {
        // 刻处理期间希望执行的任何操作
        // 例如，可在此处更改合成进度值或消耗能量
    }
}
```

请注意 `#tick` 方法实际每刻调用。因此应避免在此进行大量复杂计算，例如每 X 刻计算一次或缓存结果。

## 同步

方块实体逻辑通常在服务端运行。因此需要告知客户端操作内容。有三种方式实现：区块加载时、方块更新时或使用自定义数据包。通常应仅在必要时同步信息，避免不必要地阻塞网络。

### 区块加载时同步

区块每次从网络或磁盘加载时（即使用此方法）调用。要在此发送数据，需重写以下方法：

```java
public class MyBlockEntity extends BlockEntity {
    // ...

    // 在此创建更新标签。对于只有少量字段的方块实体，可直接调用 #saveWithoutMetadata
    @Override
    public CompoundTag getUpdateTag(HolderLookup.Provider registries) {
        return this.saveWithoutMetadata(registries);
    }

    // 在此处理接收的更新标签。默认实现调用 #loadWithComponents，
    // 因此若不计划执行额外操作则无需重写此方法
    @Override
    public void handleUpdateTag(ValueInput input) {
        super.handleUpdateTag(input);
    }
}
```

### 方块更新时同步

此方法在方块更新时使用。方块更新必须手动触发，但通常比区块同步处理更快。

```java
public class MyBlockEntity extends BlockEntity {
    // ...

    // 在此创建更新标签，如上所述
    @Override
    public CompoundTag getUpdateTag(HolderLookup.Provider registries) {
        return this.saveWithoutMetadata(registries);
    }

    // 在此返回数据包。此方法返回非 null 结果告知游戏使用此数据包同步
    @Override
    public Packet<ClientGamePacketListener> getUpdatePacket() {
        // 数据包使用 #getUpdateTag 返回的 CompoundTag。存在 #create 的替代重载
        // 允许指定自定义更新标签，包括省略客户端可能不需要的数据
        return ClientboundBlockEntityDataPacket.create(this);
    }

    // 可选：数据包接收时执行自定义逻辑
    // 超类/默认实现转发到 #loadWithComponents
    @Override
    public void onDataPacket(Connection connection, ValueInput input) {
        super.onDataPacket(connection, input);
        // 在此执行所需操作
    }
}
```

要实际发送数据包，必须在服务端通过调用 `Level#sendBlockUpdated(BlockPos pos, BlockState oldState, BlockState newState, int flags)` 触发更新通知。位置应为方块实体的位置（通过 `BlockEntity#getBlockPos` 获取）。两个方块状态参数可以是方块实体位置的方块状态（通过 `BlockEntity#getBlockState` 获取）。最后，`flags` 参数是更新掩码，如 [`Level#setBlock`][setblock] 所用。

### 使用自定义数据包

通过使用专用更新数据包，可在需要时自行发送数据包。这是最灵活但也最复杂的变体，需要设置网络处理器。可使用 `PacketDistrubtor#sendToPlayersTrackingChunk` 向所有追踪方块实体的玩家发送数据包。更多信息请参阅[网络][networking]部分。

:::caution
安全检查很重要，因为数据包到达玩家时 `BlockEntity` 可能已被销毁/替换。还应通过 `Level#hasChunkAt` 检查区块是否加载。
:::

[block]: ../blocks/index.md
[blockreg]: ../blocks/index.md#basic-blocks
[blockstate]: ../blocks/states.md
[container]: container.md
[dataattachments]: ../datastorage/attachments.md
[entities]: ../entities/index.md
[modbus]: ../concepts/events.md#event-buses
[networking]: ../networking/index.md
[registration]: ../concepts/registries.md#methods-for-registering
[setblock]: ../blocks/states.md#levelsetblock
[valueio]: ../datastorage/valueio.md