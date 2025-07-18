# 模型(`Models`)

模型是决定方块或物品视觉形状和纹理的JSON文件。模型由立方体元素(`cuboid elements`)组成，每个元素有其自身尺寸，各面分配有纹理。

物品使用其[客户端物品(`client items`)][citems]定义的关联模型，方块则使用[方块状态文件(`blockstate file`)][bsfile]中的关联模型。这些位置相对于`models`目录，因此名为`examplemod:item/example_model`的模型由`assets/examplemod/models/item/example_model.json`处的JSON定义。

## 规范(`Specification`)

_另见：[Minecraft Wiki上的模型(`Model`)][mcwikimodel]_

模型是一个JSON文件，根标签(`root tag`)包含以下可选属性：

- `loader`：NeoForge新增。设置自定义模型加载器。详见[模型加载器(`Model Loaders`)][custommodelloader]。
- `parent`：设置父模型，格式为相对于`models`文件夹的[资源定位符(`resource location`)][rl]。所有父级属性将被应用，并由声明模型中的属性覆盖。常见父级包括：
    - `minecraft:block/block`：所有方块模型的通用父级。
    - `minecraft:block/cube`：所有使用1x1x1立方体模型的父级。
    - `minecraft:block/cube_all`：立方体模型变体，六面使用相同纹理（如圆石、木板）。
    - `minecraft:block/cube_bottom_top`：立方体模型变体，四面水平面使用相同纹理，顶部和底部使用独立纹理（如砂岩、錾制石英）。
    - `minecraft:block/cube_column`：立方体模型变体，有侧面纹理及顶部/底部纹理（如原木、石英柱、紫珀柱）。
    - `minecraft:block/cross`：使用两个相同纹理平面的模型，一个顺时针旋转45°，另一个逆时针旋转45°，俯视呈X形（如草、树苗、花等植物）。
    - `minecraft:item/generated`：经典2D平面物品模型的父级。游戏内大多数物品使用此模型。忽略`elements`块，因其四边形(`quads`)由纹理生成。
    - `minecraft:item/handheld`：看似被玩家手持的2D平面物品模型的父级。主要由工具使用。作为`item/generated`的子模型，同样忽略`elements`块。
    - 方块物品通常（非总是）使用对应方块模型作为其[物品模型(`item model`)][itemmodels]（如圆石客户端物品使用`minecraft:block/cobblestone`模型）。
- `ambientocclusion`：是否启用[**环境光遮蔽**(`ambient occlusion`)][ao]。仅对方块模型有效，默认为`true`。若自定义方块模型有奇怪着色，可尝试设为`false`。
- `render_type`：NeoForge新增。设置要使用的**渲染类型组**(`render type group`)。详见[渲染类型组(`Render Type Groups`)][rendertype]。
- `gui_light`：可为`"front"`或`"side"`。`"front"`时光源来自前方，适用于2D平面模型；`"side"`时光源来自侧面，适用于3D模型（尤其方块模型）。默认为`"side"`，仅对物品模型有效。
- `textures`：将名称（称为**纹理变量**）映射到[纹理位置(`texture locations`)][textures]的子对象。纹理变量可在[元素(`elements`)][elements]中使用，也可在元素中指定但留空供子模型定义。
    - 方块模型应额外指定`particle`纹理，用于方块掉落、踩踏或破坏时的粒子效果。
    - 物品模型可使用**分层纹理**(`layer textures`)，命名为`layer0`、`layer1`等，索引更高的层渲染在较低层之上（如`layer1`在`layer0`上方）。仅在父级为`item/generated`时有效，最多支持5层（`layer0`至`layer4`）。
- `elements`：立方体[元素(`elements`)][elements]列表。
- `display`：包含不同[视角(`perspectives`)][perspectives]显示选项的子对象（有效键见链接文章）。仅对物品模型有效，但常在方块模型中指定以便物品模型继承显示选项。每个视角是可选的子对象，可包含以下按序应用的选项：
    - `translation`：模型平移，格式为`[x, y, z]`。
    - `rotation`：模型旋转，格式为`[x, y, z]`。
    - `scale`：模型缩放，格式为`[x, y, z]`。
    - `right_rotation`：NeoForge新增。缩放后应用的第二旋转，格式为`[x, y, z]`。
- `transform`：见[根变换(`Root Transforms`)][roottransforms]。

:::tip
若不确定如何指定某项内容，可参考实现类似功能的原版模型。
:::

### 渲染类型组(`Render Type Groups`)

