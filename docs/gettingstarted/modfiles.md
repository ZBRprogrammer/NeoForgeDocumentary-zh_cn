# **模组文件**(`Mod Files`)

模组文件负责确定哪些模组被打包到 JAR 中，在"模组"菜单中显示哪些信息，以及模组在游戏中应如何加载。

## `gradle.properties`

`gradle.properties`文件保存模组各种通用属性（如模组 ID 或版本）。构建期间，Gradle 读取此文件的值并内联到各处（如[**neoforge.mods.toml**][neoforgemodstoml]文件）。这样只需在一处更改值，即可全局应用。

[**MDK 的 `gradle.properties` 文件**][mdkgradleproperties]中大部分值已通过注释说明。

| 属性                      | 描述                                                                                                                                                                                                                             | 示例                                    |
|---------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| `org.gradle.jvmargs`      | 允许向 Gradle 传递额外 JVM 参数。通常用于为 Gradle 分配更多/更少内存。注意：此设置针对 Gradle 本身，而非 Minecraft。                                                                 | `org.gradle.jvmargs=-Xmx3G`                |
| `org.gradle.daemon`       | Gradle 构建时是否使用守护进程。                                                                                                                                                                                     | `org.gradle.daemon=false`                  |
| `org.gradle.parallel`     | Gradle 是否应分叉 JVM 以并行执行项目。                                                                                                                                                                                     | `org.gradle.parallel=false`                  |
| `org.gradle.caching`      | Gradle 是否应复用先前构建的任务输出。                                                                                                                                                                                     | `org.gradle.caching=false`                  |
| `org.gradle.configuration-cache`  | Gradle 是否应复用先前构建的配置。                                                                                                                                                                                     | `org.gradle.configuration-cache=false`                  |
| `org.gradle.debug`        | Gradle 是否设置为调试模式。调试模式主要输出更多 Gradle 日志。注意：此设置针对 Gradle 本身，而非 Minecraft。                                                                                                | `org.gradle.debug=false`                   |
| `minecraft_version`       | 开发模组所用的 Minecraft 版本。必须与`neo_version`匹配。                                                                                                                                                                | `minecraft_version=1.20.6`                 |
| `minecraft_version_range` | 模组兼容的 Minecraft 版本范围（以[**Maven 版本范围**(`Maven Version Range`)][mvr]表示）。注意[**快照版、预发布版和候选版**(`snapshots, pre-releases and release candidates`)][mcversioning]因不遵循 Maven 版本规范，排序可能不正确。    | `minecraft_version_range=[1.20.6,1.21)`    |
| `neo_version`             | 开发模组所用的 NeoForge 版本。必须与`minecraft_version`匹配。NeoForge 版本机制详见[**NeoForge 版本规范**(`NeoForge Versioning`)][neoversioning]。                                                           | `neo_version=20.6.62`                      |
| `neo_version_range`       | 模组兼容的 NeoForge 版本范围（以[**Maven 版本范围**(`Maven Version Range`)][mvr]表示）。                                                                                                                                                           | `neo_version_range=[20.6.62,20.7)`         |
| `mod_id`                  | 详见[**模组 ID**(`The Mod ID`)][modid]。                                                                                                                                                                                                                | `mod_id=examplemod`                        |
| `mod_name`                | 模组的人类可读显示名称。默认仅在模组列表显示，但[JEI][jei]等模组也会在物品提示中显著显示模组名称。                                                | `mod_name=Example Mod`                     |
| `mod_license`             | 模组采用的许可证。建议设为使用的[**SPDX 标识符**(`SPDX identifier`)][spdx]和/或许可证链接。可通过 https://choosealicense.com/ 选择许可证。 | `mod_license=MIT`                          |
| `mod_version`             | 模组版本（显示于模组列表）。详见[**版本规范**(`the page on Versioning`)][versioning]。                                                                                                                          | `mod_version=1.0`                          |
| `mod_group_id`            | 详见[**组 ID**(`The Group ID`)][group]。                                                                                                                                                                                                              | `mod_group_id=com.example.examplemod`      |
| `mod_authors`             | 模组作者（显示于模组列表）。                                                                                                                                                                                          | `mod_authors=ExampleModder`                |
| `mod_description`         | 模组描述（多行字符串，显示于模组列表）。换行符(`\n`)可用且会被正确处理。                                                                                          | `mod_description=Example mod description.` |

### **模组 ID**(`The Mod ID`)

模组 ID 是区分模组的主要标识。广泛用于模组[**注册表**(`registries`)][registration]的命名空间，以及[**资源和数据包**(`resource and data pack`)][resource]的命名空间。两个模组 ID 相同将导致游戏无法加载。

