# 数据加载条件(`Data Load Conditions`)

有时，如果另一个模组存在，或者任何模组添加了另一种类型的矿石等，需要禁用或启用某些功能。对于这些用例，NeoForge 添加了**数据加载条件**(`data load conditions`)。这些最初被称为**配方条件**(`recipe conditions`)，因为配方是该系统的原始用例，但后来已扩展到其他系统。这也是为什么一些内置条件仅限于物品。

大多数 JSON 文件可以选择在根节点声明一个`neoforge:conditions`块，该块将在数据文件实际加载之前进行评估。当且仅当所有条件都通过时，加载才会继续，否则数据文件将被忽略。（此规则的例外是[战利品表][loottable]，它将被替换为空战利品表。）

```json5
{
    "neoforge:conditions": [
        {
            // 条件 1
        },
        {
            // 条件 2
        },
        // ...
    ],
    // 数据文件的其余部分
}
```

:::note
如果要加载的值不是映射/对象，则存储在`neoforge:value`中：

```json5
{
    "neoforge:conditions": [ /* ...*/ ],
    "neoforge:value": 2 // 要加载的值
}
```
:::

例如，如果我们只想在存在 ID 为`examplemod`的模组时加载文件，我们的文件将如下所示：

```json5
{
    // highlight-start
    "neoforge:conditions": [
        {
            "type": "neoforge:mod_loaded",
            "modid": "examplemod"
        }
    ],
    // highlight-end
    "type": "minecraft:crafting_shaped",
    // ...
}
```

:::note
大多数原版文件已使用`ConditionalCodec`包装器修补为使用条件。然而，并非所有系统，特别是那些不使用[编解码器][codec]的系统，都可以使用条件。要确定数据文件是否可以使用条件，请检查支持的编解码器定义。
:::

## 内置条件(`Built-In Conditions`)

### **`neoforge:always`** 和 **`neoforge:never`**

这些不包含数据，并返回预期值。

```json5
{
    // 始终返回 true（对于 "neoforge:never" 返回 false）
    "type": "neoforge:always"
}
```

:::tip
使用**`neoforge:never`**条件可以非常干净地禁用任何数据文件。只需在所需位置放置包含以下内容的文件：

```json5
{"neoforge:conditions":[{"type":"neoforge:never"}]}
```

以这种方式禁用文件**不会**导致日志垃圾邮件。
:::

### **`neoforge:not`**

此条件接受另一个条件并反转其结果。

```json5
{
    // 反转存储条件的结果
    "type": "neoforge:not",
    "value": {
        // 另一个条件
    }
}
```

### **`neoforge:and`** 和 **`neoforge:or`**

这些条件接受要操作的条件，并应用预期的逻辑。接受的条件数量没有限制。

```json5
{
    // 将存储的条件 AND 在一起（对于 "neoforge:or" 则是 OR）
    "type": "neoforge:and",
    "values": [
        {
            // 第一个条件
        },
        {
            // 第二个条件
        }
    ]
}
```

### **`neoforge:mod_loaded`**

如果加载了具有给定模组 ID 的模组，则此条件返回 true，否则返回 false。

```json5
{
    "type": "neoforge:mod_loaded",
    // 如果加载了 "examplemod" 则返回 true
    "modid": "examplemod"
}
```

### **`neoforge:registered`**

如果在特定注册表中使用给定注册名称注册了对象，则此条件返回 true，否则返回 false。

```json5
{
    "type": "neoforge:registered",
    // 要检查值的注册表
    // 默认为 `minecraft:item`
    "registry": "minecraft:item",
    // 如果已注册 "examplemod:example_item" 则返回 true
    "value": "examplemod:example_item"
}
```

### **`neoforge:tag_empty`**

如果给定的注册表[标签][tag]为空，则此条件返回 true，否则返回 false。

```json5
{
    "type": "neoforge:tag_empty",
    // 要检查标签的注册表
    // 默认为 `minecraft:item`
    "registry": "minecraft:item",
    // 如果 "examplemod:example_tag" 是空标签则返回 true
    "tag": "examplemod:example_tag"
}
```

### **`neoforge:feature_flags_enabled`**

如果提供的[功能标志][flags]已启用，则此条件返回 true，否则返回 false。

```json5
{
    "type": "neoforge:feature_flags_enabled",
    // 如果启用了 "examplemod:example_feature" 则返回 true
    "flags": [
        "examplemod:example_feature"
    ]
}
```

## 创建自定义条件(`Creating Custom Conditions`)

可以通过实现`ICondition`及其`#test(IContext)`方法，并为其创建[映射编解码器][codec]来创建自定义条件。`#test`中的`IContext`参数可以访问游戏状态的某些部分。目前，这只允许你从注册表中查询标签。一些带有条件的对象可能比标签加载得更早，在这种情况下，上下文将是`IContext.EMPTY`，并且根本不包含任何标签信息。

例如，假设我们要实现一个`xor`条件，那么我们的条件将如下所示：

```java
public record XorCondition(ICondition first, ICondition second) implements ICondition {
    public static final MapCodec<XorCondition> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            ICondition.CODEC.fieldOf("first").forGetter(XorCondition::first),
            ICondition.CODEC.fieldOf("second").forGetter(XorCondition::second)
    ).apply(inst, XorCondition::new));

    @Override
    public boolean test(ICondition.IContext context) {
        return this.first.test(context) ^ this.second.test(context);
    }

    @Override
    public MapCodec<? extends ICondition> codec() {
        return CODEC;
    }
}
```

条件是编解码器的注册表。因此，我们需要[注册][register]我们的编解码器，如下所示：

```java
public static final DeferredRegister<MapCodec<? extends ICondition>> CONDITION_CODECS =
        DeferredRegister.create(NeoForgeRegistries.Keys.CONDITION_CODECS, ExampleMod.MOD_ID);

public static final Supplier<MapCodec<XorCondition>> XOR =
        CONDITION_CODECS.register("xor", () -> XorCondition.CODEC);
```

然后，我们可以在某些数据文件中使用我们的条件（假设我们在`examplemod`命名空间下注册了条件）：

```json5
{
    "neoforge:conditions": [
        {
            "type": "examplemod:xor",
            "first": {
                // 要么此条件为真
                "type": "..."
            },
            "second": {
                // 要么此条件为真，但不能同时为真！
                "type": "..."
            }
        }
    ],
    // 数据文件的其余部分
}
```

## 数据生成(`Datagen`)

虽然任何数据包 JSON 文件都可以使用加载条件，但只有少数[数据提供器][datagen]已修改为能够生成它们。这些包括：

- [`RecipeProvider`][recipeprovider]（通过`RecipeOutput#withConditions`），包括配方进度
- `JsonCodecProvider`及其子类`SpriteSourceProvider`
- [`DataMapProvider`][datamapprovider]
- [`GlobalLootModifierProvider`][glmprovider]
- [`DatapackBuiltinEntriesProvider`][datapackentries]（通过`Map<ResourceKey<?>, List<ICondition>>`参数）

对于条件本身，`NeoForgeConditions`类为每个内置条件类型提供了静态辅助方法，返回相应的`ICondition`。

[codec]: ../../datastorage/codecs
[datagen]: ../index.md#data-generation
[datamapprovider]: datamaps/index.md#data-generation
[datapackentries]: ../../concepts/registries.md#data-generation-for-datapack-registries
[flags]: ../../advanced/featureflags.md
[glmprovider]: loottables/glm.md#datagen
[loottable]: loottables/index.md
[recipeprovider]: recipes/index.md#data-generation
[register]: ../../concepts/registries
[tag]: tags.md