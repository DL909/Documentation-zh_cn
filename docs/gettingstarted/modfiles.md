# 模组文件

模组文件负责确定哪些模组被打包到你的 JAR 文件中，在“模组”菜单中显示哪些信息，以及你的模组应如何在游戏中加载。

## `gradle.properties`

`gradle.properties` 文件包含了你模组的各种通用属性，例如模组 ID 或模组版本。在构建过程中，Gradle 会读取这些文件中的值，并将它们内联到各个位置，例如 [neoforge.mods.toml][neoforgemodstoml] 文件中。这样，你只需要在一个地方更改值，它们就会自动应用到所有地方。

大多数值也在[MDK 的 `gradle.properties` 文件][mdkgradleproperties]中以注释的形式进行了解释。

| 属性 | 描述 | 示例 |
|---|---|---|
| `org.gradle.jvmargs` | 允许你向 Gradle 传递额外的 JVM 参数。最常见的是，这用于为 Gradle 分配更多/更少的内存。请注意，这是针对 Gradle 本身，而不是 Minecraft。 | `org.gradle.jvmargs=-Xmx3G` |
| `org.gradle.daemon` | Gradle 在构建时是否应使用守护进程。 | `org.gradle.daemon=false` |
| `org.gradle.parallel` | Gradle 是否应该分叉 JVM 来并行执行项目。 | `org.gradle.parallel=false` |
| `org.gradle.caching` | Gradle 是否应该重用先前构建的任务输出。 | `org.gradle.caching=false` |
| `org.gradle.configuration-cache` | Gradle 是否应该重用先前构建的构建配置。 | `org.gradle.configuration-cache=false` |
| `org.gradle.debug` | Gradle 是否设置为调试模式。调试模式主要意味着更多的 Gradle 日志输出。请注意，这是针对 Gradle 本身，而不是 Minecraft。 | `org.gradle.debug=false` |
| `minecraft_version` | 你正在进行模组开发的 Minecraft 版本。必须与 `neo_version` 匹配。 | `minecraft_version=1.20.6` |
| `minecraft_version_range` | 此模组可以使用的 Minecraft 版本范围，以 [Maven 版本范围][mvr] 表示。请注意，[快照版、预发布版和发布候选版][mcversioning]不能保证正确排序，因为它们不遵循 Maven 版本控制。 | `minecraft_version_range=[1.20.6,1.21)` |
| `neo_version` | 你正在进行模组开发的 NeoForge 版本。必须与 `minecraft_version` 匹配。有关 NeoForge 版本控制工作原理的更多信息，请参阅[NeoForge 版本控制][neoversioning]。 | `neo_version=20.6.62` |
| `neo_version_range` | 此模组可以使用的 NeoForge 版本范围，以 [Maven 版本范围][mvr] 表示。 | `neo_version_range=[20.6.62,20.7)` |
| `mod_id` | 参见[模组 ID][modid]。 | `mod_id=examplemod` |
| `mod_name` | 你的模组的对人类可读的显示名称。默认情况下，这只能在模组列表中看到，但是，像 [JEI][jei] 这样的模组也会在物品提示中显著显示模组名称。 | `mod_name=Example Mod` |
| `mod_license` | 你的模组所依据的许可证。建议将其设置为你正在使用的 [SPDX 标识符][spdx] 和/或许可证的链接。你可以访问 https://choosealicense.com/ 来帮助选择你想使用的许可证。 | `mod_license=MIT` |
| `mod_version` | 你的模组的版本，显示在模组列表中。更多信息请参阅[关于版本控制的页面][versioning]。 | `mod_version=1.0` |
| `mod_group_id` | 参见[组 ID][group]。 | `mod_group_id=com.example.examplemod` |
| `mod_authors` | 模组的作者，显示在模组列表中。 | `mod_authors=ExampleModder` |
| `mod_description` | 模组的描述，作为一个多行字符串，显示在模组列表中。可以使用换行符（`\n`）并会被正确替换。 | `mod_description=Example mod description.` |

### 模组 ID

模组 ID 是区分你的模组与其他模组的主要方式。它被用于各种地方，包括作为你模组[注册表][registration]的命名空间，以及你的[资源和数据包][resource]的命名空间。存在两个具有相同 ID 的模组将阻止游戏加载。

