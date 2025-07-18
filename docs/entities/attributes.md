---
sidebar_position: 4
---
# **属性**(`Attributes`)

**属性**(`Attributes`)是[活体实体][livingentity]的特殊字段，用于决定基础属性如最大生命值、速度或护甲值。所有属性都以双精度值存储并自动同步。原版提供了广泛的默认属性，你也可以添加自己的属性。

由于历史实现原因，并非所有属性都适用于所有实体。例如，飞行速度被恶魂忽略，跳跃强度只影响马匹而不影响玩家。

## **内置属性**(`Built-In Attributes`)

### Minecraft

以下属性位于 `minecraft` 命名空间，其代码值可在 `Attributes` 类中找到。

| 名称                             | 代码值                          | 范围          | 默认值 | 用途                                                                                                                                                                 |
|----------------------------------|----------------------------------|----------------|---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `armor`                          | `ARMOR`                          | `[0,30]`       | 0             | 实体的护甲值。值为1表示快捷栏上方显示半个胸甲图标。                                                                                                               |
| `armor_toughness`                | `ARMOR_TOUGHNESS`                | `[0,20]`       | 0             | 实体的护甲韧性值。更多信息请参见 [Minecraft Wiki][wiki] 上的[护甲韧性][toughness]。                                                                               |
| `attack_damage`                  | `ATTACK_DAMAGE`                  | `[0,2048]`     | 2             | 实体造成的基础攻击伤害，不包含任何武器或类似物品。                                                                                                                |
| `attack_knockback`               | `ATTACK_KNOCKBACK`               | `[0,5]`        | 0             | 实体造成的额外击退效果。击退还有基础强度，不由该属性表示。                                                                                                      |
| `attack_speed`                   | `ATTACK_SPEED`                   | `[0,1024]`     | 4             | 实体的攻击冷却时间。数值越高冷却越长，设为0可有效恢复1.9版本前的战斗机制。                                                                                     |
| `block_break_speed`              | `BLOCK_BREAK_SPEED`              | `[0,1024]`     | 1             | 实体挖掘方块的速度，作为乘数修饰符。更多信息请参见[挖掘速度][miningspeed]。                                                                                     |
| `block_interaction_range`        | `BLOCK_INTERACTION_RANGE`        | `[0,64]`       | 4.5           | 实体可与方块交互的距离，以方块为单位。                                                                                                                          |
| `burning_time`                   | `BURNING_TIME`                   | `[0,1024]`     | 1             | 实体被点燃时燃烧时间的乘数。                                                                                                                                  |
| `explosion_knockback_resistance` | `EXPLOSION_KNOCKBACK_RESISTANCE` | `[0,1]`        | 0             | 实体的爆炸击退抗性。这是一个百分比值，0表示无抗性，0.5表示半抗性，1表示完全抗性。                                                                              |
| `entity_interaction_range`       | `ENTITY_INTERACTION_RANGE`       | `[0,64]`       | 3             | 实体可与其他实体交互的距离，以方块为单位。                                                                                                                      |
| `fall_damage_multiplier`         | `FALL_DAMAGE_MULTIPLIER`         | `[0,100]`      | 1             | 实体承受坠落伤害的乘数。                                                                                                                                      |
| `flying_speed`                   | `FLYING_SPEED`                   | `[0,1024]`     | 0.4           | 飞行速度的乘数。并非所有飞行实体都实际使用此属性，例如恶魂会忽略它。                                                                                          |
| `follow_range`                   | `FOLLOW_RANGE`                   | `[0,2048]`     | 32            | 实体将追踪/跟随玩家的距离，以方块为单位。                                                                                                                       |
| `gravity`                        | `GRAVITY`                        | `[1,1]`        | 0.08          | 实体所受重力，以每刻平方方块数表示。                                                                                                                            |
| `jump_strength`                  | `JUMP_STRENGTH`                  | `[0,32]`       | 0.42          | 实体的跳跃强度。值越高跳跃越高。                                                                                                                              |
| `knockback_resistance`           | `KNOCKBACK_RESISTANCE`           | `[0,1]`        | 0             | 实体的击退抗性。这是一个百分比值，0表示无抗性，0.5表示半抗性，1表示完全抗性。                                                                                  |
| `luck`                           | `LUCK`                           | `[-1024,1024]` | 0             | 实体的幸运值。在生成[战利品表][loottables]时使用，以提供额外生成机会或修改生成物品品质。                                                                       |
| `max_absorption`                 | `MAX_ABSORPTION`                 | `[0,2048]`     | 0             | 实体的最大吸收值（黄心）。值为1表示半颗心。                                                                                                                    |
| `max_health`                     | `MAX_HEALTH`                     | `[1,1024]`     | 20            | 实体的最大生命值。值为1表示半颗心。                                                                                                                            |
| `mining_efficiency`              | `MINING_EFFICIENCY`              | `[0,1024]`     | 0             | 实体挖掘方块的速度，作为加法修饰符，仅当使用正确工具时生效。更多信息请参见[挖掘速度][miningspeed]。                                                             |
| `movement_efficiency`            | `MOVEMENT_EFFICIENCY`            | `[0,1]`        | 0             | 实体在灵魂沙等减速方块上行走时应用的线性插值移动速度加成。                                                                                                      |
| `movement_speed`                 | `MOVEMENT_SPEED`                 | `[0,1024]`     | 0.7           | 实体的移动速度。值越高速度越快。                                                                                                                              |
| `oxygen_bonus`                   | `OXYGEN_BONUS`                   | `[0,1024]`     | 0             | 实体的氧气加成。值越高，实体开始溺水所需时间越长。                                                                                                              |
| `safe_fall_distance`             | `SAFE_FALL_DISTANCE`             | `[-1024,1024]` | 3             | 实体安全的坠落距离，即在此距离内不会受到坠落伤害。                                                                                                              |
| `scale`                          | `SCALE`                          | `[0.0625,16]`  | 1             | 实体渲染的缩放比例。                                                                                                                                          |
| `sneaking_speed`                 | `SNEAKING_SPEED`                 | `[0,1]`        | 0.3           | 实体潜行时应用的移动速度乘数。                                                                                                                                |
| `spawn_reinforcements`           | `SPAWN_REINFORCEMENTS_CHANCE`    | `[0,1]`        | 0             | 僵尸生成其他僵尸的几率。仅在困难难度下相关，因为普通或更低难度下不会发生僵尸增援。                                                                              |
| `step_height`                    | `STEP_HEIGHT`                    | `[0,10]`       | 0.6           | 实体的步高，以方块为单位。如果为1，玩家可像走上半砖一样走上1方块高的台阶。                                                                                    |
| `submerged_mining_speed`         | `SUBMERGED_MINING_SPEED`         | `[0,20]`       | 0.2           | 实体挖掘方块的速度，作为乘数修饰符，仅当实体在水下时生效。更多信息请参见[挖掘速度][miningspeed]。                                                              |
| `sweeping_damage_ratio`          | `SWEEPING_DAMAGE_RATIO`          | `[0,1]`        | 0             | 横扫攻击造成的伤害量，以主攻击的百分比表示。这是一个百分比值，0表示无伤害，0.5表示半伤害，1表示全伤害。                                                         |
| `tempt_range`                    | `TEMPT_RANGE`                    | `[0,2048]`     | 10            | 实体可被物品吸引的范围。主要用于被动生物，如牛或猪。                                                                                                            |
| `water_movement_efficiency`      | `WATER_MOVEMENT_EFFICIENCY`      | `[0,1]`        | 0             | 实体在水下时应用的移动速度乘数。                                                                                                                              |
| `waypoint_transmit_range`        | `WAYPOINT_TRANSMIT_RANGE`        | `[0,60000000]` | 0             | 实体可将其位置传输到路径点追踪器的范围。                                                                                                                        |
| `waypoint_receive_range`         | `WAYPOINT_RECEIVE_RANGE`         | `[0,60000000]` | 0             | 实体可接收其他发射器信号的范围。                                                                                                                                |

