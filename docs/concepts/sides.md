---
sidebar_position: 2
---
# 侧

和许多其他程序一样，我的世界遵循客户端-服务器概念，其中客户端负责显示数据，而服务器负责更新它们。在使用这些术语时，我们对它们的含义有一个相当直观的理解……对吧？

事实证明，并非如此。很多困惑源于我的世界根据上下文有两种不同的侧的概念：物理侧和逻辑侧。

## 逻辑侧 vs. 物理侧

### 物理侧

当你打开你的我的世界启动器，选择一个我的世界安装版本并按下“开始游戏”时，你启动了一个**物理客户端**。“物理”这个词在这里的含义是“这是一个客户端程序”。这尤其意味着客户端的功能，比如所有的渲染相关的东西，在这里是可用的，并且可以根据需要使用。相比之下，**物理服务器**，也被称为专用服务器，是你启动我的世界服务器 JAR 文件时打开的东西。虽然我的世界服务器带有一个基本的图形用户界面，但它缺少所有仅限客户端的功能。最值得注意的是，这意味着各种客户端类在服务器 JAR 中是缺失的。在物理服务器上调用这些类会导致类缺失错误，也就是崩溃，所以我们需要防范这种情况。

### 逻辑侧

逻辑侧主要关注我的世界的内部程序结构。**逻辑服务器**是游戏逻辑运行的地方。时间流逝、天气变化、实体更新、实体生成等都在服务器上运行。各种数据，比如物品栏内容，也由服务器负责。而**逻辑客户端**则负责显示所有需要显示的东西。我的世界将所有的客户端代码都放在一个独立的 `net.minecraft.client` 包中，并在一个名为渲染线程的独立线程中运行它，而其他所有东西都被认为是通用（即客户端和服务器）代码。

### 有什么区别？

物理侧和逻辑侧之间的区别通过两种情况可以最好地说明：

- 玩家加入一个**多人游戏**世界。这非常直接：玩家的物理（和逻辑）客户端连接到其他地方的物理（和逻辑）服务器——玩家不关心服务器在哪里；只要他们能连接上，这就是客户端所知道的全部，也是客户端需要知道的全部。
- 玩家加入一个**单人游戏**世界。这里事情就变得有趣了。玩家的物理客户端启动一个逻辑服务器，然后，现在作为逻辑客户端的角色，连接到同一台机器上的那个逻辑服务器。如果你熟悉网络，你可以把它想象成连接到 `localhost`（只是概念上的；没有实际的套接字或类似的东西参与其中）。

这两种情况也显示了其中的主要问题：如果一个逻辑服务器能用你的代码正常工作，这本身并不能保证物理服务器也能正常工作。这就是为什么你应该总是使用专用服务器进行测试以检查意外行为。由于不正确的客户端和服务器分离导致的 `NoClassDefFoundError` 和 `ClassNotFoundException` 是模组开发中最常见的错误之一。另一个常见的错误是使用静态字段并从两个逻辑侧访问它们；这尤其棘手，因为通常没有任何迹象表明出了问题。

:::tip
如果你需要将数据从一侧传输到另一侧，你必须[发送一个数据包][networking]。
:::

在 NeoForge 的代码库中，物理侧由一个名为 `Dist` 的枚举表示，而逻辑侧由一个名为 `LogicalSide` 的枚举表示。

:::info
历史上，服务器 JAR 文件中曾经有过客户端没有的类。在现代版本中，情况不再如此；如果你愿意，可以说物理服务器是物理客户端的一个子集。
:::

## 执行特定侧的操作

### `Level#isClientSide()`

这个布尔值检查将是你最常用的检查侧的方式。在一个 `Level` 对象上查询这个字段可以确定该 `Level` 所属的**逻辑**侧：如果这个字段是 `true`，那么这个 `level` 正在逻辑客户端上运行。如果这个字段是 `false`，那么这个 `level` 正在逻辑服务器上运行。由此可知，物理服务器上的这个字段将永远是 `false`，但我们不能假设 `false` 就意味着是物理服务器，因为这个字段在物理客户端内部的逻辑服务器（即单人游戏世界）中也可能是 `false`。

在你需要确定是否应该运行游戏逻辑和其他机制时，请使用这个检查。例如，如果你想在玩家每次点击你的方块时对玩家造成伤害，或者让你的机器将泥土加工成钻石，你应该在确保 `#isClientSide` 为 `false` 之后再这样做。将游戏逻辑应用于逻辑客户端，在最好的情况下会导致不同步（幽灵实体、不同步的统计数据等），在最坏的情况下会导致崩溃。

:::tip
这个检查应该作为你的首选默认方法。只要你有可用的 `Level`，就使用这个检查。
:::

### `FMLEnvironment.dist`

`FMLEnvironment.dist` 是 `Level#isClientSide()` 检查的**物理**对应物。如果这个字段是 `Dist.CLIENT`，你就在物理客户端上。如果这个字段是 `Dist.DEDICATED_SERVER`，你就在物理服务器上。


#### `@Mod`

在处理仅限客户端的类时，检查物理环境很重要。分离只应在一个物理客户端上执行的代码的推荐方法是指定一个单独的 [`@Mod` 注解][mod]，将 `dist` 参数设置为该模组类应该加载的物理侧：

```java
@Mod("examplemod")
public class ExampleMod {
    public ExampleMod(IEventBus modBus) {
        // Perform logic in that should be executed on both sides
    }
}

@Mod(value = "examplemod", dist = Dist.CLIENT) 
public class ExampleModClient {
    public ExampleModClient(IEventBus modBus) {
        // Perform logic in that should only be executed on the physical client
    }
}

@Mod(value = "examplemod", dist = Dist.DEDICATED_SERVER) 
public class ExampleModDedicatedServer {
    public ExampleModDedicatedServer(IEventBus modBus) {
        // Perform logic in that should only be executed on the physical server
    }
}
```

:::tip
模组通常被期望在任何一侧都能工作。这尤其意味着，如果你正在开发一个仅限客户端的模组，你应该验证该模组确实在物理客户端上运行，并且在它没有运行时不做任何操作。
:::

[networking]: ../networking/index.md
[mod]: ../gettingstarted/modfiles.md#javafml-and-mod