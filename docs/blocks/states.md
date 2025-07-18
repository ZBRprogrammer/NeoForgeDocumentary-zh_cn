# **方块状态**(`Blockstates`)

通常，您会遇到需要方块具有不同状态的情况。例如，小麦作物有八个生长阶段，为每个阶段创建单独的方块感觉不妥。或者您有台阶或类似台阶的方块——一种底部状态、一种顶部状态，以及一种同时具有两者的状态。

这正是**方块状态**(`Blockstates`)的用武之地。方块状态是表示方块可能具有不同状态（如生长阶段或台阶放置类型）的简单方式。

## **方块状态属性**(`Blockstate Properties`)

方块状态使用属性系统。一个方块可以具有多种类型的多个属性。例如，末地传送门框架有两个属性：是否有末影之眼（`eye`，2 个选项）和放置方向（`facing`，4 个选项）。因此，末地传送门框架总共有 8 种（2 * 4）不同的方块状态：

```
minecraft:end_portal_frame[facing=north,eye=false]
minecraft:end_portal_frame[facing=east,eye=false]
minecraft:end_portal_frame[facing=south,eye=false]
minecraft:end_portal_frame[facing=west,eye=false]
minecraft:end_portal_frame[facing=north,eye=true]
minecraft:end_portal_frame[facing=east,eye=true]
minecraft:end_portal_frame[facing=south,eye=true]
minecraft:end_portal_frame[facing=west,eye=true]
```

`blockid[property1=value1,property2=value,...]` 表示法是文本形式表示方块状态的标准化方式，在原版的某些位置使用（如命令中）。

若方块未定义任何方块状态属性，它仍有一个方块状态——即没有任何属性的状态（因为没有属性可指定）。可表示为 `minecraft:oak_planks[]` 或简写为 `minecraft:oak_planks`。

与方块类似，每个 `BlockState` 在内存中仅存在一次。这意味着可使用 `==` 比较 `BlockState`。`BlockState` 也是最终类(`final class`)，意味着无法扩展。**任何功能都应放在对应的[方块][block]类中！**

## 何时使用方块状态

### **方块状态 vs. 独立方块**

良好经验法则是：**若名称不同，应为独立方块**。例如制作椅子方块：椅子方向应为属性，而不同木材类型应分离为独立方块。因此每种木材类型有一个椅子方块，每个椅子方块有四种方块状态（每个方向一种）。

### **方块状态 vs. [方块实体][blockentity]**

此处经验法则是：**若状态数量有限，使用方块状态；若状态数量无限或接近无限，使用方块实体**。方块实体可存储任意数据，但比方块状态慢。

方块状态和方块实体可结合使用。例如，箱子使用方块状态属性处理方向、是否含水或成为双箱，而存储库存、当前是否打开或与漏斗交互则由方块实体处理。

"方块状态多少算过多？"没有明确答案，但若需要超过 8-9 位数据（即超过几百种状态），建议改用方块实体。

## 实现方块状态

要实现方块状态属性，需在方块类中创建或引用 `public static final Property<?>` 常量。虽然可自定义 `Property<?>` 实现，但原版代码提供了几种便捷实现，应涵盖大多数用例：

- `IntegerProperty`
    - 实现 `Property<Integer>`。定义保存整数值的属性。注意不支持负值
    - 通过 `IntegerProperty#create(String propertyName, int minimum, int maximum)` 创建
- `BooleanProperty`
    - 实现 `Property<Boolean>`。定义保存 `true` 或 `false` 值的属性
    - 通过 `BooleanProperty#create(String propertyName)` 创建
- `EnumProperty<E extends Enum<E>>`
    - 实现 `Property<E>`。定义可接受枚举类值的属性
    - 通过 `EnumProperty#create(String propertyName, Class<E> enumClass)` 创建
    - 也可仅使用枚举值的子集（如 16 种染料颜色中的 4 种），参见 `EnumProperty#create` 的重载

`BlockStateProperties` 类包含共享的原版属性，应尽可能使用或引用，而非创建自定义属性。

拥有属性常量后，在方块类中重写 `Block#createBlockStateDefinition(StateDefinition.Builder)`。在此方法中调用 `StateDefinition.Builder#add(YOUR_PROPERTY);`。`StateDefinition.Builder#add` 具有可变参数，因此若有多个属性，可一次性添加所有。

每个方块也有默认状态。若未指定其他内容，默认状态使用每个属性的默认值。可在构造函数中调用 `Block#registerDefaultState(BlockState)` 方法更改默认状态。

若希望更改放置方块时使用的 `BlockState`，请重写 `Block#getStateForPlacement(BlockPlaceContext)`。例如，可根据玩家放置时的位置或视线方向设置方块方向。

为进一步说明，以下是 `EndPortalFrameBlock` 类的相关部分：

