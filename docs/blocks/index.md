# 方块

方块是 Minecraft 世界的基础。它们构成了所有的地形、结构和机器。如果你有兴趣制作一个模组，那么你很可能会想要添加一些方块。本页面将指导你完成方块的创建，以及你可以对它们做的一些事情。

## 万物归一的方块

在开始之前，重要的是要理解游戏中每种方块永远只有一个实例。一个世界由成千上万个对那个方块在不同位置的引用组成。换句话说，同一个方块只是被显示了很多次。

因此，一个方块应该只被实例化一次，那就是在[注册][registration]期间。一旦方块被注册，你就可以根据需要使用已注册的引用。

与大多数其他注册表不同，方块可以使用一个特殊版本的 `DeferredRegister`，称为 `DeferredRegister.Blocks`。`DeferredRegister.Blocks` 的作用基本上就像一个 `DeferredRegister<Block>`，但有一些细微的差别：

-   它们通过 `DeferredRegister.createBlocks("yourmodid")` 创建，而不是常规的 `DeferredRegister.create(...)` 方法。
-   `#register` 返回一个 `DeferredBlock<T extends Block>`，它扩展了 `DeferredHolder<Block, T>`。`T` 是我们正在注册的方块类的类型。
-   有一些用于注册方块的辅助方法。更多详情请见[下文][below]。

那么现在，让我们来注册我们的方块：

```java
//BLOCKS is a DeferredRegister.Blocks
public static final DeferredBlock<Block> MY_BLOCK = BLOCKS.register("my_block", registryName -> new Block(...));
```

注册方块后，所有对新 `my_block` 的引用都应使用这个常量。例如，如果你想检查给定位置的方块是否是 `my_block`，代码会是这样的：

```java
level.getBlockState(position) // returns the blockstate placed in the given level (world) at the given position
    //highlight-next-line
    .is(MyBlockRegistrationClass.MY_BLOCK);
```

这种方法还有一个方便的效果，即 `block1 == block2` 是有效的，并且可以用来替代 Java 的 `equals` 方法（当然，使用 `equals` 仍然有效，但没有意义，因为它也是通过引用进行比较的）。

:::danger
不要在注册之外调用 `new Block()`！一旦你这么做，事情就会出问题：

-   方块必须在注册表未冻结时创建。NeoForge 会为你解冻注册表，并在之后冻结它们，所以注册是你创建方块的时间窗口。
-   如果你试图在注册表再次冻结时创建和/或注册一个方块，游戏将会崩溃并报告一个 `null` 方块，这可能会非常令人困惑。
-   如果你仍然设法拥有一个悬空的方块实例，游戏在同步和保存时将无法识别它，并会将其替换为空气。
:::

## 创建方块

如前所述，我们首先创建我们的 `DeferredRegister.Blocks`：

```java
public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("yourmodid");
```

### 基本方块

对于不需要特殊功能的简单方块（比如圆石、木板等），可以直接使用 `Block` 类。为此，在注册期间，用一个 `BlockBehaviour.Properties` 参数实例化 `Block`。这个 `BlockBehaviour.Properties` 参数可以使用 `BlockBehaviour.Properties#of` 创建，并且可以通过调用其方法进行自定义。其中最重要的方法有：

-   `setId` - 设置方块的资源键。
    -   这**必须**在每个方块上设置；否则将抛出异常。
-   `destroyTime` - 决定方块被破坏所需的时间。
    -   石头的破坏时间是 1.5，泥土是 0.5，黑曜石是 50，而基岩是 -1（不可破坏）。
-   `explosionResistance` - 决定方块的抗爆性。
    -   石头的抗爆性是 6.0，泥土是 0.5，黑曜石是 1,200，而基岩是 3,600,000。
-   `sound` - 设置方块在被敲击、破坏或放置时发出的声音。
    -   默认值是 `SoundType.STONE`。更多详情请见[声音页面][sounds]。
-   `lightLevel` - 设置方块的发光等级。接受一个带有 `BlockState` 参数的函数，该函数返回一个 0 到 15 之间的值。
    -   例如，荧石使用 `state -> 15`，而火把使用 `state -> 14`。
-   `friction` - 设置方块的摩擦力（滑度）。
    -   默认值是 0.6。冰使用 0.98。

因此，举个例子，一个简单的实现会是这样的：

