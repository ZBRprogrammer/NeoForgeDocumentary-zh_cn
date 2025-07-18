# **屏幕**(`Screens`)

**屏幕**(`Screens`)通常是Minecraft中所有**图形用户界面**(`GUIs`)的基础：接收用户输入，在服务器端验证，并将结果操作同步回客户端。它们可以与[菜单(`menus`)][menus]结合创建类似物品栏视图的通信网络，也可以作为独立组件由模组开发者通过自己的[网络(`network`)][network]实现处理。

屏幕由多个部分组成，因此很难完全理解Minecraft中"屏幕"的实际含义。因此，本文将在讨论屏幕本身之前，先详细介绍屏幕的各个组件及其应用方式。

## 渲染GUI

GUI渲染分为两个阶段：提交阶段(`submission phase`)和渲染阶段(`render phase`)。

提交阶段负责收集所有要渲染到屏幕的元素（如按钮、文本、物品）。每次提交都会存储在`GuiRenderState`中，以便在渲染阶段进行处理和渲染。原版提供了四种元素类型：`GuiElementRenderState`、`GuiItemRenderState`、`GuiTextRenderState`和`PictureInPictureRenderState`。下文讨论的所有内容都发生在提交阶段，通过内部创建上述渲染状态之一实现。

渲染阶段，顾名思义，负责将元素渲染到屏幕。首先将`PictureInPictureRenderState`、`GuiItemRenderState`和`GuiTextRenderState`准备并处理为`GuiElementRenderState`。接着对元素进行排序，最终绘制到屏幕上。最后重置`GuiRenderState`，准备用于下一个GUI或渲染周期。

### 相对坐标(`Relative Coordinates`)

向渲染状态提交任何内容时，都需要指定元素渲染位置的坐标。经过多层抽象后，Minecraft的大多数渲染调用都接受X和Y坐标：X值从左向右递增，Y值从上向下递增。但这些坐标不固定于特定范围——其范围会根据屏幕尺寸和游戏选项中设置的**GUI缩放比例**(`GUI Scale`)而变化。因此必须特别注意传递给渲染调用的坐标值是否针对可变屏幕尺寸进行了正确缩放——即是否正确相对化(`relativized`)。

关于如何相对化坐标的信息详见[屏幕(`screen`)][screen]章节。

:::caution
如果使用固定坐标或不正确缩放屏幕，渲染对象可能显示异常或错位。检查坐标相对化是否正确的简单方法是点击视频设置中的"GUI缩放"按钮。该值在决定GUI渲染比例时，会作为显示器宽高的除数使用。
:::

### **GuiGraphics**

提交到`GuiRenderState`的元素通常通过`GuiGraphics`处理。`GuiRenderState`是提交阶段几乎所有方法的第一个参数，包含提交常用渲染对象的方法。

`GuiGraphics`将当前位姿(`pose`)作为`Matrix3x2fStack`公开以应用XY变换：

```java
// 对于某个GuiGraphics实例graphics

// 将新矩阵压入栈
graphics.pose().pushMatrix();

// 应用元素渲染所需的变换

// 接受XY偏移量
graphics.pose().translate(10, 10);
// 接受弧度制旋转角度
graphics.pose().rotate((float) Math.PI);
// 接受XY缩放因子
graphics.pose().scale(2f, 2f);

// 向`GuiRenderState`提交元素
graphics.blitSprite(...);

// 弹出矩阵以重置变换
graphics.pose().popMatrix();
```

此外，可使用`enableScissor`和`disableScissor`将元素裁剪到特定区域：

```java
// 对于某个GuiGraphics实例graphics

// 启用裁剪并设置渲染边界
graphics.enableScissor(
    // 左侧X坐标
    0,
    // 顶部Y坐标
    0,
    // 右侧X坐标
    10,
    // 底部Y坐标
    10
);

// 向`GuiRenderState`提交元素
graphics.blitSprite(...);

// 禁用裁剪以重置渲染区域
graphics.disableScissor();
```

### 节点树与层级(`Node Trees and Strata`)

向`GuiRenderState`提交元素时，并非简单添加到列表。若如此，某些元素可能因提交顺序被完全覆盖。为解决此问题，元素会先按层级(`stratum`)分类到节点树中。元素排序基于其定义的`ScreenArea#bounds`；未定义边界的元素不会被提交渲染。

`GuiRenderState`由双向链表`GuiRenderState.Node`组成，使用"上"(`up`)和"下"(`down`)替代"首"(`first`)和"尾"(`last`)。节点从"下"到"上"渲染，每个节点持有包含渲染状态的自有图层数据。元素首次提交到`GuiRenderState`时，会根据其`ScreenArea#bounds`决定使用或创建哪个节点。所选（或新建）节点位于具有相交元素的最高节点之上。

