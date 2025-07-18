# 自定义战利品对象

由于战利品表系统的复杂性，存在多个[注册表(`registries`)]共同运作，模组开发者可以利用这些注册表添加更多行为。

所有战利品表相关的注册表都遵循相似的模式。要添加新的注册表条目，通常需要扩展某个类或实现某个包含功能的接口。然后，定义一个用于序列化的[编解码器(`codec`)]，并使用`DeferredRegister`像常规操作一样将其注册到对应的注册表中。这与大多数注册表（例如方块/方块状态和物品/物品堆）采用的"一个基础对象，多个实例"方法一致。

## 自定义战利品条目类型

要创建自定义战利品条目类型，需扩展`LootPoolEntryContainer`或其两个直接子类之一：`LootPoolSingletonContainer`或`CompositeEntryBase`。出于示例目的，假设我们要创建一个返回[实体(`entity`)]掉落物的战利品条目类型（这仅为示例，实际应用中直接引用其他战利品表更理想）。首先创建战利品条目类：

```java
// 继承LootPoolSingletonContainer，因为我们有"有限"的掉落物集合
// 部分代码改编自NestedLootTable
public class EntityLootEntry extends LootPoolSingletonContainer {
    // 用于存储目标实体类型的Holder
    private final Holder<EntityType<?>> entity;

    // 通常使用私有构造函数和静态工厂方法
    // 因为weight, quality, conditions和functions由下方的lambda提供
    private EntityLootEntry(Holder<EntityType<?>> entity, int weight, int quality, List<LootItemCondition> conditions, List<LootItemFunction> functions) {
        // 将lambda提供的参数传递给父类
        super(weight, quality, conditions, functions);
        // 设置自定义值
        this.entity = entity;
    }

    // 静态构建器方法，接收自定义参数并与提供通用值的lambda结合
    public static LootPoolSingletonContainer.Builder<?> entityLoot(Holder<EntityType<?>> entity) {
        // 使用LootPoolSingletonContainer中定义的simpleBuilder()
        return simpleBuilder((weight, quality, conditions, functions) -> new EntityLootEntry(entity, weight, quality, conditions, functions));
    }

    // 核心逻辑：通常通过调用consumer的#accept方法添加物品堆
    // 但此处我们让#getRandomItems代劳
    @Override
    public void createItemStack(Consumer<ItemStack> consumer, LootContext context) {
        // 获取实体的战利品表（不存在则返回空表，无需null检查）
        LootTable table = context.getLevel().reloadableRegistries().getLootTable(entity.value().getDefaultLootTable());
        // 此处使用原始版本（因为原版也这样做）
        // #getRandomItemsRaw会为掷骰结果调用consumer#accept
        table.getRandomItemsRaw(context, consumer);
    }
}
```

接下来为战利品条目创建`MapCodec`：

```java
// 在EntityLootEntry中定义为常量
public static final MapCodec<EntityLootEntry> CODEC = RecordCodecBuilder.mapCodec(inst ->
        // 添加自定义字段
        inst.group(
                        // 引用实体类型ID的值
                        BuiltInRegistries.ENTITY_TYPE.holderByNameCodec().fieldOf("entity").forGetter(e -> e.entity)
                )
                // 添加通用字段：weight, display, conditions和functions
                .and(singletonFields(inst))
                .apply(inst, EntityLootEntry::new)
);
```

然后在[注册(`registries`)]中使用该编解码器：

```java
public static final DeferredRegister<LootPoolEntryType> LOOT_POOL_ENTRY_TYPES =
        DeferredRegister.create(Registries.LOOT_POOL_ENTRY_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootPoolEntryType> ENTITY_LOOT =
        LOOT_POOL_ENTRY_TYPES.register("entity_loot", () -> new LootPoolEntryType(EntityLootEntry.CODEC));
```

最后在战利品条目类中重写`getType()`：

```java
public class EntityLootEntry extends LootPoolSingletonContainer {
    // 其他代码...

    @Override
    public LootPoolEntryType getType() {
        return ENTITY_LOOT.get();
    }
}
```

## 自定义数值提供器

