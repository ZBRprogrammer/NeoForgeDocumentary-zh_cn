# 数据映射(Data Maps)

**数据映射(Data Maps)** 包含数据驱动、可重载的对象，这些对象可以附加到注册对象上。该系统使游戏行为更易于数据驱动，因为它们提供同步或冲突解决等功能，从而带来更好、更可配置的用户体验。你可以将[标签(Tags)]视为注册对象➜布尔值的映射，而数据映射则是更灵活的注册对象➜对象映射。与[标签(Tags)]类似，数据映射会添加到对应的数据映射中，而不是覆盖。

数据映射可以附加到静态的内置注册表以及动态数据驱动的数据包注册表上。数据映射支持通过`/reload`命令或其他重新加载服务器资源的方式进行重载。

NeoForge为常见用例提供了多种[内置数据映射][builtin]，取代了硬编码的原版字段。更多信息可在链接文章中找到。

## 文件位置

数据映射从JSON文件加载，路径为：  
`<mapNamespace>/data_maps/<registryNamespace>/<registryPath>/<mapPath>.json`，其中：
- `<mapNamespace>` 是数据映射ID的命名空间
- `<mapPath>` 是数据映射ID的路径
- `<registryNamespace>` 是注册表ID的命名空间（若为`minecraft`则可省略）
- `<registryPath>` 是注册表ID的路径

示例：
- 数据映射`mymod:drop_healing`对应`minecraft:item`注册表（如下方示例），路径为：`mymod/data_maps/item/drop_healing.json`
- 数据映射`somemod:somemap`对应`minecraft:block`注册表，路径为：`somemod/data_maps/block/somemap.json`
- 数据映射`example:stuff`对应`somemod:custom`注册表，路径为：`example/data_maps/somemod/custom/stuff.json`

## JSON结构

数据映射文件可包含以下字段：
- `replace`：布尔值，在添加此文件值前清空数据映射。模组不应包含此字段，仅由包开发者用于覆盖映射
- `neoforge:conditions`：[加载条件][conditions]列表
- `values`：注册表ID或标签ID到值的映射，这些值将被添加到数据映射。值结构由数据映射的编解码器(Codec)定义（见下文）
- `remove`：要从数据映射中移除的注册表ID或标签ID列表

### 添加值

例如，假设我们有一个针对`minecraft:item`注册表的数据映射对象，包含两个浮点键`amount`和`chance`。对应的数据映射文件如下：
```json5
{
    "values": {
        // 为胡萝卜物品附加值
        "minecraft:carrot": {
            "amount": 12,
            "chance": 1
        },
        // 为logs标签中的所有物品附加值
        "#minecraft:logs": {
            "amount": 1,
            "chance": 0.1
        }
    }
}
```

数据映射可能支持[合并器(Mergers)][mergers]，在冲突时（如两个模组为同一物品添加值）触发自定义合并行为。要避免触发合并器，可在元素级指定`replace`字段：
```json5
{
    "values": {
        // 覆盖胡萝卜物品的值
        "minecraft:carrot": {
            // highlight-next-line
            "replace": true,
            // 新值位于value子对象下
            "value": {
                "amount": 12,
                "chance": 1
            }
        }
    }
}
```

### 移除现有值

通过指定要移除的物品ID或标签ID列表实现移除：
```json5
{
    // 即使其他模组的数据映射添加了值，我们也不希望马铃薯有值
    "remove": [
        "minecraft:potato"
    ]
}
```

移除操作在添加后执行，因此可先包含标签再排除特定元素：
```json5
{
    "values": {
        "#minecraft:logs": { /* ... */ }
    },
    // 再次排除绯红菌柄
    "remove": [
        "minecraft:crimson_stem"
    ]
}
```

数据映射可能支持带额外参数的自定义移除器(Removers)。此时`remove`列表可转换为JSON对象：键为待移除元素，值为额外数据。例如，假设移除器对象序列化为字符串：
```json5
{
    "remove": {
        // 移除器将从值（此处为"somekey1"）反序列化
        // 并应用于胡萝卜物品的附加值
        "minecraft:carrot": "somekey1"
    }
}
```

## 自定义数据映射

首先定义数据映射条目的格式。**数据映射条目必须不可变**，因此记录类(Record)是理想选择。沿用前文示例（含浮点值`amount`和`chance`）：
```java
public record ExampleData(float amount, float chance) {}
```

与其他组件类似，数据映射使用编解码器(Codecs)序列化和反序列化。需为条目提供编解码器：
```java
public record ExampleData(float amount, float chance) {
    public static final Codec<ExampleData> CODEC = RecordCodecBuilder.create(instance -> instance.group(
            Codec.FLOAT.fieldOf("amount").forGetter(ExampleData::amount),
            Codec.floatRange(0, 1).fieldOf("chance").forGetter(ExampleData::chance)
    ).apply(instance, ExampleData::new));
}
```

