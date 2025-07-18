# **可扩展枚举**(`Extensible Enums`)

**可扩展枚举**(`Extensible Enums`)是对特定原版枚举的增强，允许添加新条目。通过在运行时修改枚举的编译字节码来实现元素的添加。

## **IExtensibleEnum**

所有可添加新条目的枚举都实现了 `IExtensibleEnum` 接口。该接口作为标记，使 `RuntimeEnumExtender` 启动插件服务能识别哪些枚举应被转换。

:::warning
您**不应**在自己的枚举上实现此接口。请根据用例改用映射或注册表。  
未通过补丁实现此接口的枚举无法通过混入(`mixins`)或核心模组(`coremods`)添加该接口，因为转换器的运行顺序不允许。
:::

### 创建枚举条目

要创建新的枚举条目，需创建 JSON 文件并在 `neoforge.mods.toml` 的 `[[mods]]` 块中通过 `enumExtensions` 条目引用。指定路径必须相对于 `resources` 目录：
```toml
# 在 neoforge.mods.toml 中：
[[mods]]
## 文件路径相对于资源的输出目录，或编译后 jar 内的根路径
## 'resources' 目录表示资源的根输出目录
enumExtensions="META-INF/enumextensions.json"
```

条目定义包含目标枚举的类名、新字段名（必须以模组 ID 为前缀）、用于构造条目的构造函数**描述符**(`descriptor`)以及传递给该构造函数的参数。

```json5
{
    "entries": [
        {
            // 要添加条目的枚举类
            "enum": "net/minecraft/world/item/ItemDisplayContext",
            // 新条目的字段名，必须以模组 ID 为前缀
            "name": "EXAMPLEMOD_STANDING",
            // 要使用的构造函数
            "constructor": "(ILjava/lang/String;Ljava/lang/String;)V",
            // 直接提供的常量参数
            "parameters": [ -1, "examplemod:standing", null ]
        },
        {
            "enum": "net/minecraft/world/item/Rarity",
            "name": "EXAMPLEMOD_CUSTOM",
            "constructor": "(ILjava/lang/String;Ljava/util/function/UnaryOperator;)V",
            // 要使用的参数，作为对给定类中 EnumProxy<Rarity> 字段的引用提供
            "parameters": {
                "class": "example/examplemod/MyEnumParams",
                "field": "CUSTOM_RARITY_ENUM_PROXY"
            }
        },
        {
            "enum": "net/minecraft/world/damagesource/DamageEffects",
            "name": "EXAMPLEMOD_TEST",
            "constructor": "(Ljava/lang/String;Ljava/util/function/Supplier;)V",
            // 要使用的参数，作为对给定类中方法的引用提供
            "parameters": {
                "class": "example/examplemod/MyEnumParams",
                "method": "getTestDamageEffectsParameter"
            }
        }
    ]
}
```

```java
public class MyEnumParams {
    public static final EnumProxy<Rarity> CUSTOM_RARITY_ENUM_PROXY = new EnumProxy<>(
            Rarity.class, -1, "examplemod:custom", (UnaryOperator<Style>) style -> style.withItalic(true)
    );
    
    public static Object getTestDamageEffectsParameter(int idx, Class<?> type) {
        return type.cast(switch (idx) {
            case 0 -> "examplemod:test";
            case 1 -> (Supplier<SoundEvent>) () -> SoundEvents.DONKEY_ANGRY;
            default -> throw new IllegalArgumentException("Unexpected parameter index: " + idx);
        });
    }
}
```

#### 构造函数

构造函数必须指定为[方法描述符][jvmdescriptors]，且只能包含源代码中可见的参数，忽略隐藏的常量名和序数参数。  
若构造函数标记了 `@ReservedConstructor` 注解，则不能用于模组化枚举常量。

#### 参数

参数可通过三种方式指定，具体取决于参数类型：

