---
sidebar_position: 1
---
# 交互

本页面旨在让玩家左键、右键或中键点击物品这个相当复杂且令人困惑的过程变得更容易理解，并阐明在何处、为何使用何种结果。

## `HitResult`

为了确定玩家当前正在看什么，Minecraft 使用 `HitResult`。`HitResult` 有点类似于其他游戏引擎中的射线投射结果，最值得注意的是它包含一个 `#getLocation` 方法。

一个命中结果可以是三种类型之一，通过 `HitResult.Type` 枚举表示：`BLOCK`、`ENTITY` 或 `MISS`。类型为 `BLOCK` 的 `HitResult` 可以强制转换为 `BlockHitResult`，而类型为 `ENTITY` 的 `HitResult` 可以强制转换为 `EntityHitResult`；这两种类型都提供了关于命中了哪个[方块][block]或[实体][entity]的额外上下文。如果类型是 `MISS`，则表示既没有命中方块也没有命中实体，不应强制转换为任何一个子类。

在[物理客户端][physicalside]上，`Minecraft` 类每一帧都会更新并存储当前注视的 `HitResult` 到 `hitResult` 字段中。然后可以通过 `Minecraft.getInstance().hitResult` 访问此字段。

## 左键点击物品

-   检查主手中的 [`ItemStack`][itemstack] 所需的所有[特性标志][featureflag]是否已启用。如果此检查失败，则流程结束。
-   使用鼠标左键和主手触发 `InputEvent.InteractionKeyMappingTriggered` 事件。如果[事件][event]被[取消][cancel]，则流程结束。
-   根据你正在看的东西（使用 `Minecraft` 中的 [`HitResult`][hitresult]），会发生不同的事情：
    -   如果你正在看一个在你触及范围内的[实体][entity]：
        -   触发 `AttackEntityEvent` 事件。如果事件被取消，则流程结束。
        -   调用 `IItemExtension#onLeftClickEntity`。如果返回 true，则流程结束。
        -   在目标上调用 `Entity#isAttackable`。如果返回 false，则流程结束。
        -   在目标上调用 `Entity#skipAttackInteraction`。如果返回 true，则流程结束。
        -   如果目标在 `minecraft:redirectable_projectile` 标签中（默认为火球和风弹）并且是 `Projectile` 的实例，则目标被偏转，流程结束。
        -   实体基础伤害（`minecraft:generic.attack_damage` [属性][attribute]的值）和附魔加成伤害被计算为两个独立的浮点数。如果两者都为 0，则流程结束。
            -   请注意，这不包括主手物品的[属性修饰符][attributemodifier]，这些是在检查后添加的。
        -   主手物品的 `minecraft:generic.attack_damage` 属性修饰符被添加到基础伤害中。
        -   触发 `CriticalHitEvent` 事件。如果事件的 `#isCriticalHit` 方法返回 true，则基础伤害将乘以事件的 `#getDamageMultiplier` 方法返回的值，如果[满足一系列条件][critical]，默认为 1.5，否则为 1.0，但可能会被事件修改。
        -   附魔加成伤害被添加到基础伤害中，得出最终的伤害值。
        -   触发 `SweepAttackEvent` 事件。如果事件的 `isSweeping` 方法返回 true，则玩家将执行一次横扫攻击。默认情况下，这会检查攻击冷却是否 > 90%，攻击不是暴击，玩家在地面上并且移动速度不超过其 `minecraft:generic.movement_speed` 属性值。
        -   调用 [`Entity#hurtOrSimulate`][hurt]。如果返回 false，则流程结束。
        -   如果目标是 `LivingEntity` 的实例，攻击强度大于 90%，玩家正在冲刺，并且经过附魔修改后的 `minecraft:attack_knockback` 属性值大于 0，则调用 `LivingEntity#knockback`。
            -   在该方法内部，会触发 `LivingKnockBackEvent` 事件。
        -   根据 `SweepAttackEvent#isSweeping`，玩家对附近的 `LivingEntity` 执行横扫攻击。
            -   在该方法内部，如果实体在玩家的触及范围内并且 `Entity#hurtServer` 返回 true，则会再次调用 `LivingEntity#knockback`，这反过来又会再次触发 `LivingKnockBackEvent` 事件。
        -   调用 `Item#hurtEnemy`。这可用于攻击后效果。例如，如果适用，钉头锤会在这里将玩家弹回空中。
        -   调用 `Item#postHurtEnemy`。耐久度伤害在此处应用。
    -   如果你正在看一个在你触及范围内的[方块][block]：
        -   启动[方块破坏子流程][blockbreak]。
    -   否则：
        -   触发 `PlayerInteractEvent.LeftClickEmpty` 事件。

## 右键点击物品

在右键点击流程中，会调用许多返回两种结果类型之一（见下文）的方法。如果返回了明确的成功或明确的失败，大多数这些方法都会取消流程。为便于阅读，从现在起，这种“明确的成功或明确的失败”将被称为“确定性结果”。

