# 进度(`Advancements`)

**进度**(`Advancements`)是玩家可完成的类似任务的目标。进度根据**进度条件**(`advancement criteria`)授予，完成时可运行特定行为。

新进度可通过在命名空间的`advancement`子文件夹中创建 JSON 文件添加。例如，如果我们要为模组 ID `examplemod`添加名为`example_name`的进度，它将位于`data/examplemod/advancement/example_name.json`。进度的 ID 相对于`advancement`目录，因此在本例中为`examplemod:example_name`。可选择任何名称，进度将自动被游戏加载。仅当需要添加新条件或从代码触发特定条件时才需要 Java 代码（见下文）。

## 规范(`Specification`)

进度 JSON 文件可包含以下条目：

- `parent`：此进度的父进度 ID。检测到循环引用将导致加载失败。可选；如果不存在，此进度将被视为根进度。根进度是没有设置父级的进度，它们将是其[进度树][tree]的根节点。
- `display`：包含在进度 GUI 中用于显示进度的多个属性的对象。可选；如果不存在，此进度将不可见，但仍可被触发。
    - `icon`：物品堆栈的[JSON 表示][itemstackjson]。
    - `title`：用作进度标题的[文本组件][text]。
    - `description`：用作进度描述的[文本组件][text]。
    - `frame`：进度的框架类型。接受`challenge`、`goal`和`task`。可选，默认为`task`。
    - `background`：用于树背景的纹理。相对于`textures`目录，即不应包含`textures/`文件夹前缀。可选，默认为缺失纹理。仅对根进度有效。
    - `show_toast`：完成时是否在右上角显示提示(`toast`)。可选，默认为 true。
    - `announce_to_chat`：是否在聊天中宣布进度完成。可选，默认为 true。
    - `hidden`：是否在完成前从进度 GUI 中隐藏此进度及其所有子进度。对根进度本身无效，但仍会隐藏其所有子进度。可选，默认为 false。
- `criteria`：此进度应跟踪的条件映射。每个条件由其映射键标识。Minecraft 添加的条件触发器列表可在`CriteriaTriggers`类中找到，JSON 规范可在[Minecraft Wiki][triggers]上找到。有关实现自定义条件或从代码触发条件，请参见下文。
- `requirements`：确定所需条件的列表的列表。这是一个 AND 在一起的 OR 列表列表，换句话说，每个子列表必须至少有一个匹配的条件。可选，默认为所有条件均为必需。
- `rewards`：表示完成此进度时授予的奖励的对象。可选，对象的所有值也是可选的。
    - `experience`：授予玩家的经验值数量。
    - `recipes`：要解锁的[配方][recipe] ID 列表。
    - `loot`：要滚动并给予玩家的[战利品表][loottable]列表。
    - `function`：要运行的[函数][function]。如果要运行多个函数，请创建一个运行所有其他函数的包装函数。
- `sends_telemetry_event`：确定完成此进度时是否应收集遥测数据。仅在`minecraft`命名空间中实际有效。可选，默认为 false。
- `neoforge:conditions`：NeoForge 添加。必须通过才能使进度加载的[条件][conditions]列表。可选。

### 进度树(`Advancement Trees`)

进度文件可分组到目录中，这告诉游戏创建多个进度标签页。一个进度标签页可包含一个或多个进度树，具体取决于根进度的数量。空的进度标签页会自动隐藏。

:::tip
Minecraft 每个标签页只有一个根进度，且始终将根进度命名为`root`。建议遵循此做法。
:::

## 条件触发器(`Criteria Triggers`)

要解锁进度，必须满足指定的条件。条件通过**触发器**(`triggers`)跟踪，当关联动作发生时从代码执行（例如，当玩家杀死指定[实体][entity]时执行`player_killed_entity`触发器）。每当进度加载到游戏中时，定义的条件会被读取并作为侦听器添加到触发器。当触发器执行时，所有具有相应条件侦听器的进度会重新检查是否完成。如果进度完成，则移除侦听器。

