# Game Tests

Game Tests are a way to run in-game unit tests. The system was designed to be scalable and in parallel to run large numbers of different tests efficiently. Testing object interactions and behaviors are simply a few of the many applications of this framework.

## Creating a Game Test

A standard Game Test follows three basic steps:

1. A structure, or template, is loaded holding the scene on which the interaction or behavior is tested.
1. A method conducts the logic to perform on the scene.
1. The method logic executes. If a successful state is reached, then the test succeeds. Otherwise, the test fails and the result is stored within a lectern adjacent to the scene.

As such, to create a Game Test, there must be an existing template holding the initial start state of the scene and a method which provides the logic of execution.

### The Test Method

A Game Test method is a `Consumer<GameTestHelper>` reference, meaning it takes in a `GameTestHelper` and returns nothing. For a Game Test method to be recognized, it must have a `@GameTest` annotation:

```java
public class ExampleGameTests {
    @GameTest
    public static void exampleTest(GameTestHelper helper) {
        // Do stuff
    }
}
```

The `@GameTest` annotation also contains members which configure how the game test should run.

```java
// In some class
@GameTest(
    setupTicks = 20L, // The test spends 20 ticks to set up for execution
    required = false // The failure is logged but does not affect the execution of the batch
)
public static void exampleConfiguredTest(GameTestHelper helper) {
    // Do stuff
}
```

#### Relative Positioning

All `GameTestHelper` methods translate relative coordinates within the structure template scene to its absolute coordinates using the structure block's current location. To allow for easy conversion between relative and absolute positioning, `GameTestHelper#absolutePos` and `GameTestHelper#relativePos` can be used respectively.

The relative position of a structure template can be obtained in-game by loading the structure via the [test command][test], placing the player at the wanted location, and finally running the `/test pos` command. This will grab the coordinates of the player relative to the closest structure within 200 blocks of the player. The command will export the relative position as a copyable text component in the chat to be used as a final local variable.

:::tip
The local variable generated by `/test pos` can specify its reference name by appending it to the end of the command:

```bash
/test pos <var> # Exports 'final BlockPos <var> = new BlockPos(...);'
```
:::

#### Successful Completion

A Game Test method is responsible for one thing: marking the test was successful on a valid completion. If no success state was achieved before the timeout is reached (as defined by `GameTest#timeoutTicks`), then the test automatically fails.

There are many abstracted methods within `GameTestHelper` which can be used to define a successful state; however, four are extremely important to be aware of.

Method               | Description
:---:                | :---
`#succeed`           | The test is marked as successful.
`#succeedIf`         | The supplied `Runnable` is tested immediately and succeeds if no `GameTestAssertException` is thrown. If the test does not succeed on the immediate tick, then it is marked as a failure.
`#succeedWhen`       | The supplied `Runnable` is tested every tick until timeout and succeeds if the check on one of the ticks does not throw a `GameTestAssertException`.
`#succeedOnTickWhen` | The supplied `Runnable` is tested on the specified tick and will succeed if no `GameTestAssertException` is thrown. If the `Runnable` succeeds on any other tick, then it is marked as a failure.

:::caution
Game Tests are executed every tick until the test is marked as a success. As such, methods which schedule success on a given tick must be careful to always fail on any previous tick.
:::

#### Scheduling Actions

Not all actions will occur when a test begins. Actions can be scheduled to occur at specific times or intervals:

Method           | Description
:---:            | :---
`#runAtTickTime` | The action is ran on the specified tick.
`#runAfterDelay` | The action is ran `x` ticks after the current tick.
`#onEachTick`    | The action is ran every tick.

#### Assertions

At any time during a Game Test, an assertion can be made to check if a given condition is true. There are numerous assertion methods within `GameTestHelper`; however, it simplifies to throwing a `GameTestAssertException` whenever the appropriate state is not met.

### Generated Test Methods

If Game Test methods need to be generated dynamically, a test method generator can be created. These methods take in no parameters and return a collection of `TestFunction`s. For a test method generator to be recognized, it must have a `@GameTestGenerator` annotation:

```java
public class ExampleGameTests {
    @GameTestGenerator
    public static Collection<TestFunction> exampleTests() {
        // Return a collection of TestFunctions
    }
}
```

#### TestFunction

A `TestFunction` is the boxed information held by the `@GameTest` annotation and the method running the test.

:::tip
Any methods annotated using `@GameTest` are translated into a `TestFunction` using `GameTestRegistry#turnMethodIntoTestFunction`. That method can be used as a reference for creating `TestFunction`s without the use of the annotation.
:::

### Batching

Game Tests can be executed in batches instead of registration order. A test can be added to a batch by having the same supplied `GameTest#batch` string.

On its own, batching does not provide anything useful. However, batching can be used to perform setup and teardown states on the current level the tests are running in. This is done by annotating a method with either `@BeforeBatch` for setup or `@AfterBatch` for takedown. The `#batch` methods must match the string supplied to the game test.

Batch methods are `Consumer<ServerLevel>` references, meaning they take in a `ServerLevel` and return nothing:

```java
public class ExampleGameTests {
    @BeforeBatch(batch = "firstBatch")
    public static void beforeTest(ServerLevel level) {
        // Perform setup
    }

    @GameTest(batch = "firstBatch")
    public static void exampleTest2(GameTestHelper helper) {
        // Do stuff
    }
}
```

## Registering a Game Test

A Game Test must be registered to be ran in-game. There are two methods of doing so: via the `@GameTestHolder` annotation or `RegisterGameTestsEvent`. Both registration methods still require the test methods to be annotated with either `@GameTest`, `@GameTestGenerator`, `@BeforeBatch`, or `@AfterBatch`.

### GameTestHolder

