---
sidebar_position: 3
---
# 使用配置任务

客户端和服务器的网络协议存在一个特定阶段，服务器可在玩家实际加入游戏前配置客户端。此阶段称为**配置阶段**(`configuration phase`)，例如原版服务器(`vanilla server`)会利用此阶段向客户端发送资源包信息。

模组(`mods`)也可利用此阶段在玩家加入游戏前配置客户端。

## 注册配置任务

使用配置阶段的第一步是注册配置任务。可通过在 `RegisterConfigurationTasksEvent` 事件中注册新配置任务实现：

```java
@SubscribeEvent // 在模组事件总线上
public static void register(final RegisterConfigurationTasksEvent event) {
    event.register(new MyConfigurationTask());
}
```

`RegisterConfigurationTasksEvent` 事件在模组总线(`mod bus`)上触发，暴露了服务器用于配置相关客户端的当前监听器(`listener`)。模组开发者可通过此监听器判断客户端是否运行该模组，若是则注册配置任务。

## 实现配置任务

配置任务是一个简单接口：`ICustomConfigurationTask`。该接口包含两个方法：`void run(Consumer<CustomPacketPayload> sender);` 和返回配置任务类型的 `ConfigurationTask.Type type();`。类型用于标识配置任务。示例如下：

```java
public record MyConfigurationTask implements ICustomConfigurationTask {
    public static final ConfigurationTask.Type TYPE = new ConfigurationTask.Type(ResourceLocation.fromNamespaceAndPath("mymod", "my_task"));
    
    @Override
    public void run(final Consumer<CustomPacketPayload> sender) {
        final MyData payload = new MyData();
        sender.accept(payload);
    }

    @Override
    public ConfigurationTask.Type type() {
        return TYPE;
    }
}
```

## 确认配置任务

配置在服务器端执行，服务器需要知道何时可执行下一个配置任务。这通过确认(`acknowledging`)当前配置任务的执行实现。

主要有两种实现方式：

### 捕获监听器

当客户端无需确认配置任务时，可捕获监听器并在服务器端直接确认：

```java
public record MyConfigurationTask(ServerConfigurationPacketListener listener) implements ICustomConfigurationTask {
    public static final ConfigurationTask.Type TYPE = new ConfigurationTask.Type(ResourceLocation.fromNamespaceAndPath("mymod", "my_task"));
    
    @Override
    public void run(final Consumer<CustomPacketPayload> sender) {
        final MyData payload = new MyData();
        sender.accept(payload);
        this.listener().finishCurrentTask(this.type());
    }

    @Override
    public ConfigurationTask.Type type() {
        return TYPE;
    }
}
```

使用此类配置任务需在 `RegisterConfigurationTasksEvent` 事件中捕获监听器：

```java
@SubscribeEvent // 在模组事件总线上
public static void register(final RegisterConfigurationTasksEvent event) {
    event.register(new MyConfigurationTask(event.getListener()));
}
```

此时当前配置任务完成后会立即执行下一个任务，且客户端无需确认。服务器也不会等待客户端处理发送的数据载荷(`payloads`)。

### 客户端确认配置任务

当客户端需要确认时，需向客户端发送自定义载荷：

```java
public record AckPayload() implements CustomPacketPayload {
    public static final CustomPacketPayload.Type<AckPayload> TYPE = new CustomPacketPayload.Type<>(ResourceLocation.fromNamespaceAndPath("mymod", "ack"));
    
    // 无数据的单位编解码器
    public static final StreamCodec<ByteBuf, AckPayload> STREAM_CODEC = StreamCodec.unit(new AckPayload());

    @Override
    public CustomPacketPayload.Type<? extends CustomPacketPayload> type() {
        return TYPE;
    }
}
```

当客户端正确处理服务器配置任务发送的载荷后，可向服务器发送此载荷以确认任务：

```java
public void onMyData(MyData data, IPayloadContext context) {
    context.enqueueWork(() -> {
        blah(data.name());
    })
    .exceptionally(e -> {
        // 处理异常
        context.disconnect(Component.translatable("my_mod.configuration.failed", e.getMessage()));
        return null;
    })
    .thenAccept(v -> {
        context.reply(new AckPayload());
    });     
}
```

其中 `onMyData` 是服务器配置任务所发送载荷的处理程序。

当服务器收到此载荷时，将确认配置任务并执行下一个任务：

```java
public void onAck(AckPayload payload, IPayloadContext context) {
    context.finishCurrentTask(MyConfigurationTask.TYPE);
}
```

其中 `onAck` 是客户端所发送载荷的处理程序。

## 延迟登录过程

若配置未被确认，服务器将无限等待，客户端永远无法加入游戏。因此除非配置任务失败（此时可断开客户端连接），否则必须始终确认配置任务。