自定义条件触发器由两部分组成：在代码中通过调用`#trigger`激活的**触发器**(`trigger`)，以及定义触发器应授予条件的条件的**实例**(`instance`)。触发器扩展`SimpleCriterionTrigger<T>`，而实例实现`SimpleCriterionTrigger.SimpleInstance`。泛型值`T`表示触发器实例类型。

### `SimpleCriterionTrigger.SimpleInstance`

`SimpleCriterionTrigger.SimpleInstance`表示在`criteria`对象中定义的单个条件。触发器实例负责保存定义的条件，并返回输入是否匹配条件。

条件通常通过构造函数传入。`SimpleCriterionTrigger.SimpleInstance`接口仅需要一个函数`#player`，它返回玩家必须满足的条件作为`Optional<ContextAwarePredicate>`。如果子类是带有此类型`player`参数的记录（如下所示），则自动生成的`#player`方法就足够了。

```java
public record ExampleTriggerInstance(Optional<ContextAwarePredicate> player/*, 其他参数在这里*/)
        implements SimpleCriterionTrigger.SimpleInstance {}
```

通常，触发器实例具有静态辅助方法，这些方法从实例参数构造完整的`Criterion<T>`对象。这允许在数据生成期间轻松创建这些实例，但它们是可选的。

```java
// 在此示例中，EXAMPLE_TRIGGER 是 DeferredHolder<CriterionTrigger<?>, ExampleTrigger>。
// 有关如何注册触发器，请参见下文。
public static Criterion<ExampleTriggerInstance> instance(ContextAwarePredicate player, ItemPredicate item) {
    return EXAMPLE_TRIGGER.get().createCriterion(new ExampleTriggerInstance(Optional.of(player), item));
}
```

最后，应添加一个方法，该方法接收当前数据状态并返回用户是否满足必要条件。玩家的条件已通过`SimpleCriterionTrigger#trigger(ServerPlayer, Predicate)`检查。大多数触发器实例调用此方法`#matches`。

```java
// 假设我们有一个额外的 ItemPredicate 参数。这可以是您需要的任何内容。
// 例如，这也可以是 Predicate<LivingEntity>。
public record ExampleTriggerInstance(Optional<ContextAwarePredicate> player, ItemPredicate predicate)
        implements SimpleCriterionTrigger.SimpleInstance {
    // 此方法对每个实例都是唯一的，因此不会被覆盖。
    // 参数可以是正确匹配所需的任何内容，例如，这也可以是 LivingEntity。
    // 如果除玩家外不需要其他上下文，此方法也可以不带任何参数。
    public boolean matches(ItemStack stack) {
        // 由于 ItemPredicate 匹配堆栈，我们在此使用堆栈作为输入。
        return this.predicate.test(stack);
    }
}
```

### `SimpleCriterionTrigger`

`SimpleCriterionTrigger<T>`实现有两个目的：提供检查触发器实例的方法并在成功时运行附加的侦听器，以及指定用于序列化触发器实例(`T`)的[编解码器][codec]。

首先，我们要添加一个方法，该方法接收我们需要的输入并调用`SimpleCriterionTrigger#trigger`以正确处理检查所有侦听器。大多数触发器实例也命名此方法为`#trigger`。重用我们上面的示例触发器实例，我们的触发器将如下所示：

```java
public class ExampleCriterionTrigger extends SimpleCriterionTrigger<ExampleTriggerInstance> {
    // 此方法对每个触发器都是唯一的，因此不是要覆盖的方法
    public void trigger(ServerPlayer player, ItemStack stack) {
        this.trigger(player,
                // SimpleCriterionTrigger.SimpleInstance 子类中的条件检查方法
                triggerInstance -> triggerInstance.matches(stack)
        );
    }
}
```

触发器必须注册到`Registries.TRIGGER_TYPE`[注册表][registration]：

