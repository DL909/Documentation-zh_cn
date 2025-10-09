# 物品

与方块一样，物品是 Minecraft 的一个关键组成部分。方块构成了你周围的世界，而物品则存在于物品栏中。

## 物品到底是什么？

在我们进一步深入创建物品之前，了解物品实际上是什么，以及它与比如说一个[方块][block]有什么区别是很重要的。让我们用一个例子来说明这一点：

-   在世界中，你遇到了一个泥土方块并想挖掘它。这是一个**方块**，因为它被放置在世界中。（实际上，它不是一个方块，而是一个方块状态。更多详细信息请参阅[方块状态文章][blockstates]。)
    -   并非所有方块在破坏时都会掉落自身（例如树叶），更多信息请参阅关于[战利品表][loottables]的文章。
-   一旦你[挖掘了方块][breaking]，它就会被移除（= 替换为空气方块）并且泥土会掉落。掉落的泥土是一个物品**[实体][entity]**。这意味着像其他实体（猪、僵尸、箭等）一样，它天生就可以被水推动，或者被火和熔岩烧毁。
-   一旦你捡起泥土物品实体，它就变成了你物品栏中的一个**物品堆**。一个物品堆，简单来说，就是一个带有一些额外信息的物品实例，比如堆叠大小。
-   物品堆由其对应的**物品**（也就是我们正在创建的东西）支持。物品持有[数据组件][datacomponents]，其中包含所有物品堆初始化时的默认信息（例如，每把铁剑的最大耐久度为 250），而物品堆可以修改这些数据组件，从而允许同一物品的两个不同堆叠具有不同的信息（例如，一把铁剑还剩 100 次使用次数，而另一把铁剑还剩 200 次）。关于哪些是通过物品完成的，哪些是通过物品堆完成的更多信息，请继续阅读。
    -   物品和物品堆之间的关系大致与[方块][block]和[方块状态][blockstates]之间的关系相同，即方块状态总是由一个方块支持。这个比较不是很准确（例如，物品堆不是单例），但它给出了一个关于这里概念的基本概念。

## 创建物品

现在我们了解了物品是什么，让我们来创建一个吧！

与基本方块一样，对于不需要特殊功能的基本物品（比如木棍、糖等），可以直接使用 `Item` 类。为此，在注册期间，用一个 `Item.Properties` 参数实例化 `Item`。这个 `Item.Properties` 参数可以使用 `Item.Properties#of` 创建，并且可以通过调用其方法进行自定义：

-   `setId` - 设置物品的资源键。
    -   这**必须**在每个物品上设置；否则将抛出异常。
-   `overrideDescription` - 设置物品的翻译键。创建的 `Component` 存储在 `DataComponents#ITEM_NAME` 中。
-   `useBlockDescriptionPrefix` - 便捷辅助方法，使用翻译键 `block.<modid>.<registry_name>` 调用 `overrideDescription`。这应该在任何 `BlockItem` 上调用。
-   `requiredFeatures` - 设置此物品所需的特性标志。这主要用于原版在次要版本中的特性锁定系统。不鼓励使用此功能，除非你正在与原版特性标志锁定的系统集成。
-   `stacksTo` - 设置此物品的最大堆叠数量（通过 `DataComponents#MAX_STACK_SIZE`）。默认为 64。例如末影珍珠或其他只能堆叠到 16 的物品使用。
-   `durability` - 设置此物品的耐久度（通过 `DataComponents#MAX_DAMAGE`）并将初始伤害设置为 0（通过 `DataComponents#DAMAGE`）。默认为 0，表示“无耐久度”。例如，铁工具在这里使用 250。请注意，设置耐久度会自动将最大堆叠数量锁定为 1。
-   `fireResistant` - 使使用此物品的物品实体对火和熔岩免疫（通过 `DataComponents#FIRE_RESISTANT`）。用于各种下界合金物品。
-   `rarity` - 设置此物品的稀有度（通过 `DataComponents#RARITY`）。目前，这只会改变物品的颜色。`Rarity` 是一个枚举，包含四个值：`COMMON`（白色，默认）、`UNCOMMON`（黄色）、`RARE`（水绿色）和 `EPIC`（浅紫色）。请注意，模组可能会添加更多稀有度类型。
-   `setNoCombineRepair` - 禁用此物品的砂轮和工作台修复。原版中未使用。
-   `jukeboxPlayable` - 设置插入唱片机时播放的数据包 `JukeboxSong` 的资源键。
-   `food` - 设置此物品的 [`FoodProperties`][food]（通过 `DataComponents#FOOD`）。