```java
//BLOCKS is a DeferredRegister.Blocks
public static final DeferredBlock<Block> MY_BETTER_BLOCK = BLOCKS.register(
    "my_better_block", 
    registryName -> new Block(BlockBehaviour.Properties.of()
        //highlight-start
        .setId(ResourceKey.create(Registries.BLOCK, registryName))
        .destroyTime(2.0f)
        .explosionResistance(10.0f)
        .sound(SoundType.GRAVEL)
        .lightLevel(state -> 7)
        //highlight-end
    ));
```

如需更多文档，请参阅 `BlockBehaviour.Properties` 的源代码。如需更多示例，或查看 Minecraft 使用的值，请查看 `Blocks` 类。

:::note
重要的是要理解，世界中的方块与物品栏中的方块不是一回事。物品栏中看起来像方块的东西实际上是一个 `BlockItem`，这是一种特殊的[物品][item]，使用时会放置一个方块。这也意味着像创造模式物品栏或最大堆叠数量这样的东西是由相应的 `BlockItem` 处理的。

`BlockItem` 必须与方块分开注册。这是因为一个方块不一定需要一个物品，例如，如果它不打算被收集（就像火一样）。
:::

### 更多功能

直接使用 `Block` 只允许创建非常基本的方块。如果你想添加功能，比如玩家交互或不同的碰撞箱，就需要一个扩展 `Block` 的自定义类。`Block` 类有许多可以被覆写以实现不同功能的方法；更多信息请参阅 `Block`、`BlockBehaviour` 和 `IBlockExtension` 类。另请参阅下面的[使用方块][usingblocks]部分，了解一些最常见的方块用例。

如果你想制作一个有不同变种的方块（比如有底部、顶部和双层变种的台阶），你应该使用[方块状态][blockstates]。最后，如果你想要一个能存储额外数据的方块（比如存储其物品栏的箱子），应该使用[方块实体][blockentities]。这里的经验法则是，如果你有有限且数量合理的状态（最多几百个状态），就使用方块状态；如果你有无限或接近无限数量的状态，就使用方块实体。

#### 方块类型

方块类型是用于序列化和反序列化方块对象的 [`MapCodec`][codec]。此 `MapCodec` 通过 `BlockBehaviour#codec` 设置，并[注册][registration]到方块类型注册表。目前，它唯一的用途是在生成方块列表报告时。应为 `Block` 的每个子类创建一个方块类型。例如，`FlowerBlock#CODEC` 代表大多数花的方块类型，而其子类 `WitherRoseBlock` 有一个单独的方块类型。

如果方块子类只接收 `BlockBehaviour.Properties`，那么可以使用 `BlockBehaviour#simpleCodec` 来创建 `MapCodec`。

```java
// For some block subclass
public class SimpleBlock extends Block {
    public SimpleBlock(BlockBehavior.Properties properties) {
        // ...
    }

    @Override
    public MapCodec<SimpleBlock> codec() {
        return SIMPLE_CODEC.get();
    }
}

// In some registration class
public static final DeferredRegister<MapCodec<? extends Block>> REGISTRAR = DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");

public static final Supplier<MapCodec<SimpleBlock>> SIMPLE_CODEC = REGISTRAR.register(
    "simple",
    () -> BlockBehaviour.simpleCodec(SimpleBlock::new)
);
```

如果方块子类包含更多参数，则应使用 [`RecordCodecBuilder#mapCodec`][codec] 来创建 `MapCodec`，为 `BlockBehaviour.Properties` 参数传入 `BlockBehaviour#propertiesCodec`。

```java
// For some block subclass
public class ComplexBlock extends Block {
    public ComplexBlock(int value, BlockBehavior.Properties properties) {
        // ...
    }

    @Override
    public MapCodec<ComplexBlock> codec() {
        return COMPLEX_CODEC.get();
    }

    public int getValue() {
        return this.value;
    }
}

// In some registration class
public static final DeferredRegister<MapCodec<? extends Block>> REGISTRAR = DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");

public static final Supplier<MapCodec<ComplexBlock>> COMPLEX_CODEC = REGISTRAR.register(
    "simple",
    () -> RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            Codec.INT.fieldOf("value").forGetter(ComplexBlock::getValue),
            BlockBehaviour.propertiesCodec() // represents the BlockBehavior.Properties parameter
        ).apply(instance, ComplexBlock::new)
    )
);
```

:::note
尽管方块类型目前基本上没有被使用，但随着 Mojang 继续向以编解码器为中心的结构发展，预计它在未来会变得更加重要。
:::

### `DeferredRegister.Blocks` 辅助方法

我们已经在[上文][above]讨论了如何创建一个 `DeferredRegister.Blocks`，以及它会返回 `DeferredBlock`。现在，让我们看看这个专门的 `DeferredRegister` 还提供了哪些其他实用功能。让我们从 `#registerBlock` 开始：