The `@GameTestHolder` annotation registers any test methods within the type (class, interface, enum, or record). `@GameTestHolder` contains a single method which has multiple uses. In this instance, the supplied `#value` must be the mod id of the mod; otherwise, the test will not run under default configurations.

```java
@GameTestHolder(MODID)
public class ExampleGameTests {
    // ...
}
```

### RegisterGameTestsEvent

`RegisterGameTestsEvent` can also register either classes or methods using `#register`. The event listener must be [added][event] to the mod event bus. Test methods registered this way must supply their mod id to `GameTest#templateNamespace` on every method annotated with `@GameTest`.

```java
// In some event handler class
@SubscribeEvent // on the mod event bus
public static void registerTests(RegisterGameTestsEvent event) {
    event.register(ExampleGameTests.class);
}

// In ExampleGameTests
@GameTest(templateNamespace = MODID)
public static void exampleTest3(GameTestHelper helper) {
    // Perform setup
}
```

:::note
The value supplied to `GameTestHolder#value` and `GameTest#templateNamespace` can be different from the current mod id. The configuration within the [buildscript][namespaces] would need to be changed.
:::

## Structure Templates

Game Tests are performed within scenes loaded by structures, or templates. All templates define the dimensions of the scene and the initial data (blocks and entities) that will be loaded. The template must be stored as an `.nbt` file within `data/<namespace>/structure`.

:::tip
A structure template can be created and saved using a structure block.
:::

The location of the template is specified by a few factors:

- If the namespace of the template is specified.
- If the class should be prepended to the name of the template.
- If the name of the template is specified.

The namespace of the template is determined by `GameTest#templateNamespace`, then `GameTestHolder#value` if not specified, then `minecraft` if neither is specified.

The simple class name is not prepended to the name of the template if the `@PrefixGameTestTemplate` is applied to a class or method with the test annotations and set to `false`. Otherwise, the simple class name is made lowercase and prepended and followed by a dot before the template name.

The name of the template is determined by `GameTest#template`. If not specified, then the lowercase name of the method is used instead.

```java
// Modid for all structures will be MODID
@GameTestHolder(MODID)
public class ExampleGameTests {

    // Class name is prepended, template name is not specified
    // Template Location at 'modid:examplegametests.exampletest'
    @GameTest
    public static void exampleTest(GameTestHelper helper) { /*...*/ }

    // Class name is not prepended, template name is not specified
    // Template Location at 'modid:exampletest2'
    @PrefixGameTestTemplate(false)
    @GameTest
    public static void exampleTest2(GameTestHelper helper) { /*...*/ }

    // Class name is prepended, template name is specified
    // Template Location at 'modid:examplegametests.test_template'
    @GameTest(template = "test_template")
    public static void exampleTest3(GameTestHelper helper) { /*...*/ }

    // Class name is not prepended, template name is specified
    // Template Location at 'modid:test_template2'
    @PrefixGameTestTemplate(false)
    @GameTest(template = "test_template2")
    public static void exampleTest4(GameTestHelper helper) { /*...*/ }
}
```

## Running Game Tests

Game Tests can be run using the `/test` command. The `test` command is highly configurable; however, only a few are of importance to running tests:

| Subcommand  | Description                                           |
|:-----------:|:------------------------------------------------------|
| `run`       | Runs the specified test: `run <test_name>`.           |
| `runall`    | Runs all available tests.                             |
| `runclosest`| Runs the nearest test to the player within 15 blocks. |
| `runthese`  | Runs tests within 200 blocks of the player.           |
| `runfailed` | Runs all tests that failed in the previous run.       |

:::note
Subcommands follow the test command: `/test <subcommand>`.
:::

## Buildscript Configurations

Game Tests provide additional configuration settings within a buildscript (the `build.gradle` file) to run and integrate into different settings.

### Enabling Other Namespaces

If the buildscript was [setup as recommended][buildscript], then only Game Tests under the current mod id would be enabled. To enable other namespaces to load Game Tests from, a run configuration must set the property `neoforge.enabledGameTestNamespaces` to a string specifying each namespace separated by a comma. If the property is empty or not set, then all namespaces will be loaded.

```gradle
// Inside a run configuration
property 'neoforge.enabledGameTestNamespaces', 'modid1,modid2,modid3'
```

:::caution
There must be no spaces in-between namespaces; otherwise, the namespace will not be loaded correctly.
:::

### Game Test Server Run Configuration

The Game Test Server is a special configuration which runs a build server. The build server returns an exit code of the number of required, failed Game Tests. All failed tests, whether required or optional, are logged. This server can be run using `gradlew runGameTestServer`.

<details>
<summary>Important information on NeoGradle</summary>

:::caution
Due to a quirk in how Gradle works, by default, if a task forces a system exit, the Gradle daemon will be killed, causing the Gradle runner to report a build failure. NeoGradle sets by default a force exit on run tasks such that any subprojects are not executed in sequence. However, as such, the Game Test Server will always fail.

This can be fixed by disabling the force exit on the run configuration using the `#setForceExit` method:

```gradle
// Game Test Server run configuration
gameTestServer {
    // ...
    setForceExit false
}
```
:::
</details>


### Enabling Game Tests in Other Run Configurations

By default, only the `client`, `server`, and `gameTestServer` run configurations have Game Tests enabled. If another run configuration should run Game Tests, then the `neoforge.enableGameTest` property must be set to `true`.

```gradle
// Inside a run configuration
property 'neoforge.enableGameTest', 'true'
```

[test]: #running-game-tests
[namespaces]: #enabling-other-namespaces
[event]: ../concepts/events.md#registering-an-event-handler
[buildscript]: ../gettingstarted/index.md#simple-buildgradle-customizations
