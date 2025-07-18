---
sidebar_position: 1
---
# 注册载荷

**载荷**(`Payloads`)是在客户端和服务器之间发送任意数据的方式。可通过 `RegisterPayloadHandlersEvent` 事件中的 `PayloadRegistrar` 注册。

```java
@SubscribeEvent // 在模组事件总线上
public static void register(RegisterPayloadHandlersEvent event) {
    // 设置当前网络版本
    final PayloadRegistrar registrar = event.registrar("1");
}
```

假设需发送如下数据：

```java
public record MyData(String name, int age) {}
```

可为实现 `CustomPacketPayload` 接口创建用于发送/接收此数据的载荷：

```java
public record MyData(String name, int age) implements CustomPacketPayload {
    
    public static final CustomPacketPayload.Type<MyData> TYPE = new CustomPacketPayload.Type<>(ResourceLocation.fromNamespaceAndPath("mymod", "my_data"));

    // 每对元素定义用于编码/解码元素的流编解码器(`stream codec`)和编码元素的获取器(`getter`)
    // 'name' 将作为字符串编码/解码
    // 'age' 将作为整数编码/解码
    // 最终参数按顺序接收前述参数以构建载荷对象
    public static final StreamCodec<ByteBuf, MyData> STREAM_CODEC = StreamCodec.composite(
        ByteBufCodecs.STRING_UTF8,
        MyData::name,
        ByteBufCodecs.VAR_INT,
        MyData::age,
        MyData::new
    );
    
    @Override
    public CustomPacketPayload.Type<? extends CustomPacketPayload> type() {
        return TYPE;
    }
}
```

如上所示，`CustomPacketPayload` 接口要求实现 `type` 方法。该方法负责返回载荷的唯一标识符。此外还需通过 `StreamCodec` 注册**读取器**(`reader`)以读写载荷数据。

最后向注册器注册此载荷：

```java
// 在某个公共事件类中

@SubscribeEvent // 在模组事件总线上
public static void register(RegisterPayloadHandlersEvent event) {
    final PayloadRegistrar registrar = event.registrar("1");
    registrar.playBidirectional(
        MyData.TYPE,
        MyData.STREAM_CODEC,
        ServerPayloadHandler::handleDataOnMain
    );
}

// 在某个仅客户端事件类中

@SubscribeEvent // 仅在物理客户端(physical client)的模组事件总线
public static void register(RegisterClientPayloadHandlersEvent event) {
    event.register(
        MyData.TYPE,
        ClientPayloadHandler::handleDataOnMain
    );
}
```

分析上述代码可见：
- 注册器的 `play*` 方法用于注册游戏**游玩阶段**(`play phase`)发送的载荷
    - 未展示的方法 `configuration*` 和 `common*` 也可用于配置阶段(`configuration phase`)。`common` 方法可同时注册配置阶段和游玩阶段的载荷