:::warning
Mojang 设置的某些属性上限相对随意。护甲上限为30尤其明显。NeoForge 不会修改这些上限，但有模组可以更改它们。
:::

### NeoForge

以下属性位于 `neoforge` 命名空间，其代码值可在 `NeoForgeMod` 类中找到。

| 名称               | 代码值            | 范围      | 默认值 | 用途                                                                                                                                                |
|--------------------|--------------------|------------|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `creative_flight`  | `CREATIVE_FLIGHT`  | `[0,1]`    | 0             | 决定实体的创造模式飞行是否启用 (\> 0) 或禁用 (\<\= 0)。                                                                                                |
| `nametag_distance` | `NAMETAG_DISTANCE` | `[0,32]`   | 32            | 实体的名称标签可见距离，以方块为单位。                                                                                                                |
| `swim_speed`       | `SWIM_SPEED`       | `[0,1024]` | 1             | 实体在水下时应用的移动速度乘数。此属性独立于 `minecraft:water_movement_efficiency` 应用。                                                              |

## **默认属性**(`Default Attributes`)

创建 `LivingEntity` 时，需要为其注册一组默认属性。当实体[生成][spawning]时，其默认属性会被设置。默认属性在 [`EntityAttributeCreationEvent`][event] 中注册如下：

```java
@SubscribeEvent // 在模组事件总线上
public static void createDefaultAttributes(EntityAttributeCreationEvent event) {
    event.put(
        // 你的实体类型
        MY_ENTITY.get(),
        // AttributeSupplier。通常通过调用 LivingEntity#createLivingAttributes 创建，
        // 在其上设置你的值，然后调用 #build。如果需要，你也可以从头创建 AttributeSupplier，
        // 参见 LivingEntity#createLivingAttributes 的源代码作为示例。
        LivingEntity.createLivingAttributes()
            // 添加一个属性及其默认值
            .add(Attributes.MAX_HEALTH)
            // 添加一个非默认值的属性
            .add(Attributes.MAX_HEALTH, 50)
            // 构建 AttributeSupplier
            .build()
    );
}
```

