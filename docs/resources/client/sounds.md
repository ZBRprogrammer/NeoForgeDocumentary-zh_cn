# **音效**(`Sounds`)

音效能显著增强模组的细节感和生动性。Minecraft 提供多种注册和播放音效的方式。

## 术语

Minecraft 音效引擎使用以下术语：
- **音效事件**(`Sound event`)：触发音效引擎播放特定音效的代码触发器。`SoundEvent` 是注册到游戏的对象。
- **音效类别**(`Sound category`) 或 **音效源**(`sound source`)：可单独开关的音效分组（如 `master`, `block`, `player`）。代码中对应 `SoundSource` 枚举。
- **音效定义**(`Sound definition`)：音效事件到音效对象的映射（含元数据）。位于命名空间的 [`sounds.json` 文件][soundsjson]。
- **音效对象**(`Sound object`)：包含音效文件路径和元数据的 JSON 对象。
- **音效文件**(`Sound file`)：磁盘上的音效文件（仅支持 `.ogg` 格式）。

:::danger
由于 OpenAL（Minecraft 的音频库）实现，要使音效随玩家距离衰减：
1. 音效文件必须是**单声道**(`mono`)（单通道）
2. **立体声**(`stereo`)（多通道）文件不会衰减，始终在玩家位置播放（适合环境音和背景音乐）
参见 [MC-146721][bug]。
:::

## 创建 `SoundEvent`

`SoundEvent` 是[注册对象][registration]，需通过 `DeferredRegister` 注册为单例：

```java
public class MySoundsClass {
    // 假设模组 ID 为 examplemod
    public static final DeferredRegister<SoundEvent> SOUND_EVENTS =
            DeferredRegister.create(BuiltInRegistries.SOUND_EVENT, "examplemod");
    
    // 原版音效使用可变范围事件
    public static final Holder<SoundEvent> MY_SOUND = SOUND_EVENTS.register(
            "my_sound",
            // 传入注册名
            SoundEvent::createVariableRangeEvent
    );
    
    // 当前未使用的固定范围（不衰减）事件注册方法：
    public static final Holder<SoundEvent> MY_FIXED_SOUND = SOUND_EVENTS.register(
            "my_fixed_sound",
            // 16 是音效默认范围（OpenAL 限制：超过 16 的值无效）
            registryName -> SoundEvent.createFixedRangeEvent(registryName, 16f)
    );
}
```

在[模组构造器][modctor]中将注册表添加到[模组事件总线(`mod bus`)][modbus]：

```java
public ExampleMod(IEventBus modBus) {
    MySoundsClass.SOUND_EVENTS.register(modBus);
    // 其他初始化
}
```

至此音效事件创建完成！

## `sounds.json`

_另见：[Minecraft Wiki 上的 sounds.json][mcwikisounds]_

通过音效定义文件关联音效事件和音效文件。命名空间的所有音效定义存储在 `sounds.json` 文件中（位于命名空间根目录）。每个音效定义将音效事件 ID（如 `my_sound`）映射到 JSON 音效对象。示例：

```json5
{
    // 音效事件 "examplemod:my_sound" 的定义
    "my_sound": {
        // 音效对象列表（多个元素时随机选择）
        "sounds": [
            // 仅 name 必填
            {
                // 音效文件路径（相对于命名空间的 sounds 文件夹）
                // 示例路径：assets/examplemod/sounds/sound_1.ogg
                "name": "examplemod:sound_1",
                // 类型："sound"（音效文件）或 "event"（其他音效事件）。默认 "sound"
                "type": "sound",
                // 音量（0.0-1.0，默认 1.0）
                "volume": 0.8,
                // 音高（0.0-2.0，默认 1.0）
                "pitch": 1.1,
                // 权重（随机选择概率，默认 1）
                "weight": 3,
                // 是否流式加载（长音效推荐，默认 false）
                "stream": true,
                // 衰减距离覆盖（默认 16，固定范围音效事件忽略）
                "attenuation_distance": 8,
                // 是否在资源包加载时预加载（默认 false）
                "preload": true
            },
            // { "name": "examplemod:sound_2" } 的简写
            "examplemod:sound_2"
        ]
    },
    "my_fixed_sound": {
        // 是否替换下层资源包音效（见合并章节）
        "replace": true,
        // 触发时显示的字幕翻译键
        "subtitle": "examplemod.my_fixed_sound",
        "sounds": [
            "examplemod:sound_1",
            "examplemod:sound_2"
        ]
    }
}
```

