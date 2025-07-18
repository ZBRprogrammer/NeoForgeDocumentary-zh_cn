# **战利品表** (`Loot Tables`)

**战利品表** (`Loot Tables`) 是用于定义随机战利品掉落的数据文件。执行战利品表时会返回一个（可能为空的）物品堆叠列表，其结果取决于（伪）随机性。战利品表位于 `data/<mod_id>/loot_table/<name>.json`。例如，泥土方块使用的 `minecraft:blocks/dirt` 战利品表位于 `data/minecraft/loot_table/blocks/dirt.json`。

Minecraft 在游戏多个环节使用战利品表，包括[方块][block]掉落、[实体][entity]掉落、宝箱战利品、钓鱼战利品等。引用战利品表的方式取决于上下文：

- 默认每个方块都有关联的战利品表，位于 `<block_namespace>:blocks/<block_name>`。可通过在方块 `Properties` 上调用 `#noLootTable` 禁用，此时方块不生成战利品表且无掉落物（主要用于空气类或技术性方块）。
- 默认每个未调用 `EntityType.Builder#noLootTable` 的实体（通常是 `MobCategory#MISC` 类实体）都有关联战利品表，位于 `<entity_namespace>:entities/<entity_name>`。可通过重写 `#getLootTable` 修改（例如绵羊根据羊毛颜色使用不同战利品表）。
- 结构中的宝箱在其方块实体数据中指定战利品表。Minecraft 将所有宝箱战利品表存储在 `minecraft:chests/<chest_name>`；模组建议但不强制遵循此惯例。
- 袭击后村民向玩家投掷的礼物战利品表在 [`neoforge:raid_hero_gifts` 数据映射][raidherogifts] 中定义。
- 其他战利品表（如钓鱼战利品表）通过 `level.getServer().reloadableRegistries().getLootTable(lootTableKey)` 获取。所有原版战利品表位置列表见 `BuiltInLootTables`。

:::warning
战利品表通常应仅为你模组所属内容创建。修改现有战利品表应使用[全局战利品修改器 (GLM)][glm]。
:::

因战利品表系统复杂性，其由多个子系统组成：

## **战利品条目** (`Loot Entry`)

**战利品条目** (`loot entry`)，代码中由抽象类 `LootPoolEntryContainer` 表示，是单一战利品元素。可指定掉落一个或多个物品。

原版提供 8 种战利品条目类型。通过 `LootPoolEntryContainer` 超类，所有条目具有以下属性：

- `weight`：权重值（默认为 1）。用于控制物品出现概率（例如权重 3 和 1 的条目，选中概率分别为 75% 和 25%）。
- `quality`：质量值（默认为 0）。若非零，此值乘以[战利品上下文][context]的幸运值后加到权重上。
- `conditions`：[战利品条件][lootcondition]列表。任一条件失败则忽略该条目。
- `functions`：应用于条目输出的[战利品函数][lootfunction]列表。

战利品条目通常分为两类：**单体条目** (`singletons`，公共超类 `LootPoolSingletonContainer`) 和 **复合条目** (`composites`，公共超类 `CompositeEntryBase`)。原版单体条目类型：

- `minecraft:empty`：空条目（无物品）。代码中通过 `EmptyLootItem#emptyItem` 创建。
- `minecraft:item`：单体物品条目，掉落指定物品。代码中通过 `LootItem#lootTableItem` 创建。
  - 堆叠大小、数据组件等可通过战利品函数设置。
- `minecraft:tag`：标签条目，掉落指定标签内所有物品。`expand` 布尔属性决定两种形式：`true` 时为标签内每件物品生成独立条目；`false` 时单一条目掉落所有物品。代码中通过 `TagEntry#tagContents` (`expand=false`) 或 `TagEntry#expandTag` (`expand=true`) 创建。
  - 例如：`#minecraft:planks` 标签在 `expand=true` 时生成 11 个原版木板条目（每个权重/函数独立）；`expand=false` 时单一条目掉落所有木板。
