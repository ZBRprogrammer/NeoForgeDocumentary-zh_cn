# NeoForge 入门指南

本节包含如何设置 NeoForge 工作区，以及如何运行和测试模组的信息。

## 先决条件

- 熟悉 Java 编程语言，特别是其面向对象、多态、泛型和函数式特性。
- 安装 Java 21 **开发工具包**（`Development Kit, JDK`）和 64 位 **Java 虚拟机**（`Java Virtual Machine, JVM`）。NeoForge 推荐并官方支持[**Microsoft OpenJDK 构建版**][jdk]，但其他 JDK 也可使用。

:::caution
请确保使用 64 位 JVM。可通过终端运行`java -version`验证。Minecraft 不支持 32 位 JVM。
:::

- 熟悉所选**集成开发环境**（`Integrated Development Environment, IDE`）。
      - NeoForge 官方支持 [**IntelliJ IDEA**][intellij] 和 [**Eclipse**][eclipse]，两者均内置 Gradle 支持。但也可使用任何 IDE（如 Netbeans、Visual Studio Code、Vim 或 Emacs）。
- 熟悉 [**Git**][git] 和 [**GitHub**][github]。技术上非必需，但会大幅简化工作流程。

## 设置工作区

- 访问 [**模组生成器**(`Mod Generator`)][modgen] 网页，输入模组名称（可选填模组 ID）、包名、Minecraft 版本和 Gradle 插件（[**ModDevGradle**][mdg] 或 [**NeoGradle**][ng]），点击"下载模组项目"，解压下载的 ZIP 文件。
- 在 IDE 中导入 Gradle 项目。Eclipse 和 IntelliJ IDEA 会自动完成。若 IDE 不支持，可通过`gradlew`终端命令导入。
      - 首次操作时，Gradle 会下载 NeoForge 所有依赖（包括 Minecraft 本身）并反编译。可能耗时较长（根据硬件和网络状况，最长可达一小时）。
      - 修改 Gradle 文件后，需通过 IDE 的"重新加载 Gradle"按钮或`gradlew`终端命令重新加载 Gradle 变更。

## 自定义模组信息

模组的基本属性可在`gradle.properties`文件中修改，包括模组名称和版本等基础信息。更多详情请参阅`gradle.properties`文件内注释或[**gradle.properties 文件文档**][properties]。

如需进一步修改构建流程，可编辑`build.gradle`和`settings.gradle`文件。NeoForge 提供的 Gradle 插件（[**ModDevGradle**][mdg] 或 [**NeoGradle**][ng]）提供多项配置选项，部分在构建脚本中以注释说明。

:::caution
仅当明确操作目的时才编辑`build.gradle`和`settings.gradle`文件。所有基础属性均可通过`gradle.properties`设置。
:::

## 构建与测试模组

运行`gradlew build`构建模组。将在`build/libs`目录生成`<archivesBaseName>-<version>.jar`文件。`<archivesBaseName>`和`<version>`由`build.gradle`设定，默认分别对应`gradle.properties`中的`mod_id`和`mod_version`值（可在`build.gradle`中修改）。生成的 JAR 文件可放入支持 NeoForge 的 Minecraft 实例的`mods`文件夹，或上传至模组分发平台。

要在测试环境运行模组，可使用生成的运行配置或相关任务（如`gradlew runClient`）。这将从对应运行目录（如`runs/client`或`runs/server`）启动 Minecraft，并应用所有指定源码集。默认 MDK 包含`main`源码集，因此`src/main/java`中的代码均会生效。

### 服务器测试

若通过运行配置或`gradlew runServer`启动**专用服务器**（`dedicated server`），服务器会立即关闭。需编辑运行目录中的`eula.txt`文件接受 Minecraft EULA。

接受后，服务器将加载并在`localhost`（默认`127.0.0.1`）可用。但仍无法加入，因为服务器默认开启**在线模式**（`online mode`），需要身份验证（开发环境玩家不具备）。解决方法：停止服务器，在`server.properties`文件中将`online-mode`设为`false`。重启服务器即可连接。

:::tip
务必在专用服务器环境测试模组。这包括[**纯客户端模组**](`client-only mods`)[client]，因其在服务器加载时应无任何操作。
:::

[client]: ../concepts/sides.md
[eclipse]: https://www.eclipse.org/downloads/
[git]: https://www.git-scm.com/
[github]: https://github.com/
[intellij]: https://www.jetbrains.com/idea/
[jdk]: https://learn.microsoft.com/en-us/java/openjdk/download#openjdk-21
[mdg]: https://github.com/neoforged/ModDevGradle
[modgen]: https://neoforged.net/mod-generator/
[ng]: https://github.com/neoforged/NeoGradle
[properties]: modfiles.md#gradleproperties