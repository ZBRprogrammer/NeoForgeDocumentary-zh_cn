---
sidebar_position: 3
---
# **值输入输出**(`Value I/O`)

**值输入输出**(`Value I/O`)系统是标准化序列化方法，用于操作某些底层对象（如 NBT 的[**复合标签**](`CompoundTag`)）的数据。

## 输入与输出

值 I/O 系统由两部分组成：`ValueOutput`（序列化期间写入对象）和 `ValueInput`（反序列化期间从对象读取）。实现方法通常以 `ValueOutput` 或 `ValueInput` 为唯一参数，无返回值。值 I/O 期望底层对象是字符串键到对象值的字典，通过提供的方法读写底层对象数据。

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    // Write data to the output
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);
    // Read data from the input
}

// For some Entity subclass
@Override
protected void addAdditionalSaveData(ValueOutput output) {
    super.addAdditionalSaveData(output);
    // Write data to the output
}

@Override
protected void readAdditionalSaveData(ValueInput input) {
    super.readAdditionalSaveData(input);
    // Read data from the input
}
```

### 基础类型

值 I/O 包含读写基础类型的方法。`ValueOutput` 方法以 `put*` 为前缀，接收键和基础类型值。`ValueInput` 方法命名为 `get*Or`，接收键和默认值（若不存在）。

| Java 类型 | `ValueOutput` | `ValueInput`                 |
|:---------:|:-------------:|:----------------------------:|
| `boolean` | `putBoolean`  | `getBooleanOr`               |
| `byte`    | `putByte`     | `getByteOr`                  |
| `short`   | `putShort`    | `getShortOr`                 |
| `int`     | `putInt`      | `getInt`\*, `getIntOr`       |
| `long`    | `putLong`     | `getLong`\*, `getLongOr`     |
| `float`   | `putFloat`    | `getFloatOr`                 |
| `double`  | `putDouble`   | `getDoubleOr`                |
| `String`  | `putString`   | `getString`\*, `getStringOr` |
| `int[]`   | `putIntArray` | `getIntArray`\*              |

\* 这些 `ValueInput` 方法返回 `Optional` 包装的基础类型（而非接收并返回回退值）。

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    
    // Write data to the output
    output.putBoolean(
        // The string key
        "boolValue",
        // The value associated with this key
        true
    );
    output.putString("stringValue", "Hello world!");
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // Read data from the input

    // Defaults to false if not present
    boolean boolValue = input.getBooleanOr(
        // The string key to retrieve
        "boolValue",
        // The default value to return if the key is not present
        false
    );

    // Defaults to 'Dummy!' if not present
    String stringValue = input.getStringOr("stringValue", "Dummy!");
    // Returns an optional-wrapped value
    Optional<String> stringValueOpt = input.getString("stringValue");
}
```

### **编解码器**(`Codecs`)

[**编解码器**](`Codec`)也可用于在值 I/O 中存储和读取值。在原版中，所有 `Codec` 均通过 `RegistryOps` 处理（支持存储数据包条目）。`ValueOutput#store` 和 `storeNullable` 接收键、写入对象的编解码器及对象本身。`storeNullable` 在对象为 `null` 时不写入任何内容。`ValueInput#read` 可通过接收键和编解码器读取对象，返回 `Optional` 包装的对象。

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    
    // Write data to the output
    output.storeNullable("codecValue", Rarity.CODEC, Rarity.EPIC);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // Read data from the input
    Optional<Rarity> codecValue = input.read("codecValue", Rarity.CODEC);
}
```

`ValueOutput` 和 `ValueInput` 还提供 `MapCodec` 的 `store`/`read` 方法。相比 `Codec`，`MapCodec` 变体将值合并到当前根节点。

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    
    // Write data to the output
    output.store(
        SingleFile.MAP_CODEC,
        new SingleFile(ResourceLocation.fromNamespaceAndPath("examplemod", "example"))
    );
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // Read data from the input

    // No key is needed as they are stored on the root value access
    Optional<SingleFile> file = input.read(SingleFile.MAP_CODEC);
    // This is present as `SingleFile` writes the `resource` parameter
    String resource = input.getStringOr("resource", "Not present!");
}
```

:::warning
`MapCodec` 会将任意键写入值访问器，可能覆盖现有数据。请确保 `MapCodec` 中的键与其他键不同。
:::

### **列表**(`Lists`)

可通过两种方法创建和读取列表：子值 I/O 或[**编解码器**](`Codec`)。

通过 `ValueOutput#childrenList` 创建列表（接收键）。返回 `ValueOutput.ValueOutputList`（作为值对象的只写列表）。通过 `ValueOutputList#addChild` 添加新值对象到列表，返回 `ValueOutput` 以写入值对象数据。然后可通过 `ValueInput#childrenList` 或 `childrenListOrEmpty`（不存在时默认为空列表）读取列表。这些方法返回 `ValueInput.ValueInputList`（作为只读可迭代对象或流，通过 `stream` 访问）。

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    
    // Write data to the output

    // Create List
    ValueOutput.ValueOutputList listValue = output.childrenList("listValue");
    // Add elements
    ValueOutput childIdx0 = listValue.addChild();
    childIdx0.putBoolean("boolChild", false);
    ValueOutput childIdx1 = listValue.addChild();
    childIdx1.putInt("boolChild", true);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // Read data from the input

    // Read values of list
    for (ValueInput childInput : input.childrenListOrEmpty("listValue")) {
        boolean boolChild = childInput.getBooleanOr("boolChild", false);
    }
}
```

编解码器通过 `ValueOutput#list` 提供数据对象的列表变体（接收键和 `Codec`），返回 `ValueOutput.TypedOutputList`。`TypedOutputList` 与 `ValueOutputList` 相同，但操作数据对象而非使用另一个值 I/O。通过 `TypedOutputList#add` 添加元素到列表。类似地，可通过 `ValueInput#list` 或 `listOrEmpty` 读取列表（返回 `TypedValueInput`）。

