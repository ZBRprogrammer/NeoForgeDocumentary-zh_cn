# **战利品函数**(`Loot Functions`)

**战利品函数**(`Loot Functions`)可用于修改[战利品条目(`loot entry`)][entry]的结果，或修改[战利品池(`loot pool`)][pool]或[战利品表(`loot table`)][table]的多个结果。在这两种情况下，都会定义一个函数列表，并按顺序运行。在数据生成期间，可以通过调用 `#apply` 将战利品函数应用于 `LootPoolSingletonContainer.Builder<?>`、`LootPool.Builder` 和 `LootTable.Builder`。本文将概述可用的战利品函数。要创建自定义战利品函数，请参阅[自定义战利品函数(`Custom Loot Functions`)][custom]。

:::note
战利品函数不能应用于复合战利品条目（`CompositeEntryBase` 的子类及其关联的构建器类）。必须手动添加到每个单例条目。
:::

除了 `minecraft:sequence` 外，所有原版战利品函数都可以在 `conditions` 块中指定[战利品条件(`loot conditions`)][conditions]。如果其中任一条件失败，该函数将不会应用。在代码端，这由 `LootItemConditionalFunction` 控制，除 `SequenceFunction` 外的所有战利品函数都继承此类。

## `minecraft:set_item`

设置不同的物品用于结果物品堆。

```json5
{
    "function": "minecraft:set_item",
    // 要使用的物品。
    "item": "minecraft:dirt"
}
```

目前无法在数据生成期间创建此函数。

## `minecraft:set_count`

设置结果物品堆的物品数量。使用[数字提供器(`number provider`)][numberprovider]。

```json5
{
    "function": "minecraft:set_count",
    // 要设置的数量。
    "count": {
        "type": "minecraft:uniform",
        "min": 1,
        "max": 3
    },
    // 是否添加到现有值而非设置新值。可选，默认为 false。
    "add": true
}
```

在数据生成期间，调用 `SetItemCountFunction#setCount` 并传入所需的数字提供器和可选的 `add` 布尔值来构造此函数的构建器。

## `minecraft:explosion_decay`

应用爆炸衰减。物品有 1 / `explosion_radius` 的概率"存活"。此操作会根据物品数量多次运行。需要 `minecraft:explosion_radius` 战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:explosion_decay"
}
```

在数据生成期间，调用 `ApplyExplosionDecay#explosionDecay` 来构造此函数的构建器。

## `minecraft:limit_count`

将物品堆的数量限制在给定的 `IntRange` 范围内。

```json5
{
    "function": "minecraft:limit_count",
    // 要使用的限制。可以有 min、max 或两者。
    "limit": {
        "max": 32
    }
}
```

在数据生成期间，调用 `LimitCount#limitCount` 并传入所需的 `IntRange` 来构造此函数的构建器。

## `minecraft:set_custom_data`

在结果物品堆上设置自定义 NBT 数据。

```json5
{
    "function": "minecraft:set_custom_data",
    "tag": {
        "exampleproperty": 0
    }
}
```

在数据生成期间，调用 `SetCustomDataFunction#setCustomData` 并传入所需的[复合标签(`CompoundTag`)][nbt]来构造此函数的构建器。

:::warning
此函数通常应被视为已弃用。请改用 `minecraft:set_components`。
:::

## `minecraft:copy_custom_data`

将自定义 NBT 数据从方块实体或实体源复制到物品堆。对于方块实体，不鼓励使用此方法，请改用 `minecraft:copy_components` 或 `minecraft:set_contents`。对于实体，需要设置[实体目标(`entity target`)][entitytarget]。需要与指定源（实体目标或方块实体）对应的战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:copy_custom_data",
    // 要使用的源。有效值可以是实体目标、"block_entity"（使用战利品上下文的方块实体参数）或"storage"（命令存储）。如果是"storage"，也可以是额外指定要使用的命令存储路径的 JSON 对象。
    "source": "this",
    // 使用"storage"的示例。
    "source": {
        "type": "storage",
        "source": "examplepath"
    },
    // 复制操作。
    "ops": [
        {
            // 源路径和目标路径。在此示例中，我们将源中的"src"复制到目标中的"dest"。
            "source": "src",
            "target": "dest",
            // 合并策略。有效值为"replace"、"append"和"merge"。
            "op": "merge"
        }
    ]
}
```

在数据生成期间，调用 `CopyCustomDataFunction#copyData` 并传入关联的 `NbtProvider` 来获取构建器。然后调用 `Builder#copy` 并传入所需的源值、目标值以及合并策略（可选，默认为 `replace`）来构造此函数的构建器。