:::warning
尽管`ScreenArea#bounds`标记为可空(`nullable`)，但未定义边界的元素不会被添加到渲染状态。该方法仅因渲染阶段提交的元素会添加到当前节点（而非基于边界计算节点）而可空。
:::

每个节点列表称为渲染状态中的层级(`stratum`)。通过调用`GuiGraphics#nextStratum`可创建新节点列表（即新层级），新层级会渲染在所有先前层级元素之上（如物品提示）。调用`nextStratum`后无法返回先前层级。

### `GuiElementRenderState`

`GuiElementRenderState`保存**GUI元素**(`GUI element`)如何渲染到屏幕的元数据。该元素渲染状态扩展`ScreenArea`以定义屏幕上的`bounds`，边界应始终包含整个渲染元素以确保在节点列表中正确排序。边界计算通常考虑以下部分参数（包括位置和位姿）。

`scissorArea`裁剪元素可渲染区域。若`scissorArea`为`null`，则整个元素渲染到屏幕；若`scissorArea`矩形不与`bounds`相交，则不渲染任何内容。

其余三个方法处理元素的实际渲染：`pipeline`定义元素使用的着色器和元数据；`textureSetup`可为片段着色器指定`Sampler0`、`Sampler1`、`Sampler2`或其组合；`buildVertices`将顶点传递到缓冲区，接受目标`VertexConsumer`和使用的Z坐标。

若`GuiGraphics`提供的方法不足，NeoForge添加了`GuiGraphics#submitGuiElementRenderState`提交自定义元素渲染状态：

```java
// 对于某个GuiGraphics实例graphics
graphics.submitGuiElementRenderState(new GuiElementRenderState() {

    // 存储栈的当前位姿
    private final Matrix3x2f pose = new Matrix3x2f(graphics.pose());
    // 存储当前裁剪区域
    @Nullable
    private final ScreenRectangle scissorArea = graphics.peekScissorStack();

    @Override
    public ScreenRectangle bounds() {
        // 假设边界为0, 0, 10, 10
        
        // 计算初始矩形
        ScreenRectangle rectangle = new ScreenRectangle(
            // XY位置
            0, 0,
            // 元素宽高
            10, 10
        );

        // 使用位姿将矩形变换到正确位置
        rectangle = rectangle.transformMaxBounds(this.pose);

        // 若定义了裁剪区域，返回两矩形交集；否则返回完整边界
        return this.scissorArea != null
            ? this.scissorArea.intersection(rectangle)
            : rectangle;
    }

    @Override
    @Nullable
    public ScreenRectangle scissorArea() {
        return this.scissorArea;
    }

    @Override
    public RenderPipeline pipeline() {
        return RenderPipelines.GUI;
    }

    @Override
    public TextureSetup textureSetup() {
        // 返回片段着色器中采样器使用的纹理
        // 片段着色器中的使用情况：
        // - Sampler0通常包含元素纹理
        // - Sampler1通常提供第二元素纹理（当前仅末地传送门管线使用）
        // - Sampler2通常包含游戏光照纹理

        // 通常至少应在Sampler0指定一个纹理
        return TextureSetup.noTexture();
    }

    @Override
    public void buildVertices(VertexConsumer consumer, float z) {
        // 使用管线指定的顶点格式构建顶点
        // GUI使用带位置和颜色的四边形
        // 颜色必须为ARGB格式
        consumer.addVertexWith2DPose(this.pose, 0,  0,  z).setUv(0, 0).setColor(0xFFFFFFFF);
        consumer.addVertexWith2DPose(this.pose, 0,  10, z).setUv(0, 1).setColor(0xFFFFFFFF);
        consumer.addVertexWith2DPose(this.pose, 10, 10, z).setUv(1, 1).setColor(0xFFFFFFFF);
        consumer.addVertexWith2DPose(this.pose, 10, 0,  z).setUv(1, 0).setColor(0xFFFFFFFF);
    }
});
```

### 元素排序(`Element Ordering`)

目前所示元素仅操作XY坐标。而Z坐标基于元素渲染顺序自动计算：从Z=0开始，每个元素在前一元素前方0.01处渲染。3D元素先渲染到2D纹理再显示到屏幕，因此几乎不会发生**Z冲突**(`Z-fighting`)。

