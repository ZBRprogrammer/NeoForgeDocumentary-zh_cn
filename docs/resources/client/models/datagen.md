# 模型数据生成(`Model Datagen`)

与大多数JSON数据类似，方块和物品模型及其必要的方块状态文件与[客户端物品(`client items`)][citems]，都可以通过[数据生成(`datagen`)][datagen]来创建。这一切都由原版`ModelProvider`处理，同时NeoForge通过`ExtendedModelTemplateBuilder`提供了扩展功能。由于方块模型和物品模型的JSON结构相似，因此数据生成的代码也相对类似。

## 模型模板(`Model Templates`)

每个模型都起始于一个`ModelTemplate`。在原版中，`ModelTemplate`作为预生成模型文件的父级，定义了父模型、必需的纹理槽(`texture slots`)以及要应用的文件后缀。对于NeoForge的情况，`ExtendedModelTemplate`通过`ExtendedModelTemplateBuilder`构建，允许用户生成模型的基础元素和面(`faces`)，以及任何NeoForge新增的功能。

`ModelTemplate`可通过`ModelTemplates`中的方法或调用构造函数创建。构造函数接收相对于`models`目录的父模型`ResourceLocation`（可选）、要附加到文件路径末尾的字符串（例如按压按钮会添加`_pressed`后缀），以及必须定义的`TextureSlot`可变参数（若未定义会导致数据生成崩溃）。`TextureSlot`是一个定义纹理映射(`textures map`)中纹理"键"的字符串。每个键还可有一个父级`TextureSlot`，当特定槽未指定纹理时将解析到该父级。例如，`TextureSlot#PARTICLE`会先查找定义的`particle`纹理，然后检查`texture`值，最后检查`all`。如果该槽或其父级未定义，数据生成期间将抛出崩溃。

```java
// 假设存在引用为'#base'的纹理
// 可通过指定'base'或'all'解析
public static final TextureSlot BASE = TextureSlot.create("base", TextureSlot.ALL);

// 假设存在模型'examplemod:block/example_template'
public static final ModelTemplate EXAMPLE_TEMPLATE = new ModelTemplate(
    // 父模型位置
    Optional.of(
        ModelLocationUtils.decorateBlockModelLocation("examplemod:example_template")
    ),
    // 应用到此模板所有模型末尾的后缀
    Optional.of("_example"),
    // 所有必须定义的纹理槽
    // 应根据父模型中未定义的内容尽可能具体
    TextureSlot.PARTICLE,
    BASE
);
```

NeoForge新增的`ExtendedModelTemplate`可通过`ExtendedModelTemplateBuilder#builder`构建，或通过`ModelTemplate#extend`扩展现有原版模板。构建器(`builder`)可通过`#build`解析为模板。构建器的方法提供了对模型JSON构造的完全控制：

| 方法                                           | 效果                                                                                                                                                                                                                                                                                                                                                  |
|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `#parent(ResourceLocation parent)`                                         | 设置相对于`models`目录的父模型位置。 |
| `#suffix(String suffix)`                                                   | 将字符串附加到模型文件路径末尾。 |
| `#requiredTextureSlot(TextureSlot slot)`                                   | 添加必须在`TextureMapping`中定义的纹理槽。 |
| `#renderType(ResourceLocation renderType)`                                 | 设置渲染类型。提供接受`String`的重载方法。有效值列表参见`RenderType`类。                                                                                                                                                                                                                        |
| `transform(ItemDisplayContext type, Consumer<TransformVecBuilder> action)` | 添加通过消费者配置的`TransformVecBuilder`，用于设置模型的`display`。 |
| `#ambientOcclusion(boolean ambientOcclusion)`                              | 设置是否启用[**环境光遮蔽**(`ambient occlusion`)][ao]。                                                                                                                                                                                                                                                                                                     |
| `#guiLight(UnbakedModel.GuiLight light)`                                   | 设置GUI光照。可选`GuiLight.FRONT`或`GuiLight.SIDE`。                                                                                                                                                                                                                                                                                         |
| `#element(Consumer<ElementBuilder> action)`                                | 添加通过消费者配置的新`ElementBuilder`（相当于向模型添加新[元素(`element`)][elements]）。                                                                                                                                                                                                 |                                                                                                                                                                                                                                                            |
| `#customLoader(Supplier customLoaderFactory, Consumer action)`            | 使用给定工厂使模型使用[自定义加载器(`custom loader`)][custommodelloader]，并通过消费者配置自定义加载器构建器。这会改变构建器类型，因此可能根据加载器实现使用不同方法。NeoForge内置了一些自定义加载器，详见链接文章（含数据生成部分）。 |
| `#rootTransforms(Consumer<RootTransformsBuilder> action)`                  | 通过消费者配置模型在物品显示和方块状态变换前应用的变换。 |