- 注册器使用 `*Bidirectional` 方法注册双向（**逻辑服务器**(`logical server`)/**逻辑客户端**(`logical client`)）载荷
    - 未展示的方法 `*ToClient` 和 `*ToServer` 分别用于仅向逻辑客户端或逻辑服务器注册载荷
- 载荷类型用作唯一标识符
- [流编解码器(`stream codec`)]用于在网络缓冲区读写载荷
- **载荷处理程序**(`payload handler`)是载荷到达逻辑端时的回调函数
    - 使用 `*ToServer` 方法时，处理程序作为方法的最后一个参数
    - 使用 `*ToClient` 方法时，需通过 `RegisterClientPayloadHandlersEvent` 注册处理程序
    - 使用 `*Bidirectional` 方法时需同时使用两种处理程序

:::note
客户端载荷处理程序有独立事件 `RegisterClientPayloadHandlersEvent`，防止代码跨越逻辑端和物理端([sides])。
:::

注册载荷后需实现处理程序。以下以客户端处理程序为例（服务端类似）：

```java
public class ClientPayloadHandler {
    
    public static void handleDataOnMain(final MyData data, final IPayloadContext context) {
        // 在主线程处理数据
        blah(data.age());
    }
}
```

注意：
- 处理程序接收载荷和上下文对象
- 默认在主线程(`main thread`)调用处理程序

若需执行资源密集型计算，应在**网络线程**(`network thread`)进行以避免阻塞主线程：
- 服务端绑定连接：注册前通过 `PayloadRegistrar#executesOn` 设置 `HandlerThread#NETWORK`
- 客户端绑定连接：通过 `RegisterClientPayloadHandlersEvent#register` 传入 `HandlerThread`

```java
// 在某个公共事件类中

@SubscribeEvent // 在模组事件总线
public static void register(RegisterPayloadHandlersEvent event) {
    final PayloadRegistrar registrar = event.registrar("1")
        .executesOn(HandlerThread.NETWORK); // 后续所有载荷将在网络线程注册
    registrar.playBidirectional(
        MyData.TYPE,
        MyData.STREAM_CODEC,
        ServerPayloadHandler::handleDataOnNetwork
    );
}

// 在某个仅客户端事件类中

@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void register(RegisterClientPayloadHandlersEvent event) {
    event.register(
        MyData.TYPE,
        HandlerThread.NETWORK // 处理程序将在网络线程调用
        ClientPayloadHandler::handleDataOnNetwork
    );
}
```

:::note
`executesOn` 调用后注册的所有载荷将保持相同线程执行位置，直至再次调用 `executesOn`。

```java
PayloadRegistrar registrar = event.registrar("1");

registrar.playBidirectional(...); // 在主线程
registrar.playBidirectional(...); // 在主线程

// 配置方法通过创建新实例修改注册器状态
// 需存储结果以更新状态
registrar = registrar.executesOn(HandlerThread.NETWORK);

registrar.playBidirectional(...); // 在网络线程
registrar.playBidirectional(...); // 在网络线程

registrar = registrar.executesOn(HandlerThread.MAIN);

registrar.playBidirectional(...); // 在主线程
registrar.playBidirectional(...); // 在主线程
```
:::

注意：
- 需在主游戏线程运行代码时，可用 `enqueueWork` 提交任务
    - 方法返回在主线程完成的 `CompletableFuture`
    - 注意：未处理的 `CompletableFuture` 异常将被吞没且无通知

```java
public class ClientPayloadHandler {
    
    public static void handleDataOnNetwork(final MyData data, final IPayloadContext context) {
        // 在网络线程处理数据
        blah(data.name());
        
        // 在主线程处理数据
        context.enqueueWork(() -> {
            blah(data.age());
        })
        .exceptionally(e -> {
            // 处理异常
            context.disconnect(Component.translatable("my_mod.networking.failed", e.getMessage()));
            return null;
        });
    }
}
```

可通过自定义载荷配合[配置任务(`Configuration Tasks`)][configuration]配置客户端和服务器。

## 发送载荷

`CustomPacketPayload` 通过原版(`vanilla`)封包系统发送：
- 发往服务器：封装为 `ServerboundCustomPayloadPacket`
- 发往客户端：封装为 `ClientboundCustomPayloadPacket`
- 客户端载荷上限：1 MiB
- 服务器载荷上限：32 KiB

所有载荷均通过 `Connection#send` 发送，但按条件群发时直接调用不便。因此：
- `PacketDistributor` 提供便捷方法向客户端发载荷
- `ClientPacketDistributor` 提供 `sendToServer` 向服务器发包

```java
// 在客户端

// 向服务器发送载荷
ClientPacketDistributor.sendToServer(new MyData(...));

// 在服务器

// 发送给特定玩家 (ServerPlayer serverPlayer)
PacketDistributor.sendToPlayer(serverPlayer, new MyData(...));

/// 发送给追踪此区块的所有玩家 (ServerLevel serverLevel, ChunkPos chunkPos)
PacketDistributor.sendToPlayersTrackingChunk(serverLevel, chunkPos, new MyData(...));

/// 发送给所有在线玩家
PacketDistributor.sendToAllPlayers(new MyData(...));
```

更多实现详见 `PacketDistributor` 和 `ClientPacketDistributor` 类。

[configuration]: configuration-tasks.md
[sides]: ../concepts/sides.md
[streamcodec]: streamcodecs.md