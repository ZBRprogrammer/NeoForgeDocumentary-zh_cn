---
sidebar_position: 5
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# **护甲**(`Armor`)

**护甲**(`Armor`)是主要用途为通过多种抗性和效果保护[`LivingEntity`](`livingentity`)免受伤害的[物品](`item`)。许多模组添加新护甲套装（例如铜护甲）。

## 自定义护甲套装(`Custom Armor Sets`)

人形实体的护甲套装通常包含四件物品：头部头盔、胸部胸甲、腿部护腿和脚部靴子。还有适用于狼、马和羊驼的"身体"护甲槽位的护甲。所有这些物品通常通过七个[数据组件](`datacomponents`)实现：

- `DataComponents#MAX_DAMAGE`和`#DAMAGE`用于耐久度
- `#MAX_STACK_SIZE`设置堆叠大小为`1`
- `#REPAIRABLE`用于铁砧修复护甲
- `#ENCHANTABLE`设置最大[附魔](`enchantment`)值
- `#ATTRIBUTE_MODIFIERS`提供护甲值、护甲韧性和击退抗性
- `#EQUIPPABLE`定义实体如何装备物品

通常，人形实体使用`Item.Properties#humanoidArmor`，狼用`wolfArmor`，马用`horseArmor`设置护甲。人形护甲结合`ArmorMaterial`和`ArmorType`设置组件。参考值可在`ArmorMaterials`中找到。此示例使用铜护甲材料，可按需调整值：

```java
public static final ArmorMaterial COPPER_ARMOR_MATERIAL = new ArmorMaterial(
    // 护甲材料耐久度乘数
    // ArmorType有不同基础耐久度单位：
    // - 头盔(HELMET): 11
    // - 胸甲(CHESTPLATE): 16
    // - 护腿(LEGGINGS): 15
    // - 靴子(BOOTS): 13
    // - 身体(BODY): 16
    15,
    // 决定防御值（护甲条上的半盔甲数量）
    // 基于ArmorType
    Util.make(new EnumMap<>(ArmorType.class), map -> {
        map.put(ArmorItem.Type.BOOTS, 2);
        map.put(ArmorItem.Type.LEGGINGS, 4);
        map.put(ArmorItem.Type.CHESTPLATE, 6);
        map.put(ArmorItem.Type.HELMET, 2);
        map.put(ArmorItem.Type.BODY, 4);
    }),
    // 决定护甲附魔能力，值越高附魔越好
    // 金为25，铜稍低
    20,
    // 装备时播放的音效（Holder包装）
    SoundEvents.ARMOR_EQUIP_GENERIC,
     // 护甲韧性值（伤害计算中的额外值）
    // 仅钻石和下界合金大于0
    0,
    // 击退抗性值（总抗性≥1时免疫击退）
    // 仅下界合金大于0
    0,
    // 修复此护甲可用的物品标签
    Tags.Items.INGOTS_COPPER,
    // 下文EquipmentClientInfo JSON的资源键
    // 指向assets/examplemod/equipment/copper.json
    ResourceKey.create(EquipmentAssets.ROOT_ID, ResourceLocation.fromNamespaceAndPath("examplemod", "copper"))
);
```

创建`ArmorMaterial`后，可用于[注册](`registering`)护甲：

```java
// ITEMS是DeferredRegister.Items
public static final DeferredItem<Item> COPPER_HELMET = ITEMS.registerItem(
    "copper_helmet",
    props -> new Item(
        props.humanoidArmor(
            // 使用的材料
            COPPER_ARMOR_MATERIAL,
            // 创建的护甲类型
            ArmorType.HELMET
        )
    )
);

public static final DeferredItem<Item> COPPER_CHESTPLATE =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));
public static final DeferredItem<Item> COPPER_LEGGINGS =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));
public static final DeferredItem<Item> COPPER_BOOTS =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));

public static final DeferredItem<Item> COPPER_WOLF_ARMOR = ITEMS.registerItem(
    "copper_wolf_armor",
    props -> new Item(
        // 使用的材料
        props.wolfArmor(COPPER_ARMOR_MATERIAL)
    )
);

public static final DeferredItem<Item> COPPER_HORSE_ARMOR =
    ITEMS.registerItem("copper_horse_armor", props -> new Item(props.horseArmor(...)));
```