通过NeoForge新增的可选`render_type`字段，可为模型设置**渲染类型组**。渲染类型组由两部分组成：`ChunkSectionLayer`（决定作为方块时的渲染方式）和`RenderType`（决定作为物品时的渲染方式）。若未设置（所有原版模型均如此），游戏将回退到`ItemBlockRenderTypes`中硬编码的层和渲染类型。若`ItemBlockRenderTypes`不含对应层或渲染类型，将回退到`ChunkSectionLayer#SOLID`（方块）和`Sheets#TRANSLUCENT_ITEM_CULL_BLOCK_SHEET`（物品）。原版和NeoForge提供以下渲染类型组：

- `minecraft:solid`：用于完全实心模型（如石头）。
- `minecraft:cutout`：用于像素全透明或全不透明的模型（如玻璃）。
- `minecraft:cutout_mipped`：`minecraft:cutout`变体，在远距离缩小纹理以避免视觉伪影[**(`mipmapping`)**][mipmapping]。物品渲染不应用mipmapping（因通常不需要且可能导致伪影），如树叶使用此类型。
- `minecraft:cutout_mipped_all`：`minecraft:cutout_mipped`变体，物品模型也应用mipmapping。
- `minecraft:translucent`：用于含半透明像素的模型（如染色玻璃）。
- `minecraft:tripwire`：用于需渲染到天气目标的特殊模型（如绊线）。
- `neoforge:item_unlit`：NeoForge新增。物品渲染时不考虑光源方向的模型应使用此类型。

选择正确的渲染类型组部分涉及性能考量：实心渲染快于镂空渲染，镂空渲染快于半透明渲染。因此应为用例指定"最严格"的渲染类型以获得最佳性能。

如需添加自定义渲染类型组，可在[模组总线(`mod bus`)][modbus][事件(`event`)][event]`RegisterNamedRenderTypesEvent`中订阅并`#register`。`#register`有三个参数：

- 渲染类型组名称：带模组ID前缀的`ResourceLocation`。
- 区块层：任意`ChunkSectionLayer`。
- 实体渲染类型：必须使用`DefaultVertexFormat.NEW_ENTITY`顶点格式的渲染类型。

### 元素(`Elements`)

元素是立方体对象的JSON表示，包含以下属性：

- `from`：立方体起始角坐标，格式为`[x, y, z]`（1/16方块单位）。如`[0, 0, 0]`为"左下角"，`[8, 8, 8]`为中心点，`[16, 16, 16]`为"右上角"。
- `to`：立方体结束角坐标，格式为`[x, y, z]`（同`from`为1/16方块单位）。

:::tip
Minecraft限制`from`和`to`值在`[-16, 32]`范围，但强烈建议勿超出`[0, 16]`，否则可能导致光照或剔除问题。
:::

- `neoforge_data`：见[额外面数据(`Extra Face Data`)][extrafacedata]。
- `faces`：包含最多6个面的对象，分别命名为`north`、`south`、`east`、`west`、`up`和`down`。每个面含以下数据：
    - `uv`：面的UV坐标，格式为`[u1, v1, u2, v2]`，其中`u1, v1`是左上UV坐标，`u2, v2`是右下UV坐标。
    - `texture`：面使用的纹理。必须是以`#`为前缀的纹理变量（如纹理命名为`wood`，则用`#wood`引用）。技术上可选，未指定时使用缺失纹理。
    - `rotation`：可选。将纹理顺时针旋转90°、180°或270°。
    - `cullface`：可选。指定方向有完整方块接触时，告知渲染引擎跳过渲染该面（方向可为`north`、`south`、`east`、`west`、`up`或`down`）。
    - `tintindex`：可选。指定**着色索引**(`tint index`)，供颜色处理器使用（详见[着色(`Tinting`)][tinting]）。默认为-1（无着色）。
    - `neoforge_data`：见[额外面数据(`Extra Face Data`)][extrafacedata]。

可选属性：
- `shade`：仅限方块模型。是否启用面的方向依赖性着色，默认为`true`。
- `rotation`：对象旋转子对象，包含：
    - `angle`：旋转角度（度），支持-45°至45°（步长22.5°）。
    - `axis`：旋转轴。当前不支持多轴旋转。
    - `origin`：可选。旋转原点`[x, y, z]`（绝对坐标，非相对于立方体位置）。未指定时为`[0, 0, 0]`。

#### 额外面数据(`Extra Face Data`)

