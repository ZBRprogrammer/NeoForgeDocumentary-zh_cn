---
sidebar_position: 2
---
# **编解码器**(`Codecs`)

编解码器是 Mojang [**DataFixerUpper**(`DataFixerUpper`)][DataFixerUpper] 中的序列化工具，用于描述对象如何在不同格式之间转换，例如用于 JSON 的 `JsonElement` 和用于 NBT 的 `Tag`。

## 使用编解码器

编解码器主要用于将 Java 对象**编码**(`encode`)或序列化为某种数据格式类型，以及将格式化数据对象**解码**(`decode`)或反序列化回其关联的 Java 类型。这通常分别通过 `Codec#encodeStart` 和 `Codec#parse` 实现。

### **动态操作**(`DynamicOps`)

要确定编码和解码使用的中间文件格式，`#encodeStart` 和 `#parse` 都需要一个 `DynamicOps` 实例来定义该格式中的数据。

[**DataFixerUpper**(`DataFixerUpper`)][DataFixerUpper] 库包含 `JsonOps`，用于编解码存储在 [`Gson`][gson] 的 `JsonElement` 实例中的 JSON 数据。`JsonOps` 支持两种 `JsonElement` 序列化版本：`JsonOps#INSTANCE` 定义标准 JSON 文件，`JsonOps#COMPRESSED` 允许将数据压缩为单个字符串。

```java
// Let exampleCodec represent a Codec<ExampleJavaObject>
// Let exampleObject be a ExampleJavaObject
// Let exampleJson be a JsonElement

// Encode Java object to regular JsonElement
exampleCodec.encodeStart(JsonOps.INSTANCE, exampleObject);

// Encode Java object to compressed JsonElement
exampleCodec.encodeStart(JsonOps.COMPRESSED, exampleObject);

// Decode JsonElement into Java object
// Assume JsonElement was parsed normally
exampleCodec.parse(JsonOps.INSTANCE, exampleJson);
```

Minecraft 还提供 `NbtOps` 来编解码存储在 `Tag` 实例中的 NBT 数据，可通过 `NbtOps#INSTANCE` 引用。

```java
// Let exampleCodec represent a Codec<ExampleJavaObject>
// Let exampleObject be a ExampleJavaObject
// Let exampleNbt be a Tag

// Encode Java object to Tag
exampleCodec.encodeStart(NbtOps.INSTANCE, exampleObject);

// Decode Tag into Java object
exampleCodec.parse(NbtOps.INSTANCE, exampleNbt);
```

为处理注册表条目，Minecraft 提供 `RegistryOps`，它包含一个查找提供器(`lookup provider`)以获取可用注册表元素。可通过 `RegistryOps#create` 创建，该方法接收存储数据的特定类型的 `DynamicOps` 和包含可用注册表访问权限的查找提供器。NeoForge 扩展了 `RegistryOps` 以创建 `ConditionalOps`：一种可处理[条目加载条件(`conditions`)][conditions]的注册表编解码器查找器。

```java
// Let lookupProvider be a HolderLookup.Provider
// Let exampleCodec represent a Codec<ExampleJavaObject>
// Let exampleObject be a ExampleJavaObject
// Let exampleJson be a JsonElement

// Get the registry ops for JsonElement
RegistryOps<JsonElement> ops = RegistryOps.create(JsonOps.INSTANCE, lookupProvider);

// Encode Java object to JsonElement
exampleCodec.encodeStart(ops, exampleObject);

// Decode JsonElement into Java object
exampleCodec.parse(ops, exampleJson);
```

#### 格式转换

`DynamicOps` 还可单独用于在两种不同编码格式之间转换，通过 `#convertTo` 并提供目标 `DynamicOps` 格式及待转换的编码对象实现。

```java
// Convert Tag to JsonElement
// Let exampleTag be a Tag
JsonElement convertedJson = NbtOps.INSTANCE.convertTo(JsonOps.INSTANCE, exampleTag);
```

