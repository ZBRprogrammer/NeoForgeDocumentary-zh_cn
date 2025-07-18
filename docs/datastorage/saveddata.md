---
sidebar_position: 5
---
# **保存的数据**(`Saved Data`)

**保存的数据**(`Saved Data, SD`)系统可用于在**世界层级**(`levels`)上保存额外数据。

_若数据特定于某些方块实体、区块或实体，请考虑改用**数据附件**(`data attachment`)。_

## `SavedData`

每个 SD 实现必须继承 `SavedData` 类。可像其他对象一样实现（包含自定义字段和方法），但若需存储数据或写入磁盘，必须调用 `setDirty`。`setDirty` 会通知游戏有需要写入的更改。若未调用，则数据仅在当前世界层级（或主世界层级的整个世界）加载期间保留。

```java
// For some saved data implementation
public class ExampleSavedData extends SavedData {

    public void foo() {
        // Change data in saved data
        // Call set dirty if data changes
        this.setDirty();
    }
}
```

## `SavedDataType`

由于 `SavedData` 仅为对象，需有某种关联标识符。此外还需将数据读写到磁盘，这就是 `SavedDataType` 的作用。它接收保存数据的标识符、无数据时的默认构造函数，以及用于编码/解码数据的[**编解码器**(`codec`)][codec]。标识符被视为世界文件夹和世界维度内的路径位置：`./<world_folder>/<level_name>/data/<identifier>.dat`。所有缺失目录（包括标识符中的目录）都会被创建。

:::note
`SavedDataType` 有第四个参数 `DataFixTypes`，但因 NeoForge 不支持**数据修复器**(`data fixers`)，所有原版用例已修补为允许空值。
:::

`SavedDataType` 构造函数有两种变体。第一种接收构造函数的简单 `Supplier` 和用于磁盘处理的常规 `Codec`。但若需存储当前 `ServerLevel` 或世界种子，有一个重载版本接收 `Function`（两者均提供 `SavedData.Context`）。

```java
// For some saved data implementation
public class NoContextExampleSavedData extends SavedData {

    public static final SavedDataType<NoContextExampleSavedData> ID = new SavedDataType<>(
        // The identifier of the saved data
        // Used as the path within the level's `data` folder
        "example",
        // The initial constructor
        NoContextExampleSavedData::new,
        // The codec used to serialize the data
        RecordCodecBuilder.create(instance -> instance.group(
            Codec.INT.fieldOf("val1").forGetter(sd -> sd.val1),
            BuiltInRegistries.BLOCK.byNameCodec().fieldOf("val2").forGetter(sd -> sd.val2)
        ).apply(instance, NoContextExampleSavedData::new))
    );

    // Initial constructor
    public NoContextExampleSavedData() {
        // ...
    }

    // Data constructor
    public NoContextExampleSavedData(int val1, Block val2) {
        // ...
    }

    public void foo() {
        // Change data in saved data
        // Call set dirty if data changes
        this.setDirty();
    }
}

// For some saved data implementation
public class ContextExampleSavedData extends SavedData {

    public static final SavedDataType<ContextExampleSavedData> ID = new SavedDataType<>(
        // The identifier of the saved data
        // Used as the path within the level's `data` folder
        "example",
        // The initial constructor
        ContextExampleSavedData::new,
        // The codec used to serialize the data
        ctx -> RecordCodecBuilder.create(instance -> instance.group(
            RecordCodecBuilder.point(ctx),
            Codec.INT.fieldOf("val1").forGetter(sd -> sd.val1),
            BuiltInRegistries.BLOCK.byNameCodec().fieldOf("val2").forGetter(sd -> sd.val2)
        ).apply(instance, ContextExampleSavedData::new))
    );

    // Initial constructor
    public ContextExampleSavedData(SavedData.Context ctx) {
        // ...
    }

    // Data constructor
    public ContextExampleSavedData(SavedData.Context ctx, int val1, Block val2) {
        // ...
    }

    public void foo() {
        // Change data in saved data
        // Call set dirty if data changes
        this.setDirty();
    }
}
```

## 关联到世界层级

任何 `SavedData` 都是动态加载/关联到世界层级的。因此若从未在世界层级上创建，则不会存在。

可通过调用 `ServerChunkCache#getDataStorage` 或 `ServerLevel#getDataStorage` 访问 `DimensionDataStorage` 来创建和加载 `SavedData`。然后通过调用 `DimensionDataStorage#computeIfAbsent`（传入 `SavedDataType`）获取或创建 SD 实例。这将尝试获取当前 SD 实例（若存在），或创建新实例并加载所有可用数据。

```java
// In some method with access to the DimensionDataStorage
netherDataStorage.computeIfAbsent(ContextExampleSavedData.ID);
```

若 SD 不特定于某个世界层级，应将其关联到**主世界**(`Overworld`)，可通过 `MinecraftServer#overworld` 获取。主世界是唯一永远不会完全卸载的维度，因此非常适合存储多层级数据。

[codec]: codecs.md