**额外面数据**(`Extra face data`)可应用于元素或其单个面，在所有可用上下文中均为可选。若同时指定元素级和面级数据，面级数据将覆盖元素级数据。额外数据可包含：

- `color`：用指定颜色着色面（ARGB格式）。可为字符串或十进制整数（JSON不支持十六进制字面量）。默认为`0xFFFFFFFF`，可替代固定颜色的着色。
- `block_light`：覆盖面使用的**方块光照值**(`block light value`)，默认为0。
- `sky_light`：覆盖面使用的**天空光照值**(`sky light value`)，默认为0。
- `ambient_occlusion`：禁用或启用于此面的环境光遮蔽，默认为模型中设置的值。

### 根变换(`Root Transforms`)

在模型顶层添加`transform`属性，告知加载器在应用[方块状态文件(`blockstate file`)][bsfile]的旋转（方块模型）或`display`块的变换（物品模型）前，对所有几何体应用变换。此为NeoForge新增功能。

根变换有两种指定方式：
1. 作为名为`matrix`的属性，包含嵌套JSON数组形式的3x4变换矩阵（行主序，省略最后一行）。例如：
```json5
{
    // ...
    "transform": {
        "matrix": [
            [0, 0, 0, 0],
            [0, 0, 0, 0],
            [0, 0, 0, 0]
        ]
    }
}
```
2. 作为包含以下任意条目的JSON对象（按序应用）：
    - `translation`：相对平移（三维向量`[x, y, z]`），默认为`[0, 0, 0]`。
    - `rotation`或`left_rotation`：缩放前绕平移原点的旋转，默认为无旋转。指定方式：
        - 单轴映射的JSON对象（如`{"x": 90}`）
        - 单轴映射JSON对象数组（按序应用，如`[{"x": 90}, {"y": 45}, {"x": -22.5}]`）
        - 三值数组（每轴旋转角度，如`[90, 45, -22.5]`）
        - 四值数组（直接指定四元数，如`[0.38268346, 0, 0, 0.9238795]`表示绕X轴旋转45°）
    - `scale`：相对于平移原点的缩放（三维向量`[x, y, z]`），默认为`[1, 1, 1]`。
    - `post_rotation`或`right_rotation`：缩放后绕平移原点的旋转，指定方式同`rotation`。
    - `origin`：旋转和缩放的原点（最终变换移动到此）。可为三维向量`[x, y, z]`或内置值`"corner"`（`[0, 0, 0]`）、`"center"`（`[0.5, 0.5, 0.5]`）、`"opposing-corner"`（`[1, 1, 1]`，默认值）。

## 方块状态文件(`Blockstate Files`)

_另见：[Minecraft Wiki上的方块状态文件(`Blockstate files`)][mcwikiblockstate]_

方块状态文件用于将不同模型分配给不同[方块状态(`blockstates`)]。每个注册方块必须有且仅有一个方块状态文件。为方块状态指定方块模型有三种互斥方式：变体(`variants`)、多部分(`multipart`)或NeoForge新增的定义类型(`definition type`)。

在`variants`块中，每个方块状态对应一个元素（主流方式，绝大多数方块使用）：
- 键：不带方块名的方块状态字符串表示（如非含水上半台阶为`"type=top,waterlogged=false"`，无属性方块为`""`）。未使用属性可省略（如`waterlogged`属性不影响模型选择时，`type=top,waterlogged=false`和`type=top,waterlogged=true`可合并为`type=top`）。因此空字符串对所有方块有效。
- 值：单个模型对象或模型对象数组。若为数组，将随机选择模型。模型对象包含：
    - `type`：NeoForge新增。设置自定义方块状态模型加载器，详见[方块状态模型加载器(`Block State Model Loaders`)][bsmmodelloader]。
    - `model`：模型文件路径（相对于命名空间`models`文件夹），如`minecraft:block/cobblestone`。
    - `x`和`y`：模型绕x/y轴旋转（限90°倍数）。可选，默认为0。
    - `uvlock`：旋转时是否锁定模型UV。可选，默认为`false`。
    - `weight`：仅对模型对象数组有效。设置权重供随机选择使用，默认为1。

在`multipart`块中，元素根据方块状态属性组合（主要用于栅栏和墙，基于布尔属性启用四个方向部分）。多部分元素包含`when`块和`apply`块：
- `when`块：指定方块状态字符串或必须满足的属性列表。列表可命名为`"OR"`或`"AND"`执行逻辑运算。单一方块状态和列表值可通过`|`分隔指定多个实际值（如`facing=east|facing=west`）。
- `apply`块：指定使用的模型对象或数组（同`variants`块）。