### **数据结果**(`DataResult`)

使用编解码器编码或解码的数据会返回 `DataResult`，其中包含转换后的实例或错误数据（取决于转换是否成功）。转换成功时，`#result` 提供的 `Optional` 将包含成功转换的对象。转换失败时，`#error` 提供的 `Optional` 将包含 `PartialResult`（其中包含错误消息和根据编解码器得到的部分转换对象）。

此外，`DataResult` 上有许多方法可用于将结果或错误转换为所需格式。例如，`#resultOrPartial` 在成功时返回包含结果的 `Optional`，失败时返回部分转换的对象。该方法接收字符串消费者(`string consumer`)以确定如何报告错误消息（如果存在）。

```java
// Let exampleCodec represent a Codec<ExampleJavaObject>
// Let exampleJson be a JsonElement

// Decode JsonElement into Java object
DataResult<ExampleJavaObject> result = exampleCodec.parse(JsonOps.INSTANCE, exampleJson);

result
    // Get result or partial on error, report error message
    .resultOrPartial(errorMessage -> /* Do something with error message */)
    // If result or partial is present, do something
    .ifPresent(decodedObject -> /* Do something with decoded object */);
```

## 现有编解码器

### 基础类型

`Codec` 类包含某些已定义基础类型的编解码器静态实例。

编解码器         | Java 类型
:---:         | :---
`BOOL`        | `Boolean`
`BYTE`        | `Byte`
`SHORT`       | `Short`
`INT`         | `Integer`
`LONG`        | `Long`
`FLOAT`       | `Float`
`DOUBLE`      | `Double`
`STRING`      | `String`\*
`BYTE_BUFFER` | `ByteBuffer`
`INT_STREAM`  | `IntStream`
`LONG_STREAM` | `LongStream`
`PASSTHROUGH` | `Dynamic<?>`\*\*
`EMPTY`       | `Unit`\*\*\*

\* 可通过 `Codec#string` 或 `Codec#sizeLimitedString` 将 `String` 限制在特定字符数内。

\*\* `Dynamic` 是持有以支持的 `DynamicOps` 格式编码值的对象，通常用于将编码对象格式转换为其他编码对象格式。

\*\*\* `Unit` 是用于表示 `null` 对象的对象。

### Vanilla 和 NeoForge

Minecraft 和 NeoForge 为常需编解码的对象定义了许多编解码器。例如 `ResourceLocation#CODEC`（用于 `ResourceLocation`）、`ExtraCodecs#INSTANT_ISO8601`（用于 `DateTimeFormatter#ISO_INSTANT` 格式的 `Instant`）和 `CompoundTag#CODEC`（用于 `CompoundTag`）。

:::caution
`CompoundTag` 无法使用 `JsonOps` 从 JSON 解码数字列表。`JsonOps` 在转换时将数字设为其最窄类型，而 `ListTag` 强制其数据为特定类型，因此不同类型数字（如 `64` 为 `byte`，`384` 为 `short`）会在转换时抛出错误。
:::

Vanilla 和 NeoForge 注册表也包含其注册对象类型的编解码器（例如 `BuiltInRegistries#BLOCK` 有 `Codec<Block>`）。`Registry#byNameCodec` 将注册表对象编码为其注册名称。Vanilla 注册表还有 `Registry#holderByNameCodec`，可将对象编码为注册名称并解码为包装在 `Holder` 中的注册表对象。

## 创建编解码器

可为任何对象创建编解码器以进行编码和解码。为便于理解，将展示等效的编码 JSON。

### **记录**(`Records`)

编解码器可通过记录定义对象。每个记录编解码器定义具有显式命名字段的任何对象。创建记录编解码器的方法有多种，最简单的是通过 `RecordCodecBuilder#create`。

