# **配置**(`Configuration`)

配置用于定义模组实例的设置和用户偏好。NeoForge 使用 [TOML][toml] 文件和 [NightConfig][nightconfig] 库实现配置系统。

## 创建配置

配置可通过 `IConfigSpec` 的子类型创建。NeoForge 通过 `ModConfigSpec` 实现，并借助 `ModConfigSpec.Builder` 构建。构建器可通过 `Builder#push` 创建配置分区，`Builder#pop` 退出分区。随后通过以下两种方法构建配置：

 方法         | 描述
 :----------- | :---
`build`       | 创建 `ModConfigSpec`
`configure`   | 创建包含配置值的类与 `ModConfigSpec` 的键值对

:::note
`ModConfigSpec.Builder#configure` 通常与 `static` 代码块结合使用，通过类的构造函数关联配置值：

```java
// 定义字段存储配置与规范
public static final ExampleConfig CONFIG;
public static final ModConfigSpec CONFIG_SPEC;

private ExampleConfig(ModConfigSpec.Builder builder) {
    // 定义配置属性
    // ...
}

// 使用静态块分离属性构建过程
static {
    Pair<ExampleConfig, ModConfigSpec> pair =
            new ModConfigSpec.Builder().configure(ExampleConfig::new);
        
    // 存储结果值
    CONFIG = pair.getLeft();
    CONFIG_SPEC = pair.getRight();
}
```
:::

每个配置值可附加上下文以扩展行为。上下文需在配置值构建前定义：

| 方法           | 描述                                                                          |
|:---------------|:------------------------------------------------------------------------------|
| `comment`      | 添加配置值功能描述（支持多行注释）                                            |
| `translation`  | 提供配置值名称的翻译键                                                        |
| `worldRestart` | 需重启世界才能生效                                                            |
| `gameRestart`  | 需重启游戏才能生效                                                            |

### **配置值**(`ConfigValue`)

配置值可通过任意 `#define` 方法构建（若已定义上下文）。

所有配置值方法至少接受两个参数：
- 变量路径：用 `.` 分隔的字符串，表示配置值所在分区
- 默认值（当无有效配置时使用）

`ConfigValue` 专属方法额外接受两个参数：
- 验证器：确保反序列化对象有效
- 数据类型类

```java
// 将配置属性存储为公共final字段
public final ModConfigSpec.ConfigValue<String> welcomeMessage;

private ExampleConfig(ModConfigSpec.Builder builder) {
    // 定义属性（如游戏初始化时输出的欢迎消息）
    welcomeMessage = builder.define("welcome_message", "Hello from the config!");
}
```

通过 `ConfigValue#get` 获取值。值会被缓存以避免重复读取文件。

#### 其他配置值类型

- **范围值**
    - 描述：值必须在指定范围内
    - 类型：`Comparable<T>`
    - 方法：`#defineInRange`
    - 额外参数：
        - 最小/最大值
        - 数据类型类

:::note
`DoubleValue`、`IntValue` 和 `LongValue` 是范围值特化，分别指定 `Double`、`Integer` 和 `Long` 类型。
:::

- **白名单值**
    - 描述：值必须在指定集合中
    - 类型：`T`
    - 方法：`#defineInList`
    - 额外参数：允许的值集合

- **列表值**
    - 描述：值为条目列表
    - 类型：`List<T>`
    - 方法：`#defineList`（或 `#defineListAllowEmpty` 允许空列表）
    - 额外参数：
        - 新增条目时的默认值供应器
        - 元素验证器
        - （可选）列表长度验证器

- **枚举值**
    - 描述：枚举值（需在指定集合中）
    - 类型：`Enum<T>`
    - 方法：`#defineEnum`
    - 额外参数：
        - 字符串/整数转枚举的转换器
        - 允许的枚举值集合

- **布尔值**
    - 描述：布尔值
    - 类型：`Boolean`
    - 方法：`#define`

## 注册配置

构建 `ModConfigSpec` 后需注册，以便 NeoForge 加载、跟踪和同步配置。应在模组构造函数中通过 `ModContainer#registerConfig` 注册，并指定[配置类型](configtype)（代表所属端）和 `ModConfigSpec`，可选指定配置文件名。

