# 内置配方类型（`Built-In Recipe Types`）
《我的世界》（`Minecraft`）提供了多种现成的**配方类型**（`recipe types`）和**序列化器**（`serializers`）供你使用。本文将介绍每种配方类型以及生成它们的方法。

## 合成（`Crafting`）
合成配方通常在工作台（`crafting tables`）、工匠台（`crafters`）或模组化的工作台、机器中制作。它们的配方类型为`minecraft:crafting`。

### 有形状合成（`Shaped Crafting`）
一些最重要的配方——比如工作台、木棍或大多数工具——都是通过有形状配方制作的。这些配方由物品必须放入的合成图案或形状（因此称为“有形状”）来定义。让我们看看示例是什么样的：
```json5
{
    "type": "minecraft:crafting_shaped",
    "category": "equipment",
    "key": {
        "#": "minecraft:stick",
        "X": "minecraft:iron_ingot"
    },
    "pattern": [
        "XXX",
        " # ",
        " # "
    ],
    "result": {
        "count": 1,
        "id": "minecraft:iron_pickaxe"
    }
}
```

让我们逐行解读：
- `type`：这是有形状配方序列化器的ID，即`minecraft:crafting_shaped`。
- `category`：这个可选字段定义了合成书中的`CraftingBookCategory`。
- `key`和`pattern`：这两个一起定义了物品必须放入合成网格的方式。
    - 图案定义了最多三行、每行最多三个字符的字符串，以此确定形状。所有行的长度必须相同，也就是说，图案必须形成矩形。空格可用于表示应保持为空的格子。
    - `key`将图案中使用的字符与[配料][ingredient]相关联。在上面的示例中，图案中的所有`X`都必须是铁锭，所有`#`都必须是木棍。
- `result`：配方的结果。这是[物品堆的JSON表示形式][itemjson]。
- 示例中未显示的是`group`键。这个可选的字符串属性会在配方书中创建一个组。同一组中的配方在配方书中会显示为一个。
- 示例中未显示的是`show_notification`。这个可选的布尔值，当设为`false`时，会禁用首次使用或解锁时右上角显示的提示。

接下来，让我们看看如何在`RecipeProvider#buildRecipes`中生成这个配方：
```java
// 我们使用构建器模式，因此不会创建变量。通过调用
// ShapedRecipeBuilder#shaped并传入配方类别（可在RecipeCategory枚举中找到）
// 以及结果物品、结果物品和数量，或者结果物品堆来创建一个新的构建器。
ShapedRecipeBuilder.shaped(this.registries.lookupOrThrow(Registries.ITEM), RecipeCategory.TOOLS, Items.IRON_PICKAXE)
        // 创建图案的行。每次调用#pattern都会添加新的一行。
        // 图案会经过验证，即其形状会被检查。
        .pattern("XXX")
        .pattern(" # ")
        .pattern(" # ")
        // 为图案中的字符创建对应关系。图案中使用的所有非空格字符都必须定义。
        // 这可以接受配料、物品标签键（TagKey<Item>）或物品类似物（ItemLikes），即物品或方块。
        .define('X', Items.IRON_INGOT)
        .define('#', Items.STICK)
        // 创建配方进度。虽然后台系统不强制要求，但
        // 如果省略这一步，配方构建器会崩溃。第一个参数是进度名称，
        // 第二个参数是条件。通常，你会使用has()快捷方式来设置条件。
        // 可以通过多次调用#unlockedBy来添加多个进度要求。
        .unlockedBy("has_iron_ingot", this.has(Items.IRON_INGOT))
        // 将配方存储到传入的RecipeOutput中，以便写入磁盘。
        // 如果你想为配方添加条件，可以在输出上进行设置。
        .save(this.output);
```

此外，你可以调用`#group`和`#showNotification`分别设置配方书组和切换提示弹窗。

