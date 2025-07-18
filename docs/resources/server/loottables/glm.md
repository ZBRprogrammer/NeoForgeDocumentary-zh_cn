# **全局战利品修改器** (`Global Loot Modifiers`)

**全局战利品修改器** (`Global Loot Modifiers`)，简称 **GLM**，是一种**数据驱动的方式** (`data-driven way`)，用于修改战利品掉落，无需覆写大量原版**战利品表** (`loot table`)，也无需处理与其他模组的战利品表交互（尤其在未知加载模组的情况下）。

GLM 的工作流程是：先执行关联的[战利品表][loottable]的掉落计算，然后将 GLM 应用于计算结果。GLM **具有叠加性** (`stacking`)，而非**最后加载者生效** (`last-load-wins`)，允许多个模组修改同一战利品表，这点与[标签][tags]类似。

注册一个 GLM 需要以下四部分：

- 一个 `global_loot_modifiers.json` 文件，位于 `data/neoforge/loot_modifiers/global_loot_modifiers.json` (**不在你的模组命名空间内**)。此文件告知 NeoForge 应用哪些修改器及其顺序。
- 一个表示战利品修改器的 JSON 文件。该文件包含修改所需的所有数据，允许数据包调整效果。它位于 `data/<命名空间>/loot_modifiers/<路径>.json`。
- 一个实现 `IGlobalLootModifier` 或继承 `LootModifier`（其实现了 `IGlobalLootModifier`）的类。该类包含实现修改器功能的代码。
- 一个用于编码和解码战利品修改器类的映射 [编解码器][codec] (`codec`)。通常作为 `public static final` 字段在战利品修改器类中实现。

## `global_loot_modifiers.json`

`global_loot_modifiers.json` 文件告知 NeoForge 将哪些修改器应用于战利品表。该文件可包含两个键：

- `entries` 是要加载的修改器列表。指定的 [`ResourceLocation`][resloc] 指向其在 `data/<命名空间>/loot_modifiers/<路径>.json` 中的关联条目。此列表是**有序的** (`ordered`)，意味着修改器将按指定顺序应用，这在处理模组兼容性问题时有时很重要。
- `replace` 表示修改器应替换旧的 (`true`) 还是仅添加到现有列表 (`false`)。其工作方式类似[标签][tags]中的 `replace` 键，但**此处该键是必需的**。通常，模组开发者应始终使用 `false`；`true` 的功能面向模组包或数据包开发者。

示例用法：
```json5
{
    "replace": false, // 必须存在
    "entries": [
        // 指向 data/examplemod/loot_modifiers/example_glm_1.json 中的战利品修改器
        "examplemod:example_glm_1",
        "examplemod:example_glm_2"
        // ...
    ]
}
```

## 战利品修改器 JSON 文件

此文件包含与修改器相关的所有值，例如应用概率、要添加的物品等。建议尽可能避免硬编码值，以便数据包制作者调整平衡性。一个战利品修改器必须至少包含两个字段，并可根据情况包含更多：

- `type` 字段包含战利品修改器的注册名。
- `conditions` 字段是此修改器激活所需的**战利品表条件** (`loot table conditions`) 列表。
- 其他属性可能是必需或可选的，具体取决于使用的编解码器。

:::tip
GLM 的常见用例是向特定战利品表添加额外战利品。为此，可使用 [`neoforge:loot_table_id` 条件][loottableid]。
:::

示例用法如下：
```json5
{
    // 这是战利品修改器的注册名
    "type": "examplemod:my_loot_modifier",
    "conditions": [
        // 战利品表条件写在此处
    ],
    // 编解码器指定的额外属性
    "field1": "somestring",
    "field2": 10,
    "field3": "minecraft:dirt"
}
```

## `IGlobalLootModifier` 与 `LootModifier`

要将战利品修改器实际应用于战利品表，必须指定 `IGlobalLootModifier` 的实现。大多数情况下，你会使用其子类 `LootModifier`，它已为你处理条件等逻辑。首先在战利品修改器类中继承 `LootModifier`：

```java
// 无法使用 record，因为 record 不能继承其他类。
public class MyLootModifier extends LootModifier {
    // 编解码器实现见下文。
    public static final MapCodec<MyLootModifier> CODEC = ...;
    // 我们的额外属性。
    private final String field1;
    private final int field2;
    private final Item field3;
    
    // 第一个构造参数是条件列表，其余是额外属性。
    public MyLootModifier(LootItemCondition[] conditions, String field1, int field2, Item field3) {
        super(conditions);
        this.field1 = field1;
        this.field2 = field2;
        this.field3 = field3;
    }
    
    // 在此返回编解码器。
    @Override
    public MapCodec<? extends IGlobalLootModifier> codec() {
        return CODEC;
    }
    
    // 核心逻辑在此实现。如需使用额外属性，请在此处操作。
    // 参数：现有战利品 (generatedLoot) 和战利品上下文 (context)。
    @Override
    protected ObjectArrayList<ItemStack> doApply(ObjectArrayList<ItemStack> generatedLoot, LootContext context) {
        // 在此将你的物品添加到 generatedLoot
        return generatedLoot;
    }
}
```

