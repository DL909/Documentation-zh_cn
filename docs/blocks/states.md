# 方块状态

通常，你会发现自己需要一个方块的不同状态。例如，小麦作物有八个生长阶段，为每个阶段制作一个单独的方块感觉不对。或者你有一个台阶或类似台阶的方块——一个底部状态，一个顶部状态，以及一个两者都有的状态。

这就是方块状态发挥作用的地方。方块状态是一种表示方块可以拥有的不同状态的简单方法，比如生长阶段或台阶的放置类型。

## 方块状态属性

方块状态使用一个属性系统。一个方块可以有多个不同类型的属性。例如，一个末地传送门框架有两个属性：它是否有眼睛（`eye`，2 个选项）和它放置的方向（`facing`，4 个选项）。因此，末地传送门框架总共有 8（2 * 4）个不同的方块状态：

```
minecraft:end_portal_frame[facing=north,eye=false]
minecraft:end_portal_frame[facing=east,eye=false]
minecraft:end_portal_frame[facing=south,eye=false]
minecraft:end_portal_frame[facing=west,eye=false]
minecraft:end_portal_frame[facing=north,eye=true]
minecraft:end_portal_frame[facing=east,eye=true]
minecraft:end_portal_frame[facing=south,eye=true]
minecraft:end_portal_frame[facing=west,eye=true]
```

`blockid[property1=value1,property2=value,...]` 这种表示法是以文本形式表示方块状态的标准化方式，并且在原版的某些地方（例如命令中）使用。

如果你的方块没有任何定义的方块状态属性，它仍然只有一个方块状态——那就是没有任何属性的状态，因为没有属性可以指定。这可以表示为 `minecraft:oak_planks[]` 或简单地 `minecraft:oak_planks`。

与方块一样，每个 `BlockState` 在内存中只存在一次。这意味着 `==` 可以而且应该被用来比较 `BlockState`。`BlockState` 也是一个 final 类，意味着它不能被扩展。**任何功能都应该在相应的[方块][block]类中实现！**

## 何时使用方块状态

### 方块状态 vs. 单独的方块

一个好的经验法则是：**如果它有不同的名称，它就应该是一个单独的方块**。一个例子是制作椅子方块：椅子的方向应该是一个属性，而不同类型的木材应该被分成不同的方块。所以你会为每种木材类型制作一个椅子方块，每个椅子方块有四个方块状态（每个方向一个）。

### 方块状态 vs. [方块实体][blockentity]

在这里，经验法则是：**如果你有有限数量的状态，就使用方块状态；如果你有无限或接近无限数量的状态，就使用方块实体。**方块实体可以存储任意数量的数据，但比方块状态慢。

方块状态和方块实体可以结合使用。例如，箱子使用方块状态属性来表示方向、是否被水淹没，或者是否成为一个双箱子，而存储物品栏、当前是否打开，或者与漏斗交互则由一个方块实体来处理。

对于“多少状态对于一个方块状态来说太多了？”这个问题没有明确的答案，但我们建议，如果你需要超过 8-9 位的数据（即超过几百个状态），你应该使用一个方块实体。

## 实现方块状态

要实现一个方块状态属性，在你的方块类中，创建或引用一个 `public static final Property<?>` 常量。虽然你可以自由地制作自己的 `Property<?>` 实现，但原版代码提供了一些方便的实现，应该能涵盖大多数用例：

-   `IntegerProperty`
    -   实现 `Property<Integer>`。定义一个持有整数值的属性。请注意，不支持负值。
    -   通过调用 `IntegerProperty#create(String propertyName, int minimum, int maximum)` 创建。
-   `BooleanProperty`
    -   实现 `Property<Boolean>`。定义一个持有 `true` 或 `false` 值的属性。
    -   通过调用 `BooleanProperty#create(String propertyName)` 创建。
-   `EnumProperty<E extends Enum<E>>`
    -   实现 `Property<E>`。定义一个可以取枚举类值的属性。
    -   通过调用 `EnumProperty#create(String propertyName, Class<E> enumClass)` 创建。
    -   也可以只使用枚举值的一个子集（例如 16 种 `DyeColor` 中的 4 种），请参阅 `EnumProperty#create` 的重载方法。

`BlockStateProperties` 类包含了共享的原版属性，应尽可能使用或引用它们，而不是创建自己的属性。

一旦你有了你的属性常量，就在你的方块类中覆盖 `Block#createBlockStateDefinition(StateDefinition.Builder)`。在该方法中，调用 `StateDefinition.Builder#add(YOUR_PROPERTY);`。`StateDefinition.Builder#add` 有一个可变参数，所以如果你有多个属性，你可以一次性将它们全部添加。

每个方块也会有一个默认状态。如果没有另外指定，默认状态将使用每个属性的默认值。你可以通过在你的构造函数中调用 `Block#registerDefaultState(BlockState)` 方法来更改默认状态。

如果你希望在放置你的方块时改变所使用的 `BlockState`，请覆盖 `Block#getStateForPlacement(BlockPlaceContext)`。这可以用来，例如，根据玩家放置方块时站立或朝向的位置来设置方块的方向。

为了进一步说明这一点，下面是 `EndPortalFrameBlock` 类相关部分的样子：

