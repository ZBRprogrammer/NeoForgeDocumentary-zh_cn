---
sidebar_position: 1
---
# **命名二进制标签**(`Named Binary Tag, NBT`)

NBT 是 Minecraft 早期由 Notch 本人引入的格式，广泛用于整个 Minecraft 代码库的数据存储。

## 规范

NBT 规范与 JSON 规范相似，但存在以下差异：

- 存在字节(`byte`)、短整数(`short`)、长整数(`long`)和浮点数(`float`)的独立类型，分别以 `b`、`s`、`l` 和 `f` 为后缀（类似 Java 代码表示）。
    - 双精度数(`double`)也可添加 `d` 后缀（非必需），类似 Java 代码。但 Java 中整数可选的 `i` 后缀在 NBT 中不允许。
    - 后缀不区分大小写。例如 `64b` 等同于 `64B`，`0.5F` 等同于 `0.5f`。
- 布尔值(`boolean`)不存在，由字节(`byte`)表示：`true` 表示为 `1b`，`false` 表示为 `0b`。
    - 当前实现将所有非零值视为 `true`，因此 `2b` 也会被视为 `true`。
- NBT 中没有 `null` 的等价物。
- 键(`key`)周围的引号可选。因此 JSON 属性 `"duration": 20` 在 NBT 中可表示为 `duration: 20` 或 `"duration": 20`。
- JSON 中的子对象在 NBT 中称为**复合标签**(`compound tag`)（或简称复合体）。
- NBT 列表(`list`)不能混合类型（与 JSON 不同）。列表类型由首元素决定或在代码中定义。
    - 但列表的列表(`lists of lists`)可混合不同类型。例如第一个是字符串列表，第二个是字节列表的双层列表是允许的。
- 存在特殊的**数组**(`array`)类型（不同于列表），同样使用方括号包含元素，共三种数组类型：
    - **字节数组**(`byte arrays`)：以 `B;` 开头。例如：`[B;0b,30b]`
    - **整数数组**(`integer arrays`)：以 `I;` 开头。例如：`[I;0,-300]`
    - **长整数数组**(`long arrays`)：以 `L;` 开头。例如：`[L;0l,240l]`
- 列表、数组和复合标签中允许尾随逗号。

## NBT 文件

Minecraft 广泛使用 `.nbt` 文件，例如[数据包(`datapack`)][datapack]中的结构文件。包含区域（即区块集合）内容的区域文件（`.mca`）以及游戏中各处使用的各种 `.dat` 文件也是 NBT 文件。

NBT 文件通常使用 GZip 压缩。因此它们是二进制文件，无法直接编辑。

## 代码中的 NBT

与 JSON 类似，所有 NBT 对象都是封闭对象的子对象。创建一个复合标签：

```java
CompoundTag tag = new CompoundTag();
```

向标签添加数据：

```java
tag.putInt("Color", 0xffffff);
tag.putString("Level", "minecraft:overworld");
tag.putDouble("IAmRunningOutOfIdeasForNamesHere", 1d);
```

此处存在多个辅助方法，例如 `putIntArray` 除标准 `int[]` 版本外，还有接收 `List<Integer>` 的便捷版本。

当然，也可从标签获取值：

```java
Optional<Integer> color = tag.getInt("Color");
Optional<String> level = tag.getString("Level");
Optional<Double> d = tag.getDouble("IAmRunningOutOfIdeasForNamesHere");
```

由于标签是否存在未知，返回值为 `Optional` 包装。基础类型可使用 `*Or*` 方法指定默认值。`ListTag` 可通过 `getListOrEmpty` 默认，`CompoundTag` 通过 `getCompoundOrEmpty` 默认。基础类型数组无 `*Or*` 等效方法。

```java
int color = tag.getIntOr("Color", 0xffffff);
String level = tag.getStringOr("Level", "minecraft:overworld");
double d = tag.getDoubleOr("IAmRunningOutOfIdeasForNamesHere", 1d);
```

所有标签类型均实现 `Tag` 接口。除 `CompoundTag` 外的大多数标签类型（如 `ByteTag` 或 `StringTag`）多为内部使用，但若遇到可直接通过 `CompoundTag#get` 和 `#put` 方法操作。

但有一个明显例外：`ListTag`。因其内部关联特定标签类型，操作较为特殊：

```java
ListTag newList = new ListTag();
// 将标签添加到列表
newList.add(StringTag.valueOf("Value1"));
newList.add(StringTag.valueOf("Value2"));

// 获取标签
ListTag getList = tag.getListOrEmpty("SomeListHere");
```

最后，在其他 `CompoundTag` 内操作 `CompoundTag` 直接使用 `CompoundTag#get` 和 `#put`：

```java
tag.put("Tag", new CompoundTag());

// 也可使用常规 `get` 处理 null 情况
tag.getCompoundOrEmpty("Tag");
```

## NBT 的用途

NBT 在 Minecraft 中应用广泛。[**方块实体**(`BlockEntity`)][blockentity]和[**实体**(`Entity`)][entity]将 NBT 使用抽象为**值访问**(`value accesses`)。**物品堆**(`ItemStack`)则将其抽象为[**数据组件**(`data components`)][datacomponents]。

## 另请参阅

- [Minecraft Wiki 上的 NBT 格式](`nbtwiki`)

[blockentity]: ../blockentities/index.md
[datapack]: ../resources/index.md#data
[datacomponents]: ../items/datacomponents.md
[entity]: ../entities/index.md
[nbtwiki]: https://minecraft.wiki/w/NBT_format
[valueio]: valueio.md