## `minecraft:set_components`

在物品堆上设置[数据组件(`data component`)][datacomponent]值。大多数原版用例有专门的函数，将在下文说明。

```json5
{
    "function": "minecraft:set_components",
    // 可以使用任何组件。在此示例中，我们将物品的染色颜色设置为红色。
    "components": {
        "dyed_color": {
            "rgb": 16711680
        }
    }
}
```

在数据生成期间，调用 `SetComponentsFunction#setComponent` 并传入所需的数据组件和值来构造此函数的构建器。

## `minecraft:copy_components`

将[数据组件(`data component`)][datacomponent]值从方块实体复制到物品堆。需要 `minecraft:block_entity` 战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:copy_components",
    // 该系统设计为允许多个源，但目前仅方块实体可用。
    "source": "block_entity",
    // 默认复制所有组件。"exclude"列表允许排除特定组件，"include"列表允许显式重新包含组件。两个字段都是可选的。
    "exclude": [],
    "include": []
}
```

在数据生成期间，调用 `CopyComponentsFunction#copyComponents` 并传入所需的数据源（通常是 `CopyComponentsFunction.Source.BLOCK_ENTITY`）来构造此函数的构建器。

## `minecraft:copy_state`

将方块状态属性复制到物品堆的 `block_state` [数据组件(`data component`)][datacomponent]中，用于尝试放置方块时。必须显式指定要复制的方块状态属性。需要 `minecraft:block_state` 战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:copy_state",
    // 预期的方块。如果与实际破坏的方块不匹配，则函数不运行。
    "block": "minecraft:oak_slab",
    // 要保存的方块状态属性。
    "properties": {
        "type": "top"
    }
}
```

在数据生成期间，调用 `CopyBlockState#copyState` 并传入方块来构造此条件的构建器。然后可以在构建器上使用 `#copy` 设置所需的方块状态属性值。

## `minecraft:set_contents`

设置物品堆的内容。

```json5
{
    "function": "minecraft:set_contents",
    // 要使用的内容组件。有效值为"container"、"bundle_contents"和"charged_projectiles"。
    "component": "container",
    // 要添加到内容中的战利品条目列表。
    "entries": [
        {
            "type": "minecraft:empty",
            "weight": 3
        },
        {
            "type": "minecraft:item",
            "item": "minecraft:arrow"
        }
    ]
}
```

在数据生成期间，调用 `SetContainerContents#setContents` 并传入所需的内容组件来构造此函数的构建器。然后在构建器上调用 `#withEntry` 来添加条目。

## `minecraft:modify_contents`

对物品堆的内容应用函数。

```json5
{
    "function": "minecraft:modify_contents",
    // 要使用的内容组件。有效值为"container"、"bundle_contents"和"charged_projectiles"。
    "component": "container",
    // 要使用的函数。
    "modifier": "apply_explosion_decay"
}
```

目前无法在数据生成期间创建此函数。

## `minecraft:set_loot_table`

在结果物品堆上设置容器战利品表。适用于箱子和其他在放置时保留此属性的战利品容器。

```json5
{
    "function": "minecraft:set_loot_table",
    // 要使用的战利品表的 ID。
    "name": "minecraft:entities/enderman",
    // 目标方块实体的方块实体类型 ID。
    "type": "minecraft:chest",
    // 生成战利品表的随机种子。可选，默认为 0。
    "seed": 42
}
```

在数据生成期间，调用 `SetContainerLootTable#withLootTable` 并传入所需的方块实体类型、战利品表资源键和可选的种子来构造此函数的构建器。

## `minecraft:set_name`

为结果物品堆设置名称。名称可以是[组件(`Component`)][component]而非文字字符串。也可以从[实体目标(`entity target`)][entitytarget]解析。如果适用，需要相应的实体战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:set_name",
    "name": "Funny Item",
    // 要使用的实体目标。
    "entity": "this",
    // 是设置自定义名称("custom_name")还是物品名称本身("item_name")。
    // 自定义名称以斜体显示并可在铁砧中更改，而物品名称不能。
    "target": "custom_name"
}
```

在数据生成期间，调用 `SetNameFunction#setName` 并传入所需的名称组件、目标名称和可选的实体目标来构造此函数的构建器。

