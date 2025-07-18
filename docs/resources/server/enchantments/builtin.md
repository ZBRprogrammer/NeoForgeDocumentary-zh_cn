import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 内置附魔效果组件

原版Minecraft为[附魔(Enchantment)]定义提供了多种效果组件。本文将解释每种组件的用法及其代码实现。

## 数值效果组件(Value Effect Components)

*另见Minecraft Wiki上的[数值效果组件]*

数值效果组件用于修改游戏中数值的附魔，由`EnchantmentValueEffect`类实现。若多个数值效果组件影响同一数值（如多个附魔），所有效果均会生效。

数值效果组件可对给定值执行以下操作：
- **`minecraft:set`(`set`)**：覆盖基于等级的给定值
- **`minecraft:add`(`add`)**：将指定等级值加到原值
- **`minecraft:multiply`(`multiply`)**：将指定等级系数乘以原值
- **`minecraft:remove_binomial`(`remove_binomial`)**：按二项分布概率计算（基于等级）。成功时从值中减1（许多值本质是标志位，1表示开启，0表示关闭）
- **`minecraft:all_of`(`all_of`)**：接受其他数值效果列表并按顺序应用

锋利(Sharpness)附魔使用数值效果组件`minecraft:damage`实现效果：

<Tabs>
<TabItem value="sharpness.json" label="JSON">

```json5
"effects": {
    // 效果组件类型为"minecraft:damage"
    // 表示修改武器伤害
    "minecraft:damage": [
        {
            "effect": {
                // 使用"minecraft:add"数值效果类型
                "type": "minecraft:add",
                // 基于等级的值：基础值1.0，每级增加0.5
                "value": {
                    "type": "minecraft:linear",
                    "base": 1.0,
                    "per_level_above_first": 0.5
                }
            }
        }
    ]
}
```

</TabItem>
<TabItem value="sharpness.datagen" label="Datagen">

```java
// 数据生成时传入附魔的'effects'
DataComponentMap.builder().set(
    EnchantmentEffectComponents.DAMAGE,
    List.of(new ConditionalEffect<>(
        new AddValue(LevelBasedValue.perLevel(1.0F, 0.5F)),
        Optional.empty()))
).build()
```

</TabItem>
</Tabs>

`value`块内的对象是[基于等级的值(LevelBasedValue)]，可使效果强度随等级变化。

使用`EnchantmentValueEffect#process`方法基于数值操作调整值：
```java
// `valueEffect`是EnchantmentValueEffect实例
// `enchantLevel`表示附魔等级
float baseValue = 1.0;
float modifiedValue = valueEffect.process(enchantLevel, server.random, baseValue);
```

### 原版数值效果组件类型

#### 定义为`DataComponentType<EnchantmentValueEffect>`
- **`minecraft:crossbow_charge_time`(`crossbow_charge_time`)**：修改弩的蓄力时间（秒）。快速装填(Quick Charge)使用
- **`minecraft:trident_spin_attack_strength`(`trident_spin_attack_strength`)**：修改三叉戟旋击强度。激流(Riptide)使用

#### 定义为`DataComponentType<List<ConditionalEffect<EnchantmentValueEffect>>>`

护甲相关：
- **`minecraft:armor_effectiveness`(`armor_effectiveness`)**：决定武器对护甲的效果（0无保护，1正常保护）。穿透(Breach)使用
- **`minecraft:damage_protection`(`damage_protection`)**：每点伤害减免减少4%伤害（最高80%）。爆炸保护(Blast Protection)/摔落保护(Feather Falling)/火焰保护(Fire Protection)/保护(Protection)/弹射物保护(Projectile Protection)使用

攻击相关：
- **`minecraft:damage`(`damage`)**：修改武器攻击伤害。锋利(Sharpness)/穿刺(Impaling)/节肢杀手(Bane of Arthropods)/力量(Power)/亡灵杀手(Smite)使用
- **`minecraft:smash_damage_per_fallen_block`(`smash_damage_per_fallen_block`)**：为锤添加每下落方块的伤害。密度(Density)使用
- **`minecraft:knockback`(`knockback`)**：修改击退距离（游戏单位）。击退(Knockback)/冲击(Punch)使用
- **`minecraft:mob_experience`(`mob_experience`)**：修改击杀生物经验（未使用）

耐久相关：
- **`minecraft:item_damage`(`item_damage`)**：修改物品耐久损耗。小于1的值表示不损耗概率。耐久(Unbreaking)使用
- **`minecraft:repair_with_xp`(`repair_with_xp`)**：使物品用经验修复并决定效率。经验修补(Mending)使用