渲染阶段中，各层级按顺序渲染，节点列表从"下"到"上"渲染。但同一节点内如何排序？这由`GuiRenderer#ELEMENT_SORT_COMPARATOR`处理，根据`GuiElementRenderState#scissorArea`、`pipeline`和`textureSetup`排序。

:::warning
文本渲染的字形(`Glyphs`)不参与排序，始终在当前节点所有元素之后渲染。
:::

未指定`scissorArea`的元素始终最先渲染，其次按顶部Y、底部Y、左侧X、右侧X排序。若两元素`scissorArea`相同，则使用`pipeline`的排序键（通过`RenderPipeline#getSortKey`）。排序键基于`RenderPipeline`构建顺序（原版中即`RenderPipelines`内静态常量的类加载顺序）。若排序键相同，则使用`textureSetup`：未指定`textureSetup`的元素优先，其次按纹理元素的排序键（通过`TextureSetup#getSortKey`）排序。

:::warning
技术层面，因`RenderPipeline`和`TextureSetup`的存在，元素排序不具备确定性。因为排序目标并非确定性，而是尽可能减少管线(`pipeline`)和纹理切换(`texture switches`)。
:::

## `GuiGraphics`中的方法

`GuiGraphics`包含提交常用渲染对象的方法，分为六类：彩色矩形、字符串、纹理、物品、提示和画中画(`Picture-in-Picture`)。这些方法提交的元素均继承自`pose`的当前位姿，以及基于`enableScissor`/`disableScissor`从`peekScissorStack`获取的裁剪区域。传给方法的颜色必须为[ARGB][argb]格式。

### 彩色矩形(`Colored Rectangles`)

彩色矩形使用`ColoredRectangleRenderState`提交。所有填充方法可接受可选的`RenderPipeline`和`TextureSetup`来指定矩形渲染方式。可提交三类彩色矩形：

首先是一像素宽的水平和垂直线条（分别为`hLine`和`vLine`）。`hLine`接受定义左右（含）的两个X坐标、顶部Y坐标和颜色；`vLine`接受左侧X坐标、定义上下（含）的两个Y坐标和颜色。

其次是`fill`方法，提交要绘制的矩形。线条方法内部调用此方法，参数为左侧X、顶部Y、右侧X、底部Y坐标和颜色。

第三是`renderOutline`方法，提交四条一像素宽矩形作为轮廓。参数为左侧X、顶部Y坐标，轮廓宽高和颜色。

最后是`fillGradient`方法，绘制带垂直渐变的矩形。参数为左侧X、顶部Y、右侧X、底部Y坐标，以及底部和顶部颜色。

### 字符串(`Strings`)

字符串、[`组件`(`Component`s)][component]和`FormattedCharSequence`s使用`GuiTextRenderState`提交。每个字符串通过指定`Font`绘制，使用`GlyphRenderTypes#guiPipeline`创建`BakedGlyph.GlyphInstance`和可选的`BakedGlyph.Effect`。渲染阶段中，文本渲染状态会按字符串字符转换为`GlyphRenderState`和可能的`GlyphEffectRenderState`。

字符串有两种对齐渲染方式：左对齐(`drawString`)和居中对齐(`drawCenteredString`)。两者均接受渲染字符串的字体、要绘制的字符串、分别代表左/中位置的X坐标、顶部Y坐标和颜色。左对齐字符串还可接受是否绘制文本阴影(`drop shadow`)。

若文本需在边界内换行，可使用`drawWordWrap`；若需矩形背景，则用`drawStringWithBackdrop`。两者默认提交左对齐字符串。

:::note
字符串通常应作为[`组件`(`Component`s)][component]传递，因其处理多种用例（包括方法的另外两个重载）。
:::

### 纹理(`Textures`)

纹理通过`BlitRenderState`提交（方法名`blit`由此而来）。`BlitRenderState`复制图像位并通过`RenderPipeline`参数渲染到屏幕。每个`blit`还接受代表纹理绝对位置的`ResourceLocation`：

```java
// 指向'assets/examplemod/textures/gui/container/example_container.png'
private static final ResourceLocation TEXTURE = ResourceLocation.fromNamespaceAndPath("examplemod", "textures/gui/container/example_container.png");
```

尽管`blit`有多种重载，此处仅讨论两种。

第一种`blit`接受两个整数、两个浮点数和四个整数（假设图像在PNG文件中）。参数为屏幕左侧X和顶部Y坐标，PNG内的左侧X和顶部Y坐标，要渲染图像的宽高，以及PNG文件的宽高。

:::tip
必须指定PNG文件尺寸，以便标准化坐标获取关联的UV值。
:::

