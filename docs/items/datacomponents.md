---
sidebar_position: 2
---

# **数据组件**(`Data Components`)

**数据组件**(`Data Components`)是`ItemStack`中用于存储数据的键值对映射。每份数据（如烟花爆炸效果或工具属性）都以实际对象形式存储在堆栈上，使得无需动态转换通用编码实例（如`CompoundTag`、`JsonElement`）即可查看和操作数据值。

## `DataComponentType`

每个数据组件都有关联的`DataComponentType<T>`（其中`T`为组件值类型）。`DataComponentType`代表引用存储组件值的键，以及用于处理磁盘读写和网络传输的编解码器（按需）。

现有组件列表见`DataComponents`。

### 创建自定义数据组件(`Creating Custom Data Components`)

与`DataComponentType`关联的组件值必须实现`hashCode`和`equals`，且在存储时应视为**不可变**(`immutable`)。

:::note
组件值可轻松通过记录类(`record`)实现。记录字段不可变且自动实现`hashCode`和`equals`。
:::

```java
// 记录类示例
public record ExampleRecord(int value1, boolean value2) {}

// 类示例
public class ExampleClass {

    private final int value1;
    // 可变的，但使用时需谨慎
    private boolean value2;

    public ExampleClass(int value1, boolean value2) {
        this.value1 = value1;
        this.value2 = value2;
    }

    @Override
    public int hashCode() {
        return Objects.hash(this.value1, this.value2);
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == this) {
            return true;
        } else {
            return obj instanceof ExampleClass ex
                && this.value1 == ex.value1
                && this.value2 == ex.value2;
        }
    }
}
```

标准`DataComponentType`可通过`DataComponentType#builder`创建，并用`DataComponentType.Builder#build`构建。构建器包含三个设置：`persistent`、`networkSynchronized`、`cacheEncoding`。

`persistent`指定用于读写组件值到磁盘的[`Codec`](`codec`)。`networkSynchronized`指定用于网络传输读写组件的`StreamCodec`。若未指定`networkSynchronized`，则`persistent`提供的`Codec`将被包装为[`StreamCodec`](`streamcodec`)使用。

:::warning
构建器中必须提供`persistent`或`networkSynchronized`，否则会抛出`NullPointerException`。若无数据需网络传输，将`networkSynchronized`设为`StreamCodec#unit`（提供默认组件值）。
:::

`cacheEncoding`缓存`Codec`编码结果，使组件值未更改时后续编码使用缓存值。仅当组件值预期极少或永不更改时使用。

`DataComponentType`是注册对象，必须[注册](`registered`)。

```java
// 使用ExampleRecord(int, boolean)
// 以下仅应使用一个Codec和/或StreamCodec
// 提供多个作示例

// 基础编解码器
public static final Codec<ExampleRecord> BASIC_CODEC = RecordCodecBuilder.create(instance ->
    instance.group(
        Codec.INT.fieldOf("value1").forGetter(ExampleRecord::value1),
        Codec.BOOL.fieldOf("value2").forGetter(ExampleRecord::value2)
    ).apply(instance, ExampleRecord::new)
);
public static final StreamCodec<ByteBuf, ExampleRecord> BASIC_STREAM_CODEC = StreamCodec.composite(
    ByteBufCodecs.INT, ExampleRecord::value1,
    ByteBufCodecs.BOOL, ExampleRecord::value2,
    ExampleRecord::new
);

// 若无数据需网络传输，使用单位流编解码器
public static final StreamCodec<ByteBuf, ExampleRecord> UNIT_STREAM_CODEC = StreamCodec.unit(new ExampleRecord(0, false));


// 在另一类中
// 专用DeferredRegister.DataComponents简化数据组件注册，避免`DataComponentType.Builder`在`Supplier`内的泛型推断问题
public static final DeferredRegister.DataComponents REGISTRAR = DeferredRegister.createDataComponents(Registries.DATA_COMPONENT_TYPE, "examplemod");

public static final Supplier<DataComponentType<ExampleRecord>> BASIC_EXAMPLE = REGISTRAR.registerComponentType(
    "basic",
    builder -> builder
        // 读写数据到磁盘的编解码器
        .persistent(BASIC_CODEC)
        // 网络传输读写数据的编解码器
        .networkSynchronized(BASIC_STREAM_CODEC)
);

/// 组件不保存到磁盘
public static final Supplier<DataComponentType<ExampleRecord>> TRANSIENT_EXAMPLE = REGISTRAR.registerComponentType(
    "transient",
    builder -> builder.networkSynchronized(BASIC_STREAM_CODEC)
);

// 无数据网络同步
public static final Supplier<DataComponentType<ExampleRecord>> NO_NETWORK_EXAMPLE = REGISTRAR.registerComponentType(
   "no_network",
   builder -> builder
        .persistent(BASIC_CODEC)
        // 注意此处使用单位流编解码器
        .networkSynchronized(UNIT_STREAM_CODEC)
);
```