- `minecraft:dynamic`：引用动态掉落的条目。动态掉落用于在代码中添加无法预定义的条目（包含 ID 和实际添加物品的 `Consumer<ItemStack>`）。代码中通过 `DynamicLoot#dynamicEntry` 创建。
- `minecraft:loot_table`：执行另一战利品表的条目，将其结果作为单一条目添加。目标表可通过 ID 指定或内联。代码中通过 `NestedLootTable#lootTableReference`（ID 指定）或 `NestedLootTable#inlineLootTable`（内联表）创建。

原版复合条目类型：

- `minecraft:group`：包含有序条目列表的条目（按顺序执行）。代码中通过 `EntryGroup#list` 或 `LootPoolSingletonContainer.Builder#append` 创建。
- `minecraft:sequence`：类似 `group`，但任一子条目失败即终止执行（丢弃后续条目）。代码中通过 `SequentialEntry#sequential` 或 `LootPoolSingletonContainer.Builder#then` 创建。
- `minecraft:alternatives`：类似 `sequence` 的反向逻辑，任一子条目成功即终止执行（丢弃后续条目）。代码中通过 `AlternativesEntry#alternatives` 或 `LootPoolSingletonContainer.Builder#otherwise` 创建。

模组开发者也可定义[自定义战利品条目类型][customentry]。

## **战利品池** (`Loot Pool`)

**战利品池** (`loot pool`) 本质是战利品条目列表。战利品表可含多个池，每个池独立执行。

战利品池可包含以下内容：

- `entries`：战利品条目列表。
- `conditions`：应用于池的[战利品条件][lootcondition]列表。任一条件失败则跳过整个池。
- `functions`：应用于池内所有条目输出的[战利品函数][lootfunction]列表。
- `rolls` 和 `bonus_rolls`：两个数字提供器（见下文），共同决定池的执行次数（公式：rolls + bonus_rolls * luck，幸运值在[战利品参数][parameters]中设置）。
- `name`：池名称（NeoForge 新增）。[GLM][glm] 可使用此名称。未指定时为 `custom#` 前缀的池哈希值。

## **数字提供器** (`Number Provider`)

**数字提供器** (`number providers`) 是在数据包上下文中获取（伪）随机数的方式。主要用于战利品表，也见于世界生成等场景。原版提供六类数字提供器：

- `minecraft:constant`：常量浮点值（按需取整）。通过 `ConstantValue#exactly` 创建。
- `minecraft:uniform`：均匀分布随机整数/浮点数（设最小/最大值）。通过 `UniformGenerator#between` 创建。
- `minecraft:binomial`：二项分布随机整数（设 n 和 p 值）。通过 `BinomialDistributionGenerator#binomial` 创建。
- `minecraft:score`：给定实体目标、记分项和（可选）缩放值，获取实体记分板值并缩放。通过 `ScoreboardValue#fromScoreboard` 创建。
- `minecraft:storage`：从命令存储的指定 NBT 路径获取值。通过 `new StorageValue` 创建。
- `minecraft:enchantment_level`：附魔等级值提供器。通过 `EnchantmentLevelProvider#forEnchantmentLevel` 创建（需提供 `LevelBasedValue`）。有效 `LevelBasedValue` 类型：
  - 常量值（无类型）：`LevelBasedValue#constant`
  - `minecraft:linear`：随附魔等级线性增长的值（可加基础值）。`LevelBasedValue#perLevel`
  - `minecraft:levels_squared`：附魔值平方后加基础值。`new LevelBasedValue.LevelsSquared`
  - `minecraft:fraction`：接受两个 `LevelBasedValue` 创建分数。`new LevelBasedValue.Fraction`
  - `minecraft:clamped`：接受另一 `LevelBasedValue` 及最小/最大值，计算结果后钳制。`new LevelBasedValue.Clamped`
  - `minecraft:lookup`：接受 `List<Float>` 和备用 `LevelBasedValue`，按等级查表（等级 1 对应首元素），缺省时用备用值。`LevelBasedValue#lookup`

模组开发者可注册[自定义数字提供器][customnumber]和[自定义等级基值][customlevelbased]。

## **战利品参数** (`Loot Parameters`)

