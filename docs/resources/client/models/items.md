import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# **客户端物品**(`Client Items`)

**客户端物品**(`Client Items`)是游戏中表示`ItemStack`应如何渲染的代码表示形式，指定在给定状态下应使用的模型。客户端物品位于[`assets`文件夹][assets]的`items`子目录中，由`DataComponents#ITEM_MODEL`内的相对位置指定。默认情况下，这是对象的注册名称（例如`minecraft:apple`默认位于`assets/minecraft/items/apple.json`）。

客户端物品存储在`ModelManager`中，可通过`Minecraft.getInstance().modelManager`访问。然后可调用`ModelManager#getItemModel`或`getItemProperties`通过[`ResourceLocation`][rl]获取客户端物品信息。

:::warning
注意不要与游戏中实际烘焙和渲染的[模型][models]混淆。
:::

## 概述(`Overview`)

客户端物品的JSON可分为两部分：由`model`定义的模型；以及由`properties`定义的属性。`model`负责定义在给定上下文中渲染`ItemStack`时使用的模型JSON。`properties`则负责渲染器使用的设置。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    // 定义要渲染的模型
    "model": {
        "type": "minecraft:model",
        // 指向相对于'models'目录的模型JSON
        // 位于'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item"
    },
    // 定义渲染过程中使用的设置
    "properties": {
        // 为false时禁用物品切换时物品升至正常位置的动画
        "hand_animation_on_swap": false,
        // 为true时允许模型在GUI中渲染到其定义的插槽边界外
        // (由GuiItemRenderState#bounds定义)而非被裁剪
        "oversized_in_gui": false
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.register(
        EXAMPLE_ITEM.get(),
        new ClientItem(
            // 定义要渲染的模型
            new BlockModelWrapper.Unbaked(
                // 指向相对于'models'目录的模型JSON
                // 位于'assets/examplemod/models/item/example_item.json'
                ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                Collections.emptyList()
            ),
            // 定义渲染过程中使用的设置
            new ClientItem.Properties(
                // 为false时禁用物品切换时物品升至正常位置的动画
                false
            )
        )
    );
}
```

</TabItem>
</Tabs>

关于物品模型如何渲染的更多信息见[下文][itemmodel]。

## 基础模型(`A Basic Model`)

`model`内的`type`字段决定如何为物品选择渲染模型。最简单的类型由`minecraft:model`（或`BlockModelWrapper`）处理，功能上定义要渲染的模型JSON（相对于`models`目录，如`assets/<命名空间>/models/<路径>.json`）。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:model",
        // 指向相对于'models'目录的模型JSON
        // 位于'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item"
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new BlockModelWrapper.Unbaked(
            // 指向相对于'models'目录的模型JSON
            // 位于'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            Collections.emptyList()
        )
    );
}
```

</TabItem>
</Tabs>

### 着色(`Tinting`)

