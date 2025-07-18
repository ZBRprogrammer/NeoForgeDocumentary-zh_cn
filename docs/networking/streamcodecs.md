---
sidebar_position: 2
---
# **流编解码器**(`Stream Codecs`)

**流编解码器**(`Stream codecs`)是用于描述如何将对象存储/读取到流（如缓冲区）的序列化工具，主要被原版(`vanilla`)的[网络系统][networking]用于数据同步。

:::info
流编解码器与[编解码器(`codecs`)][codecs]类似，本页采用相同格式展示其相似性。
:::

## 使用流编解码器

流编解码器通过 `StreamCodec#encode` 和 `StreamCodec#decode` 分别将对象编码/解码到流中。`encode` 接收流和待编码对象，`decode` 从流中读取并返回解码对象。通常流类型为 `ByteBuf`、`FriendlyByteBuf` 或 `RegistryFriendlyByteBuf`。

```java
// 设 exampleStreamCodec 表示 StreamCodec<ExampleJavaObject>
// 设 exampleObject 为 ExampleJavaObject 实例
// 设 buffer 为 RegistryFriendlyByteBuf 实例

// 将 Java 对象编码到缓冲区流
exampleStreamCodec.encode(buffer, exampleObject);

// 从缓冲区流读取 Java 对象
ExampleJavaObject obj = exampleStreamCodec.decode(buffer);
```

:::note
除非手动处理缓冲区对象，否则通常无需调用 `encode` 和 `decode`。
:::

## 现有流编解码器

### `ByteBufCodecs`

`ByteBufCodecs` 包含特定基元类型和对象的静态编解码器实例。

| 流编解码器        | Java 类型        |
|-------------------|------------------|
| `BOOL`            | `Boolean`        |
| `BYTE`            | `Byte`           |
| `SHORT`           | `Short`          |
| `INT`             | `Integer`        |
| `LONG`            | `Long`           |
| `FLOAT`           | `Float`          |
| `DOUBLE`          | `Double`         |
| `BYTE_ARRAY`      | `byte[]`\*       |
| `LONG_ARRAY`      | `long[]`         |
| `STRING_UTF8`     | `String`\*\*     |
| `TAG`             | `Tag`            |
| `COMPOUND_TAG`    | `CompoundTag`    |
| `VECTOR3F`        | `Vector3f`       |
| `QUATERNIONF`     | `Quaternionf`    |
| `GAME_PROFILE`    | `GameProfile`    |

\* `byte[]` 可通过 `ByteBufCodecs#byteArray` 限制元素数量  
\*\* `String` 可通过 `ByteBufCodecs#stringUtf8` 限制字符数  

#### 无符号短整型

`UNSIGNED_SHORT` 是 `SHORT` 的替代方案，作为无符号数处理。由于 Java 中数字有符号，无符号短整型以 `Integer` 发送/接收，高位两个字节被屏蔽。

#### 变长数值

`VAR_INT` 和 `VAR_LONG` 是按需压缩的流编解码器。通过每次编码 7 位（高位标记是否还有数据）实现：
- 整数范围 0 到 2^28-1 或长整数范围 0 到 2^56-1 时，传输字节数 ≤ 标准整数/长整数
- 若数值通常在此范围内且偏小，应使用变长编解码器

:::note
`VAR_INT` 和 `VAR_LONG` 分别是 `INT` 和 `LONG` 的替代方案。
:::

#### 可信标签

`TRUSTED_TAG` 和 `TRUSTED_COMPOUND_TAG` 是 `TAG`/`COMPOUND_TAG` 的变体，解除 2MiB 堆大小限制。理想情况下仅用于客户端发包，如原版处理[方块实体数据包][blockentity]和[实体数据序列化器][entity]。

需不同限制时，可通过 `ByteBufCodecs#tagCodec` 或 `#compoundTagCodec` 指定 `NbtAccounter` 大小。可选标签可通过 `#optionalTagCodec` 获取。

#### 宽松 JSON

`ByteBufCodecs#lenientJson` 处理任意 `JsonElement`，允许 `nan`/`infinite` 等浮点描述符或多顶层对象，需指定 JSON 最大尺寸。

### 原版与 NeoForge

Minecraft 和 NeoForge 为常用对象定义流编解码器，如 `ResourceLocation#STREAM_CODEC` 或 `NeoForgeStreamCodecs#CHUNK_POS`。大多数可在对象类本身、`StreamCodec`、`ByteBufCodecs` 或 `NeoForgeStreamCodecs` 中找到。

## 创建流编解码器

流编解码器可为任意对象创建，本文聚焦**缓冲区**(`buffer`)流处理。流编解码器有两个泛型：
- `B` 代表缓冲区类型（`ByteBuf`/`FriendlyByteBuf`/`RegistryFriendlyByteBuf`）
- `V` 代表对象值类型
- 构造时应使用最通用的缓冲区类型

### 成员编码器