要实现自定义数值提供器(`NumberProvider`)，需实现`NumberProvider`接口。例如创建反转数值符号的提供器：

```java
// 接受另一个数值提供器作为基础
public record InvertedSignProvider(NumberProvider base) implements NumberProvider {
    public static final MapCodec<InvertedSignProvider> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            NumberProviders.CODEC.fieldOf("base").forGetter(InvertedSignProvider::base)
    ).apply(inst, InvertedSignProvider::new));

    // 返回浮点值
    @Override
    public float getFloat(LootContext context) {
        return -this.base.getFloat(context);
    }

    // 返回整数值（可选重写）
    @Override
    public int getInt(LootContext context) {
        return -this.base.getInt(context);
    }

    // 返回此提供器使用的战利品上下文参数集合
    // 此处直接委托给基础提供器
    @Override
    public Set<ContextKey<?>> getReferencedContextParams() {
        return this.base.getReferencedContextParams();
    }
}
```

同样在[注册表(`registries`)]中注册：

```java
public static final DeferredRegister<LootNumberProviderType> LOOT_NUMBER_PROVIDER_TYPES =
        DeferredRegister.create(Registries.LOOT_NUMBER_PROVIDER_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootNumberProviderType> INVERTED_SIGN =
        LOOT_NUMBER_PROVIDER_TYPES.register("inverted_sign", () -> new LootNumberProviderType(InvertedSignProvider.CODEC));
```

并在数值提供器中重写`getType()`：

```java
public record InvertedSignProvider(NumberProvider base) implements NumberProvider {
    // 其他代码...

    @Override
    public LootNumberProviderType getType() {
        return INVERTED_SIGN.get();
    }
}
```

## 自定义基于等级的值

通过实现`LevelBasedValue`接口创建自定义**基于等级的值(`LevelBasedValue`)**。例如反转另一个`LevelBasedValue`的输出：

```java
public record InvertedSignLevelBasedValue(LevelBasedValue base) implements LevelBaseValue {
    public static final MapCodec<InvertedLevelBasedValue> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            LevelBasedValue.CODEC.fieldOf("base").forGetter(InvertedLevelBasedValue::base)
    ).apply(inst, InvertedLevelBasedValue::new));

    // 执行操作
    @Override
    public float calculate(int level) {
        return -this.base.calculate(level);
    }

    // 直接返回编解码器（与NumberProviders不同）
    @Override
    public MapCodec<InvertedLevelBasedValue> codec() {
        return CODEC;
    }
}
```

在[注册表(`registries`)]中直接注册编解码器：

```java
public static final DeferredRegister<MapCodec<? extends LevelBasedValue>> LEVEL_BASED_VALUES =
        DeferredRegister.create(Registries.ENCHANTMENT_LEVEL_BASED_VALUE_TYPE, ExampleMod.MOD_ID);

public static final Supplier<MapCodec<? extends LevelBasedValue>> INVERTED_SIGN =
        LEVEL_BASED_VALUES.register("inverted_sign", () -> InvertedSignLevelBasedValue.CODEC);
```

## 自定义战利品条件

创建实现`LootItemCondition`的战利品条件类。例如仅当击杀怪物的玩家达到特定经验等级时条件才通过：

```java
public record HasXpLevelCondition(int level) implements LootItemCondition {
    // 定义条件所需上下文（玩家需达到的经验等级）
    public static final MapCodec<HasXpLevelCondition> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            Codec.INT.fieldOf("level").forGetter(HasXpLevelCondition::level)
    ).apply(inst, HasXpLevelCondition::new));
    
    // 在此评估条件
    @Override
    public boolean test(LootContext context) {
        @Nullable
        Entity entity = context.getOptionalParameter(LootContextParams.KILLER_ENTITY);
        return entity instanceof Player player && player.experienceLevel >= level; 
    }
    
    // 声明需要的战利品上下文参数（用于验证）
    @Override
    public Set<ContextKey<?>> getReferencedContextParams() {
        return ImmutableSet.of(LootContextParams.KILLER_ENTITY);
    }
}
```

在注册表中注册条件类型：

