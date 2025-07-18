import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# **游戏测试**(`Game Tests`)

游戏测试是运行游戏内单元测试的方法。该系统设计为可扩展且并行运行，能高效处理大量测试。其应用广泛，包括测试对象交互和行为等。支持全代码或[数据包][datapacks]实现，下文将展示两种方式。

## 创建游戏测试

标准游戏测试包含四个步骤：
1. 加载结构模板（场景）用于测试交互或行为
2. 创建测试运行环境
3. 注册执行逻辑的函数（成功则测试通过，失败结果存储于场景旁讲台）
4. 链接前三者的测试实例

## 测试数据

所有测试实例包含 `TestData`，定义测试的初始配置、环境及结构模板。`TestData` 通过 `MapCodec` 序列化存储：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 示例游戏测试 examplemod:example_test
// 位于 'data/examplemod/test_instance/example_test.json'
{
    // `TestData`

    // 运行环境（指向 'data/examplemod/test_environment/example_environment.json'）
    "environment": "examplemod:example_environment",

    // 结构模板（指向 'data/examplemod/structure/example_structure.nbt'）
    "structure": "examplemod:example_structure",

    // 测试超时刻数（超时自动失败）
    "max_ticks": 400,

    // 准备刻数（不计入超时计数，默认0）
    "setup_ticks": 50,

    // 是否必须成功（默认true）
    "required": true,

    // 结构旋转（默认无旋转，可选：'none', 'clockwise_90', '180', 'counterclockwise_90'）
    "rotation": "clockwise_90",

    // 是否仅限命令运行（默认false）
    "manual_only": true,

    // 最大重试次数（默认1）
    "max_attempts": 3,

    // 最低成功次数（需≤最大重试次数，默认1）
    "required_successes": 1,

    // 是否保留顶部空间（仅块基测试使用，默认false）
    "sky_access": false

    // ...
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设已有测试环境
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 注册到模组事件总线
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_INSTANCE, bootstrap -> {
            // 获取测试环境
            HolderGetter<TestEnvironmentDefinition> environments = bootstrap.lookup(Registries.TEST_ENVIRONMENT);

            // 注册游戏测试
            bootstrap.register(..., new FunctionGameTestInstance(...,
                new TestData<>(
                    // 运行环境（指向 'data/examplemod/test_environment/example_environment.json'）
                    environments.getOrThrow(EXAMPLE_ENVIRONMENT),

                    // 结构模板（指向 'data/examplemod/structure/example_structure.nbt'）
                    ResourceLocation.fromNamespaceAndPath("examplemod", "example_structure"),

                    // 超时刻数
                    400,

                    // 准备刻数
                    50,

                    // 是否必须成功
                    true,

                    // 结构旋转
                    Rotation.CLOCKWISE_90,

                    // 是否仅限命令运行
                    true,

                    // 最大重试次数
                    3,

                    // 最低成功次数
                    1,

                    // 是否保留顶部空间
                    false
                )
            ));
        })
    );
}
```

</TabItem>
</Tabs>

## 结构模板

游戏测试在结构模板加载的场景中执行。模板定义场景尺寸及初始数据（方块/实体），存储为 `.nbt` 文件于 `data/<命名空间>/structure`。`TestData#structure` 通过相对 `ResourceLocation` 引用（如 `examplemod:example_structure` 指向 `data/examplemod/structure/example_structure.nbt`）。

## 测试环境

所有测试在 `TestEnvironmentDefinition` 定义的环境中运行，决定 `ServerLevel` 的设置方式。测试结束后环境被拆除，供后续实例使用。相同环境的测试实例将批量并行运行。环境文件位于 `data/<命名空间>/test_environment/<路径>.json`。

原版提供 `minecraft:default`（不修改 `ServerLevel`）。其他环境类型如下：

### **游戏规则**(`Game Rules`)

设置测试用游戏规则，结束后重置：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// examplemod:example_environment
// 位于 'data/examplemod/test_environment/example_environment.json'
{
    "type": "minecraft:game_rules",

    // 布尔规则列表
    "bool_rules": [
        {
            "rule": "doFireTick",
            "value": false
        }
        // ...
    ],

    // 整数规则列表
    "int_rules": [
        {
            "rule": "playersSleepingPercentage",
            "value": 50
        }
        // ...
    ]
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(...);

@SubscribeEvent
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_ENVIRONMENT, bootstrap -> {
            bootstrap.register(
                EXAMPLE_ENVIRONMENT,
                new TestEnvironmentDefinition.SetGameRules(
                    // 布尔规则列表
                    List.of(
                        new TestEnvironmentDefinition.SetGameRules.Entry(
                            GameRules.RULE_DOFIRETICK,
                            GameRules.BooleanValue.create(false)
                    ),
                    // 整数规则列表
                    List.of(
                        new TestEnvironmentDefinition.SetGameRules.Entry(
                            GameRules.RULE_PLAYERS_SLEEPING_PERCENTAGE,
                            GameRules.IntegerValue.create(50))
                    )
                )
            );
        })
    );
}
```

</TabItem>
</Tabs>

### **时间**(`Time of Day`)

设置世界时间（类似 `/time set`）：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "minecraft:time_of_day",
    // 时间值（白天1000, 正午6000, 夜晚13000, 午夜18000）
    "time": 13000
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(
    EXAMPLE_ENVIRONMENT,
    new TestEnvironmentDefinition.TimeOfDay(13000)
);
```