```java
public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("yourmodid");

public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.register(
    "example_block", registryName -> new Block(
        BlockBehaviour.Properties.of()
            // The ID must be set on the block
            .setId(ResourceKey.create(Registries.BLOCK, registryName))
    )
);

// Same as above, except that the block properties are constructed eagerly.
// setId is also called internally on the properties object.
public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerBlock(
    "example_block",
    Block::new, // The factory that the properties will be passed into.
    BlockBehaviour.Properties.of() // The properties to use.
);
```

如果你想使用 `Block::new`，你可以完全省略工厂方法：

```java
public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerSimpleBlock(
    "example_block",
    BlockBehaviour.Properties.of() // The properties to use.
);
```

这与上一个例子的作用完全相同，但稍微短一些。当然，如果你想使用 `Block` 的子类而不是 `Block` 本身，你就必须使用前一种方法。

### 资源

如果你注册了你的方块并将其放置在世界中，你会发现它缺少像纹理之类的东西。这是因为[纹理][textures]等是由 Minecraft 的资源系统处理的。要将纹理应用于方块，你必须提供一个[模型][model]和一个将方块与纹理和形状关联起来的[方块状态文件][bsfile]。请阅读链接的文章以获取更多信息。

## 使用方块

方块很少被直接用来做事情。事实上，Minecraft 中可能最常见的两种操作——获取一个位置的方块，以及在一个位置设置一个方块——都使用方块状态，而不是方块。一般的设计方法是让方块定义行为，但让行为实际通过方块状态来运行。因此，`BlockState` 经常作为参数传递给 `Block` 的方法。关于如何使用方块状态以及如何从一个方块中获取一个方块状态的更多信息，请参阅[使用方块状态][usingblockstates]。

在几种情况下，`Block` 的多个方法在不同时间被使用。以下小节列出了最常见的与方块相关的流程。除非另有说明，所有方法都在两个逻辑侧调用，并且应在两侧返回相同的结果。

### 放置方块

方块放置逻辑是从 `BlockItem#useOn`（或其子类的实现，例如用于睡莲的 `PlaceOnWaterBlockItem`）调用的。有关游戏如何到达此处的更多信息，请参阅[右键点击物品][rightclick]。实际上，这意味着一旦一个 `BlockItem` 被右键点击（例如一个圆石物品），这个行为就会被调用。

-   检查几个先决条件，例如你不在观察者模式下，方块所需的所有特性标志都已启用，或者目标位置没有超出世界边界。如果至少有一个检查失败，流程结束。
-   在尝试放置方块的位置上，对当前方块调用 `BlockBehaviour#canBeReplaced`。如果它返回 `false`，流程结束。这里返回 `true` 的著名案例是高草或雪层。
-   调用 `Block#getStateForPlacement`。根据上下文（包括位置、旋转和方块放置的面等信息），这里可以返回不同的方块状态。这对于例如可以朝不同方向放置的方块很有用。
-   用上一步获得的方块状态调用 `BlockBehaviour#canSurvive`。如果它返回 `false`，流程结束。
-   通过调用 `Level#setBlock` 将方块状态设置到世界中。
    -   在该 `Level#setBlock` 调用中，会调用 `BlockBehaviour#onPlace`。
-   调用 `Block#setPlacedBy`。

### 破坏方块

破坏方块要复杂一些，因为它需要时间。这个过程大致可以分为三个阶段：“初始化”、“挖掘中”和“实际破坏”。

-   当鼠标左键被点击时，进入“初始化”阶段。
-   现在，需要按住鼠标左键，进入“挖掘中”阶段。**这个阶段的方法每 tick 都会被调用。**
-   如果“挖掘中”阶段没有被中断（通过释放鼠标左键）并且方块被破坏，则进入“实际破坏”阶段。

对于喜欢伪代码的人：

```java
leftClick();
initiatingStage();
while (leftClickIsBeingHeld()) {
    miningStage();
    if (blockIsBroken()) {
        actuallyBreakingStage();
        break;
    }
}
```

以下小节将这些阶段进一步分解为实际的方法调用。有关游戏如何从左键点击到这个流程的信息，请参阅[左键点击物品][leftclick]。

#### “初始化”阶段

-   检查几个先决条件，例如你不在观察者模式下，你主手中的 `ItemStack` 所需的所有特性标志都已启用，或者所讨论的方块没有超出世界边界。如果至少有一个检查失败，流程结束。
-   触发 `PlayerInteractEvent.LeftClickBlock` 事件。如果事件被取消，流程结束。
    -   请注意，当事件在客户端被取消时，不会向服务器发送数据包，因此服务器上不会运行任何逻辑。
    -   然而，在服务器上取消此事件仍会导致客户端代码运行，这可能导致不同步！
