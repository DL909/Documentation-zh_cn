---
sidebar_position: 4
---
# 工具

工具是主要用于破坏[方块][block]的[物品][item]。许多模组会添加新的工具集（例如铜工具）或新的工具类型（例如锤子）。

## 自定义工具集

一个工具集通常由五种物品组成：镐、斧、锹、锄和剑（剑在传统意义上不是工具，但为了保持一致性也包含在此处）。所有这些工具都使用以下八个[数据组件][datacomponents]来实现：

- `DataComponents#MAX_DAMAGE` 和 `#DAMAGE` 用于耐久度
- `#MAX_STACK_SIZE` 用于将堆叠大小设置为 `1`
- `#REPAIRABLE` 用于在铁砧中修复工具
- `#ENCHANTABLE` 用于最大[附魔][enchantment]值
- `#ATTRIBUTE_MODIFIERS` 用于攻击伤害和攻击速度
- `#TOOL` 用于挖掘信息
- `#WEAPON` 用于物品受到的伤害和禁用盾牌

通常，每个工具都使用 `Item.Properties#tool`、`#sword` 或 tool 的某个委托方法（`pickaxe`、`axe`、`hoe`、`shovel`）来设置。这些通常通过传入实用记录类 `ToolMaterial` 来处理。请注意，其他通常被认为是工具的物品，例如剪刀，其通用的挖掘逻辑不是通过数据组件实现的。相反，它们直接扩展 `Item` 并通过重写相关方法来处理挖掘。交互行为（默认为右键点击）也没有数据组件，这意味着锹、斧和锄都有它们自己的工具类 `ShovelItem`、`AxeItem` 和 `HoeItem`。

要创建一套标准的工具，您必须首先定义一个 `ToolMaterial`。参考值可以在 `ToolMaterial` 的常量中找到。本示例使用铜工具，您可以在此处使用自己的材料并根据需要调整值。

```java
// We place copper somewhere between stone and iron.
public static final ToolMaterial COPPER_MATERIAL = new ToolMaterial(
        // The tag that determines what blocks this material cannot break. See below for more information.
        MyBlockTags.INCORRECT_FOR_COPPER_TOOL,
        // Determines the durability of the material.
        // Stone is 131, iron is 250.
        200,
        // Determines the mining speed of the material. Unused by swords.
        // Stone uses 4, iron uses 6.
        5f,
        // Determines the attack damage bonus. Different tools use this differently. For example, swords do (getAttackDamageBonus() + 4) damage.
        // Stone uses 1, iron uses 2, corresponding to 5 and 6 attack damage for swords, respectively; our sword does 5.5 damage now.
        1.5f,
        // Determines the enchantability of the material. This represents how good the enchantments on this tool will be.
        // Gold uses 22, we put copper slightly below that.
        20,
        // The tag that determines what items can repair this material.
        Tags.Items.INGOTS_COPPER
);
```

现在我们有了 `ToolMaterial`，我们可以用它来[注册]工具。所有 `tool` 的委托方法都有相同的三个参数：

```java
// ITEMS is a DeferredRegister.Items
public static final DeferredItem<Item> COPPER_SWORD = ITEMS.registerItem(
    "copper_sword",
    props -> new Item(
        // The item properties.
        props.sword(
            // The material to use.
            COPPER_MATERIAL,
            // The type-specific attack damage bonus. 3 for swords, 1.5 for shovels, 1 for pickaxes, varying for axes and hoes.
            3,
            // The type-specific attack speed modifier. The player has a default attack speed of 4, so to get to the desired
            // value of 1.6f, we use -2.4f. -2.4f for swords, -3f for shovels, -2.8f for pickaxes, varying for axes and hoes.
            -2.4f,
        )
    )
);

public static final DeferredItem<Item> COPPER_AXE = ITEMS.registerItem("copper_axe", props -> new Item(props.axe(...)));
public static final DeferredItem<Item> COPPER_PICKAXE = ITEMS.registerItem("copper_pickaxe", props -> new Item(props.pickaxe(...)));
public static final DeferredItem<Item> COPPER_SHOVEL = ITEMS.registerItem("copper_shovel", props -> new Item(props.shovel(...)));
public static final DeferredItem<Item> COPPER_HOE = ITEMS.registerItem("copper_hoe", props -> new Item(props.hoe(...)));
```

:::note
`tool` 接受两个额外的参数：代表可以挖掘哪些方块的 `TagKey`，以及被击中时阻挡物（例如盾牌）被禁用的秒数。
:::

### 标签

创建 `ToolMaterial` 时，会为其分配一个方块[标签][tags]，其中包含如果用此工具破坏将不会掉落任何东西的方块。例如，`minecraft:incorrect_for_stone_tool` 标签包含像钻石矿石这样的方块，而 `minecraft:incorrect_for_iron_tool` 标签包含像黑曜石和远古残骸这样的方块。为了更容易地将方块分配到其不正确的挖掘等级，还存在一个标签，用于表示需要此工具才能挖掘的方块。例如，`minecraft:needs_iron_tool` 标签包含像钻石矿石这样的方块，而 `minecraft:needs_diamond_tool` 标签包含像黑曜石和远古残骸这样的方块。

如果你觉得可以，可以为你的工具重用其中一个不正确的标签。例如，如果我们希望我们的铜工具只是更耐用的石头工具，我们会传入 `BlockTags#INCORRECT_FOR_STONE_TOOL`。

或者，我们可以创建自己的标签，就像这样：