## `minecraft:copy_name`

将[实体目标(`entity target`)][entitytarget]或方块实体的名称复制到结果物品堆中。需要与指定源（实体目标或方块实体）对应的战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:copy_name",
    // 实体目标，或"block_entity"如果要复制方块实体的名称。
    "source": "this"
}
```

在数据生成期间，调用 `CopyNameFunction#copyName` 并传入所需的实体源来构造此函数的构建器。

## `minecraft:set_lore`

为结果物品堆设置传说（工具提示行）。行可以是[组件(`Component`)][component]而非文字字符串。也可以从[实体目标(`entity target`)][entitytarget]解析。如果适用，需要相应的实体战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:set_lore",
    "lore": [
        "Funny Lore",
        "Funny Lore 2"
    ],
    // 使用的合并模式。有效值为：
    // - "append": 将条目附加到任何现有传说条目之后。
    // - "insert": 在特定位置插入条目。位置由名为"offset"的额外字段表示。"offset"是可选的，默认为 0。
    // - "replace_all": 移除所有先前条目，然后附加新条目。
    // - "replace_section": 移除部分条目，然后在该位置添加新条目。
    //   移除的部分由"offset"和可选的"size"字段表示。
    //   如果省略"size"，则使用"lore"中的行数。
    "mode": {
        "type": "insert",
        "offset": 0
    },
    // 要使用的实体目标。
    "entity": "this"
}
```

在数据生成期间，调用 `SetLoreFunction#setLore` 来构造此函数的构建器。然后在构建器上根据需要调用 `#addLine`、`#setMode` 和 `#setResolutionContext`。

## `minecraft:toggle_tooltips`

启用或禁用特定组件工具提示。

```json5
{
    "function": "minecraft:toggle_tooltips",
    "toggles": {
        // 所有值都是可选的。如果省略，这些值将使用物品堆上预先存在的值。
        // 预先存在的值通常为 true，除非已被其他函数修改。
        "minecraft:attribute_modifiers": false,
        "minecraft:can_break": false,
        "minecraft:can_place_on": false,
        "minecraft:dyed_color": false,
        "minecraft:enchantments": false,
        "minecraft:jukebox_playable": false,
        "minecraft:stored_enchantments": false,
        "minecraft:trim": false,
        "minecraft:unbreakable": false
    }
}
```

目前无法在数据生成期间创建此函数。

## `minecraft:enchant_with_levels`

使用指定等级数量随机附魔物品堆。使用[数字提供器(`number provider`)][numberprovider]。

```json5
    {
    "function": "minecraft:enchant_with_levels",
    // 要使用的等级数量。
    "levels": {
        "type": "minecraft:uniform",
        "min": 10,
        "max": 30
    },
    // 可能的附魔列表。可选，默认为物品适用的所有附魔。
    "options": [
        "minecraft:sharpness",
        "minecraft:fire_aspect"
    ]
}
```

在数据生成期间，调用 `EnchantWithLevelsFunction#enchantWithLevels` 并传入所需的数字提供器来构造此函数的构建器。然后，如果需要，在构建器上使用 `#fromOptions` 设置附魔列表。

## `minecraft:enchant_randomly`

为物品添加一个随机附魔。

```json5
{
    "function": "minecraft:enchant_randomly",
    // 可能的附魔列表。可选，默认为所有附魔。
    "options": [
        "minecraft:sharpness",
        "minecraft:fire_aspect"
    ],
    // 是否仅允许兼容的附魔或任何附魔。可选，默认为 true。
    "only_compatible": true
}
```

在数据生成期间，调用 `EnchantRandomlyFunction#randomEnchantment` 或 `EnchantRandomlyFunction#randomApplicableEnchantment` 来构造此函数的构建器。然后，如果需要，在构建器上调用 `#withEnchantment` 或 `#withOneOf`。

## `minecraft:set_enchantments`

在结果物品堆上设置附魔。

```json5
{
    "function": "minecraft:set_enchantments",
    // 附魔到数字提供器的映射。
    "enchantments": {
        "minecraft:fire_aspect": 2,
        "minecraft:sharpness": {
        "type": "minecraft:uniform",
        "min": 3,
        "max": 5,
        }
    },
    // 是否将附魔等级添加到现有等级而非覆盖。可选，默认为 false。
    "add": true
}
```

