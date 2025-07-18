---
sidebar_position: 3
---
# **消耗品**(`Consumables`)

**消耗品**(`Consumables`)是可在一段时间内使用并在过程中被"消耗"的[物品(`item`)][item]。Minecraft中任何可食用或饮用的物品都属于消耗品。

## `Consumable`数据组件

任何可消耗物品都有[`DataComponents#CONSUMABLE`组件][datacomponent]。底层记录`Consumable`定义物品如何消耗及消耗后应用的效果。

可通过记录构造函数直接创建`Consumable`，或使用`Consumable#builder`设置字段默认值后`build`：

- `consumeSeconds` - 表示完全消耗物品所需的秒数（`float`），时间结束后调用`Item#finishUsingItem`。默认为1.6秒（32刻）。
- `animation` - 设置使用物品时播放的[`ItemUseAnimation`][animation]。默认为`ItemUseAnimation#EAT`。
- `sound` - 设置消耗时播放的[`SoundEvent`][sound]（必须为`Holder`实例）。默认为`SoundEvents#GENERIC_EAT`。
    - 若原版实例非`Holder<SoundEvent>`，可调用`BuiltInRegistries.SOUND_EVENT.wrapAsHolder(soundEvent)`获取Holder包装版本。
- `soundAfterConsume` - 设置完全消耗后播放的[`SoundEvent`][sound]，委托给[`PlaySoundConsumeEffect`][consumeeffect]。
- `hasConsumeParticles` - 为`true`时每4刻及完全消耗时生成物品[粒子(`particles`)][particles]。默认为`true`。
- `onConsume` - 添加[`ConsumeEffect`][consumeeffect]，在`Item#finishUsingItem`完全消耗物品时应用。

原版在`Consumables`类中提供部分消耗品，如[食物(`food`)][food]的`#defaultFood`和[药水(`potions`)][potions]与牛奶桶的`#defaultDrink`。

通过`Item.Properties#component`添加`Consumable`组件：

```java
// 假设有DeferredRegister.Items ITEMS
public static final DeferredItem<Item> CONSUMABLE = ITEMS.registerSimpleItem(
    "consumable",
    new Item.Properties().component(
        DataComponents.CONSUMABLE,
        Consumable.builder()
            // 消耗时间2秒（40刻）
            .consumeSeconds(2f)
            // 设置消耗时播放的动画
            .animation(ItemUseAnimation.BLOCK)
            // 每刻播放消耗音效
            .sound(SoundEvents.ARMOR_EQUIP_CHAIN)
            // 完全消耗后播放音效
            .soundAfterConsume(SoundEvents.BREEZE_WIND_CHARGE_BURST)
            // 消耗时不显示粒子
            .hasConsumeParticles(false)
            .onConsume(
                // 完全消耗后以30%几率应用效果
                new ApplyStatusEffectsConsumeEffect(new MobEffectInstance(MobEffects.HUNGER, 600, 0), 0.3F)
            )
            // 可添加多个
            .onConsume(
                // 在50格半径内随机传送实体
                new TeleportRandomlyConsumeEffect(100f)
            )
            .build()
    )
);
```

### `ConsumeEffect`

消耗品完全使用后，可能需要触发逻辑（如添加药水效果）。这由`ConsumeEffect`处理，通过`Consumable.Builder#onConsume`添加到`Consumable`。

原版效果列表见`ConsumeEffect`。

每个`ConsumeEffect`有两个方法：`getType`指定注册对象`ConsumeEffect.Type`；`apply`在物品完全消耗时调用（参数：消耗实体所在`Level`、调用的`ItemStack`、消耗的`LivingEntity`）。效果成功应用返回`true`。

可通过实现接口并向`BuiltInRegistries#CONSUME_EFFECT_TYPE`[注册][registering]带关联`MapCodec`和`StreamCodec`的`ConsumeEffect.Type`创建`ConsumeEffect`：

```java
public record UsePortalConsumeEffect(ResourceKey<Level> level)
    implements ConsumeEffect, Portal {

    @Override
    public boolean apply(Level level, ItemStack stack, LivingEntity entity) {
        if (entity.canUsePortal(false)) {
            entity.setAsInsidePortal(this, entity.blockPosition());

            // 成功使用传送门
            return true;
        }

        // 无法使用传送门
        return false;
    }

    @Override
    public ConsumeEffect.Type<? extends ConsumeEffect> getType() {
        // 设置为注册对象
        return USE_PORTAL.get();
    }

    @Override
    @Nullable
    public TeleportTransition getPortalDestination(ServerLevel level, Entity entity, BlockPos pos) {
        // 设置传送位置
    }
}

// 在某个注册类
// 假设有DeferredRegister<ConsumeEffect.Type<?>> CONSUME_EFFECT_TYPES
public static final Supplier<ConsumeEffect.Type<UsePortalConsumeEffect>> USE_PORTAL =
    CONSUME_EFFECT_TYPES.register("use_portal", () -> new ConsumeEffect.Type<>(
        ResourceKey.codec(Registries.DIMENSION).optionalFieldOf("dimension")
            .xmap(UsePortalConsumeEffect::new, UsePortalConsumeEffect::level),
        ResourceKey.streamCodec(Registries.DIMENSION)
            .map(UsePortalConsumeEffect::new, UsePortalConsumeEffect::level)
    ));

// 为添加CONSUMABLE组件的Item.Properties
Consumable.builder()
    .onConsume(
        new UsePortalConsumeEffect(Level.END)
    )
    .build();
```