:::info
修改器返回的战利品列表将按注册顺序输入给其他修改器。因此，修改后的战利品预期会被另一个战利品修改器再次修改。
:::

## 战利品修改器编解码器 (`Codec`)

为了让游戏识别我们的战利品修改器，必须为其定义并[注册][register]一个[编解码器][codec]。沿用之前三个字段的示例，代码如下：

```java
public static final MapCodec<MyLootModifier> CODEC = RecordCodecBuilder.mapCodec(inst -> 
        // LootModifier#codecStart 添加 conditions 字段。
        LootModifier.codecStart(inst).and(inst.group(
                Codec.STRING.fieldOf("field1").forGetter(e -> e.field1),
                Codec.INT.fieldOf("field2").forGetter(e -> e.field2),
                BuiltInRegistries.ITEM.byNameCodec().fieldOf("field3").forGetter(e -> e.field3)
        )).apply(inst, MyLootModifier::new)
);
```

随后，将编解码器[注册][register]到注册表：
```java
public static final DeferredRegister<MapCodec<? extends IGlobalLootModifier>> GLOBAL_LOOT_MODIFIER_SERIALIZERS =
        DeferredRegister.create(NeoForgeRegistries.Keys.GLOBAL_LOOT_MODIFIER_SERIALIZERS, ExampleMod.MOD_ID);

public static final Supplier<MapCodec<MyLootModifier>> MY_LOOT_MODIFIER =
        GLOBAL_LOOT_MODIFIER_SERIALIZERS.register("my_loot_modifier", () -> MyLootModifier.CODEC);
```

## 内置战利品修改器 (`Builtin Loot Modifiers`)

NeoForge 提供了一个开箱即用的战利品修改器：

### `neoforge:add_table`

此战利品修改器会执行第二个战利品表的掉落计算，并将结果添加到应用此修改器的战利品表中。

```json5
{
    "type": "neoforge:add_table",
    "conditions": [], // 所需的战利品条件
    "table": "minecraft:chests/abandoned_mineshaft" // 要执行的第二个战利品表
}
```

## 数据生成 (`Datagen`)

GLM 支持[数据生成][datagen]。通过继承 `GlobalLootModifierProvider` 实现：

```java
public class MyGlobalLootModifierProvider extends GlobalLootModifierProvider {
    // 从 `GatherDataEvent` 获取参数。
    public MyGlobalLootModifierProvider(PackOutput output, CompletableFuture<HolderLookup.Provider> registries) {
        super(output, registries, ExampleMod.MOD_ID);
    }
    
    @Override
    protected void start() {
        // 调用 #add 添加新 GLM，同时会在 global_loot_modifiers.json 添加对应条目。
        this.add(
                // 修改器名称，即文件名。
                "my_loot_modifier_instance",
                // 要添加的战利品修改器。示例添加了天气条件。
                new MyLootModifier(new LootItemCondition[] {
                        WeatherCheck.weather().setRaining(true).build()
                }, "somestring", 10, Items.DIRT),
                // 数据加载条件列表。注意：这些条件与修改器自身的战利品条件无关。
                // 示例添加了模组加载条件。
                // 另有 #add 重载方法，接受条件可变参数 (vararg) 而非列表。
                List.of(new ModLoadedCondition("create"))
        );
    }
}
```

与其他数据提供器一样，需在 `GatherDataEvent` 中注册此提供器：
```java
@SubscribeEvent // 在模组事件总线上
public static void onGatherData(GatherDataEvent event) { // 注意: 原文件为 Client 事件，但实际通常使用主事件
    // 如需添加数据包对象，先调用 event.createDatapackRegistryObjects(...)

    event.getGenerator().addProvider(
        event.includeServer(), // 通常包含在服务端数据中
        new MyGlobalLootModifierProvider(event.getGenerator().getPackOutput(), event.getLookupProvider())
    );
}
```

[codec]: ../../../datastorage/codecs.md
[datagen]: ../../index.md#data-generation
[loottable]: index.md
[loottableid]: lootconditions#neoforgeloot_table_id
[register]: ../../../concepts/registries.md#methods-for-registering
[resloc]: ../../../misc/resourcelocation.md
[tags]: ../tags.md