### 合并

与其他资源文件不同，`sounds.json` 不覆盖下层资源包的值，而是合并后解释。示例：
- **RP1**（上层包）和 **RP2**（下层包）定义音效 `sound_1` 到 `sound_4`
- 合并规则：
  - 双方 `replace: false`：下层包 + 上层包
  - 上层 `replace: true`：仅上层包
  - 下层 `replace: true`：下层包 + 上层包（仍丢弃更下层包）
  - 双方 `replace: true`：仅上层包

## 播放音效

Minecraft 提供多种播放音效的方法（`SoundEvent` 可为自定义或原版）。以下方法描述中，“客户端”和“服务器”指[逻辑端][sides]。

### `Level` 方法

- `playSeededSound(Entity entity, double x, double y, double z, Holder<SoundEvent> soundEvent, SoundSource soundSource, float volume, float pitch, long seed)`
    - 客户端：若传入玩家是本地玩家，在指定位置播放音效
    - 服务器：向除传入玩家外的所有玩家发送播放数据包
    - 用途：客户端发起且在双端运行的代码。服务器跳过发起玩家避免重复播放
- `playSound(...)` 系列方法：转发到 `playSeededSound` 并添加位置简化

### `ClientLevel` 方法

- `playLocalSound(...)`：转发到 `Level#playLocalSound` 并添加位置简化

### `Entity` 方法

- `playSound(...)`：转发到 `Level#playSound`，使用实体位置和音效源

### `Player` 方法

- `playSound(...)`：转发到 `Level#playSound`，指定玩家为发起者

## 数据生成

音效文件本身无法[数据生成][datagen]，但 `sounds.json` 文件可以。扩展 `SoundDefinitionsProvider` 并重写 `registerSounds()`：

```java
public class MySoundDefinitionsProvider extends SoundDefinitionsProvider {
    // 从 GatherDataEvent.Client 获取参数
    public MySoundDefinitionsProvider(PackOutput output) {
        // 使用实际模组 ID 替换 examplemod
        super(output, "examplemod");
    }

    @Override
    public void registerSounds() {
        // 第一参数：Supplier<SoundEvent>, SoundEvent 或 ResourceLocation
        add(MySoundsClass.MY_SOUND, SoundDefinition.definition()
            // 添加音效对象（可变参数）
            .with(
                // 第一参数：字符串或 ResourceLocation
                // 第二参数：SOUND 或 EVENT（可省略，默认 SOUND）
                sound("examplemod:sound_1", SoundDefinition.SoundType.SOUND)
                    .volume(0.8f) // 设置音量
                    .pitch(1.2f)  // 设置音高
                    .weight(2)    // 设置权重
                    .attenuationDistance(8) // 设置衰减距离
                    .stream(true)  // 启用流式加载
                    .preload(true), // 启用预加载
                // 最简形式
                sound("examplemod:sound_2")
            )
            .subtitle("sound.examplemod.sound_1") // 设置字幕
            .replace(true) // 启用替换
        );
    }
}
```

在事件中注册提供器：

```java
@SubscribeEvent // 在模组事件总线
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(MySoundDefinitionsProvider::new);
}
```

[bug]: https://bugs.mojang.com/browse/MC-146721
[datagen]: ../index.md#数据生成
[mcwiki]: https://minecraft.wiki
[mcwikisounds]: https://minecraft.wiki/w/Sounds.json
[modbus]: ../../concepts/events.md#事件总线
[modctor]: ../../gettingstarted/modfiles.md#javafml-与-mod-构造器
[registration]: ../../concepts/registries.md
[sides]: ../../concepts/sides.md#逻辑端
[soundsjson]: #soundsjson