```java
public class EndPortalFrameBlock extends Block {
    // 注意：可直接使用 BlockStateProperties 中的值，而无需在此处重新引用
    // 但为简洁性和可读性，建议添加此类常量
    public static final EnumProperty<Direction> FACING = BlockStateProperties.FACING;
    public static final BooleanProperty EYE = BlockStateProperties.EYE;

    public EndPortalFrameBlock(BlockBehaviour.Properties pProperties) {
        super(pProperties);
        // stateDefinition.any() 从内部集合返回随机 BlockState
        // 我们不关心，因为无论如何都会自己设置所有值
        this.registerDefaultState(stateDefinition.any()
                .setValue(FACING, Direction.NORTH)
                .setValue(EYE, false)
        );
    }

    @Override
    protected void createBlockStateDefinition(StateDefinition.Builder<Block, BlockState> pBuilder) {
        // 属性实际添加到状态的位置
        pBuilder.add(FACING, EYE);
    }

    @Override
    @Nullable
    public BlockState getStateForPlacement(BlockPlaceContext pContext) {
        // 根据 BlockPlaceContext 确定放置此方块时将使用的状态的代码
    }
}
```

## 使用方块状态

要从 `Block` 获取 `BlockState`，调用 `Block#defaultBlockState()`。如上所述，可通过 `Block#registerDefaultState` 更改默认方块状态。

通过调用 `BlockState#getValue(Property<?>)` 并传入要获取值的属性，可获取属性的值。以末地传送门框架为例：

```java
// EndPortalFrameBlock.FACING 是 EnumPropery<Direction>，因此可用于从 BlockState 获取 Direction
Direction direction = endPortalFrameBlockState.getValue(EndPortalFrameBlock.FACING);
```

若希望获取具有不同值集的 `BlockState`，只需在现有方块状态上调用 `BlockState#setValue(Property<T>, T)` 并传入属性及其值。以末地传送门框架为例：

```java
endPortalFrameBlockState = endPortalFrameBlockState.setValue(EndPortalFrameBlock.FACING, Direction.SOUTH);
```

:::note
`BlockState`s 是**不可变的**(`immutable`)。这意味着调用 `#setValue(Property<T>, T)` 时并未实际修改方块状态，而是在内部执行查找并返回您请求的方块状态对象（即具有这些确切属性值的唯一对象）。这也意味着仅调用 `state#setValue` 而不保存到变量（例如存回 `state`）中无效。
:::

要从世界中获取 `BlockState`，使用 `Level#getBlockState(BlockPos)`。

### **Level#setBlock**

要在世界中设置 `BlockState`，使用 `Level#setBlock(BlockPos, BlockState, int)`。

`int` 参数值得额外解释，因其含义并不直观。它表示**更新标志**(`update flags`)。

为帮助正确设置更新标志，`Block` 中有多个以 `UPDATE_` 为前缀的 `int` 常量。若希望组合，可对这些常量进行位或运算（例如 `Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS`）。

- `Block.UPDATE_NEIGHBORS` 向相邻方块发送更新。更具体地说，调用 `Block#neighborChanged`，进而调用若干方法（其中多数与红石相关）
- `Block.UPDATE_CLIENTS` 将方块更新同步到客户端
- `Block.UPDATE_INVISIBLE` 明确不在客户端更新。这会覆盖 `Block.UPDATE_CLIENTS`，导致更新不同步。方块始终在服务端更新
- `Block.UPDATE_IMMEDIATE` 强制在客户端主线程重新渲染
- `Block.UPDATE_KNOWN_SHAPE` 停止相邻更新递归
- `Block.UPDATE_SUPPRESS_DROPS` 禁用该位置旧方块的掉落
- `Block.UPDATE_MOVE_BY_PISTON` 仅由活塞代码使用，表示方块被活塞移动。主要负责延迟光照引擎更新
- `Block.UPDATE_SKIP_SHAPE_UPDATE_ON_WIRE` 由 `ExperimentalRedstoneWireEvaluator` 使用，表示是否跳过形状更新。仅在信号强度非放置更改或信号源非当前红石线时设置
- `Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS` 阻止调用 `BlockEntity#preRemoveSideEffects`。通常防止方块实体清空其内容
- `Block.UPDATE_SKIP_ON_PLACE` 阻止调用 `Block#onPlace`。通常防止任何方块处理其初始行为（例如更新铁轨附着到其他方块、生成铁傀儡）
- `Block.UPDATE_NONE` 是 `Block.UPDATE_INVISIBLE | Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS` 的别名
- `Block.UPDATE_ALL` 是 `Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS` 的别名
- `Block.UPDATE_ALL_IMMEDIATE` 是 `Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS | Block.UPDATE_IMMEDIATE` 的别名
- `Block.UPDATE_SKIP_ALL_SIDEEFFECTS` 是 `Block.UPDATE_SKIP_ON_PLACE | Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS | Block.UPDATE_SUPPRESS_DROPS | Block.UPDATE_KNOWN_SHAPE` 的别名

还有一个便捷方法 `Level#setBlockAndUpdate(BlockPos pos, BlockState state)`，内部调用 `setBlock(pos, state, Block.UPDATE_ALL)`。

[block]: index.md
[blockentity]: ../blockentities/index.md