在数据生成期间，调用 `new SetEnchantmentsFunction.Builder` 并传入 `add` 布尔值（可选）来构造此函数的构建器。然后调用 `#withEnchantment` 来添加要设置的附魔。

## `minecraft:enchanted_count_increase`

基于附魔值增加物品堆数量。使用[数字提供器(`number provider`)][numberprovider]。需要 `minecraft:attacking_entity` 战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:enchanted_count_increase",
    // 要使用的附魔。
    "enchantment": "minecraft:fortune",
    // 每等级增加的数量。数字提供器每函数滚动一次，而非每等级一次。
    "count": {
        "type": "minecraft:uniform",
        "min": 1,
        "max": 3
    },
    // 堆叠大小限制，无论附魔等级如何都不会超过此限制。可选。
    "limit": 5
}
```

在数据生成期间，调用 `EnchantedCountIncreaseFunction#lootingMultiplier` 并传入所需的数字提供器来构造此函数的构建器。然后，可选地在构建器上调用 `#setLimit`。

## `minecraft:apply_bonus`

基于附魔值和各种公式应用物品堆数量的增加。需要 `minecraft:tool` 战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:apply_bonus",
    // 要查询的附魔值。
    "enchantment": "minecraft:fortune",
    // 要使用的公式。有效值为：
    // - "minecraft:binomial_with_bonus_count": 基于二项分布应用加成，其中
    //   n = 附魔等级 + 额外值，p = 概率。
    // - "minecraft:ore_drops": 基于矿石掉落的特殊公式应用加成，包括随机性。
    // - "minecraft:uniform_bonus_count": 基于附魔等级乘以常数乘数添加加成。
    "formula": "ore_drops",
    // 参数值，取决于公式。
    // 如果公式是"minecraft:binomial_with_bonus_count"，需要"extra"和"probability"。
    // 如果公式是"minecraft:ore_drops"，不需要参数。
    // 如果公式是"minecraft:uniform_bonus_count"，需要"bonusMultiplier"。
    "parameters": {}
}
```

在数据生成期间，调用 `ApplyBonusCount#addBonusBinomialDistributionCount`、`ApplyBonusCount#addOreBonusCount` 或 `ApplyBonusCount#addUniformBonusCount` 并传入附魔和其他所需参数（取决于公式）来构造此函数的构建器。

## `minecraft:furnace_smelt`

尝试像在熔炉中一样熔炼物品，如果无法熔炼则返回未修改的物品堆。

```json5
{
    "function": "minecraft:furnace_smelt"
}
```

在数据生成期间，调用 `SmeltItemFunction#smelted` 来构造此函数的构建器。

## `minecraft:set_damage`

在结果物品堆上设置耐久度伤害值。使用[数字提供器(`number provider`)][numberprovider]。

```json5
{
    "function": "minecraft:set_damage",
    // 要设置的伤害值。
    "damage": {
        "type": "minecraft:uniform",
        "min": 10,
        "max": 300
    },
    // 是否添加到现有伤害值而非设置新值。可选，默认为 false。
    "add": true
}
```

在数据生成期间，调用 `SetItemDamageFunction#setDamage` 并传入所需的数字提供器和可选的 `add` 布尔值来构造此函数的构建器。

## `minecraft:set_attributes`

向结果物品堆添加[属性修饰符(`attribute modifiers`)][attributemodifier]列表。

```json5
{
    "function": "minecraft:set_attributes",
    // 属性修饰符列表。
    "modifiers": [
        {
            // 修饰符的资源位置 ID。应以前缀`您的模组 ID`。
            "id": "examplemod:example_modifier",
            // 修饰符对应的属性 ID。
            "attribute": "minecraft:generic.attack_damage",
            // 属性修饰符操作。
            // 有效值为"add_value"、"add_multiplied_base"和"add_multiplied_total"。 
            "operation": "add_value",
            // 修饰符的数量。也可以是数字提供器。
            "amount": 5,
            // 修饰符适用的槽位。有效值为"any"（任何物品栏槽位）、
            // "mainhand"、"offhand"、"hand"（主手/副手/双手）、
            // "feet"、"legs"、"chest"、"head"、"armor"（靴子/护腿/胸甲/头盔/任何盔甲槽）
            // 和"body"（马铠及类似槽位）。
            "slot": "armor"
        }
    ],
    // 是否替换现有值而非添加到其中。可选，默认为 true。
    "replace": false
}
```

