# 自定义模型加载器

模型本质上就是一个形状。它可以是一个立方体、一组立方体、一组三角形，或是任何其他几何形状（或几何形状的集合）。在大多数情况下，模型的定义方式并不重要，因为最终所有内容都会被**烘焙**(`bake`)成一个`QuadCollection`。因此，NeoForge 提供了注册自定义模型加载器的能力，可以将任何模型转换为游戏使用的烘焙格式。

## 模型加载器

方块模型的入口点仍然是模型 JSON 文件。但是，你可以在 JSON 的根节点指定一个`loader`字段，这将把默认加载器替换为你自己的加载器。自定义模型加载器可以忽略默认加载器所需的所有字段。

除了默认模型加载器外，NeoForge 还提供了几个内置加载器，每个都有不同的用途。

### 组合模型(`Composite Model`)

组合模型可用于在父模型中指定不同的模型部件，并在子模型中仅应用其中一部分。最好通过示例来说明。考虑位于`examplemod:example_composite_model`的以下父模型：

```json5
{
    "loader": "neoforge:composite",
    // 指定模型部件
    "children": {
        // 这些可以是对另一个模型的引用，也可以是模型本身
        "part_1": {
            "parent": "examplemod:some_model_1"
        },
        "part_2": {
            "parent": "examplemod:some_model_2"
        }
    },
    "visibility": {
        // 默认禁用部件2
        "part_2": false
    }
}
```

然后，我们可以在`examplemod:example_composite_model`的子模型中禁用和启用各个部件：

```json5
{
    "parent": "examplemod:example_composite_model",
    // 覆盖可见性。如果缺少某个部件，将使用父模型的可见性值
    "visibility": {
        "part_1": false,
        "part_2": true
    }
}
```

要为此模型进行[**数据生成**(`datagen`)][modeldatagen]，请使用自定义加载器类`CompositeModelBuilder`。

:::warning
组合模型加载器不应用于[**客户端物品**(`client items`)][citems]使用的模型。相反，它们应使用定义本身提供的[组合模型][itemcomposite]。
:::

### 空模型(`Empty Model`)

空模型完全不渲染任何内容。

```json5
{
    "loader": "neoforge:empty"
}
```

### OBJ 模型(`OBJ Model`)

OBJ 模型加载器允许你在游戏中使用 Wavefront `.obj` 3D 模型，从而可以在模型中包含任意形状（包括三角形、圆形等）。`.obj` 模型必须放在`models`文件夹（或其子文件夹）中，并且必须提供同名的`.mtl`文件（或手动设置）。例如，位于`models/block/example.obj`的 OBJ 模型必须有一个对应的 MTL 文件位于`models/block/example.mtl`。

```json5
{
    "loader": "neoforge:obj",
    // 必需。引用模型文件。注意：这是相对于命名空间根目录的路径，而不是模型文件夹。
    "model": "examplemod:models/example.obj",
    // 通常，.mtl文件必须与.obj文件放在同一位置，仅文件扩展名不同。
    // 加载器会自动拾取它们。但你也可以根据需要手动设置
    // .mtl文件的位置。
    "mtl_override": "examplemod:models/example_other_name.mtl",
    // 这些纹理可以在.mtl文件中引用为 #texture0, #particle 等。
    // 这通常需要手动编辑.mtl文件。
    "textures": {
        "texture0": "minecraft:block/cobblestone",
        "particle": "minecraft:block/stone"
    },
    // 启用或禁用模型的自动剔除。可选，默认为 true。
    "automatic_culling": false,
    // 是否对模型进行着色。可选，默认为 true。
    "shade_quads": false,
    // 某些建模软件会假设V=0是底部而不是顶部。此属性将V上下翻转。
    // 可选，默认为 false。
    "flip_v": true,
    // 是否启用自发光。可选，默认为 true。
    "emissive_ambient": false
}
```