```java
public static final DeferredRegister<LootItemConditionType> LOOT_CONDITION_TYPES =
        DeferredRegister.create(Registries.LOOT_CONDITION_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootItemConditionType> MIN_XP_LEVEL =
        LOOT_CONDITION_TYPES.register("min_xp_level", () -> new LootItemConditionType(HasXpLevelCondition.CODEC));
```

重写`#getType`方法：

```java
public record HasXpLevelCondition(int level) implements LootItemCondition {
    // 其他代码...

    @Override
    public LootItemConditionType getType() {
        return MIN_XP_LEVEL.get();
    }
}
```

## 自定义战利品函数

创建扩展`LootItemFunction`的类。多数战利品函数不直接扩展`LootItemFunction`，而是扩展`LootItemConditionalFunction`（内置战利品条件应用功能）。例如为物品添加随机指定等级附魔：

```java
// 代码改编自原版EnchantRandomlyFunction
// LootItemConditionalFunction是抽象类，无法使用record
public class RandomEnchantmentWithLevelFunction extends LootItemConditionalFunction {
    // 上下文：可选附魔列表和等级
    private final Optional<HolderSet<Enchantment>> enchantments;
    private final int level;
    // 编解码器
    public static final MapCodec<RandomEnchantmentWithLevelFunction> CODEC =
            // #commonFields添加conditions字段
            RecordCodecBuilder.mapCodec(inst -> commonFields(inst).and(inst.group(
                    RegistryCodecs.homogeneousList(Registries.ENCHANTMENT).optionalFieldOf("enchantments").forGetter(e -> e.enchantments),
                    Codec.INT.fieldOf("level").forGetter(e -> e.level)
            ).apply(inst, RandomEnchantmentWithLevelFunction::new));
    
    public RandomEnchantmentWithLevelFunction(List<LootItemCondition> conditions, Optional<HolderSet<Enchantment>> enchantments, int level) {
        super(conditions);
        this.enchantments = enchantments;
        this.level = level;
    }
    
    // 执行附魔逻辑（大部分复制自EnchantRandomlyFunction#run）
    @Override
    public ItemStack run(ItemStack stack, LootContext context) {
        RandomSource random = context.getRandom();
        List<Holder<Enchantment>> stream = this.enchantments
                .map(HolderSet::stream)
                .orElseGet(() -> context.getLevel().registryAccess().lookupOrThrow(Registries.ENCHANTMENT).listElements().map(Function.identity()))
                .filter(e -> e.value().canEnchant(stack))
                .toList();
        Optional<Holder<Enchantment>> optional = Util.getRandomSafe(list, random);
        if (optional.isEmpty()) {
            LOGGER.warn("Couldn't find a compatible enchantment for {}", stack);
        } else {
            Holder<Enchantment> enchantment = optional.get();
            if (stack.is(Items.BOOK)) {
                stack = new ItemStack(Items.ENCHANTED_BOOK);
            }
            stack.enchant(enchantment, Mth.nextInt(random, enchantment.value().getMinLevel(), enchantment.value().getMaxLevel()));
        }
        return stack;
    }
}
```

注册函数类型：

```java
public static final DeferredRegister<LootItemFunctionType<?>> LOOT_FUNCTION_TYPES =
        DeferredRegister.create(Registries.LOOT_FUNCTION_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootItemFunctionType<RandomEnchantmentWithLevelFunction>> RANDOM_ENCHANTMENT_WITH_LEVEL =
        LOOT_FUNCTION_TYPES.register("random_enchantment_with_level", () -> new LootItemFunctionType(RandomEnchantmentWithLevelFunction.CODEC));
```

重写`#getType`方法：

```java
public class RandomEnchantmentWithLevelFunction extends LootItemConditionalFunction {
    // 其他代码...

    @Override
    public LootItemFunctionType<?> getType() {
        return RANDOM_ENCHANTMENT_WITH_LEVEL.get();
    }
}
```

[编解码器(`codec`)]: ../../../datastorage/codecs.md  
[实体(`entity`)]: ../../../entities/index.md  
[注册表(`registries`)]: ../../../concepts/registries.md#methods-for-registering  