### 无形状合成（`Shapeless Crafting`）
与有形状合成配方不同，无形状合成配方不关心配料放入的顺序。因此，它没有图案和键，而是只有一个配料列表：
```json5
{
    "type": "minecraft:crafting_shapeless",
    "category": "misc",
    "ingredients": [
        "minecraft:brown_mushroom",
        "minecraft:red_mushroom",
        "minecraft:bowl"
    ],
    "result": {
        "count": 1,
        "id": "minecraft:mushroom_stew"
    }
}
```

和之前一样，我们逐行解读：
- `type`：这是无形状配方序列化器的ID，即`minecraft:crafting_shapeless`。
- `category`：这个可选字段定义了合成书中的类别。
- `ingredients`：[配料][ingredient]的列表。为了配方查看，列表的顺序在代码中会保留，但配方本身接受任意顺序的配料。
- `result`：配方的结果。这是[物品堆的JSON表示形式][itemjson]。
- 示例中未显示的是`group`键。这个可选的字符串属性会在配方书中创建一个组。同一组中的配方在配方书中会显示为一个。

接下来，让我们看看如何在`RecipeProvider#buildRecipes`中生成这个配方：
```java
// 我们使用构建器模式，因此不会创建变量。通过调用
// ShapelessRecipeBuilder#shapeless并传入配方类别（可在RecipeCategory枚举中找到）
// 以及结果物品、结果物品和数量，或者结果物品堆来创建一个新的构建器。
ShapelessRecipeBuilder.shapeless(this.registries.lookupOrThrow(Registries.ITEM), RecipeCategory.MISC, Items.MUSHROOM_STEW)
        // 添加配方配料。这可以接受配料、物品标签键（TagKey<Item>）或物品类似物（ItemLikes）。
        // 也有一些重载方法可以额外接受数量，用于多次添加相同的配料。
        .requires(Blocks.BROWN_MUSHROOM)
        .requires(Blocks.RED_MUSHROOM)
        .requires(Items.BOWL)
        // 创建配方进度。虽然后台系统不强制要求，但
        // 如果省略这一步，配方构建器会崩溃。第一个参数是进度名称，
        // 第二个参数是条件。通常，你会使用has()快捷方式来设置条件。
        // 可以通过多次调用#unlockedBy来添加多个进度要求。
        .unlockedBy("has_mushroom_stew", this.has(Items.MUSHROOM_STEW))
        .unlockedBy("has_bowl", this.has(Items.BOWL))
        .unlockedBy("has_brown_mushroom", this.has(Blocks.BROWN_MUSHROOM))
        .unlockedBy("has_red_mushroom", this.has(Blocks.RED_MUSHROOM))
        // 将配方存储到传入的RecipeOutput中，以便写入磁盘。
        // 如果你想为配方添加条件，可以在输出上进行设置。
        .save(this.output);
```

此外，你可以调用`#group`来设置配方书组。

:::info
单一物品配方（例如存储块拆解）应遵循 vanilla 标准，采用无形状配方。
:::

### 转化合成（`Transmute Crafting`）
转化配方是一种特殊的单一物品合成配方，其中输入堆的数据组件会完全复制到结果堆中。转化通常发生在两个不同物品之间，其中一个是另一个的染色版本。例如：
```json5
{
    "type": "minecraft:crafting_transmute",
    "category": "misc",
    "group": "shulker_box_dye",
    "input": "#minecraft:shulker_boxes",
    "material": "minecraft:blue_dye",
    "result": {
        "id": "minecraft:blue_shulker_box"
    }
}
```

和之前一样，我们逐行解读：
- `type`：这是无形状配方序列化器的ID，即`minecraft:crafting_transmute`。
- `category`：这个可选字段定义了合成书中的类别。
- `group`：这个可选的字符串属性会在配方书中创建一个组。同一组中的配方在配方书中会显示为一个，这对于转化配方通常是有意义的。
- `input`：要转化的[配料][ingredient]。
- `material`：将堆转化为结果的[配料][ingredient]。
- `result`：配方的结果。这是[物品堆的JSON表示形式][itemjson]。

