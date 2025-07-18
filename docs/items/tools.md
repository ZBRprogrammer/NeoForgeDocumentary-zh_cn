---
sidebar_position: 4
---
# **工具**(`Tools`)

[**物品**][item]的主要用途是破坏[**方块**][block]。许多模组添加新工具套装（如铜质工具）或新工具类型（如锤子）。

## 自定义工具套装

工具套装通常包含五件物品：镐、斧、锹、锄和剑（剑非传统工具，但为统一性包含在此）。所有工具均通过以下八个[**数据组件**][datacomponents]实现：

- `DataComponents#MAX_DAMAGE` 和 `#DAMAGE`（耐久度）
- `#MAX_STACK_SIZE`（堆叠数设为 `1`）
- `#REPAIRABLE`（铁砧修复）
- `#ENCHANTABLE`（最大[附魔][enchantment]等级）
- `#ATTRIBUTE_MODIFIERS`（攻击伤害与攻击速度）
- `#TOOL`（挖掘信息）
- `#WEAPON`（物品伤害与禁用盾牌）

通常通过 `Item.Properties#tool`、`#sword` 或工具委托方法（`pickaxe`、`axe`、`hoe`、`shovel`）配置工具。这些方法通常传入工具材料记录 `ToolMaterial`。注意：其他工具（如剪刀）未通过数据组件实现通用挖掘逻辑，而是直接继承 `Item` 并重写相关方法。交互行为（默认右键）也无专用数据组件，故锹、斧、锄分别有专属工具类 `ShovelItem`、`AxeItem`、`HoeItem`。

创建标准工具套装需先定义 `ToolMaterial`。参考值见 `ToolMaterial` 常量。此示例使用铜质工具：

```java
// 铜质介于石质与铁质之间
public static final ToolMaterial COPPER_MATERIAL = new ToolMaterial(
        // 标记此材料无法破坏的方块（详见下文）
        MyBlockTags.INCORRECT_FOR_COPPER_TOOL,
        // 耐久度（石质131，铁质250）
        200,
        // 挖掘速度（剑类不使用此值）（石质4，铁质6）
        5f,
        // 攻击伤害加成（不同工具有不同计算方式，如剑伤害为 getAttackDamageBonus() + 4）
        // 石质1（剑5伤害），铁质2（剑6伤害），铜剑为5.5伤害
        1.5f,
        // 附魔能力（影响附魔品质）（金质22，铜质略低）
        20,
        // 修复材料标记
        Tags.Items.INGOTS_COPPER
);
```

定义 `ToolMaterial` 后，可用于[注册][registering]工具。所有 `tool` 委托方法参数相同：

```java
// ITEMS 是 DeferredRegister.Items
public static final DeferredItem<Item> COPPER_SWORD = ITEMS.registerItem(
    "copper_sword",
    props -> new Item(
        // 物品属性
        props.sword(
            // 使用材料
            COPPER_MATERIAL,
            // 类型专属攻击伤害加成（剑3，锹1.5，镐1，斧/锄各异）
            3,
            // 类型专属攻击速度修正（玩家默认4，剑需1.6故设-2.4；锹-3，镐-2.8，斧/锄各异）
            -2.4f,
        )
    )
);

public static final DeferredItem<Item> COPPER_AXE = ITEMS.registerItem("copper_axe", props -> new Item(props.axe(...)));
public static final DeferredItem<Item> COPPER_PICKAXE = ITEMS.registerItem("copper_pickaxe", props -> new Item(props.pickaxe(...)));
public static final DeferredItem<Item> COPPER_SHOVEL = ITEMS.registerItem("copper_shovel", props -> new Item(props.shovel(...)));
public static final DeferredItem<Item> COPPER_HOE = ITEMS.registerItem("copper_hoe", props -> new Item(props.hoe(...)));
```

:::note
`tool` 方法额外接受两个参数：表示可挖掘方块的 `TagKey`，以及击中目标时禁用格挡（如盾牌）的秒数。
:::

### **标记**(`Tags`)

创建 `ToolMaterial` 时需分配方块[标记][tags]，包含无法被此工具破坏的方块（如 `minecraft:incorrect_for_stone_tool` 含钻石矿）。为便于分配挖掘等级，存在"需特定工具挖掘"的标记（如 `minecraft:needs_iron_tool` 含钻石矿，`minecraft:needs_diamond_tool` 含黑曜石）。

可直接复用现有标记（如铜质工具作为强化石质工具时传入 `BlockTags#INCORRECT_FOR_STONE_TOOL`）。