-   使用鼠标右键和主手触发 `InputEvent.InteractionKeyMappingTriggered` 事件。如果[事件][event]被[取消][cancel]，则流程结束。
-   检查几种情况，例如你不是在观察者模式下，或者你主手中的 [`ItemStack`][itemstack] 所需的所有[特性标志][featureflag]都已启用。如果至少有一个检查失败，则流程结束。
-   根据你正在看的东西（使用 `Minecraft` 中的 [`HitResult`][hitresult]），会发生不同的事情：
    -   如果你正在看一个在你触及范围内且未超出世界边界的[实体][entity]：
        -   触发 `PlayerInteractEvent.EntityInteractSpecific` 事件。如果事件被取消，则流程结束。
        -   将在**你正在看的实体上**调用 `Entity#interactAt`。如果它返回一个确定性结果，则流程结束。
            -   如果你想为自己的实体添加行为，请重写此方法。如果你想为原版实体添加行为，请使用事件。
        -   如果实体打开了一个界面（例如村民交易 GUI 或箱子矿车 GUI），则流程结束。
        -   触发 `PlayerInteractEvent.EntityInteract` 事件。如果事件被取消，则流程结束。
        -   在**你正在看的实体上**调用 `Entity#interact`。如果它返回一个确定性结果，则流程结束。
            -   如果你想为自己的实体添加行为，请重写此方法。如果你想为原版实体添加行为，请使用事件。
            -   对于[`Mob`][livingentity]，`Entity#interact` 的覆写方法会处理诸如拴绳、当主手中的 `ItemStack` 是刷怪蛋时生下幼崽等事情，然后将特定于生物的处理委托给 `Mob#mobInteract`。`Entity#interact` 的结果规则在这里也同样适用。
        -   如果你正在看的实体是 `LivingEntity`，则会在你主手中的 `ItemStack` 上调用 `Item#interactLivingEntity`。如果它返回一个确定性结果，则流程结束。
    -   如果你正在看一个在你触及范围内且未超出世界边界的[方块][block]：
        -   触发 `PlayerInteractEvent.RightClickBlock` 事件。如果事件被取消，则流程结束。你也可以在此事件中明确地只拒绝方块或物品的使用。
        -   调用 `IItemExtension#onItemUseFirst`。如果它返回一个确定性结果，则流程结束。
        -   如果玩家没有潜行并且事件没有拒绝方块使用，则触发 `UseItemOnBlockEvent` 事件。如果事件被取消，则使用取消结果。否则，调用 `BlockBehaviour#useItemOn`。如果它返回一个确定性结果，则流程结束。
        -   如果 `InteractionResult` 是 `TRY_WITH_EMPTY_HAND` 并且执行手是主手，则调用 `BlockBehaviour#useWithoutItem`。如果它返回一个确定性结果，则流程结束。
        -   如果事件没有拒绝物品使用，则调用 `Item#useOn`。如果它返回一个确定性结果，则流程结束。
-   调用 `Item#use`。如果它返回一个确定性结果，则流程结束。
-   上述过程会再次运行，但这次是副手而不是主手。

### `InteractionResult`

`InteractionResult` 是一个封闭接口，表示物品或空手与某些对象（例如实体、方块等）之间某种交互的结果。该接口被分解为四个记录类，共有六个可能的默认状态。

首先是 `InteractionResult.Success`，它表示操作应被视为成功，从而结束流程。一个成功的状态有两个参数：`SwingSource`，它指示实体是否应该在相应的[逻辑侧][side]挥动手臂；以及 `InteractionResult.ItemContext`，它记录了交互是否由手持物品引起，以及使用后手持物品变成了什么。挥动来源由其中一个默认状态确定：`InteractionResult#SUCCESS` 用于客户端挥动，`InteractionResult#SUCCESS_SERVER` 用于服务器端挥动，而 `InteractionResult#CONSUME` 则表示不挥动。如果 `ItemStack` 发生了变化，则通过 `Success#heldItemTransformedTo` 设置物品上下文，如果手持物品与对象之间没有交互，则通过 `withoutItem` 设置。默认情况下，认为有物品交互但没有转化。

```java
// 在某个返回交互结果的方法中

// 手中的物品会变成一个苹果
return InteractionResult.SUCCESS.heldItemTransformedTo(new ItemStack(Items.APPLE));
```

:::note
`SUCCESS` 和 `SUCCESS_SERVER` 通常不应在同一个方法中使用。如果客户端有足够的信息来决定何时挥动手臂，那么应始终使用 `SUCCESS`。否则，如果它依赖于客户端上不存在的服务器信息，则应使用 `SUCCESS_SERVER`。
:::

然后是 `InteractionResult.Fail`，由 `InteractionResult#FAIL` 实现，它表示操作应被视为失败，不允许发生进一步的交互。流程将结束。这可以在任何地方使用，但在 `Item#useOn` 和 `Item#use` 之外应谨慎使用。在许多情况下，使用 `InteractionResult#PASS` 更有意义。