</TabItem>
</Tabs>

### **天气**(`Weather`)

设置天气（类似 `/weather`）：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "minecraft:weather",
    // 天气类型：clear（晴）, rain（雨）, thunder（雷雨）
    "weather": "thunder"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(
    EXAMPLE_ENVIRONMENT,
    new TestEnvironmentDefinition.Weather(TestEnvironmentDefinition.Weather.Type.THUNDER)
);
```

</TabItem>
</Tabs>

### **Minecraft 函数**(`Minecraft Functions`)

通过 `mcfunction` 设置/拆除环境：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "minecraft:function",
    // 设置函数（指向 'data/examplemod/function/example/setup.mcfunction'）
    "setup": "examplemod:example/setup",
    // 拆除函数（指向 'data/examplemod/function/example/teardown.mcfunction'）
    "teardown": "examplemod:example/teardown"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(
    EXAMPLE_ENVIRONMENT,
    new TestEnvironmentDefinition.Functions(
        Optional.of(ResourceLocation.fromNamespaceAndPath("examplemod", "example/setup")),
        Optional.of(ResourceLocation.fromNamespaceAndPath("examplemod", "example/teardown"))
);
```

</TabItem>
</Tabs>

### **复合环境**(`Composites`)

合并多个环境：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "minecraft:all_of",
    // 环境列表（可引用或内联定义）
    "definitions": [
        "minecraft:default", // 引用原版默认环境
        { "type": "..." }    // 内联环境定义
    ]
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(
    EXAMPLE_ENVIRONMENT,
    new TestEnvironmentDefinition.AllOf(
        List.of(
            environments.getOrThrow(GameTestEnvironments.DEFAULT_KEY),
            Holder.direct(...) // 创建新环境
        )
    )
);
```

</TabItem>
</Tabs>

### **自定义环境类型**

自定义 `TestEnvironmentDefinition` 需实现：
- `setup`：修改 `ServerLevel`
- `teardown`：恢复修改
- `codec`：提供序列化的 `MapCodec`

```java
public record ExampleEnvironmentType(int value1, boolean value2) implements TestEnvironmentDefinition {

    // 注册MapCodec
    public static final MapCodec<ExampleEnvironmentType> CODEC = ...;

    @Override
    public void setup(ServerLevel level) { /* 环境设置 */ }

    @Override
    public void teardown(ServerLevel level) { /* 环境恢复 */ }

    @Override
    public MapCodec<ExampleEnvironmentType> codec() {
        return EXAMPLE_ENVIRONMENT_CODEC.get();
    }
}

// 注册自定义类型
public static final DeferredRegister<MapCodec<? extends TestEnvironmentDefinition>> TEST_ENVIRONMENT_DEFINITION_TYPES = ...;

public static final Supplier<MapCodec<ExampleEnvironmentType>> EXAMPLE_ENVIRONMENT_CODEC = TEST_ENVIRONMENT_DEFINITION_TYPES.register(
    "example_environment_type", () -> ...);
```

使用自定义环境：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "examplemod:example_environment_type",
    "value1": 0,
    "value2": true
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(
    EXAMPLE_ENVIRONMENT,
    new ExampleEnvironmentType(0, true)
);
```

</TabItem>
</Tabs>

## 测试函数

游戏测试核心是运行接收 `GameTestHelper` 的函数。通过调用 `GameTestHelper` 方法决定测试成功/失败。测试函数需[注册][registered]：

```java
public class ExampleFunctions {
    // 示例测试函数
    public static void exampleTest(GameTestHelper helper) {
        // 执行测试逻辑
    }
}

// 注册函数
public static final DeferredRegister<Consumer<GameTestHelper>> TEST_FUNCTION = ...;

public static final DeferredHolder<Consumer<GameTestHelper>, Consumer<GameTestHelper>> EXAMPLE_FUNCTION = TEST_FUNCTION.register(
    "example_function", () -> ExampleFunctions::exampleTest);
```

### **相对定位**(`Relative Positioning`)

测试函数通过结构块位置将场景相对坐标转换为绝对坐标。使用 `GameTestHelper#absolutePos` 和 `GameTestHelper#relativePos` 转换坐标。

游戏中可通过 `/test` 命令获取相对坐标：加载结构 → 移动到目标位置 → 执行 `/test pos`。命令将生成可复制的相对坐标文本。

:::tip
可通过 `/test pos <变量名>` 指定变量名：
```bash
/test pos varName # 生成 'final BlockPos varName = new BlockPos(...);'
```
:::

### **成功完成**(`Successful Completion`)

测试函数需在超时前（`TestData#maxTicks` 定义）标记成功，否则自动失败。关键方法：

