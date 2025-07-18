# 战利品条件（`Loot Conditions`）
战利品条件（`Loot Conditions`）可用于检查[战利品条目][entry]或[战利品池][pool]在当前情境下是否应被使用。在这两种情况下，都会定义一个条件列表；只有当所有条件都满足时，条目或池才会被使用。在数据生成（`datagen`）期间，通过调用`#when`并传入所需条件的实例，可将它们添加到`LootPoolEntryContainer.Builder<?>`或`LootPool.Builder`中。本文将概述可用的战利品条件。如需创建自己的战利品条件，请参阅[自定义战利品条件][custom]。

## `minecraft:inverted`
该条件接收另一个条件并反转其结果。需要其他条件所需的任何战利品参数。
```json5
{
    "condition": "minecraft:inverted",
    "term": {
        // 其他某个战利品条件。
    }
}
```
在数据生成期间，调用`InvertedLootItemCondition#invert`并传入要反转的条件，以构建该条件的构建器（`builder`）。

## `minecraft:all_of`
该条件接收任意数量的其他条件，且仅当所有子条件都返回true时才返回true。如果列表为空，则返回false。需要其他条件所需的任何战利品参数。
```json5
{
    "condition": "minecraft:all_of",
    "terms": [
        {
            // 一个战利品条件。
        },
        {
            // 另一个战利品条件。
        },
        {
            // 又一个战利品条件。
        }
    ]
}
```
在数据生成期间，调用`AllOfCondition#allOf`并传入所需的一个或多个条件，以构建该条件的构建器。

## `minecraft:any_of`
该条件接收任意数量的其他条件，且当至少有一个子条件返回true时返回true。如果列表为空，则返回false。需要其他条件所需的任何战利品参数。
```json5
{
    "condition": "minecraft:any_of",
    "terms": [
        {
            // 一个战利品条件。
        },
        {
            // 另一个战利品条件。
        },
        {
            // 又一个战利品条件。
        }
    ]
}
```
在数据生成期间，调用`AnyOfCondition#anyOf`并传入所需的一个或多个条件，以构建该条件的构建器。

## `minecraft:random_chance`
该条件接收一个代表0到1之间概率的[数字提供器][numberprovider]，并根据该概率随机返回true或false。数字提供器（`number provider`）通常不应返回`[0, 1]`区间之外的值。
```json5
{
    "condition": "minecraft:random_chance",
    // 该条件适用的恒定50%概率。
    "chance": 0.5
}
```
在数据生成期间，调用`LootItemRandomChanceCondition#randomChance`并传入数字提供器（`number provider`）或（恒定的）浮点值，以构建该条件的构建器。

## `minecraft:random_chance_with_enchanted_bonus`
该条件接收一个附魔ID、一个[数字提供器][numberprovider]（`LevelBasedValue`）和一个恒定的备用浮点值。如果存在指定的附魔，则查询`LevelBasedValue`以获取值。如果不存在指定的附魔，或者无法从`LevelBasedValue`中获取值，则使用恒定的备用值。然后该条件会随机返回true或false，之前确定的值表示返回true的概率。需要`minecraft:attacking_entity`参数，如果该参数不存在，则默认使用等级0。
```json5
{
    "condition": "minecraft:random_chance_with_enchanted_bonus",
    // 每级掠夺附魔增加20%的成功概率。
    "enchantment": "minecraft:looting",
    "enchanted_chance": {
        "type": "linear",
        "base": 0.2,
        "per_level_above_first": 0.2
    },
    // 如果没有掠夺附魔，则始终失败。
    "unenchanted_chance": 0.0
}
```
在数据生成期间，调用`LootItemRandomChanceWithEnchantedBonusCondition#randomChanceAndLootingBoost`并传入注册表查找器（`HolderLookup.Provider`）、基础值和每级增加值，以构建该条件的构建器。或者，调用`new LootItemRandomChanceWithEnchantedBonusCondition`以进一步指定值。

## `minecraft:value_check`
该条件接收一个[数字提供器][numberprovider]和一个`IntRange`，如果数字提供器的结果在该范围内，则返回true。
```json5
{
    "condition": "minecraft:value_check",
    // 可以是任何数字提供器（`number provider`）。
    "value": {
        "type": "minecraft:uniform",
        "min": 0.0,
        "max": 10.0
    },
    // 带有最小值/最大值的范围。
    "range": {
        "min": 2.0,
        "max": 5.0
    }
}
```
在数据生成期间，调用`ValueCheckCondition#hasValue`并传入数字提供器（`number provider`）和范围，以构建该条件的构建器。