**战利品参数** (`loot parameter`)，内部称为 `ContextKey<T>`，是执行战利品表时提供的参数（`T` 为参数类型，如 `BlockPos` 或 `Entity`）。可供[战利品条件][lootcondition]和[战利品函数][lootfunction]使用（例如 `minecraft:killed_by_player` 条件检查 `minecraft:player` 参数）。

原版提供以下战利品参数：

- `minecraft:origin`：关联位置（如宝箱坐标）。通过 `LootContextParams.ORIGIN` 访问。
- `minecraft:tool`：关联物品堆叠（如破坏方块的物品）。不一定是工具。通过 `LootContextParams.TOOL` 访问。
- `minecraft:enchantment_level`：附魔等级（用于附魔逻辑）。通过 `LootContextParams.ENCHANTMENT_LEVEL` 访问。
- `minecraft:enchantment_active`：使用物品是否有附魔（如精准采集检查）。通过 `LootContextParams.ENCHANTMENT_ACTIVE` 访问。
- `minecraft:block_state`：关联方块状态（如被破坏的方块）。通过 `LootContextParams.BLOCK_STATE` 访问。
- `minecraft:block_entity`：关联方块实体（如被破坏方块的实体）。用于潜影盒保存物品栏到掉落物等场景。通过 `LootContextParams.BLOCK_ENTITY` 访问。
- `minecraft:explosion_radius`：当前爆炸半径。主要用于对掉落物应用爆炸衰减。通过 `LootContextParams.EXPLOSION_RADIUS` 访问。
- `minecraft:this_entity`：关联实体（通常是被击杀实体）。通过 `LootContextParams.THIS_ENTITY` 访问。
- `minecraft:damage_source`：关联[伤害源][damagesource]（通常是击杀实体的伤害源）。通过 `LootContextParams.DAMAGE_SOURCE` 访问。
- `minecraft:attacking_entity`：攻击实体（通常是击杀者）。通过 `LootContextParams.ATTACKING_ENTITY` 访问。
- `minecraft:direct_attacking_entity`：直接攻击实体（例如攻击实体是骷髅时，直接攻击实体是箭）。通过 `LootContextParams.DIRECT_ATTACKING_ENTITY` 访问。
- `minecraft:last_damage_player`：关联玩家（通常是最后攻击被击杀实体的玩家，含间接击杀）。用于玩家击杀专属掉落等场景。通过 `LootContextParams.LAST_DAMAGE_PLAYER` 访问。

自定义战利品参数可通过 `new ContextKey<T>` 创建（仅需资源定位符包装，无需注册）。

### **实体目标** (`Entity Targets`)

**实体目标** (`entity targets`) 是战利品条件和函数使用的类型（代码中为 `LootContext.EntityTarget` 枚举），用于指定条件/函数中查询的实体参数。有效值：

- `"this"` 或 `LootContext.EntityTarget.THIS`：代表 `"minecraft:this_entity"` 参数。
- `"attacker"` 或 `LootContext.EntityTarget.ATTACKER`：代表 `"minecraft:attacking_entity"` 参数。
- `"direct_attacker"` 或 `LootContext.EntityTarget.DIRECT_ATTACKER`：代表 `"minecraft:direct_attacking_entity"` 参数。
- `"attacking_player"` 或 `LootContext.EntityTarget.ATTACKING_PLAYER`：代表 `"minecraft:last_damage_player"` 参数。

例如 `minecraft:entity_properties` 战利品条件接受实体目标参数，允许检查全部四类参数。

### **战利品参数集** (`Loot Parameter Sets`)

**战利品参数集** (`loot parameter sets`)，又称战利品表类型（代码中为 `ContextKeySet`），是必需和可选战利品参数的集合。其本质是包装两个 `Set<ContextKey<?>>`（必需参数 `#required` 和允许参数 `#allowed`），用于验证参数使用合法性及执行表时必需参数存在性。也用于进度和附魔逻辑。

原版提供以下战利品参数集（必需参数**加粗**，可选参数*斜体*；代码常量位于 `LootContextParamSets`）：

