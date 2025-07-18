---
sidebar_position: 1
---
# 交互机制(`Interactions`)

本文旨在解释玩家左键、右键或中键点击物体的复杂过程，并阐明在何处使用何种结果及其原因。

## **命中结果**(`HitResult`s)

为确定玩家当前注视的目标，Minecraft 使用 `HitResult`。`HitResult` 与其他游戏引擎中的射线检测结果类似，最重要的是包含 `#getLocation` 方法。

命中结果有三种类型，由 `HitResult.Type` 枚举表示：`BLOCK`（方块）、`ENTITY`（实体）或 `MISS`（未命中）。类型为 `BLOCK` 的 `HitResult` 可转换为 `BlockHitResult`，而类型为 `ENTITY` 的可转换为 `EntityHitResult`；两者均提供关于被命中[方块](block)或[实体](entity)的额外信息。若类型为 `MISS`，则表示未命中任何方块或实体，且不应强制转换为任何子类。

在[物理客户端](physicalside)的每一帧中，`Minecraft` 类会更新当前注视的 `HitResult` 并将其存储在 `hitResult` 字段中。可通过 `Minecraft.getInstance().hitResult` 访问此字段。

## 左键点击物品

- 检查主手[`物品堆栈`](itemstack)所需的所有[功能标志](featureflag)是否启用。若检查失败，流程终止。
- 触发 `InputEvent.InteractionKeyMappingTriggered` 事件（左键 + 主手）。若[事件](event)被[取消](cancel)，流程终止。
- 根据注视目标（使用 `Minecraft` 中的 [`HitResult`](hitresult)）执行不同操作：
    - **注视有效范围内的[实体](entity)**：
        - 触发 `AttackEntityEvent`。若事件被取消，流程终止。
        - 调用 `IItemExtension#onLeftClickEntity`。若返回 true，流程终止。
        - 对目标调用 `Entity#isAttackable`。若返回 false，流程终止。
        - 对目标调用 `Entity#skipAttackInteraction`。若返回 true，流程终止。
        - 若目标属于 `minecraft:redirectable_projectile` 标签（默认为火球和风弹）且是 `Projectile` 实例，则目标被弹开，流程终止。
        - 分别计算实体基础伤害（`minecraft:generic.attack_damage` [属性](attribute)值）和附魔加成伤害（两值均为浮点数）。若两者均为 0，流程终止。
            - 注意：此步骤**不包含**主手物品的[属性修饰符](attributemodifier)，这些将在检查后添加。
        - 将主手物品的 `minecraft:generic.attack_damage` 属性修饰符加入基础伤害。
        - 触发 `CriticalHitEvent`。若事件的 `#isCriticalHit` 返回 true，则基础伤害乘以事件的 `#getDamageMultiplier` 返回值（默认满足[特定条件](critical)时为 1.5，否则为 1.0，但事件可能修改此值）。
        - 将附魔加成伤害加入基础伤害，得到最终伤害值。
        - 触发 `SweepAttackEvent`。若事件的 `isSweeping` 返回 true，则玩家执行横扫攻击。默认条件为：攻击冷却 > 90%、非暴击、玩家在地面且移动速度不超过 `minecraft:generic.movement_speed` 属性值。
        - 调用 [`Entity#hurtOrSimulate`](hurt)。若返回 false，流程终止。
        - 若目标是 `LivingEntity` 实例，攻击强度 > 90%，玩家正在疾跑，且附魔修饰后的 `minecraft:attack_knockback` 属性值 > 0，则调用 `LivingEntity#knockback`。
            - 此方法内部触发 `LivingKnockBackEvent`。
        - 根据 `SweepAttackEvent#isSweeping` 对附近的 `LivingEntity` 执行横扫攻击。
            - 此方法内部若实体在玩家范围内且 `Entity#hurtServer` 返回 true，则再次调用 `LivingEntity#knockback`，从而二次触发 `LivingKnockBackEvent`。
        - 调用 `Item#hurtEnemy`（用于攻击后效果，如锤子击退玩家）。
        - 调用 `Item#postHurtEnemy`（耐久度损耗在此应用）。
    - **注视有效范围内的[方块](block)**：
        - 启动[方块破坏子流程](blockbreak)。
    - **其他情况**：
        - 触发 `PlayerInteractEvent.LeftClickEmpty`。

## 右键点击物品

在右键流程中，多个方法会返回两种结果类型之一（见下文）。若返回明确成功或明确失败，多数方法会终止流程。为便于阅读，下文将"明确成功或明确失败"统称为"明确结果"。