或创建自定义标记：
```java
// 此标记用于添加需铜质工具挖掘的方块
public static final TagKey<Block> NEEDS_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "needs_copper_tool"));

// 此标记传入材料
public static final TagKey<Block> INCORRECT_FOR_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "incorrect_for_cooper_tool"));
```

填充标记（如铜质工具可挖掘金矿/金块/红石矿，但不可挖钻石/绿宝石）。标记文件位于 `src/main/resources/data/mod_id/tags/block/needs_copper_tool.json`：
```json5
{
    "values": [
        "minecraft:gold_block",
        "minecraft:raw_gold_block",
        "minecraft:gold_ore",
        "minecraft:deepslate_gold_ore",
        "minecraft:redstone_ore",
        "minecraft:deepslate_redstone_ore"
    ]
}
```

传入材料的标记需排除铜质可挖的方块。文件位于 `src/main/resources/data/mod_id/tags/block/incorrect_for_cooper_tool.json`：
```json5
{
    "values": [
        "#minecraft:incorrect_for_stone_tool"
    ],
    "remove": [
        "#mod_id:needs_copper_tool"
    ]
}
```

最后将标记传入材料实例。

检查工具能否使方块掉落物品，调用 `Tool#isCorrectForDrops`（通过 `ItemStack#get` + `DataComponents#TOOL` 获取 `Tool` 实例）。

## 自定义工具

通过 `Item.Properties#component` 添加 `Tool` [数据组件][datacomponents]（通过 `DataComponents#TOOL`）可创建自定义工具。

`Tool` 包含：
- `Tool.Rule` 列表（规则集）
- 默认挖掘速度（默认为 `1`）
- 挖掘时的耐久损耗（默认为 `1`）

`Tool.Rule` 含三部分信息：
1. 应用规则的方块 `HolderSet`
2. 可选：挖掘速度
3. 可选：是否可掉落方块

若可选值未设置，则检查其他规则。无匹配规则时使用默认挖掘速度且方块不可掉落。

:::note
通过 `Registry#getOrThrow` 可从 `TagKey` 创建 `HolderSet`。
:::

无需使用现有 `ToolMaterial` 即可创建任意工具或多功能工具（如镐斧一体），通过组合以下部分实现：
- 通过 `Item.Properties#component` 设置 `DataComponents#TOOL` 添加自定义规则
- 通过 `Item.Properties#attributes` 添加[属性修饰符][attributemodifier]（如攻击伤害/速度）
- 通过 `Item.Properties#durability` 添加耐久度
- 通过 `Item.Properties#repairable` 允许修复
- 通过 `Item.Properties#enchantable` 允许附魔
- 通过 `Item.Properties#component` 设置 `DataComponents#WEAPON` 允许作为武器并禁用格挡
- 重写 `IItemExtension#canPerformAction` 定义可执行的[**物品能力**][itemability]
- 若需右键修改方块状态，调用 `IBlockExtension#getToolModifiedState`
- 将工具加入 `minecraft:enchantable/*` 物品标记以允许特定附魔
- 将工具加入 `minecraft:*_preferred_weapons` 标记以使生物偏好使用

盾牌可应用 `DataComponents#EQUIPPABLE`（副手装备）和 `DataComponents#BLOCKS_ATTACKS`（激活时减伤）。

## **物品能力**(`ItemAbility`s)

`ItemAbility` 抽象定义物品可执行的操作（含左键/右键行为）。NeoForge 在 `ItemAbilities` 类中提供默认能力：
- 斧：剥离（原木）、刮削（氧化铜）、脱蜡（涂蜡铜）
- 锹：铲平（土径）、熄灭（营火）
- 剪刀：采集（蜂巢）、卸甲（狼）、雕刻（南瓜）、拆除（绊线）、修剪（植物）
- 其他：剑横扫、锄耕地、盾格挡、鱼竿抛掷、三叉戟投掷、刷子清扫、打火石点火

创建自定义 `ItemAbility` 使用 `ItemAbility#get`（按需创建新能力）。在自定义工具中按需重写 `IItemExtension#canPerformAction`。

查询 `ItemStack` 能否执行某能力，调用 `IItemStackExtension#canPerformAction`（适用于任意物品，不限工具）。

[block]: ../blocks/index.md
[datacomponents]: datacomponents.md
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[equippable]: armor.md#equippable
[item]: index.md
[itemability]: #itemabilitys
[registering]: ../concepts/registries.md#methods-for-registering
[tags]: ../resources/server/tags.md