```java
public static final DeferredRegister<CriterionTrigger<?>> TRIGGER_TYPES =
        DeferredRegister.create(Registries.TRIGGER_TYPE, ExampleMod.MOD_ID);

public static final Supplier<ExampleCriterionTrigger> EXAMPLE_TRIGGER =
        TRIGGER_TYPES.register("example", ExampleCriterionTrigger::new);
```

然后，触发器必须通过覆盖`#codec`定义用于序列化和反序列化触发器实例的[编解码器][codec]。此编解码器通常在实例实现中作为常量创建。

```java
public record ExampleTriggerInstance(Optional<ContextAwarePredicate> player/*, 其他参数在这里*/)
        implements SimpleCriterionTrigger.SimpleInstance {
    public static final Codec<ExampleTriggerInstance> CODEC = ...;

    // ...
}

public class ExampleTrigger extends SimpleCriterionTrigger<ExampleTriggerInstance> {
    @Override
    public Codec<ExampleTriggerInstance> codec() {
        return ExampleTriggerInstance.CODEC;
    }

    // ...
}
```

对于具有`ContextAwarePredicate`和`ItemPredicate`的记录，编解码器可以是：

```java
public static final Codec<ExampleTriggerInstace> CODEC = RecordCodecBuilder.create(instance -> instance.group(
        EntityPredicate.ADVANCEMENT_CODEC.optionalFieldOf("player").forGetter(ExampleTriggerInstance::player),
        ItemPredicate.CODEC.fieldOf("item").forGetter(ExampleTriggerInstance::item)
).apply(instance, ExampleTriggerInstance::new));
```

### 调用条件触发器(`Calling Criterion Triggers`)

每当执行被检查的动作时，应调用由我们的`SimpleCriterionTrigger`子类定义的`#trigger`方法。当然，您也可以调用在`CriteriaTriggers`中找到的原版触发器。

```java
// 在执行动作的代码片段中
// 再次说明，EXAMPLE_TRIGGER 是自定义条件触发器注册实例的提供者
public void performExampleAction(ServerPlayer player, additionalContextParametersHere) {
    // 在此运行执行动作的代码
    EXAMPLE_TRIGGER.get().trigger(player, additionalContextParametersHere);
}
```

## 数据生成(`Data Generation`)

可使用`AdvancementProvider`对进度进行[数据生成][datagen]。`AdvancementProvider`接受`AdvancementSubProviders`列表，这些提供者实际使用`Advancement.Builder`生成进度。

首先，在`GatherDataEvent`之一中创建`AdvancementProvider`实例：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    // 如果添加数据包对象，首先调用 event.createDatapackRegistryObjects(...)

    event.createProvider((output, lookupProvider) -> new AdvancementProvider(
        output, lookupProvider,
        // 在此添加生成器
        List.of(...)
    ));

     // 其他提供者
}
```

现在，下一步是用我们的生成器填充列表。为此，我们可以添加类或 lambda 生成器，然后将它们每个的实例添加到构造函数参数中当前为空的列表中。

```java
// 类示例
public class MyAdvancementGenerator implements AdvancementSubProvider {

    @Override
    public void generate(HolderLookup.Provider registries, Consumer<AdvancementHolder> saver) {
        // 在此生成您的进度。
    }
}

// 方法示例
public class ExampleClass {

    // 匹配 AdvancementSubProvider#generate 提供的参数
    public static void generateExampleAdvancements(HolderLookup.Provider registries, Consumer<AdvancementHolder> saver) {
        // 在此生成您的进度。
    }
}

// 在其中一个 `GatherDataEvent` 中
event.createProvider((output, lookupProvider) -> new AdvancementProvider(
    output, lookupProvider,
    // 在此添加生成器
    List.of(
        // 将我们的生成器实例添加到列表参数。这可以执行任意多次。
        // 拥有多个生成器纯粹是为了组织，所有功能都可以通过单个生成器实现。
        new MyAdvancementGenerator(),
        ExampleClass::generateExampleAdvancements
    )
));
```

要生成进度，您需要使用`Advancement.Builder`：

```java
// 所有方法都遵循构建器模式，意味着可以且鼓励链式调用。
// 为便于解释的可读性，此处不进行链式调用。