| ID                               | 代码常量名           | 战利品参数                                                                                                                                                                                                                                                                                            | 用途                                                     |
|----------------------------------|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| `minecraft:empty`                | `EMPTY`                | 无                                                                                                                                                                                                                                                                                                                  | 回退用途                                        |
| `minecraft:generic`              | `ALL_PARAMS`           | **`minecraft:origin`**, **`minecraft:tool`**, **`minecraft:block_state`**, **`minecraft:block_entity`**, **`minecraft:explosion_radius`**, **`minecraft:this_entity`**, **`minecraft:damage_source`**, **`minecraft:attacking_entity`**, **`minecraft:direct_attacking_entity`**, **`minecraft:last_damage_player`** | 验证                                              |
| `minecraft:command`              | `COMMAND`              | **`minecraft:origin`**, _`minecraft:this_entity`_                                                                                                                                                                                                                                                                    | 命令                                                 |
| `minecraft:selector`             | `SELECTOR`             | **`minecraft:origin`**, _`minecraft:this_entity`_                                                                                                                                                                                                                                                                    | 命令中的实体选择器                             |
| `minecraft:block`                | `BLOCK`                | **`minecraft:origin`**, **`minecraft:tool`**, **`minecraft:block_state`**, _`minecraft:block_entity`_, _`minecraft:explosion_radius`_, _`minecraft:this_entity`_                                                                                                                                                     | 方块破坏                                           |
| `minecraft:block_use`            | `BLOCK_USE`            | **`minecraft:origin`**, **`minecraft:block_state`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                     | 无原版用途                                          |
| `minecraft:hit_block`            | `HIT_BLOCK`            | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:block_state`**, **`minecraft:this_entity`**                                                                                                                                                                                                  | 引雷附魔                               |
| `minecraft:chest`                | `CHEST`                | **`minecraft:origin`**, _`minecraft:this_entity`_, _`minecraft:attacking_entity`_                                                                                                                                                                                                                                    | 宝箱、运输矿车等容器                               |
| `minecraft:archaeology`          | `ARCHAEOLOGY`          | **`minecraft:origin`**, **`minecraft:this_entity`**, **`minecraft:tool`**                                                                                                                                                                                                                                                                    | 考古                                              |
| `minecraft:vault`                | `VAULT`                | **`minecraft:origin`**, _`minecraft:this_entity`_, _`minecraft:tool`_                                                                                                                                                                                                                                                                    | 试炼密室宝库奖励                             |
| `minecraft:entity`               | `ENTITY`               | **`minecraft:origin`**, **`minecraft:this_entity`**, **`minecraft:damage_source`**, _`minecraft:attacking_entity`_, _`minecraft:direct_attacking_entity`_, _`minecraft:last_damage_player`_                                                                                                                          | 实体击杀                                             |
| `minecraft:shearing`             | `SHEARING`             | **`minecraft:origin`**, **`minecraft:this_entity`**, **`minecraft:tool`**                                                                                                                                                                                                                                                                    | 实体剪毛（如绵羊）                            |
| `minecraft:equipment`            | `EQUIPMENT`            | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | 实体装备（如僵尸）                            |
| `minecraft:gift`                 | `GIFT`                 | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | 袭击英雄礼物                                  |
| `minecraft:barter`               | `PIGLIN_BARTER`        | **`minecraft:this_entity`**                                                                                                                                                                                                                                                                                          | 猪灵以物易物                                    |
| `minecraft:fishing`              | `FISHING`              | **`minecraft:origin`**, **`minecraft:tool`**, _`minecraft:this_entity`_, _`minecraft:attacking_entity`_                                                                                                                                                                                                              | 钓鱼                                              |
| `minecraft:enchanted_item`       | `ENCHANTED_ITEM`       | **`minecraft:tool`**, **`minecraft:enchantment_level`**                                                                                                                                                                                                                                                              | 多个附魔                                        |
| `minecraft:enchanted_entity`     | `ENCHANTED_ENTITY`     | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:this_entity`**                                                                                                                                                                                                                               | 多个附魔                                        |
| `minecraft:enchanted_damage`     | `ENCHANTED_DAMAGE`     | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:this_entity`**, **`minecraft:damage_source`**, _`minecraft:attacking_entity`_, _`minecraft:direct_attacking_entity`_                                                                                                                         | 伤害与保护类附魔                         |
| `minecraft:enchanted_location`   | `ENCHANTED_LOCATION`   | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:enchantment_active`**, **`minecraft:this_entity`**                                                                                                                                                                                           | 冰霜行者、灵魂疾行附魔                     |
| `minecraft:advancement_entity`   | `ADVANCEMENT_ENTITY`   | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | 多个[进度条件][advancement]              |
| `minecraft:advancement_location` | `ADVANCEMENT_LOCATION` | **`minecraft:origin`**, **`minecraft:tool`**, **`minecraft:block_state`**, **`minecraft:this_entity`**                                                                                                                                                                                                               | 多个[进度触发器][advancement]             |
| `minecraft:advancement_reward`   | `ADVANCEMENT_REWARD`   | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | [进度奖励][advancement]                  |