在数据生成期间，调用 `SetAttributesFunction#setAttributes` 来构造此函数的构建器。然后在构建器上使用 `#withModifier` 添加修饰符。使用 `SetAttributesFunction#modifier` 获取修饰符。

## `minecraft:set_potion`

在结果物品堆上设置药水。

```json5
{
    "function": "minecraft:set_potion",
    // 药水的 ID。
    "id": "minecraft:strength"
}
```

在数据生成期间，调用 `SetPotionFunction#setPotion` 并传入所需的药水来构造此函数的构建器。

## `minecraft:set_stew_effect`

在结果物品堆上设置炖煮效果列表。

```json5
{
    "function": "minecraft:set_stew_effect",
    // 要设置的效果。
    "effects": [
        {
        // 效果 ID。
        "type": "minecraft:fire_resistance",
        // 效果持续时间（刻）。也可以是数字提供器。
        "duration": 100
        }
    ]
}
```

在数据生成期间，调用 `SetStewEffectFunction#stewEffect` 来构造此函数的构建器。然后在构建器上调用 `#withModifier`。

## `minecraft:set_ominous_bottle_amplifier`

在结果物品堆上设置预兆瓶放大器。使用[数字提供器(`number provider`)][numberprovider]。

```json5
{
    "function": "minecraft:set_ominous_bottle_amplifier",
    // 要使用的放大器。
    "amplifier": {
        "type": "minecraft:uniform",
        "min": 1,
        "max": 3
    }
}
```

在数据生成期间，调用 `SetOminousBottleAmplifierFunction#setAmplifier` 并传入所需的数字提供器来构造此函数的构建器。

## `minecraft:exploration_map`

如果结果物品是地图，则将其转换为探险家地图。需要 `minecraft:origin` 战利品参数，如果该参数不存在则不执行修改。

```json5
{
    "function": "minecraft:exploration_map",
    // 结构标签，包含探险家地图可以引导到的结构。
    // 可选，默认为"minecraft:on_treasure_maps"，默认仅包含埋藏的宝藏。
    "destination": "minecraft:eye_of_ender_located",
    // 要使用的地图装饰类型。有关可用值，请参阅 MapDecorationTypes 类。
    // 可选，默认为"minecraft:mansion"。
    "decoration": "minecraft:target_x",
    // 要使用的缩放级别。可选，默认为 2。
    "zoom": 4,
    // 要使用的搜索半径。可选，默认为 50。
    "search_radius": 25,
    // 搜索结构时是否跳过现有区块。可选，默认为 true。
    "skip_existing_chunks": true
}
```

在数据生成期间，调用 `ExplorationMapFunction#makeExplorationMap` 来构造此函数的构建器。然后，如果需要，在构建器上调用各种设置器。

## `minecraft:fill_player_head`

基于给定的[实体目标(`entity target`)][entitytarget]在结果物品堆上设置玩家头颅所有者。需要相应的战利品参数，如果该参数不存在或不解析为玩家，则不修改物品堆。

```json5
{
    "function": "minecraft:fill_player_head",
    // 要使用的实体目标。如果未解析为玩家，则不修改物品堆。
    "entity": "this_entity"
}
```

在数据生成期间，调用 `FillPlayerHead#fillPlayerHead` 并传入所需的实体目标来构造此函数的构建器。

## `minecraft:set_banner_pattern`

在结果物品堆上设置旗帜图案。此函数适用于旗帜本身，而非旗帜图案物品。

```json5
{
    "function": "minecraft:set_banner_patterns",
    // 旗帜图案层列表。
    "patterns": [
        {
            // 要使用的旗帜图案 ID。
            "pattern": "minecraft:globe",
            // 图层的染料颜色。
            "color": "light_blue"
        }
    ],
    // 是否附加到现有图层而非替换它们。
    "append": true
}
```

在数据生成期间，调用 `SetBannerPatternFunction#setBannerPattern` 并传入 `append` 布尔值来构造此函数的构建器。然后调用 `#addPattern` 向函数添加图案。

## `minecraft:set_instrument`

在结果物品堆上设置乐器标签。

```json5
{
    "function": "minecraft:set_instrument",
    // 要使用的乐器标签。
    "options": "minecraft:goat_horns"
}
```