## **组件映射**(`Component Map`)

所有数据组件存储在`DataComponentMap`中（以`DataComponentType`为键，对象为值）。`DataComponentMap`功能类似只读`Map`，因此有方法通过`DataComponentType`获取条目（`#get`），或提供默认值（`#getOrDefault`）。

```java
// 对于某DataComponentMap实例map

// 若存在染色颜色组件则获取，否则返回null
@Nullable
DyeColor color = map.get(DataComponents.BASE_COLOR);
```

### `PatchedDataComponentMap`

默认`DataComponentMap`仅提供读操作，写操作通过子类`PatchedDataComponentMap`支持。包括`#set`组件值或`#remove`移除组件。

`PatchedDataComponentMap`使用原型(`prototype`)和补丁(`patch`)映射存储变更。原型是包含默认组件及其值的`DataComponentMap`。补丁映射是`DataComponentType`到`Optional`值的映射，包含对默认组件的更改。

```java
// 对于某PatchedDataComponentMap实例map

// 将基础颜色设为白色
map.set(DataComponents.BASE_COLOR, DyeColor.WHITE);

// 移除基础颜色：
// - 若无默认值则移除补丁
// - 若有默认值则设空Optional
map.remove(DataComponents.BASE_COLOR);
```

:::danger
原型和补丁映射都是`PatchedDataComponentMap`哈希码的一部分。因此映射内任何组件值应视为**不可变**(`immutable`)。修改数据组件值后务必调用`#set`或下文提及的相关方法。
:::

## **组件持有者**(`Component Holder`)

所有可持有数据组件的实例都实现`DataComponentHolder`。`DataComponentHolder`实质上是`DataComponentMap`内只读方法的委托。

```java
// 对于某ItemStack实例stack

// 委托给'DataComponentMap#get'
@Nullable
DyeColor color = stack.get(DataComponents.BASE_COLOR);
```

### `MutableDataComponentHolder`

`MutableDataComponentHolder`是NeoForge提供的接口，支持对组件映射的写操作。原版和NeoForge的所有实现均使用`PatchedDataComponentMap`存储数据组件，因此`#set`和`#remove`方法也有同名委托。

此外，`MutableDataComponentHolder`还提供`#update`方法：获取组件值（未设置时用默认值），操作值，然后设回映射。操作符为`UnaryOperator`（接收并返回组件值）或`BiFunction`（接收组件值和另一对象，返回组件值）。

```java
// 对于某ItemStack实例stack

FireworkExplosion explosion = stack.get(DataComponents.FIREWORK_EXPLOSION);

// 修改组件值
explosion = explosion.withFadeColors(new IntArrayList(new int[] {1, 2, 3}));

// 修改后应调用'set'
stack.set(DataComponents.FIREWORK_EXPLOSION, explosion);

// 更新组件值（内部调用'set'）
stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    // 无组件值时的默认值
    FireworkExplosion.DEFAULT,
    // 返回要设置的新FireworkExplosion
    explosion -> explosion.withFadeColors(new IntArrayList(new int[] {4, 5, 6}))
);

stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    // 无组件值时的默认值
    FireworkExplosion.DEFAULT,
    // 提供给函数的对象
    new IntArrayList(new int[] {7, 8, 9}),
    // 返回要设置的新FireworkExplosion
    FireworkExplosion::withFadeColors
);
```

## 向物品添加默认数据组件(`Adding Default Data Components to Items`)

尽管数据组件存储在`ItemStack`上，但可在`Item`上设置默认组件映射，在构造时作为原型传递给`ItemStack`。通过`Item.Properties#component`向`Item`添加组件。

