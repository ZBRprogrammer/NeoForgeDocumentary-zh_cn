import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 附魔(Enchantments)

**附魔(`Enchantments`)** 是可应用于工具和其他物品的特殊效果。自1.21版本起，附魔作为[数据组件(`Data Components`)]存储在物品上，通过JSON定义，由附魔效果组件组成。游戏中，物品的附魔信息存储在`DataComponents.ENCHANTMENTS`组件的`ItemEnchantments`实例中。

添加新附魔需在数据包的`enchantment`子文件夹创建JSON文件。例如，创建`examplemod:example_enchant`需创建文件：`data/examplemod/enchantment/example_enchantment.json`。

## 附魔JSON格式

```json5
{
    // 游戏内显示名称（可为翻译键或字面字符串）
    "description": {
        "translate": "enchantment.examplemod.enchant_name"
    },
    
    // 可附魔物品（单个ID/ID列表/物品标签）
    "supported_items": "#examplemod:enchantable/enchant_name",

    // (可选)附魔台显示物品（默认同supported_items）
    "primary_items": [
        "examplemod:item_a",
        "examplemod:item_b"
    ],

    // (可选)互斥附魔（单个ID/ID列表/附魔标签）
    "exclusive_set": "#examplemod:exclusive_to_enchant_name",
    
    // 附魔台出现概率 [1, 1024]
    "weight": 6,
    
    // 最高等级 [1, 255]
    "max_level": 3,
    
    // 最大附魔成本（基于"附魔能量"）
    "max_cost": {
        "base": 45,
        "per_level_above_first": 9
    },
    
    // 最小附魔成本
    "min_cost": {
        "base": 2,
        "per_level_above_first": 8
    },

    // 铁砧修复等级消耗（乘以附魔等级）
    "anvil_cost": 2,
    
    // (可选)生效装备槽位组
    "slots": [
        "mainhand"
    ],

    // 附魔效果组件
    "effects": {
        "examplemod:custom_effect": [
            {
                "effect": {
                    "type": "minecraft:add",
                    "value": {
                        "type": "minecraft:linear",
                        "base": 1,
                        "per_level_above_first": 1
                    }
                }
            }
        ]
    }
}
```

### 附魔成本与等级

`max_cost`和`min_cost`字段决定附魔所需能量阈值。计算过程如下：
1. 通过`IBlockExtension#getEnchantPowerBonus()`获取周围方块加成
2. 调用`EnchantmentHelper#getEnchantmentCost`计算基础等级（游戏内绿字）
3. 基于物品附魔能力值(`DataComponents#ENCHANTABLE`)随机调整：
   `调整等级 = 基础等级 + random.nextInt(e/4+1) + random.nextInt(e/4+1)`  
   （`e`为附魔能力值）
4. 最终等级需在附魔的cost范围内才会被选中

## 附魔效果组件(`Enchantment Effect Components`)

附魔效果组件是特殊注册的[数据组件(`Data Components`)]，决定附魔功能逻辑。原版提供了多种[内置附魔效果组件]。

### 自定义附魔效果组件

需完全自定义逻辑实现。首先定义数据承载类：
```java
// 示例数据记录类
public record Increment(int value) {
    public static final Codec<Increment> CODEC = RecordCodecBuilder.create(instance ->
            instance.group(
                    Codec.INT.fieldOf("value").forGetter(Increment::value)
            ).apply(instance, Increment::new)
    );

    public int add(int x) {
        return value() + x;
    }
}
```

在`BuiltInRegistries.ENCHANTMENT_EFFECT_COMPONENT_TYPE`注册组件类型：
```java
// 在注册类中
public static final DeferredRegister.DataComponents ENCHANTMENT_COMPONENT_TYPES =
    DeferredRegister.createDataComponents(BuiltInRegistries.ENCHANTMENT_EFFECT_COMPONENT_TYPE, "examplemod");

public static final Supplier<DataComponentType<Increment>> INCREMENT =
    ENCHANTMENT_COMPONENT_TYPES.registerComponentType(
        "increment",
        builder -> builder.persistent(Increment.CODEC)
    );
```

在游戏逻辑中应用效果：
```java
// 获取物品堆栈
AtomicInteger atomicValue = new AtomicInteger(value);

EnchantmentHelper.runIterationOnItem(stack, (enchantmentHolder, enchantLevel) -> {
    // 从附魔持有者获取Increment实例
    Increment increment = enchantmentHolder.value().effects().get(INCREMENT.get());
    if(increment != null){
        atomicValue.set(increment.add(atomicValue.get()));
    }
});

int modifiedValue = atomicValue.get(); // 使用修改后的值
```

### 条件效果(`ConditionalEffect`)

通过`ConditionalEffect<?>`包装可在特定[战利品上下文(`LootContext`)]条件下生效：
```java
// 使用辅助方法应用条件效果
enchant.applyEffects(
    enchant.getEffects(EnchantmentEffectComponents.KNOCKBACK),
    lootContext,
    (effectData) -> // 使用满足条件的effectData
);
```

注册条件效果组件类型：
```java
public static final DeferredHolder<DataComponentType<?>, DataComponentType<ConditionalEffect<Increment>>> CONDITIONAL_INCREMENT =
    ENCHANTMENT_COMPONENT_TYPES.register("conditional_increment",
        () -> DataComponentType.ConditionalEffect<Increment>builder()
            .persistent(ConditionalEffect.codec(Increment.CODEC, LootContextParamSets.ENCHANTED_DAMAGE))
            .build());
```

## 附魔数据生成(`Enchantment Data Generation`)

通过[数据生成(`data generation`)]系统自动创建附魔JSON文件，使用`RegistrySetBuilder`和`DatapackBuiltinEntriesProvider`实现。

<Tabs>
<TabItem value="datagen" label="Datagen">

```java
RegistrySetBuilder BUILDER = new RegistrySetBuilder();
BUILDER.add(
    Registries.ENCHANTMENT,
    bootstrap -> bootstrap.register(
        ResourceKey.create(
            Registries.ENCHANTMENT,
            ResourceLocation.fromNamespaceAndPath("examplemod", "example_enchantment")
        ),
        new Enchantment(
            Component.literal("Example Enchantment"),  
            new Enchantment.EnchantmentDefinition(
                HolderSet.direct(...), 
                Optional.empty(), 
                30, 
                3, 
                Enchantment.dynamicCost(3, 1), 
                Enchantment.dynamicCost(4, 2), 
                2, 
                List.of(EquipmentSlotGroup.ANY) 
            ),
            HolderSet.empty(), 
            DataComponentMap.builder() 
                .set(MY_ENCHANTMENT_EFFECT_COMPONENT_TYPE, new ExampleData())
                .build()
        )
    )
);
```

</TabItem>

<TabItem value="json" label="JSON" default>

```json5
{
    "anvil_cost": 2,
    "description": "Example Enchantment",
    "effects": { /* <效果组件> */ },
    "max_cost": { "base": 4, "per_level_above_first": 2 },
    "max_level": 3,
    "min_cost": { "base": 3, "per_level_above_first": 1 },
    "slots": [ "any" ],
    "supported_items": /* <支持物品列表> */,
    "weight": 30
}
```

</TabItem>
</Tabs>

[数据组件(`Data Components`)]: ../../../items/datacomponents.md
[内置附魔效果组件]: builtin.md
[数据生成(`data generation`)]: ../../../resources/index.md#data-generation
[战利品上下文(`LootContext`)]: ../loottables/index.md#loot-context
[数据包注册表数据生成]: https://docs.neoforged.net/docs/concepts/registries/#data-generation-for-datapack-registries