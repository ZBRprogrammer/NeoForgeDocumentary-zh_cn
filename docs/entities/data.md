---
sidebar_position: 2
---
# **数据与网络**(`Data and Networking`)

没有数据的实体是相当无用的，因此，在实体上存储数据至关重要。所有实体都存储一些默认数据，例如它们的类型和位置。本文将解释如何添加你自己的数据，以及如何同步这些数据。

添加数据的最简单方法是作为 `Entity` 类中的一个字段。然后你可以以任何方式与此数据交互。然而，一旦你需要同步该数据，这很快就会变得非常烦人。这是因为大多数实体逻辑仅在服务器上运行，并且只有偶尔（取决于 [`EntityType`][entitytype] 的 `clientUpdateInterval` 值）才会向客户端发送更新；这也是当服务器刻速度太慢时实体"卡顿"明显的原因。

因此，原版引入了几个系统来帮助解决这个问题，每个系统都有特定用途。你也可以在必要时选择[发送自定义数据][custom]。

## `SynchedEntityData`

`SynchedEntityData` 是一个用于在运行时存储值并通过网络同步的系统。它分为三个类：

- `EntityDataSerializer` 基本上是 [`StreamCodec`][streamcodec] 的包装器。
    - Minecraft 使用一个硬编码的序列化器映射。NeoForge 将此映射转换为注册表，这意味着如果你想添加新的 `EntityDataSerializer`，必须通过[注册][registration]添加。
    - Minecraft 在 `EntityDataSerializers` 类中定义了各种默认的 `EntityDataSerializer`。
- `EntityDataAccessor` 由实体持有，用于获取和设置数据值。
- `SynchedEntityData` 本身持有实体的所有 `EntityDataAccessor`，并根据需要自动调用 `EntityDataSerializer` 来同步值。

首先，在你的实体类中创建一个 `EntityDataAccessor`：

```java
public class MyEntity extends Entity {
    // 泛型类型必须与下面第二个参数的类型匹配。
    public static final EntityDataAccessor<Integer> MY_DATA =
        SynchedEntityData.defineId(
            // 实体的类。
            MyEntity.class,
            // 实体数据访问器类型。
            EntityDataSerializers.INT
        );
}
```

:::danger
虽然编译器允许你在 `SynchedEntityData#defineId()` 中使用除所属类以外的类作为第一个参数，但这样做可能导致难以调试的问题，因此应不惜一切代价避免。（这也包括通过混入或其他类似方法添加字段。）
:::

然后我们必须在 `defineSynchedData` 方法中定义默认值，如下所示：

```java
public class MyEntity extends Entity {
    public static final EntityDataAccessor<Integer> MY_DATA = SynchedEntityData.defineId(MyEntity.class, EntityDataSerializers.INT);

    @Override
    protected void defineSynchedData(SynchedEntityData.Builder builder) {
        // 我们的默认值为零。
        builder.define(MY_DATA, 0);
    }
}
```

最后，我们可以这样获取和设置实体数据（假设我们在 `MyEntity` 的一个方法中）：

```java
int data = this.getEntityData().get(MY_DATA);
this.getEntityData().set(MY_DATA, 1);
```

## `readAdditionalSaveData` 和 `addAdditionalSaveData`

这两个方法用于将数据读写到磁盘。它们通过从[值输入输出][valueio]加载/保存你的值来工作，如下所示：

```java
// 假设类中存在一个 `int data`。
@Override
protected void readAdditionalSaveData(ValueInput input) {
    this.data = input.getIntOr("my_data", 0);
}

@Override
protected void addAdditionalSaveData(ValueOutput output) {
    output.putInt("my_data", this.data);
}
```

## **自定义生成数据**(`Custom Spawn Data`)

在某些情况下，你的实体在生成时客户端需要一些自定义数据，但这些数据不会随时间改变。此时，你可以在实体上实现 `IEntityWithComplexSpawn` 接口，并使用其两个方法 `#writeSpawnData` 和 `#readSpawnData` 向网络缓冲区写入/读取数据：

```java
@Override
public void writeSpawnData(RegistryFriendlyByteBuf buf) {
    buf.writeInt(1234);
}

@Override
public void readSpawnData(RegistryFriendlyByteBuf buf) {
    int i = buf.readInt();
}
```

此外，你可以在生成时发送自己的数据包。为此，覆盖 `IEntityExtension#sendPairingData` 并像发送任何其他数据包一样在那里发送你的数据包：

```java
@Override
public void sendPairingData(ServerPlayer player, Consumer<CustomPacketPayload> packetConsumer) {
    // 调用超类以获得一些基本功能。
    super.sendPairingData(player, packetConsumer);
    // 添加你自己的数据包。
    packetConsumer.accept(new MyPacket(...));
}
```

有关自定义网络数据包的更多信息，请参阅[网络文章][networking]。

## **数据附件**(`Data Attachments`)

实体已被修补以扩展 `AttachmentHolder`，因此支持通过[数据附件][attachment]进行数据存储。其主要用途是在不属于你自己的实体（即由 Minecraft 或其他模组添加的实体）上定义自定义数据。更多信息请参见链接文章。

## **自定义网络消息**(`Custom Network Messages`)

对于同步，你也可以选择在需要时使用自定义数据包发送附加信息。更多信息请参阅[网络文章][networking]。

[attachment]: ../datastorage/attachments.md
[custom]: #custom-network-messages
[entitytype]: index.md#entitytype
[networking]: ../networking/index.md
[registration]: ../concepts/registries.md#methods-for-registering
[streamcodec]: ../networking/streamcodecs.md
[valueio]: ../datastorage/valueio.md