### `ItemUseAnimation`

`ItemUseAnimation`是功能上仅含ID和名称的枚举，其用途硬编码于第一人称的`ItemHandRenderer#renderArmWithItem`和第三人称的`PlayerRenderer#getArmPose`。因此仅创建新`ItemUseAnimation`会类似`ItemUseAnimation#NONE`。

要应用动画，需为第一人称实现`IClientItemExtensions#applyForgeHandTransform`和/或为第三人称实现`IClientItemExtensions#getArmPose`。

#### 创建`ItemUseAnimation`

首先使用[可扩展枚举(`extensibleenum`)][extensibleenum]系统创建新`ItemUseAnimation`：

```json5
{
    "entries": [
        {
            "enum": "net/minecraft/world/item/ItemUseAnimation",
            "name": "EXAMPLEMOD_ITEM_USE_ANIMATION",
            "constructor": "(ILjava/lang/String;)V",
            "parameters": [
                // ID（始终为-1）
                -1,
                // 唯一标识符名称
                "examplemod:item_use_animation"
            ]
        }
    ]
}
```

然后通过`valueOf`获取枚举常量：

```java
public static final ItemUseAnimation EXAMPLE_ANIMATION = ItemUseAnimation.valueOf("EXAMPLEMOD_ITEM_USE_ANIMATION");
```

接着开始应用变换。需创建新`IClientItemExtensions`，实现所需方法，并在[**模组事件总线**](`modbus`)的`RegisterClientExtensionsEvent`中注册：

```java
public class ConsumableClientItemExtensions implements IClientItemExtensions {
    // 在此实现方法
}

// 在某个事件处理类
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerClientExtensions(RegisterClientExtensionsEvent event) {
    event.registerItem(
        // 物品扩展实例
        new ConsumableClientItemExtensions(),
        // 使用此扩展的物品（可变参数）
        CONSUMABLE
    )
}
```

#### 第一人称(`First Person`)

所有消耗品的第一人称变换通过`IClientItemExtensions#applyForgeHandTransform`实现：

```java
public class ConsumableClientItemExtensions implements IClientItemExtensions {

    // ...

    @Override
    public boolean applyForgeHandTransform(
        PoseStack poseStack, LocalPlayer player, HumanoidArm arm, ItemStack itemInHand,
        float partialTick, float equipProcess, float swingProcess
    ) {
        // 先检查物品是否正使用且为自定义动画
        HumanoidArm usingArm = entity.getUsedItemHand() == InteractionHand.MAIN_HAND
            ? entity.getMainArm()
            : entity.getMainArm().getOpposite();
        if (
            entity.isUsingItem() && entity.getUseItemRemainingTicks() > 0
            && usingArm == arm && itemInHand.getUseAnimation() == EXAMPLE_ANIMATION
        ) {
            // 应用变换到位姿栈（平移、缩放、旋转）
            // ...
            return true;
        }

        // 无操作
        return false;
    }
}
```

#### 第三人称(`Third Person`)

除`EAT`和`DRINK`外都有特殊逻辑的第三人称变换通过`IClientItemExtensions#getArmPose`实现（`HumanoidModel.ArmPose`也可扩展自定义变换）。

因`ArmPose`构造函数需lambda，必须使用`EnumProxy`引用：

```json5
{
    "entries": [
        {
            "name": "EXAMPLEMOD_ITEM_USE_ANIMATION",
            // ...
        },
        {
            "enum": "net/minecraft/client/model/HumanoidModel$ArmPose",
            "name": "EXAMPLEMOD_ARM_POSE",
            "constructor": "(ZLnet/neoforged/neoforge/client/IArmPoseTransformer;)V",
            "parameters": {
                // 指向代理所在类（客户端专属）
                "class": "example/examplemod/client/MyClientEnumParams",
                // 枚举代理字段名
                "field": "CUSTOM_ARM_POSE"
            }
        }
    ]
}
```

```java
// 创建枚举参数
public class MyClientEnumParams {
    public static final EnumProxy<HumanoidModel.ArmPose> CUSTOM_ARM_POSE = new EnumProxy<>(
        HumanoidModel.ArmPose.class,
        // 是否使用双臂
        false,
        // 姿势变换器
        (IArmPoseTransformer) MyClientEnumParams::applyCustomModelPose
    );

    private static void applyCustomModelPose(
        HumanoidModel<?> model, HumanoidRenderState state, HumanoidArm arm
    ) {
        // 在此应用模型变换
        // ...
    }
}

// 在某个客户端专属类
public static final HumanoidModel.ArmPose EXAMPLE_POSE = HumanoidModel.ArmPose.valueOf("EXAMPLEMOD_ARM_POSE");
```