与大多数模型类似，客户端物品可根据堆叠属性改变指定纹理的颜色。因此`minecraft:model`类型有`tints`字段定义要应用的不透明颜色。这些称为`ItemTintSource`，在`ItemTintSources`中定义，并通过`type`字段指定要使用的源。它们应用的`tintindex`由其在列表中的索引指定。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:model",
        // 指向'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item",
        // 要应用的着色列表
        "tints": [
            {
                // 当tintindex为0时
                "type": "minecraft:constant",
                // 0x00FF00（纯绿色）
                "value": 65280
            },
            {
                // 当tintindex为1时
                "type": "minecraft:dye",
                // 0x0000FF（纯蓝色）
                // 仅在未设置`DataComponents#DYED_COLOR`时调用
                "default": 255
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new BlockModelWrapper.Unbaked(
            // 指向'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            // 要应用的着色列表
            List.of(
                // 当tintindex为0时
                new Constant(
                    // 纯绿色
                    0x00FF00
                ),
                // 当tintindex为1时
                new Dye(
                    // 纯蓝色
                    // 仅在未设置`DataComponents#DYED_COLOR`时调用
                    0x0000FF
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建自定义`ItemTintSource`类似于其他基于编解码器(`codec`)的注册对象。创建实现`ItemTintSource`的类，创建用于编码/解码对象的`MapCodec`，并通过[模组事件总线(`mod bus`)][modbus]上的`RegisterColorHandlersEvent.ItemTintSources`将编解码器注册到注册表中。`ItemTintSource`仅包含一个方法`calculate`，接收当前`ItemStack`、堆叠所在世界、持有堆叠的实体，返回ARGB格式的不透明颜色（高8位为0xFF）。

```java
public record DamageBar(int defaultColor) implements ItemTintSource {

    // 要注册的MapCodec
    public static final MapCodec<DamageBar> MAP_CODEC = ExtraCodecs.RGB_COLOR_CODEC.fieldOf("default")
        .xmap(DamageBar::new, DamageBar::defaultColor);

    public DamageBar(int defaultColor) {
        // 确保传入颜色为不透明
        this.defaultColor = ARGB.opaque(defaultColor);
    }

    @Override
    public int calculate(ItemStack stack, @Nullable ClientLevel level, @Nullable LivingEntity entity) {
        return stack.isDamaged() ? ARGB.opaque(stack.getBarColor()) : defaultColor;
    }

    @Override
    public MapCodec<DamageBar> type() {
        return MAP_CODEC;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerItemTintSources(RegisterColorHandlersEvent.ItemTintSources event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "damage_bar"),
        // MapCodec
        DamageBar.MAP_CODEC
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:model",
        // 指向'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item",
        // 要应用的着色列表
        "tints": [
            {
                // 当tintindex为0时
                "type": "examplemod:damage_bar",
                // 0x00FF00（纯绿色）
                "default": 65280
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new BlockModelWrapper.Unbaked(
            // 指向'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            // 要应用的着色列表
            List.of(
                // 当tintindex为0时
                new DamageBar(
                    // 纯绿色
                    0x00FF00
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 复合模型(`Composite Models`)

有时可能需要为单个物品注册多个模型。虽然可直接使用[复合模型加载器(`composite model loader`)][composite]，但对于物品模型，有自定义的`minecraft:composite`类型接收要渲染的模型列表。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:composite",

        // 要渲染的模型
        // 按列表顺序渲染
        "models": [
            {
                "type": "minecraft:model",
                // 指向'assets/examplemod/models/item/example_item_1.json'
                "model": "examplemod:item/example_item_1"
            },
            {
                "type": "minecraft:model",
                // 指向'assets/examplemod/models/item/example_item_2.json'
                "model": "examplemod:item/example_item_2"
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new CompositeModel.Unbaked(
            // 要渲染的模型
            // 按列表顺序渲染
            List.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                    // 要应用的着色列表
                    Collections.emptyList()
                ),
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 属性模型(`Property Models`)

某些物品根据堆叠中存储的数据改变状态（如拉弓、鞘翅破裂、特定维度的时钟等）。为使模型能基于状态改变，物品模型可指定要跟踪的属性并根据条件选择模型。属性模型有三种类型：**范围分派**(`range dispatch`)、**选择**(`select`)和**条件**(`conditional`)，分别处理浮点数、切换案例和布尔值。

### 范围分派模型(`Range Dispatch Models`)

范围分派模型定义`RangeSelectItemModelProperty`类型以获取用于切换模型的浮点数。每个条目有阈值，属性值必须大于该值才渲染。选择的模型是最接近但不大于属性值的阈值对应模型（如属性值为`4`，阈值为`3`和`5`时选择`3`；值为`6`时选择`5`）。可用`RangeSelectItemModelProperty`见`RangeSelectItemModelProperties`。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:range_dispatch",

        // 使用的`RangeSelectItemModelProperty`
        "property": "minecraft:count",
        // 计算属性值的乘数标量
        // 如count为0.3，scale为0.2，则检查阈值为0.3*0.2=0.06
        "scale": 1,
        "fallback": {
            // 无阈值匹配时使用的后备模型
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item.json'
            "model": "examplemod:item/example_item"
        },

        // `Count`定义的属性
        // 为true时使用最大堆叠大小归一化计数
        "normalize": true,

        // 含阈值信息的条目
        "entries": [
            {
                // 当计数为当前最大堆叠大小的三分之一时
                "threshold": 0.33,
                "model": {
                    // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当计数为当前最大堆叠大小的三分之二时
                "threshold": 0.66,
                "model": {
                    // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new RangeSelectItemModel.Unbaked(
            new Count(
                // 为true时使用最大堆叠大小归一化计数
                true
            ),
            // 计算属性值的乘数标量
            // 如count为0.3，scale为0.2，则检查阈值为0.3*0.2=0.06
            1,
            // 含阈值信息的条目
            List.of(
                new RangeSelectItemModel.Entry(
                    // 当计数为当前最大堆叠大小的三分之一时
                    0.33,
                    // 可为任意未烘焙模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_1.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                ),
                new RangeSelectItemModel.Entry(
                    // 当计数为当前最大堆叠大小的三分之二时
                    0.66,
                    // 可为任意未烘焙模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_2.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                )
            ),
            // 无阈值匹配时使用的后备模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建自定义`RangeSelectItemModelProperty`类似于其他基于编解码器的注册对象。创建实现`RangeSelectItemModelProperty`的类，创建用于编码/解码对象的`MapCodec`，并通过[模组事件总线][modbus]上的`RegisterRangeSelectItemModelPropertyEvent`将编解码器注册到注册表中。`RangeSelectItemModelProperty`仅包含一个方法`get`，接收当前`ItemStack`、堆叠所在世界、持有堆叠的实体及种子值，返回由范围分派模型解释的任意浮点数。

```java
public record AppliedEnchantments() implements RangeSelectItemModelProperty {

    public static final MapCodec<AppliedEnchantments> MAP_CODEC = MapCodec.unit(new AppliedEnchantments());

    @Override
    public float get(ItemStack stack, @Nullable ClientLevel level, @Nullable LivingEntity entity, int seed) {
        return (float) stack.getTagEnchantments().size();
    }

    @Override
    public MapCodec<AppliedEnchantments> type() {
        return MAP_CODEC;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerRangeProperties(RegisterRangeSelectItemModelPropertyEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "applied_enchantments"),
        // MapCodec
        AppliedEnchantments.MAP_CODEC
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:range_dispatch",

        // 使用的`RangeSelectItemModelProperty`
        "property": "examplemod:applied_enchantments",
        // 计算属性值的乘数标量
        // 如count为0.3，scale为0.2，则检查阈值为0.3*0.2=0.06
        "scale": 0.5,
        "fallback": {
            // 无阈值匹配时使用的后备模型
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item.json'
            "model": "examplemod:item/example_item"
        },

        // 含阈值信息的条目
        "entries": [
            {
                // 当至少有一个附魔存在时
                // 因为1 * 0.5 = 0.5
                "threshold": 0.5,
                "model": {
                    // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当至少有两个附魔存在时
                // 因为2 * 0.5 = 1
                "threshold": 1,
                "model": {
                    // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new RangeSelectItemModel.Unbaked(
            new AppliedEnchantments(),
            // 计算属性值的乘数标量
            // 如count为0.3，scale为0.2，则检查阈值为0.3*0.2=0.06
            0.5,
            // 含阈值信息的条目
            List.of(
                new RangeSelectItemModel.Entry(
                    // 当至少有一个附魔存在时
                    0.5,
                    // 可为任意未烘焙模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_1.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                ),
                new RangeSelectItemModel.Entry(
                    // 当至少有两个附魔存在时
                    1,
                    // 可为任意未烘焙模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_2.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                )
            ),
            // 无阈值匹配时使用的后备模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

### 选择模型(`Select Models`)

选择模型类似于范围分派模型，但基于`SelectItemModelProperty`定义的值切换，类似于枚举的switch语句。选择的模型是精确匹配属性值的案例。可用`SelectItemModelProperty`见`SelectItemModelProperties`。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:select",

        // 使用的`SelectItemModelProperty`
        "property": "minecraft:display_context",
        "fallback": {
            // 无案例匹配时使用的后备模型
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            "model": "examplemod:item/example_item"
        },

        // 基于可选属性的切换案例
        "cases": [
            {
                // 当显示上下文为`ItemDisplayContext#GUI`时
                "when": "gui",
                "model": {
                    // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当显示上下文为`ItemDisplayContext#FIRST_PERSON_RIGHT_HAND`时
                "when": "firstperson_righthand",
                "model": {
                     // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SelectItemModel.Unbaked(
            new SelectItemModel.UnbakedSwitch(
                // 使用的`SelectItemModelProperty`
                new DisplayContext(),
                // 基于可选属性的切换案例
                List.of(
                    new SelectItemModel.SwitchCase(
                        // 匹配此模型的案例列表
                        List.of(ItemDisplayContext.GUI),
                        // 可为任意未烘焙模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_1.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    ),
                    new SelectItemModel.SwitchCase(
                        // 匹配此模型的案例列表
                        List.of(ItemDisplayContext.FIRST_PERSON_RIGHT_HAND),
                        // 可为任意未烘焙模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_2.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    )
                )
            ),
            // 无案例匹配时使用的后备模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建自定义`SelectItemModelProperty`类似于基于编解码器的注册对象。创建实现`SelectItemModelProperty<T>`的类，创建用于序列化/反序列化属性值的`Codec`，创建用于编码/解码对象的`MapCodec`，并通过[模组事件总线][modbus]上的`RegisterSelectItemModelPropertyEvent`将编解码器注册到注册表中。`SelectItemModelProperty`有泛型`T`表示要切换的值，仅包含一个方法`get`，接收当前`ItemStack`、堆叠所在世界、持有堆叠的实体、种子值及物品显示上下文，返回由选择模型解释的任意`T`。

```java
// 选择属性类
public record StackRarity() implements SelectItemModelProperty<Rarity> {

    // 包含相关编解码器的注册对象
    public static final SelectItemModelProperty.Type<StackRarity, Rarity> TYPE = SelectItemModelProperty.Type.create(
        // 此属性的MapCodec
        MapCodec.unit(new StackRarity()),
        // 被选择对象的编解码器
        // 用于序列化案例条目("when": <属性值>)
        Rarity.CODEC
    );

    @Nullable
    @Override
    public Rarity get(ItemStack stack, @Nullable ClientLevel level, @Nullable LivingEntity entity, int seed, ItemDisplayContext displayContext) {
        // 返回null时使用后备模型
        return stack.get(DataComponents.RARITY);
    }

    @Override
    public SelectItemModelProperty.Type<StackRarity, Rarity> type() {
        return TYPE;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerSelectProperties(RegisterSelectItemModelPropertyEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "rarity"),
        // 属性类型
        StackRarity.TYPE
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:select",

        // 使用的`SelectItemModelProperty`
        "property": "examplemod:rarity",
        "fallback": {
            // 无案例匹配时使用的后备模型
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            "model": "examplemod:item/example_item"
        },

        // 基于可选属性的切换案例
        "cases": [
            {
                // 当稀有度为`Rarity#UNCOMMON`时
                "when": "uncommon",
                "model": {
                    // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当稀有度为`Rarity#RARE`时
                "when": "rare",
                "model": {
                     // 可为任意未烘焙模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SelectItemModel.Unbaked(
            new SelectItemModel.UnbakedSwitch(
                // 使用的`SelectItemModelProperty`
                new StackRarity(),
                // 基于可选属性的切换案例
                List.of(
                    new SelectItemModel.SwitchCase(
                        // 匹配此模型的案例列表
                        List.of(Rarity.UNCOMMON),
                        // 可为任意未烘焙模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_1.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    ),
                    new SelectItemModel.SwitchCase(
                        // 匹配此模型的案例列表
                        List.of(Rarity.RARE),
                        // 可为任意未烘焙模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_2.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    )
                )
            ),
            // 无案例匹配时使用的后备模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

### 条件模型(`Conditional Models`)

条件模型是三者中最简单的。类型定义`ConditionalItemModelProperty`以获取用于切换模型的布尔值。根据返回的布尔值选择模型。可用`ConditionalItemModelProperty`见`ConditionalItemModelProperties`。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:condition",

        // 使用的`ConditionalItemModelProperty`
        "property": "minecraft:damaged",

        // 布尔结果
        "on_true": {
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_1.json'
            "model": "examplemod:item/example_item_1"
            
        },
        "on_false": {
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_2.json'
            "model": "examplemod:item/example_item_2"
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new ConditionalItemModel.Unbaked(
            // 要检查的属性
            new Damaged(),
            // 当布尔值为true时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_1.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                // 要应用的着色列表
                Collections.emptyList()
            ),
            // 当布尔值为false时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_2.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                // 要应用的着色列表
                Collections.emptyList()
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建自定义`ConditionalItemModelProperty`类似于其他基于编解码器的注册对象。创建实现`ConditionalItemModelProperty`的类，创建用于编码/解码对象的`MapCodec`，并通过[模组事件总线][modbus]上的`RegisterConditionalItemModelPropertyEvent`将编解码器注册到注册表中。`RangeSelectItemModelProperty`仅包含一个方法`get`，接收当前`ItemStack`、堆叠所在世界、持有堆叠的实体、种子值及物品显示上下文，返回由条件模型解释的任意布尔值（`on_true`或`on_false`）。

```java
public record BarVisible() implements ConditionalItemModelProperty {

    public static final MapCodec<BarVisible> MAP_CODEC =  MapCodec.unit(new BarVisible());

    @Override
    public boolean get(ItemStack stack, @Nullable ClientLevel level, @Nullable LivingEntity entity, int seed, ItemDisplayContext context) {
        return stack.isBarVisible();
    }

    @Override
    public MapCodec<BarVisible> type() {
        return MAP_CODEC;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerConditionalProperties(RegisterConditionalItemModelPropertyEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "bar_visible"),
        // MapCodec
        BarVisible.MAP_CODEC
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:condition",

        // 使用的`ConditionalItemModelProperty`
        "property": "examplemod:bar_visible",

        // 布尔结果
        "on_true": {
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_1.json'
            "model": "examplemod:item/example_item_1"
            
        },
        "on_false": {
            // 可为任意未烘焙模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_2.json'
            "model": "examplemod:item/example_item_2"
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new ConditionalItemModel.Unbaked(
            // 要检查的属性
            new BarVisible(),
            // 当布尔值为true时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_1.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                // 要应用的着色列表
                Collections.emptyList()
            ),
            // 当布尔值为false时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_2.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                // 要应用的着色列表
                Collections.emptyList()
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 特殊模型(`Special Models`)

并非所有模型都能使用基础模型JSON渲染。某些模型可动态渲染，或使用为[`BlockEntityRenderer`][ber]创建的现有模型。此类情况下有特殊模型类型允许用户定义自己的渲染逻辑。这些称为`SpecialModelRenderer`，在`SpecialModelRenderers`中定义。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:special",

        // 从中读取粒子纹理和显示变换的父模型
        // 指向'assets/minecraft/models/item/template_skull.json'
        "base": "minecraft:item/template_skull",
        "model": {
            // 使用的特殊模型渲染器
            "type": "minecraft:head",

            // `SkullSpecialRenderer.Unbaked`定义的属性
            // 头颅方块类型
            "kind": "wither_skeleton",
            // 渲染头部时使用的纹理
            // 指向'assets/examplemod/textures/entity/heads/skeleton_override.png'
            "texture": "examplemod:heads/skeleton_override",
            // 用于动画化头部模型的动画浮点数
            "animation": 0.5
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SpecialModelWrapper.Unbaked(
            // 从中读取粒子纹理和显示变换的父模型
            // 指向'assets/minecraft/models/item/template_skull.json'
            ResourceLocation.fromNamespaceAndPath("minecraft", "item/template_skull"),
            // 使用的特殊模型渲染器
            new SkullSpecialRenderer.Unbaked(
                // 头颅方块类型
                SkullBlock.Types.WITHER_SKELETON,
                // 渲染头部时使用的纹理
                // 指向'assets/examplemod/textures/entity/heads/skeleton_override.png'
                Optional.of(
                    ResourceLocation.fromNamespaceAndPath("examplemod", "heads/skeleton_override")
                ),
                // 用于动画化头部模型的动画浮点数
                0.5f
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建自定义`SpecialModelRenderer`分为三部分：用于渲染物品的`SpecialModelRenderer`实例；用于读写JSON的`SpecialModelRenderer.Unbaked`实例；以及将渲染器注册为物品或（必要时）方块。

首先是`SpecialModelRenderer`。其工作方式类似于其他渲染器类（如方块实体渲染器、实体渲染器）。应接收渲染过程中使用的静态数据（如`Model`实例、纹理的`Material`等）。有两个方法需注意：
1. `extractArgument`：通过仅提供`ItemStack`中必要的数据来限制`render`方法可用的数据量。
:::note
若不确定需要哪些数据，可返回`ItemStack`本身。若不需要堆叠数据，可使用`NoDataSpecialModelRenderer`（已实现此方法）。
:::
2. `render`方法：接收`extractArgument`返回的值、物品显示上下文、渲染的位姿栈(`pose stack`)、可用的缓冲区源(`buffer sources`)、打包光照(`packed light`)、覆盖纹理(`overlay texture`)以及堆叠是否附魔(`foiled`)。所有渲染应在此方法中完成。

```java
public record ExampleSpecialRenderer(Model model, Material material) implements SpecialModelRenderer<Boolean> {

    @Nullable
    public Boolean extractArgument(ItemStack stack) {
        // 提取要使用的数据
        return stack.isBarVisible();
    }

    // 渲染模型
    @Override
    public void render(@Nullable Boolean barVisible, ItemDisplayContext displayContext, PoseStack pose, MultiBufferSource bufferSource, int light, int overlay, boolean hasFoil) {
        this.model.renderToBuffer(pose, this.material.buffer(bufferSource, barVisible ? RenderType::entityCutout : RenderType::entitySolid), light, overlay);
    }
}
```

接下来是`SpecialModelRenderer.Unbaked`实例。应包含可从文件读取以确定传递给特殊渲染器的数据。也包含两个方法：
- `bake`：用于构造特殊渲染器实例
- `type`：定义用于文件编码/解码的`MapCodec`

```java
public record ExampleSpecialRenderer(Model model, Material material) implements SpecialModelRenderer<Boolean> {

    // ...

    public record Unbaked(ResourceLocation texture) implements SpecialModelRenderer.Unbaked {
        // 为编解码器创建渲染类型映射
        private static final BiMap<String, RenderType> RENDER_TYPES = Util.make(HashBiMap.create(), map -> {
            map.put("translucent_item", Sheets.translucentItemSheet());
            map.put("cutout_block", Sheets.cutoutBlockSheet());
        });
        private static final Codec<RenderType> RENDER_TYPE_CODEC = ExtraCodecs.idResolverCodec(Codec.STRING, RENDER_TYPES::get, RENDER_TYPES.inverse()::get);

        // 要注册的MapCodec
        public static final MapCodec<ExampleSpecialRenderer.Unbaked> MAP_CODEC = RecordCodecBuilder.mapCodec(instance ->
            instance.group(
                ResourceLocation.CODEC.fieldOf("texture").forGetter(ExampleSpecialRenderer.Unbaked::texture)
            )
            .apply(instance, ExampleSpecialRenderer.Unbaked::new)
        );

        @Override
        public MapCodec<ExampleSpecialRenderer.Unbaked> type() {
            return MAP_CODEC;
        }

        @Override
        public SpecialModelRenderer<?> bake(EntityModelSet modelSet) {
            // 解析资源定位符为绝对路径
            ResourceLocation textureLoc = this.texture.withPath(path -> "textures/entity/" + path + ".png");

            // 获取模型和要渲染的材质
            return new ExampleSpecialRenderer(...);
        }
    }
}
```

最后，将对象注册到必要位置。客户端物品通过[模组事件总线][modbus]上的`RegisterSpecialModelRendererEvent`注册。若特殊渲染器也应用于`BlockEntityRenderer`（如在类物品上下文中渲染，如末影人手持方块），则方块的`Unbaked`版本应通过[模组事件总线][modbus]上的`RegisterSpecialBlockModelRendererEvent`注册。

```java
// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerSpecialRenderers(RegisterSpecialModelRendererEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "example_special"),
        // MapCodec
        ExampleSpecialRenderer.Unbaked.MAP_CODEC
    )
}

// 为在类物品上下文中渲染方块
// 假设存在DeferredBlock<ExampleBlock> EXAMPLE_BLOCK
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerSpecialBlockRenderers(RegisterSpecialBlockModelRendererEvent event) {
    event.register(
        // 要渲染的方块
        EXAMPLE_BLOCK.get()
        // 要使用的未烘焙实例
        new ExampleSpecialRenderer.Unbaked(ResourceLocation.fromNamespaceAndPath("examplemod", "entity/example_special"))
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:special",

        // 从中读取粒子纹理和显示变换的父模型
        // 指向'assets/minecraft/models/item/template_skull.json'
        "base": "minecraft:item/template_skull",
        "model": {
            // 使用的特殊模型渲染器
            "type": "examplemod:example_special",

            // `ExampleSpecialRenderer.Unbaked`定义的属性
            // 渲染时使用的纹理
            // 指向'assets/examplemod/textures/entity/example/example_texture.png'
            "texture": "examplemod:example/example_texture"
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SpecialModelWrapper.Unbaked(
            // 从中读取粒子纹理和显示变换的父模型
            // 指向'assets/minecraft/models/item/template_skull.json'
            ResourceLocation.fromNamespaceAndPath("minecraft", "item/template_skull"),
            // 使用的特殊模型渲染器
            new ExampleSpecialRenderer.Unbaked(
                // 渲染时使用的纹理
                // 指向'assets/examplemod/textures/entity/example/example_texture.png'
                ResourceLocation.fromNamespaceAndPath("examplemod", "example/example_texture")
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 动态流体容器(`Dynamic Fluid Container`)

NeoForge新增了构造动态流体容器的物品模型，能在运行时根据所含流体重新纹理化。

:::note
为使流体着色应用于流体纹理，相关物品必须附加`Capabilities.FluidHandler.ITEM`。若物品未直接使用`BucketItem`（或子类），则需[将能力注册到物品][capability]。
:::

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "neoforge:fluid_container",

        // 构造容器使用的纹理
        // 这些纹理引用方块图集，因此相对于`textures`目录
        "textures": {
            // 设置模型粒子精灵
            // 未设置时使用首个非空纹理：
            // - 流体静止纹理
            // - 容器基础纹理
            // - 容器覆盖纹理（若未用作遮罩）
            // 指向'assets/minecraft/textures/item/bucket.png'
            "particle": "minecraft:item/bucket",
            // 设置第一层使用的纹理（通常是流体容器）
            // 未设置时不会添加该层
            // 指向'assets/minecraft/textures/item/bucket.png'
            "base": "minecraft:item/bucket",
            // 设置用作流体静止纹理遮罩的纹理
            // 流体可见区域应为纯白色
            // 未设置或流体为空时不渲染该层
            // 指向'assets/neoforge/textures/item/mask/bucket_fluid.png'
            "fluid": "neoforge:item/mask/bucket_fluid",
            // 设置用作以下之一的纹理：
            // - `cover_is_mask`为false时的覆盖纹理
            // - `cover_is_mask`为true时应用于基础纹理的遮罩（应为纯白色可见）
            // 未设置或`cover_is_mask`为true时无基础纹理则不渲染该层
            // 指向'assets/neoforge/textures/item/mask/bucket_fluid_cover.png'
            "cover": "neoforge:item/mask/bucket_fluid_cover",
        },

        // 为true时，密度为负或零的流体会将模型旋转180度
        // 默认为false
        "flip_gas": true,
        // 为true时，将覆盖纹理用作基础纹理的遮罩
        // 默认为true
        "cover_is_mask": true,
        // 为true时，为光照等级大于零的流体将流体纹理层的光照图设为其最大值
        // 默认为true
        "apply_fluid_luminosity": false
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<ExampleFluidContainerItem> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new DynamicFluidContainerModel.Unbaked(
            // 构造容器使用的纹理
            // 这些纹理引用方块图集，因此相对于`textures`目录
            new DynamicFluidContainerModel.Textures(
                // 设置模型粒子精灵
                // 未设置时使用首个非空纹理：
                // - 流体静止纹理
                // - 容器基础纹理
                // - 容器覆盖纹理（若未用作遮罩）
                // 指向'assets/minecraft/textures/item/bucket.png'
                Optional.of(ResourceLocation.withDefaultNamespace("item/bucket")),
                // 设置第一层使用的纹理（通常是流体容器）
                // 未设置时不会添加该层
                // 指向'assets/minecraft/textures/item/bucket.png'
                Optional.of(ResourceLocation.withDefaultNamespace("item/bucket")),
                // 设置用作流体静止纹理遮罩的纹理
                // 流体可见区域应为纯白色
                // 未设置或流体为空时不渲染该层
                // 指向'assets/neoforge/textures/item/mask/bucket_fluid.png'
                Optional.of(ResourceLocation.fromNamespaceAndPath("neoforge", "item/mask/bucket_fluid")),
                // 设置用作以下之一的纹理：
                // - `cover_is_mask`为false时的覆盖纹理
                // - `cover_is_mask`为true时应用于基础纹理的遮罩（应为纯白色可见）
                // 未设置或`cover_is_mask`为true时无基础纹理则不渲染该层
                // 指向'assets/neoforge/textures/item/mask/bucket_fluid_cover.png'
                Optional.of(ResourceLocation.fromNamespaceAndPath("neoforge", "item/mask/bucket_fluid_cover"))
            ),
            // 为true时，密度为负或零的流体会将模型旋转180度
            // 默认为false
            true,
            // 为true时，将覆盖纹理用作基础纹理的遮罩
            // 默认为true
            true,
            // 为true时，为光照等级大于零的流体将流体纹理层的光照图设为其最大值
            // 默认为true
            false
        )
    );
}
```

</TabItem>
</Tabs>

## 手动渲染物品(`Manually Rendering an Item`)

若需手动渲染物品（如在`BlockEntityRenderer`或`EntityRenderer`中），可通过三步实现：
1. 渲染器创建`ItemStackRenderState`保存堆叠的渲染信息
2. `ItemModelResolver`通过其方法之一更新`ItemStackRenderState`为当前要渲染的物品
3. 使用渲染状态的`render`方法渲染物品

`ItemStackRenderState`跟踪渲染使用的数据。每个"模型"有自己的`ItemStackRenderState.LayerRenderState`，包含要渲染的`BakedQuad`及其渲染类型、附魔状态、着色信息、动画标志、范围(`extents`)和使用的任何特殊渲染器。使用`newLayer`方法创建层，使用`clear`方法清除渲染。若使用预定层数，则`ensureCapacity`确保有足够数量的`LayerRenderStates`正确渲染。

:::note
[屏幕(`screens`)][screens]使用子类`TrackingItemStackRenderState`保存模型标识元素，用于跨帧缓存渲染状态。
:::

`ItemModelResolver`负责更新`ItemStackRenderState`。通过`updateForLiving`（活体实体持有的物品）、`updateForNonLiving`（其他实体持有的物品）或`updateForTopItem`（其他情况）实现。这些方法接收渲染状态、要渲染的堆叠和当前显示上下文。其他参数更新手持信息、世界、实体和种子值。每个方法在调用从`DataComponents#ITEM_MODEL`获取的`ItemModel`的`update`前调用`ItemStackRenderState#clear`。若不在渲染器上下文（如`BlockEntityRenderer`、`EntityRenderer`）中，可通过`Minecraft#getItemModelResolver`获取`ItemModelResolver`。

## 自定义物品模型定义(`Custom Item Model Definitions`)

创建自定义`ItemModel`分为三部分：用于更新渲染状态的`ItemModel`实例；用于读写JSON的`ItemModel.Unbaked`实例；以及使用`ItemModel`的注册。

:::warning
请确保现有系统无法创建所需物品模型。大多数情况下无需创建自定义`ItemModel`。
:::

首先是`ItemModel`。负责更新`ItemStackRenderState`以正确渲染物品。应接收渲染过程中使用的静态数据（如`BakedQuad`列表、属性信息等）。唯一方法是`update`，接收渲染状态、堆叠、模型解析器、显示上下文、世界、持有实体及种子值来更新`ItemStackRenderState`。仅应修改`ItemStackRenderState`参数，其余视为只读数据。

```java
public record RenderTypeModelWrapper(List<BakedQuad> quads, List<ItemTintSource> tints, ModelRenderProperties properties, RenderType type) implements ItemModel {

    // 更新渲染状态
    @Override
    public void update(ItemStackRenderState state, ItemStack stack, ItemModelResolver resolver, ItemDisplayContext displayContext, @Nullable ClientLevel level, @Nullable LivingEntity entity, int seed) {
        // 设置模型使用的标识元素
        state.appendModelIdentityElement(this);

        // 创建新层渲染模型
        ItemStackRenderState.LayerRenderState layerState = state.newLayer();

        // 设置要使用的附魔类型
        if (stack.hasFoil()) {
            layerState.setFoilType(ItemStackRenderState.FoilType.STANDARD);
            state.appendModelIdentityElement(ItemStackRenderState.FoilType.STANDARD);
            layerState.setAnimated();
        }


        // 应用着色源
        int tintSize = this.tints.size();
        int[] tintLayers = layerState.prepareTintLayers(tintSize);

        for (int idx = 0; idx < tintSize; idx++) {
            int tintColor = this.tints.get(idx).calculate(stack, level, entity);
            tintLayers[idx] = tintColor;
            state.appendModelIdentityElement(tintColor);
        }

        // 计算屏幕上渲染模型的边界
        // 用于GUI渲染边界（当过大时）和物品实体浮动
        layerState.setExtents(BlockModelWrapper.computeExtents(this.quads));

        // 设置当前渲染类型
        layerState.setRenderType(this.type);

        // 设置其他通用模型属性
        this.properties.applyToLayer(layerState, displayContext);

        // 添加要渲染的四边形
        layerState.prepareQuadList().addAll(this.quads);
    }
}
```

接下来是`ItemModel.Unbaked`实例。应包含可从文件读取以确定传递给物品模型的数据。也包含两个方法：
- `bake`：用于构造`ItemModel`实例
- `type`：定义用于文件编码/解码的`MapCodec`

```java
public record RenderTypeModelWrapper(List<BakedQuad> quads, List<ItemTintSource> tints, ModelRenderProperties properties, RenderType type) implements ItemModel {

    // ...

     public record Unbaked(ResourceLocation model, List<ItemTintSource> tints, RenderType type) implements ItemModel.Unbaked {
        // 为编解码器创建渲染类型映射
        private static final BiMap<String, RenderType> RENDER_TYPES = Util.make(HashBiMap.create(), map -> {
            map.put("translucent_item", Sheets.translucentItemSheet());
            map.put("cutout_block", Sheets.cutoutBlockSheet());
        });
        private static final Codec<RenderType> RENDER_TYPE_CODEC = ExtraCodecs.idResolverCodec(Codec.STRING, RENDER_TYPES::get, RENDER_TYPES.inverse()::get);

        // 要注册的MapCodec
        public static final MapCodec<RenderTypeModelWrapper.Unbaked> MAP_CODEC = RecordCodecBuilder.mapCodec(instance ->
            instance.group(
                ResourceLocation.CODEC.fieldOf("model").forGetter(RenderTypeModelWrapper.Unbaked::model),
                ItemTintSources.CODEC.listOf().optionalFieldOf("tints", List.of()).forGetter(RenderTypeModelWrapper.Unbaked::tints)
                RENDER_TYPE_CODEC.fieldOf("render_type").forGetter(RenderTypeModelWrapper.Unbaked::type)
            )
            .apply(instance, RenderTypeModelWrapper.Unbaked::new)
        );

        @Override
        public void resolveDependencies(ResolvableModel.Resolver resolver) {
            // 标记此物品模型的所有依赖项
            resolver.markDependency(this.model);
        }

        @Override
        public ItemModel bake(ItemModel.BakingContext context) {
            // 获取烘焙四边形并返回
            ModelBaker baker = context.blockModelBaker();
            ResolvedModel resolvedModel = baker.getModel(this.model);
            TextureSlots slots = resolvedModel.getTopTextureSlots();

            return new RenderTypeModelWrapper(
                resolvedModel.bakeTopGeometry(slots, baker, BlockModelRotation.X0_Y0).getAll(),
                this.tints,
                ModelRenderProperties.fromResolvedModel(baker, resolvedModel, slots),
                this.type
            );
        }

        @Override
        public MapCodec<RenderTypeModelWrapper.Unbaked> type() {
            return MAP_CODEC;
        }
    }
}
```

然后，通过[模组事件总线][modbus]上的`RegisterItemModelsEvent`注册MapCodec。

```java
// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerItemModels(RegisterItemModelsEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "render_type"),
        // MapCodec
        RenderTypeModelWrapper.Unbaked.MAP_CODEC
    )
}
```

最后，可在JSON或数据生成过程中使用`ItemModel`。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "examplemod:render_type",
        // 指向'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item",
        // 应用于模型纹理的任何着色
        "tints": []
        // 设置渲染时使用的渲染类型
        "render_type": "cutout_block"
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设存在DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider中
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new RenderTypeModelWrapper.Unbaked(
            // 指向'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            // 应用于模型纹理的任何着色
            List.of(),
            // 设置渲染时使用的渲染类型
            Sheets.cutoutBlockSheet()
        )
    );
}
```

</TabItem>
</Tabs>

[assets]: ../../index.md#assets
[ber]: ../../../blockentities/ber.md
[capability]: ../../../datastorage/capabilities.md#registering-capabilities
[composite]: modelloaders.md#composite-model
[itemmodel]: #manually-rendering-an-item
[modbus]: ../../../concepts/events.md#event-buses
[models]: modelsystem.md
[rl]: ../../../misc/resourcelocation.md
[screens]: ../../../gui/screens.md#items