# 内置数据映射(`Built-In Data Maps`)

NeoForge 为常见用例提供了各种内置[**数据映射**][datamap]，取代了硬编码的原版字段。原版值由 NeoForge 中的数据映射文件提供，因此对玩家来说功能上没有差异。

## **`neoforge:compostables`**

允许配置堆肥器值，替代`ComposterBlock.COMPOSTABLES`（现已被忽略）。此数据映射位于`neoforge/data_maps/item/compostables.json`，其对象具有以下结构：

```json5
{
    // 0 到 1（含）之间的浮点数，表示物品更新堆肥器等级的几率
    "chance": 1,
    // 可选，默认为 false - 村民农夫是否可以堆肥此物品
    "can_villager_compost": false
}
```

示例：

```json5
{
    "values": {
        // 给金合欢原木 50% 的几率填满堆肥器
        "minecraft:acacia_log": {
            "chance": 0.5
        }
    }
}
```

## **`neoforge:furnace_fuels`**

允许配置物品燃烧时间。此数据映射位于`neoforge/data_maps/item/furnace_fuels.json`，其对象具有以下结构：

```json5
{
    // 正整数，表示物品的燃烧时间（以游戏刻为单位）
    "burn_time": 1000
}
```

示例：

```json5
{
    "values": {
        // 给铁砧 2 秒的燃烧时间
        "minecraft:anvil": {
            "burn_time": 40
        }
    }
}
```

:::info
NeoForge 额外添加了`IItemExtension#getBurnTime`方法，可在自定义物品中覆盖，覆盖此数据映射。`#getBurnTime`应仅在数据映射不足时使用，例如依赖于[**数据组件**][datacomponent]的燃烧时间。
:::

:::warning
原版为`#minecraft:logs`和`#minecraft:planks`隐式添加了 300 游戏刻（15 秒）的燃烧时间，并硬编码移除了绯红和诡异物品。这意味着如果添加另一种不可燃木材，应从该映射中移除该木材类型的物品，如下所示：

```json5
{
    "replace": false,
    "values": [
        // 此处添加值
    ],
    "remove": [
        "examplemod:example_nether_wood_planks",
        "#examplemod:example_nether_wood_stems",
        "examplemod:example_nether_wood_door",
        // 等等
        // 其他要移除的项在此
    ]
}
```
:::

## **`neoforge:monster_room_mobs`**

允许配置怪物房间中刷怪笼可能生成的生物，替代`MonsterRoomFeature#MOBS`（现已被忽略）。此数据映射位于`neoforge/data_maps/entity_type/monster_room_mobs.json`，其对象具有以下结构：

```json5
{
    // 此生物的权重，相对于数据映射中的其他生物
    "weight": 100
}
```

示例：

```json5
{
    "values": {
        // 让鱿鱼以权重 100 出现在怪物房间刷怪笼中
        "minecraft:squid": {
            "weight": 100
        }
    }
}
```

## **`neoforge:oxidizables`**

允许配置氧化阶段，替代`WeatheringCopper#NEXT_BY_BLOCK`。此数据映射也用于构建反向脱氧映射（用于用斧头刮除）。位于`neoforge/data_maps/block/oxidizables.json`，其对象具有以下结构：

```json5
{
    // 此方块氧化后将变成的方块
    "next_oxidation_stage": "examplemod:oxidized_block"
}
```

:::note
自定义方块必须实现`WeatheringCopperFullBlock`或`WeatheringCopper`，并在`randomTick`中调用`changeOverTime`以自然氧化。
:::

示例：

```json5
{
    "values": {
        "mymod:custom_copper": {
            // 让自定义铜方块氧化为自定义氧化铜
            "next_oxidation_stage": "mymod:custom_oxidized_copper"
        }
    }
}
```

## **`neoforge:parrot_imitations`**

允许配置鹦鹉模仿生物时产生的声音，替代`Parrot#MOB_SOUND_MAP`（现已被忽略）。此数据映射位于`neoforge/data_maps/entity_type/parrot_imitations.json`，其对象具有以下结构：

```json5
{
    // 鹦鹉模仿生物时将产生的声音ID
    "sound": "minecraft:entity.parrot.imitate.creeper"
}
```

示例：

```json5
{
    "values": {
        // 让鹦鹉模仿悦灵时产生洞穴环境音效
        "minecraft:allay": {
            "sound": "minecraft:ambient.cave"
        }
    }
}
```

## **`neoforge:raid_hero_gifts`**

允许配置在阻止袭击后，具有特定`VillagerProfession`的村民可能赠予的礼物，替代`GiveGiftToHero#GIFTS`（现已被忽略）。此数据映射位于`neoforge/data_maps/villager_profession/raid_hero_gifts.json`，其对象具有以下结构：

```json5
{
    // 袭击后村民职业将发放的战利品表ID
    "loot_table": "minecraft:gameplay/hero_of_the_village/armorer_gift"
}
```

示例：

```json5
{
    "values": {
        "minecraft:armorer": {
            // 让盔甲匠给袭击英雄发放盔甲匠礼物战利品表
            "loot_table": "minecraft:gameplay/hero_of_the_village/armorer_gift"
        }
    }
}
```

## **`neoforge:vibration_frequencies`**

允许配置游戏事件发出的幽匿振动频率，替代`VibrationSystem#VIBRATION_FREQUENCY_FOR_EVENT`（现已被忽略）。此数据映射位于`neoforge/data_maps/game_event/vibration_frequencies.json`，其对象具有以下结构：

```json5
{
    // 1 到 15（含）之间的整数，表示事件的振动频率
    "frequency": 2
}
```

示例：

```json5
{
    "values": {
        // 让溅水游戏事件在第二频率振动
        "minecraft:splash": {
            "frequency": 2
        }
    }
}
```

## **`neoforge:villager_types`**

允许配置基于生物群系生成的村民类型，替代`VillagerType#BY_BIOME`（将在 1.22 中被忽略）。位于`neoforge/data_maps/worldgen/biome/villager_types.json`，其对象具有以下结构：

```json5
{
    // 此生物群系中生成的村民类型
    // 如果未为生物群系指定村民类型，则使用`minecraft:plains`
    "villager_type": "minecraft:desert"
    
}
```

示例：

```json5
{
    "values": {
        // 让丛林生物群系中的村民为沙漠类型
        "minecraft:jungle": {
            "villager_type": "minecraft:desert"
        }
    }
}
```

## **`neoforge:waxables`**

允许配置方块打蜡（用蜜脾右击）后变成的方块，替代`HoneycombItem#WAXABLES`。此数据映射也用于构建反向去蜡映射（用于用斧头刮除）。位于`neoforge/data_maps/block/waxables.json`，其对象具有以下结构：

```json5
{
    // 此方块的打蜡变体
    "waxed": "minecraft:iron_block"
}
```

示例：

```json5
{
    "values": {
        // 让金方块打蜡后变成铁方块
        "minecraft:gold_block": {
            "waxed": "minecraft:iron_block"
        }
    }
}
```

[datacomponent]: ../../../items/datacomponents.md
[datamap]: index.md