要从零创建护甲或类护甲物品，可组合以下部分实现：

- 通过`Item.Properties#component`设置`DataComponents#EQUIPPABLE`添加自定义需求的`Equippable`
- 通过`Item.Properties#attributes`添加物品属性（如护甲值、韧性、击退抗性）
- 通过`Item.Properties#durability`添加物品耐久度
- 通过`Item.Properties#repariable`允许物品修复
- 通过`Item.Properties#enchantable`允许物品附魔
- 将护甲添加到`minecraft:enchantable/*`的`ItemTags`以便应用特定附魔

### `Equippable`

`Equippable`是定义实体如何装备物品及游戏内渲染方式的数据组件。任何物品只要拥有此组件即可被装备（如鞍、羊驼地毯）。每个含此组件的物品只能装备到单个`EquipmentSlot`。

可通过记录构造函数直接创建`Equippable`，或使用`Equippable#builder`设置字段默认值后`build`：

```java
// 假设有DeferredRegister.Items ITEMS
public static final DeferredItem<Item> EQUIPPABLE = ITEMS.registerSimpleItem(
    "equippable",
    new Item.Properties().component(
        DataComponents.EQUIPPABLE,
        // 设置物品可装备的槽位
        Equippable.builder(EquipmentSlot.HELMET)
            // 装备时播放的音效（Holder包装，默认ARMOR_EQUIP_GENERIC）
            .setEquipSound(SoundEvents.ARMOR_EQUIP_GENERIC)
            // EquipmentClientInfo JSON的资源键（指向assets/examplemod/equipment/equippable.json）
            // 未设置时不渲染装备
            .setAsset(ResourceKey.create(EquipmentAssets.ROOT_ID, ResourceLocation.fromNamespaceAndPath("examplemod", "equippable")))
            // 穿戴时覆盖玩家屏幕的纹理相对路径（如南瓜模糊）
            // 指向assets/examplemod/textures/equippable.png
            .setCameraOverlay(ResourceLocation.withDefaultNamespace("examplemod", "equippable"))
            // 可装备此物品的实体类型HolderSet（直接或标签）
            // 未设置时任何实体可装备
            .setAllowedEntities(EntityType.ZOMBIE)
            // 是否可从发射器装备（默认true）
            .setDispensable(true),
            // 是否可在快速装备时交换（默认true）
            .setSwappable(false),
            // 受攻击时物品是否受损（护甲通常为true，需同时为可损坏物品）
            .setDamageOnHurt(false)
            // 是否可通过交互装备到其他实体（如右键，默认false）
            .setEquipOnInteract(true)
            // 是否可用带SHEAR_REMOVE_ARMOR能力的物品剪下（默认false）
            .setCanBeSheared(true)
            // 剪下装备时播放的音效（Holder包装，默认SHEARS_SNIP）
            .setShearingSound(SoundEvents.SADDLE_UNEQUIP)
            .build()
    )
);
```

## **装备资源**(`Equipment Assets`)

游戏中有护甲后，穿戴时不会渲染（因未指定渲染方式）。需在[资源包](`respack`)的`equipment`文件夹（`assets`目录）创建`EquipmentClientInfo` JSON，位置由`Equippable#assetId`指定。`EquipmentClientInfo`定义每个渲染层使用的关联纹理。

`EquipmentClientInfo`功能上是`EquipmentClientInfo.LayerType`到`EquipmentClientInfo.Layer`列表的映射。

`LayerType`可视为某实例要渲染的纹理组。例如`LayerType#HUMANOID`由`HumanoidArmorLayer`用于渲染人形实体头胸脚；`LayerType#WOLF_BODY`由`WolfArmorLayer`渲染身体护甲。相同可装备类型（如铜护甲）可组合到一个装备信息JSON。

`LayerType`映射到要按顺序应用渲染的`Layer`列表。`Layer`代表单个要渲染的纹理。第一个参数是纹理位置（相对于`textures/entity/equipment`）。