投射物相关：
- **`minecraft:ammo_use`(`ammo_use`)**：修改弓/弩的弹药消耗。整数值（小于1时为0）。无限(Infinity)使用
- **`minecraft:projectile_piercing`(`projectile_piercing`)**：修改投射物穿透实体数。穿透(Piercing)使用
- **`minecraft:projectile_count`(`projectile_count`)**：修改弓发射的投射物数量。多重箭(Multishot)使用
- **`minecraft:projectile_spread`(`projectile_spread`)**：修改投射物最大散布角度（度）。多重箭(Multishot)使用
- **`minecraft:trident_return_acceleration`(`trident_return_acceleration`)**：使三叉戟返回持有者并修改加速度。忠诚(Loyalty)使用

其他：
- **`minecraft:block_experience`(`block_experience`)**：修改破坏方块的XP。精准采集(Silk Touch)使用
- **`minecraft:fishing_time_reduction`(`fishing_time_reduction`)**：减少鱼漂下沉时间（秒）。海之眷顾(Lure)使用
- **`minecraft:fishing_luck_bonus`(`fishing_luck_bonus`)**：修改钓鱼战利品表的[幸运(`luck`)]值。饵钓(Luck of the Sea)使用

#### 定义为`DataComponentType<List<TargetedConditionalEffect<EnchantmentValueEffect>>>`
- **`minecraft:equipment_drops`(`equipment_drops`)**：修改被击杀实体的装备掉落概率。掠夺(Looting)使用

## 基于位置的效果组件(Location Based Effect Components)

*另见Minecraft Wiki上的[基于位置的效果组件]*

基于位置的效果组件实现`EnchantmentLocationBasedEffect`接口，定义需要知道持有者位置的操作。主要通过两个方法运作：`EnchantmentEntityEffect#onChangedBlock`（物品装备或持有者位置变化时调用）和`onDeactivate`（物品卸下时调用）。

以下示例使用`minecraft:attributes`修改持有者体型：

<Tabs>
<TabItem value="attribute.json" label="JSON">

```json5
"minecraft:attributes": [
    {
        // 基于等级的值
        "amount": {
            "type": "minecraft:linear",
            "base": 1,
            "per_level_above_first": 1
        },
        // 修改"minecraft:generic.scale"属性
        "attribute": "minecraft:generic.scale",
        "id": "examplemod:enchantment.size_change",
        "operation": "add_value"
    }
],
```

</TabItem>
<TabItem value="attribute.datagen" label="Datagen">

```java
DataComponentMap.builder().set(
    EnchantmentEffectComponents.ATTRIBUTES,
    List.of(new EnchantmentAttributeEffect(
        ResourceLocation.fromNamespaceAndPath("examplemod", "enchantment.size_change"),
        Attributes.SCALE,
        LevelBasedValue.perLevel(1F, 1F),
        AttributeModifier.Operation.ADD_VALUE
    ))
).build()
```

</TabItem>
</Tabs>

原版提供以下基于位置的效果：
- **`minecraft:all_of`(`all_of`)**：顺序执行实体效果列表
- **`minecraft:apply_mob_effect`(`apply_mob_effect`)**：对目标应用[状态效果(`mob effect`)]
- **`minecraft:attribute`(`attribute`)**：为持有者添加[属性修饰符(`attribute modifier`)]
- **`minecraft:change_item_damage`(`change_item_damage`)**：损耗物品耐久
- **`minecraft:damage_entity`(`damage_entity`)**：对目标实体造成伤害（攻击场景下与攻击伤害叠加）
- **`minecraft:explode`(`explode`)**：召唤爆炸
- **`minecraft:ignite`(`ignite`)**：点燃实体
- **`minecraft:play_sound`(`play_sound`)**：播放指定音效
- **`minecraft:replace_block`(`replace_block`)**：替换偏移位置的方块
- **`minecraft:replace_disk`(`replace_disk`)**：替换圆盘状区域方块
- **`minecraft:run_function`(`run_function`)**：运行指定[数据包函数(`datapack function`)]
- **`minecraft:set_block_properies`(`set_block_properies`)**：修改方块状态属性
- **`minecraft:spawn_particles`(`spawn_particles`)**：生成粒子
- **`minecraft:summon_entity`(`summon_entity`)**：召唤实体

### 原版基于位置的效果组件类型

#### 定义为`DataComponentType<List<ConditionalEffect<EnchantmentLocationBasedEffect>>>`
- **`minecraft:location_changed`(`location_changed`)**：持有者位置变化或装备时触发。冰霜行者(Frost Walker)/灵魂疾行(Soul Speed)使用

#### 定义为`DataComponentType<List<EnchantmentAttributeEffect>>`
- **`minecraft:attributes`(`attributes`)**：为持有者添加属性修饰符（卸下时移除）