-   调用 `BlockBehaviour#attack`。

#### “挖掘中”阶段

-   触发 `PlayerInteractEvent.LeftClickBlock` 事件。如果事件被取消，流程将进入“结束”阶段。
    -   请注意，当事件在客户端被取消时，不会向服务器发送数据包，因此服务器上不会运行任何逻辑。
    -   然而，在服务器上取消此事件仍会导致客户端代码运行，这可能导致不同步！
-   调用 `BlockBehaviour#getDestroyProgress` 并将其值加到内部的破坏进度计数器上。
    -   `BlockBehaviour#getDestroyProgress` 返回一个 0 到 1 之间的浮点值，表示每 tick 应该增加多少破坏进度计数器。
-   进度覆盖层（裂纹纹理）相应更新。
-   如果破坏进度大于 1.0（即完成，即方块应该被破坏），则退出“挖掘中”阶段，进入“实际破坏”阶段。

#### “实际破坏”阶段

-   调用 `Item#canDestroyBlock`。如果它返回 `false`（确定方块不应被破坏），流程将进入“结束”阶段。
-   如果方块是 `GameMasterBlock` 的实例，则调用 `Player#canUseGameMasterBlocks`。这决定了玩家是否有能力破坏仅限创造模式的方块。如果为 `false`，流程将进入“结束”阶段。
-   仅限服务器：调用 `Player#blockActionRestricted`。这决定了当前玩家是否不能破坏该方块。如果为 `true`，流程将进入“结束”阶段。
-   仅限服务器：触发 `BlockEvent.BreakEvent` 事件。如果被取消，流程将进入“结束”阶段。初始的取消状态由上述三种方法决定。
-   调用 `Block#playerWillDestroy`。
-   仅限服务器：调用 `IBlockExtension#canHarvestBlock`。这决定了方块是否可以被采集，即破坏时掉落物品。如果 `Player#preventsBlockDrops` 返回 true，这将别忽略。
    -   仅限服务器：如果 `IBlockExtension#canHarvestBlock` 没有被覆写（或覆写时调用了父类方法），则触发 `PlayerEvent.HarvestCheck` 事件。如果 `canHarvest` 返回 `false`，那么 `Block#playerDestroy` 将不会被调用，从而阻止任何资源或经验掉落。
-   调用 `IBlockExtension#onDestroyedByPlayer`。如果它返回 `false`，流程将进入“结束”阶段。
    -   通过 `Level#setBlock` 调用，并将 `Blocks.AIR.defaultBlockState()` 或当前记录的流体作为方块状态参数，将方块状态从关卡中移除。
        -   在该 `Level#setBlock` 调用中，会调用 `Block#onRemove`。
    -   如果 `IBlockExtension#onDestroyedByPlayer` 返回 `true`，则调用 `Block#destroy`。
-   仅限服务器：如果之前对 `IBlockExtension#canHarvestBlock` 和 `IBlockExtension#onDestroyedByPlayer` 的调用都返回 `true`，则调用 `Block#playerDestroy`。
    -   仅限服务器：调用 `Block#dropResources`。这决定了方块被挖掘时掉落什么，包括经验。
        -   仅限服务器：触发 `BlockDropsEvent` 事件。如果事件被取消，则方块破坏时不会掉落任何东西。否则，`BlockDropsEvent#getDrops` 中的每个 `ItemEntity` 都会被添加到当前关卡中。此外，如果 `getDroppedExperience` 大于 0，则调用 `Block#popExperience`。
            -   仅限服务器：调用 `IBlockExtension#getExpDrop`，并由 `EnchantmentHelper#processBlockExperience` 增强。这是 `BlockDropsEvent#getDroppedExperience` 在可能被修改之前的初始值。
-   仅限服务器：如果在上述过程中用于挖掘方块的物品损坏，则触发 `PlayerDestroyItemEvent` 事件。

#### 挖掘速度

挖掘速度是根据方块的硬度、所用[工具][tool]的速度以及几个实体[属性][attributes]按以下规则计算的：