第二个参数是可选的[可染色](`tinting`)标记（作为`EquipmentClientInfo.Dyeable`）。`Dyeable`包含未染色时的默认RGB颜色。未设置时使用纯白色。

:::warning
要使染色生效，物品必须在[`ItemTags#DYEABLE`](`tag`)中且设置`DataComponents#DYED_COLOR`为RGB值。
:::

第三个参数是布尔值，指示是否使用渲染时传入的纹理替代`Layer`内定义的纹理（如玩家自定义披风或鞘翅纹理）。

为铜护甲材料创建装备信息。假设每层有两个纹理：护甲本体和可染色覆盖层。动物护甲使用可传入的动态纹理。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 位于assets/examplemod/equipment/copper.json
{
    "layers": {
        // 人形头胸脚
        "humanoid": [
            {
                // 护甲纹理（指向assets/examplemod/textures/entity/equipment/humanoid/copper/outer.png）
                "texture": "examplemod:copper/outer"
            },
            {
                // 覆盖层纹理（指向assets/examplemod/textures/entity/equipment/humanoid/copper/outer_overlay.png）
                "texture": "examplemod:copper/outer_overlay",
                // 可染色（未染色时颜色）
                "dyeable": {
                    "color_when_undyed": 7767006 // 0x7683DE
                }
            }
        ],
        // 人形腿部
        "humanoid_leggings": [
            {
                "texture": "examplemod:copper/inner"
            },
            {
                "texture": "examplemod:copper/inner_overlay",
                "dyeable": {
                    "color_when_undyed": 7767006
                }
            }
        ],
        // 狼护甲
        "wolf_body": [
            {
                "texture": "examplemod:copper/wolf",
                // 使用传入纹理替代
                "use_player_texture": true
            }
        ],
        // 马护甲
        "horse_body": [
            {
                "texture": "examplemod:copper/horse",
                "use_player_texture": true
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="数据生成">

```java
public class MyEquipmentInfoProvider implements DataProvider {

    private final PackOutput.PathProvider path;

    public MyEquipmentInfoProvider(PackOutput output) {
        this.path = output.createPathProvider(PackOutput.Target.RESOURCE_PACK, "equipment");
    }

    private void add(BiConsumer<ResourceLocation, EquipmentClientInfo> registrar) {
        registrar.accept(
            // 必须匹配Equippable#assetId
            ResourceLocation.fromNamespaceAndPath("examplemod", "copper"),
            EquipmentClientInfo.builder()
                // 人形头胸脚
                .addLayers(
                    EquipmentClientInfo.LayerType.HUMANOID,
                    // 基础纹理
                    new EquipmentClientInfo.Layer(
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/outer"),
                        Optional.empty(),
                        false
                    ),
                    // 覆盖层纹理
                    new EquipmentClientInfo.Layer(
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/outer_overlay"),
                        // 未染色时颜色
                        Optional.of(new EquipmentClientInfo.Dyeable(Optional.of(0x7683DE))),
                        false
                    )
                )
                // 人形腿部
                .addLayers(
                    EquipmentClientInfo.LayerType.HUMANOID_LEGGINGS,
                    new EquipmentClientInfo.Layer(
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/inner"),
                        Optional.empty(),
                        false
                    ),
                    new EquipmentClientInfo.Layer(
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/inner_overlay"),
                        Optional.of(new EquipmentClientInfo.Dyeable(Optional.of(0x7683DE))),
                        false
                    )
                )
                // 狼护甲
                .addLayers(
                    EquipmentClientInfo.LayerType.WOLF_BODY,
                    new EquipmentClientInfo.Layer(
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/wolf"),
                        Optional.empty(),
                        true // 使用传入纹理
                    )
                )
                // 马护甲
                .addLayers(
                    EquipmentClientInfo.LayerType.HORSE_BODY,
                    new EquipmentClientInfo.Layer(
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/horse"),
                        Optional.empty(),
                        true
                    )
                )
                .build()
        );
    }

    @Override
    public CompletableFuture<?> run(CachedOutput cache) {
        Map<ResourceLocation, EquipmentClientInfo> map = new HashMap<>();
        this.add((name, info) -> {
            if (map.putIfAbsent(name, info) != null) {
                throw new IllegalStateException("重复注册装备客户端信息: " + name);
            }
        });
        return DataProvider.saveAll(cache, EquipmentClientInfo.CODEC, this.pathProvider, map);
    }

    @Override
    public String getName() {
        return "装备客户端信息: " + MOD_ID;
    }
}

@SubscribeEvent // 在模组事件总线
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(MyEquipmentInfoProvider::new);
}
```

</TabItem>
</Tabs>

## **装备渲染**(`Equipment Rendering`)

装备信息通过`EntityRenderer`或其`RenderLayer`渲染函数中的`EquipmentLayerRenderer`渲染。`EquipmentLayerRenderer`通过`EntityRendererProvider.Context#getEquipmentRenderer`从渲染上下文获取。若需要`EquipmentClientInfo`，可通过`EntityRendererProvider.Context#getEquipmentAssets`获取。

默认以下层渲染关联的`EquipmentClientInfo.LayerType`：

| `LayerType`             | `RenderLayer`          | 使用者                                                        |
|:-----------------------:|:----------------------:|:---------------------------------------------------------------|
| `HUMANOID`              | `HumanoidArmorLayer`   | 玩家、人形生物（僵尸、骷髅）、盔甲架 |
| `HUMANOID_LEGGINGS`     | `HumanoidArmorLayer`   | 玩家、人形生物、盔甲架 |
| `WINGS`                 | `WingsLayer`           | 玩家、人形生物、盔甲架 |
| `WOLF_BODY`             | `WolfArmorLayer`       | 狼                                                           |
| `HORSE_BODY`            | `HorseArmorLayer`      | 马                                                          |
| `LLAMA_BODY`            | `LlamaDecorLayer`      | 羊驼、行商羊驼                                            |
| `PIG_SADDLE`            | `SimpleEquipmentLayer` | 猪                                                            |
| `STRIDER_SADDLE`        | `SimpleEquipmentLayer` | 炽足兽                                                        |
| `CAMEL_SADDLE`          | `SimpleEquipmentLayer` | 骆驼                                                          |
| `HORSE_SADDLE`          | `SimpleEquipmentLayer` | 马                                                          |
| `DONKEY_SADDLE`         | `SimpleEquipmentLayer` | 驴                                                         |
| `MULE_SADDLE`           | `SimpleEquipmentLayer` | 骡                                                           |
| `ZOMBIE_HORSE_SADDLE`   | `SimpleEquipmentLayer` | 僵尸马                                                   |
| `SKELETON_HORSE_SADDLE` | `SimpleEquipmentLayer` | 骷髅马                                                 |
| `HAPPY_GHAST_BODY`      | `SimpleEquipmentLayer` | 悦灵                                                    |

`EquipmentLayerRenderer`仅有一个方法渲染装备层`renderLayers`：

```java
// 在某个渲染方法中（equipmentLayerRenderer为字段）
this.equipmentLayerRenderer.renderLayers(
    // 要渲染的层类型
    EquipmentClientInfo.LayerType.HUMANOID,
    // 代表EquipmentClientInfo JSON的资源键（通过assetId在EQUIPPABLE中设置）
    stack.get(DataComponents.EQUIPPABLE).assetId().orElseThrow(),
    // 应用装备信息的模型（通常与实体模型分离）
    model,
    // 代表模型渲染的物品堆（仅用于获取染色、光效和盔甲纹饰信息）
    stack,
    // 在位姿栈中正确位置渲染模型
    poseStack,
    // 获取渲染类型顶点消费者的缓冲区源
    bufferSource,
    // 打包光照纹理
    lighting,
    // 当某层use_player_texture为true时使用的绝对纹理路径
    ResourceLocation.fromNamespaceAndPath("examplemod", "textures/other_texture.png")
);
```

[item]: index.md
[datacomponents]: datacomponents.md
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[livingentity]: ../entities/livingentity.md
[registering]: ../concepts/registries.md#methods-for-registering
[rendering]: #equipment-rendering
[respack]: ../resources/index.md#assets
[tag]: ../resources/server/tags.md
[tinting]: ../resources/client/models/index.md#tinting