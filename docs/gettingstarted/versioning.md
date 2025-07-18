# **版本规范**(`Versioning`)

本文解析 Minecraft 和 NeoForge 的版本机制，并提供模组版本规范建议。

## **Minecraft**(`Minecraft`)

Minecraft 采用[**语义化版本**(`semantic versioning`)][semver]。语义化版本（简称"semver"）格式为`主版本.次版本.修订号`。例如，Minecraft 1.20.2 的主版本为 1，次版本为 20，修订号为 2。

自 2011 年 Minecraft 1.0 发布以来，主版本始终为`1`。此前版本规范多变，存在`a1.1`（Alpha 1.1）、`b1.7.3`（Beta 1.7.3）甚至不遵循明确规范的`infdev`版本。因主版本`1`已沿用十余年，且存在"Minecraft 2"的梗，普遍认为此值永不改变。

### **快照版**(`Snapshots`)

快照版偏离标准 semver 规范。标记为`YYwWWa`，其中`YY`代表年份后两位（如`23`），`WW`代表该年第几周（如`01`）。例如，快照`23w01a`是 2023 年第一周发布的快照。

`a`后缀用于同一周发布两个快照的情况（第二个快照命名为`23w01b`）。Mojang 过去偶尔使用此规则。替代后缀也曾用于特殊快照（如`20w14infinite`即[**2020 无限维度愚人节彩蛋**(`2020 infinite dimensions April Fool's joke`)][infinite]。

### **预发布版与候选版**(`Pre-releases and Release Candidates`)

快照周期接近完成时，Mojang 发布预发布版。预发布版被视为功能完整版本，仅专注于修复错误。采用 semver 规范并后缀`-preX`。例如，1.20.2 的首个预发布版命名为`1.20.2-pre1`。通常存在多个预发布版，后缀依次为`-pre2`、`-pre3`等。

类似地，预发布周期结束后，Mojang 发布候选版 1（版本后缀`-rc1`，如`1.20.2-rc1`）。Mojang 目标是一个候选版即可发布（若无后续错误）。但若出现意外错误，也可能发布`-rc2`、`-rc3`等版本（类似预发布版）。

## **NeoForge**(`NeoForge`)

NeoForge 采用改进的 semver 系统：主版本对应 Minecraft 次版本，次版本对应 Minecraft 修订号，修订号对应"实际"NeoForge 版本。例如，NeoForge 20.2.59 是 Minecraft 1.20.2 的第 60 个版本（从 0 开始）。开头的`1`被省略（因其极不可能改变），原因见[**上文**(`above`)][minecraft]。

NeoForge 部分位置使用[**Maven 版本范围**(`Maven version ranges`)][mvr]（如[`neoforge.mods.toml`][neoforgemodstoml]文件中的 Minecraft 和 NeoForge 版本范围）。其与 semver 大部分兼容但不完全兼容（例如不识别`pre`标签）。

## **模组**(`Mods`)

无绝对最佳版本系统。不同开发风格、项目范围等均影响版本系统选择。有时也可组合使用版本系统。本节概述常用版本系统及实例。

通常模组文件名形如`modid-<版本>.jar`。若模组 ID 为`examplemod`且版本为`1.2.3`，则文件名为`examplemod-1.2.3.jar`。

:::note
版本系统是建议而非严格规则。尤其涉及版本变更（"升级"）时机和方式时。若使用不同系统，无人会阻止。
:::

### **语义化版本**(`Semantic Versioning`)

语义化版本（"semver"）包含三部分：`主版本.次版本.修订号`。主版本升级于代码库重大变更时（通常关联重大新功能和错误修复）。次版本升级于引入次要功能时。修订号升级于仅包含错误修复的更新时。

普遍认为`0.x.x`均为开发版本，首个（完整）发布时应升级至`1.0.0`。

"次版本用于功能，修订号用于修复"的规则实践中常被忽视。典型例子是 Minecraft 本身：重大功能通过次版本号实现，次要功能通过修订号实现，错误修复在快照版中完成（见上文）。

根据模组更新频率，这些数字可大可小。例如[**Supplementaries**][supplementaries]（撰写时）版本为`2.6.31`。三位甚至四位数（尤其修订号）完全可能。

### **简化版与扩展版 Semver**(`"Reduced" and "Expanded" Semver`)