然后通过`IClientItemExtensions#getArmPose`设置手臂姿势：

```java
public class ConsumableClientItemExtensions implements IClientItemExtensions {

    // ...

    @Override
    public HumanoidModel.ArmPose getArmPose(
        LivingEntity entity, InteractionHand hand, ItemStack stack
    ) {
        // 先检查物品是否正使用且为自定义动画
        if (
            entity.isUsingItem() && entity.getUseItemRemainingTicks() > 0
            && entity.getUsedItemHand() == hand
            && itemInHand.getUseAnimation() == EXAMPLE_ANIMATION
        ) {
            // 返回要应用的姿势
            return EXAMPLE_POSE;
        }

        // 否则返回null
        return null;
    }
}
```

### 覆盖实体音效(`Overriding Sounds on Entity`)

有时实体消耗物品时需播放不同音效。此时[`LivingEntity`][livingentity]实例可实现`Consumable.OverrideConsumeSound`，使`getConsumeSound`返回实体要播放的`SoundEvent`。

```java
public class MyEntity extends LivingEntity implements Consumable.OverrideConsumeSound {
    
    // ...

    @Override
    public SoundEvent getConsumeSound(ItemStack stack) {
        // 返回要播放的音效
    }
}
```

## `ConsumableListener`

消耗品和消耗后应用的效果虽有用，但有时效果属性需作为其他[数据组件][datacomponents]外部可用。例如猫和狼也吃[食物(`food`)][food]并查询其营养值，或药水物品为渲染查询颜色。此时数据组件实现`ConsumableListener`提供消耗逻辑。

`ConsumableListener`仅有一个方法：`#onConsume`（参数：当前世界、消耗实体、被消耗物品、物品上的`Consumable`实例）。`onConsume`在物品完全消耗时于`Item#finishUsingItem`中调用。

添加自定义`ConsumableListener`只需[注册新数据组件(`datacompreg`)][datacompreg]并实现`ConsumableListener`。

```java
public record MyConsumableListener() implements ConsumableListener {

    @Override
    public void onConsume(
        Level level, LivingEntity entity, ItemStack stack, Consumable consumable
    ) {
        // 在此执行操作
    }
}
```

### **食物**(`Food`)

食物是饥饿系统的`ConsumableListener`类型。食物物品的所有功能已在`Item`类处理，因此只需向`DataComponents#FOOD`添加`FoodProperties`和消耗品（未指定时使用`Consumables#DEFAULT_FOOD`）。辅助方法`food`接受`FoodProperties`和`Consumable`对象。

`FoodProperties`可通过记录构造函数直接创建，或使用`new FoodProperties.Builder()`后`build`：

- `nutrition` - 恢复的饥饿值（半饥饿点计数，如牛排恢复8点）。
- `saturationModifier` - 计算[饱和度(`hunger`)][hunger]恢复的修饰符（公式：`min(2 * nutrition * saturationModifier, playerNutrition)`）。0.5时有效饱和度等于营养值。
- `alwaysEdible` - 是否始终可食用（即使饥饿条满）。默认为`false`，金苹果等提供额外效果的物品为`true`。

```java
// 假设有DeferredRegister.Items ITEMS
public static final DeferredItem<Item> FOOD = ITEMS.registerSimpleItem(
    "food",
    new Item.Properties().food(
        new FoodProperties.Builder()
            // 恢复1.5颗心
            .nutrition(3)
            // 胡萝卜0.3
            // 生鳕鱼0.1
            // 熟鸡肉0.6
            // 熟牛肉0.8
            // 金苹果1.2
            .saturationModifier(0.3f)
            // 设置后即使饥饿条满也可食用
            .alwaysEdible()
    )
);
```

示例或Minecraft使用的各种值见`Foods`类。

获取物品的`FoodProperties`可调用`ItemStack.get(DataComponents.FOOD)`（可能返回null，因非所有物品可食用）。判断物品是否可食用需检查返回值非空。

### **药水内容物**(`Potion Contents`)

通过`PotionContents`的[药水][potions]内容是另一种在消耗时应用效果的`ConsumableListener`。包含要应用的可选药水、药水颜色的可选色调、与药水一起应用的自定义[`MobEffectInstance`](`mobeffectinstance`)列表，以及获取堆叠名称时使用的可选翻译键。若非`PotionItem`子类，模组开发者需重写`Item#getName`。

[animation]: #itemuseanimation
[consumeeffect]: #consumeeffect
[datacomponent]: datacomponents.md
[datacompreg]: datacomponents.md#creating-custom-data-components
[extensibleenum]: ../advanced/extensibleenums.md
[food]: #food
[hunger]: https://minecraft.wiki/w/Hunger#Mechanics
[item]: index.md
[livingentity]: ../entities/livingentity.md
[modbus]: ../concepts/events.md#event-buses
[mobeffectinstance]: mobeffects.md#mobeffectinstances
[particles]: ../resources/client/particles.md
[potions]: mobeffects.md#potions
[sound]: ../resources/client/sounds.md#creating-soundevents
[registering]: ../concepts/registries.md#methods-for-registering