### **战利品上下文** (`Loot Context`)

**战利品上下文** (`loot context`) 是包含战利品表执行环境信息的对象，包括：

- 执行战利品表的 `ServerLevel`（通过 `#getLevel` 获取）。
- 用于执行表的 `RandomSource`（通过 `#getRandom` 获取）。
- 战利品参数（通过 `#hasParameter` 检查存在性，`#getParameter` 获取单个参数）。
- 幸运值（用于奖励次数和质量值计算，通常来自实体的幸运属性，通过 `#getLuck` 获取）。
- 动态掉落消费者（见[上文][entry]，通过 `#addDynamicDrops` 设置，无获取方法）。

## **战利品表** (`Loot Table`)

综合以上元素即构成战利品表。其 JSON 文件可包含以下值：

- `pools`：战利品池列表。
- `neoforge:conditions`：[数据加载条件][conditions]列表（**注意：这是数据加载条件，非[战利品条件][lootcondition]！**）。
- `functions`：应用于表内所有条目输出的[战利品函数][lootfunction]列表。
- `type`：战利品参数集（用于参数使用验证）。可选；缺失时跳过验证。
- `random_sequence`：战利品表的随机序列（资源定位符形式）。随机序列由 `Level` 提供，用于相同条件下生成一致的战利品结果（通常使用战利品表自身位置）。

示例战利品表格式：
```json5
{
    "type": "chest", // 战利品参数集
    "neoforge:conditions": [
        // 数据加载条件
    ],
    "functions": [
        // 表级战利品函数
    ],
    "pools": [ // 战利品池列表
        {
            "rolls": 1, // 池执行次数（设为 5 则生成 5 个结果）
            "bonus_rolls": 0.5, // 奖励次数
            "name": "my_pool",
            "conditions": [
                // 池级战利品条件
            ],
            "functions": [
                // 池级战利品函数
            ],
            "entries": [ // 战利品条目列表
                {
                    "type": "minecraft:item", // 条目类型
                    "name": "minecraft:dirt", // 类型特定属性（如物品名称）
                    "weight": 3, // 条目权重
                    "quality": 1, // 条目质量
                    "conditions": [
                        // 条目级战利品条件
                    ],
                    "functions": [
                        // 条目级战利品函数
                    ]
                }
            ]
        }
    ]
}
```

## **执行战利品表** (`Rolling a Loot Table`)

执行战利品表需两部分：表本身和战利品上下文。

获取战利品表：使用 `level.getServer().reloadableRegistries().getLootTable(lootTableId)`（战利品数据仅服务器可用，此逻辑需在[逻辑服务端][sides]运行）。

:::tip
原版战利品表 ID 见 `BuiltInLootTables` 类。方块战利品表通过 `BlockBehaviour#getLootTable` 获取；实体战利品表通过 `EntityType#getDefaultLootTable` 或 `Entity#getLootTable` 获取。
:::

构建参数集：创建 `LootParams.Builder` 实例：
```java
// 确保在服务端（否则转换失败）
LootParams.Builder builder = new LootParams.Builder((ServerLevel) level);
```

