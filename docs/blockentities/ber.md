# **方块实体渲染器**(`BlockEntityRenderer`)

**方块实体渲染器**(`BlockEntityRenderer`)，常缩写为 BER，用于以[静态烘焙模型][model]（JSON、OBJ 等）无法表示的方式渲染[方块][block]。例如，可用于动态渲染类似箱子方块的容器内容。方块实体渲染器要求方块具有[`BlockEntity`][blockentity]，即使该方块不存储任何数据。

要创建 BER，需创建继承自 `BlockEntityRenderer` 的类。它接受一个泛型参数，指定方块的 `BlockEntity` 类，该类用作 BER 的 `render` 方法中的参数类型。

```java
// 假设 MyBlockEntity 作为 BlockEntity 的子类存在
public class MyBlockEntityRenderer implements BlockEntityRenderer<MyBlockEntity> {
    // 为下方的 lambda 添加构造函数参数。也可用于获取某些上下文
    // 存储在局部字段中，例如实体渲染器分发器（如有需要）
    public MyBlockEntityRenderer(BlockEntityRendererProvider.Context context) {
    }
    
    // 此方法每帧调用以渲染方块实体。参数为：
    // - blockEntity:   正在渲染的方块实体实例。使用传递给父接口的泛型类型
    // - partialTick:   自上次刻以来经过的时间量（0.0 到 1.0），单位为刻的小数
    // - poseStack:     要渲染到的姿态堆栈
    // - bufferSource:  从中获取顶点缓冲区的缓冲源
    // - packedLight:   方块实体的光照值
    // - packedOverlay: 方块实体的当前覆盖值，通常为 OverlayTexture.NO_OVERLAY
    // - cameraPos:     渲染器相机的位置
    @Override
    public void render(MyBlockEntity blockEntity, float partialTick, PoseStack stack, MultiBufferSource bufferSource, int packedLight, int packedOverlay, Vec3 cameraPos) {
        // 在此进行渲染
    }
}
```

对于给定的 `BlockEntityType<?>`，只能存在一个 BER。因此，特定于单个方块实体实例的值应存储在该方块实体实例中，而非 BER 本身。

创建 BER 后，还必须将其注册到 `EntityRenderersEvent.RegisterRenderers`（在[模组事件总线][eventbus]上触发的[事件][event]）：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerEntityRenderers(EntityRenderersEvent.RegisterRenderers event) {
    event.registerBlockEntityRenderer(
            // 要为其注册渲染器的方块实体类型
            MyBlockEntities.MY_BLOCK_ENTITY.get(),
            // 从 BlockEntityRendererProvider.Context 到 BlockEntityRenderer 的函数
            MyBlockEntityRenderer::new
    );
}
```

若 BER 中不需要提供上下文，也可移除构造函数：

```java
public class MyBlockEntityRenderer implements BlockEntityRenderer<MyBlockEntity> {
    @Override
    public void render( /* ... */ ) { /* ... */ }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerEntityRenderers(EntityRenderersEvent.RegisterRenderers event) {
    event.registerBlockEntityRenderer(MyBlockEntities.MY_BLOCK_ENTITY.get(),
            // 将上下文传递给空（默认）构造函数调用
            context -> new MyBlockEntityRenderer()
    );
}
```

## 物品方块渲染

由于并非所有带渲染器的方块实体都能使用静态模型渲染，可创建特殊渲染器自定义物品渲染过程。这通过[`SpecialModelRenderer`s][special]实现。此类情况下，必须创建特殊模型渲染器以正确渲染物品，并为方块作为物品渲染的场景（例如末影人搬运方块）注册相应的特殊方块模型渲染器。

更多信息请参阅[客户端物品文档][special]。

[block]: ../blocks/index.md
[blockentity]: index.md
[event]: ../concepts/events.md#registering-an-event-handler
[eventbus]: ../concepts/events.md#event-buses
[item]: ../items/index.md
[model]: ../resources/client/models/index.md
[special]: ../resources/client/models/items.md#special-models