方法               | 描述
:---:              | :---
`#succeed`         | 标记测试成功
`#succeedIf`       | 立即检查 `Runnable`，无异常则成功（否则失败）
`#succeedWhen`     | 每刻检查 `Runnable` 直到成功或无异常
`#succeedOnTickWhen` | 指定刻检查 `Runnable`，无异常则成功（其他刻失败）

:::caution
测试每刻执行直到标记成功。需确保计划刻测试在非目标刻始终失败。
:::

### **计划操作**(`Scheduling Actions`)

方法           | 描述
:---:          | :---
`#runAtTickTime` | 指定刻运行操作
`#runAfterDelay` | 延迟X刻运行操作
`#onEachTick`    | 每刻运行操作

### **断言**(`Assertions`)

测试中随时可通过断言检查条件。`GameTestHelper` 提供多种断言方法，本质是在条件不满足时抛出 `GameTestAssertException`。

## 注册测试实例

通过 `GameTestInstance` 链接 `TestData`、`TestEnvironmentDefinition` 和测试函数。测试实例文件位于 `data/<命名空间>/test_instance/<路径>.json`。

### **函数基测试**(`Function-Based Tests`)

`FunctionGameTestInstance` 链接 `TestData` 与注册的测试函数：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    // `TestData` 字段...
    "type": "minecraft:function",
    // 指向测试函数注册项
    "function": "examplemod:example_function"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(EXAMPLE_TEST_INSTANCE,
    new FunctionGameTestInstance(
        EXAMPLE_FUNCTION.getKey(),
        new TestData<>(...)
    )
);
```

</TabItem>
</Tabs>

### **块基测试**(`Block-Based Tests`)

`BlockBasedTestInstance` 依赖 `Blocks#TEST_BLOCK` 的红石信号。结构需含：
- 一个 `TestBlockMode#START` 块（发送15信号脉冲）
- 一个 `TestBlockMode#ACCEPT` 块（触发则测试成功）
- `LOG` 块（触发时发送脉冲）
- `FAIL` 块（触发则测试失败）

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    // `TestData` 字段...
    "type": "minecraft:block_based"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(EXAMPLE_TEST_INSTANCE,
    new BlockBasedTestInstance(new TestData<>(...))
);
```

</TabItem>
</Tabs>

### **自定义测试实例**

扩展 `GameTestInstance` 需实现：
- `run`：测试逻辑
- `typeDescription`：测试描述
- `codec`：提供序列化的 `MapCodec`（用于数据生成）

```java
public class ExampleTestInstance extends GameTestInstance {
    public ExampleTestInstance(int value1, boolean value2, TestData<Holder<TestEnvironmentDefinition>> info) {
        super(info);
    }

    @Override
    public void run(GameTestHelper helper) {
        helper.assertBlockPresent(...);
        helper.succeedIf(() -> ...);
    }

    @Override
    public MapCodec<ExampleTestInstance> codec() { ... }

    @Override
    protected MutableComponent typeDescription() {
        return Component.literal("示例测试实例");
    }
}

// 注册自定义实例类型
public static final DeferredRegister<MapCodec<? extends GameTestInstance>> TEST_INSTANCE = ...;
public static final Supplier<MapCodec<? extends GameTestInstance>> EXAMPLE_INSTANCE_CODEC = ...;
```

使用自定义实例：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    // `TestData` 字段...
    "type": "examplemod:example_test_instance",
    "value1": 0,
    "value2": true
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
bootstrap.register(EXAMPLE_TEST_INSTANCE,
    new ExampleTestInstance(0, true, new TestData<>(...))
);
```

</TabItem>
</Tabs>

### **跳过数据包**

通过 `RegisterGameTestsEvent` 事件直接注册环境与测试实例：

```java
@SubscribeEvent // 注册到模组事件总线
public static void registerTests(RegisterGameTestsEvent event) {
    Holder<TestEnvironmentDefinition> environment = event.registerEnvironment(
        EXAMPLE_ENVIRONMENT.location(),
        new ExampleEnvironmentType(0, true)
    );

    event.registerTest(
        EXAMPLE_TEST_INSTANCE.location(),
        new ExampleTestInstance(0, true, new TestData<>(...))
    );
}
```

## 运行游戏测试

通过 `/test` 命令运行测试：

| 子命令       | 描述                     |
|:-----------:|:------------------------|
| `run`       | 运行指定测试             |
| `runall`    | 运行所有可用测试         |
| `runclosest`| 运行玩家15格内最近测试   |
| `runthese`  | 运行玩家200格内测试      |
| `runfailed` | 运行上次失败的测试       |

:::note
命令格式：`/test <子命令>`
:::

## 构建脚本配置

`build.gradle` 中可配置测试运行：

### **游戏测试服务器配置**

专用配置运行构建服务器，返回失败测试数。通过 `gradlew runGameTestServer` 运行。

### **在其他配置启用测试**

默认仅 `client` 和 `gameTestServer` 配置启用测试。其他配置需添加：

```gradle
// 在运行配置内
property 'neoforge.enableGameTest', 'true'
```

[datapacks]: ../resources/index.md#data
[registered]: ../concepts/registries.md#methods-for-registering
[test]: #running-game-tests
[event]: ../concepts/events.md#registering-an-event-handler