接下来，让我们看看如何在`RecipeProvider#buildRecipes`中生成这个配方：
```java
// 我们使用构建器模式，因此不会创建变量。通过调用
// TransmuteRecipeBuilder#transmute并传入配方类别（可在RecipeCategory枚举中找到）、
// 输入配料、材料配料和结果物品来创建一个新的构建器。
TransmuteRecipeBuilder.transmute(RecipeCategory.MISC, this.tag(ItemTags.SHULKER_BOXES),
    Ingredient.of(DyeItem.byColor(DyeColor.BLUE)), ShulkerBoxBlock.getBlockByColor(DyeColor.BLUE).asItem())
        // 设置配方在配方书中显示的组。
        .group("shulker_box_dye")
        // 创建配方进度。虽然后台系统不强制要求，但
        // 如果省略这一步，配方构建器会崩溃。第一个参数是进度名称，
        // 第二个参数是条件。通常，你会使用has()快捷方式来设置条件。
        // 可以通过多次调用#unlockedBy来添加多个进度要求。
        .unlockedBy("has_shulker_box", this.has(ItemTags.SHULKER_BOXES))
        // 将配方存储到传入的RecipeOutput中，以便写入磁盘。
        // 如果你想为配方添加条件，可以在输出上进行设置。
        .save(this.output);
```

### 特殊合成（`Special Crafting`）
在某些情况下，输出必须根据输入动态生成。大多数时候，这是为了通过从输入堆计算值来在输出上设置数据组件。这些配方通常只指定类型，其他所有内容都硬编码。例如：
```json5
{
    "type": "minecraft:crafting_special_armordye"
}
```

这个用于皮革 armor 染色的配方只指定了类型，其他所有内容都硬编码——最值得注意的是颜色计算，这很难用JSON表达。《我的世界》（`Minecraft`）的大多数特殊合成配方都以`crafting_special_`为前缀，但这一做法并非必须遵循。

在`RecipeProvider#buildRecipes`中生成这个配方的方式如下：
```java
// #special的参数是一个函数（Function<CraftingBookCategory, Recipe<?>>）。
// 所有vanilla特殊配方都为此使用带有一个CraftingBookCategory参数的构造函数。
SpecialRecipeBuilder.special(ArmorDyeRecipe::new)
        // 这个#save的重载允许我们指定名称。它也可以用于有形状或无形状构建器。
        .save(this.output, "armor_dye");
```

Vanilla 提供了以下特殊合成序列化器（模组可能会添加更多）：
- `minecraft:crafting_special_armordye`：用于皮革 armor 和其他可染色物品的染色。
- `minecraft:crafting_special_bannerduplicate`：用于复制旗帜。
- `minecraft:crafting_special_bookcloning`：用于复制成书。这会将结果书的生成属性增加1。
- `minecraft:crafting_special_firework_rocket`：用于制作烟花火箭。
- `minecraft:crafting_special_firework_star`：用于制作烟花之星。
- `minecraft:crafting_special_firework_star_fade`：用于为烟花之星添加渐变色。
- `minecraft:crafting_special_mapcloning`：用于复制已填充的地图。也适用于宝藏地图。
- `minecraft:crafting_special_mapextending`：用于扩展已填充的地图。
- `minecraft:crafting_special_repairitem`：用于将两个破损物品修复为一个。
- `minecraft:crafting_special_shielddecoration`：用于将旗帜应用到盾牌上。
- `minecraft:crafting_special_shulkerboxcoloring`：用于给潜影盒染色，同时保留其内容。
- `minecraft:crafting_special_tippedarrow`：用于根据输入药水制作药箭。
- `minecraft:crafting_decorated_pot`：用于用陶片制作装饰性陶罐。

## 熔炉类配方（`Furnace-like Recipes`）
第二重要的配方组是通过冶炼或类似过程制作的配方。所有在熔炉（类型为`minecraft:smelting`）、烟熏炉（`minecraft:smoking`）、高炉（`minecraft:blasting`）和营火（`minecraft:campfire_cooking`）中制作的配方都使用相同的格式：
```json5
{
    "type": "minecraft:smelting",
    "category": "food",
    "cookingtime": 200,
    "experience": 0.1,
    "ingredient": {
        "item": "minecraft:kelp"
    },
    "result": {
        "id": "minecraft:dried_kelp"
    }
}
```