要为此模型进行[**数据生成**(`datagen`)][modeldatagen]，请使用自定义加载器类`ObjModelBuilder`。

### 创建自定义模型加载器

要创建自己的模型加载器，你需要四个类，外加一个事件处理器：

- 一个`UnbakedModelLoader`类
- 一个`UnbakedGeometry`类，通常是`ExtendedUnbakedGeometry`实例
- 一个`UnbakedModel`类，通常是`AbstractUnbakedModel`实例
- 一个`QuadCollection`类来保存烘焙后的四边形面(`quads`)，通常是类本身
- 一个用于`ModelEvent.RegisterLoaders`的[**客户端**(`client-side`)][sides][事件处理器][event]，用于注册**未烘焙模型**(`unbaked model`)加载器
- 可选：用于缓存加载数据的模型加载器，需要一个用于`AddClientReloadListenersEvent`的[**客户端**(`client-side`)][sides][事件处理器][event]

为了说明这些类如何连接，我们将跟踪一个模型的加载过程：

- 在模型加载期间，设置了`loader`属性为你的加载器的模型 JSON 会被传递给你的**未烘焙模型加载器**(`UnbakedModelLoader`)。加载器读取模型 JSON 后，使用 JSON 的属性和模型的**未烘焙四边形面**(`unbaked quads`)返回一个`UnbakedModel`对象。
- 在模型**烘焙**(`bake`)期间，调用`UnbakedGeometry#bake`，返回一个`QuadCollection`。
- 在模型渲染期间，`QuadCollection`以及[**客户端物品**(`client item`)][citems]或[**方块状态定义**(`block state definition`)][blockstatedefinition]所需的任何其他信息将被用于渲染。

:::note
如果你正在为物品或方块状态使用的模型创建自定义模型加载器，根据用例，创建新的`ItemModel`或`BlockStateModel`可能更好。例如，使用或生成`QuadCollection`的模型更适合作为`ItemModel`或`BlockStateModel`，而解析不同数据格式（如`.obj`）的模型应使用新的模型加载器。
:::

让我们通过一个基本的类设置进一步说明。加载器类名为`MyUnbakedModelLoader`，**未烘焙类**(`unbaked class`)名为`MyUnbakedModel`，**未烘焙几何体**(`unbaked geometry`)名为`MyUnbakedGeometry`。我们还将假设模型加载器需要一些缓存：

