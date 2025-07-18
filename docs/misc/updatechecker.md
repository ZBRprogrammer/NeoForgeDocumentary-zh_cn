# **NeoForge 更新检查器**(`NeoForge Update Checker`)

NeoForge 提供轻量级、可选的更新检查框架。当模组有可用更新时，主菜单和模组列表的"模组"按钮将显示闪烁图标及更新日志。**不会**自动下载更新。

## 快速开始

在 `mods.toml` 文件中指定 `updateJSONURL` 参数，值为指向更新 JSON 文件的有效 URL。文件可托管于自有服务器、GitHub 等任意位置，需确保所有用户可访问。

## 更新 JSON 格式

JSON 格式如下：

```json5
{
    "homepage": "<模组主页/下载页>",
    "<mcversion>": {
        "<modversion>": "<此版本更新日志>", 
        // 列出此Minecraft版本的所有模组版本及更新日志
        // ...
    },
    "promos": {
        "<mcversion>-latest": "<modversion>",
        // 声明此Minecraft版本的最新"前沿"版本
        "<mcversion>-recommended": "<modversion>",
        // 声明此Minecraft版本的最新"稳定"版本
        // ...
    }
}
```

注意事项：
- `homepage` 链接在模组过时展示给用户
- NeoForge 内部算法比较版本号新旧，推荐遵循 [Maven 版本规范][mvnver]
- 更新日志用 `\n` 分隔行（可简写日志并外链详情）
- 可配置 `build.gradle` 自动生成此文件（Groovy 原生支持 JSON 解析）
- 参考示例：[nocubes]、[Corail Tombstone][corail]、[Chisels & Bits 2][chisel]

## 获取更新检查结果

通过 `VersionChecker#getResult(IModInfo)` 获取结果：
- 通过模组构造函数参数获取 `ModContainer`
- 通过 `ModContainer#getModInfo` 获取 `IModInfo`
- 其他模组通过 `ModList.get().getModContainerById(<modId>)` 获取

返回对象的 `#status` 方法表示检查状态：

| 状态              | 描述                          |
|:----------------- |:-----------------------------|
| `FAILED`         | 无法连接提供的 URL            |
| `UP_TO_DATE`     | 当前版本等于推荐版本          |
| `AHEAD`          | 无最新版本时当前版本高于推荐版 |
| `OUTDATED`       | 存在新的推荐版或最新版        |
| `BETA_OUTDATED`  | 存在新的最新版                |
| `BETA`           | 当前版本等于或高于最新版      |
| `PENDING`        | 结果未就绪（需稍后重试）      |

返回对象同时包含目标版本及 `update.json` 中指定的更新日志。

[mvnver]: ../gettingstarted/versioning.md
[nocubes]: https://cadiboo.github.io/projects/nocubes/update.json
[corail]: https://github.com/Corail31/tombstone_lite/blob/master/update.json
[chisel]: https://github.com/Aeltumn/Chisels-and-Bits-2/blob/master/update.json