- 触发 `InputEvent.InteractionKeyMappingTriggered` 事件（右键 + 主手）。若[事件](event)被[取消](cancel)，流程终止。
- 检查若干条件（如非旁观者模式、主手[`物品堆栈`](itemstack)的[功能标志](featureflag)全启用）。任一条件失败则流程终止。
- 根据注视目标（使用 `Minecraft` 中的 [`HitResult`](hitresult)）执行不同操作：
    - **注视有效范围内且未超出世界边界的[实体](entity)**：
        - 触发 `PlayerInteractEvent.EntityInteractSpecific`。若事件被取消，流程终止。
        - 调用**目标实体**的 `Entity#interactAt`。若返回明确结果，流程终止。
            - 为自定义实体添加行为：重写此方法；为原版实体添加行为：使用事件。
        - 若实体开启交互界面（如村民交易 GUI/运输矿车 GUI），流程终止。
        - 触发 `PlayerInteractEvent.EntityInteract`。若事件被取消，流程终止。
        - 调用**目标实体**的 `Entity#interact`。若返回明确结果，流程终止。
            - 行为添加规则同上。
            - 对于[`生物`](livingentity)，`Entity#interact` 重写处理拴绳绑定和刷怪蛋生成幼体等逻辑，并转交 `Mob#mobInteract` 处理生物特定行为。结果规则同上。
        - 若目标为 `LivingEntity`，则调用主手 `ItemStack` 的 `Item#interactLivingEntity`。若返回明确结果，流程终止。
    - **注视有效范围内且未超出世界边界的[方块](block)**：
        - 触发 `PlayerInteractEvent.RightClickBlock`。若事件被取消，流程终止（可在此事件中单独禁止方块或物品使用）。
        - 调用 `IItemExtension#onItemUseFirst`。若返回明确结果，流程终止。
        - 若玩家未潜行且事件未禁止方块使用：
            - 触发 `UseItemOnBlockEvent`。若事件被取消，使用取消结果；否则调用 `BlockBehaviour#useItemOn`。若返回明确结果，流程终止。
        - 若 `InteractionResult` 为 `TRY_WITH_EMPTY_HAND` 且执行手为主手，则调用 `BlockBehaviour#useWithoutItem`。若返回明确结果，流程终止。
        - 若事件未禁止物品使用，调用 `Item#useOn`。若返回明确结果，流程终止。
- 调用 `Item#use`。若返回明确结果，流程终止。
- 以上流程使用副手**重复执行一次**。

### **交互结果**(`InteractionResult`)

`InteractionResult` 是密封接口，表示物品或空手与对象（如实体、方块等）的交互结果。该接口分为四种记录类型，含六种默认状态。

首先是 `InteractionResult.Success`，表示操作成功并终止流程。成功状态有两个参数：
1. `SwingSource`：指示实体应在哪个[逻辑端](side)挥动手臂
2. `InteractionResult.ItemContext`：记录交互是否由手持物品引发，以及使用后物品的转换结果

挥臂源由默认状态决定：
- `InteractionResult#SUCCESS`：客户端挥臂
- `InteractionResult#SUCCESS_SERVER`：服务端挥臂
- `InteractionResult#CONSUME`：不挥臂

物品上下文通过以下方式设置：
- `Success#heldItemTransformedTo`：当 `ItemStack` 变化时
- `withoutItem`：当手持物品未参与交互
默认状态表示存在物品交互但无转换。

```java
// 返回交互结果的方法示例

// 手中物品将转换为苹果
return InteractionResult.SUCCESS.heldItemTransformedTo(new ItemStack(Items.APPLE));
```

:::note
同一方法中通常**不应混用** `SUCCESS` 和 `SUCCESS_SERVER`。若客户端有足够信息决定何时挥臂，应始终用 `SUCCESS`；若需依赖服务端独有信息，则用 `SUCCESS_SERVER`。
:::

其次是 `InteractionResult.Fail`，由 `InteractionResult#FAIL` 实现，表示操作失败并终止流程。虽可随处使用，但在 `Item#useOn` 和 `Item#use` 外需谨慎。多数情况下 `InteractionResult#PASS` 更合适。

