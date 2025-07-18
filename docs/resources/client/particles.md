# **粒子**(`Particles`)

粒子是用于提升游戏表现和沉浸感的 2D 特效。可在客户端和服务器[端][side]生成，但因本质是视觉特效，关键部分仅存在于物理（和逻辑）客户端。

## 注册粒子

### `ParticleType`

粒子通过 `ParticleType` 注册（类似 `EntityType` 或 `BlockEntityType`）：
- `Particle` 类：每个生成粒子都是其实例
- `ParticleType` 类：保存公共信息用于注册
- `ParticleType` 是注册表(`registry`)，需用 `DeferredRegister` 注册

```java
public class MyParticleTypes {
    // 假设模组 ID 为 examplemod
    public static final DeferredRegister<ParticleType<?>> PARTICLE_TYPES =
        DeferredRegister.create(BuiltInRegistries.PARTICLE_TYPE, "examplemod");
    
    // 最简单方式：复用原版 SimpleParticleType
    public static final Supplier<SimpleParticleType> MY_PARTICLE = PARTICLE_TYPES.register(
        // 粒子类型名称
        "my_particle",
        // 供应商。布尔参数表示视频设置中的“粒子”选项设为“最少”时是否影响此粒子类型
        // 原版粒子多为 false，但爆炸/篝火烟雾/墨汁等为 true
        () -> new SimpleParticleType(false)
    );
}
```

:::info
仅在服务端操作粒子时需要 `ParticleType`，客户端可直接使用 `Particle`。
:::

### `Particle`

`Particle` 是生成到世界中并显示给玩家的实体。通常扩展 `TextureSheetParticle` 而非 `Particle`，因其提供动画/缩放等辅助功能并处理渲染逻辑。

```java
public class MyParticle extends TextureSheetParticle {
    private final SpriteSet spriteSet;
    
    // 前四个参数自解释。SpriteSet 由 ParticleProvider 提供（见下文）
    public MyParticle(ClientLevel level, double x, double y, double z, SpriteSet spriteSet) {
        super(level, x, y, z);
        this.spriteSet = spriteSet;
        this.gravity = 0; // 粒子悬停空中

        // 初始精灵设置（确保渲染前有精灵）
        this.setSpriteFromAge(spriteSet);
    }
    
    @Override
    public void tick() {
        // 根据粒子年龄设置精灵（推进动画）
        this.setSpriteFromAge(spriteSet);
        // 调用父类处理移动（可重写 move() 自定义移动）
        super.tick();
    }
}
```

### `ParticleProvider`

`ParticleProvider` 是仅客户端的类，通过 `createParticle` 方法创建 `Particle`：

```java
// ParticleProvider 的泛型需匹配粒子类型
public class MyParticleProvider implements ParticleProvider<SimpleParticleType> {
    private final SpriteSet spriteSet;
    
    // 注册函数传入 SpriteSet
    public MyParticleProvider(SpriteSet spriteSet) {
        this.spriteSet = spriteSet;
    }
    
    // 每次调用返回新粒子
    @Override
    public Particle createParticle(SimpleParticleType type, ClientLevel level,
            double x, double y, double z, double xSpeed, double ySpeed, double zSpeed) {
        // 未使用 type 和速度参数（按需使用）
        return new MyParticle(level, x, y, z, spriteSet);
    }
}
```

在客户端[模组总线(`mod bus`)][event]事件 `RegisterParticleProvidersEvent` 中关联粒子类型和提供器：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerParticleProviders(RegisterParticleProvidersEvent event) {
    // #registerSpriteSet 表示 Function<SpriteSet, ParticleProvider<?>>
    event.registerSpriteSet(MyParticleTypes.MY_PARTICLE.get(), MyParticleProvider::new);
    // 其他方法：#registerSprite（Supplier<TextureSheetParticle>）和 #registerSpecial（Supplier<Particle>）
}
```

### 粒子描述文件

需将粒子类型关联纹理（类似物品模型）。**粒子描述文件**(`Particle Descriptions`)是 JSON 文件，路径为 `assets/<命名空间>/particles/粒子类型名称.json`（如 `my_particle.json`）。格式如下：

```json5
{
    // 按顺序播放的纹理列表（会循环）
    // 纹理路径相对于 textures/particle
    "textures": [
        "examplemod:my_particle_0",
        "examplemod:my_particle_1",
        "examplemod:my_particle_2",
        "examplemod:my_particle_3"
    ]
}
```

:::danger
粒子描述文件与提供器不匹配（如描述文件无对应工厂或反之）会抛出异常！
:::

:::note
仅当 `Particle#getRenderType` 返回的 `ParticleRenderType` 使用 `TextureAtlas#LOCATION_PARTICLES` 作为着色器纹理时，粒子描述文件才生效。原版粒子渲染类型：`PARTICLE_SHEET_OPAQUE` 和 `PARTICLE_SHEET_TRANSLUCENT`。
:::