```java
public class EndPortalFrameBlock extends Block {
    // Note: It is possible to directly use the values in BlockStateProperties instead of referencing them here again.
    // However, for the sake of simplicity and readability, it is recommended to add constants like this.
    public static final EnumProperty<Direction> FACING = BlockStateProperties.FACING;
    public static final BooleanProperty EYE = BlockStateProperties.EYE;

    public EndPortalFrameBlock(BlockBehaviour.Properties pProperties) {
        super(pProperties);
        // stateDefinition.any() returns a random BlockState from an internal set,
        // we don't care because we're setting all values ourselves anyway
        this.registerDefaultState(stateDefinition.any()
                .setValue(FACING, Direction.NORTH)
                .setValue(EYE, false)
        );
    }

    @Override
    protected void createBlockStateDefinition(StateDefinition.Builder<Block, BlockState> pBuilder) {
        // this is where the properties are actually added to the state
        pBuilder.add(FACING, EYE);
    }

    @Override
    @Nullable
    public BlockState getStateForPlacement(BlockPlaceContext pContext) {
        // code that determines which state will be used when
        // placing down this block, depending on the BlockPlaceContext
    }
}
```

## 使用方块状态

要从 `Block` 得到 `BlockState`，调用 `Block#defaultBlockState()`。默认的方块状态可以通过 `Block#registerDefaultState` 来改变，如上所述。

你可以通过调用 `BlockState#getValue(Property<?>)` 来获取一个属性的值，将你想要获取值的属性传递给它。再次使用我们的末地传送门框架的例子，这会是这样的：

```java
// EndPortalFrameBlock.FACING is an EnumPropery<Direction> and thus can be used to obtain a Direction from the BlockState
Direction direction = endPortalFrameBlockState.getValue(EndPortalFrameBlock.FACING);
```

如果你想获得一个具有不同值集的 `BlockState`，只需在一个现有的方块状态上调用 `BlockState#setValue(Property<T>, T)`，并传入属性及其值。对于我们的拉杆，这会是这样的：

```java
endPortalFrameBlockState = endPortalFrameBlockState.setValue(EndPortalFrameBlock.FACING, Direction.SOUTH);
```

:::note
`BlockState` 是不可变的。这意味着当你调用 `#setValue(Property<T>, T)` 时，你实际上并没有修改方块状态。相反，内部会进行一次查找，并返回你所请求的方块状态对象，该对象是具有这些确切属性值的唯一存在的对象。这也意味着仅仅调用 `state#setValue` 而不将其保存到变量中（例如，保存回 `state`）是没有任何作用的。
:::

要从世界中获取一个 `BlockState`，使用 `Level#getBlockState(BlockPos)`。

### `Level#setBlock`

要在世界中设置一个 `BlockState`，使用 `Level#setBlock(BlockPos, BlockState, int)`。

`int` 参数值得额外解释一下，因为它的含义不是那么直观。它表示所谓的更新标志。

为了帮助正确设置更新标志，`Block` 类中有一些以 `UPDATE_` 为前缀的 `int` 常量。如果你想组合它们，可以对这些常量进行按位或运算（例如 `Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS`）。

-   `Block.UPDATE_NEIGHBORS` 向相邻方块发送更新。更具体地说，它调用 `Block#neighborChanged`，该方法会调用一系列方法，其中大部分都与红石有某种关系。
-   `Block.UPDATE_CLIENTS` 将方块更新同步到客户端。
-   `Block.UPDATE_INVISIBLE` 明确不在客户端上更新。这也覆盖了 `Block.UPDATE_CLIENTS`，导致更新不会被同步。方块总是在服务器上更新。
-   `Block.UPDATE_IMMEDIATE` 强制在客户端的主线程上重新渲染。
-   `Block.UPDATE_KNOWN_SHAPE` 停止邻居更新递归。
-   `Block.UPDATE_SUPPRESS_DROPS` 禁用该位置旧方块的掉落物。
-   `Block.UPDATE_MOVE_BY_PISTON` 仅由活塞代码使用，以表示方块被活塞移动。这主要负责延迟光照引擎更新。
-   `Block.UPDATE_SKIP_SHAPE_UPDATE_ON_WIRE` 由 `ExperimentalRedstoneWireEvaluator` 使用，以表示是否应跳过形状更新。仅当信号强度改变（非放置操作）或信号的原始来源不是当前导线时设置此标志。
-   `Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS` 阻止 `BlockEntity#preRemoveSideEffects` 被调用。这通常会阻止方块实体清空其内容。
-   `Block.UPDATE_SKIP_ON_PLACE` 阻止 `Block#onPlace` 被调用。这通常会阻止任何方块处理其初始行为（例如，更新铁轨以连接到其他方块，生成一个铁傀儡）。
-   `Block.UPDATE_NONE` 是 `Block.UPDATE_INVISIBLE | Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS` 的别名。
-   `Block.UPDATE_ALL` 是 `Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS` 的别名。
-   `Block.UPDATE_ALL_IMMEDIATE` 是 `Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS | Block.UPDATE_IMMEDIATE` 的别名。
-   `Block.UPDATE_SKIP_ALL_SIDEEFFECTS` 是 `Block.UPDATE_SKIP_ON_PLACE | Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS | Block.UPDATE_SUPPRESS_DROPS | Block.UPDATE_KNOWN_SHAPE` 的别名

还有一个方便的方法 `Level#setBlockAndUpdate(BlockPos pos, BlockState state)`，它在内部调用 `setBlock(pos, state, Block.UPDATE_ALL)`。

[block]: index.md
[blockentity]: ../blockentities/index.md