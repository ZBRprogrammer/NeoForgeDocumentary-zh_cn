# **访问转换器**(`Access Transformers`)

**访问转换器**(`Access Transformers`，简称 AT) 允许扩大可见性范围并修改类、方法和字段的 `final` 标志。它们允许模组开发者访问和修改其控制范围之外类中原本不可访问的成员。

[规范文档][specs] 可在 NeoForged GitHub 上查看。

## 添加 AT

在您的模组项目中添加**访问转换器**(`Access Transformer`)只需在 `build.gradle` 中添加一行代码：

访问转换器需在 `build.gradle` 中声明。AT 文件可指定在任何位置，只要它们在编译时被复制到 `resources` 输出目录即可。

```groovy
// 在 build.gradle 中：
// 此代码块也用于指定您的映射版本
minecraft {
    accessTransformers {
        file('src/main/resources/META-INF/accesstransformer.cfg')
    }
}
```

默认情况下，NeoForge 会搜索 `META-INF/accesstransformer.cfg`。如果 `build.gradle` 在其他位置指定了访问转换器，则需要在 `neoforge.mods.toml` 中定义它们的位置：

```toml
# 在 neoforge.mods.toml 中：
[[accessTransformers]]
## 文件路径相对于资源的输出目录，或编译后 jar 内的根路径
## 'resources' 目录表示资源的根输出目录
file="META-INF/accesstransformer.cfg"
```

此外，可以指定多个 AT 文件，它们将按顺序应用。这对于具有多个包的大型模组非常有用。

```groovy
// 在 build.gradle 中：
minecraft {
    accessTransformers {
        file('src/main/resources/accesstransformer_main.cfg')
        file('src/additions/resources/accesstransformer_additions.cfg')
    }
}
```

```toml
# 在 neoforge.mods.toml 中
[[accessTransformers]]
file="accesstransformer_main.cfg"

[[accessTransformers]]
file="accesstransformer_additions.cfg"
```

添加或修改任何**访问转换器**(`Access Transformer`)后，必须刷新 Gradle 项目才能使转换生效。

## **访问转换器**(`Access Transformer`)规范

### 注释

`#` 之后直到行尾的所有文本将被视为注释，不会被解析。

### **访问修饰符**(`Access Modifiers`)

**访问修饰符**(`Access modifiers`)指定将给定目标转换到何种新的成员可见性。按可见性从高到低排序：

- `public` - 对包内外所有类可见
- `protected` - 仅对包内类和子类可见
- `default` - 仅对包内类可见
- `private` - 仅对类内部可见

特殊修饰符 `+f` 和 `-f` 可附加到上述修饰符后，分别用于添加或移除 `final` 修饰符（该修饰符在应用时会阻止类继承、方法重写或字段修改）。

:::danger
指令仅修改它们直接引用的方法；任何重写方法不会被访问转换。建议确保转换后的方法没有未转换且限制可见性的重写方法，否则会导致 JVM 抛出错误。

可安全转换的方法示例包括：`private` 方法、`final` 方法（或 `final` 类中的方法）以及 `static` 方法。
:::

### 目标与指令

#### 类

标记类：

```
<access modifier> <fully qualified class name>
```

内部类通过将外部类的完全限定名和内部类名用 `$` 连接表示。

#### 字段

标记字段：

```
<access modifier> <fully qualified class name> <field name>
```

#### 方法

标记方法需要使用特殊语法表示方法参数和返回类型：

```
<access modifier> <fully qualified class name> <method name>(<parameter types>)<return type>
```

##### 指定类型

也称为“**描述符**(`descriptors`)”：更多技术细节请参阅 [Java 虚拟机规范, SE 21, 第 4.3.2 和 4.3.3 节][jvmdescriptors]。

- `B` - `byte`，有符号字节
- `C` - `char`，UTF-16 中的 Unicode 字符码点
- `D` - `double`，双精度浮点值
- `F` - `float`，单精度浮点值
- `I` - `integer`，32 位整数
- `J` - `long`，64 位整数
- `S` - `short`，有符号短整数
- `Z` - `boolean`，`true` 或 `false` 值
- `[` - 引用数组的一个维度
    - 示例：`[[S` 表示 `short[][]`
- `L<class name>;` - 引用一个引用类型
    - 示例：`Ljava/lang/String;` 表示 `java.lang.String` 引用类型（注意使用斜杠而非点号）
- `(` - 引用方法描述符，参数应在此处提供，若无参数则为空
    - 示例：`<method>(I)Z` 表示接受一个整数参数并返回布尔值的方法
- `V` - 表示方法无返回值，只能用于方法描述符末尾
    - 示例：`<method>()V` 表示无参数且无返回值的方法

### 示例

```
# 将 Crypt 中的 ByteArrayToKeyFunction 接口设为 public
public net.minecraft.util.Crypt$ByteArrayToKeyFunction

# 将 MinecraftServer 中的 'random' 设为 protected 并移除 final 修饰符
protected-f net.minecraft.server.MinecraftServer random

# 将 Util 中的 'makeExecutor' 方法设为 public，
# 接受一个 String 并返回 TracingExecutor
public net.minecraft.Util makeExecutor(Ljava/lang/String;)Lnet/minecraft/TracingExecutor;

# 将 UUIDUtil 中的 'leastMostToIntArray' 方法设为 public，
# 接受两个 long 并返回 int[]
public net.minecraft.core.UUIDUtil leastMostToIntArray(JJ)[I
```

[specs]: https://github.com/NeoForged/AccessTransformers/blob/main/FMLAT.md
[jvmdescriptors]: https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.3.2