```java
// 这是用于将模型加载到其未烘焙格式的类
public class MyUnbakedModelLoader implements UnbakedModelLoader<MyUnbakedModel>, ResourceManagerReloadListener {
    // 强烈建议对未烘焙模型加载器使用单例模式，因为所有模型都可以通过一个加载器加载。
    public static final MyUnbakedModelLoader INSTANCE = new MyUnbakedModelLoader();
    // 我们将用于注册此加载器的id。也用于加载器的数据生成类。
    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_loader");

    // 根据单例模式，将构造函数设为私有。
    private MyUnbakedModelLoader() {}

    @Override
    public void onResourceManagerReload(ResourceManager resourceManager) {
        // 处理任何缓存清除逻辑
    }

    @Override
    public MyUnbakedModel read(JsonObject obj, JsonDeserializationContext context) throws JsonParseException {
        // 使用给定的JsonObject和（如果需要）JsonDeserializationContext从模型JSON中获取属性。
        // MyUnbakedModel的构造函数可以有构造参数（见下文）。

        // 读取用于创建四边形面(`quads`)的数据
        MyUnbakedGeometry geometry;

        // 对于Vanilla和NeoForge提供的基本参数，你可以使用StandardModelParameters
        StandardModelParameters params = StandardModelParameters.parse(obj, context);

        return new MyUnbakedModel(params, geometry);
    }
}

// 保存要渲染的未烘焙四边形面(`unbaked quads`)
// 存储在未烘焙模型中的其他信息应传递到上下文映射(`context map`)
public class MyUnbakedGeometry implements ExtendedUnbakedGeometry {

    public MyUnbakedGeometry(...) {
        // 存储要烘焙的未烘焙四边形面(`quads`)
    }

    // 负责模型烘焙的方法，返回四边形集合(`quad collection`)。此方法的参数是：
    // - 纹理名称到其关联材质的映射。
    // - 模型烘焙器(`model baker`)。可用于获取子模型进行烘焙以及从纹理槽(`texture slots`)获取精灵(`sprites`)。
    // - 模型状态(`model state`)。这保存来自方块状态文件的变换，通常来自旋转和uvlock。
    // - 模型的名称。
    // - 由NeoForge和你的未烘焙模型提供的设置ContextMap。有关所有可用属性，请参阅'NeoForgeModelProperties'类。
    @Override
    public QuadCollection bake(TextureSlots textureSlots, ModelBaker baker, ModelState state, ModelDebugName debugName, ContextMap additionalProperties) {
        // 用于创建集合的构建器
        var builder = new QuadCollection.Builder();
        // 构建用于烘焙的四边形面(`quads`)
        builder.addUnculledFace(...); // 或 addCulledFace(Direction, BakedQuad)
        // 创建四边形集合(`quad collection`)
        return builder.build();
    }
}

// 未烘焙模型包含从JSON读取的所有信息。
// 它提供基本设置和几何体。
// 使用AbstractUnbakedModel会设置Vanilla和NeoForge的属性方法
public class MyUnbakedModel extends AbstractUnbakedModel {

    private final MyUnbakedGeometry geometry;

    public MyUnbakedModel(StandardModelParameters params, MyUnbakedGeometry geometry) {
        super(params);
        this.geometry = geometry;
    }

    @Override
    public UnbakedGeometry geometry() {
        // 用于构造烘焙四边形面(`baked quads`)的几何体
        return this.geometry;
    }

    @Override
    public void fillAdditionalProperties(ContextMap.Builder propertiesBuilder) {
        super.fillAdditionalProperties(propertiesBuilder);
        // 通过调用withParameter(ContextKey<T>, T)添加附加属性
        // 然后可以在UnbakedGeometry#bake中提供的ContextMap中访问它们
    }
}
```

完成后，别忘了实际注册你的加载器：

```java
@SubscribeEvent // 仅在物理客户端的mod事件总线上
public static void registerLoaders(ModelEvent.RegisterLoaders event) {
    event.register(MyUnbakedModelLoader.ID, MyUnbakedModelLoader.INSTANCE);
}

// 如果你在模型加载器中缓存数据：
@SubscribeEvent // 仅在物理客户端的mod事件总线上
public static void addClientResourceListeners(AddClientReloadListenersEvent event) {
    // 用我们的id注册监听器
    event.addListener(MyUnbakedModelLoader.ID, MyUnbakedModelLoader.INSTANCE);
    // 添加依赖项，使我们的模型加载器在模型加载之前运行
    // 允许在新数据填充之前清除缓存
    event.addDependency(MyUnbakedModelLoader.ID, VanillaClientListeners.MODELS);
}
```

#### 模型加载器数据生成(`Model Loader Datagen`)

当然，我们也可以为模型进行**数据生成**(`datagen`)。为此，我们需要一个继承`CustomLoaderBuilder`的类：

```java
public class MyLoaderBuilder extends CustomLoaderBuilder {
    public MyLoaderBuilder() {
        super(
            // 你的模型加载器的id。
            MyUnbakedModelLoader.ID,
            // 如果加载器缺失，是否允许内联的Vanilla元素作为后备。
            false
        );
    }
    
    // 在此处添加字段及其设置器(`setters`)。这些字段可以在下面使用。

    @Override
    protected CustomLoaderBuilder copyInternal() {
        // 创建加载器构建器的新实例，并将此构建器的属性复制
        // 到新实例。
        MyLoaderBuilder builder = new MyLoaderBuilder();
        // builder.<field> = this.<field>;
        return builder;
    }
    
    // 将模型序列化为JSON。
    @Override
    public JsonObject toJson(JsonObject json) {
        // 将你的字段添加到给定的JsonObject中。
        // 然后调用super，它会添加loader属性和其他一些内容。
        return super.toJson(json);
    }
}
```