添加战利品上下文参数：
```java
// 按需添加参数（原版参数见 LootContextParams）
builder.withParameter(LootContextParams.ORIGIN, position);
// 此形式可接受 null（移除该参数现有值）
builder.withOptionalParameter(LootContextParams.ORIGIN, null);
// 添加动态掉落
builder.withDynamicDrop(ResourceLocation.fromNamespaceAndPath("examplemod", "example_dynamic_drop"), stackAcceptor -> {
    // 逻辑代码
});
// 设置幸运值（假设玩家存在）。无玩家时设为 0
builder.withLuck(player.getLuck());
```

创建 `LootParams` 并执行战利品表：
```java
// 如需可指定战利品参数集
LootParams params = builder.create(LootContextParamSets.EMPTY);
// 获取战利品表
LootTable table = level.getServer().reloadableRegistries().getLootTable(location);
// 执行战利品表
List<ItemStack> list = table.getRandomItems(params);
// 容器战利品（如宝箱）使用此方法（正确分割物品到容器栏位）
List<ItemStack> containerList = table.fill(container, params, someSeed);
```

:::danger
`LootTable` 额外提供 `#getRandomItemsRaw` 方法。与各 `#getRandomItems` 变体不同，此方法**不应用[全局战利品修改器][glm]**。仅在明确需求时使用。
:::

## **数据生成** (`Datagen`)

战利品表可通过注册 `LootTableProvider` 并传入 `LootTableSubProvider` 列表进行[数据生成][datagen]：

```java
@SubscribeEvent // 在模组事件总线
public static void onGatherData(GatherDataEvent event) { // 注意：原文件为 Client 事件，但实际通常使用主事件
    // 如需添加数据包对象，先调用 event.createDatapackRegistryObjects(...)

    event.getGenerator().addProvider(
        event.includeServer(), // 通常包含在服务端数据中
        new LootTableProvider(
            event.getGenerator().getPackOutput(),
            // 必需表资源定位符集合（通常模组不验证存在性，传空集）
            Set.of(),
            // 子提供器条目列表
            List.of(...),
            // 注册表访问器
            event.getLookupProvider()
        )
    );
}
```

### `LootTableSubProvider`s

`LootTableSubProvider` 是实际生成逻辑所在。实现接口并重写 `#generate`：
```java
public class MyLootTableSubProvider implements LootTableSubProvider {
    // 通过 lambda 传入参数（见下方），可用于查询其他注册表条目
    public MyLootTableSubProvider(HolderLookup.Provider lookupProvider) {
        // 存储 lookupProvider 到字段
    }

    @Override
    public void generate(BiConsumer<ResourceLocation, LootTable.Builder> consumer) {
        // LootTable.lootTable() 返回战利品表构建器
        consumer.accept(ResourceLocation.fromNamespaceAndPath(ExampleMod.MOD_ID, "example_loot_table"), LootTable.lootTable()
                // 添加表级战利品函数（示例使用数字提供器）
                .apply(SetItemCountFunction.setCount(ConstantValue.exactly(5)))
                // 添加战利品池
                .withPool(LootPool.lootPool()
                        // 添加池级函数（类似上方）
                        .apply(...)
                        // 添加池级条件（示例：仅下雨时执行）
                        .when(WeatherCheck.weather().setRaining(true))
                        // 设置执行次数和奖励次数（均使用数字提供器）
                        .setRolls(UniformGenerator.between(5, 9))
                        .setBonusRolls(ConstantValue.exactly(1))
                        // 添加战利品条目（示例为物品条目）。更多条目类型见下文
                        .add(LootItem.lootTableItem(Items.DIRT))
                )
        );
    }
}
```

将子提供器添加到战利品提供器构造函数：
```java
new LootTableProvider(output, Set.of(), List.of(
        new SubProviderEntry(
                // 子提供器构造器引用（Function<HolderLookup.Provider, ? extends LootTableSubProvider>）
                MyLootTableSubProvider::new,
                // 关联战利品参数集（不确定时用 EMPTY）
                LootContextParamSets.EMPTY
        ),
        // 其他子提供器（如有）
    ), lookupProvider
);
```

### `BlockLootSubProvider`