## `minecraft:time_check`
该条件检查世界时间是否在`IntRange`内。可选地，可以提供一个`period`参数来对时间进行取模运算；例如，如果`period`为24000（一个游戏日/夜周期为24000刻），则可用于检查一天中的时间。
```json5
{
    "condition": "minecraft:time_check",
    // 可选，可省略。如果省略，则不进行取模运算。
    // 这里我们使用24000，这是一个游戏日/夜周期的长度。
    "period": 24000,
    // 带有最小值/最大值的范围。此示例检查时间是否在0到12000之间。
    // 结合上面指定的24000取模运算，此示例检查当前是否为白天。
    "value": {
        "min": 0,
        "max": 12000
    }
}
```
在数据生成期间，调用`TimeCheck#time`并传入所需范围，以构建该条件的构建器。然后可以使用`#setPeriod`在构建器上设置`period`值。

## `minecraft:weather_check`
该条件检查当前天气是否在下雨和打雷。
```json5
{
    "condition": "minecraft:weather_check",
    // 可选。如果未指定，则不检查下雨状态。
    "raining": true,
    // 可选。如果未指定，则不检查打雷状态。
    // 指定"raining": true和"thundering": true在功能上等同于仅指定"thundering": true，因为雷暴发生时总会下雨。
    "thundering": false
}
```
在数据生成期间，调用`WeatherCheck#weather`以构建该条件的构建器。然后可以分别使用`#setRaining`和`#setThundering`在构建器上设置`raining`和`thundering`的值。

## `minecraft:location_check`
该条件接收一个`LocationPredicate`和每个轴方向的可选偏移值。`LocationPredicate`允许检查诸如位置本身、该位置的方块或流体状态、该位置的维度、生物群系或结构、光照等级、天空是否可见等条件。所有可能的值可以在`LocationPredicate`类定义中查看。需要`minecraft:origin`战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:location_check",
    "predicate": {
        // 如果目标在 Nether（下界）中的任何位置，则成功。
        "dimension": "the_nether"
    },
    // 可选的位置偏移值。仅在以某种方式检查位置时相关。
    // 必须同时提供所有偏移值，或者都不提供。
    "offsetX": 10,
    "offsetY": 10,
    "offsetZ": 10
}
```
在数据生成期间，调用`LocationCheck#checkLocation`并传入`LocationPredicate`以及可选的`BlockPos`，以构建该条件的构建器。

## `minecraft:block_state_property`
该条件检查被破坏的方块状态中指定的方块状态属性是否具有指定的值。需要`minecraft:block_state`战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:block_state_property",
    // 预期的方块。如果与实际被破坏的方块不匹配，则该条件失败。
    "block": "minecraft:oak_slab",
    // 要匹配的方块状态属性。未指定的属性可以是任何值。
    // 在本示例中，我们希望仅当顶部台阶（无论是否含水）被破坏时才成功。
    // 如果此处指定了方块不存在的属性，将打印日志警告。
    "properties": {
        "type": "top"
    }
}
```
在数据生成期间，调用`LootItemBlockStatePropertyCondition#hasBlockStateProperties`并传入方块，以构建该条件的构建器。然后可以使用`#setProperties`在构建器上设置所需的方块状态属性值。

## `minecraft:survives_explosion`
该条件随机销毁掉落物。掉落物幸存的概率为1除以`explosion_radius`战利品参数。除信标或龙蛋等极少数例外情况外，所有方块掉落物都使用此函数。需要`minecraft:explosion_radius`战利品参数，如果该参数不存在，则始终成功。
```json5
{
    "condition": "minecraft:survives_explosion"
}
```
在数据生成期间，调用`ExplosionCondition#survivesExplosion`以构建该条件的构建器。

## `minecraft:match_tool`
该条件接收一个`ItemPredicate`，并根据`tool`战利品参数进行检查。`ItemPredicate`可以指定有效物品ID的列表（`items`）、物品数量的最小/最大范围（`count`）、`DataComponentPredicate`（`components`）以及`ItemSubPredicate`的映射（`predicates`）；所有字段都是可选的。需要`minecraft:tool`战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:match_tool",
    // 匹配下界合金镐或下界合金斧。
    "predicate": {
        "items": [
            "minecraft:netherite_pickaxe",
            "minecraft:netherite_axe"
        ]
    }
}
```
在数据生成期间，调用`MatchTool#toolMatches`并传入`ItemPredicate.Builder`以构建该条件的构建器。

## `minecraft:enchantment_active`
该条件返回某个附魔是否激活。需要`minecraft:enchantment_active`战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:enchantment_active",
    // 附魔是否应处于激活状态（true为激活，false为未激活）。
    "active": true
}
```
在数据生成期间，调用`EnchantmentActiveCheck#enchantmentActiveCheck`或`#enchantmentInactiveCheck`以构建该条件的构建器。

## `minecraft:table_bonus`
该条件与`minecraft:random_chance_with_enchanted_bonus`类似，但使用固定值而非随机值。需要`minecraft:tool`战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:table_bonus",
    // 如果存在时运附魔，则应用奖励。
    "enchantment": "minecraft:fortune",
    // 每级对应的概率。本示例中，未附魔时成功概率为20%，
    // 附魔1级时为30%，附魔2级及以上时为60%。
    "chances": [0.2, 0.3, 0.6]
}
```
在数据生成期间，调用`BonusLevelTableCondition#bonusLevelFlatChance`并传入附魔ID和概率，以构建该条件的构建器。