:::note
`TypedValueOutput`/`TypedValueInput` 与 `Codec#listOf` 的主要区别在于错误处理方式。`Codec#listOf` 中单个条目失败会导致整个对象标记为错误 `DataResult`。而类型化值 I/O 通常通过 `ProblemReporter` 处理错误。原版中 `Codec#listOf` 提供更大灵活性（因 `ProblemReporter` 在创建值 I/O 时指定）。但自定义值 I/O 用法可根据用例实现任一方式。
:::

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    
    // Write data to the output

    // Create List
    ValueOutput.TypedInputList<Rarity> listValue = output.list("listValue", Rarity.CODEC);
    // Add elements
    listValue.add(Rarity.COMMON);
    listValue.add(Rarity.EPIC);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // Read data from the input

    // Read values of list
    for (Rarity rarity : input.listOrEmpty("listValue", Rarity.CODEC)) {
        // ...
    }
}
```

:::warning
即使列表为空仍会写入 `ValueOutput`。若不想写入列表，`TypedOutputList` 或 `ValueOutputList` 应检查是否 `isEmpty`，然后用列表键调用 `discard`。

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    
    // Write data to the output

    // Create List
    ValueOutput.TypedInputList<Rarity> listValue = output.list("listValue", Rarity.CODEC);
    
    // Check if list is empty
    if (listValue.isEmpty()) {
        // Discard from output
        output.discard("listValue");
    }
}
```
:::

### **对象**(`Objects`)

可通过子节点创建和读取对象。`ValueOutput#child` 给定键创建新 `ValueObject`。然后可通过 `ValueInput#child` 或 `childOrEmpty`（应默认为带空底层值的 `ValueInput`）读取对象。

```java
// For some BlockEntity subclass
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    
    // Write data to the output

    // Create object
    ValueOutput objectValue = output.child("objectValue");
    // Add data to object
    objectValue.putBoolean("boolChild", true);
    objectValue.putInt("intChild", 20);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // Read data from the input

    // Read object
    ValueInput objectValue = input.childOrEmpty("objectValue");
    // Get data from object
    boolean boolChild = objectValue.getBooleanOr("boolChild", false);
    int intChild = objectValue.getIntOr("intChild", 0);
}
```

## ValueIOSerializable

`ValueIOSerializable` 是 NeoForge 新增接口，用于可通过值 I/O 序列化和反序列化的对象。NeoForge 使用此 API 处理[**数据附件**](`data attachments`)。接口提供两个方法：`serialize`（将对象写入 `ValueOutput`）和 `deserialize`（从 `ValueInput` 读取对象）。

```java
public class ExampleObject implements ValueIOSerializable {
    
    @Override
    public void serialize(ValueOutput output) {
        // Write the object data here
    }

    @Override
    public void deserialize(ValueInput input) {
        // Read the object data here
    }
}
```

## 实现

### NBT

[NBT](`nbt`) 的值 I/O 通过 `TagValueOutput` 和 `TagValueInput` 处理。

`TagValueOutput` 可通过 `createWithContext` 或 `createWithoutContext` 创建。`createWithContext` 表示输出可访问 `HolderLookup.Provider`（提供所有注册表条目，包括静态和数据包条目），而 `createWithoutContext` 不提供任何数据包访问。原版仅使用 `createWithContext`。使用 `ValueOutput` 后，可通过 `TagValueOutput#buildResult` 获取 `CompoundTag`。`TagValueInput` 则通过 `create` 创建（接收 `HolderLookup.Provider` 和输入访问的 `CompoundTag`）。

两种值 I/O 均接收 `ProblemReporter`（用于收集读写过程中的所有内部错误）。当前仅跟踪 `Codec` 错误，错误处理方式由模组开发者决定。原版实现在 `ProblemReporter` 非空时抛出异常。

```java
// Assume we have access to a HolderLookup.Provider lookupProvider

TagValueOutput output = TagValueOutput.createWithContext(
    ProblemReporter.DISCARDING, // Choose to discard all errors
    lookupProvider
);

// Write to the output...

CompoundTag tag = output.buildResult();

// Collect the errors
ProblemReporter.Collector reporter = new ProblemReporter.Collector(
    // Optionally takes in the root path element
    // Some objects (e.g., block entities, entities) have a #problemPath() method that can be supplied
    new RootFieldPathElement("example_object")
);

TagValueInput input = TagValueInput.create(
    reporter,
    lookupProvider,
    tag
);

// Read from the input...
```

[attachments]: attachments.md
[codec]: codecs.md
[nbt]: nbt.md