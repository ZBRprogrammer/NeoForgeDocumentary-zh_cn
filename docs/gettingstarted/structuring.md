# **构建模组结构**(`Structuring Your Mod`)

结构化的模组有益于维护、贡献以及更清晰地理解底层代码库。以下是来自 Java、Minecraft 和 NeoForge 的建议。

:::note
不必遵循以下建议；可按需构建模组结构。但仍强烈推荐采用。
:::

## **包结构**(`Packaging`)

构建模组时，选择独特的顶级包结构。许多程序员会为不同类、接口等使用相同名称。Java 允许类在不同包中同名。因此，若两个类在相同包中有相同名称，只有一个会被加载（很可能导致游戏崩溃）。

```
a.jar
    - com.example.ExampleClass
b.jar
    - com.example.ExampleClass // 此类通常不会被加载
```

加载模块时此问题更突出。若不同模块中存在同名包下的类文件，将导致模组加载器启动崩溃（因模组模块会导出到游戏和其他模组）。

```
module A
    - package X
        - class I
        - class J
module B
    - package X // 此包将导致模组加载器崩溃，因已存在导出的包 X
        - class R
        - class S
        - class T
```

因此，顶级包应为自有标识：域名、邮箱、网站（子域名）等。只要在目标环境中可唯一标识，甚至可用姓名或用户名。此外，顶级包应匹配[**组 ID**(`group id`)][group]。

|   类型   |       值       | 顶级包   |
|:---------:|:-----------------:|:--------------------|
|  域名  |    example.com    | `com.example`       |
| 子域名 | example.github.io | `io.github.example` |
|  邮箱  | example@gmail.com | `com.gmail.example` |

下一级包应为模组 ID（如`com.example.examplemod`，其中`examplemod`为模组 ID）。这将确保除非存在相同模组 ID（不应发生），否则包加载无任何问题。

更多命名规范参见[**Oracle 教程页**][naming]。

### **子包组织**(`Sub-package Organization`)

除顶级包外，强烈建议将模组类拆分到子包中。主要有两种方法：

- **按功能分组**(`Group By Function`)：为具有共同目的的类创建子包。例如，方块在`block`下，物品在`item`下，实体在`entity`下等。Minecraft 本身采用类似结构（有例外）。
- **按逻辑分组**(`Group By Logic`)：为具有共同逻辑的类创建子包。例如，若创建新型工作台，可将其方块、菜单、物品等置于`feature.crafting_table`下。

#### **客户端、服务器和数据包**(`Client, Server, and Data Packages`)

通常，仅用于特定端或运行时的代码应与其他类隔离在单独子包中。例如，[**数据生成**(`data generation`)][datagen]相关代码应在`data`包中，仅专用服务器的代码应在`server`包中。

强烈建议将[**仅客户端代码**(`client-only code`)][sides]隔离在`client`子包中。因为专用服务器无法访问 Minecraft 中任何仅客户端包，若模组尝试访问将崩溃。因此，专用包可有效检查模组内是否跨端访问。

## **类命名规范**(`Class Naming Schemes`)

统一的类命名规范便于理解类用途或快速定位特定类。

类通常以其类型为后缀，例如：

- 名为`PowerRing`的`Item` -> `PowerRingItem`
- 名为`NotDirt`的`Block` -> `NotDirtBlock`
- `Oven`的菜单 -> `OvenMenu`

:::tip
除实体外，Mojang 通常对所有类采用类似结构。实体类仅用名称表示（如`Pig`、`Zombie`等）。
:::

## **从多种方法中选择一种**(`Choose One Method from Many`)

执行任务的方法多样：注册对象、监听事件等。通常建议使用单一方法完成任务以保持一致性。这能提高可读性，避免可能发生的奇怪交互或冗余（如事件监听器运行两次）。

[group]: index.md#组-id
[naming]: https://docs.oracle.com/javase/tutorial/java/package/namingpkgs.html
[datagen]: ../resources/index.md#数据生成
[sides]: ../concepts/sides.md