最后是 `InteractionResult.Pass` 和 `InteractionResult.TryWithEmptyHandInteraction`，分别由 `InteractionResult#PASS` 和 `InteractionResult#TRY_WITH_EMPTY_HAND` 实现。这些记录表示操作既非成功也非失败，流程应继续。除 `BlockBehaviour#useItemOn` 返回 `TRY_WITH_EMPTY_HAND` 外，其他方法默认返回 `PASS`。具体来说，若 `BlockBehaviour#useItemOn` 返回值非 `TRY_WITH_EMPTY_HAND`，则无论主手是否持物，`BlockBehaviour#useWithoutItem` 都不会被调用。

部分方法有特殊行为或要求，详见下文。

#### `Item#useOn`

若需操作成功但**不触发挥臂**或**不统计** `ITEM_USED` 点数，使用 `InteractionResult#CONSUME` 并调用 `#withoutItem`。

```java
// 在 Item#useOn 中
return InteractionResult.CONSUME.withoutItem();
```

#### `Item#use`

这是唯一使用 `Success` 变体（`SUCCESS`, `SUCCESS_SERVER`, `CONSUME`）中转换后 `ItemStack` 的场景。通过 `Success#heldItemTransformedTo` 设置的 `ItemStack` 会替换原始物品堆栈（若已变更）。

`Item#use` 的默认实现：
- 若物品可食用（含 `DataComponents#CONSUMABLE`）且玩家可进食（饥饿或物品始终可食），返回 `InteractionResult#CONSUME`
- 若物品可食用但玩家不可进食，返回 `InteractionResult#FAIL`
- 若物品可装备（含 `DataComponents#EQUIPPABLE`）：
    - 替换手持物品后返回 `InteractionResult#SUCCESS`（通过 `heldItemTransformedTo`）
    - 若盔甲附魔含 `EnchantmentEffectComponents#PREVENT_ARMOR_CHANGE`，返回 `InteractionResult#FAIL`
- 其他情况返回 `InteractionResult#PASS`

**注意**：主手操作返回 `InteractionResult#FAIL` 会阻止副手流程。通常应返回 `InteractionResult#PASS` 以允许副手执行。

## 中键点击

- 若 `Minecraft.getInstance().hitResult` 中的 [`HitResult`](hitresult) 为 null 或 `MISS` 类型，流程终止。
- 触发 `InputEvent.InteractionKeyMappingTriggered` 事件（中键 + 主手）。若[事件](event)被[取消](cancel)，流程终止。
- 根据注视目标（使用 `Minecraft.getInstance().hitResult`）执行不同操作：
    - **注视有效范围内的[实体](entity)**：
        - 若 `Entity#isPickable` 返回 false，流程终止。
        - 若 `Player#canInteractWithEntity` 返回 false，流程终止。
        - 调用 `Entity#getPickResult`。若存在匹配快捷栏槽位，则激活该槽位；若玩家在创造模式，则将返回的 `ItemStack` 加入背包。
            - 默认调用 `Entity#getPickResult`，模组开发者可重写此方法。
    - **注视有效范围内的[方块](block)**：
        - 调用 `IBlockExtension#getCloneItemStack`（默认委托给 `BlockBehaviour#getCloneItemStack`）作为"选中"的 `ItemStack`。
            - 默认返回方块的 `Item` 形式。
        - **若按住 Control 键、玩家在创造模式且目标方块含 [`方块实体`](blockentity)**：
            - 通过 `BlockEntity#saveCustomOnly` 获取方块实体数据
                - 后续调用 `BlockEntity#removeComponentsFromTag` 进行清理。
            - 通过 `DataComponents#BLOCK_ENTITY_DATA` 将方块实体数据加入"选中"的 `ItemStack`。
        - 若存在匹配快捷栏槽位，则激活该槽位；若玩家在创造模式，则将"选中"的 `ItemStack` 加入背包。

[attribute]: ../entities/attributes.md
[attributemodifier]: ../entities/attributes.md#attribute-modifiers
[block]: ../blocks/index.md
[blockbreak]: ../blocks/index.md#breaking-a-block
[blockentity]: ../blockentities/index.md
[cancel]: ../concepts/events.md#cancellable-events
[critical]: https://minecraft.wiki/w/Damage#Critical_hit
[effect]: mobeffects.md
[entity]: ../entities/index.md
[event]: ../concepts/events.md
[featureflag]: ../advanced/featureflags.md
[hitresult]: #hitresults
[hurt]: ../entities/index.md#damaging-entities
[itemstack]: index.md#itemstacks
[itemuseon]: #itemuseon
[livingentity]: ../entities/livingentity.md
[physicalside]: ../concepts/sides.md#the-physical-side
[side]: ../concepts/sides.md#the-logical-side