`RecordCodecBuilder#create` 接收一个函数，该函数定义 `Instance` 并返回对象的应用（`App`）。可类比创建类*实例*(`instance`)和用于将类*应用*(`apply`)到构造对象的构造函数。

```java
// Some object to create a codec for
public class SomeObject {

    public SomeObject(String s, int i, boolean b) { /* ... */ }

    public String s() { /* ... */ }

    public int i() { /* ... */ }

    public boolean b() { /* ... */ }
}
```

#### **字段**(`Fields`)

`Instance` 可通过 `#group` 定义最多 16 个字段。每个字段必须是一个应用，定义对象所属实例及其类型。满足此要求的最简方法是通过获取 `Codec`，设置待解码字段的名称，并设置用于编码字段的**获取器**(`getter`)。

可通过 `#fieldOf`（若字段必需）或 `#optionalFieldOf`（若字段包装在 `Optional` 中或具有默认值）从 `Codec` 创建字段。任一方法都需要一个字符串，其中包含编码对象中的字段名称。然后可使用 `#forGetter` 设置用于编码字段的获取器，接收一个函数（给定对象，返回字段数据）。

:::warning
如果存在解析时抛出错误的元素，`#optionalFieldOf` 会抛出错误。若应消耗错误，请改用 `#lenientOptionalFieldOf`。
:::

之后，可通过 `#apply` 应用结果产物以定义实例应如何为应用构造对象。为方便起见，分组字段应按其在构造函数中出现的顺序列出，以便该函数可直接作为构造函数方法引用。

```java
public static final Codec<SomeObject> RECORD_CODEC = RecordCodecBuilder.create(instance -> // Given an instance
    instance.group( // Define the fields within the instance
        Codec.STRING.fieldOf("s").forGetter(SomeObject::s), // String
        Codec.INT.optionalFieldOf("i", 0).forGetter(SomeObject::i), // Integer, defaults to 0 if field not present
        Codec.BOOL.fieldOf("b").forGetter(SomeObject::b) // Boolean
    ).apply(instance, SomeObject::new) // Define how to create the object
);
```

```json5
// Encoded SomeObject
{
    "s": "value",
    "i": 5,
    "b": false
}

// Another encoded SomeObject
{
    "s": "value2",
    // i is omitted, defaults to 0
    "b": true
}

// Another encoded SomeObject
{
    "s": "value2",
    // Will throw an error as lenientOptionalFieldOf is not used
    "i": "bad_value",
    "b": true
}
```

### **转换器**(`Transformers`)

编解码器可通过映射方法转换为等效或部分等效的表示形式。每个映射方法接收两个函数：一个将当前类型转换为新类型，另一个将新类型转换回当前类型。这通过 `#xmap` 函数实现。

```java
// A class
public class ClassA {

    public ClassB toB() { /* ... */ }
}

// Another equivalent class
public class ClassB {

    public ClassA toA() { /* ... */ }
}

// Assume there is some codec A_CODEC
public static final Codec<ClassB> B_CODEC = A_CODEC.xmap(ClassA::toB, ClassB::toA);
```

若类型部分等效（即转换期间存在某些限制），可使用返回 `DataResult` 的映射函数，以便在遇到异常或无效状态时返回错误状态。

A 是否完全等效于 B | B 是否完全等效于 A | 转换方法
:---:                      | :---:                      | :---
是                        | 是                        | `#xmap`
是                        | 否                         | `#flatComapMap`
否                         | 是                        | `#comapFlatMap`
否                         | 否                         | `#flatXMap`

```java
// Given an string codec to convert to a integer
// Not all strings can become integers (A is not fully equivalent to B)
// All integers can become strings (B is fully equivalent to A)
public static final Codec<Integer> INT_CODEC = Codec.STRING.comapFlatMap(
    s -> { // Return data result containing error on failure
        try {
            return DataResult.success(Integer.valueOf(s));
        } catch (NumberFormatException e) {
            return DataResult.error(s + " is not an integer.");
        }
    },
    Integer::toString // Regular function
);
```

