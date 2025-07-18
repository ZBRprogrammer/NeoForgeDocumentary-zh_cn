---
sidebar_position: 6
---
# 生物效果与药水(`Mob Effects & Potions`)

状态效果（有时称为药水效果，代码中称为 `MobEffect`s）是每刻影响[`生物实体`][livingentity]的机制。本文阐述其用法、效果与药水的区别，以及如何添加自定义效果。

## 术语

- **`MobEffect`**：每刻影响实体的效果。如同[方块][block]或[物品][item]，`MobEffect` 是注册对象，需[注册][registration]且为单例。
    - **瞬时生物效果**(`instant mob effect`)：特指设计为仅应用一刻的效果（原版含瞬间治疗与瞬间伤害）。
- **`MobEffectInstance`**：`MobEffect` 的实例，含持续时间、强度等级等属性（见下文）。其关系类比于 [`ItemStack`][itemstack] 与 `Item`。
- **`Potion`**：一组 `MobEffectInstance` 的集合。原版主要用于四种药水物品，但可应用于任意物品，具体效果取决于物品实现。
- **药水物品**(`potion item`)：特指可关联药水的物品（非正式术语）。原版含四种：普通药水、喷溅药水、滞留药水与药箭，模组可扩展。

## **`MobEffect`s**

创建自定义 `MobEffect` 需继承 `MobEffect` 类：

```java
public class MyMobEffect extends MobEffect {
    public MyMobEffect(MobEffectCategory category, int color) {
        super(category, color);
    }
    
    @Override
    public boolean applyEffectTick(ServerLevel level, LivingEntity entity, int amplifier) {
        // 在此实现效果逻辑

        // 当 shouldApplyEffectTickThisTick 返回 true 而此方法返回 false 时，效果将被立即移除
        return true;
    }
    
    // 判断是否在当前刻应用效果（如生命恢复效果每隔X刻应用一次）
    @Override
    public boolean shouldApplyEffectTickThisTick(int tickCount, int amplifier) {
        return tickCount % 2 == 0; // 替换为自定义条件
    }
    
    // 首次添加效果至实体时调用（实体无此效果时触发）
    @Override
    public void onEffectAdded(LivingEntity entity, int amplifier) {
        super.onEffectAdded(entity, amplifier);
    }

    // 每次添加效果至实体时调用
    @Override
    public void onEffectStarted(LivingEntity entity, int amplifier) {
    }
}
```

与其他注册对象相同，`MobEffect` 需[注册][registration]：

```java
// MOB_EFFECTS 是 DeferredRegister<MobEffect>
public static final Holder<MobEffect> MY_MOB_EFFECT = MOB_EFFECTS.register("my_mob_effect", () -> new MyMobEffect(
        // 效果类型：BENEFICIAL（有益）、NEUTRAL（中性）或 HARMFUL（有害），决定药水提示颜色
        MobEffectCategory.BENEFICIAL,
        // 效果粒子颜色（RGB格式）
        0xffffff
));
```

`MobEffect` 类提供默认功能，可为受影响的实体添加[属性修饰符][attributemodifier]，并在效果过期或被移除时清除（如速度效果添加移动速度修饰符）。添加方式如下：

```java
public static final Holder<MobEffect> MY_MOB_EFFECT = MOB_EFFECTS.register("my_mob_effect", () -> new MyMobEffect(...)
        .addAttributeModifier(Attributes.ATTACK_DAMAGE, ResourceLocation.fromNamespaceAndPath("examplemod", "effect.strength"), 2.0, AttributeModifier.Operation.ADD_VALUE)
);
```

### **瞬时效果**(`InstantenousMobEffect`)

创建瞬时效果可使用辅助类 `InstantenousMobEffect` 替代普通 `MobEffect`：

```java
public class MyMobEffect extends InstantenousMobEffect {
    public MyMobEffect(MobEffectCategory category, int color) {
        super(category, color);
    }

    @Override
    public void applyEffectTick(ServerLevel level, LivingEntity entity, int amplifier) {
        // 在此实现效果逻辑
    }
}
```