在数据生成期间，调用 `SetInstrumentFunction#setInstrumentOptions` 并传入所需的乐器标签来构造此函数的构建器。

## `minecraft:set_fireworks`

```json5
{
    "function": "minecraft:set_fireworks",
    // 要使用的爆炸效果。可选，如果省略则使用现有的数据组件值。
    "explosions": [
        {
            // 要使用的烟花爆炸形状。有效的原版值为"small_ball"、"large_ball"、"star"、"creeper"和"burst"。可选，默认为"small_ball"。
            "shape": "star",
            // 要使用的颜色。可选，默认为空列表。
            "colors": [
                16711680,
                65280
            ],
            // 要使用的褪色颜色。可选，默认为空列表。
            "fade_colors": [
                65280,
                255
            ],
            // 爆炸是否有拖尾。可选，默认为 false。
            "has_trail": true,
            // 爆炸是否有闪烁。可选，默认为 false。
            "has_twinkle": true
        }
    ],
    // 烟花的飞行持续时间。可选，如果省略则使用现有的数据组件值。
    "flight_duration": 5
}
```

目前无法在数据生成期间创建此函数。

## `minecraft:set_firework_explosion`

在结果物品堆上设置烟花爆炸效果。

```json5
{
    "function": "minecraft:set_firework_explosion",
    // 要使用的烟花爆炸形状。有效的原版值为"small_ball"、"large_ball"、"star"、"creeper"和"burst"。可选，默认为"small_ball"。
    "shape": "star",
    // 要使用的颜色。可选，默认为空列表。
    "colors": [
        16711680,
        65280
    ],
    // 要使用的褪色颜色。可选，默认为空列表。
    "fade_colors": [
        65280,
        255
    ],
    // 爆炸是否有拖尾。可选，默认为 false。
    "trail": true,
    // 爆炸是否有闪烁。可选，默认为 false。
    "twinkle": true
}
```

目前无法在数据生成期间创建此函数。

## `minecraft:set_book_cover`

设置成书的非页面特定内容。

```json5
{
    "function": "minecraft:set_book_cover",
    // 书名。可选，如果省略则书名保持不变。
    "title": "Hello World!",
    // 书作者。可选，如果省略则作者保持不变。
    "author": "Steve",
    // 书代次，即被复制的次数。限制在 0 到 3 之间。
    // 可选，如果省略则代次保持不变。
    "generation": 2
}
```

在数据生成期间，调用 `new SetBookCoverFunction` 并传入所需参数来构造此函数的构建器。

## `minecraft:set_written_book_pages`

设置成书的页面。

```json5
{
    "function": "minecraft:set_written_book_pages",
    // 要设置的页面列表。
    "pages": [
        "Hello World!",
        "Hello World on page 2!",
        "Never Gonna Give You Up!"
    ],
    // 使用的合并模式。有效值为：
    // - "append": 将条目附加到任何现有条目之后。
    // - "insert": 在特定位置插入条目。位置由名为"offset"的额外字段表示。"offset"是可选的，默认为 0。
    // - "replace_all": 移除所有先前条目，然后附加新条目。
    // - "replace_section": 移除部分条目，然后在该位置添加新条目。
    //   移除的部分由"offset"和可选的"size"字段表示。
    //   如果省略"size"，则使用"lore"中的行数。
    "mode": {
        "type": "insert",
        "offset": 0
    }
}
```

目前无法在数据生成期间创建此函数。

## `minecraft:set_writable_book_pages`

设置书与笔的页面。

```json5
{
    "function": "minecraft:set_writable_book_pages",
    // 要设置的页面列表。
    "pages": [
        "Hello World!",
        "Hello World on page 2!",
        "Never Gonna Give You Up!"
    ],
    // 使用的合并模式。有效值为：
    // - "append": 将条目附加到任何现有条目之后。
    // - "insert": 在特定位置插入条目。位置由名为"offset"的额外字段表示。"offset"是可选的，默认为 0。
    // - "replace_all": 移除所有先前条目，然后附加新条目。
    // - "replace_section": 移除部分条目，然后在该位置添加新条目。
    //   移除的部分由"offset"和可选的"size"字段表示。
    //   如果省略"size"，则使用"lore"中的行数。
    "mode": {
        "type": "insert",
        "offset": 0
    }
}
```