```java
// This will return the tool's mining speed, or 1 if the held item is either empty, not a tool,
// or not applicable for the block being broken.
float destroySpeed = item.getDestroySpeed(blockState);
// If we have an applicable tool, add the minecraft:mining_efficiency attribute as an additive modifier.
if (destroySpeed > 1) {
    destroySpeed += player.getAttributeValue(Attributes.MINING_EFFICIENCY);
}
// Apply effects from haste or conduit power.
if (player.hasEffect(MobEffects.HASTE) || player.hasEffect(MobEffects.CONDUIT_POWER)) {
    int haste = player.hasEffect(MobEffects.HASTE)
        ? player.getEffect(MobEffects.HASTE).getAmplifier()
        : 0;
    int conduitPower = player.hasEffect(MobEffects.CONDUIT_POWER)
        ? player.getEffect(MobEffects.CONDUIT_POWER).getAmplifier()
        : 0;
    int amplifier = Math.max(haste, conduitPower);
    destroySpeed *= 1 + (amplifier + 1) * 0.2f;
}
// Apply slowness effect.
if (player.hasEffect(MobEffects.MINING_FATIGUE)) {
    destroySpeed *= switch (player.getEffect(MobEffects.MINING_FATIGUE).getAmplifier()) {
        case 0 -> 0.3F;
        case 1 -> 0.09F;
        case 2 -> 0.0027F;
        default -> 8.1E-4F;
    };
}
// Add the minecraft:block_break_speed attribute as a multiplicative modifier.
destroySpeed *= player.getAttributeValue(Attributes.BLOCK_BREAK_SPEED);
// If the player is underwater, apply the underwater mining speed penalty multiplicatively.
if (player.isEyeInFluid(FluidTags.WATER)) {
    destroySpeed *= player.getAttributeValue(Attributes.SUBMERGED_MINING_SPEED);
}
// If the player is trying to break a block in mid-air, make the player mine 5 times slower.
if (!player.onGround()) {
    destroySpeed /= 5;
}
destroySpeed = /* The PlayerEvent.BreakSpeed event is fired here, allowing modders to further modify this value. */;
return destroySpeed;
```

此确切代码可在 `Player#getDestroySpeed` 中找到以供参考。

### 游戏刻（Ticking）

游戏刻是一种机制，每 1/20 秒，即 50 毫秒（“一个游戏刻”）更新（tick）游戏的某些部分。方块提供了不同的游戏刻方法，它们以不同的方式被调用。

#### 服务端游戏刻与计划刻

`BlockBehaviour#tick` 在两种情况下被调用：通过默认的[随机刻][randomtick]（见下文），或通过计划刻。计划刻可以通过 `Level#scheduleTick(BlockPos, Block, int)` 创建，其中 `int` 表示延迟。这在原版的很多地方都有使用，例如，大型垂滴叶的倾斜机制就严重依赖这个系统。其他著名的使用者是各种红石元件。

#### 客户端游戏刻

`Block#animateTick` 仅在客户端上每帧调用一次。这是客户端独有行为发生的地方，例如火把粒子生成。

#### 天气刻

天气刻由 `Block#handlePrecipitation` 处理，并且独立于常规的游戏刻运行。它仅在服务器上调用，仅在下雨或下雪时，有 1/16 的几率。例如，炼药锅在雨雪天气中会接水，就是利用了这个机制。

#### 随机刻

随机刻系统独立于常规的游戏刻运行。必须通过方块的 `BlockBehaviour.Properties` 调用 `BlockBehaviour.Properties#randomTicks()` 方法来启用随机刻。这使得方块能够参与随机刻机制。

在一个区块中，每个游戏刻都会对一定数量的方块进行随机刻。这个数量由 `randomTickSpeed` 游戏规则定义。其默认值为 3，即每个游戏刻，从区块中选择 3 个随机方块。如果这些方块启用了随机刻，那么它们各自的 `BlockBehaviour#randomTick` 方法就会被调用。

Minecraft 中的许多机制都使用了随机刻，例如植物生长、冰雪融化或铜氧化。

[above]: #万物归一的方块
[attributes]: ../entities/attributes.md
[below]: #deferredregisterblocks-辅助方法
[blockentities]: ../blockentities/index.md
[blockstates]: states.md
[bsfile]: ../resources/client/models/index.md#blockstate-files
[codec]: ../datastorage/codecs.md#records
[item]: ../items/index.md
[leftclick]: ../items/interactions.md#left-clicking-an-item
[model]: ../resources/client/models/index.md
[randomtick]: #random-ticking
[registration]: ../concepts/registries.md#注册方法
[resources]: ../resources/index.md#assets
[rightclick]: ../items/interactions.md#right-clicking-an-item
[sounds]: ../resources/client/sounds.md
[textures]: ../resources/client/textures.md
[tool]: ../items/tools.md
[usingblocks]: #使用方块
[usingblockstates]: states.md#使用方块状态