如需示例，或查看 Minecraft 使用的各种值，请查看 `Items` 类。

### 残留物和冷却

物品可能具有额外的属性，这些属性在使用时应用或在一段时间内阻止物品使用：

-   `craftRemainder` - 设置此物品的合成残留物。原版用于装满的桶，在合成后留下空桶。
-   `usingConvertsTo` - 设置物品在通过 `Item#use`、`IItemExtension#finishUsingItem` 或 `Item#releaseUsing` 使用完毕后返回的物品。`ItemStack` 存储在 `DataComponents#USE_REMAINDER` 上。
-   `useCooldown` - 设置物品可以再次使用前的秒数（通过 `DataComponents#USE_COOLDOWN`）。

### 工具和盔甲

有些物品的作用类似于[工具][tools]和[盔甲][armor]。它们是通过一系列物品属性构建的，只有部分用法委托给其关联的类：

-   `enchantable` - 设置物品堆的最大[附魔][enchantment]值，允许物品被附魔（通过 `DataComponents#ENCHANTABLE`）。
-   `repairable` - 设置可用于修复此物品耐久度的物品或标签（通过 `DataComponents#REPAIRABLE`）。必须具有耐久度组件且没有 `DataComponents#UNBREAKABLE`。
-   `equippable` - 设置物品可以装备到的槽位（通过 `DataComponents#EQUIPPABLE`）。
-   `equippableUnswappable` - 与 `equippable` 相同，但禁用通过使用物品按钮（默认为右键）进行快速交换。

更多信息可以在它们的相关页面上找到。

### 更多功能

直接使用 `Item` 只允许创建非常基本的物品。如果你想添加功能，例如右键交互，就需要一个扩展 `Item` 的自定义类。`Item` 类有许多可以被覆写以实现不同功能的方法；更多信息请参阅 `Item` 和 `IItemExtension` 类。

物品最常见的两个用例是左键点击和右键点击。由于它们的复杂性以及它们涉及到其他系统，它们在一个单独的[交互文章][interactions]中进行了解释。

### `DeferredRegister.Items`

所有注册表都使用 `DeferredRegister` 来注册其内容，物品也不例外。然而，由于添加新物品是绝大多数模组的一个基本功能，NeoForge 提供了 `DeferredRegister.Items` 辅助类，它扩展了 `DeferredRegister<Item>` 并提供了一些特定于物品的辅助方法：

```java
public static final DeferredRegister.Items ITEMS = DeferredRegister.createItems(ExampleMod.MOD_ID);

public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerItem(
    "example_item",
    Item::new, // The factory that the properties will be passed into.
    new Item.Properties() // The properties to use.
);
```

在内部，这将通过将属性参数应用于提供的物品工厂（通常是构造函数）来简单地调用 `ITEMS.register("example_item", registryName -> new Item(new Item.Properties().setId(ResourceKey.create(Registries.ITEM, registryName))))`。ID 会在属性上设置。

如果你想使用 `Item::new`，你可以完全省略工厂方法，并使用 `simple` 方法变体：

```java
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerSimpleItem(
    "example_item",
    new Item.Properties() // The properties to use.
);
```

这与上一个例子的作用完全相同，但稍微短一些。当然，如果你想使用 `Item` 的子类而不是 `Item` 本身，你就必须使用前一种方法。

这两个方法也都有省略 `new Item.Properties()` 参数的重载：

```java
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerItem("example_item", Item::new);

// Variant that also omits the Item::new parameter
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerSimpleItem("example_item");
```

最后，还有方块物品的快捷方式。除了 `setId`，这些方法还会调用 `useBlockDescriptionPrefix` 来将翻译键设置为用于方块的键：

