# 组织你的模组结构

结构化的模组有利于维护、贡献代码，并能更清晰地理解底层代码库。下面列出了一些来自 Java、Minecraft 和 NeoForge 的建议。

:::note
你不必遵循下面的建议；你可以按照任何你认为合适的方式来组织你的模组。然而，我们仍然强烈建议你这样做。
:::

## 打包

在组织你的模组结构时，选择一个独特的顶层包结构。许多程序员会为不同的类、接口等使用相同的名称。Java 允许类具有相同的名称，只要它们位于不同的包中。因此，如果两个类在同一个包中具有相同的名称，只有一个类会被加载，这很可能会导致游戏崩溃。

```
a.jar
    - com.example.ExampleClass
b.jar
    - com.example.ExampleClass // This class will not normally be loaded
```

这在加载模块时尤为重要。如果在不同模块中存在两个同名包，其中包含类文件，这将导致模组加载器在启动时崩溃，因为模组模块会导出到游戏和其他模组中。

```
module A
    - package X
        - class I
        - class J
module B
    - package X // This package will cause the mod loader to crash, as there already is a module with package X being exported
        - class R
        - class S
        - class T
```

因此，你的顶层包应该是你拥有的东西：一个域名、一个电子邮件地址、一个网站（或其子域名）等。它甚至可以是你的名字或用户名，只要你能保证它在预期目标中是唯一可识别的。此外，顶层包也应该与你的[组 ID][group]匹配。

| 类型 | 值 | 顶层包 |
|:---:|:---:|:---:|
| 域名 | example.com | `com.example` |
| 子域名 | example.github.io | `io.github.example` |
| 电子邮件 | example@gmail.com | `com.gmail.example` |

下一级包则应该是你的模组 ID（例如 `com.example.examplemod`，其中 `examplemod` 是模组 ID）。这将保证，除非你有两个具有相同 ID 的模组（这绝不应该发生），否则你的包在加载时不会有任何问题。

你可以在[Oracle 的教程页面][naming]上找到一些额外的命名约定。

### 子包组织

除了顶层包之外，强烈建议将你的模组类分散到子包中。主要有两种方法可以做到这一点：

- **按功能分组**：为具有共同目的的类创建子包。例如，方块可以放在 `block` 下，物品在 `item` 下，实体在 `entity` 下，等等。Minecraft 本身也使用类似的结构（有一些例外）。
- **按逻辑分组**：为具有共同逻辑的类创建子包。例如，如果你正在创建一种新型的工作台，你会把它相关的方块、菜单、物品等都放在 `feature.crafting_table` 下。

#### 客户端、服务端和数据包

通常，仅用于特定侧或运行时的代码应与其它类隔离，放在单独的子包中。例如，与[数据生成][datagen]相关的代码应放在 `data` 包中，而仅在专用服务器上运行的代码应放在 `server` 包中。

强烈建议将[仅限客户端的代码][sides]隔离在 `client` 子包中。这是因为专用服务器无法访问 Minecraft 中任何仅限客户端的包，如果你的模组试图访问它们，会导致游戏崩溃。因此，拥有一个专门的包可以提供一个不错的健全性检查，以验证你没有在模组内部跨侧访问。

## 类命名方案

一个通用的类命名方案可以更容易地解读类的用途或轻松定位特定的类。

类通常以后缀表示其类型，例如：

- 一个名为 `PowerRing` 的 `Item` -> `PowerRingItem`。
- 一个名为 `NotDirt` 的 `Block` -> `NotDirtBlock`。
- 一个 `Oven` 的菜单 -> `OvenMenu`。

:::tip
Mojang 通常对除了实体之外的所有类都遵循类似的结构。实体仅用它们的名字表示（例如 `Pig`, `Zombie` 等）。
:::

## 多种方法选其一

执行某个特定任务有很多种方法：注册一个对象、监听事件等。通常建议保持一致性，使用单一方法来完成给定的任务。这可以提高可读性，并避免可能出现的奇怪交互或冗余（例如，你的事件监听器运行两次）。

[group]: modfiles.md#组-id
[naming]: https://docs.oracle.com/javase/tutorial/java/package/namingpkgs.html
[datagen]: ../resources/index.md#data-generation
[sides]: ../concepts/sides.md