:::tip
某些类有 `LivingEntity#createLivingAttributes` 的专用版本。例如，`Monster` 类有一个名为 `Monster#createMonsterAttributes` 的方法可替代使用。
:::

在某些情况下，例如[创建自定义属性][custom]时，需要向现有实体的 `AttributeSupplier` 添加属性。这通过 `EntityAttributeModificationEvent` 实现如下：

```java
@SubscribeEvent // 在模组事件总线上
public static void modifyDefaultAttributes(EntityAttributeModificationEvent event) {
    event.add(
        // 要添加属性的 EntityType
        EntityType.VILLAGER,
        // 要添加到 EntityType 的 Holder<Attribute>。也可以是自定义属性
        Attributes.ARMOR,
        // 要添加的属性值
        // 可省略，若省略则使用属性的默认值
        10.0
    );
    // 我们也可以检查给定 EntityType 是否已有某属性
    // 在此示例中，如果村民还没有护甲属性，我们添加它
    if (!event.has(EntityType.VILLAGER, Attributes.ARMOR)) {
        event.add(...);
    }
}
```

请注意，与其他注册表不同，自定义属性的存在不会阻止原版客户端连接到 NeoForge 服务器。如果原版客户端连接，它只会接收 `minecraft` 命名空间中的属性。

## **查询属性**(`Querying Attributes`)

属性值存储在实体的 `AttributeMap` 中，本质上是 `Map<Attribute, AttributeInstance>`。属性实例类似于物品堆叠与物品的关系，即属性是注册的单例，而属性实例是绑定到具体实体的具体属性对象。

可通过调用 `LivingEntity#getAttributes` 获取实体的 `AttributeMap`。然后可以这样查询映射：

```java
// 获取属性映射
AttributeMap attributes = livingEntity.getAttributes();
// 获取属性实例。如果实体没有该属性，可能为 null
AttributeInstance instance = attributes.getInstance(Attributes.ARMOR);
// 获取属性的值。如果需要，将回退到实体的默认值
double value = attributes.getValue(Attributes.ARMOR);
// 当然，我们也可以检查属性是否存在
if (attributes.hasAttribute(Attributes.ARMOR)) { ... }

// 或者，LivingEntity 也提供快捷方式：
AttributeInstance instance = livingEntity.getAttribute(Attributes.ARMOR);
double value = livingEntity.getAttributeValue(Attributes.ARMOR);
```

:::info
处理属性时，你几乎总是使用 `Holder<Attribute>` 而不是 `Attribute`。这也是为什么对于自定义属性（见下文），我们显式存储 `Holder<Attribute>`。
:::

## **属性修饰符**(`Attribute Modifiers`)

与查询相比，更改属性值并不简单。这主要是因为可能同时需要对属性进行多次更改。

考虑这种情况：你是一个玩家，攻击伤害属性为1。你手持一把钻石剑，提供6点额外攻击伤害，因此总攻击伤害为7。然后你喝下一瓶力量药水，添加伤害乘数。接着你还装备了某种饰品，又添加了另一个乘数。

为避免计算错误并更好地传达属性值的修改方式，Minecraft 引入了属性修饰符系统。在此系统中，每个属性都有一个**基础值**，通常来自我们之前讨论的默认属性。然后我们可以添加任意数量的**属性修饰符**，这些修饰符可以单独移除，无需担心正确应用操作的问题。

首先，让我们创建一个属性修饰符：

```java
// 修饰符的名称。稍后用于从属性映射查询修饰符，
// 因此必须（语义上）唯一
ResourceLocation id = ResourceLocation.fromNamespaceAndPath("yourmodid", "my_modifier");
// 修饰符本身
AttributeModifier modifier = new AttributeModifier(
    // 我们之前定义的名称
    id,
    // 修改属性值的量
    2.0,
    // 应用修饰符的操作。可能值有：
    // - AttributeModifier.Operation.ADD_VALUE：将值加到总属性值上
    // - AttributeModifier.Operation.ADD_MULTIPLIED_BASE：将值与属性基础值相乘
    //   并加到总属性值上
    // - AttributeModifier.Operation.ADD_MULTIPLIED_TOTAL：将值与总属性值相乘，
    //   即已执行所有先前修改的属性基础值，
    //   并加到总属性值上
    AttributeModifier.Operation.ADD_VALUE
);
```