## 实体效果组件(Entity Effect Components)

*另见Minecraft Wiki上的[实体效果组件]*

实体效果组件实现`EnchantmentEntityEffect`接口（`EnchantmentLocationBasedEffect`子类）。通过`EnchantmentEntityEffect#apply`方法实现效果（无需等待位置变化）。

所有基于位置的效果组件类型均适用于实体效果组件（除`minecraft:attribute`仅注册为位置效果）。

以下是火焰附加(Fire Aspect)附魔的JSON定义示例：

<Tabs>
<TabItem value="fire.json" label="JSON">

```json5
"minecraft:post_attack": [
    {
        // 效果接收者："victim"(受害者)/"attacker"(攻击者)/"damaging_entity"(伤害实体)
        "affected": "victim",
        "effect": {
            "type": "minecraft:ignite",
            // 点燃持续时间（基于等级）
            "duration": {
                "type": "minecraft:linear",
                "base": 4.0,
                "per_level_above_first": 4.0
            }
        },
        // 附魔持有者
        "enchanted": "attacker",
        // 可选的生效条件
        "requirements": {
            "condition": "minecraft:damage_source_properties",
            "predicate": { "is_direct": true }
        }
    }
]
```

</TabItem>
<TabItem value="fire.datagen" label="Datagen">

```java
DataComponentMap.builder().set(
    EnchantmentEffectComponents.POST_ATTACK,
    List.of(new TargetedConditionalEffect<>(
        EnchantmentTarget.ATTACKER,
        EnchantmentTarget.VICTIM,
        new Ignite(LevelBasedValue.perLevel(4.0F, 4.0F)),
        Optional.of(new DamageSourceCondition(/*...*/))
    )
).build()
```

</TabItem>
</Tabs>

### 原版实体效果组件类型

#### 定义为`DataComponentType<List<ConditionalEffect<EnchantmentEntityEffect>>>`
- **`minecraft:hit_block`(`hit_block`)**：实体击中方块时触发。引雷(Channeling)使用
- **`minecraft:tick`(`tick`)**：每刻触发。灵魂疾行(Soul Speed)使用
- **`minecraft:projectile_spawned`(`projectile_spawned`)**：弓/弩发射投射物后触发。火矢(Flame)使用

#### 定义为`DataComponentType<List<TargetedConditionalEffect<EnchantmentEntityEffect>>>`
- **`minecraft:post_attack`(`post_attack`)**：攻击命中实体后触发。节肢杀手(Bane of Arthropods)/引雷(Channeling)/火焰附加(Fire Aspect)/荆棘(Thorns)/风爆(Wind Burst)使用

详情参见[Minecraft Wiki相关页面]。

## 其他原版附魔组件类型

#### 定义为`DataComponentType<List<ConditionalEffect<DamageImmunity>>>`
- **`minecraft:damage_immunity`(`damage_immunity`)**：提供指定伤害类型免疫。冰霜行者(Frost Walker)使用

#### 定义为`DataComponentType<Unit>`
- **`minecraft:prevent_equipment_drop`(`prevent_equipment_drop`)**：死亡时阻止物品掉落。消失诅咒(Curse of Vanishing)使用
- **`minecraft:prevent_armor_change`(`prevent_armor_change`)**：阻止卸下护甲。绑定诅咒(Curse of Binding)使用

#### 定义为`DataComponentType<List<CrossbowItem.ChargingSounds>>`
- **`minecraft:crossbow_charge_sounds`(`crossbow_charge_sounds`)**：决定弩蓄力音效（每级对应一个条目）

#### 定义为`DataComponentType<List<Holder<SoundEvent>>>`
- **`minecraft:trident_sound`(`trident_sound`)**：决定三叉戟使用音效（每级对应一个条目）

[附魔(`Enchantment`)]: index.md
[数值效果组件]: https://minecraft.wiki/w/Enchantment_definition#Components_with_value_effects
[实体效果组件]: https://minecraft.wiki/w/Enchantment_definition#Components_with_entity_effects
[基于位置的效果组件]: https://minecraft.wiki/w/Enchantment_definition#location_changed
[基于等级的值(`LevelBasedValue`)]: ../loottables/index.md#number-provider
[数据包函数(`datapack function`)]: https://minecraft.wiki/w/Function_(Java_Edition)
[幸运(`luck`)]: https://minecraft.wiki/w/Luck
[状态效果(`mob effect`)]: ../../../items/mobeffects.md
[属性修饰符(`attribute modifier`)]: ../../../entities/attributes.md#attribute-modifiers
[Minecraft Wiki相关页面]: https://minecraft.wiki/w/Enchantment_definition#Components_with_entity_effects
