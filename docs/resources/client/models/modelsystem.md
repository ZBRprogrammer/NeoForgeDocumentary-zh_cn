# 理解模型系统(`Model System`)

Minecraft 中的模型本质上是一个带有纹理的**四边形面**(`quads`)列表。建模过程的每个部分都有独立的实现，底层模型 JSON 会被反序列化为**未烘焙模型**(`UnbakedModel`)。最终，管道的每个部分都会接收一些`List<BakedQuad>`及其自身管道所需的属性。某些[**方块实体渲染器**(`block entity renderers`)][ber]也会使用这些模型。模型的复杂度没有限制。

模型存储在`ModelManager`中，可通过`Minecraft.getInstance().getModelManager()`访问。对于物品管道，可以通过传递[`ResourceLocation`][rl]调用`ModelManager#getItemModel`获取关联的[`ItemModel`][itemmodels]。对于方块状态管道，可以通过传递`BlockState`调用`ModelManager.getBlockModelShaper().getBlockModel()`获取关联的`BlockStateModel`。Mod 通常会复用先前自动加载和烘焙过的模型。

## 通用模型和几何体(`Common Models and Geometry`)

基础模型 JSON (位于`assets/<namespace>/models`)会被反序列化为`UnbakedModel`。`UnbakedModel`通常距离其烘焙输出仅一步之遥，包含一些通用属性的基本形式。它最重要的内容是通过`UnbakedModel#geometry`获取的`UnbakedGeometry`，这代表了将成为`BakedQuad`的数据。这些四边形面通过（最终）调用`UnbakedGeometry#bake`被内联到物品和方块状态模型中。这通常会构建一个`QuadCollection`，其中包含可在任何时候渲染的`BakedQuad`列表，或仅当给定方向未被剔除时渲染。在建模软件（以及大多数其他游戏）中，四边形相当于三角形，但由于 Minecraft 主要关注方块，开发者选择在 Minecraft 渲染中使用**四边形**(`quads`)(4 个顶点)而非**三角形**(`triangles`)(3 个顶点)。

`UnbakedModel`包含被[**方块状态定义**(`block state definition`)][bsd]、[**物品模型**(`item models`)][itemmodelsection]或两者使用的信息。例如：
- `useAmbientOcclusion`专用于方块状态定义
- `guiLight`和`transforms`专用于物品模型
- `textureSlots`和`parent`两者共用

在**烘焙过程**(`baking process`)中，每个`UnbakedModel`都被包装在`ResolvedModel`中，由`ModelBaker`为物品或方块状态获取。顾名思义，`ResolvedModel`是一个已解析所有悬空引用的`UnbakedModel`。相关数据可通过`getTop*`方法获取，这些方法从当前模型及其父模型计算属性和几何体。通过调用`ResolvedModel#bakeTopGeometry`将`ResolvedModel`烘焙为其`QuadCollection`通常在此完成。

## 方块状态定义(`Block State Definitions`)

方块状态定义 JSON (位于`assets/<namespace>/blockstates`)会被编译并烘焙为每个`BlockState`的`BlockStateModel`。创建`BlockStateModel`的过程如下：

- **加载过程**:
    - 方块状态定义 JSON 被加载到`BlockStateModel.UnbakedRoot`中。根是一个通用的共享缓存系统，用于将`BlockState`链接到某些`BlockStateModel`集合
    - `BlockStateModel.UnbakedRoot`加载`BlockStateModel.Unbaked`并准备将它们链接到对应的`BlockState`
    - `BlockStateModel.Unbaked`加载其`BlockModelPart.Unbaked`，用于获取通用的`UnbakedModel`（或更具体地说，是`ResolvedModel`)