:::tip
虽然可通过数据生成创建精细复杂的模型，但建议使用[Blockbench(`blockbench`)][blockbench]等建模软件创建更复杂的模型，然后直接使用导出模型或将其作为其他模型的父级。
:::

### 创建模型实例(`Creating the Model Instance`)

有了`ModelTemplate`后，可通过调用`ModelTemplate#create*`方法之一生成模型本身。尽管每个创建方法参数不同，但核心都接收表示文件名的`ResourceLocation`、将`TextureSlot`映射到`textures`目录下某`ResourceLocation`的`TextureMapping`，以及作为`BiConsumer<ResourceLocation, ModelInstance>`的模型输出。该方法会创建用于生成模型的`JsonObject`，若提供重复项则抛出错误。

:::note
调用基础`create`方法不会应用存储的后缀。只有接收方块或物品的`create*`方法才会应用后缀。
:::

```java
// 给定某BiConsumer<ResourceLocation, ModelInstance> modelOutput
// 假设存在DeferredBlock<Block> EXAMPLE_BLOCK
EXAMPLE_TEMPLATE.create(
    // 在'assets/minecraft/models/block/example_block_example.json'创建模型
    EXAMPLE_BLOCK.get(),
    // 在槽中定义纹理
    new TextureMapping()
        // "particle": "examplemod:item/example_block"
        .put(TextureSlot.PARTICLE, TextureMapping.getBlockTexture(EXAMPLE_BLOCK.get()))
        // "base": "examplemod:item/example_block_base"
        .put(TextureSlot.BASE, TextureMapping.getBlockTexture(EXAMPLE_BLOCK.get(), "_base")),
    // 生成模型JSON的消费者
    modelOutput
);
```

有时生成的模型使用相似的模型模板和纹理命名模式（例如普通方块的纹理就是方块名称）。此时可创建`TexturedModel.Provider`来帮助消除冗余。该提供者实质上是接收某`Block`并返回`TexturedModel`（`ModelTemplate`/`TextureMapping`对）的功能接口。接口通过`TexturedModel#createDefault`构建，接收将`Block`映射到其`TextureMapping`的函数及要使用的`ModelTemplate`。然后可通过调用带生成目标`Block`的`TexturedModel.Provider#create`生成模型。

```java
public static final TexturedModel.Provider EXAMPLE_TEMPLATE_PROVIDER = TexturedModel.createDefault(
    // 方块到纹理映射
    block -> new TextureMapping()
        .put(TextureSlot.PARTICLE, TextureMapping.getBlockTexture(block))
        .put(TextureSlot.BASE, TextureMapping.getBlockTexture(block, "_base")),
    // 生成用的模板
    EXAMPLE_TEMPLATE
);

// 给定某BiConsumer<ResourceLocation, ModelInstance> modelOutput
// 假设存在DeferredBlock<Block> EXAMPLE_BLOCK
EXAMPLE_TEMPLATE_PROVIDER.create(
    // 在'assets/minecraft/models/block/example_block_example.json'创建模型
    EXAMPLE_BLOCK.get(),
    // 生成模型JSON的消费者
    modelOutput
);
```

## `ModelProvider`

方块和物品模型的数据生成都通过`registerModels`提供的生成器实现，分别为`BlockModelGenerators`和`ItemModelGenerators`。每个生成器会生成模型JSON及任何额外必需文件（方块状态、客户端物品）。每个生成器包含各种辅助方法，将文件批量构建为单一易用方法，如`ItemModelGenerators#generateFlatItem`创建基础`item/generated`模型，或`BlockModelGenerators#createTrivialCube`创建基础`block/cube_all`模型。

```java
public class ExampleModelProvider extends ModelProvider {

    public ExampleModelProvider(PackOutput output) {
        // 将"examplemod"替换为你的模组ID
        super(output, "examplemod");
    }

    @Override
    protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
        // 在此生成模型及相关文件
    }
}
```