// 使用静态方法 #advancement() 创建进度构建器。
// 使用 #advancement() 会自动启用遥测事件。如果不希望如此，
// 可以使用 #recipeAdvancement()，没有其他功能差异。
Advancement.Builder builder = Advancement.Builder.advancement();

// 设置进度的父级。可以使用已生成的另一个进度，
// 或使用静态方法 AdvancementSubProvider#createPlaceholder 创建占位符进度。
builder.parent(AdvancementSubProvider.createPlaceholder("minecraft:story/root"));

// 设置进度的显示属性。这可以是 DisplayInfo 对象，
// 或直接传入值。如果直接传入值，将为您创建 DisplayInfo 对象。
builder.display(
        // 进度图标。可以是 ItemStack 或 ItemLike。
        new ItemStack(Items.GRASS_BLOCK),
        // 进度标题和描述。不要忘记为这些添加翻译！
        Component.translatable("advancements.examplemod.example_advancement.title"),
        Component.translatable("advancements.examplemod.example_advancement.description"),
        // 背景纹理。如果不需要背景纹理（对于非根进度），请使用 null。
        null,
        // 框架类型。有效值为 AdvancementType.TASK、CHALLENGE 或 GOAL。
        AdvancementType.GOAL,
        // 是否显示进度提示。
        true,
        // 是否在聊天中宣布进度。
        true,
        // 进度是否应隐藏。
        false
);

// 进度奖励构建器。可以使用四种奖励类型中的任何一种创建，并且可以使用前缀为 add 的方法添加更多奖励。
// 也可以预先构建，然后生成的 AdvancementRewards 可以在多个进度构建器中重用。
builder.rewards(
    // 或者，使用 addExperience() 添加到现有构建器。
    AdvancementRewards.Builder.experience(100)
    // 或者，使用 loot() 创建新构建器。
    .addLootTable(ResourceKey.create(Registries.LOOT_TABLE, ResourceLocation.fromNamespaceAndPath("minecraft", "chests/igloo")))
    // 或者，使用 recipe() 创建新构建器。
    .addRecipe(ResourceKey.create(Registries.RECIPE, ResourceLocation.fromNamespaceAndPath("minecraft", "iron_ingot")))
    // 或者，使用 function() 创建新构建器。
    .runs(ResourceLocation.fromNamespaceAndPath("examplemod", "example_function"))
);

// 向进度添加具有给定名称的条件。使用相应触发器实例的静态方法。
builder.addCriterion("pickup_dirt", InventoryChangeTrigger.TriggerInstance.hasItems(Items.DIRT));

// 添加需求处理器。Minecraft 原生提供 allOf() 和 anyOf()，更复杂的需求
// 必须手动实现。仅在有两个或更多条件时有效。
builder.requirements(AdvancementRequirements.allOf(List.of("pickup_dirt")));

// 使用给定的资源位置将进度保存到磁盘。这会返回一个 AdvancementHolder，
// 可以存储在变量中并被其他进度构建器用作父级。
builder.save(saver, ResourceLocation.fromNamespaceAndPath("examplemod", "example_advancement"));
```

[codec]: ../../datastorage/codecs.md
[conditions]: conditions.md
[datagen]: ../index.md#data-generation
[entity]: ../../entities/index.md
[function]: https://minecraft.wiki/w/Function_(Java_Edition)
[itemstackjson]: ../../items/index.md#json-representation
[loottable]: loottables/index.md
[recipe]: recipes/index.md
[registration]: ../../concepts/registries.md#methods-for-registering
[root]: #root-advancements
[text]: ../client/i18n.md#components
[tree]: #advancement-trees
[triggers]: https://minecraft.wiki/w/Advancement/JSON_format#List_of_triggers