```json5
// Will return 5
"5"

// Will error, not an integer
"value"
```

#### **范围编解码器**(`Range Codecs`)

范围编解码器是 `#flatXMap` 的实现，若值不在设定的最小值和最大值之间（含边界），则返回错误 `DataResult`。若超出边界，仍将值作为部分结果提供。可通过 `#intRange`、`#floatRange` 和 `#doubleRange` 分别实现整数、浮点数和双精度数的范围编解码器。

```java
public static final Codec<Integer> RANGE_CODEC = Codec.intRange(0, 4); 
```

```json5
// Will be valid, inside [0, 4]
4

// Will error, outside [0, 4]
5
```

#### **字符串解析器**(`String Resolver`)

`Codec#stringResolver` 是 `flatXmap` 的实现，将字符串映射到某种对象。

```java
public record StringResolverObject(String name) { /* ... */ }

// Assume there is some Map<String, StringResolverObject> OBJECT_MAP
public static final Codec<StringResolverObject> STRING_RESOLVER_CODEC = Codec.stringResolver(StringResolverObject::name, OBJECT_MAP::get);
```

```json5
// Will map this string to its associated object
"example_name"
```

### **默认值**(`Defaults`)

若编码或解码失败，可通过 `Codec#orElse` 或 `Codec#orElseGet` 提供默认值。

```java
public static final Codec<Integer> DEFAULT_CODEC = Codec.INT.orElse(
    errorMessage -> /* Do something with the error message */,
    0 // Can also be a supplied value via #orElseGet
); 
```

```json5
// Not an integer, defaults to 0
"value"
```

### **单位**(`Unit`)

可使用 `Codec#unit` 表示提供代码内值且编码为空内容的编解码器，这在编解码器使用数据对象中的非可编码条目时很有用。

```java
public static final Codec<IEventBus> UNIT_CODEC = Codec.unit(
    () -> NeoForge.EVENT_BUS // Can also be a raw value
);
```

```json5
// Nothing here, will return the NeoForge event bus
```

### **延迟初始化**(`Lazy Initialized`)

有时编解码器可能依赖构造时不存在的数据。此时可使用 `Codec#lazyInitialized` 让编解码器在首次编码/解码时构造自身。该方法接收一个提供的编解码器。

```java
public static final Codec<IEventBus> LAZY_CODEC = Codec.lazyInitialized(
    () -> Codec.Unit(NeoForge.EVENT_BUS)
);
```

```json5
// Nothing here, will return the NeoForge event bus
// Encodes/decodes the same way as the normal codec
```

### **列表**(`List`)

可通过对象编解码器使用 `Codec#listOf` 生成对象列表的编解码器。`listOf` 也可接收表示列表最小和最大长度的整数。`sizeLimitedListOf` 功能相同但仅指定最大边界。

```java
// BlockPos#CODEC is a Codec<BlockPos>
public static final Codec<List<BlockPos>> LIST_CODEC = BlockPos.CODEC.listOf();
```

```json5
// Encoded List<BlockPos>
[
    [1, 2, 3], // BlockPos(1, 2, 3)
    [4, 5, 6], // BlockPos(4, 5, 6)
    [7, 8, 9]  // BlockPos(7, 8, 9)
]
```

使用列表编解码器解码的列表对象存储在**不可变**列表中。若需要可变列表，应对列表编解码器应用[转换器(`transformer`)][transformer]。

### **映射**(`Map`)

可通过两个编解码器使用 `Codec#unboundedMap` 生成键值对象映射的编解码器。无界映射可指定任何基于字符串或经字符串转换的值作为键。

```java
// BlockPos#CODEC is a Codec<BlockPos>
public static final Codec<Map<String, BlockPos>> MAP_CODEC = Codec.unboundedMap(Codec.STRING, BlockPos.CODEC);
```

