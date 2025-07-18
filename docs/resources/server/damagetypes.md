# 伤害类型与伤害来源(`Damage Types & Damage Sources`)

**伤害类型**(`damage type`)表示施加给[**实体**][entity]的伤害种类 - 物理伤害、火焰伤害、溺水伤害、魔法伤害、虚空伤害等。伤害类型的区分用于各种免疫（例如烈焰人不会受到火焰伤害）、附魔（例如爆炸保护只对爆炸伤害有效）等多种用例。

可以说，伤害类型是伤害来源的模板。换句话说，**伤害来源**(`damage source`)可视为伤害类型的实例。伤害类型在代码中以[`ResourceKey`][rk]形式存在，但其所有属性都在数据包中定义。另一方面，伤害来源由游戏根据需要基于数据包文件中的值创建。它们可以持有额外的上下文，例如攻击实体。

## 创建伤害类型(`Creating Damage Types`)

首先，你需要创建自己的`DamageType`。`DamageType`是一个[**数据包注册表**][dr]，因此新的`DamageType`不是在代码中注册的，而是在添加相应文件时自动注册的。不过，我们仍然需要提供一个让代码获取伤害来源的点。我们通过指定[**资源键**][rk]来实现：

```java
public static final ResourceKey<DamageType> EXAMPLE_DAMAGE =
        ResourceKey.create(Registries.DAMAGE_TYPE, ResourceLocation.fromNamespaceAndPath(ExampleMod.MOD_ID, "example"));
```

现在我们可以从代码中引用它了，让我们在数据文件中指定一些属性。我们的数据文件位于`data/examplemod/damage_type/example.json`（将`examplemod`和`example`替换为你的模组 ID 和资源位置名称），内容如下：

```json5
{
    // 伤害类型的死亡消息ID。完整的死亡消息翻译键将是
    // "death.attack.examplemod.example"（替换了模组ID和名称）。
    "message_id": "examplemod.example",
    // 此伤害类型的伤害量是否随难度变化。有效的原版值有：
    // - "never"：在任何难度下伤害值保持不变。常用于玩家造成的伤害类型。
    // - "when_caused_by_living_non_player"：如果伤害由非玩家的活体实体（包括间接造成，如骷髅射出的箭）造成，则伤害值会缩放。
    // - "always"：伤害值总是缩放。通常用于爆炸类伤害。
    "scaling": "when_caused_by_living_non_player",
    // 受到此类伤害时引起的消耗值。
    "exhaustion": 0.1,
    // 受到此类伤害时应用的效果（目前只有音效）。可选。
    // 有效的原版值有 "hurt"（默认）、"thorns"、"drowning"、"burning"、"poking" 和 "freezing"。
    "effects": "hurt",
    // 死亡消息类型。决定死亡消息的构建方式。可选。
    // 有效的原版值有 "default"（默认）、"fall_variants" 和 "intentional_game_design"。
    "death_message_type": "default"
}
```

:::tip
`scaling`、`effects`和`death_message_type`字段在内部分别由枚举`DamageScaling`、`DamageEffects`和`DeathMessageType`控制。如果需要，可以[扩展][extenum]这些枚举以添加自定义值。
:::

相同的格式也用于原版的伤害类型，包开发者可以根据需要更改这些值。

## 创建和使用伤害来源(`Creating and Using Damage Sources`)

**伤害来源**(`DamageSource`)通常在调用[`Entity#hurt`][entityhurt]时动态创建。请注意，由于伤害类型是[**数据包注册表**][dr]，你需要一个`RegistryAccess`来查询它们，可以通过`Level#registryAccess`获取。要创建`DamageSource`，调用`DamageSource`构造函数并传入最多四个参数：