第二种`blit`在末尾添加一个整数，表示图像的着色(`tint`)颜色。未指定时默认为`0xFFFFFFFF`。

#### `blitSprite`

`blitSprite`是`blit`的特殊实现，从GUI纹理图集(`texture atlas`)获取纹理。大多数覆盖背景的纹理（如熔炉GUI中的"燃烧进度"覆盖层）都是精灵(`sprites`)。所有精灵纹理相对于`textures/gui/sprites`，无需指定文件扩展名。

```java
// 指向'assets/examplemod/textures/gui/sprites/container/example_container/example_sprite.png'
private static final ResourceLocation SPRITE = ResourceLocation.fromNamespaceAndPath("examplemod", "container/example_container/example_sprite");
```

一组`blitSprite`方法与`blit`参数相同（除处理PNG坐标/宽高的四个整数）。

另一组`blitSprite`方法接受更多纹理信息以绘制精灵部分。参数为精灵宽高、精灵内X/Y坐标、屏幕左侧X/顶部Y坐标、着色颜色，以及要渲染图像的宽高。

若精灵尺寸与纹理尺寸不匹配，可通过三种方式缩放：拉伸(`stretch`)、平铺(`tile`)和九宫格(`nine_slice`)。`stretch`将图像从纹理尺寸拉伸到屏幕尺寸；`tile`重复渲染纹理直至填满屏幕尺寸；`nine_slice`将纹理分为一个中心、四条边和四个角，平铺到所需屏幕尺寸。

通过在与纹理文件同名的mcmeta文件中添加`gui.scaling` JSON对象设置：

```json5
// 对于某纹理文件example_sprite.png
// 位于example_sprite.png.mcmeta

// 拉伸示例
{
    "gui": {
        "scaling": {
            "type": "stretch"
        }
    }
}

// 平铺示例
{
    "gui": {
        "scaling": {
            "type": "tile",
            // 开始平铺的尺寸（通常为纹理尺寸）
            "width": 40,
            "height": 40
        }
    }
}

// 九宫格示例
{
    "gui": {
        "scaling": {
            "type": "nine_slice",
            // 开始平铺的尺寸（通常为纹理尺寸）
            "width": 40,
            "height": 40,
            "border": {
                // 将被切片为边框纹理的填充
                "left": 1,
                "right": 1,
                "top": 1,
                "bottom": 1
            },
            // true时纹理中心部分像拉伸类型而非九宫格平铺
            "stretch_inner": true
        }
    }
}
```

### 物品(`Items`)

物品使用`GuiItemRenderState`提交。渲染阶段中，物品渲染状态根据物品边界和客户端物品属性(`client item properties`)转换为`BlitRenderState`或`OversizedItemRenderState`。

`renderItem`接受`ItemStack`及屏幕左侧X/顶部Y坐标。可选参数包括持有的`LivingEntity`、物品栈所在的当前`Level`和种子值(`seeded value`)。另有`renderFakeItem`将`LivingEntity`设为`null`。

物品装饰（如耐久条、冷却时间和数量）通过`renderItemDecorations`处理。参数与基础`renderItem`相同，另加`Font`和数量文本覆盖值。

### 提示(`Tooltips`)

提示通过上述多种渲染状态提交。提示方法分为两类："下一帧"(`next frame`)和"立即"(`immediate`)。两者均接受渲染文本的`Font`、`Component`列表、特殊渲染的可选`TooltipComponent`、左侧X/顶部Y坐标、调整位置的`ClientTooltipPositioner`，以及背景和框架纹理。

下一帧提示并非在下一帧提交，而是推迟到`Screen#render`调用后才提交。提示添加到新层级，意味着将渲染在屏幕所有元素之上。下一帧方法形式为`set*Tooltip*ForNextFrame`，还可接受布尔值（指示是否覆盖当前推迟的提示）和提示应使用的`ItemStack`。

立即提示则在方法调用时立即提交。立即方法形式为`renderTooltip`，还接受提示悬停的`ItemStack`。

### 画中画(`Picture-in-Picture`)

画中画(`PiP`)允许在屏幕绘制任意对象。PiP不直接绘制到输出，而是将对象绘制到中间纹理（即"画面"(`picture`)），渲染阶段作为`BlitRenderState`提交到`GuiRenderState`。`GuiGraphics`通过`submit*RenderState`提供地图、实体、玩家皮肤、书本模型、旗帜图案、告示牌和性能分析图的方法。

:::note
当`ClientItem.Properties#oversizedInGui`为true时，超过默认16x16边界的物品使用`OversizedItemRenderer` PiP作为渲染机制。
:::