```json5
// Encoded Map<String, BlockPos>
{
    "key1": [1, 2, 3], // key1 -> BlockPos(1, 2, 3)
    "key2": [4, 5, 6], // key2 -> BlockPos(4, 5, 6)
    "key3": [7, 8, 9]  // key3 -> BlockPos(7, 8, 9)
}
```

使用无界映射编解码器解码的映射对象存储在**不可变**映射中。若需要可变映射，应对映射编解码器应用[转换器(`transformer`)][transformer]。

:::caution
无界映射仅支持编码/解码为字符串的键。可通过键值[对(`pair`)][pair]列表编解码器绕过此限制。
:::

### **对**(`Pair`)

可通过两个编解码器使用 `Codec#pair` 生成对象对的编解码器。

对编解码器通过首先解码对中的左侧对象，然后获取编码对象的剩余部分并从中解码右侧对象来解码对象。因此，编解码器必须在解码后表达编码对象的某些内容（如[记录(`records`)][records]），或需增强为 `MapCodec` 并通过 `#codec` 转换为常规编解码器。通常可通过使编解码器成为某个对象的[字段(`field`)][field]实现。

```java
public static final Codec<Pair<Integer, String>> PAIR_CODEC = Codec.pair(
    Codec.INT.fieldOf("left").codec(),
    Codec.STRING.fieldOf("right").codec()
);
```

```json5
// Encoded Pair<Integer, String>
{
    "left": 5,       // fieldOf looks up 'left' key for left object
    "right": "value" // fieldOf looks up 'right' key for right object
}
```

:::tip
可使用键值对列表应用[转换器(`transformer`)][transformer]来编码/解码具有非字符串键的映射编解码器。
:::

### **任一**(`Either`)

可通过两个编解码器使用 `Codec#either` 生成两种不同方法编码/解码对象数据的编解码器。

任一编解码器首先尝试使用第一个编解码器解码对象。若失败，则尝试使用第二个编解码器解码。若两者均失败，则 `DataResult` 将仅包含第二次失败的错误。

```java
public static final Codec<Either<Integer, String>> EITHER_CODEC = Codec.either(
    Codec.INT,
    Codec.STRING
);
```

```json5
// Encoded Either.Left<Integer, String>
5

// Encoded Either.Right<Integer, String>
"value"
```

:::tip
可结合[转换器(`transformer`)][transformer]使用，以通过两种不同编码方法获取特定对象。
:::

#### **异或**(`Xor`)

`Codec#xor` 是[任一(`either`)][either]编解码器的特例，仅当两种方法之一成功处理时才返回成功结果。若两者均可处理，则抛出错误。

```java
public static final Codec<Either<Integer, String>> XOR_CODEC = Codec.xor(
    Codec.INT.fieldOf("number").codec(),
    Codec.STRING.fieldOf("text").codec()
);
```

```json5
// Encoded Either.Left<Integer, String>
{
    "number": 4
}

// Encoded Either.Right<Integer, String>
{
    "text": "value"
}

// Throws an error as both can be decoded
{
    "number": 4,
    "text": "value"
}
```

#### **替代**(`Alternative`)

`Codec#withAlternative` 是[任一(`either`)][either]编解码器的特例，两者均尝试解码同一对象但存储格式不同。首先尝试主编解码器解码对象，失败时使用第二个编解码器。编码始终使用主编解码器。

```java
public static final Codec<BlockPos> ALTERNATIVE_CODEC = Codec.withAlternative(
    BlockPos.CODEC,
    RecordCodecBuilder.create(instance -> instance.group(
        Codec.INT.fieldOf("x").forGetter(BlockPos::getX),
        Codec.INT.fieldOf("y").forGetter(BlockPos::getY),
        Codec.INT.fieldOf("z").forGetter(BlockPos::getZ)
    ), BlockPos::new)
);
```