因此，模组 ID 应独特且易记。通常为模组显示名称（小写）或其变体。模组 ID 只能包含小写字母、数字和下划线，长度必须在 2 到 64 字符之间（含）。

:::info
在`gradle.properties`中更改此属性会自动全局应用，但主模组类的[`@Mod`注解][javafml]除外，需手动更改以匹配`gradle.properties`中的值。
:::

### **组 ID**(`The Group ID`)

`build.gradle`中的`group`属性仅当计划将模组发布到 Maven 仓库时才必需，但始终正确设置是良好实践。可通过`gradle.properties`的`mod_group_id`属性设置。

组 ID 应设为顶级包名。更多信息见[**包结构**(`Packaging`)][packaging]。

```properties
# 在 gradle.properties 文件中
mod_group_id=com.example
```

Java 源码（`src/main/java`）内的包结构也应符合此结构，内部包表示模组 ID：

```text
com
- example (group 属性指定的顶级包)
    - mymod (模组 ID)
        - MyMod.java (重命名后的 ExampleMod.java)
```

## `neoforge.mods.toml`

`neoforge.mods.toml`文件位于`src/main/resources/META-INF/neoforge.mods.toml`，采用[**TOML**][toml]格式定义模组的元数据。还包含模组加载到游戏的额外信息，以及在"模组"菜单中显示的描述信息。[**MDK 提供的 `neoforge.mods.toml` 文件**][mdkneoforgemodstoml]的注释解释了每个条目，此处将详细说明。

`neoforge.mods.toml`分为三部分：非模组特定属性（关联到模组文件）；模组属性（每个模组对应一节）；依赖配置（每个模组的依赖对应一节）。部分属性为必需项；必需项无值将抛出异常。

:::note
默认 MDK 中，Gradle 将此文件中的属性替换为`gradle.properties`中的值。例如，`license="${mod_license}"`表示`license`字段被`gradle.properties`中的`mod_license`属性替换。应通过修改`gradle.properties`而非直接修改此文件来更改此类值。
:::

### 非模组特定属性

非模组特定属性关联到 JAR 本身，指示如何加载模组及附加全局元数据。

| 属性             | 类型     | 默认值        | 描述                                                                                                                                                                                                                                                                                                                                         | 示例                                                                        |
|----------------------|----------|----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| `modLoader`          | 字符串   | `javafml`      | 模组使用的语言加载器。可用于支持替代语言结构（如主文件使用 Kotlin 对象）或不同的入口点确定方法（如接口或方法）。NeoForge 提供 Java 加载器[`"javafml"`][javafml]。  | `modLoader="javafml"`                                                          |
| `loaderVersion`      | 字符串   | `""`           | 语言加载器的可接受版本范围（以[**Maven 版本范围**(`Maven Version Range`)][mvr]表示）。`javafml`当前版本为`1`。未指定版本时，可使用任何版本的加载器。                                                                                                                                                                                      | `loaderVersion="[1,)"`                                                         |
| `license`            | 字符串   | **必需**  | 此 JAR 中模组的许可证。建议设为使用的[**SPDX 标识符**(`SPDX identifier`)][spdx]和/或许可证链接。可通过 https://choosealicense.com/ 选择许可证。                                                                                              | `license="MIT"`                                                                |
| `showAsResourcePack` | 布尔值  | `false`        | 为`true`时，模组资源将作为独立资源包显示在"资源包"菜单，而非合并到"模组资源"包。                                                                                                                                                                           | `showAsResourcePack=true`                                                      |
| `showAsDataPack`     | 布尔值  | `false`        | 为`true`时，模组数据文件将作为独立数据包显示在"数据包"菜单，而非合并到"模组数据"包。                                                                                                                                                                           | `showAsDataPack=true`                                                          |
| `services`           | 数组    | `[]`           | 模组使用的服务数组。作为 NeoForge 实现 **Java 平台模块系统**（`Java Platform Module System`）时模组创建模块的一部分被消费。                                                                                                                                                                                   | `services=["net.neoforged.neoforgespi.language.IModLanguageProvider"]`         |
| `properties`         | 表    | `{}`           | 替换属性表。供`StringSubstitutor`将`${file.<key>}`替换为对应值。                                                                                                                                                                                                                    | `properties={"example"="1.2.3"}`（可通过`${file.example}`引用） |
| `issueTrackerURL`    | 字符串   | _无_      | 报告和跟踪模组问题的 URL。                                                                                                                                                                                                                                                                            | `"https://github.com/neoforged/NeoForge/issues"`                               |