每个PiP提交`PictureInPictureRenderState`将对象渲染到屏幕。类似`GuiElementRenderState`，`PictureInPictureRenderState`也扩展`ScreenArea`以定义`bounds`和通过`scissorArea`裁剪。`PictureInPictureRenderState`定义画面的渲染位置和尺寸，指定左侧X(`x0`)、右侧X(`x1`)、顶部Y(`y0`)、底部Y(`y1`)。画面内元素还可按浮点值`scale`缩放。最后，额外`pose`可用于变换画面XY坐标（默认为单位位姿，因渲染对象通常已在画面内变换）。为简化实现，可用`PictureInPictureRenderState#getBounds`计算`bounds`；但若修改`pose`，需自定义逻辑。

```java
// 可添加其他参数，但这是实现所有方法的最小要求
public record ExampleRenderState(
    int x0, // 左侧X
    int x1, // 右侧X
    int y0, // 顶部Y
    int y1, // 底部Y
    float scale, // 绘制到画面的缩放因子
    @Nullable ScreenRectangle scissorArea, // 渲染区域
    @Nullable ScreenRectangle bounds // 元素边界
) implements PictureInPictureRenderState {

    // 额外构造器
    public ExampleRenderState(int x, int y, int width, int height, @Nullable ScreenRectangle scissorArea) {
        this(
            x, // x0
            x + width, // x1
            y, // y0
            y + height, // y1
            1f, // scale
            scissorArea,
            PictureInPictureRenderState.getBounds(x, y, x + width, y + height, scissorArea)
        );
    }
}
```

要将PiP渲染状态绘制并提交到画面，每个PiP有对应的`PictureInPictureRenderer<T>`（其中`T`为实现的`PictureInPictureRenderState`）。可重写多个方法（赋予用户近乎完整的管线控制权），但必须实现三个方法：

首先是`getRenderStateClass`，返回`PictureInPictureRenderState`的类。原版中此方法用于注册渲染器对应的渲染状态。NeoForge仍使用渲染状态类，但通过事件注册映射到动态渲染器池（而非调用`getRenderStateClass`）。

其次是`getTextureLabel`，为写入的画面提供唯一调试标签。最后是`renderToTexture`，实际将对象绘制到画面（类似其他渲染方法）。

```java
public class ExampleRenderer extends PictureInPictureRenderer<ExampleRenderState> {

    // 接受用于将对象写入画面的缓冲区
    public ExampleRenderer(MultiBufferSource.BufferSource bufferSource) {
        super(bufferSource);
    }

    @Override
    public Class<ExampleRenderState> getRenderStateClass() {
        // 返回渲染状态类
        return ExampleRenderState.class;
    }

    @Override
    protected String getTextureLabel() {
        // 可为任意字符串，但应唯一
        // 建议添加模组ID前缀
        return "examplemod: example pip";
    }

    @Override
    protected void renderToTexture(ExampleRenderState renderState, PoseStack pose) {
        // 按需修改位姿
        // 可按需压栈/弹栈，但写入画面会创建新`PoseStack`
        pose.translate(...);

        // 将对象渲染到屏幕
        VertexConsumer consumer = this.bufferSource.getBuffer(RenderType.lines());
        consumer.addVertex(...).setColor(...).setNormal(...);
        consumer.addVertex(...).setColor(...).setNormal(...);
    }

    // 额外方法

    @Override
    protected void blitTexture(ExampleRenderState renderState, GuiRenderState guiState) {
        // 默认将画面作为`BlitRenderState`提交到GUI渲染状态
        // 若需修改`BlitRenderState`可重写此方法
        // 应调用`GuiRenderState#submitBlitToCurrentLayer`
        // bounds可为`null`
        super.blitTexture(renderState, guiState);
    }

    @Override
    protected boolean textureIsReadyToBlit(ExampleRenderState renderState) {
        // 为true时重用已写入画面，而非构造新画面并用`renderToTexture`写入
        // 仅当保证两元素渲染*完全*相同时应为true
        return super.textureIsReadyToBlit(renderState);
    }

    @Override
    protected float getTranslateY(int scaledHeight, int guiScale) {
        // 设置位姿在Y方向的初始偏移
        // 常见实现用`scaledHeight / 2f`使Y坐标像X一样居中
        return scaledHeight;
    }

    @Override
    public boolean canBeReusedFor(ExampleRenderState state, int textureWidth, int textureHeight) {
        // NeoForge新增方法，检查此渲染器是否可在后续帧重用
        // true时重用前一帧构造的状态和渲染器
        // false时创建新渲染器
        return super.canBeReusedFor(state, textureWidth, textureHeight);
    }
}
```

要使用PiP，渲染器必须在[模组事件总线(`modbus`)][modbus]的`RegisterPictureInPictureRenderersEvent`中注册：

```java
@SubscribeEvent // 在模组事件总线
public static void registerPip(RegisterPictureInPictureRenderersEvent event) {
    event.register(
        // PiP渲染状态类
        ExampleRenderState.class,
        // 接收`MultiBufferSource.BufferSource`并返回PiP渲染器的工厂
        ExampleRenderer::new
    );
}
```

然后可通过NeoForge新增的`GuiGraphics#submitPictureInPictureRenderState`提交PiP渲染状态：