- 在 JSON 文件中以内联常量数组形式指定（仅允许原始值、字符串和为任何引用类型传递 null）
- 作为对模组类中 `EnumProxy<TheEnum>` 类型字段的引用（参见上方 `EnumProxy` 示例）
    - 第一个参数指定目标枚举，后续参数传递给枚举构造函数
- 作为对返回 `Object` 的方法引用，返回值即要使用的参数值。该方法必须正好有两个参数：`int`（参数索引）和 `Class<?>`（参数预期类型）
    - 应使用 `Class<?>` 对象对返回值进行强制转换（`Class#cast()`），以便在模组代码中保留 `ClassCastException`

:::warning
用作参数值来源的字段和/或方法应位于单独的类中，以避免过早意外加载模组类。
:::

特定参数有额外规则：

- 若参数是与枚举上 `@IndexedEnum` 注解相关的整型 ID 参数，则会被忽略并替换为条目的序数。若在 JSON 中内联指定该参数，则必须设为 `-1`，否则会抛出异常
- 若参数是与枚举上 `@NamedEnum` 注解相关的字符串名称参数，则必须以模组 ID 为前缀（采用 `ResourceLocation` 的 `namespace:path` 格式），否则会抛出异常

#### 获取生成的常量

可通过 `TheEnum.valueOf(String)` 获取生成的枚举常量。若使用字段引用提供参数，也可通过 `EnumProxy#getValue()` 从 `EnumProxy` 对象获取常量。

## 为 NeoForge 贡献

要向 NeoForge 添加新的可扩展枚举，至少需要完成两项操作：

- 使枚举实现 `IExtensibleEnum`，标记该枚举应通过 `RuntimeEnumExtender` 转换
- 添加返回 `ExtensionInfo.nonExtended(TheEnum.class)` 的 `getExtensionInfo` 方法

根据枚举的具体细节还需进一步操作：

- 若枚举具有应与条目序数匹配的整型 ID 参数，且该参数不是第一个参数，则枚举应使用 `@IndexedEnum` 注解，并将 ID 的参数索引作为注解值
- 若枚举具有用于序列化的字符串名称参数（因此应命名空间化），且该参数不是第一个参数，则枚举应使用 `@NamedEnum` 注解，并将名称的参数索引作为注解值
- 若枚举通过网络传输，则应使用 `@NetworkedEnum` 注解，其参数指定值可能传输的方向（客户端绑定、服务端绑定或双向）
- 若枚举存在模组不可用的构造函数（例如因为它们在模组化注册运行前初始化，需要枚举上的注册表对象），则应使用 `@ReservedConstructor` 注解标记

:::note
若枚举实际添加了任何条目，`getExtensionInfo` 方法将在运行时被转换以提供动态生成的 `ExtensionInfo`。
:::

```java
// 此为示例，非原版实际枚举

// 第一个参数必须匹配枚举常量的序数
@net.neoforged.fml.common.asm.enumextension.IndexedEnum
// 第二个参数是必须带模组 ID 前缀的字符串
@net.neoforged.fml.common.asm.enumextension.NamedEnum(1)
// 此枚举用于网络传输，需检查客户端与服务端是否匹配
@net.neoforged.fml.common.asm.enumextension.NetworkedEnum(net.neoforged.fml.common.asm.enumextension.NetworkedEnum.NetworkCheck.BIDIRECTIONAL)
public enum ExampleEnum implements net.neoforged.fml.common.asm.enumextension.IExtensibleEnum {
    // VALUE_1 在此表示名称参数
    VALUE_1(0, "value_1", false),
    VALUE_2(1, "value_2", true),
    VALUE_3(2, "value_3");

    ExampleEnum(int arg1, String arg2, boolean arg3) {
        // ...
    }

    ExampleEnum(int arg1, String arg2) {
        this(arg1, arg2, false);
    }

    public static net.neoforged.fml.common.asm.enumextension.ExtensionInfo getExtensionInfo() {
        return net.neoforged.fml.common.asm.enumextension.ExtensionInfo.nonExtended(ExampleEnum.class);
    }
}
```

[jvmdescriptors]: https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.3.2