要使用此加载器构建器，请在方块（或物品）[模型数据生成][modeldatagen]期间执行以下操作：

```java
// 这假设是ModelProvider的扩展和一个DeferredBlock<Block> EXAMPLE_BLOCK。
// customLoader()的参数是一个用于构造构建器的Supplier和一个用于设置相关属性的Consumer。
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    blockModels.createTrivialBlock(
        // 要为其生成模型的方块
        EXAMPLE_BLOCK.get(),
        TexturedModel.createDefault(
            // 用于获取纹理的映射
            block -> new TextureMapping().put(
                TextureSlot.ALL, TextureMapping.getBlockTexture(block)
            ),
            // 用于创建JSON的模型模板构建器
            ExtendedModelTemplateBuilder.builder()
                // 假设我们使用自定义模型加载器
                .customLoader(MyLoaderBuilder::new, loader -> {
                    // 在此设置任何必需的字段
                })
                // 模型所需的纹理
                .requiredTextureSlot(TextureSlot.ALL)
                // 完成后调用build
                .build()
        )
    );
}
```

#### 可见性(`Visibility`)

`CustomLoaderBuilder`的默认实现包含了应用可见性的方法。你可以选择在模型加载器中使用或忽略`visibility`属性。目前，只有[组合模型加载器][composite]和[OBJ加载器][obj]使用此属性。

## 方块状态模型加载器(`Block State Model Loaders`)

由于方块状态模型被认为独立于模型 JSON 文件，因此 NeoForge 也有自定义加载器，通过在变体(`variant`)或多部分(`multipart`)中指定`type`来处理。自定义方块状态模型加载器可以忽略加载器所需的所有字段。

### 组合方块状态模型(`Composite Block State Model`)

组合方块状态模型可用于一起渲染多个`BlockStateModel`。

```json5
{
    "variants": {
        "": {
            "type": "neoforge:composite",
            // 指定模型部件。
            "models": [
                // 这些必须是内联的方块状态模型
                {
                    "variants": {
                        // ...
                    }
                },
                {
                    "multipart": [
                        // ...
                    ]
                }
                // ...
            ]
        }
    }
}
```

要为此方块状态模型进行[**数据生成**(`datagen`)][modeldatagen]，请使用自定义加载器类`CompositeBlockStateModelBuilder`。

### 重用默认模型加载器

在某些情况下，重用 Vanilla 模型加载器并在其之上构建模型逻辑，而不是完全替换它，是有意义的。我们可以使用一个巧妙技巧：在模型加载器中，我们简单地移除`loader`属性并将其发送回模型反序列化器(`deserializer`)，欺骗它认为这是一个常规的**未烘焙模型**(`unbaked model`)。然后，在**烘焙过程**(`baking process`)之前，我们可以修改模型或其几何体，然后按我们想要的方式进行操作。

```java
public class MyUnbakedModelLoader implements UnbakedModelLoader<MyUnbakedModel> {
    public static final MyUnbakedModelLoader INSTANCE = new MyUnbakedModelLoader();
    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_loader");
    
    private MyUnbakedModelLoader() {}

    @Override
    public MyUnbakedModel read(JsonObject jsonObject, JsonDeserializationContext context) throws JsonParseException {
        // 通过移除loader字段欺骗反序列化器认为这是一个普通模型
        // 然后将其传递给反序列化器。
        jsonObject.remove("loader");
        UnbakedModel model = context.deserialize(jsonObject, UnbakedModel.class);
        return new MyUnbakedModel(model, /* 其他参数在这里 */);
    }
}

// 我们继承委托类(`Delegate class`)，因为它存储了包装的模型
public class MyUnbakedModel extends DelegateUnbakedModel {

    // 存储模型供下面使用
    public MyUnbakedModel(UnbakedModel model, /* 其他参数在这里 */) {
       super(model);
    }
}
```