因此，你的模组 ID 应该是一些独特且易于记忆的东西。通常，它会是你的模组的显示名称（但为小写），或其某种变体。模组 ID 只能包含小写字母、数字和下划线，并且长度必须在 2 到 64 个字符之间（含）。

:::info
在 `gradle.properties` 文件中更改此属性将自动在所有地方应用此更改，除了你的主模组类中的 [`@Mod` 注解][javafml]。在那里，你需要手动更改它以匹配 `gradle.properties` 文件中的值。
:::

### 组 ID

虽然 `build.gradle` 中的 `group` 属性仅在您计划将模组发布到 Maven 仓库时才是必需的，但通常认为始终正确设置此属性是一种良好实践。这通过 `gradle.properties` 的 `mod_group_id` 属性为您完成。

组 ID 应该设置为你的顶层包。更多信息请参阅[打包][packaging]。

```properties
# 在你的 gradle.properties 文件中
mod_group_id=com.example
```

你的 Java 源文件（`src/main/java`）中的包现在也应符合此结构，并带有一个代表模组 ID 的内部包：

```text
com
- example (在 group 属性中指定的顶层包)
    - mymod (模组 ID)
        - MyMod.java (重命名自 ExampleMod.java)
```

## `neoforge.mods.toml`

`neoforge.mods.toml` 文件位于 `src/main/resources/META-INF/neoforge.mods.toml`，是一个 [TOML][toml] 格式的文件，定义了你模组的元数据。它还包含了关于你的模组应如何加载到游戏中以及在“模组”菜单中显示的额外信息。[MDK 提供的 `neoforge.mods.toml` 文件][mdkneoforgemodstoml]包含了对每个条目的注释解释，这里将更详细地解释它们。

`neoforge.mods.toml` 可以分为三个部分：与模组文件相关联的非模组特定属性；每个模组一个部分的模组属性；以及每个模组或多个模组依赖项一个部分的依赖项配置。与 `neoforge.mods.toml` 文件相关的一些属性是强制性的；强制性属性需要指定一个值，否则将抛出异常。

:::note
在默认的 MDK 中，Gradle 会用 `gradle.properties` 文件中指定的值替换此文件中的各种属性。例如，行 `license="${mod_license}"` 意味着 `license` 字段被 `gradle.properties` 中的 `mod_license` 属性替换。像这样被替换的值应该在 `gradle.properties` 中更改，而不是在这里更改。
:::

### 非模组特定属性

非模组特定属性是与 JAR 文件本身相关联的属性，指示如何加载模组以及任何额外的全局元数据。

| 属性 | 类型 | 默认值 | 描述 | 示例 |
|---|---|---|---|---|
| `modLoader` | 字符串 | `javafml` | 模组使用的语言加载器。可用于支持其他语言结构，例如主文件的 Kotlin 对象，或确定入口点的不同方法，例如接口或方法。NeoForge 提供了 Java 加载器 [`"javafml"`][javafml]。 | `modLoader="javafml"` |
| `loaderVersion` | 字符串 | `""` | 语言加载器的可接受版本范围，以 [Maven 版本范围][mvr] 表示。对于 `javafml`，当前版本为 `1`。如果未指定版本，则可以使用任何版本的模组加载器。 | `loaderVersion="[1,)"` |
| `license` | 字符串 | **强制** | 此 JAR 中的模组所依据的许可证。建议将其设置为你正在使用的 [SPDX 标识符][spdx] 和/或许可证的链接。你可以访问 https://choosealicense.com/ 来帮助选择你想使用的许可证。 | `license="MIT"` |
| `showAsResourcePack` | 布尔值 | `false` | 当为 `true` 时，模组的资源将在“资源包”菜单上显示为单独的资源包，而不是与“模组资源”包合并。 | `showAsResourcePack=true` |
| `showAsDataPack` | 布尔值 | `false` | 当为 `true` 时，模组的数据文件将在“数据包”菜单上显示为单独的数据包，而不是与“模组数据”包合并。 | `showAsDataPack=true` |
| `services` | 数组 | `[]` | 你的模组使用的服务数组。这是作为 NeoForge 实现的 Java 平台模块系统中为模组创建的模块的一部分来使用的。 | `services=["net.neoforged.neoforgespi.language.IModLanguageProvider"]` |
| `properties` | 表 | `{}` | 一个替换属性表。`StringSubstitutor` 使用它来将 `${file.<key>}` 替换为其对应的值。 | `properties={"example"="1.2.3"}` (然后可通过 `${file.example}` 引用) |
| `issueTrackerURL` | 字符串 | _无_ | 一个代表报告和跟踪模组问题的地方的 URL。 | `"https://github.com/neoforged/NeoForge/issues"` |