```java
// This tag will allow us to add these blocks to the incorrect tags that cannot mine them
public static final TagKey<Block> NEEDS_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "needs_copper_tool"));

// This tag will be passed into our material
public static final TagKey<Block> INCORRECT_FOR_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "incorrect_for_cooper_tool"));
```

然后，我们填充我们的标签。例如，让我们让铜能够挖掘金矿石、金块和红石矿石，但不能挖掘钻石或绿宝石。（红石块已经可以用石制工具挖掘了。）标签文件位于 `src/main/resources/data/mod_id/tags/block/needs_copper_tool.json`（其中 `mod_id` 是你的模组 ID）：

```json5
{
    "values": [
        "minecraft:gold_block",
        "minecraft:raw_gold_block",
        "minecraft:gold_ore",
        "minecraft:deepslate_gold_ore",
        "minecraft:redstone_ore",
        "minecraft:deepslate_redstone_ore"
    ]
}
```

然后，对于我们要传递给材料的标签，我们可以为任何不适用于石制工具但在我们的铜制工具标签内的工具提供一个负向约束。标签文件位于 `src/main/resources/data/mod_id/tags/block/incorrect_for_cooper_tool.json`：

```json5
{
    "values": [
        "#minecraft:incorrect_for_stone_tool"
    ],
    "remove": [
        "#mod_id:needs_copper_tool"
    ]
}
```

最后，我们可以将我们的标签传递给我们的材料实例，如上所示。

如果你想检查一个工具是否能使一个方块状态掉落其方块，请调用 `Tool#isCorrectForDrops`。可以通过使用 `DataComponents#TOOL` 调用 `ItemStack#get` 来获取 `Tool`。

## 自定义工具

可以通过 `Item.Properties#component` 将 `Tool` [数据组件][datacomponents]（通过 `DataComponents#TOOL`）添加到物品的默认组件列表中，从而创建自定义工具。

一个 `Tool` 包含一个 `Tool.Rule` 列表、持有该工具时的默认挖掘速度（默认为 `1`），以及挖掘一个方块时该工具应受到的伤害量（默认为 `1`）。一个 `Tool.Rule` 包含三条信息：一个应用于规则的方块的 `HolderSet`、一个可选的挖掘该集合中方块的速度，以及一个可选的布尔值，用于确定这些方块是否能从此工具掉落。如果未设置可选值，则将检查其他规则。如果所有规则都失败，默认行为是使用默认的挖掘速度并且该方块不能掉落。

:::note
一个 `HolderSet` 可以通过 `Registry#getOrThrow` 从一个 `TagKey` 创建。
:::

创建任何工具或多功能工具类物品（即将两种或更多种工具组合成一个物品，例如，将斧头和镐子作为一个物品）都是可能的，而无需使用任何现有的 `ToolMaterial` 引用。它可以通过以下部分的组合来实现：

-   通过 `Item.Properties#component` 设置 `DataComponents#TOOL` 来添加带有您自己规则的 `Tool`。
-   通过 `Item.Properties#attributes` 向物品添加[属性修饰符][attributemodifier]（例如攻击伤害、攻击速度）。
-   通过 `Item.Properties#durability` 添加物品耐久度。
-   通过 `Item.Properties#repariable` 允许物品被修复。
-   通过 `Item.Properties#enchantable` 允许物品被附魔。
-   通过 `Item.Properties#component` 设置 `DataComponents#WEAPON`，允许物品用作武器并可能禁用阻挡物。
-   重写 `IItemExtension#canPerformAction` 来确定物品可以执行哪些 [`ItemAbility`][itemability]。
-   如果你希望你的物品在右键点击时根据 `ItemAbility` 修改方块状态，请调用 `IBlockExtension#getToolModifiedState`。
-   将您的工具添加到某些 `minecraft:enchantable/*` `ItemTags` 中，以便您的物品可以应用某些附魔。
-   将您的工具添加到某些 `minecraft:*_preferred_weapons` 标签中，以允许生物优先拾取和使用您的武器。

对于盾牌，您可以应用副手的 [`DataComponents#EQUIPPABLE`][equippable] 数据组件和 `DataComponents#BLOCKS_ATTACKS` 以在激活时减少对持有实体的伤害。

## `ItemAbility`

`ItemAbility` 是对一个物品能做什么和不能做什么的抽象。这包括左键和右键行为。NeoForge 在 `ItemAbilities` 类中提供了默认的 `ItemAbility`：

-   斧头的右键能力，用于剥皮（原木）、刮除（氧化的铜）和除蜡（涂蜡的铜）。
-   锹的右键能力，用于平整（泥土路）和熄灭（营火）。
-   剪刀的能力，用于收获（蜂巢）、移除盔甲（披甲的狼）、雕刻（南瓜）、拆除（绊线钩）和修剪（阻止植物生长）。
-   剑的横扫、锄的耕地、盾的格挡、鱼竿的投掷、三叉戟的投掷、刷子的刷拭以及打火石的点燃能力。

要创建您自己的 `ItemAbility`，请使用 `ItemAbility#get` - 如果需要，它会创建一个新的 `ItemAbility`。然后，在一个自定义的工具类型中，根据需要重写 `IItemExtension#canPerformAction`。

要查询一个 `ItemStack` 是否可以执行某个 `ItemAbility`，请调用 `IItemStackExtension#canPerformAction`。请注意，这适用于任何 `Item`，而不仅仅是工具。

[block]: ../blocks/index.md
[datacomponents]: datacomponents.md
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[equippable]: armor.md#equippable
[item]: index.md
[itemability]: #itemabilitys
[registering]: ../concepts/registries.md#methods-for-registering
[tags]: ../resources/server/tags.md