目前无法在数据生成期间创建此函数。

## `minecraft:set_custom_model_data`

设置结果物品堆在渲染期间使用的自定义模型数据。

```json5
{
    "function": "minecraft:set_custom_model_data",
    // 在具有 `minecraft:custom_model_data` 范围属性的客户端物品的指定索引时，用于物品模型选择的浮点数。
    "floats": [
        // 当"index": 0 时，将选择属性小于 0.5 的模型
        0.5,
        // 当"index": 1 时，将选择属性小于 0.25 的模型
        0.25
    ],
    // 在具有 `minecraft:custom_model_data` 条件属性的客户端物品的指定索引时，用于物品模型选择的布尔值。
    "flags": [
        // 当"index": 0 时，将选择条件为 true 的模型
        true,
        // 当"index": 1 时，将选择条件为 false 的模型
        false
    ],
    // 在具有 `minecraft:custom_model_data` 选择属性的客户端物品的指定索引时，用于物品模型选择的字符串。
    "strings": [
        // 当"index": 0 时，将选择"dummy"情况的模型
        "dummy",
        // 当"index": 1 时，将选择"example"情况的模型
        "example"
    ],
    // 在具有 `minecraft:custom_model_data` 着色源的客户端物品的指定索引时，使用的色调颜色。
    // 0xFF000000 与此值进行 OR 运算以得到不透明颜色。
    "colors": [
        // 当"index": 0 时为蓝色
        255,
        // 当"index": 1 时为绿色
        65280
    ]
}
```

在数据生成期间，调用 `new SetCustomModelDataFunction()` 并传入条件列表、可选的数字提供器、布尔值、字符串和数字提供器来构造关联对象。

## `minecraft:filtered`

此函数接受一个 `ItemPredicate`，该谓词会针对生成的物品堆进行检查；如果检查成功，则运行其他函数。`ItemPredicate` 可以指定有效的物品 ID 列表（`items`）、物品数量的最小/最大范围（`count`）、`DataComponentPredicate`（`components`）和 `ItemSubPredicate` 映射（`predicates`）；所有字段都是可选的。

```json5
{
    "function": "minecraft:filtered",
    // 要使用的自定义模型数据值。也可以是数字提供器。
    "item_filter": {
        "items": [
            "minecraft:diamond_shovel"
        ]
    },
    // 要运行的其他战利品函数，可以是战利品修饰符文件或函数的内联列表。
    "modifier": "examplemod:example_modifier"
}
```

目前无法在数据生成期间创建此函数。

:::warning
此函数通常应被视为已弃用。请改用带有 `minecraft:match_tool` 条件的传递函数。
:::

## `minecraft:reference`

此函数引用物品修饰符并将其应用于结果物品堆。有关更多信息，请参阅[物品修饰符(`Item Modifiers`)][itemmodifiers]。

```json5
{
    "function": "minecraft:reference",
    // 引用位于 data/examplemod/item_modifier/example_modifier.json 的物品修饰符文件。
    "name": "examplemod:example_modifier"
}
```

在数据生成期间，调用 `FunctionReference#functionReference` 并传入被引用谓词文件的 ID 来构造此函数的构建器。

## `minecraft:sequence`

此函数按顺序运行其他战利品函数。

```json5
{
    "function": "minecraft:sequence",
    // 要运行的函数列表。
    "functions": [
        {
            "function": "minecraft:set_count",
            // ...
        },
        {
            "function": "minecraft:explosion_decay"
        }
    ],
}
```

在数据生成期间，调用 `SequenceFunction#of` 并传入其他函数来构造此条件的构建器。

## 另请参阅

- [Minecraft Wiki][mcwiki]上的[物品修饰符(`Item Modifiers`)][itemmodifiers]

[attributemodifier]: ../../../entities/attributes.md#attribute-modifiers
[component]: ../../client/i18n.md#components
[conditions]: lootconditions
[custom]: custom.md#custom-loot-functions
[datacomponent]: ../../../items/datacomponents.md
[entitytarget]: index.md#entity-targets
[entry]: index.md#loot-entry
[itemmodifiers]: https://minecraft.wiki/w/Item_modifier#JSON_format
[mcwiki]: https://minecraft.wiki
[nbt]: ../../../datastorage/nbt.md
[numberprovider]: index.md#number-provider
[pool]: index.md#loot-pool
[table]: index.md#loot-table