接着创建数据映射本身：
```java
// 此示例针对minecraft:item注册表，因此泛型使用Item。
// 若为其他注册表创建，需调整类型。
public static final DataMapType<Item, ExampleData> EXAMPLE_DATA = DataMapType.builder(
        // 数据映射ID。对应文件路径：
        // <yourmodid>/data_maps/item/example_data.json
        ResourceLocation.fromNamespaceAndPath("examplemod", "example_data"),
        // 目标注册表
        Registries.ITEM,
        // 条目的编解码器
        ExampleData.CODEC
).build();
```

最后在[模组事件总线][modbus]的[`RegisterDataMapTypesEvent`][events]中注册：
```java
@SubscribeEvent // 在模组事件总线上
public static void registerDataMapTypes(RegisterDataMapTypesEvent event) {
    event.register(EXAMPLE_DATA);
}
```

### 同步(Syncing)

同步的数据映射会将其值同步到客户端。通过构建器调用`#synced`标记为同步：
```java
public static final DataMapType<Item, ExampleData> EXAMPLE_DATA = DataMapType.builder(...)
        .synced(
                // 同步用的编解码器。可与普通编解码器相同，
                // 也可省略客户端不需要的字段
                ExampleData.CODEC,
                // 数据映射是否必需。标记为必需时，
                // 缺少此映射的客户端（包括原版客户端）将被断开连接
                false
        ).build();
```

### 使用

由于数据映射可用于任何注册表，必须通过`持有者(Holder)`而非注册对象查询。且仅适用于引用持有者(Reference Holder)，而非直接持有者(Direct Holder)。但多数场景（如`Registry#wrapAsHolder`、`Registry#getHolder`或各类`builtInRegistryHolder`方法）会返回引用持有者，因此通常不是问题。

通过`Holder#getData(DataMapType)`查询数据映射值。若对象无附加值，则返回`null`。沿用`ExampleData`示例，在玩家拾取物品时治疗玩家：
```java
@SubscribeEvent // 在游戏事件总线上
public static void itemPickup(ItemEntityPickupEvent.Post event) {
    ItemStack stack = event.getItemStack();
    // 通过ItemStack#getItemHolder获取Holder<Item>
    Holder<Item> holder = stack.getItemHolder();
    // 从持有者获取数据
    //highlight-next-line
    ExampleData data = holder.getData(EXAMPLE_DATA);
    if (data != null) {
        // 存在值，执行操作！
        Player player = event.getPlayer();
        if (player.getLevel().getRandom().nextFloat() > data.chance()) {
            player.heal(data.amount());
        }
    }
}
```

此流程同样适用于NeoForge提供的所有数据映射。

## 高级数据映射

高级数据映射使用`AdvancedDataMapType`（标准`DataMapType`的子类）。额外支持自定义合并器和移除器。对于值类似集合（如`List`或`Map`）的数据映射，强烈推荐实现此功能。

`DataMapType`有两个泛型`R`（注册表类型）和`T`（值类型），而`AdvancedDataMapType`多一个`VR extends DataMapValueRemover<R, T>`，确保移除器在数据生成时类型安全。

通过`AdvancedDataMapType#builder()`（而非`DataMapType#builder()`）创建，返回`AdvancedDataMapType.Builder`。此构建器额外提供`#remover`和`#merger`方法分别指定移除器和合并器（见下文）。同步等其他功能保持不变。

### 合并器(Mergers)

合并器用于处理多个数据包为同一对象添加值的冲突。默认合并器(`DataMapValueMerger#defaultMerger`)会覆盖现有值（如低优先级数据包的值），若需不同行为则需自定义合并器。

合并器接收两个冲突值、值附加对象（`Either<TagKey<R>, ResourceKey<R>>`，因值可附加到标签或单个对象）及其注册表，返回应附加的实际值。通常应优先合并而非覆盖（仅在常规合并无效时覆盖）。数据包可通过元素级`replace`字段绕过合并器（见[添加值][add]）。

假设数据映射为物品添加整数值。通过相加解决冲突：
```java
public class IntMerger implements DataMapValueMerger<Item, Integer> {
    @Override
    public Integer merge(Registry<Item> registry,
            Either<TagKey<Item>, ResourceKey<Item>> first, Integer firstValue,
            Either<TagKey<Item>, ResourceKey<Item>> second, Integer secondValue) {
        return firstValue + secondValue;
    }
}
```