随后按常规[注册][registration]效果。

### **事件**(`Events`)

许多效果逻辑需在其他位置应用（如漂浮效果在生物移动处理器中实现）。模组 `MobEffect` 通常应在[事件处理器][events]中实现逻辑。NeoForge 提供相关事件：

- `MobEffectEvent.Applicable`：检查 `MobEffectInstance` 能否应用于实体时触发，可强制允许/阻止添加效果。
- `MobEffectEvent.Added`：`MobEffectInstance` 添加至目标时触发，含可能存在的旧效果信息。
- `MobEffectEvent.Expired`：效果实例到期（计时归零）时触发。
- `MobEffectEvent.Remove`：效果被移除（如饮用牛奶或命令）时触发。

## **`MobEffectInstance`s**

`MobEffectInstance` 即应用于实体的效果实例，通过构造函数创建：

```java
MobEffectInstance instance = new MobEffectInstance(
        // 使用的生物效果
        MobEffects.REGENERATION,
        // 持续时间（刻），未指定默认为0
        500,
        // 强度等级（即效果等级），范围0-255，未指定默认为0
        0,
        // 是否为环境效果（如信标/潮涌核心提供），未指定默认为false
        false,
        // 是否在物品栏显示，未指定默认为true
        true,
        // 是否在界面右上角显示图标，未指定默认为true
        true
);
```

提供多个构造函数重载，可省略最后1-5个参数。

:::info
`MobEffectInstance` 是可变的。如需副本，调用 `new MobEffectInstance(oldInstance)`。
:::

### 使用 `MobEffectInstance`s

添加效果至生物实体：
```java
MobEffectInstance instance = new MobEffectInstance(...);
livingEntity.addEffect(instance);
```

移除效果（因同种效果仅能存在一个实例，指定 `MobEffect` 即可）：
```java
livingEntity.removeEffect(MobEffects.REGENERATION);
```

:::info
`MobEffect` 仅能应用于 `LivingEntity` 或其子类（玩家/生物）。物品或雪球等实体不受影响。
:::

## **`Potion`s**

通过构造函数创建 `Potion` 并关联 `MobEffectInstance`：
```java
//POTIONS 是 DeferredRegister<Potion>
public static final Holder<Potion> MY_POTION = POTIONS.register("my_potion", registryName -> new Potion(
    // 药水后缀（用于生成翻译键）
    registryName.getPath(),
    // 药水效果（可变参数）
    new MobEffectInstance(MY_MOB_EFFECT, 3600)
));
```

药水名称为首个构造参数，用作翻译键后缀（如原版的"长效"和"强力"变体）。`MobEffectInstance` 参数为可变参数，可添加任意数量效果。空药水（无效果）可通过 `new Potion()` 创建（原版笨拙药水即如此实现）。

`PotionContents` 类提供药水物品的辅助方法。药水物品通过 `DataComponent#POTION_CONTENTS` 存储药水内容。

### **酿造**(`Brewing`)

添加药水后需实现生存模式获取途径。原版酿造台无[数据包][datapack]支持，需通过代码添加配方：

```java
@SubscribeEvent // 注册到游戏事件总线
public static void registerBrewingRecipes(RegisterBrewingRecipesEvent event) {
    // 获取配方构建器
    PotionBrewing.Builder builder = event.getBuilder();

    // 为所有容器药水（普通/喷溅/滞留）添加配方
    builder.addMix(
        // 基础药水
        Potions.AWKWARD,
        // 酿造原料（置于酿造台顶部）
        Items.FEATHER,
        // 产物药水
        MY_POTION
    );
}
```

[attributemodifier]: ../entities/attributes.md#attribute-modifiers
[block]: ../blocks/index.md
[commonsetup]: ../concepts/events.md#event-buses
[datapack]: ../resources/index.md#data
[events]: ../concepts/events.md
[item]: index.md
[itemstack]: index.md#itemstacks
[livingentity]: ../entities/livingentity.md
[registration]: ../concepts/registries.md#methods-for-registering
[uuidgen]: https://www.uuidgenerator.net/version4