- **烘焙过程**:
    - 为每个`BlockState`调用`BlockStateModel.UnbakedRoot#bake`
    - 为给定`BlockState`调用`BlockStateModel.Unbaked#bake`，创建`BlockStateModel`
    - 为`BlockStateModel`内的模型部件调用`BlockModelPart.Unbaked#bake`，将`ResolvedModel`内联为`QuadCollection`，同时获取**环境光遮蔽**(`ambient occlusion`)设置、**粒子图标**(`particle icon`)和默认的**渲染类型**(`render type`)

`BlockStateModel`中最重要的方法是`collectParts`，它负责返回要渲染的`BlockModelPart`列表。请记住，每个`BlockModelPart`都包含其`BakedQuad`列表（通过`BlockModelPart#getQuads`），这些列表随后会上传到**顶点消费者**(`vertex consumer`)并进行渲染。`collectParts`有四个参数：

- `BlockAndTintGetter`：表示渲染`BlockState`所在的**层级**(`level`)
- `BlockPos`：方块渲染的位置
- `BlockState`：正在渲染的[**方块状态**(`blockstate`)。可能为 null，表示正在渲染物品
- `RandomSource`：可用于随机化的客户端绑定随机源

### 模型数据(`Model Data`)

有时，`BlockStateModel`可能依赖`BlockEntity`来决定在`collectParts`中选择哪些`BlockModelPart`。NeoForge 提供了`ModelData`系统来同步和传递来自`BlockEntity`的数据。为此，`BlockEntity`必须实现`getModelData`并返回要同步的数据。然后可以通过调用`BlockEntity#requestModelDataUpdate`将数据发送到客户端。在`collectParts`中，可以在`BlockAndTintGetter`上使用`BlockPos`调用`getModelData`来获取数据。

## 物品模型(`Item Models`)

[**客户端物品**(`client item`)][clientitem] JSON (位于`assets/<namespace>/items`)会被编译并烘焙为给定`Item`的`ItemModel`，供`ItemStack`使用。创建`ItemModel`的过程如下：

- **加载过程**:
    - 客户端物品 JSON 被加载到`ClientItem`中。它保存了物品模型和一些关于如何渲染的通用属性
    - `ClientItem`加载`ItemModel.Unbaked`
- **烘焙过程**:
    - 为每个`Item`调用`ItemModel.Unbaked#bake`，将`ResolvedModel`内联为`List<BakedQuad>`，同时获取一些通用的`ModelRenderProperties`，如果`Item`是`BlockItem`则还包括渲染类型

有关物品渲染的信息可在[手动渲染物品][itemmodels]部分找到。

### 视角(`Perspectives`)

Minecraft 的渲染引擎为物品渲染识别总共 8 种视角类型（如果包括代码中的后备类型则为 9 种）。这些用于模型 JSON 的`display`块，在代码中通过`ItemDisplayContext`枚举表示。这些通常从`UnbakedModel`传递到`ItemModel`中的`ModelRenderProperties`，然后通过`ModelRenderProperties#applyToLayer`应用到`ItemStackRenderState`。

| 枚举值                   | JSON 键                  | 用途                                                                                                           |
|--------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| `THIRD_PERSON_RIGHT_HAND`| `"thirdperson_righthand"`| 第三人称右手（F5 视图或其他玩家视角）                                                                          |
| `THIRD_PERSON_LEFT_HAND` | `"thirdperson_lefthand"` | 第三人称左手（F5 视图或其他玩家视角）                                                                          |
| `FIRST_PERSON_RIGHT_HAND`| `"firstperson_righthand"`| 第一人称右手                                                                                                  |
| `FIRST_PERSON_LEFT_HAND` | `"firstperson_lefthand"` | 第一人称左手                                                                                                  |
| `HEAD`                   | `"head"`                 | 在玩家的头部盔甲槽中时（通常只能通过命令实现）                                                                |
| `GUI`                    | `"gui"`                  | 物品栏、玩家快捷栏                                                                                            |
| `GROUND`                 | `"ground"`               | 掉落物；注意掉落物的旋转由掉落物渲染器处理，而非模型                                                          |
| `FIXED`                  | `"fixed"`                | 物品展示框                                                                                                    |
| `NONE`                   | `"none"`                 | 代码中的后备用途，不应在 JSON 中使用                                                                          |

NeoForge 允许为自定义渲染调用[扩展][extended]`ItemDisplayContext`。模组的`ItemDisplayContext`可以在模型中未指定变换时指定要使用的后备变换。否则，行为将与原版相同。

## 修改烘焙结果(`Modifying a Baking Result`)

在代码中修改现有的方块状态模型或物品堆叠模型通常可以通过将模型包装在某种**委托**(`delegate`)中完成。方块状态模型有`DelegateBlockStateModel`，而物品堆叠模型没有现有实现。你的实现可以仅覆盖选定方法，如下所示：

```java
// 对于方块状态
public class MyDelegateBlockStateModel extends DelegateBlockStateModel {
    // 将原始模型传递给超类
    public MyDelegateBlockStateModel(BlockStateModel originalModel) {
        super(originalModel);
    }
    
    // 在此覆盖你需要的任何方法。如果需要，也可以访问 originalModel
}

// 对于物品模型
public class MyDelegateItemModel implements ItemModel {

    private final ItemModel originalModel;

    public MyDelegateItemModel(ItemModel originalModel) {
        this.originalModel = originalModel;
    }

    // 在此覆盖你需要的任何方法。如果需要，也可以访问 originalModel
    @Override
    public void update(ItemStackRenderState renderState, ItemStack stack, ItemModelResolver resolver, ItemDisplayContext displayContext, @Nullable ClientLevel level, @Nullable LivingEntity entity, int seed
    ) {
        this.originalModel.update(renderState, stack, resolver, displayContext, level, entity, seed);
    }
}
```

编写模型包装类后，必须将其应用到应受影响的模型上。在[**模组事件总线**][modbus]上为`ModelEvent.ModifyBakingResult`注册一个[**客户端**][sides][**事件处理器**][event]：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void modifyBakingResult(ModelEvent.ModifyBakingResult event) {
    // 对于方块状态模型
    event.getBakingResult().blockStateModels().computeIfPresent(
        // 要修改的模型的方块状态
        MyBlocksClass.EXAMPLE_BLOCK.get().defaultBlockState(),
        // 一个接收位置和原始模型作为参数的 BiFunction，返回新模型
        (location, model) -> new MyDelegateBakedModel(model);
    );

    // 对于物品模型
    event.getBakingResult().itemStackModels().computeIfPresent(
        // 要修改的模型的资源位置
        // 通常是物品注册名；但由于 ITEM_MODEL 数据组件，可以是任何内容
        MyItemsClass.EXAMPLE_ITEM.getKey().location(),
        // 一个接收位置和原始模型作为参数的 BiFunction，返回新模型
        (location, model) -> new MyDelegateItemModel(model);
    );
}
```

:::warning
在可能的情况下，通常建议使用[**自定义模型加载器**][modelloader]而非在`ModelEvent.ModifyBakingResult`中包装烘焙模型。如果需要，自定义模型加载器也可以使用委托模型。
:::

[ao]: https://en.wikipedia.org/wiki/Ambient_occlusion
[ber]: ../../../blockentities/ber.md
[blockstate]: ../../../blocks/states.md
[bsd]: #block-state-definitions
[clientitem]: items.md
[event]: ../../../concepts/events.md
[extended]: ../../../advanced/extensibleenums.md#creating-an-enum-entry
[itemmodels]: items.md#manually-rendering-an-item
[itemmodelsection]: #item-models
[livingentity]: ../../../entities/livingentity.md
[modbus]: ../../../concepts/events.md#event-buses
[modelloader]: modelloaders.md
[rl]: ../../../misc/resourcelocation.md
[perspective]: #perspectives
[rendertype]: index.md#render-types
[sides]: ../../../concepts/sides.md