:::note
`services` 属性在功能上等同于在模块中指定 [`uses` 指令][uses]，这允许[加载给定类型的服务][serviceload]。

或者，它可以在 `src/main/resources/META-INF/services` 文件夹内的服务文件中定义，其中文件名是服务的完全限定名，文件内容是要加载的服务的名称（另见[来自 AtlasViewer 模组的这个例子][atlasviewer]）。
:::

### 模组特定属性

模组特定属性通过 `[[mods]]` 头部与指定的模组绑定。这是一个[表的数组][array]；所有键/值属性都将附加到该模组，直到下一个头部出现。

```toml
# examplemod1 的属性
[[mods]]
modId = "examplemod1"

# examplemod2 的属性
[[mods]]
modId = "examplemod2"
```

| 属性 | 类型 | 默认值 | 描述 | 示例 |
|---|---|---|---|---|
| `modId` | 字符串 | **强制** | 参见[模组 ID][modid]。 | `modId="examplemod"` |
| `namespace` | 字符串 | `modId` 的值 | 模组的覆盖命名空间。也必须是有效的[模组 ID][modid]，但可以额外包含点或破折号。当前未使用。 | `namespace="example"` |
| `version` | 字符串 | `"1"` | 模组的版本，最好是 [Maven 版本控制的变体][versioning]。当设置为 `${file.jarVersion}` 时，它将被替换为 JAR 清单中 `Implementation-Version` 属性的值（在开发环境中显示为 `0.0NONE`）。 | `version="1.20.2-1.0.0"` |
| ` displayName ` | 字符串 | `modId` 的值 | 模组的显示名称。用于在屏幕上表示模组时（例如，模组列表、模组不匹配）。 | `displayName="Example Mod"` |
| `description` | 字符串 | `'''MISSING DESCRIPTION'''` | 显示在模组列表屏幕上的模组描述。建议使用[多行文本字符串][multiline]。此值也是可翻译的，更多信息请参阅[翻译模组元数据][i18n]。 | `description='''This is an example.'''` |
| `logoFile` | 字符串 | _无_ | 用于模组列表屏幕上的图像文件的名称和扩展名。位置必须是从 JAR 或源集的根目录开始的绝对路径（例如，对于主源集是 `src/main/resources`）。有效的文件名字符是小写字母（`a-z`）、数字（`0-9`）、斜杠（`/`）、下划线（`_`）、句点（`.`）和连字符（`-`）。完整的字符集是 `[a-z0-9_.-]`。 | `logoFile="test/example_logo.png"` |
| `logoBlur` | 布尔值 | `true` | 是否使用 `GL_LINEAR*` (true) 或 `GL_NEAREST*` (false) 来渲染 `logoFile`。简单来说，这意味着在尝试缩放徽标时是否应该模糊化徽标。 | `logoBlur=false` |
| `updateJSONURL` | 字符串 | _无_ | 一个指向 JSON 的 URL，[更新检查器][update]使用它来确保你正在玩的模组是最新版本。 | `updateJSONURL="https://example.github.io/update_checker.json"` |
| `features` | 表 | `{}` | 参见[特性](#features)。 | `features={java_version="[17,)"}` |
| `modproperties` | 表 | `{}` | 与此模组关联的键/值表。NeoForge 未使用，但主要供模组使用。 | `modproperties={example="value"}` |
| `modUrl` | 字符串 | _无_ | 模组下载页面的 URL。当前未使用。 | `modUrl="https://neoforged.net/"` |
| `credits` | 字符串 | _无_ | 模组的致谢和鸣谢，显示在模组列表屏幕上。 | `credits="The person over here and there."` |
| `authors` | 字符串 | _无_ | 模组的作者，显示在模组列表屏幕上。 | `authors="Example Person"` |
| `displayURL` | 字符串 | _无_ | 模组展示页面的 URL，显示在模组列表屏幕上。 | `displayURL="https://neoforged.net/"` |
| `enumExtensions` | 字符串 | _无_ | 用于[枚举扩展][enumextension]的 JSON 文件的文件路径。 | `enumExtensions="META_INF/enumextensions.json"` |
| `featureFlags` | 字符串 | _无_ | 用于[特性标志][featureflags]的 JSON 文件的文件路径。 | `featureFlags="META_INF/feature_flags.json"` |

#### 特性

特性系统允许模组要求在加载系统时某些设置、软件或硬件可用。当一个特性不被满足时，模组加载将失败，并通知用户该要求。目前，NeoForge 提供以下特性：

| 特性 | 描述 | 示例 |
|---|---|---|
| `javaVersion` | Java 版本的可接受版本范围，表示为[Maven 版本范围][mvr]。这应该是 Minecraft 支持的版本。 | `features={javaVersion="[17,)"}` |

### 访问转换器特定属性

[访问转换器特定属性][accesstransformer]通过 `[[accessTransformers]]` 头部与指定的访问转换器绑定。这是一个[表的数组][array]；所有键/值属性都将附加到该访问转换器，直到下一个头部出现。访问转换器头部是可选的；但是，如果指定了，所有元素都是强制性的。

| 属性 | 类型 | 默认值 | 描述 | 示例 |
|:---:|:---:|:---:|:---|:---|
| `file` | 字符串 | **强制** | 参见[添加 ATs][accesstransformer]。 | `file="at.cfg"` |

### Mixin 配置属性

[Mixin 配置属性][mixinconfig]通过 `[[mixins]]` 头部与指定的 mixin 配置绑定。这是一个[表的数组][array]；所有键/值属性都将附加到该 mixin 块，直到下一个头部出现。mixin 头部是可选的；但是，如果指定了，所有元素都是强制性的。

| 属性 | 类型 | 默认值 | 描述 | 示例 |
|:---:|:---:|:---:|:---|:---|
| `config` | 字符串 | **强制** | mixin 配置文件的位置。 | `config="examplemod.mixins.json"` |

### 依赖配置

模组可以指定它们的依赖项，NeoForge 会在加载模组之前检查这些依赖项。这些配置使用[表的数组][array] `[[dependencies.<modid>]]` 创建，其中 `modid` 是使用该依赖项的模组的标识符。

| 属性 | 类型 | 默认值 | 描述 | 示例 |
|---|---|---|---|---|
| `modId` | 字符串 | **强制** | 作为依赖项添加的模组的标识符。 | `modId="jei"` |
| `type` | 字符串 | `"required"` | 指定此依赖项的性质：`"required"` 是默认值，如果此依赖项缺失，将阻止模组加载；`"optional"` 在依赖项缺失时不会阻止模组加载，但仍会验证依赖项是否兼容；`"incompatible"` 如果存在此依赖项，则阻止模组加载；`"discouraged"` 如果存在依赖项，仍允许模组加载，但会向用户显示警告。 | `type="incompatible"` |
| `reason` | 字符串 | _无_ | 一个可选的面向用户的消息，用于描述为什么需要此依赖项，或为什么它不兼容。 | `reason="integration"` |
| `versionRange` | 字符串 | `""` | 语言加载器的可接受版本范围，以 [Maven 版本范围][mvr] 表示。空字符串匹配任何版本。 | `versionRange="[1, 2)"` |
| `ordering` | 字符串 | `"NONE"` | 定义模组必须在此依赖项之前（`"BEFORE"`）还是之后（`"AFTER"`）加载。如果顺序无关紧要，则返回 `"NONE"`。 | `ordering="AFTER"` |
| `side` | 字符串 | `"BOTH"` | 依赖项必须存在的[物理侧][sides]：`"CLIENT"`、`"SERVER"` 或 `"BOTH"`。 | `side="CLIENT"` |
| `referralUrl` | 字符串 | _无_ | 依赖项下载页面的 URL。当前未使用。 | `referralUrl="https://library.example.com/"` |

:::danger
两个模组的 `ordering` 可能会因为循环依赖而导致崩溃，例如，如果模组 A 必须在模组 B `"BEFORE"` 加载，而同时模组 B 又必须在模组 A `"BEFORE"` 加载。
:::

## 模组入口点

现在 `neoforge.mods.toml` 已经填写完毕，我们需要为模组提供一个入口点。入口点本质上是执行模组的起点。入口点本身由 `neoforge.mods.toml` 中使用的语言加载器决定。

### `javafml` 和 `@Mod`

`javafml` 是 NeoForge 为 Java 编程语言提供的一个语言加载器。入口点使用带 `@Mod` 注解的公共类来定义。`@Mod` 的值必须包含 `neoforge.mods.toml` 中指定的一个模组 ID。从那里开始，所有的初始化逻辑（例如[注册事件][events]或[添加 `DeferredRegister`s][registration]）都可以在类的构造函数中指定。

主模组类必须只有一个公共构造函数；否则将抛出 `RuntimeException`。构造函数可以有**任何**以下参数，顺序**任意**；它们都不是必需的。但是，不允许有重复的参数。

| 参数类型 | 描述 |
|---|---|
| `IEventBus` | [模组特定的事件总线][modbus]（用于注册、事件等） |
| `ModContainer` | 持有此模组元数据的抽象容器 |
| `FMLModContainer` | 由 `javafml` 定义的实际容器，持有此模组的元数据；是 `ModContainer` 的扩展 |
| `Dist` | 此模组正在加载的[物理侧][sides] |

```java
@Mod("examplemod") // Must match a mod id in the neoforge.mods.toml
public class ExampleMod {
    // Valid constructor, only uses two of the available argument types
    public ExampleMod(IEventBus modBus, ModContainer container) {
        // Initialize logic here
    }
}
```

默认情况下，`@Mod` 注解在两侧[sides]都会加载。这可以通过指定 `dist` 参数来改变：

```java
// Must match a mod id in the neoforge.mods.toml
// This mod class will only be loaded on the physical client
@Mod(value = "examplemod", dist = Dist.CLIENT) 
public class ExampleModClient {
    // Valid constructor
    public ExampleModClient(FMLModContainer container, IEventBus modBus, Dist dist) {
        // Initialize client-only logic here
    }
}
```

:::note
`neoforge.mods.toml` 中的一个条目不需要有对应的 `@Mod` 注解。同样地，`neoforge.mods.toml` 中的一个条目可以有多个 `@Mod` 注解，例如，如果你想分离通用逻辑和仅客户端逻辑。
:::

[accesstransformer]: ../advanced/accesstransformers.md#adding-ats
[array]: https://toml.io/en/v1.0.0#array-of-tables
[atlasviewer]: https://github.com/XFactHD/AtlasViewer/blob/1.20.2/neoforge/src/main/resources/META-INF/services/xfacthd.atlasviewer.platform.services.IPlatformHelper
[events]: ../concepts/events.md
[features]: #特性
[group]: #组-id
[i18n]: ../resources/client/i18n.md#translating-mod-metadata
[javafml]: #javafml-和-mod
[jei]: https://www.curseforge.com/minecraft/mc-mods/jei
[mcversioning]: versioning.md#minecraft
[mdkgradleproperties]: https://github.com/NeoForgeMDKs/MDK-1.21.6-NeoGradle/blob/main/gradle.properties
[mdkneoforgemodstoml]: https://github.com/NeoForgeMDKs/MDK-1.21.6-NeoGradle/blob/main/src/main/resources/META-INF/neoforge.mods.toml
[neoforgemodstoml]: #neoforgemodstoml
[mixinconfig]: https://github.com/SpongePowered/Mixin/wiki/Introduction-to-Mixins---The-Mixin-Environment#mixin-configuration-files
[modbus]: ../concepts/events.md#event-buses
[modid]: #模组-id
[multiline]: https://toml.io/en/v1.0.0#string
[mvr]: https://maven.apache.org/enforcer/enforcer-rules/versionRanges.html
[neoversioning]: versioning.md#neoforge
[packaging]: structuring.md#打包
[registration]: ../concepts/registries.md#deferredregister
[resource]: ../resources/index.md
[serviceload]: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html#load(java.lang.Class)
[sides]: ../concepts/sides.md
[spdx]: https://spdx.org/licenses/
[toml]: https://toml.io/
[update]: ../misc/updatechecker.md
[uses]: https://docs.oracle.com/javase/specs/jls/se21/html/jls-7.html#jls-7.7.3
[versioning]: versioning.md
[enumextension]: ../advanced/extensibleenums.md
[featureflags]: ../advanced/featureflags.md