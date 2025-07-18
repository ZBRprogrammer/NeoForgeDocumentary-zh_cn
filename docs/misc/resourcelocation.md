# **资源定位符**(`Resource Locations`)

**资源定位符**(`ResourceLocation`) 是 Minecraft 中最核心的概念之一。它作为键用于[注册表](registries)，作为数据/资源文件的标识符，作为代码中模型的引用，并广泛应用于其他场景。`ResourceLocation` 由两部分组成：命名空间(`namespace`)和路径(`path`)，以 `:` 分隔。

命名空间表示资源所属的模组、资源包或数据包。例如，模组 ID 为 `examplemod` 的模组使用 `examplemod` 命名空间。Minecraft 使用 `minecraft` 命名空间。额外命名空间可通过创建对应数据文件夹自由定义（通常由数据包使用，以分离逻辑）。

路径是命名空间内部的目标对象引用。例如：
- `minecraft:cow`：`minecraft` 命名空间中的牛实体引用
- `examplemod:example_item`：模组物品注册表中的物品引用

`ResourceLocation` 仅可包含小写字母、数字、下划线、点和连字符。路径可额外包含正斜杠。注意：因 Java 模块限制，模组 ID 不能含连字符，故模组命名空间也不能含连字符（路径中仍允许）。

:::info
`ResourceLocation` 本身不指定对象类型。例如 `minecraft:dirt` 存在于多个位置。具体关联对象由接收方决定。
:::

创建方式：
```java
// 显式指定命名空间和路径
ResourceLocation.fromNamespaceAndPath("examplemod", "example_item")

// 解析字符串形式
ResourceLocation.parse("examplemod:example_item")

// 使用默认命名空间 minecraft
ResourceLocation.withDefaultNamespace("example_item") // 得到 minecraft:example_item
```

通过 `ResourceLocation#getNamespace()` 和 `#getPath()` 获取命名空间和路径，`#toString` 获取完整字符串。所有工具方法（如 `withPrefix`/`withSuffix`）均返回新实例（不可变）。

## 解析 `ResourceLocation`s

不同场景解析方式各异：
- **GUI 背景**：`minecraft:textures/gui/container/furnace.png` → `assets/minecraft/textures/gui/container/furnace.png`（需后缀）
- **方块模型**：`minecraft:block/dirt` → `assets/minecraft/models/block/dirt.json`（无后缀，自动映射子目录）
- **客户端物品**：`minecraft:apple` → `assets/minecraft/items/apple.json`（无后缀，自动映射子目录）
- **配方**：`minecraft:iron_block` → `data/minecraft/recipe/iron_block.json`（无后缀，自动映射子目录）

是否需文件后缀及具体解析路径取决于使用场景。

## **资源键**(`ResourceKey`s)

**资源键**(`ResourceKey`) 组合注册表 ID 和注册名称。例如：
- 注册表 ID：`minecraft:item`
- 注册名称：`minecraft:diamond_sword`

与 `ResourceLocation` 不同，`ResourceKey` 指向唯一元素。常用于多注册表交互场景（如数据包/世界生成）。

创建方式：
```java
// 创建资源键
ResourceKey.create(
    // 注册表键（通过根注册表创建）
    ResourceKey.createRegistryKey(new ResourceLocation("minecraft:item")),
    // 注册名称
    new ResourceLocation("minecraft:diamond_sword")
)
```

`ResourceKey` 在创建时被内部化(`interned`)，支持引用相等性(`==`)比较，但创建开销较大。

[registries]: ../concepts/registries.md
[sides]: ../concepts/sides.md