### 创建自定义方块状态模型加载器

要创建自己的方块状态模型加载器，你需要五个类，外加一个事件处理器：

- 一个`CustomUnbakedBlockStateModel`类来加载方块状态模型
- 一个`BlockStateModel`类来烘焙模型，通常是`DynamicBlockStateModel`实例
- 一个`BlockModelPart.Unbaked`来加载模型 JSON
- 一个`ModelState`来应用任何变换到给定的面或模型
- 一个`BlockModelPart`来保存四边形面(`quads`)、环境光遮蔽(`ambient occlusion`)和粒子纹理，通常是`SimpleModelWrapper`
- 一个用于`RegisterBlockStateModels`的[**客户端**(`client-side`)][sides][事件处理器][event]，用于注册**未烘焙方块状态模型**(`unbaked block state model`)加载器的编解码器(`codec`)

为了说明这些类如何连接，我们将跟踪一个方块状态模型的加载过程：

- 在定义加载期间，变体、多部分或[自定义定义][customdefinition]中设置了`type`属性为你的加载器的方块状态模型被解码为你的`CustomUnbakedBlockStateModel`。
- 在模型烘焙期间，调用`CustomUnbakedBlockStateModel#bake`，返回一个包含一些`BlockModelPart`列表的`BlockStateModel`。
- 在模型渲染期间，`BlockStateModel#collectParts`收集要渲染的`BlockModelPart`列表。

让我们通过一个基本的类设置进一步说明。**烘焙模型**(`baked model`)名为`MyBlockStateModel`，**未烘焙类**(`unbaked class`)是一个内部记录`MyBlockStateModel.Unbaked`，模型部件名为`MyBlockModelPart`，**未烘焙部件类**(`unbaked part class`)是一个内部记录`MyBlockModelPart.Unbaked`，`ModelState`名为`MyModelState`：