现在，应用修饰符有两种选择：添加为瞬态修饰符或永久修饰符。永久修饰符保存到磁盘，而瞬态修饰符不会。永久修饰符的用例包括永久状态加成（如某种护甲或生命值技能），而瞬态修饰符主要用于[装备][equipment]、[状态效果][mobeffect]和其他依赖于玩家当前状态的修饰符。

```java
AttributeMap attributes = livingEntity.getAttributes();
// 添加瞬态修饰符。如果已存在相同id的修饰符，将抛出异常
attributes.getInstance(Attributes.ARMOR).addTransientModifier(modifier);
// 添加瞬态修饰符。如果已存在相同id的修饰符，先移除再添加
attributes.getInstance(Attributes.ARMOR).addOrUpdateTransientModifier(modifier);
// 添加永久修饰符。如果已存在相同id的修饰符，将抛出异常
attributes.getInstance(Attributes.ARMOR).addPermanentModifier(modifier);
// 添加永久修饰符。如果已存在相同id的修饰符，先移除再添加
attributes.getInstance(Attributes.ARMOR).addOrReplacePermanentModifier(modifier);
```

这些修饰符也可以被移除：

```java
// 通过修饰符对象移除
attributes.getInstance(Attributes.ARMOR).removeModifier(modifier);
// 通过修饰符id移除
attributes.getInstance(Attributes.ARMOR).removeModifier(id);
// 移除属性的所有修饰符
attributes.getInstance(Attributes.ARMOR).removeModifiers();
```

最后，我们还可以查询属性映射是否包含特定ID的修饰符，以及分别查询基础值和修饰符值：

```java
// 检查修饰符是否存在
if (attributes.getInstance(Attributes.ARMOR).hasModifier(id)) { ... }
// 获取护甲属性的基础值
double baseValue = attributes.getBaseValue(Attributes.ARMOR);
// 获取特定修饰符的值
double modifierValue = attributes.getModifierValue(Attributes.ARMOR, id);
```

## **自定义属性**(`Custom Attributes`)

如果需要，你也可以添加自定义属性。与其他系统类似，属性是一个[注册表][registry]，你可以注册自己的对象。首先创建一个 `DeferredRegister<Attribute>`：

```java
public static final DeferredRegister<Attribute> ATTRIBUTES = DeferredRegister.create(
    BuiltInRegistries.ATTRIBUTE, "yourmodid");
```

对于属性本身，有三种可选类：

- `RangedAttribute`：大多数属性使用，此类定义属性的上下界及默认值。
- `PercentageAttribute`：类似 `RangedAttribute`，但以百分比而非浮点值显示。NeoForge 添加。
- `BooleanAttribute`：只有语义真 (\> 0) 和假 (\<\= 0) 的属性。内部仍使用双精度。NeoForge 添加。

以 `RangedAttribute` 为例（其他两类类似），注册属性如下：

```java
public static final Holder<Attribute> MY_ATTRIBUTE = ATTRIBUTES.register("my_attribute", () -> new RangedAttribute(
    // 使用的翻译键
    "attributes.yourmodid.my_attribute",
    // 默认值
    0,
    // 最小值和最大值
    -10000,
    10000
));
```

就这样！只需记得在模组总线上注册你的 `DeferredRegister` 即可。

:::info
我们在此使用 `Holder<Attribute>` 而非像其他注册对象那样的 `Supplier<RangedAttribute>`，因为这样处理实体更方便（大多数实体方法期望 `Holder<Attribute>`）。

如果因某些原因需要 `Supplier<RangedAttribute>`（或任何其他 `Attribute` 子类的提供者），应使用 `DeferredHolder<Attribute, RangedAttribute>` 作为类型。

相同规则也适用于任何其他 `Attribute` 子类，即我们通常使用 `Holder<Attribute>` 而非 `Supplier<PercentageAttribute>` 或 `Supplier<BooleanAttribute>`。
:::

[custom]: #custom-attributes
[equipment]: ../blockentities/container.md#containers-on-entitys
[event]: ../concepts/events.md
[livingentity]: livingentity.md
[loottables]: ../resources/server/loottables/index.md
[miningspeed]: ../blocks/index.md#mining-speed
[mobeffect]: ../items/mobeffects.md
[registry]: ../concepts/registries.md
[spawning]: index.md#spawning-entities
[toughness]: https://minecraft.wiki/w/Armor#Armor_toughness
[wiki]: https://minecraft.wiki