与所有数据提供者一样，别忘了将提供者注册到事件：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(ExampleModelProvider::new);
}
```

### 方块模型数据生成(`Block Model Datagen`)

要实际生成方块状态和方块模型文件，可在`ModelProvider#registerModels`中调用`BlockModelGenerators`的众多公共方法之一，或手动将生成的文件传递给`blockStateOutput`（方块状态文件）、`itemModelOutput`（非基础客户端物品）和`modelOutput`（模型JSON）。

:::note
如果方块注册了关联的`BlockItem`且未生成客户端物品，`ModelProvider`会自动使用默认方块模型位置`assets/<namespace>/models/block/<path>.json`作为模型生成客户端物品。
:::

```java
public class ExampleModelProvider extends ModelProvider {

    public ExampleModelProvider(PackOutput output) {
        // 将"examplemod"替换为你的模组ID
        super(output, "examplemod");
    }

    @Override
    protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
        // 占位符，实际使用应替换为真实值。上文介绍了如何使用模型构建器，
        // 下文介绍构建器提供的辅助工具。
        Block block = MyBlocksClass.EXAMPLE_BLOCK.get();

        // 创建每面纹理相同的简单方块模型。
        // 纹理必须位于assets/<namespace>/textures/block/<path>.png，
        // 其中<namespace>和<path>是方块注册名的命名空间和路径。
        // 大多数（完整）方块使用此方法，如木板、圆石或砖块。
        blockModels.createTrivialCube(block);

        // 接受`TexturedModel.Provider`的重载方法。
        blockModels.createTrivialBlock(block, EXAMPLE_TEMPLATE_PROVIDER);

        // 方块物品的模型会自动生成
        // 但假设你想生成不同的物品，如平面物品
        blockModels.registerSimpleFlatItemModel(block);

        // 添加原木方块模型。需要两个纹理：
        // assets/<namespace>/textures/block/<path>.png（侧面）和
        // assets/<namespace>/textures/block/<path>_top.png（顶部）。
        // 注意此处方块输入限制为RotatedPillarBlock（原版原木使用的类）。
        blockModels.woodProvider(block).log(block);
        
        // 类似WoodProvider#logWithHorizontal。石英柱等类似方块使用。
        blockModels.createRotatedPillarWithHorizontalVariant(block, TexturedModel.COLUMN_ALT, TexturedModel.COLUMN_HORIZONTAL_ALT);

        // 使用`ExtendedModelTemplate`指定渲染类型。
        blockModels.createRotatedPillarWithHorizontalVariant(block,
            TexturedModel.COLUMN_ALT.updateTemplate(template ->
                template.extend().renderType("minecraft:cutout").build()
            ),
            TexturedModel.COLUMN_HORIZONTAL_ALT.updateTemplate(template ->
                template.extend().renderType(this.mcLocation("cutout_mipped")).build()
            )
        );

        // 指定带侧面纹理、正面纹理和顶部纹理的可水平旋转方块模型。
        // 底部也会使用侧面纹理。如果不需要正面或顶部纹理，
        // 只需传入两次侧面纹理。熔炉等类似方块使用此方法。
        blockModels.createHorizontallyRotatedBlock(
            block,
            TexturedModel.Provider.ORIENTABLE_ONLY_TOP.updateTexture(mapping ->
                mapping.put(TextureSlot.SIDE, this.modLocation("block/example_texture_side"))
                .put(TextureSlot.FRONT, this.modLocation("block/example_texture_front"))
                .put(TextureSlot.TOP, this.modLocation("block/example_texture_top"))
            )
        );

        // 指定附着在面上的可水平旋转方块模型，如按钮。
        // 考虑方块放置在地面和天花板的情况，并相应旋转。
        blockModels.familyWithExistingFullBlock(block).button(block);

        // 创建用于方块状态文件的模型
        ResourceLocation modelLoc = TexturedModel.CUBE.create(block, blockModels.modelOutput);

        // 创建通用变体以变换
        Variant variant = new Variant(modelLoc);

        // 基础单变体模型
        blockModels.blockStateOutput.accept(
            MultiVariantGenerator.dispatch(
                block,
                new MultiVariant(
                    WeightedList.of(
                        new Weighted<>(
                            // 设置模型
                            variant
                                // 设置绕x轴和y轴的旋转
                                .with(VariantMutator.X_ROT.withValue(Quadrant.R90))
                                .with(VariantMutator.Y_ROT.withValue(Quadrant.R180))
                                // 设置uv锁定
                                .with(VariantMutator.UV_LOCK.withValue(true)),
                            // 设置权重
                            5
                        )
                    )
                )
            )
        );

        // 基于方块状态属性添加一个或多个模型
        blockModels.blockStateOutput.accept(
            MultiVariantGenerator.dispatch(
                block,
                // 创建基础多变体
                BlockModelGenerators.variant(variant)
            ).with(
                // 应用属性分派
                // 将根据提供的变换器(`mutators`)改变变体
                PropertyDispatch.modify(BlockStateProperties.AXIS)
                    .select(Direction.Axis.Y, BlockModelGenerators.NOP)
                    .select(Direction.Axis.Z, BlockModelGenerators.X_ROT_90)
                    .select(Direction.Axis.X, BlockModelGenerators.X_ROT_90.then(BlockModelGenerators.Y_ROT_90))
            )
        );

        // 生成多部分(`multipart`)模型
        blockModels.blockStateOutput.accept(
            MultiPartGenerator.multiPart(block)
                // 提供基础模型
                .with(BlockModelGenerators.variant(variant))
                // 添加变体出现的条件
                .with(
                    // 添加应用条件
                    new CombinedCondition(
                        CombinedCondition.Operation.OR,
                        List.of(
                            // 至少一个条件为真
                            BlockModelGenerators.condition().term(BlockStateProperties.FACING, Direction.NORTH, Direction.SOUTH)
                            // 可嵌套任意数量的条件或组
                            new CombinedCondition(
                                CombinedCondition.Operation.AND,
                                List.of(
                                    BlockModelGenerators.condition().term(BlockStateProperties.FACING, Direction.NORTH)
                                )
                            )
                        )
                    ),
                    // 提供要变换的变体
                    BlockModelGenerators.variant(variant)
                )
        );
    }
}
```