若一个包为`minecraft:carrot`设置值12，另一个设置15，最终值为27。若任一包指定`"replace": true`，则使用该包的值；若两者都指定，则使用高优先级包的值。

最后在构建器中指定合并器：
```java
// 数据映射类型必须与合并器匹配
AdvancedDataMapType<Item, Integer> ADVANCED_MAP = AdvancedDataMapType.builder(...)
        .merger(new IntMerger())
        .build();
```

:::tip
NeoForge在`DataMapValueMerger`中为列表(List)、集合(Set)和映射(Map)提供了默认合并器。
:::

### 移除器(Removers)

类似于复杂数据的合并器，移除器用于正确处理元素的`remove`子句。默认移除器(`DataMapValueRemover.Default.INSTANCE`)会移除对象的所有关联信息，自定义移除器可仅移除部分数据。

构建器中传递的编解码器用于解码移除器实例。移除器接收当前附加值及其来源，返回替换值的`Optional`。返回空`Optional`则完全移除值。

以下示例展示从基于`Map<String, String>`的数据映射中移除特定键的移除器：
```java
public record MapRemover(String key) implements DataMapValueRemover<Item, Map<String, String>> {
    public static final Codec<MapRemover> CODEC = Codec.STRING.xmap(MapRemover::new, MapRemover::key);
    
    @Override
    public Optional<Map<String, String>> remove(Map<String, String> value, Registry<Item> registry, Either<TagKey<Item>, ResourceKey<Item>> source, Item object) {
        final Map<String, String> newMap = new HashMap<>(value);
        newMap.remove(key);
        return Optional.of(newMap);
    }
}
```

考虑以下数据文件：
```json5
{
    "values": {
        "minecraft:carrot": {
            "somekey1": "value1",
            "somekey2": "value2"
        }
    }
}
```

更高优先级的第二个数据文件：
```json5
{
    "remove": {
        // 移除器解码为字符串，因此值直接使用字符串
        // 若解码为对象，则需使用对象形式
        "minecraft:carrot": "somekey1"
    }
}
```

最终结果为：
```json5
{
    "values": {
        "minecraft:carrot": {
            "somekey2": "value2"
        }
    }
}
```

与合并器类似，需添加到构建器。注意此处直接使用编解码器：
```java
// 假设AdvancedData包含Map<String, String>属性
AdvancedDataMapType<Item, AdvancedData> ADVANCED_MAP = AdvancedDataMapType.builder(...)
        .remover(MapRemover.CODEC)
        .build();
```

## 数据生成(Data Generation)

通过继承`DataMapProvider`并重写`#gather`生成数据映射条目。沿用`ExampleData`示例：
```java
public class MyDataMapProvider extends DataMapProvider {
    public MyDataMapProvider(PackOutput packOutput, CompletableFuture<HolderLookup.Provider> lookupProvider) {
        super(packOutput, lookupProvider);
    }
    
    @Override
    protected void gather() {
        // 为EXAMPLE_DATA创建构建器，用#add添加条目
        this.builder(EXAMPLE_DATA)
                // 启用覆盖（模组切勿使用！仅用于演示）
                .replace(true)
                // 为所有台阶添加值。布尔参数控制"replace"字段（模组中应为false）
                .add(ItemTags.SLABS, new ExampleData(10, 1), false)
                // 为苹果添加值
                .add(Items.APPLE.builtInRegistryHolder(), new ExampleData(5, 0.2f), false)
                // 再次移除木质台阶
                .remove(ItemTags.WOODEN_SLABS)
                // 添加Botania模组加载条件
                .conditions(new ModLoadedCondition("botania"));
    }
}
```

生成的JSON文件：
```json5
{
    "replace": true,
    "values": {
        "#minecraft:slabs": {
            "amount": 10,
            "chance": 1.0
        },
        "minecraft:apple": {
            "amount": 5,
            "chance": 0.2
        }
    },
    "remove": [
        "#minecraft:wooden_slabs"
    ],
    "neoforge:conditions": [
        {
            "type": "neoforge:mod_loaded",
            "modid": "botania"
        }
    ]
}
```

与其他数据提供器一样，需添加到事件：
```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    // 若需添加数据包对象，先调用event.createDatapackRegistryObjects(...)
    event.createProvider(MyDataMapProvider::new);
}
```

[builtin]: builtin.md
[codecs]: ../../../datastorage/codecs.md
[conditions]: ../conditions.md
[datagen]: ../../index.md#data-generation
[events]: ../../../concepts/events.md
[add]: #adding-values
[mergers]: #mergers
[modbus]: ../../../concepts/events.md#event-buses
[removers]: #removers
[tags]: ../tags.md