```java
// 用于应用必要变换的模型状态(`Model state`)
// 如果你使用中间对象来保存模型状态，它必须可转换为ModelState
public class MyModelState implements ModelState {

    // 用于未烘焙方块模型部件
    public static final Codec<MyModelState> CODEC = Codec.unit(new MyModelState());

    public MyModelState() {}

    @Override
    public Transformation transformation() {
        // 返回要应用到烘焙顶点(`baking vertices`)的模型旋转
        return Transformation.identity();
    }

    @Override
    public Matrix4fc faceTransformation(Direction direction) {
        // 返回变换后应用于模型上给定面的矩阵
        // 目前在Vanilla中未使用
        return NO_TRANSFORM;
    }

    @Override
    public Matrix4fc inverseFaceTransformation(Direction direction) {
        // 返回应用于模型上给定面的faceTransformation的逆矩阵
        // 这被传递给FaceBakery
        return NO_TRANSFORM;
    }
}

// 表示烘焙模型的模型部件(`Model part`)
// useAmbientOcclusion 和 particleIcon 作为记录的一部分实现
public record MyBlockModelPart(QuadCollection quads, boolean useAmbientOcclusion, TextureAtlasSprite particleIcon) implements BlockModelPart {

    // 获取要渲染的烘焙四边形面(`baked quads`)
    @Override
    List<BakedQuad> getQuads(@Nullable Direction direction) {
        return this.quads.getQuads(direction);
    }

    // 从方块状态json读取的未烘焙模型
    public record Unbaked(ResourceLocation modelLocation, MyModelState modelState) implements BlockModelPart.Unbaked {

        // 用于未烘焙方块状态模型
        public static final MapCodec<MyBlockModelPart.Unbaked> CODEC = RecordCodecBuilder.mapCodec(
            instance -> instance.group(
                ResourceLocation.CODEC.fieldOf("model").forGetter(MyBlockModelPart.Unbaked::modelLocation),
                MyModelState.CODEC.fieldOf("state").forGetter(MyBlockModelPart.Unbaked::modelState)
            ).apply(instance, MyBlockModelPart.Unbaked::new)
        );

        @Override
        public void resolveDependencies(ResolvableModel.Resolver resolver) {
            // 标记模型部件使用的任何模型
            resolver.markDependency(this.modelLocation);
        }

        @Override
        public BlockModelPart bake(ModelBaker baker) {
            // 获取要烘焙的模型
            ResolvedModel resolvedModel = baker.getModel(this.modelLocation);

            // 获取模型部件所需的设置
            TextureSlots slots = resolvedModel.getTopTextureSlots();
            boolean ao = resolvedModel.getTopAmbientOcclusion();
            TextureAtlasSprite particle = resolvedModel.resolveParticleSprite(slots, baker);
            QuadCollection quads = resolvedModel.bakeTopGeometry(slots, baker, this.modelState);
            
            // 返回烘焙后的部件
            return new MyBlockModelPart(quads, ao, particle);
        }
    }
}

// 表示烘焙后方块状态的状态模型(`State model`)
public record MyBlockStateModel(MyBlockModelPart model) implements DynamicBlockStateModel {

    // 设置粒子图标
    // 虽然需要实现，但任何实际逻辑应委托给感知层级(`level-aware`)的版本
    @Override
    public TextureAtlasSprite particleIcon() {
        return this.model.particleIcon();
    }

    // 这实际上充当一个键，用于重用先前生成的几何体。这通常应尽可能确定。
    @Override
    public Object createGeometryKey(BlockAndTintGetter level, BlockPos pos, BlockState state, RandomSource random) {
        return this;
    }

    // 负责收集要渲染的部件的方法。此方法的参数是：
    // - 方块和色调(`tint`)的获取器，通常是层级(`level`)。
    // - 要渲染的方块位置。
    // - 方块的状态。
    // - 一个随机实例。
    // - 要渲染的模型部件列表。在此添加你的模型部件。
    @Override
    public void collectParts(BlockAndTintGetter level, BlockPos pos, BlockState state, RandomSource random, List<BlockModelPart> parts) {
        // 如果你希望渲染的方块依赖于方块实体（例如，你的方块实体实现了`BlockEntity#getModelData`）
        // 你可以使用方块位置调用`BlockAndTintGetter#getModelData`
        // 你可以使用`ModelProperty`键通过`get`读取属性
        // 记住，你的方块实体应调用`BlockEntity#requestModelDataUpdate`以将模型数据同步到客户端
        ModelData data = level.getModelData(pos);

        // 添加要渲染的模型
        parts.add(this.model);
    }

    @Override
    public TextureAtlasSprite particleIcon(BlockAndTintGetter level, BlockPos pos, BlockState state) {
        // 如果你想使用层级来确定渲染什么粒子，请覆盖此方法
        return self().particleIcon();
    }

    // 从方块状态json读取的未烘焙模型
    public record Unbaked(MyBlockModelPart.Unbaked model) implements CustomUnbakedBlockStateModel {

        // 要注册的编解码器(`codec`)
        public static final MapCodec<MyBlockStateModel.Unbaked> CODEC = MyBlockModelPart.Unbaked.CODEC.xmap(
            MyBlockStateModel.Unbaked::new, MyBlockStateModel.Unbaked::model
        );
        public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_model_loader");

        @Override
        public void resolveDependencies(ResolvableModel.Resolver resolver) {
            // 标记状态模型使用的任何模型
            this.model.resolveDependencies(resolver);
        }

        @Override
        public BlockStateModel bake(ModelBaker baker) {
            // 烘焙模型部件并传入方块状态模型
            return new MyBlockStateModel(this.model.bake(baker));
        }
    }
}
```

完成后，别忘了实际注册你的加载器：

```java
@SubscribeEvent // 仅在物理客户端的mod事件总线上
public static void registerDefinitions(RegisterBlockStateModels event) {
    event.registerModel(MyBlockStateModel.Unbaked.ID, MyBlockStateModel.Unbaked.CODEC);
}
```

#### 状态模型加载器数据生成(`State Model Loader Datagen`)

当然，我们也可以为模型进行**数据生成**(`datagen`)。为此，我们需要一个继承`CustomBlockStateModelBuilder`的类：

```java
// 用于构造方块状态JSON的构建器
public class MyBlockStateModelBuilder extends CustomBlockStateModelBuilder {