:::note
`services`属性在功能上等同于在模块中指定[`uses`指令][uses]，允许[**加载给定类型的服务**(`loading a service of a given type`)][serviceload]。

也可在`src/main/resources/META-INF/services`文件夹的服务文件中定义，文件名是服务的全限定名，文件内容是待加载服务的名称（参见[**AtlasViewer 模组的示例**][atlasviewer]）。
:::

### 模组特定属性

模组特定属性通过`[[mods]]`标头绑定到指定模组。这是[**表数组**(`array of tables`)][array]；所有键/值属性将附加到该模组直至下一标头。

```toml
# examplemod1 的属性
[[mods]]
modId = "examplemod1"

# examplemod2 的属性
[[mods]]
modId = "examplemod2"
```

| 属性         | 类型     | 默认值                      | 描述                                                                                                                                                                                                                                                                    | 示例                                                         |
|------------------|----------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| `modId`          | 字符串   | **必需**                | 详见[**模组 ID**(`The Mod ID`)][modid]。                                                                                                                                                                                                                                                       | `modId="examplemod"`                                            |
| `namespace`      | 字符串   | `modId`的值             | 模组的覆盖命名空间。也必须是有效的[**模组 ID**(`mod ID`)][modid]，但可额外包含点或破折号。当前未使用。                                                                                                                                        | `namespace="example"`                                           |
| `version`        | 字符串   | `"1"`                        | 模组版本（推荐使用[**Maven 版本规范变体**(`variation of Maven versioning`)][versioning]。设为`${file.jarVersion}`时，将替换为 JAR 清单中`Implementation-Version`属性的值（开发环境中显示为`0.0NONE`）。 | `version="1.20.2-1.0.0"`                                        |
| `displayName`    | 字符串   | `modId`的值             | 模组显示名称。用于在界面（如模组列表、模组不匹配）中表示模组。                                                                                                                                                                        | `displayName="Example Mod"`                                     |
| `description`    | 字符串   | `'''MISSING DESCRIPTION'''`  | 模组列表界面显示的模组描述。推荐使用[**多行字面字符串**(`multiline literal string`)][multiline]。此值可翻译，详见[**翻译模组元数据**(`Translating Mod Metadata`)][i18n]。                                                                | `description='''这是一个示例。'''`                         |
| `logoFile`       | 字符串   | _无_                    | 模组列表界面使用的图片文件名及扩展名。路径必须是从 JAR 或源码集根目录（如主源码集的`src/main/resources`）开始的绝对路径。有效字符为小写字母（`a-z`）、数字（`0-9`）、斜杠（`/`）、下划线（`_`）、点（`.`）和连字符（`-`）。完整字符集为`[a-z0-9_\-.]`。                                                                  | `logoFile="test/example_logo.png"`                              |
| `logoBlur`       | 布尔值  | `true`                       | 渲染`logoFile`时使用`GL_LINEAR*`（true）还是`GL_NEAREST*`（false）。简言之，缩放时是否模糊徽标。                                                                                    | `logoBlur=false`                                                |
| `updateJSONURL`  | 字符串   | _无_                    | [**更新检查器**(`update checker`)][update]用于确认当前模组是否最新版本的 JSON URL。                                                                                                                                                               | `updateJSONURL="https://example.github.io/update_checker.json"` |
| `features`       | 表    | `{}`                         | 详见[**功能特性**(`features`)][features]。                                                                                                                                                                                                                                                                | `features={java_version="[17,)"}`                               |
| `modproperties`  | 表    | `{}`                         | 关联此模组的键/值表。NeoForge 未使用，主要供模组使用。                                                                                                                                                                             | `modproperties={example="value"}`                               |
| `modUrl`         | 字符串   | _无_                    | 模组下载页面的 URL。当前未使用。                                                                                                                                                                                                                       | `modUrl="https://neoforged.net/"`                               |
| `credits`        | 字符串   | _无_                    | 模组列表界面显示的模组致谢信息。                                                                                                                                                                                                             | `credits="此处和彼处的人员。"`                     |
| `authors`        | 字符串   | _无_                    | 模组列表界面显示的模组作者。                                                                                                                                                                                                                           | `authors="示例人员"`                                      |
| `displayURL`     | 字符串   | _无_                    | 模组列表界面显示的模组展示页面 URL。                                                                                                                                                                                                             | `displayURL="https://neoforged.net/"`                           |
| `enumExtensions` | 字符串   | _无_                    | [**枚举扩展**(`enum extension`)][enumextension]使用的 JSON 文件路径。                                                                                                                                                                                                          | `enumExtensions="META_INF/enumextensions.json"`                 |
| `featureFlags`   | 字符串   | _无_                    | [**特性标志**(`feature flags`)][featureflags]使用的 JSON 文件路径。                                                                                                                                                                                                            | `featureFlags="META-INF/feature_flags.json"`                    |

