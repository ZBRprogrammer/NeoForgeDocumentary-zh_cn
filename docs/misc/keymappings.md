# **键盘映射**(`Key Mappings`)

**键盘映射**(`Key mapping`) 或 **键位绑定**(`key binding`) 定义特定操作与输入（鼠标点击、按键等）的关联。客户端可在任何可接受输入时检查映射操作。此外，所有键盘映射均可通过[控制选项菜单](controls)重新分配。

## 注册 `KeyMapping`

`KeyMapping` 需在[模组事件总线](eventbus)上监听 `RegisterKeyMappingsEvent`（仅限物理客户端），并调用 `#register`：

```java
// 在仅限物理客户端的类中

// 延迟初始化键盘映射（注册前不存在）
public static final Lazy<KeyMapping> EXAMPLE_MAPPING = Lazy.of(() -> /*...*/);

@SubscribeEvent // 仅限物理客户端的模组事件总线
public static void registerBindings(RegisterKeyMappingsEvent event) {
    event.register(EXAMPLE_MAPPING.get());
}
```

## 创建 `KeyMapping`

通过构造函数创建 `KeyMapping`，需提供：
- [翻译键](tk)：映射名称
- 默认输入
- [翻译键](tk)：[控制选项菜单](controls)中的分类

:::tip
自定义分类需提供非原版翻译键（如 `key.categories.examplemod.examplecategory`）。
:::

### 默认输入

每个键盘映射有默认输入，通过 `InputConstants.Key` 提供，包含：
- `InputConstants.Type`：输入设备类型
- 整数：设备上的标识符

原版提供三种输入类型：
- `KEYSYM`：键盘（使用 `GLFW` 键符）
- `SCANCODE`：键盘（使用平台特定扫描码）
- `MOUSE`：鼠标

:::note
推荐对键盘使用 `KEYSYM`（`GLFW` 键符不依赖特定系统），详见 [GLFW 文档](keyinput)。
:::

整数取决于输入类型（`GLFW` 定义）：
- `KEYSYM`：前缀 `GLFW_KEY_*`
- `MOUSE`：前缀 `GLFW_MOUSE_*`

```java
new KeyMapping(
    "key.examplemod.example1", // 名称翻译键
    InputConstants.Type.KEYSYM, // 键盘映射
    GLFW.GLFW_KEY_P,           // 默认键位P
    "key.categories.misc"      // 杂项分类
)
```

:::note
若无需默认映射，输入设为 `InputConstants#UNKNOWN`。
:::

### **键位冲突上下文**(`IKeyConflictContext`)

不同场景使用不同映射（如 GUI 中与游戏中）。为避免冲突，可为映射分配 `IKeyConflictContext`。每个上下文含：
- `#isActive`：当前游戏状态下是否可用
- `#conflicts`：是否与其他上下文冲突

NeoForge 提供三种基础上下文：
- `UNIVERSAL`（默认）：所有场景可用
- `GUI`：仅打开界面时可用
- `IN_GAME`：仅关闭界面时可用

自定义上下文需实现 `IKeyConflictContext`。

```java
new KeyMapping(
    "key.examplemod.example2",
    KeyConflictContext.GUI,     // 仅界面中可用
    InputConstants.Type.MOUSE,  // 鼠标映射
    GLFW.GLFW_MOUSE_BUTTON_LEFT,// 默认左键
    "key.categories.examplemod.examplecategory" // 自定义分类
)
```

### **键位修饰符**(`KeyModifier`)

需区分组合键行为（如 `G` 与 `CTRL+G`）时，NeoForge 构造函数支持添加修饰符：
- `KeyModifier#CONTROL`（控制键）
- `KeyModifier#SHIFT`（Shift键）
- `KeyModifier#ALT`（Alt键）
- `KeyModifier#NONE`（默认无修饰符）

在[控制选项菜单](controls)中按住修饰键分配输入即可添加修饰符。

```java
new KeyMapping(
    "key.examplemod.example3",
    KeyConflictContext.UNIVERSAL,
    KeyModifier.SHIFT,          // 默认需按住Shift
    InputConstants.Type.KEYSYM, // 键盘映射
    GLFW.GLFW_KEY_G,            // 默认键位G
    "key.categories.misc"
)
```

## 检查 `KeyMapping`

根据场景检查键盘映射是否被点击，执行关联逻辑。

### 游戏中检查

监听[事件总线](eventbus)上的 `ClientTickEvent.Post`，循环检查 `KeyMapping#consumeClick`。该方法仅在输入未被处理时返回 `true`。

```java
@SubscribeEvent // 仅限物理客户端的游戏事件总线
public static void onClientTick(ClientTickEvent.Post event) {
    while (EXAMPLE_MAPPING.get().consumeClick()) {
        // 点击时执行的逻辑
    }
}
```

:::caution
勿用 `InputEvent` 替代 `ClientTickEvent.Post`（前者仅处理键盘/鼠标输入）。
:::

### 界面中检查

在 `GuiEventListener` 方法中通过 `IKeyMappingExtension#isActiveAndMatches` 检查，常用方法：
- `#keyPressed`（按键）
- `#mouseClicked`（鼠标点击）

**检查按键**：在 `#keyPressed` 中通过 `InputConstants#getKey` 创建输入（修饰符已自动处理）。

```java
// 在 Screen 子类中
@Override
public boolean keyPressed(int key, int scancode, int mods) {
    if (EXAMPLE_MAPPING.get().isActiveAndMatches(InputConstants.getKey(key, scancode))) {
        // 按键时执行的逻辑
        return true;
    }
    return super.keyPressed(key, scancode, mods);
} 
```

:::note
非自有界面中检查按键，可监听[游戏事件总线](eventbus)上的 `ScreenEvent.KeyPressed`。
:::

**检查鼠标点击**：在 `#mouseClicked` 中通过 `InputConstants.Type#getOrCreate` 创建鼠标输入。

```java
// 在 Screen 子类中
@Override
public boolean mouseClicked(double x, double y, int button) {
    if (EXAMPLE_MAPPING.get().isActiveAndMatches(InputConstants.TYPE.MOUSE.getOrCreate(button))) {
        // 点击时执行的逻辑
        return true;
    }
    return super.mouseClicked(x, y, button);
} 
```

:::note
非自有界面中检查鼠标点击，可监听[游戏事件总线](eventbus)上的 `ScreenEvent.MouseButtonPressed`。
:::

[eventbus]: ../concepts/events.md#registering-an-event-handler
[controls]: https://minecraft.wiki/w/Options#Controls
[tk]: ../resources/client/i18n.md#components
[keyinput]: https://www.glfw.org/docs/3.3/input_guide.html#input_key