    private MyBlockModelPart.Unbaked model;

    public MyBlockStateModelBuilder() {}
    
    // 在此处添加字段及其设置器(`setters`)。这些字段可以在下面使用。

    @Override
    public MyBlockStateModelBuilder with(VariantMutator variantMutator) {
        // 如果你想应用任何假设你的未烘焙模型部件是`Variant`的变换器(`mutators`)
        // 如果不是，这应该什么都不做
        return this;
    }

    // 这是用于广义未烘焙方块状态模型
    @Override
    public MyBlockStateModelBuilder with(UnbakedMutator unbakedMutator) {
        var result = new MyBlockStateModelBuilder();

        if (this.model != null) {
            result.model = unbakedMutator.apply(this.model);
        }

        return result;
    }

    // 将构建器转换为其未烘焙变体以编码
    @Override
    public CustomUnbakedBlockStateModel toUnbaked() {
        return new MyBlockStateModel.Unbaked(this.model);
    }
}
```

要使用此状态定义加载器构建器，请在方块（或物品）[模型数据生成][modeldatagen]期间执行以下操作：

```java
// 这假设是ModelProvider的扩展和一个DeferredBlock<Block> EXAMPLE_BLOCK。
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    blockModels.blockStateOutput.accept(
        MultiVariantGenerator.dispatch(
            // 要为其生成模型的方块
            EXAMPLE_BLOCK.get(),
            // 我们的自定义方块状态构建器
            MultiVariant.of(new CustomBlockStateModelBuilder().with(...))
        )
    );
}
```

这将生成类似这样的模型：

```json5
{
  "variants": {
    "": {
        "type": "examplemod:my_custom_model_loader"
        // 其他字段
    }
  }
}
```

## 方块状态定义加载器(`Block State Definition Loaders`)

虽然单独的方块状态模型处理单个方块状态的加载，但方块状态定义加载器处理整个方块状态文件的加载，通过在`neoforge:definition_type`中指定来处理。自定义方块状态定义加载器可以忽略加载器所需的所有字段。

### 创建自定义方块状态定义加载器

要创建自己的方块状态定义加载器，你需要两个类，外加一个事件处理器：

- 一个`CustomBlockModelDefinition`类来加载方块状态定义
- 一个`BlockStateModel.UnbakedRoot`类来将方块状态烘焙为其`BlockStateModel`
- 一个用于`RegisterBlockStateModels`的[**客户端**(`client-side`)][sides][事件处理器][event]，用于注册**未烘焙方块状态模型**(`unbaked block state model`)加载器的编解码器(`codec`)

为了说明这些类如何连接，我们将跟踪一个方块状态模型的加载过程：

- 在定义加载期间，设置了`neoforge:definition_type`属性为你的加载器的方块状态定义被解码为`CustomBlockModelDefinition`。
- 然后，调用`CustomBlockModelDefinition#instantiate`将所有可能的方块状态映射到其`BlockStateModel.UnbakedRoot`。对于简单情况，这通过`BlockStateModel.Unbaked#asRoot`构造。复杂情况创建自己的`BlockStateModel.UnbakedRoot`。
- 在模型烘焙期间，调用`BlockStateModel.UnbakedRoot#bake`，为某个`BlockState`返回一个`BlockStateModel`。