```java
// 对于某个GuiGraphics实例graphics
graphics.submitPictureInPictureRenderState(new ExampleRenderState(
    0, 0,
    10, 10,
    // 从栈获取裁剪区域
    graphics.peekScissorStack()
));
```

:::note
NeoForge修复了阻止单帧提交多个PiP渲染状态实例的漏洞。
:::

## **可渲染对象**(`Renderable`)

`Renderable`本质上是可渲染的对象，包括屏幕、按钮、聊天框、列表等。`Renderable`只有一个方法：`#render`。参数包括用于渲染的`GuiGraphics`、鼠标相对屏幕尺寸缩放后的XY位置，以及刻增量(`tick delta`)（自上一帧经过的刻数）。

常见可渲染对象包括屏幕和"部件"(`widgets`)：屏幕上通常渲染的可交互元素，如`Button`及其子类`ImageButton`，以及用于屏幕输入文本的`EditBox`。

## **GUI事件监听器**(`GuiEventListener`)

Minecraft中所有渲染的屏幕都实现`GuiEventListener`。`GuiEventListener`负责处理用户与屏幕的交互，包括鼠标（移动、点击、释放、拖拽、滚动、悬停）和键盘（按下、释放、输入）事件。每个方法返回相关操作是否成功影响屏幕。按钮、聊天框、列表等部件也实现此接口。

### **容器事件处理器**(`ContainerEventHandler`)

几乎等同于`GuiEventListener`的是其子类型：`ContainerEventHandler`。它们负责处理包含部件的屏幕上的用户交互，管理当前焦点(`focus`)及应用相关交互。`ContainerEventHandler`添加三个额外功能：可交互子项、拖拽和聚焦。

事件处理器持有用于确定元素交互顺序的子项。在鼠标事件处理期间（拖拽除外），鼠标悬停到的列表第一个子项执行其逻辑。

通过`#mouseClicked`和`#mouseReleased`实现的元素拖拽提供更精确的执行逻辑。

聚焦允许在事件执行期间（如键盘事件或鼠标拖拽）优先检查和处理特定子项。焦点通常通过`#setFocused`设置。此外，可用`#nextFocusPath`循环可交互子项，根据传入的`FocusNavigationEvent`选择子项。

:::note
屏幕通过`AbstractContainerEventHandler`实现`ContainerEventHandler`，添加了拖拽和聚焦子项的设置/获取逻辑。
:::

## **可叙述条目**(`NarratableEntry`)

`NarratableEntry`是通过Minecraft无障碍叙述(`accessibility narration`)功能可描述的元素。根据悬停或选中状态，每个元素可提供不同叙述，优先级通常为焦点、悬停及其他情况。

`NarratableEntry`有四个方法：两个决定元素被朗读时的优先级（`#narrationPriority`和`#getTabOrderGroup`）；一个决定是否朗读叙述（`#isActive`）；最后一个提供叙述到关联输出（朗读或阅读）（`#updateNarration`）。 

:::note
Minecraft的所有部件都是`NarratableEntry`，因此若使用可用子类型通常无需手动实现。
:::

## 屏幕子类型(`The Screen Subtype`)

基于以上知识，可构建基础屏幕。为便于理解，将按通常出现顺序介绍屏幕组件。

首先，所有屏幕接受代表屏幕标题的`Component`。该组件通常由其子类绘制到屏幕，在基础屏幕中仅用于叙述消息。

```java
// 在某个Screen子类中
public MyScreen(Component title) {
    super(title);
}
```

### 初始化(`Initialization`)

屏幕初始化后调用`#init`方法。`init`从`Minecraft`实例设置屏幕内初始配置到游戏缩放后的相对宽高。任何设置（如添加部件或预计算相对坐标）都应在此方法完成。若游戏窗口调整大小，将通过调用`init`方法重新初始化屏幕。

添加部件到屏幕有三种方式，各有用途：