```java
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    "example_block",
    ExampleBlocksClass.EXAMPLE_BLOCK,
    new Item.Properties()
);

// Variant that omits the properties parameter:
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    "example_block",
    ExampleBlocksClass.EXAMPLE_BLOCK
);

// Variant that omits the name parameter, instead using the block's registry name:
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    // Must be an instance of `Holder<Block>`
    // DeferredBlock<T> also works
    ExampleBlocksClass.EXAMPLE_BLOCK,
    new Item.Properties()
);

// Variant that omits both the name and the properties:
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    // Must be an instance of `Holder<Block>`
    // DeferredBlock<T> also works
    ExampleBlocksClass.EXAMPLE_BLOCK
);
```

:::note
如果你将已注册的方块放在一个单独的类中，你应该在物品类之前类加载你的方块类。
:::

### 资源

如果你注册了你的物品并获得了它（通过 `/give` 或[创造模式物品栏][creativetabs]），你会发现它缺少一个合适的模型和纹理。这是因为纹理和模型是由 Minecraft 的资源系统处理的。

要将一个简单的纹理应用于物品，你必须创建一个客户端物品模型 JSON 文件和一个纹理 PNG 文件。更多信息请参阅关于[客户端物品][citems]的部分。

## `ItemStack`s

与方块和方块状态一样，在大多数你期望使用 `Item` 的地方，实际上都使用 `ItemStack`。`ItemStack` 代表了容器（例如物品栏）中一个或多个物品的堆叠。再次与方块和方块状态一样，方法应该由 `Item` 覆写并被 `ItemStack` 调用，并且 `Item` 中的许多方法都会传入一个 `ItemStack` 实例。

一个 `ItemStack` 由三个主要部分组成：

-   它所代表的 `Item`，可通过 `ItemStack#getItem` 获取，或通过 `getItemHolder` 获取 `Holder<Item>`。
-   堆叠大小，通常在 1 到 64 之间，可通过 `getCount` 获取，并通过 `setCount` 或 `shrink` 更改。
-   [数据组件][datacomponents]映射，其中存储了堆叠特定的数据。可通过 `getComponents` 获取。组件值通常通过 `has`、`get`、`set`、`update` 和 `remove` 来访问和修改。

要创建一个新的 `ItemStack`，调用 `new ItemStack(Item)`，并传入其背后的物品。默认情况下，这使用 1 的数量并且没有 NBT 数据；如果需要，还有接受数量和 NBT 数据的构造函数重载。

`ItemStack` 是可变对象（见下文），但有时需要将它们视为不可变的。如果你需要修改一个应被视为不可变的 `ItemStack`，你可以使用 `#copy` 来克隆该堆叠，或者如果需要使用特定的堆叠大小，则使用 `#copyWithCount`。

如果你想表示一个堆叠没有物品，使用 `ItemStack.EMPTY`。如果你想检查一个 `ItemStack` 是否为空，调用 `#isEmpty`。

### `ItemStack` 的可变性

`ItemStack` 是可变对象。这意味着如果你调用例如 `#setCount` 或任何数据组件映射方法，`ItemStack` 本身将被修改。原版广泛使用 `ItemStack` 的可变性，并且有几个方法依赖于它。例如，`#split` 从调用它的堆叠中分离出给定的数量，在此过程中既修改了调用者，又返回了一个新的 `ItemStack`。

然而，这有时会在同时处理多个 `ItemStack` 时导致问题。最常见的情况是在处理物品栏槽位时，因为你必须同时考虑光标当前选择的 `ItemStack`，以及你试图插入或取出的 `ItemStack`。

:::tip
如有疑问，最好是安全第一，复制一份堆叠（`#copy`）。
:::

### JSON 表示

在许多情况下，例如[配方][recipes]，物品堆需要表示为 JSON 对象。一个物品堆的 JSON 表示如下所示：

```json5
{
    // The item ID. Required.
    "id": "minecraft:dirt",
    // The item stack count [1, 99]. Optional, defaults to 1.
    "count": 4,
    // A map of data components. Optional, defaults to an empty map.
    "components": {
        "minecraft:enchantment_glint_override": true
    }
}
```

## 创造模式物品栏