## 物品模型数据生成(`Item Model Datagen`)

生成物品模型要简单得多，这主要归功于`ItemModelGenerators`和用于属性信息的`ItemModelUtils`中的辅助方法。与上文类似，可在`ModelProvider#registerModels`中调用`ItemModelGenerators`的众多公共方法之一，或手动将生成的文件传递给`itemModelOutput`（非基础客户端物品）和`modelOutput`（模型JSON）。

```java
public class ExampleModelProvider extends ModelProvider {

    public ExampleModelProvider(PackOutput output) {
        // 将"examplemod"替换为你的模组ID
        super(output, "examplemod");
    }

    @Override
    protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
        // 最常见物品
        // item/generated模型，layer0纹理为物品名称
        itemModels.generateFlatItem(MyItemsClass.EXAMPLE_ITEM.get(), ModelTemplates.FLAT_ITEM);

        // 弓类物品
        ItemModel.Unbaked bow = ItemModelUtils.plainModel(ModelLocationUtils.getModelLocation(MyItemsClass.EXAMPLE_ITEM.get()));
        ItemModel.Unbaked pullingBow0 = ItemModelUtils.plainModel(this.createFlatItemModel(MyItemsClass.EXAMPLE_ITEM.get(), "_pulling_0", ModelTemplates.BOW));
        ItemModel.Unbaked pullingBow1 = ItemModelUtils.plainModel(this.createFlatItemModel(MyItemsClass.EXAMPLE_ITEM.get(), "_pulling_1", ModelTemplates.BOW));
        ItemModel.Unbaked pullingBow2 = ItemModelUtils.plainModel(this.createFlatItemModel(MyItemsClass.EXAMPLE_ITEM.get(), "_pulling_2", ModelTemplates.BOW));
        this.itemModelOutput.accept(
            MyItemsClass.EXAMPLE_ITEM.get(),
            // 物品的条件模型
            ItemModelUtils.conditional(
                // 检查物品是否正在使用
                ItemModelUtils.isUsingItem(),
                // 若为真，根据使用时长选择模型
                ItemModelUtils.rangeSelect(
                    new UseDuration(false),
                    // 应用于阈值的标量
                    0.05F,
                    pullingBow0,
                    // 0.65阈值
                    ItemModelUtils.override(pullingBow1, 0.65F),
                    // 0.9阈值
                    ItemModelUtils.override(pullingBow2, 0.9F)
                ),
                // 若为假，使用基础弓模型
                bow
            )
        );
    }
}
```

[ao]: https://en.wikipedia.org/wiki/Ambient_occlusion
[blockbench]: https://www.blockbench.net
[citems]: items.md
[custommodelloader]: modelloaders.md#datagen
[datagen]: ../../index.md#data-generation
[elements]: index.md#elements