让我们逐行解读：
- `type`：这是配方序列化器的ID，即`minecraft:smelting`。根据你要制作的熔炉类配方类型，这个ID可能会不同。
- `category`：这个可选字段定义了合成书中的类别。
- `cookingtime`：这个字段确定配方需要处理的时间，以刻（`ticks`）为单位。所有vanilla熔炉配方使用200刻，烟熏炉和高炉使用100刻，营火使用600刻。不过，这个值可以是你想要的任何值。
- `experience`：确定制作这个配方时奖励的经验量。这个字段是可选的，如果省略，将不会奖励经验。
- `ingredient`：配方的输入[配料][ingredient]。
- `result`：配方的结果。这是[物品堆的JSON表示形式][itemjson]。

这些配方的数据生成在`RecipeProvider#buildRecipes`中如下所示：
```java
// 对于烟熏配方使用#smoking，对于高炉配方使用#blasting，对于营火配方使用#campfireCooking。
// 除此之外，所有这些构建器的工作方式都相同。
SimpleCookingRecipeBuilder.smelting(
        // 我们的输入配料。
        Ingredient.of(Items.KELP),
        // 我们的配方类别。
        RecipeCategory.FOOD,
        // 我们的结果物品。也可以是物品堆（ItemStack）。
        Items.DRIED_KELP,
        // 我们的经验奖励。
        0.1f,
        // 我们的烹饪时间。
        200
)
        // 配方进度，和上面的合成配方一样。
        .unlockedBy("has_kelp", this.has(Blocks.KELP))
        // 这个#save的重载允许我们指定名称。
        .save(this.output, "dried_kelp_smelting");
```

:::info
这些配方的配方类型与其配方序列化器相同，即熔炉使用`minecraft:smelting`，烟熏炉使用`minecraft:smoking`，依此类推。
:::

## 切石（`Stonecutting`）
切石机配方使用`minecraft:stonecutting`配方类型。它们非常简单，只包含类型、输入和输出：
```json5
{
    "type": "minecraft:stonecutting",
    "ingredient": "minecraft:andesite",
    "result": {
        "count": 2,
        "id": "minecraft:andesite_slab"
    }
}
```

`type`定义了配方序列化器（`minecraft:stonecutting`）。配料是一种[配料][ingredient]，结果是基本的[物品堆JSON][itemjson]。和合成配方一样，它们也可以可选地指定一个`group`用于在配方书中分组。

数据生成在`RecipeProvider#buildRecipes`中也很简单：
```java
SingleItemRecipeBuilder.stonecutting(Ingredient.of(Items.ANDESITE), RecipeCategory.BUILDING_BLOCKS, Items.ANDESITE_SLAB, 2)
        .unlockedBy("has_andesite", this.has(Items.ANDESITE))
        .save(this.output, "andesite_slab_from_andesite_stonecutting");
```

请注意，单一物品配方构建器不支持实际的物品堆（`ItemStack`）结果，因此不支持带有数据组件的结果。不过，配方编解码器（`codec`）支持它们，所以如果需要此功能，需要实现自定义构建器。

## 锻造（`Smithing`）
锻造台支持两种不同的配方序列化器。一种用于将输入转化为输出，复制输入的组件（如附魔），另一种用于将组件应用到输入。两者都使用`minecraft:smithing`配方类型，并且需要三个输入，分别称为基础物品（`base`）、模板（`template`）和添加物品（`addition`）。

### 转化锻造（`Transform Smithing`）
这种配方序列化器用于将两个输入物品转化为一个，保留第一个输入的数据组件。Vanilla主要将其用于下界合金装备，但任何物品都可以使用：
```json5
{
    "type": "minecraft:smithing_transform",
    "addition": "#minecraft:netherite_tool_materials",
    "base": "minecraft:diamond_axe",
    "result": {
        "id": "minecraft:netherite_axe"
    },
    "template": "minecraft:netherite_upgrade_smithing_template"
}
```

