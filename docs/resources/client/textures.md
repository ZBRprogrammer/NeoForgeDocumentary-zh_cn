# **纹理**(`Textures`)

Minecraft 中的所有纹理均为 PNG 文件，位于命名空间的 `textures` 文件夹中。不支持 JPG、GIF 等其他图像格式。[资源定位符(`Resource Locations`)][rl]的路径通常相对于 `textures` 文件夹，例如：
- `examplemod:block/example_block` 对应 `assets/examplemod/textures/block/example_block.png`

纹理尺寸通常应为 2 的幂（如 16x16 或 32x32）。现代 Minecraft 原生支持大于 16x16 的方块/物品纹理。对于非 2 的幂尺寸的自渲染纹理（如 GUI 背景）：
1. 创建下一个 2 的幂尺寸的空白文件（通常 256x256）
2. 将纹理置于文件左上角
3. 在代码中设置实际绘制尺寸

## 纹理元数据

纹理元数据通过同名附加 `.mcmeta` 后缀的文件指定。例如：
- 纹理路径：`textures/block/example.png`
- 元数据文件：`textures/block/example.png.mcmeta`

格式如下（所有字段可选）：

```json5
{
    // 通用纹理元数据
    "texture": {
        // 是否在需要时模糊纹理（默认 false）
        "blur": true,
        // 是否在需要时钳制纹理（默认 false）
        "clamp": true
    },

    // GUI 精灵纹理元数据
    "gui": {
        // 缩放类型（三选一）
        "scaling": {
            "type": "stretch" // 默认（拉伸）
        },
        "scaling": {
            "type": "tile",   // 平铺
            "width": 16,      // 平铺宽度
            "height": 16      // 平铺高度
        },
        "scaling": {
            "type": "nine_slice", // 九宫格切片
            "width": 16,
            "height": 16,
            // 边框偏移（可统一设置）
            "border": {
                "left": 0,
                "top": 0,
                "right": 0,
                "bottom": 0
            },
            // true：中心区域拉伸填充（默认平铺）
            "stretch_inner": true
        }
    },

    // 动画纹理元数据（见下文）
    "animation": {}
}
```

## **动画纹理**(`Animated Textures`)

Minecraft 原生支持方块/物品的动画纹理。动画纹理由垂直堆叠的动画帧组成（如 8 帧的 16x16 动画 → 16x128 PNG 文件）。

需在纹理元数据中添加 `animation` 对象才能激活动画（否则显示为扭曲纹理）：

```json5
{
    "animation": {
        // 自定义帧播放顺序（省略则从上到下播放）
        "frames": [1, 0],
        // 单帧持续时间（单位：游戏刻，默认 1）
        "frametime": 5,
        // 是否在帧间插值（默认 false）
        "interpolate": true,
        // 单帧宽高（省略则使用纹理宽高）
        "width": 12,
        "height": 12
    }
}
```

[rl]: ../../misc/resourcelocation.md