有时可见仅两个数字的 semver（"简化版"或"双段式" semver）。其版本号仅含`主版本.次版本`方案。常用于添加少量简单对象的小型模组（除 Minecraft 版本更新外很少更新），常永久停留于`1.0`版本。

"扩展版" semver（"四段式" semver）含四个数字（如`1.0.0.0`）。根据模组不同，格式可为`主版本.API版本.次版本.修订号`或`主版本.次版本.修订号.热修复版`等——无标准方式。

`主版本.API版本.次版本.修订号`中，`主版本`与`API版本`解耦。这意味着`主版本`（功能位）和`API版本`可独立升级。常用于提供 API 供其他模组开发者使用的模组。例如[**Mekanism**][mekanism]（撰写时）版本为 10.4.5.19。

`主版本.次版本.修订号.热修复版`将修订级别拆分为二。[**Create**][create]模组采用此方法（撰写时）版本为 0.5.1f。注意 Create 用字母而非第四位数字表示热修复版，以保持与常规 semver 兼容。

:::info
简化版 semver、扩展版 semver、双段式 semver 和四段式 semver 均非官方术语或标准化格式。
:::

### **Alpha、Beta、Release 阶段**(`Alpha, Beta, Release`)

与 Minecraft 类似，模组开发常遵循软件工程的经典`alpha`/`beta`/`release`阶段：`alpha`表示不稳定/实验版本（有时称`experimental`或`snapshot`），`beta`表示半稳定版本，`release`表示稳定版本（有时称`stable`）。

部分模组用主版本表示 Minecraft 版本升级。例如[**JEI**][jei]：`13.x.x.x`对应 1.19.2，`14.x.x.x`对应 1.19.4，`15.x.x.x`对应 1.20.1（无 1.19.3 和 1.20.0 版本）。其他模组将标签附加到模组名后，例如[**Minecolonies**][minecolonies]模组（撰写时）版本为`1.1.328-BETA`。

### **包含 Minecraft 版本**(`Including the Minecraft Version`)

模组文件名通常包含其适用的 Minecraft 版本，便于用户识别。常见位置是模组版本前后（前者更普遍）。例如，JEI 1.20.2 版本`16.0.0.28`（撰写时最新版）可命名为`jei-1.20.2-16.0.0.28`或`jei-16.0.0.28-1.20.2`。

### **包含模组加载器**(`Including the Mod Loader`)

NeoForge 非唯一模组加载器，许多模组开发者跨平台开发。因此需区分同一模组同一版本但不同加载器的文件。

通常通过在名称中包含模组加载器实现。`jei-neoforge-1.20.2-16.0.0.28`、`jei-1.20.2-neoforge-16.0.0.28`或`jei-1.20.2-16.0.0.28-neoforge`均为有效方式。其他加载器将`neoforge`替换为`forge`、`fabric`、`quilt`等。

### **关于 Maven 的说明**(`A Note on Maven`)

**Maven**（依赖托管系统）使用的版本系统与 semver 部分细节不同（但`主版本.次版本.修订号`模式相同）。相关[**Maven 版本范围系统**(`Maven Versioning Range, MVR`)][mvr]用于 NeoForge 部分位置（见[**上文**(`above`)][neoforge]）。选择版本系统时应确保兼容 MVR，否则其他模组无法依赖特定版本！

[create]: https://www.curseforge.com/minecraft/mc-mods/create
[infinite]: https://minecraft.wiki/w/Java_Edition_20w14∞
[jei]: https://www.curseforge.com/minecraft/mc-mods/jei
[mekanism]: https://www.curseforge.com/minecraft/mc-mods/mekanism
[minecolonies]: https://www.curseforge.com/minecraft/mc-mods/minecolonies
[minecraft]: #minecraft
[neoforgemodstoml]: modfiles.md#neoforgemodstoml
[mvr]: https://maven.apache.org/enforcer/enforcer-rules/versionRanges.html
[mvr]: https://maven.apache.org/ref/3.5.2/maven-artifact/apidocs/org/apache/maven/artifact/versioning/ComparableVersion.html
[neoforge]: #neoforge
[pre]: #预发布版
[rc]: #候选版
[semver]: https://semver.org/
[supplementaries]: https://www.curseforge.com/minecraft/mc-mods/supplementaries