让我们通过一个基本的类设置进一步说明。方块模型定义名为`MyBlockModelDefinition`，我们将重用`BlockStateModel.Unbaked#asRoot`来构造`BlockStateModel.UnbakedRoot`：

```java
public record MyBlockModelDefinition(MyBlockStateModel.Unbaked model) implements CustomBlockModelDefinition {

    // 要注册的编解码器(`codec`)
    public static final MapCodec<MyBlockModelDefinition> CODEC = MyBlockStateModel.Unbaked.CODEC.xmap(
        MyBlockModelDefinition::new, MyBlockModelDefinition::model
    );
    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_definition_loader");

    // 此方法将所有可能的状态映射到某个未烘焙根(`unbaked root`)
    // 由于根通常会共享方块状态模型，它们通常使用`ModelBaker.SharedOperationKey`来缓存正在加载的模型
    @Override
    public Map<BlockState, BlockStateModel.UnbakedRoot> instantiate(StateDefinition<Block, BlockState> states, Supplier<String> sourceSupplier) {
        Map<BlockState, BlockStateModel.UnbakedRoot> result = new HashMap<>();

        // 处理所有可能的状态
        var unbakedRoot = this.model.asRoot();
        states.getPossibleStates().forEach(state -> result.put(state, unbakedRoot));

        return result;
    }

    @Override
    public MapCodec<? extends CustomBlockModelDefinition> codec() {
        return CODEC;
    }
}
```

完成后，别忘了实际注册你的加载器：

```java
@SubscribeEvent // 仅在物理客户端的mod事件总线上
public static void registerDefinitions(RegisterBlockStateModels event) {
    event.registerDefinition(MyBlockModelDefinition.ID, MyBlockModelDefinition.CODEC);
}
```

#### 状态定义加载器数据生成(`State Definition Loader Datagen`)

当然，我们也可以为定义进行**数据生成**(`datagen`)。为此，我们需要一个继承`BlockModelDefinitionGenerator`的类：

```java
public class MyBlockModelDefinitionGenerator implements BlockModelDefinitionGenerator {

    private final Block block;
    private final MyBlockStateModelBuilder builder;

    private MyBlockModelDefinitionGenerator(Block block, MyBlockStateModelBuilder builder) {
        this.block = block;
        this.builder = builder;
    }

    public static MyBlockModelDefinitionGenerator dispatch(Block block, MyBlockStateModelBuilder builder) {
        return new MyBlockModelDefinitionGenerator(block, builder);
    }

    @Override
    public Block block() {
        // 返回你正在为其生成定义文件的方块
        return this.block;
    }

    @Override
    public BlockModelDefinition create() {
        // 创建用于编码和解码文件的方块模型定义
        return new MyBlockModelDefinition(this.builder.toUnbaked());
    }
} 
```

要使用此状态定义加载器构建器，请在方块（或物品）[模型数据生成][modeldatagen]期间执行以下操作：

```java
// 这假设一个DeferredBlock<Block> EXAMPLE_BLOCK。
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    blockModels.blockStateOutput.accept(
        MyBlockModelDefinitionGenerator.dispatch(
            // 要为其生成模型的方块
            EXAMPLE_BLOCK.get(),
            new CustomBlockStateModelBuilder(...)
        )
    );
}
```

这将生成类似这样的模型：

```json5
{
    "neoforge:definition_type": "examplemod:my_custom_definition_loader"
    // 其他字段
}
```

[citems]: items.md
[composite]: #composite-model
[customdefinition]: #block-state-definition-loaders
[datagen]: ../../index.md#data-generation
[event]: ../../../concepts/events.md#registering-an-event-handler
[itemcomposite]: items.md#composite-models
[modeldatagen]: datagen.md
[obj]: #obj-model
[sides]: ../../../concepts/sides.md