`BlockLootSubProvider` 是抽象辅助类，提供常见方块战利品表生成方法（如单物品掉落 `#createSingleItemTable`、掉落自身 `#dropSelf`、精准采集专属掉落 `#createSilkTouchOnlyTable`、台阶类掉落 `#createSlabItemTable` 等）。模组使用需额外模板代码：

```java
public class MyBlockLootSubProvider extends BlockLootSubProvider {
    // 若此类是战利品提供器的内部类，构造器可私有
    public MyBlockLootSubProvider(HolderLookup.Provider lookupProvider) {
        // 第一参数为生成战利品表的方块集合（用空集，通过 getKnownBlocks 返回实际集合）
        // 第二参数为特性标志集（默认用 DEFAULT_FLAGS）
        super(Set.of(), FeatureFlags.DEFAULT_FLAGS, lookupProvider);
    }

    // 此 Iterable 用于验证
    @Override
    protected Iterable<Block> getKnownBlocks() {
        // 返回方块注册表条目流
        return MyRegistries.BLOCK_REGISTRY.getEntries()
                .stream()
                .map(e -> (Block) e.value())
                .toList();
    }

    // 实际添加战利品表
    @Override
    protected void generate() {
        // 等效于 add(MyBlocks.EXAMPLE_BLOCK.get(), createSingleItemTable(MyBlocks.EXAMPLE_BLOCK.get()));
        this.dropSelf(MyBlocks.EXAMPLE_BLOCK.get());
        // 添加精准采集专属表
        this.add(MyBlocks.EXAMPLE_SILK_TOUCHABLE_BLOCK.get(),
                this.createSilkTouchOnlyTable(MyBlocks.EXAMPLE_SILK_TOUCHABLE_BLOCK.get()));
        // 其他战利品表
    }
}
```

添加子提供器到战利品提供器：
```java
new LootTableProvider(output, Set.of(), List.of(new SubProviderEntry(
        MyBlockLootTableSubProvider::new,
        LootContextParamSets.BLOCK // 此处用 BLOCK 合理
    )), lookupProvider
);
```

### `EntityLootSubProvider`

类似 `BlockLootSubProvider`，`EntityLootSubProvider` 提供实体战利品表生成辅助方法。类似地需提供已知实体类型的 `Stream<EntityType<?>>`（非 `Iterable<Block>`）。实现类似：

```java
public class MyEntityLootSubProvider extends EntityLootSubProvider {
    public MyEntityLootSubProvider(HolderLookup.Provider lookupProvider) {
        // 不同于方块，不提供已知实体类型集合（原版用自定义检查）
        super(FeatureFlags.DEFAULT_FLAGS, lookupProvider);
    }

    // 此类用 Stream 替代 Iterable
    @Override
    protected Stream<EntityType<?>> getKnownEntityTypes() {
        return MyRegistries.ENTITY_TYPES.getEntries()
                .stream()
                .map(e -> (EntityType<?>) e.value());
    }

    @Override
    protected void generate() {
        this.add(MyEntities.EXAMPLE_ENTITY.get(), LootTable.lootTable());
        // 其他战利品表
    }
}
```

添加子提供器到战利品提供器：
```java
new LootTableProvider(output, Set.of(), List.of(new SubProviderEntry(
        MyEntityLootTableSubProvider::new,
        LootContextParamSets.ENTITY
    )), lookupProvider
);
```

[advancement]: ../advancements.md
[binomial]: https://en.wikipedia.org/wiki/Binomial_distribution
[block]: ../../../blocks/index.md
[conditions]: ../conditions.md
[context]: #战利品上下文
[customentry]: custom.md#自定义战利品条目类型
[customlevelbased]: custom.md#自定义等级基值
[customnumber]: custom.md#自定义数字提供器
[damagesource]: ../damagetypes.md#创建和使用伤害源
[datagen]: ../../index.md#数据生成
[entity]: ../../../entities/index.md
[entry]: #战利品条目
[glm]: glm.md
[lootcondition]: lootconditions
[lootfunction]: lootfunctions
[parameters]: #战利品参数
[raidherogifts]: ../datamaps/builtin.md#neoforgeraid_hero_gifts
[sides]: ../../../concepts/sides.md
[tags]: ../tags.md