最后，`neoforge:definition_type`可指定自定义加载器注册方块状态文件，详见[方块状态定义加载器(`Block State Definition Loaders`)][bsdmodelloader]。

## 客户端物品(`Client Items`)

[客户端物品(`client items`)][citems]用于将模型分配给`ItemStack`的状态。虽然模型JSON中有物品特定字段，但客户端物品基于上下文消费模型渲染，因此其大部分信息已移至独立的[章节(`section`)][citems]。

## 着色(`Tinting`)

草、树叶等方块会根据位置和/或属性改变纹理颜色。[模型元素(`model elements`)][elements]可在面上指定**着色索引**(`tint index`)，供颜色处理器处理对应面。代码端通过三个事件实现：方块颜色处理器、基于生物群系的方块着色（与方块颜色处理器结合使用）和物品着色源事件。它们工作方式相似，先看方块处理器示例：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerBlockColorHandlers(RegisterColorHandlersEvent.Block event) {
    // 参数：方块状态、所在世界、方块位置、着色索引
    // 世界和位置可为null
    event.register((state, level, pos, tintIndex) -> {
        // 替换为自定义计算。原版参考见BlockColors类
        // 颜色为ARGB格式。若着色索引为-1，表示不应着色，应使用默认值
        return 0xFFFFFFFF;
    },
    // 应用着色的方块（可变参数）
    EXAMPLE_BLOCK.get(), ...);
}
```

颜色解析器(`color resolver`)示例：
```java
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerColorResolvers(RegisterColorHandlersEvent.ColorResolvers event) {
    // 参数：当前生物群系、方块X坐标、方块Z坐标
    event.register((biome, x, z) -> {
        // 替换为自定义计算。原版参考见BiomeColors类
        // 颜色为ARGB格式
        return 0xFFFFFFFF;
    });
}
```

物品着色详见[客户端物品文章的相关章节(`relevant section`)][itemtints]。

## 注册独立模型(`Registering Standalone Models`)

未与方块或物品关联但仍需在其他上下文（如[方块实体渲染器(`block entity renderers`)][ber]）使用的模型，可通过`ModelEvent.RegisterStandalone`注册：

```java
// 类型可为任意能从ResolvedModel和ModelBaker获取的类型
// 泛型应为UnbakedStandaloneModel<T>的泛型类型
public static final StandaloneModelKey<QuadCollection> EXAMPLE_KEY = new StandaloneModelKey<>(
    new ModelDebugName() {
        @Override
        public String debugName() {
            // 独立模型名称
            // 可为任意字符串，但应包含模组ID
            return "examplemod: Example Model";
        }
    }
);

@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerAdditional(ModelEvent.RegisterStandalone event) {
    event.register(
        // 目标模型
        EXAMPLE_KEY,
        // 所需的UnbakedStandaloneModel<T>（本例返回QuadCollection）
        // 可用SimpleUnbakedStandaloneModel<T>的静态方法简化
        SimpleUnbakedStandaloneModel.quadCollection(
            // 模型ID（相对于`assets/<命名空间>/models/<路径>.json`）
            ResourceLocation.fromNamespaceAndPath("examplemod", "block/example_unused_model")
        )
    );
}
```

[ao]: https://en.wikipedia.org/wiki/Ambient_occlusion
[ber]: ../../../blockentities/ber.md
[bsfile]: #blockstate-files
[bsdmodelloader]: modelloaders.md#block-state-definition-loaders
[bsmmodelloader]: modelloaders.md#block-state-model-loaders
[custommodelloader]: modelloaders.md#model-loaders
[elements]: #elements
[event]: ../../../concepts/events.md
[extrafacedata]: #extra-face-data
[citems]: items.md
[itemmodel]: items.md#a-basic-model
[itemtints]: items.md#tinting
[mcwiki]: https://minecraft.wiki
[mcwikiblockstate]: https://minecraft.wiki/w/Tutorials/Models#Block_states
[mcwikimodel]: https://minecraft.wiki/w/Model
[mipmapping]: https://en.wikipedia.org/wiki/Mipmap
[modbus]: ../../../concepts/events.md#event-buses
[perspectives]: modelsystem.md#perspectives
[rendertype]: #render-types
[roottransforms]: #root-transforms
[rl]: ../../../misc/resourcelocation.md
[textures]: ../textures.md
[tinting]: #tinting