```java
// 在主模组文件中使用 ModConfigSpec CONFIG
public ExampleMod(ModContainer container) {
    ...
    // 注册配置
    container.registerConfig(ModConfig.Type.COMMON, ExampleConfig.CONFIG);
    ...
}
```

### **配置类型**(`Configuration Types`)

配置类型决定文件位置、加载时机及网络同步策略。所有配置默认从物理客户端的 `.minecraft/config` 或物理服务器的 `<server_folder>/config` 加载。各类型差异如下：

:::tip
NeoForge 在代码库中[详细说明配置类型](type)。
:::

- `STARTUP`
    - 从配置文件夹加载（客户端/服务器均适用）
    - 注册时立即读取
    - **不**网络同步
    - 默认后缀 `-startup`

:::warning
`STARTUP` 类型配置可能导致客户端与服务端不同步（如用于禁用内容注册）。强烈建议不要在此类型中启用/禁用可能改变模组内容的功能。
:::

- `CLIENT`
    - **仅**从物理客户端配置文件夹加载（无服务器位置）
    - 在触发 `FMLCommonSetupEvent` 前读取
    - **不**网络同步
    - 默认后缀 `-client`
- `COMMON`
    - 从配置文件夹加载（客户端/服务器均适用）
    - 在触发 `FMLCommonSetupEvent` 前读取
    - **不**网络同步
    - 默认后缀 `-common`
- `SERVER`
    - 从配置文件夹加载（客户端/服务器均适用）
        - 可通过以下路径按世界覆盖：
            - 客户端：`.minecraft/saves/<世界名>/serverconfig`
            - 服务器：`<服务器文件夹>/world/serverconfig`
    - 在触发 `ServerAboutToStartEvent` 前读取
    - 网络同步至客户端
    - 默认后缀 `-server`

## **配置事件**(`Configuration Events`)

配置加载/重载/卸载时的操作可通过 `ModConfigEvent.Loading`、`ModConfigEvent.Reloading` 和 `ModConfigEvent.Unloading` 事件实现。事件需[注册](events)到模组事件总线。

:::caution
这些事件针对模组所有配置触发，需通过提供的 `ModConfig` 对象区分具体配置。
:::

## **配置界面**(`Configuration Screen`)

配置界面允许玩家在游戏中编辑模组配置，无需手动修改文件。界面会自动解析注册的配置文件并填充内容。

模组可使用 NeoForge 内置配置界面。可通过继承 `ConfigurationScreen` 修改默认界面行为，或完全自定义界面。自定义界面需在模组构造期间通过[客户端](client)注册 `IConfigScreenFactory` 扩展点：

```java
// 在客户端主模组文件中
public ExampleModClient(ModContainer container) {
    ...
    // 使用 NeoForge 的 ConfigurationScreen 显示此模组配置
    container.registerExtensionPoint(IConfigScreenFactory.class, ConfigurationScreen::new);
    ...
}
```

在游戏中通过"模组"页面 → 选择模组 → 点击"配置"按钮访问界面。启动、通用和客户端配置随时可编辑。服务器配置仅在本地世界游戏中可编辑（连接服务器或他人局域网世界时禁用）。配置界面首页显示所有已注册配置文件供选择编辑。

:::warning
若创建自定义界面，需为所有配置条目添加翻译键并在语言 JSON 中定义文本。可通过 `ModConfigSpec$Builder#translation` 指定翻译键：
```java
ConfigValue<T> value = builder.comment("此值名为'config_value_name'，无配置时默认为 defaultValue")
    .translation("modid.config.config_value_name")
    .define("config_value_name", defaultValue);
```

为简化翻译，可打开配置界面浏览所有配置及子分区，然后返回模组列表界面。此时控制台将打印所有未翻译的配置条目及其键名。
:::

[toml]: https://toml.io/
[nightconfig]: https://github.com/TheElectronWill/night-config
[configtype]: #configuration-types
[type]: https://github.com/neoforged/FancyModLoader/blob/b4a1040118547a37c0f5f58146ac4cecc7817f82/loader/src/main/java/net/neoforged/fml/config/ModConfig.java#L86-L119
[events]: ../concepts/events.md#registering-an-event-handler
[client]: ../concepts/sides.md#mod