`StreamMemberEncoder` 是 `StreamEncoder` 的替代方案，对象优先于缓冲区。通常用于对象含写入缓冲区的实例方法时，可通过 `StreamCodec#ofMember` 创建流编解码器。

```java
// 待创建流编解码器的对象
public class ExampleObject {
    
    // 常规构造器
    public ExampleObject(String arg1, int arg2, boolean arg3) { /* ... */ }

    // 流解码器引用
    public ExampleObject(ByteBuf buffer) { /* ... */ }

    // 流编码器引用
    public void encode(ByteBuf buffer) { /* ... */ }
}

// 流编解码器实现
public static StreamCodec<ByteBuf, ExampleObject> =
    StreamCodec.ofMember(ExampleObject::encode, ExampleObject::new);
```

### 复合类型

通过 `StreamCodec#composite` 创建复合流编解码器，按顺序定义字段的编解码器和**获取器**(`getter`)。支持最多 10 个参数。

```java
// 待创建流编解码器的对象
public record SimpleExample(String arg1, int arg2, boolean arg3) {}
public record RegistryExample(double arg1, Holder<Item> arg2) {}

// 流编解码器实现
public static final StreamCodec<ByteBuf, SimpleExample> SIMPLE_STREAM_CODEC =
    StreamCodec.composite(
        // 编解码器与获取器对
        ByteBufCodecs.STRING_UTF8, SimpleExample::arg1,
        ByteBufCodecs.VAR_INT, SimpleExample::arg2,
        ByteBufCodecs.BOOL, SimpleExample::arg3,
        SimpleExample::new
    );

// 含持有器的对象需使用 RegistryFriendlyByteBuf
public static final StreamCodec<RegistryFriendlyByteBuf, RegistryExample> REGISTRY_STREAM_CODEC =
    StreamCodec.composite(
        // 注意 ByteBuf 编解码器在此可用
        ByteBufCodecs.DOUBLE, RegistryExample::arg1,
        ByteBufCodecs.holderRegistry(Registries.ITEM), RegistryExample::arg2,
        RegistryExample::new
    );
```

### 转换器

流编解码器可通过映射方法转换：
- `map`：通过双向函数转换值类型
- `apply`：通过 `StreamCodec.CodecOperation` 转换值类型
- `mapStream`：通过函数转换缓冲区类型（极少使用）

```java
// map 示例
public static final StreamCodec<ByteBuf, ResourceLocation> STREAM_CODEC = 
    ByteBufCodecs.STRING_UTF8.map(
        // String -> ResourceLocation
        ResourceLocation::new,
        // ResourceLocation -> String
        ResourceLocation::toString
    );

// apply 示例
public static final StreamCodec<ByteBuf, List<ResourceLocation>> STREAM_CODEC =
    ResourceLocation.STREAM_CODEC.apply(ByteBufCodecs.list());

// mapStream 示例
public static final StreamCodec<RegistryFriendlyByteBuf, Integer> STREAM_CODEC =
    ByteBufCodecs.VAR_INT.mapStream(buffer -> (ByteBuf) buffer);
```

### 单位类型

无网络同步需求时，可用 `StreamCodec#unit` 表示固定值流编解码器。

:::warning
单位流编解码器要求编码对象必须匹配指定单位对象，否则报错。因此所有对象必须实现 `equals`，或始终提供相同实例。
:::

```java
public static final StreamCodec<ByteBuf, Item> UNIT_STREAM_CODEC =
    StreamCodec.unit(Items.AIR);
```

### 延迟初始化

依赖未初始化数据时，可通过 `NeoForgeStreamCodecs#lazy` 延迟初始化流编解码器。

```java
public static final StreamCodec<ByteBuf, Item> LAZY_STREAM_CODEC = 
    NeoForgeStreamCodecs.lazy(
        () -> StreamCodec.unit(Items.AIR)
    );
```

### 集合

通过对象流编解码器生成集合流编解码器：
- `collection`：接收构造空集合的 `IntFunction` 和最大尺寸（可选）
- `list`：列表专用方法

```java
// 集合示例
public static final StreamCodec<ByteBuf, Set<BlockPos>> COLLECTION_STREAM_CODEC =
    ByteBufCodecs.collection(
        HashSet::new,
        BlockPos.STREAM_CODEC,
        256 // 集合最多 256 个元素
    );

// apply 重载示例
public static final StreamCodec<ByteBuf, Set<BlockPos>> COLLECTION_STREAM_CODEC =
    BlockPos.STREAM_CODEC.apply(
        ByteBufCodecs.collection(HashSet::new)
    );

// 列表示例
public static final StreamCodec<ByteBuf, List<BlockPos>> LIST_STREAM_CODEC =
    BlockPos.STREAM_CODEC.apply(
        ByteBufCodecs.list(256) // 列表最多 256 个元素
    );
```

### 映射

通过键/值对象流编解码器生成映射流编解码器，支持最大尺寸（可选）。