```java
// 对于某DeferredRegister.Items实例REGISTRAR
public static final Item COMPONENT_EXAMPLE = REGISTRAR.register("component",
    // 使用register而非其他重载，因DataComponentType尚未注册
    registryName -> new Item(
        new Item.Properties()
        .setId(ResourceKey.create(Registries.ITEM, registryName))
        .component(BASIC_EXAMPLE.get(), new ExampleRecord(24, true))
    )
);
```

若需向原版或其他模组的已有物品添加数据组件，应在[**模组事件总线**](`modbus`)监听`ModifyDefaultComponentEvent`。事件提供`modify`和`modifyMatching`方法，允许修改关联物品的`DataComponentPatch.Builder`。构建器可`#set`组件或`#remove`现有组件。

```java
@SubscribeEvent // 在模组事件总线
public static void modifyComponents(ModifyDefaultComponentsEvent event) {
    // 为西瓜种子设置组件
    event.modify(Items.MELON_SEEDS, builder ->
        builder.set(BASIC_EXAMPLE.get(), new ExampleRecord(10, false))
    );

    // 为有合成残留物的物品移除组件
    event.modifyMatching(
        item -> !item.getCraftingRemainder().isEmpty(),
        builder -> builder.remove(DataComponents.BUCKET_ENTITY_DATA)
    );
}
```

## 使用自定义组件持有者(`Using Custom Component Holders`)

要创建自定义数据组件持有者，持有对象只需实现`MutableDataComponentHolder`并补全缺失方法。持有对象必须包含代表`PatchedDataComponentMap`的字段以实现关联方法。

```java
public class ExampleHolder implements MutableDataComponentHolder {

    private int data;
    private final PatchedDataComponentMap components;

    // 可提供重载以传入映射
    public ExampleHolder() {
        this.data = 0;
        this.components = new PatchedDataComponentMap(DataComponentMap.EMPTY);
    }

    @Override
    public DataComponentMap getComponents() {
        return this.components;
    }

    @Nullable
    @Override
    public <T> T set(DataComponentType<? super T> componentType, @Nullable T value) {
        return this.components.set(componentType, value);
    }

    @Nullable
    @Override
    public <T> T remove(DataComponentType<? extends T> componentType) {
        return this.components.remove(componentType);
    }

    @Override
    public void applyComponents(DataComponentPatch patch) {
        this.components.applyPatch(patch);
    }

    @Override
    public void applyComponents(DataComponentMap components) {
        this.components.setAll(components);
    }

    // 其他方法
}
```

### `DataComponentPatch`与编解码器(`DataComponentPatch` and Codecs)

为持久化组件到磁盘或网络传输，持有者可发送整个`DataComponentMap`。但这通常冗余，因默认值在数据发送处已存在。因此改用`DataComponentPatch`发送关联数据。`DataComponentPatch`仅含组件映射的补丁信息（无默认值）。接收方将补丁应用到其位置的原型上。

可通过`PatchedDataComponentMap#patch`从`PatchedDataComponentMap`创建`DataComponentPatch`。类似地，`PatchedDataComponentMap#fromPatch`可基于原型`DataComponentMap`和`DataComponentPatch`构建`PatchedDataComponentMap`。

```java
public class ExampleHolder implements MutableDataComponentHolder {

    public static final Codec<ExampleHolder> CODEC = RecordCodecBuilder.create(instance ->
        instance.group(
            Codec.INT.fieldOf("data").forGetter(ExampleHolder::getData),
            DataCopmonentPatch.CODEC.optionalFieldOf("components", DataComponentPatch.EMPTY).forGetter(holder -> holder.components.asPatch())
        ).apply(instance, ExampleHolder::new)
    );

    public static final StreamCodec<RegistryFriendlyByteBuf, ExampleHolder> STREAM_CODEC = StreamCodec.composite(
        ByteBufCodecs.INT, ExampleHolder::getData,
        DataComponentPatch.STREAM_CODEC, holder -> holder.components.asPatch(),
        ExampleHolder::new
    );

    // ...

    public ExampleHolder(int data, DataComponentPatch patch) {
        this.data = data;
        this.components = PatchedDataComponentMap.fromPatch(
            // 待应用的原型映射
            DataComponentMap.EMPTY,
            // 关联补丁
            patch
        );
    }

    // ...
}
```

[网络同步持有者数据](`network`)及磁盘读写需手动完成。

[registered]: ../concepts/registries.md
[codec]: ../datastorage/codecs.md
[modbus]: ../concepts/events.md#event-buses
[network]: ../networking/payload.md
[streamcodec]: ../networking/streamcodecs.md