#### **功能特性**(`Features`)

功能特性系统允许模组在加载时要求特定设置、软件或硬件可用。当功能不满足时，模组加载将失败并通知用户要求。当前 NeoForge 提供以下功能：

| 功能特性          | 描述                                                                                                                                                                                                | 示例                             |
|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| `javaVersion`   | Java 版本的可接受范围（以[**Maven 版本范围**(`Maven Version Range`)][mvr]表示）。应为 Minecraft 支持的 Java 版本。                                                       | `features={javaVersion="[17,)"}`  |

### **访问转换器特定属性**(`Access Transformer-Specific Properties`)

[**访问转换器特定属性**(`Access Transformer-specific properties`)][accesstransformer]通过`[[accessTransformers]]`标头绑定到指定访问转换器。这是[**表数组**(`array of tables`)][array]；所有键/值属性将附加到该访问转换器直至下一标头。此标头可选；但若指定，所有元素均为必需项。

| 属性 |  类型  |    默认值    |             描述              |     示例      |
|:--------:|:------:|:-------------:|:------------------------------------:|:----------------|
| `file`   | 字符串 | **必需** | 详见[**添加 AT**(`Adding ATs`)][accesstransformer]。 | `file="at.cfg"` |

### **Mixin 配置属性**(`Mixin Configuration Properties`)

[**Mixin 配置属性**(`Mixin Configuration Properties`)][mixinconfig]通过`[[mixins]]`标头绑定到指定 Mixin 配置。这是[**表数组**(`array of tables`)][array]；所有键/值属性将附加到该 Mixin 块直至下一标头。此标头可选；但若指定，所有元素均为必需项。

| 属性 |  类型  |    默认值    |             描述                       |     示例                       |
|:--------:|:------:|:-------------:|:---------------------------------------------:|:----------------------------------|
| `config` | 字符串 | **必需** | Mixin 配置文件的位置。 | `config="examplemod.mixins.json"` |

### **依赖配置**(`Dependency Configurations`)

模组可指定依赖项，由 NeoForge 在加载模组前检查。这些配置通过[**表数组**(`array of tables`)][array]`[[dependencies.<modid>]]`创建，其中`modid`是消费依赖项的模组标识符。

| 属性       | 类型    | 默认值        | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                | 示例                                      |
|----------------|---------|----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| `modId`        | 字符串  | **必需**  | 作为依赖项添加的模组标识符。                                                                                                                                                                                                                                                                                                                                                                                                                           | `modId="jei"`                                |
| `type`         | 字符串  | `"required"`   | 指定依赖性质：`"required"`（默认）表示依赖缺失时阻止模组加载；`"optional"`表示依赖缺失时不阻止加载，但仍验证兼容性；`"incompatible"`表示依赖存在时阻止加载；`"discouraged"`表示依赖存在时仍允许加载，但向用户显示警告。 | `type="incompatible"`                        |
| `reason`       | 字符串  | _无_      | 面向用户的描述性消息（可选），说明为何需要此依赖或为何不兼容。                                                                                                                                                                                                                                                                                                                                                                    | `reason="集成"`                       |
| `versionRange` | 字符串  | `""`           | 语言加载器的可接受版本范围（以[**Maven 版本范围**(`Maven Version Range`)][mvr]表示）。空字符串匹配任意版本。                                                                                                                                                                                                                                                                                                                                       | `versionRange="[1, 2)"`                      |
| `ordering`     | 字符串  | `"NONE"`       | 定义模组必须在此依赖项之前（`"BEFORE"`）还是之后（`"AFTER"`）加载。若顺序无关，设为`"NONE"`。                                                                                                                                                                                                                                                                                                                                    | `ordering="AFTER"`                           |
| `side`         | 字符串  | `"BOTH"`       | 依赖项必须存在的[**物理端**(`physical side`)][sides]：`"CLIENT"`、`"SERVER"`或`"BOTH"`。                                                                                                                                                                                                                                                                                                                                                                         | `side="CLIENT"`                              |
| `referralUrl`  | 字符串  | _无_      | 依赖项下载页面的 URL。当前未使用。                                                                                                                                                                                                                                                                                                                                                                                                            | `referralUrl="https://library.example.com/"` |

:::danger
两个模组的`ordering`可能导致循环依赖崩溃。例如，模组 A 需在模组 B`"之前"`加载，而模组 B 需在模组 A`"之前"`加载。
:::