```java
public static final StreamCodec<ByteBuf, Map<String, BlockPos>> MAP_STREAM_CODEC =
    ByteBufCodecs.map(
        HashMap::new,
        ByteBufCodecs.STRING_UTF8,
        BlockPos.STREAM_CODEC,
        256 // 映射最多 256 个元素
    );
```

### 二选一

通过两个流编解码器生成 `Either` 流编解码器，先读写布尔值指示选择。

```java
public static final StreamCodec<ByteBuf, Either<Integer, String>> EITHER_STREAM_CODEC = 
    ByteBufCodecs.either(
        ByteBufCodecs.VAR_INT,
        ByteBufCodecs.STRING_UTF8
    );
```

### ID 映射

跨网络同步对象时，通常发送代表 ID 的整数以减少数据量。枚举和**注册表**(`registries`)常用此方式。

```java
// 枚举示例
public enum ExampleIdObject {
    ;

    // 获取 ID -> 枚举的映射
    public static final IntFunction<ExampleIdObject> BY_ID = 
        ByIdMap.continuous(
            ExampleIdObject::getId,
            ExampleIdObject.values(),
            ByIdMap.OutOfBoundsStrategy.ZERO
    );
    
    ExampleIdObject(int id) { /* ... */ }
}

// 流编解码器实现
public static final StreamCodec<ByteBuf, ExampleIdObject> ID_STREAM_CODEC =
    ByteBufCodecs.idMapper(ExampleIdObject.BY_ID, ExampleIdObject::getId);
```

### 可选值

通过流编解码器生成 `Optional` 包装值，先读写布尔值指示是否存在。

```java
public static final StreamCodec<RegistryFriendlyByteBuf, Optional<DataComponentType<?>>> OPTIONAL_STREAM_CODEC =
    DataComponentType.STREAM_CODEC.apply(ByteBufCodecs::optional);
```

### 注册表对象

注册表对象可通过三种方法跨网络同步：`registry`、`holderRegistry` 或 `holder`。均需传入代表注册表的 `ResourceKey`。

:::warning
自定义注册表必须通过 `RegistryBuilder#sync` 启用同步，否则编码器报错。
:::

```java
// 注册表对象
public static final StreamCodec<RegistryFriendlyByteBuf, Item> VALUE_STREAM_CODEC =
    ByteBufCodecs.registry(Registries.ITEM);

// 持有器包装的注册表对象
public static final StreamCodec<RegistryFriendlyByteBuf, Holder<Item>> HOLDER_STREAM_CODEC =
    ByteBufCodecs.holderRegistry(Registries.ITEM);

// 直接引用持有器
public static final StreamCodec<RegistryFriendlyByteBuf, Holder<SoundEvent>> STREAM_CODEC =
    ByteBufCodecs.holder(
        Registries.SOUND_EVENT, SoundEvent.DIRECT_STREAM_CODEC
    );
```

### 持有器集合

通过 `holderSet` 同步标签或持有器集合。

```java
public static final StreamCodec<RegistryFriendlyByteBuf, HolderSet<Item>> HOLDER_SET_STREAM_CODEC =
    ByteBufCodecs.holderSet(Registries.ITEM);
```

### 递归类型

对象字段引用同类对象时（如 `MobEffectInstance` 含隐藏效果），可用 `StreamCodec#recursive` 递归创建流编解码器。

```java
// 递归对象定义
public record RecursiveObject(Optional<RecursiveObject> inner) { /* ... */ }

public static final StreamCodec<ByteBuf, RecursiveObject> RECURSIVE_CODEC = StreamCodec.recursive(
    recursedStreamCodec -> StreamCodec.composite(
        recursedStreamCodec.apply(ByteBufCodecs::optional),
        RecursiveObject::inner,
        RecursiveObject::new
    )
);
```

### 分发类型

通过 `StreamCodec#dispatch` 实现基于类型的分流编解码，常用于 `ParticleType` 等代表类型的注册表对象。

```java
// 对象定义
public abstract class ExampleObject {
    // 指定对象类型以编码
    public abstract StreamCodec<? super RegistryFriendlyByteBuf, ? extends ExampleObject> streamCodec();
}

// 假设存在 ResourceKey<StreamCodec<? super RegistryFriendlyByteBuf, ? extends ExampleObject>> DISPATCH
public static final StreamCodec<RegistryFriendlyByteBuf, ExampleObject> DISPATCH_STREAM_CODEC =
    ByteBufCodecs.registry(DISPATCH).dispatch(
        // 从具体对象获取流编解码器
        ExampleObject::streamCodec,
        // 从注册表对象获取流编解码器
        Function.identity()
    );
```

[networking]: payload.md
[codecs]: ../datastorage/codecs.md
[blockentity]: ../blockentities/index.md#syncing-on-block-update
[entity]: ../entities/data.md
[transformers]: ../datastorage/codecs.md#transformers