### 数据生成

可通过扩展 `ParticleDescriptionProvider` 并重写 `#addDescriptions()` [数据生成][datagen]粒子描述文件：

```java
public class MyParticleDescriptionProvider extends ParticleDescriptionProvider {
    // 从 GatherDataEvent.Client 获取参数
    public MyParticleDescriptionProvider(PackOutput output) {
        super(output);
    }

    // 假设所有引用粒子已存在（替换 examplemod 为你的模组 ID）
    @Override
    protected void addDescriptions() {
        // 添加单精灵粒子描述（纹理路径 assets/examplemod/textures/particle/my_single_particle.png）
        sprite(MyParticleTypes.MY_SINGLE_PARTICLE.get(), ResourceLocation.fromNamespaceAndPath("examplemod", "my_single_particle"));
        // 添加多精灵粒子描述（可变参数，也可接收列表）
        spriteSet(MyParticleTypes.MY_MULTI_PARTICLE.get(),
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle_0"),
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle_1"),
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle_2")
        );
        // 替代方案：为指定数量的纹理添加 "_<索引>" 后缀
        spriteSet(MyParticleTypes.MY_ALT_MULTI_PARTICLE.get(),
            // 基础名称
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle"),
            // 纹理数量
            3,
            // 是否反转列表（从末尾开始）
            false
        );
    }
}
```

在 `GatherDataEvent.Client` 中添加提供器：

```java
@SubscribeEvent // 在模组事件总线
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(MyParticleDescriptionProvider::new);
}
```

### 自定义 `ParticleType`

`SimpleParticleType` 通常足够，但服务端附加数据时需自定义 `ParticleType` 和关联的 `ParticleOptions`：

```java
// ParticleOptions 实现
public class MyParticleOptions implements ParticleOptions {
    // 命令读写信息（无信息时为空字符串）
    public static final MapCodec<MyParticleOptions> CODEC = MapCodec.unit(new MyParticleOptions());
    // 网络缓冲区读写
    public static final StreamCodec<ByteBuf, MyParticleOptions> STREAM_CODEC = StreamCodec.unit(new MyParticleOptions());

    public MyParticleOptions() {} // 可定义必要字段

    @Override
    public ParticleType<?> getType() {
        // 返回注册的粒子类型
    }
}

// 自定义 ParticleType
public class MyParticleType extends ParticleType<MyParticleOptions> {
    // overrideLimiter 参数：是否在低粒子设置下限制
    public MyParticleType(boolean overrideLimiter) {
        super(overrideLimiter);
    }

    @Override
    public MapCodec<MyParticleOptions> codec() {
        return MyParticleOptions.CODEC;
    }

    @Override
    public StreamCodec<? super RegistryFriendlyByteBuf, MyParticleOptions> streamCodec() {
        return MyParticleOptions.STREAM_CODEC;
    }
}

// 注册
public static final Supplier<MyParticleType> MY_CUSTOM_PARTICLE = PARTICLE_TYPES.register(
    "my_custom_particle",
    () -> new MyParticleType(false)
);

// ParticleOptions 中返回粒子类型
public class MyParticleOptions implements ParticleOptions {
    @Override
    public ParticleType<?> getType() {
        return MY_CUSTOM_PARTICLE.get();
    }
}
```

## 生成粒子

服务端仅识别 `ParticleType` 和 `ParticleOption`，客户端通过 `ParticleProvider` 处理 `Particle`，因此生成方式因端而异：

- **通用代码**：调用 `Level#addParticle` 或 `Level#addAlwaysVisibleParticle`（推荐，所有人可见）
- **客户端代码**：使用通用代码方式，或直接创建 `new Particle()` 并通过 `Minecraft.getInstance().particleEngine#add(Particle)` 添加（仅当前客户端可见）
- **服务端代码**：调用 `ServerLevel#sendParticles`（原版 `/particle` 命令使用此方式）

[datagen]: ../index.md#数据生成
[event]: ../../concepts/events.md
[modbus]: ../../concepts/events.md#事件总线
[registry]: ../../concepts/registries.md#注册方法
[side]: ../../concepts/sides.md