最后是 `InteractionResult.Pass` 和 `InteractionResult.TryWithEmptyHandInteraction`，分别由 `InteractionResult#PASS` 和 `InteractionResult#TRY_WITH_EMPTY_HAND` 实现。这些记录类表示当一个操作应被视为既不成功也不失败时，流程应继续。`PASS` 是所有 `InteractionResult` 方法的默认行为，除了 `BlockBehaviour#useItemOn`，它返回 `TRY_WITH_EMPTY_HAND`。更具体地说，如果 `BlockBehaviour#useItemOn` 返回的不是 `TRY_WITH_EMPTY_HAND`，那么无论物品是否在主手，`BlockBehaviour#useWithoutItem` 都不会被调用。

有些方法有特殊的行为或要求，将在下面的章节中解释。

#### `Item#useOn`

如果你希望操作被视为成功，但你不想让手臂挥动或获得 `ITEM_USED` 统计点数，请使用 `InteractionResult#CONSUME` 并调用 `#withoutItem`。

```java
// 在 Item#useOn 中
return InteractionResult.CONSUME.withoutItem();
```

<h4><code>Item#use</code></h4>

这是唯一一个从 `Success` 变体 (`SUCCESS`, `SUCCESS_SERVER`, `CONSUME`) 中使用转换后的 `ItemStack` 的实例。由 `Success#heldItemTransformedTo` 设置的最终 `ItemStack` 会替换发起使用的 `ItemStack`，如果它发生了变化。

`Item#use` 的默认实现，当物品可食用（具有 `DataComponents#CONSUMABLE`）且玩家可以吃掉该物品（因为他们饿了，或者因为该物品总是可以食用）时返回 `InteractionResult#CONSUME`，当物品可食用但玩家不能吃掉该物品时返回 `InteractionResult#FAIL`。如果物品可装备（具有 `DataComponents#EQUIPPABLE`），则在与手持物品交换时返回 `InteractionResult#SUCCESS` 并将手持物品替换为交换的物品（通过 `heldItemTransformedTo`），或者如果盔甲上的附魔具有 `EnchantmentEffectComponents#PREVENT_ARMOR_CHANGE` 组件，则返回 `InteractionResult#FAIL`。否则返回 `InteractionResult#PASS`。

在考虑主手的情况下，此处返回 `InteractionResult#FAIL` 将阻止副手行为的运行。如果你希望副手行为运行（通常你都希望如此），请返回 `InteractionResult#PASS`。

## 中键点击

-   如果 `Minecraft.getInstance().hitResult` 中的 [`HitResult`][hitresult] 为 null 或类型为 `MISS`，则流程结束。
-   使用鼠标左键和主手触发 `InputEvent.InteractionKeyMappingTriggered` 事件。如果[事件][event]被[取消][cancel]，则流程结束。
-   根据你正在看的东西（使用 `Minecraft.getInstance().hitResult` 中的 `HitResult`），会发生不同的事情：
    -   如果你正在看一个在你触及范围内的[实体][entity]：
        -   如果 `Entity#isPickable` 返回 false，则流程结束。
        -   如果 `Player#canInteractWithEntity` 返回 false，则流程结束。
        -   调用 `Entity#getPickResult`。如果存在与生成的 `ItemStack` 匹配的热键栏槽位，则该槽位被激活。否则，如果玩家处于创造模式，则生成的 `ItemStack` 将被添加到玩家的物品栏中。
            -   默认情况下，此方法会转发到 `Entity#getPickResult`，mod 开发者可以重写该方法。
    -   如果你正在看一个在你触及范围内的[方块][block]：
        -   调用 `IBlockExtension#getCloneItemStack`（默认情况下委托给 `BlockBehaviour#getCloneItemStack`）并成为“选定的” `ItemStack`。
            -   默认情况下，这会返回该 `Block` 的 `Item` 表示。
        -   如果按住 Control 键，玩家处于创造模式并且目标方块有一个[`方块实体`][blockentity]：
            -   通过 `BlockEntity#saveCustomOnly` 获取`方块实体` 的数据
                -   `BlockEntity#removeComponentsFromTag` 作为后处理步骤被调用。
            -   通过 `DataComponents#BLOCK_ENTITY_DATA` 将`方块实体`的数据添加到“选定的”`ItemStack`中。
        -   如果存在与“选定的” `ItemStack` 匹配的热键栏槽位，则该槽位被激活。否则，如果玩家处于创造模式，则“选定的”`ItemStack`将被添加到玩家的物品栏中。

[attribute]: ../entities/attributes.md
[attributemodifier]: ../entities/attributes.md#attribute-modifiers
[block]: ../blocks/index.md
[blockbreak]: ../blocks/index.md#breaking-a-block
[blockentity]: ../blockentities/index.md
[cancel]: ../concepts/events.md#cancellable-events
[critical]: https://minecraft.wiki/w/Damage#Critical_hit
[effect]: mobeffects.md
[entity]: ../entities/index.md
[event]: ../concepts/events.md
[featureflag]: ../advanced/featureflags.md
[hitresult]: #hitresults
[hurt]: ../entities/index.md#damaging-entities
[itemstack]: index.md#itemstacks
[itemuseon]: #itemuseon
[livingentity]: ../entities/livingentity.md
[physicalside]: ../concepts/sides.md#the-physical-side
[side]: ../concepts/sides.md#the-logical-side