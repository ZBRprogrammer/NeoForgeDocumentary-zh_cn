---
sidebar_position: 4
---
# **数据附件**(`Data Attachments`)

**数据附件系统**(`data attachment system`)允许模组在**方块实体**(`block entities`)、**区块**(`chunks`)、**实体**(`entities`)和**维度**(`levels`)上附加存储额外数据。

_存储额外维度数据可使用 [SavedData][saveddata]。_

:::note
物品堆栈的数据附件已被原版的[数据组件][datacomponents]取代。
:::

## 创建附件类型

使用此系统需注册 `AttachmentType`。附件类型包含以下配置：
- **默认值供应器**(`default value supplier`)：首次访问数据时创建实例
- **序列化器**(`serializer`)（可选）：若附件需持久化则配置
- （若配置序列化器）`copyOnDeath` 标志：在死亡时自动复制实体数据（见下文）

:::tip
若附件无需持久化，请勿提供序列化器。
:::

提供附件序列化器有几种方式：直接实现 `IAttachmentSerializer`；实现 [`ValueIOSerializable`][valueio] 并使用静态 `AttachmentType#serializable` 方法构建；或向构建器提供 **Map编解码器**(`map codec`)。

无论何种方式，附件**必须注册**到 `NeoForgeRegistries.ATTACHMENT_TYPES` 注册表。示例如下：

```java
// 为附件类型创建 DeferredRegister
private static final DeferredRegister<AttachmentType<?>> ATTACHMENT_TYPES = DeferredRegister.create(NeoForgeRegistries.ATTACHMENT_TYPES, MOD_ID);

// 通过 ValueIOSerializable 序列化
private static final Supplier<AttachmentType<ItemStackHandler>> HANDLER = ATTACHMENT_TYPES.register(
    "handler", () -> AttachmentType.serializable(() -> new ItemStackHandler(1)).build()
);
// 通过 Map编解码器 序列化
private static final Supplier<AttachmentType<Integer>> MANA = ATTACHMENT_TYPES.register(
    "mana", () -> AttachmentType.builder(() -> 0).serialize(Codec.INT.fieldOf("mana")).build()
);
// 无序列化
private static final Supplier<AttachmentType<SomeCache>> SOME_CACHE = ATTACHMENT_TYPES.register(
    "some_cache", () -> AttachmentType.builder(() -> new SomeCache()).build()
);

// 在模组构造函数中，别忘了将 DeferredRegister 注册到模组总线：
ATTACHMENT_TYPES.register(modBus);
```

## 使用附件类型

附件类型注册后，可用于任何持有者对象。若数据不存在时调用 `getData` 将附加新默认实例。

```java
// 若存在则获取 ItemStackHandler，否则附加新实例：
ItemStackHandler stackHandler = chunk.getData(HANDLER);
// 若存在则获取玩家法力值，否则附加0：
int playerMana = player.getData(MANA);
// 以此类推...
```

若不希望附加默认实例，可添加 `hasData` 检查：

```java
// 操作前检查区块是否有 HANDLER 附件
if (chunk.hasData(HANDLER)) {
    ItemStackHandler stackHandler = chunk.getData(HANDLER);
    // 使用 chunk.getData(HANDLER) 执行操作
}
```

数据可通过 `setData` 更新：

```java
// 法力值增加10
player.setData(MANA, player.getData(MANA) + 10);
```

:::important
通常，方块实体和区块修改后需标记为**脏数据**(`dirty`)（通过 `setChanged` 和 `setUnsaved(true)`）。调用 `setData` 会自动处理：

```java
chunk.setData(MANA, chunk.getData(MANA) + 10); // 自动调用 setUnsaved
```

但若修改通过 `getData` 获取的数据（包括新建的默认实例），则需显式标记方块实体和区块为脏数据：

```java
var mana = chunk.getData(MUTABLE_MANA);
mana.set(10);
chunk.setUnsaved(true); // 因未使用 setData 需手动执行
```
:::

## 与客户端共享数据

若需将方块实体、区块或实体附件同步至客户端，需[自行发送数据包][network]到客户端。对于区块，可使用 `ChunkWatchEvent.Sent` 确定何时向玩家发送区块数据。

## 玩家死亡时复制数据

默认情况下，[实体][entity]数据附件在玩家死亡时**不会复制**(`not copied`)。要自动复制附件，请在附件构建器中设置 `copyOnDeath`。

更复杂的处理可通过 `PlayerEvent.Clone` 实现：从原始实体读取数据并分配给新实体。此事件中，`#isWasDeath` 方法可区分死亡重生与末地返回。这很重要，因为从末地返回时数据已存在，需避免重复赋值。

例如：

```java
@SubscribeEvent // 在游戏事件总线上
public static void onClone(PlayerEvent.Clone event) {
    if (event.isWasDeath() && event.getOriginal().hasData(MY_DATA)) {
        event.getEntity().getData(MY_DATA).fieldToCopy = event.getOriginal().getData(MY_DATA).fieldToCopy;
    }
}
```

[datacomponents]: ../items/datacomponents.md
[entity]: ../entities/index.md
[network]: ../networking/index.md
[saveddata]: saveddata.md
[valueio]: valueio.md#valueioserializable