| 方法                | 描述                                                                 |
|:-------------------:|:---------------------------------------------------------------------|
|`addWidget`          | 添加可交互且叙述但**不渲染**的部件                                   |
|`addRenderableOnly`  | 添加**仅渲染**的部件（不可交互且不叙述）                             |
|`addRenderableWidget`| 添加**可交互、叙述且渲染**的部件                                     |

通常最常用`addRenderableWidget`。

```java
// 在某个Screen子类中
@Override
protected void init() {
    super.init();

    // 添加部件和预计算值
    this.addRenderableWidget(new EditBox(/* ... */));
}
```

### 屏幕刻处理(`Ticking Screens`)

屏幕也使用`#tick`方法进行刻处理(`tick`)，为渲染目的执行客户端逻辑。

```java
// 在某个Screen子类中
@Override
public void tick() {
    super.tick();

    // 每帧执行某些逻辑
}
```

### 输入处理(`Input Handling`)

因屏幕是`GuiEventListener`子类型，可重写输入处理器（如处理特定[按键(`keymapping`)][keymapping]的逻辑）。

### 渲染屏幕(`Rendering the Screen`)

屏幕通过`#renderWithTooltip`在三个层级提交元素：背景层(`background stratum`)、渲染层(`render stratum`)和可选的提示层(`tooltip stratum`)。

背景层元素首先通过`#renderBackground`提交，通常包含模糊化或背景纹理。

:::warning
通过`GuiGraphics#blurBeforeThisStratum`处理的模糊化在单帧只能调用一次。尝试第二次渲染模糊将引发异常。
:::

渲染层元素接着通过`#render`方法提交（由`Renderable`子类型提供），主要提交部件和标签，并设置要在下一层级渲染的提示。

最后，提示层提交设置的提示。提示在渲染层通过`GuiGraphics#setTooltipForNextFrame`或`GuiGraphics#setComponentTooltipFromElementsForNextFrame`提交（可接受被提交的文本/提示组件及其在屏幕上的相对XY坐标）。

```java
// 在某个Screen子类中

// mouseX和mouseY表示光标在屏幕上的缩放坐标
@Override
public void renderBackground(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    // 在背景层提交内容
    this.renderTransparentBackground(graphics);
}

@Override
public void render(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    // 在部件前提交内容

    // 若是Screen的直接子类则提交部件
    super.render(graphics, mouseX, mouseY, partialTick);

    // 在部件后提交内容

    // 设置提示（将添加到此方法所有内容之上）
    graphics.setTooltipForNextFrame(...);
}
```

### 关闭屏幕(`Closing the Screen`)

屏幕关闭时，两个方法处理拆卸：`#onClose`和`#removed`。

`onClose`在用户输入关闭当前屏幕时调用，通常作为回调销毁/保存屏幕内进程（包括向服务器发包）。

`removed`在屏幕切换前调用（即将释放到垃圾收集器），处理屏幕打开前未重置回初始状态的任何内容。

```java
// 在某个Screen子类中

@Override
public void onClose() {
    // 在此停止所有处理器

    // 最后调用（防止干扰重写）
    super.onClose();
}

@Override
public void removed() {
    // 在此重置初始状态

    // 最后调用（防止干扰重写）
    super.removed()
;}
```

## `AbstractContainerScreen`

若屏幕直接关联[菜单(`menus`)][menus]，应子类化`AbstractContainerScreen`。`AbstractContainerScreen`作为菜单的屏幕和输入处理器，包含与槽位同步和交互的逻辑。因此通常只需重写或实现两个方法即可拥有可工作的容器屏幕。为便于理解，将按通常出现顺序介绍容器屏幕组件。

`AbstractContainerScreen`通常需要三个参数：打开的容器菜单（泛型`T`表示）、玩家物品栏（仅用于显示名称）和屏幕标题本身。在此可设置若干定位字段：

字段              | 描述
:---:             | :---
`imageWidth`      | 背景纹理宽度（通常在256x256 PNG内，默认为176）
`imageHeight`     | 背景纹理高度（通常在256x256 PNG内，默认为166）
`titleLabelX`     | 屏幕标题渲染的相对X坐标
`titleLabelY`     | 屏幕标题渲染的相对Y坐标
`inventoryLabelX` | 玩家物品栏名称渲染的相对X坐标
`inventoryLabelY` | 玩家物品栏名称渲染的相对Y坐标

:::caution
前文提到预计算相对坐标应在`#init`方法设置。此规则仍适用，因上述值非预计算坐标而是静态值和相对化坐标。