默认情况下，你的物品只能通过 `/give` 获得，不会出现在创造模式物品栏中。让我们来改变这一点！

将你的物品放入创造模式菜单的方式取决于你想将它添加到哪个标签页。

### 已有的创造模式物品栏

:::note
此方法用于将你的物品添加到 Minecraft 的标签页，或其他模组的标签页。要将物品添加到你自己的标签页，请参见下文。
:::

一个物品可以通过 `BuildCreativeModeTabContentsEvent` 事件添加到现有的 `CreativeModeTab` 中，该事件在[模组事件总线][modbus]上触发，并且仅在[逻辑客户端][sides]上。通过调用 `event#accept` 来添加物品。

```java
//MyItemsClass.MY_ITEM is a Supplier<? extends Item>, MyBlocksClass.MY_BLOCK is a Supplier<? extends Block>
@SubscribeEvent // on the mod event bus
public static void buildContents(BuildCreativeModeTabContentsEvent event) {
    // Is this the tab we want to add to?
    if (event.getTabKey() == CreativeModeTabs.INGREDIENTS) {
        event.accept(MyItemsClass.MY_ITEM.get());
        // Accepts an ItemLike. This assumes that MY_BLOCK has a corresponding item.
        event.accept(MyBlocksClass.MY_BLOCK.get());
    }
}
```

该事件还提供了一些额外信息，例如 `getFlags` 用于获取已启用的特性标志列表，或 `hasPermissions` 用于检查玩家是否有权限查看操作员物品标签页。

### 自定义创造模式物品栏

`CreativeModeTab` 是一个注册表，这意味着自定义的 `CreativeModeTab` 必须被[注册][registering]。创建一个创造模式物品栏使用一个构建器系统，该构建器可通过 `CreativeModeTab#builder` 获得。构建器提供了设置标题、图标、默认物品以及许多其他属性的选项。此外，NeoForge 还提供了额外的方法来自定义标签页的图像、标签和槽位颜色、标签页的排序位置等。

```java
//CREATIVE_MODE_TABS is a DeferredRegister<CreativeModeTab>
public static final Supplier<CreativeModeTab> EXAMPLE_TAB = CREATIVE_MODE_TABS.register("example", () -> CreativeModeTab.builder()
    //Set the title of the tab. Don't forget to add a translation!
    .title(Component.translatable("itemGroup." + MOD_ID + ".example"))
    //Set the icon of the tab.
    .icon(() -> new ItemStack(MyItemsClass.EXAMPLE_ITEM.get()))
    //Add your items to the tab.
    .displayItems((params, output) -> {
        output.accept(MyItemsClass.MY_ITEM.get());
        // Accepts an ItemLike. This assumes that MY_BLOCK has a corresponding item.
        output.accept(MyBlocksClass.MY_BLOCK.get());
    })
    .build()
);
```

## `ItemLike`

`ItemLike` 是一个由原版中的 `Item` 和 [`Block`][block] 实现的接口。它定义了 `#asItem` 方法，该方法返回对象实际内容的物品表示：`Item` 直接返回自身，而 `Block` 则返回其关联的 `BlockItem`（如果可用），否则返回 `Blocks.AIR`。`ItemLike` 被用于各种“来源”不重要的上下文中，例如在许多[数据生成器][datagen]中。

也可以在你自己的自定义对象上实现 `ItemLike`。只需覆写 `#asItem` 即可。

[armor]: armor.md
[block]: ../blocks/index.md
[blockstates]: ../blocks/states.md
[breaking]: ../blocks/index.md#breaking-a-block
[citems]: ../resources/client/models/items.md
[creativetabs]: #creative-tabs
[datacomponents]: datacomponents.md
[datagen]: ../resources/index.md#data-generation
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[entity]: ../entities/index.md
[food]: consumables.md#food
[hunger]: https://minecraft.wiki/w/Hunger#Mechanics
[interactions]: interactions.md
[loottables]: ../resources/server/loottables/index.md
[modbus]: ../concepts/events.md#event-buses
[recipes]: ../resources/server/recipes/index.md
[registering]: ../concepts/registries.md#methods-for-registering
[sides]: ../concepts/sides.md
[tools]: tools.md