让我们逐行解读：
- `type`：这是配方序列化器的ID，即`minecraft:smithing_transform`。
- `base`：配方的基础[配料][ingredient]。通常，这是某种装备。
- `template`：配方的模板[配料][ingredient]。通常，这是一个锻造模板。
- `addition`：配方的添加[配料][ingredient]。通常，这是某种材料，例如下界合金锭。
- `result`：配方的结果。这是[物品堆的JSON表示形式][itemjson]。

在数据生成期间，在`RecipeProvider#buildRecipes`中调用`SmithingTransformRecipeBuilder#smithing`来添加你的配方：
```java
SmithingTransformRecipeBuilder.smithing(
        // 模板配料。
        Ingredient.of(Items.NETHERITE_UPGRADE_SMITHING_TEMPLATE),
        // 基础配料。
        Ingredient.of(Items.DIAMOND_AXE),
        // 添加配料。
        this.tag(ItemTags.NETHERITE_TOOL_MATERIALS),
        // 配方书类别。
        RecipeCategory.TOOLS,
        // 结果物品。请注意，虽然配方编解码器在此处接受物品堆，但构建器不接受。
        // 如果你需要物品堆输出，你需要使用自己的构建器。
        Items.NETHERITE_AXE
)
        // 配方进度，和上面的其他配方一样。
        .unlocks("has_netherite_ingot", this.has(ItemTags.NETHERITE_TOOL_MATERIALS))
        // 这个#save的重载允许我们指定名称。
        .save(this.output, "netherite_axe_smithing");
```

### 修剪锻造（`Trim Smithing`）
修剪锻造是将 armor 装饰应用到 armor 上的过程：
```json5
{
    "type": "minecraft:smithing_trim",
    "addition": "#minecraft:trim_materials",
    "base": "#minecraft:trimmable_armor",
    "pattern": "minecraft:spire",
    "template": "minecraft:bolt_armor_trim_smithing_template"
}
```

同样，让我们逐行解读：
- `type`：这是配方序列化器的ID，即`minecraft:smithing_trim`。
- `base`：配方的基础[配料][ingredient]。所有vanilla使用场景都在此处使用`minecraft:trimmable_armor`标签。
- `template`：配方的模板[配料][ingredient]。所有vanilla使用场景都在此处使用锻造修剪模板。
- `addition`：配方的添加[配料][ingredient]。所有vanilla使用场景都在此处使用`minecraft:trim_materials`标签。
- `pattern`：应用到基础配料上的修剪图案。

这种配方序列化器明显缺少结果字段。这是因为它使用基础输入，并将模板和添加物品“应用”到其上，也就是说，它根据其他输入设置基础的组件，并将该操作的结果用作配方的结果。

在数据生成期间，在`RecipeProvider#buildRecipes`中调用`SmithingTrimRecipeBuilder#smithingTrim`来添加你的配方：
```java
SmithingTrimRecipeBuilder.smithingTrim(
        // 模板配料。
        Ingredient.of(Items.BOLT_ARMOR_TRIM_SMITHING_TEMPLATE),
        // 基础配料。
        this.tag(ItemTags.TRIMMABLE_ARMOR),
        // 添加配料。
        this.tag(ItemTags.TRIM_MATERIALS),
        // 应用到基础上的修剪图案。
        this.registries.lookupOrThrow(Registries.TRIM_PATTERN).getOrThrow(TrimPatterns.SPIRE),
        // 配方书类别。
        RecipeCategory.MISC
)
        // 配方进度，和上面的其他配方一样。
        .unlocks("has_smithing_trim_template", this.has(Items.BOLT_ARMOR_TRIM_SMITHING_TEMPLATE))
        // 这个#save的重载允许我们指定名称。是的，这个名称是从vanilla复制的。
        .save(this.output, "bolt_armor_trim_smithing_template_smithing_trim");
```

[ingredient]: ingredients.md
[itemjson]: ../../../items/index.md#json-representation