```json5
// Normal method to decode BlockPos
[ 1, 2, 3 ]

// Alternative method to decode BlockPos
{
    "x": 1,
    "y": 2,
    "z": 3
}
```

### **递归**(`Recursive`)

有时对象可能将同类型对象的引用作为字段。例如 `EntityPredicate` 为载具、乘客和目标实体接收 `EntityPredicate`。此时可使用 `Codec#recursive` 通过函数提供编解码器以创建编解码器。

```java
// Define our recursive object
public record RecursiveObject(Optional<RecursiveObject> inner) { /* ... */ }

public static final Codec<RecursiveObject> RECURSIVE_CODEC = Codec.recursive(
    RecursiveObject.class.getSimpleName(), // This is for the toString method
    recursedCodec -> RecordCodecBuilder.create(instance -> instance.group(
        recursedCodec.optionalFieldOf("inner").forGetter(RecursiveObject::inner)
    ).apply(instance, RecursiveObject::new))
);
```

```json5
// An encoded recursive object
{
    "inner": {
        "inner": {}
    }
}
```

### **分发**(`Dispatch`)

编解码器可包含**子编解码器**(`subcodecs`)，通过 `Codec#dispatch` 基于指定类型解码特定对象。通常用于包含编解码器的注册表（如规则测试或方块放置器）。

分发编解码器首先尝试从字符串键（通常为 `type`）获取编码类型。之后解码该类型，调用用于解码实际对象的特定编解码器的获取器。若用于解码对象的 `DynamicOps` 压缩其映射，或对象编解码器本身未增强为 `MapCodec`（如记录或带字段的基础类型），则对象需存储在 `value` 键中。否则，对象在与数据其余部分相同的层级解码。

```java
// Define our object
public abstract class ExampleObject {

    // Define the method used to specify the object type for encoding
    public abstract MapCodec<? extends ExampleObject> type();
}

// Create simple object which stores a string
public class StringObject extends ExampleObject {

    public StringObject(String s) { /* ... */ }

    public String s() { /* ... */ }

    public MapCodec<? extends ExampleObject> type() {
        // A registered registry object
        // "string":
        //   Codec.STRING.xmap(StringObject::new, StringObject::s).fieldOf("string")
        return STRING_OBJECT_CODEC.get();
    }
}

// Create complex object which stores a string and integer
public class ComplexObject extends ExampleObject {

    public ComplexObject(String s, int i) { /* ... */ }

    public String s() { /* ... */ }

    public int i() { /* ... */ }

    public MapCodec<? extends ExampleObject> type() {
        // A registered registry object
        // "complex":
        //   RecordCodecBuilder.mapCodec(instance ->
        //     instance.group(
        //       Codec.STRING.fieldOf("s").forGetter(ComplexObject::s),
        //       Codec.INT.fieldOf("i").forGetter(ComplexObject::i)
        //     ).apply(instance, ComplexObject::new)
        //   )
        return COMPLEX_OBJECT_CODEC.get();
    }
}

// Assume there is an Registry<MapCodec<? extends ExampleObject>> DISPATCH
public static final Codec<ExampleObject> = DISPATCH.byNameCodec() // Gets Codec<MapCodec<? extends ExampleObject>>
    .dispatch(
        ExampleObject::type, // Get the codec from the specific object
        Function.identity() // Get the codec from the registry
    );
```

```json5
// Simple object
{
    "type": "string", // For StringObject
    "value": "value" // Codec type is not augmented from MapCodec, needs field
}

// Complex object
{
    "type": "complex", // For ComplexObject

    // Codec type is augmented from MapCodec, can be inlined
    "s": "value",
    "i": 0
}
```

[DataFixerUpper]: https://github.com/Mojang/DataFixerUpper
[gson]: https://github.com/google/gson
[conditions]: ../resources/server/conditions.md
[transformer]: #transformer-codecs
[pair]: #pair
[records]: #records
[field]: #fields
[either]: #either