## `minecraft:entity_properties`
该条件根据[实体目标][entitytarget]检查给定的`EntityPredicate`。`EntityPredicate`可以检查实体类型、药水效果、nbt值、装备、位置等。
```json5
{
    "condition": "minecraft:entity_properties",
    // 要使用的实体目标。有效值为"this"、"attacker"、"direct_attacker"或"attacking_player"。
    // 它们分别对应"this_entity"、"attacking_entity"、"direct_attacking_entity"和"last_damage_player"战利品参数。
    "entity": "attacker",
    // 仅当目标是猪时才成功。该谓词也可以为空，这可用于检查指定的实体目标是否已设置。
    "predicate": {
        "type": "minecraft:pig"
    }
}
```
在数据生成期间，调用`LootItemEntityPropertyCondition#entityPresent`并传入实体目标，或者调用`LootItemEntityPropertyCondition#hasProperties`并传入实体目标和`EntityPredicate`，以构建该条件的构建器。

## `minecraft:damage_source_properties`
该条件根据伤害源战利品参数检查给定的`DamageSourcePredicate`。需要`minecraft:origin`和`minecraft:damage_source`战利品参数，如果这些参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:damage_source_properties",
    "predicate": {
        // 检查来源实体是否为僵尸。
        "source_entity": {
            "type": "zombie"
        }
    }
}
```
在数据生成期间，调用`DamageSourceCondition#hasDamageSource`并传入`DamageSourcePredicate.Builder`，以构建该条件的构建器。

## `minecraft:killed_by_player`
该条件判断击杀是否为玩家击杀。某些实体掉落物会使用此条件，例如烈焰人掉落的烈焰棒。需要`minecraft:last_player_damage`战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:killed_by_player"
}
```
在数据生成期间，调用`LootItemKilledByPlayerCondition#killedByPlayer`以构建该条件的构建器。

## `minecraft:entity_scores`
该条件检查[实体目标][entitytarget]的计分板。需要与指定实体目标对应的战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "minecraft:entity_scores"
    // 要使用的实体目标。有效值为"this"、"attacker"、"direct_attacker"或"attacking_player"。
    // 它们分别对应"this_entity"、"attacking_entity"、"direct_attacking_entity"和"last_damage_player"战利品参数。
    "entity": "attacker",
    // 必须在给定范围内的一系列计分板值。
    "scores": {
        "score1": {
            "min": 0,
            "max": 100
        },
        "score2": {
            "min": 10,
            "max": 20
        }
    }
}
```
在数据生成期间，调用`EntityHasScoreCondition#hasScores`并传入实体目标，以构建该条件的构建器。然后可以使用`#withScore`在构建器上添加所需的分数。

## `minecraft:reference`
该条件引用一个谓词文件并返回其结果。有关更多信息，请参见[物品谓词][predicate]。
```json5
{
    "condition": "minecraft:reference",
    // 引用位于data/examplemod/predicate/example_predicate.json的谓词文件。
    "name": "examplemod:example_predicate"
}
```
在数据生成期间，调用`ConditionReference#conditionReference`并传入所引用谓词文件的ID，以构建该条件的构建器。

## `neoforge:loot_table_id`
该条件仅当周围的战利品表ID匹配时才返回true。这通常在[全局战利品修改器][glm]中使用。
```json5
{
    "condition": "neoforge:loot_table_id",
    // 仅当战利品表为泥土的战利品表时适用。
    "loot_table_id": "minecraft:blocks/dirt"
}
```
在数据生成期间，调用`LootTableIdCondition#builder`并传入所需的战利品表ID，以构建该条件的构建器。

## `neoforge:can_item_perform_ability`
该条件仅当`tool`战利品上下文参数（`LootContextParams.TOOL`）中的物品（通常是用于破坏方块或击杀实体的物品）能够执行指定的[`ItemAbility`][itemability]时才返回true。需要`minecraft:tool`战利品参数，如果该参数不存在，则始终失败。
```json5
{
    "condition": "neoforge:can_item_perform_ability",
    // 仅当工具能够像斧头一样剥原木时适用。
    "ability": "axe_strip"
}
```
在数据生成期间，调用`CanItemPerformAbility#canItemPerformAbility`并传入所需物品能力的ID，以构建该条件的构建器。

## 另请参阅
[Minecraft Wiki][mcwiki]上的[物品谓词][predicatejson]

[custom]: custom.md#custom-loot-conditions
[entitytarget]: index.md#entity-targets
[entry]: index.md#loot-entry
[glm]: glm.md
[itemability]: ../../../items/tools.md#itemabilitys
[mcwiki]: https://minecraft.wiki
[numberprovider]: index.md#number-provider
[pool]: index.md#loot-pool
[predicate]: https://minecraft.wiki/w/Predicate
[predicatejson]: https://minecraft.wiki/w/Predicate#JSON_format
[registry]: ../../../concepts/registries.md