```java
DamageSource damageSource = new DamageSource(
        // 要使用的伤害类型持有器。从注册表查询。这是唯一必需的参数。
        registryAccess.lookupOrThrow(Registries.DAMAGE_TYPE).getOrThrow(EXAMPLE_DAMAGE),
        // 直接实体。例如，如果骷髅射中你，骷髅将是造成伤害的实体
        // （= 上面的参数），而箭将是直接实体（= 此参数）。与
        // 造成伤害的实体类似，这不总是适用，因此可为null。可选，默认为null。
        null,
        // 造成伤害的实体。这不总是适用（例如从世界掉落）
        // 因此可为null。可选，默认为null。
        null,
        // 伤害来源位置。很少使用，一个例子是故意游戏设计
        // （= 下界床爆炸）。可为null且可选，默认为null。
        null
);
```

:::warning
`DamageSources#source`是`new DamageSource`的包装器，它交换了第二和第三个参数（直接实体和造成伤害的实体）。确保将正确的值提供给正确的参数。
:::

如果`DamageSource`没有任何实体或位置上下文，将其缓存在字段中是合理的。对于具有实体或位置上下文的`DamageSource`，通常添加辅助方法，如下所示：

```java
public static DamageSource exampleDamage(Entity causer) {
    return new DamageSource(
            causer.level().registryAccess().lookupOrThrow(Registries.DAMAGE_TYPE).getOrThrow(EXAMPLE_DAMAGE),
            causer);
}
```

:::tip
原版的`DamageSource`工厂可以在`DamageSources`中找到，原版的`DamageType`资源键可以在`DamageTypes`中找到。实体还有`Entity#damageSources`方法，这是获取`DamageSources`实例的便捷方法。
:::

伤害来源的首要用例是`Entity#hurt`。每当实体受到伤害时都会调用此方法。要用我们自己的伤害类型伤害实体，我们只需自己调用`Entity#hurt`：

```java
// 第二个参数是伤害量，以半颗心为单位。
entity.hurt(exampleDamage(player), 10);
```

其他伤害类型特定的行为，如无敌检查，通常通过伤害类型[**标签**][tags]运行。这些由 Minecraft 和 NeoForge 添加，分别可以在`DamageTypeTags`和`Tags.DamageTypes`中找到。

## 数据生成(`Datagen`)

_更多信息，请参见[数据包注册表的数据生成][drdatagen]。_

伤害类型 JSON 文件可以进行[**数据生成**][datagen]。由于伤害类型是数据包注册表，我们通过`GatherDataEvent#createDatapackRegistryObjects`添加`DatapackBuiltinEntriesProvider`，并将我们的伤害类型放入`RegistrySetBuilder`：

```java
// 在你的数据生成类中
@SubscribeEvent // 在模组事件总线上
public static void onGatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(new RegistrySetBuilder()
        // 为伤害类型添加数据包内置条目提供器。如果此lambda变长，
        // 为了可读性，应将其提取到单独的方法中。
        .add(Registries.DAMAGE_TYPE, bootstrap -> {
            // 使用 new DamageType() 创建伤害类型的代码内表示。
            // 参数映射到JSON文件的值，顺序如上所示。
            // 除消息ID和消耗值外的所有参数都是可选的。
            bootstrap.register(EXAMPLE_DAMAGE, new DamageType(EXAMPLE_DAMAGE.location(),
                DamageScaling.WHEN_CAUSED_BY_LIVING_NON_PLAYER,
                0.1f,
                DamageEffects.HURT,
                DeathMessageType.DEFAULT)
            )
        })
        // 如果适用，添加其他数据包条目的数据包提供器。
        .add(...)
    );

    // ...
}
```

[datagen]: ../index.md#data-generation
[dr]: ../../concepts/registries.md#datapack-registries
[drdatagen]: ../../concepts/registries.md#data-generation-for-datapack-registries
[entity]: ../../entities/index.md
[entityhurt]: ../../entities/index.md#damaging-entities
[extenum]: ../../advanced/extensibleenums.md
[rk]: ../../misc/resourcelocation.md#resourcekeys
[tags]: tags.md