图像值静态不变（代表背景纹理尺寸）。为便于渲染，`init`方法预计算两个额外值（`leftPos`和`topPos`），标记背景渲染的左上角。标签坐标相对于这些值。

`leftPos`和`topPos`也便于渲染背景（因已代表传入`GuiGraphics#blit`的位置）。
:::

```java
// 在某个AbstractContainerScreen子类中
public MyContainerScreen(MyMenu menu, Inventory playerInventory, Component title) {
    super(menu, playerInventory, title);

    this.titleLabelX = 10;
    this.inventoryLabelX = 10;

    // 若修改'imageHeight'，必须同时修改'inventoryLabelY'（因该值依赖'imageHeight'）
}
```

### 菜单访问(`Menu Access`)

因菜单传入屏幕，可通过`menu`字段访问菜单内已同步的值（通过槽位、数据槽或自定义系统）。

### 容器刻处理(`Container Tick`)

当玩家存活并查看屏幕时，容器屏幕在`#tick`方法中通过`#containerTick`进行刻处理。这本质上替代了容器屏幕中的`tick`，其最常见用途是处理配方书刻处理。

```java
// 在某个AbstractContainerScreen子类中
@Override
protected void containerTick() {
    super.containerTick();

    // 在此处理刻逻辑
}
```

### 渲染容器屏幕(`Rendering the Container Screen`)

容器屏幕使用全部三个层级提交元素。首先，背景层通过`#renderBg`提交背景纹理；接着，渲染层通过`#render`提交部件，再通过`#renderLabels`提交文本；最后，`AbstractContainerScreen`还提供辅助方法`renderTooltip`向提示层提交提示。

从`render`开始，最常见重写（通常也是唯一情况）调用父类提交容器屏幕元素，再调用`renderTooltip`。

```java
// 在某个AbstractContainerScreen子类中
@Override
public void render(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    // 提交要渲染的部件和标签
    super.render(graphics, mouseX, mouseY, partialTick);

    // 此方法由容器屏幕添加，用于在提示层提交悬停槽位的提示
    this.renderTooltip(graphics, mouseX, mouseY);
}
```

`renderBg`调用以向背景层提交屏幕背景元素。

```java
// 在某个AbstractContainerScreen子类中

// 背景纹理位置（assets/<命名空间>/<路径>）
private static final ResourceLocation BACKGROUND_LOCATION = ResourceLocation.fromNamespaceAndPath(MOD_ID, "textures/gui/container/my_container_screen.png");

@Override
protected void renderBg(GuiGraphics graphics, float partialTick, int mouseX, int mouseY) {
    // 提交背景纹理。'leftPos'和'topPos'应已代表纹理渲染的左上角（因根据'imageWidth'和'imageHeight'预计算）
    // 两个0代表PNG文件内的整数u/v坐标，尺寸由最后两个整数表示（通常为256x256）
    graphics.blit(
        RenderPipelines.GUI_TEXTURED,
        BACKGROUND_LOCATION,
        this.leftPos, this.topPos,
        0, 0,
        this.imageWidth, this.imageHeight,
        256, 256
    );
}
```

`renderLabels`调用以在渲染层部件后提交文本。此方法使用屏幕字体调用`drawString`提交关联组件。

```java
// 在某个AbstractContainerScreen子类中
@Override
protected void renderLabels(GuiGraphics graphics, int mouseX, int mouseY) {
    super.renderLabels(graphics, mouseX, mouseY);

    // 假设有某Component 'label'
    // 'label'在'labelX'和'labelY'绘制
    // 颜色为ARGB值
    // 最后布尔值true时渲染阴影
    graphics.drawString(this.font, this.label, this.labelX, this.labelY, 0xFF404040, false);
}
```

:::note
提交标签时**不**需指定`leftPos`和`topPos`偏移。这些偏移已在`Matrix3x2fStack`中平移，因此此方法内所有提交均相对于这些坐标。
:::

## 注册AbstractContainerScreen

要使`AbstractContainerScreen`与菜单关联使用，需注册。可在[**模组事件总线**(`modbus`)][modbus]的`RegisterMenuScreensEvent`中调用`register`实现。

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线
public static void registerScreens(RegisterMenuScreensEvent event) {
    event.register(MY_MENU.get(), MyContainerScreen::new);
}
```

[menus]: menus.md
[network]: ../networking/index.md
[screen]: #the-screen-subtype
[argb]: https://en.wikipedia.org/wiki/RGBA_color_model#ARGB32
[component]: ../resources/client/i18n.md#components
[keymapping]: ../misc/keymappings.md#inside-a-gui
[modbus]: ../concepts/events.md#event-buses