## **模组入口点**(`Mod Entrypoints`)

完成`neoforge.mods.toml`后，需提供模组入口点。入口点本质上是模组执行的起点。入口点本身由`neoforge.mods.toml`中使用的语言加载器确定。

### `javafml` 与 `@Mod`

`javafml`是 NeoForge 为 Java 编程语言提供的语言加载器。入口点通过带`@Mod`注解的公共类定义。`@Mod`的值必须包含`neoforge.mods.toml`中指定的模组 ID 之一。所有初始化逻辑（如[**注册事件**(`registering events`)][events]或[**添加 `DeferredRegister`**(`adding DeferredRegister`s)][registration]）可在类的构造函数中指定。

主模组类只能有一个公共构造函数，否则抛出`RuntimeException`。构造函数可包含以下任意参数（任意顺序）；无显式必需参数，但禁止重复参数。

参数类型     | 描述                                                                                              |
------------------|----------------------------------------------------------------------------------------------------------|
`IEventBus`       | [**模组特定事件总线**(`mod-specific event bus`)][modbus]（用于注册、事件等）                             |
`ModContainer`    | 保存此模组元数据的抽象容器                                                       |
`FMLModContainer` | 由`javafml`定义的保存此模组元数据的实际容器（`ModContainer`的扩展） |
`Dist`            | 模组加载的[**物理端**(`physical side`)][sides]                                                        |

```java
@Mod("examplemod") // 必须匹配 neoforge.mods.toml 中的模组 ID
public class ExampleMod {
    // 有效构造函数，仅使用两种可用参数类型
    public ExampleMod(IEventBus modBus, ModContainer container) {
        // 在此初始化逻辑
    }
}
```

默认情况下，`@Mod`注解在[**双端**(`sides`)][sides]加载。可通过`dist`参数更改：

```java
// 必须匹配 neoforge.mods.toml 中的模组 ID
// 此模组类仅在物理客户端加载
@Mod(value = "examplemod", dist = Dist.CLIENT) 
public class ExampleModClient {
    // 有效构造函数
    public ExampleModClient(FMLModContainer container, IEventBus modBus, Dist dist) {
        // 在此初始化仅客户端逻辑
    }
}
```

:::note
`neoforge.mods.toml`中的条目无需对应`@Mod`注解。同理，一个条目可有多个`@Mod`注解（例如分离通用逻辑和仅客户端逻辑）。
:::

[accesstransformer]: ../advanced/accesstransformers.md#添加访问转换器
[array]: https://toml.io/zh/v1.0.0#数组表格
[atlasviewer]: https://github.com/XFactHD/AtlasViewer/blob/1.20.2/neoforge/src/main/resources/META-INF/services/xfacthd.atlasviewer.platform.services.IPlatformHelper
[events]: ../concepts/events.md
[features]: #功能特性
[group]: #组-id
[i18n]: ../resources/client/i18n.md#翻译模组元数据
[javafml]: #javafml-与-mod
[jei]: https://www.curseforge.com/minecraft/mc-mods/jei
[mcversioning]: versioning.md#minecraft
[mdkgradleproperties]: https://github.com/NeoForgeMDKs/MDK-1.21.6-NeoGradle/blob/main/gradle.properties
[mdkneoforgemodstoml]: https://github.com/NeoForgeMDKs/MDK-1.21.6-NeoGradle/blob/main/src/main/resources/META-INF/neoforge.mods.toml
[neoforgemodstoml]: #neoforgemodstoml
[mixinconfig]: https://github.com/SpongePowered/Mixin/wiki/Introduction-to-Mixins---The-Mixin-Environment#mixin-配置文件
[modbus]: ../concepts/events.md#事件总线
[modid]: #模组-id
[multiline]: https://toml.io/zh/v1.0.0#字符串
[mvr]: https://maven.apache.org/enforcer/enforcer-rules/versionRanges.html
[neoversioning]: versioning.md#neoforge
[packaging]: structuring.md#包结构
[registration]: ../concepts/registries.md#deferredregister
[resource]: ../resources/index.md
[serviceload]: https://docs.oracle.com/zh-cn/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html#load(java.lang.Class)
[sides]: ../concepts/sides.md
[spdx]: https://spdx.org/licenses/
[toml]: https://toml.io/zh/
[update]: ../misc/updatechecker.md
[uses]: https://docs.oracle.com/javase/specs/jls/se21/html/jls-7.html#jls-7.7.3
[versioning]: versioning.md
[enumextension]: ../advanced/extensibleenums.md
[featureflags]: ../advanced/featureflags.md