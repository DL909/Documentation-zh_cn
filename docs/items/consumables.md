---
sidebar_position: 3
---
# 消耗品

消耗品是指可以在一段时间内使用，并在该过程中被“消耗”掉的[物品][item]。在 Minecraft 中任何可以吃或喝的东西都属于某种消耗品。

## `Consumable` 数据组件

任何可被消耗的物品都具有[`DataComponents#CONSUMABLE` 组件][datacomponent]。其背后的记录类 `Consumable` 定义了物品如何被消耗以及消耗后应用何种效果。

可以通过直接调用记录类构造函数或通过 `Consumable#builder` 来创建 `Consumable`，后者会为每个字段设置默认值，完成后再调用 `build`：

- `consumeSeconds` - 一个 `float` 类型，表示完全消耗该物品所需的秒数。经过指定时间后，会调用 `Item#finishUsingItem`。默认为 1.6 秒，即 32 刻。
- `animation` - 设置在使用物品时播放的 [`ItemUseAnimation`][animation]。默认为 `ItemUseAnimation#EAT`。
- `sound` - 设置在消耗物品时播放的 [`SoundEvent`][sound]。这必须是一个 `Holder` 实例。默认为 `SoundEvents#GENERIC_EAT`。
    - 如果一个原版实例不是 `Holder<SoundEvent>`，可以通过调用 `BuiltInRegistries.SOUND_EVENT.wrapAsHolder(soundEvent)` 来获取一个包装为 `Holder` 的版本。
- `soundAfterConsume` - 设置物品消耗完毕后播放的 [`SoundEvent`][sound]。这会委托给 [`PlaySoundConsumeEffect`][consumeeffect]。
- `hasConsumeParticles` - 当为 `true` 时，每四刻生成一次物品[粒子][particles]，并在物品完全消耗后再次生成。默认为 `true`。
- `onConsume` - 添加一个[`ConsumeEffect`][consumeeffect]，在物品通过 `Item#finishUsingItem` 完全消耗后应用。

原版在 `Consumables` 类中提供了一些消耗品，例如用于[食物][food]物品的 `#defaultFood` 和用于[药水][potions]和牛奶桶的 `#defaultDrink`。

可以通过调用 `Item.Properties#component` 来添加 `Consumable` 组件：

```java
// Assume there is some DeferredRegister.Items ITEMS
public static final DeferredItem<Item> CONSUMABLE = ITEMS.registerSimpleItem(
    "consumable",
    new Item.Properties().component(
        DataComponents.CONSUMABLE,
        Consumable.builder()
            // Spend 2 seconds, or 40 ticks, to consume
            .consumeSeconds(2f)
            // Sets the animation to play while consuming
            .animation(ItemUseAnimation.BLOCK)
            // Play sound while consuming every tick
            .sound(SoundEvents.ARMOR_EQUIP_CHAIN)
            // Play sound once finished consuming
            .soundAfterConsume(SoundEvents.BREEZE_WIND_CHARGE_BURST)
            // Don't show particles while eating
            .hasConsumeParticles(false)
            .onConsume(
                // When finished consuming, applies the effects with a 30% chance
                new ApplyStatusEffectsConsumeEffect(new MobEffectInstance(MobEffects.HUNGER, 600, 0), 0.3F)
            )
            // Can have multiple
            .onConsume(
                // Teleports the entity randomly in a 50 block radius
                new TeleportRandomlyConsumeEffect(100f)
            )
            .build()
    )
);
```

### `ConsumeEffect`

当一个消耗品使用完毕后，你可能想触发某种逻辑来执行，比如添加一个药水效果。这些都由 `ConsumeEffect` 处理，通过调用 `Consumable.Builder#onConsume` 添加到 `Consumable` 中。

原版效果的列表可以在 `ConsumeEffect` 中找到。

每个 `ConsumeEffect` 都有两个方法：`getType`，它指定注册表对象 `ConsumeEffect.Type`；以及 `apply`，当物品完全消耗时会在物品上调用该方法。`apply` 接受三个参数：消耗实体所在的 `Level`，被调用的消耗品的 `ItemStack`，以及消耗该对象的 `LivingEntity`。当效果成功应用时，该方法返回 `true`，如果失败则返回 `false`。

可以通过实现该接口并将带有相关 `MapCodec` 和 `StreamCodec` 的 `ConsumeEffect.Type` [注册][registering]到 `BuiltInRegistries#CONSUME_EFFECT_TYPE` 来创建 `ConsumeEffect`：

```java
public record UsePortalConsumeEffect(ResourceKey<Level> level)
    implements ConsumeEffect, Portal {

    @Override
    public boolean apply(Level level, ItemStack stack, LivingEntity entity) {
        if (entity.canUsePortal(false)) {
            entity.setAsInsidePortal(this, entity.blockPosition());

            // Can successfully use portal
            return true;
        }

        // Cannot use portal
        return false;
    }

    @Override
    public ConsumeEffect.Type<? extends ConsumeEffect> getType() {
        // Set to registered object
        return USE_PORTAL.get();
    }

    @Override
    @Nullable
    public TeleportTransition getPortalDestination(ServerLevel level, Entity entity, BlockPos pos) {
        // Set teleport location
    }
}

// In some registrar class
// Assume there is some DeferredRegister<ConsumeEffect.Type<?>> CONSUME_EFFECT_TYPES
public static final Supplier<ConsumeEffect.Type<UsePortalConsumeEffect>> USE_PORTAL =
    CONSUME_EFFECT_TYPES.register("use_portal", () -> new ConsumeEffect.Type<>(
        ResourceKey.codec(Registries.DIMENSION).optionalFieldOf("dimension")
            .xmap(UsePortalConsumeEffect::new, UsePortalConsumeEffect::level),
        ResourceKey.streamCodec(Registries.DIMENSION)
            .map(UsePortalConsumeEffect::new, UsePortalConsumeEffect::level)
    ));

// For some Item.Properties that is adding a CONSUMABLE component
Consumable.builder()
    .onConsume(
        new UsePortalConsumeEffect(Level.END)
    )
    .build();
```

### `ItemUseAnimation`

`ItemUseAnimation` 在功能上是一个枚举，除了其 id 和名称外，没有定义任何东西。它的用途被硬编码在 `ItemHandRenderer#renderArmWithItem`（用于第一人称）和 `PlayerRenderer#getArmPose`（用于第三人称）中。因此，仅仅创建一个新的 `ItemUseAnimation` 只会起到类似于 `ItemUseAnimation#NONE` 的作用。

要应用某种动画，你需要为第一人称实现 `IClientItemExtensions#applyForgeHandTransform` 和/或为第三人称渲染实现 `IClientItemExtensions#getArmPose`。

#### 创建 `ItemUseAnimation`

首先，让我们创建一个新的 `ItemUseAnimation`。这是通过[可扩展枚举][extensibleenum]系统完成的：

```json5
{
    "entries": [
        {
            "enum": "net/minecraft/world/item/ItemUseAnimation",
            "name": "EXAMPLEMOD_ITEM_USE_ANIMATION",
            "constructor": "(ILjava/lang/String;)V",
            "parameters": [
                // The id, should always be -1
                -1,
                // The name, should be a unique identifier
                "examplemod:item_use_animation"
            ]
        }
    ]
}
```

然后我们可以通过 `valueOf` 获取枚举常量：

```java
public static final ItemUseAnimation EXAMPLE_ANIMATION = ItemUseAnimation.valueOf("EXAMPLEMOD_ITEM_USE_ANIMATION");
```

从那里，我们就可以开始应用变换了。为此，我们必须创建一个新的 `IClientItemExtensions`，实现我们想要的方法，并通过[**模组事件总线**][modbus]上的 `RegisterClientExtensionsEvent` 注册它：

```java
public class ConsumableClientItemExtensions implements IClientItemExtensions {
    // Implement methods here
}

// In some event handler class
@SubscribeEvent // on the mod event bus only on the physical client
public static void registerClientExtensions(RegisterClientExtensionsEvent event) {
    event.registerItem(
        // The instance of the item extensions
        new ConsumableClientItemExtensions(),
        // A vararg of items that use this
        CONSUMABLE
    )
}
```

#### 第一人称

第一人称变换是所有消耗品都具有的，通过 `IClientItemExtensions#applyForgeHandTransform` 实现：

```java
public class ConsumableClientItemExtensions implements IClientItemExtensions {

    // ...

    @Override
    public boolean applyForgeHandTransform(
        PoseStack poseStack, LocalPlayer player, HumanoidArm arm, ItemStack itemInHand,
        float partialTick, float equipProcess, float swingProcess
    ) {
        // We first need to check if the item is being used and has our animation
        HumanoidArm usingArm = entity.getUsedItemHand() == InteractionHand.MAIN_HAND
            ? entity.getMainArm()
            : entity.getMainArm().getOpposite();
        if (
            entity.isUsingItem() && entity.getUseItemRemainingTicks() > 0
            && usingArm == arm && itemInHand.getUseAnimation() == EXAMPLE_ANIMATION
        ) {
            // Apply transformations to pose stack (translate, scale, mulPose)
            // ...
            return true;
        }

        // Do nothing
        return false;
    }
}
```

#### 第三视角

第三人称变换，除了 `EAT` 和 `DRINK` 之外都有特殊的逻辑，通过 `IClientItemExtensions#getArmPose` 实现，其中 `HumanoidModel.ArmPose` 也可以被扩展以实现自定义变换。

由于 `ArmPose` 的构造函数需要一个 lambda 作为其一部分，因此必须使用 `EnumProxy` 引用：

```json5
{
    "entries": [
        {
            "name": "EXAMPLEMOD_ITEM_USE_ANIMATION",
            // ...
        },
        {
            "enum": "net/minecraft/client/model/HumanoidModel$ArmPose",
            "name": "EXAMPLEMOD_ARM_POSE",
            "constructor": "(ZLnet/neoforged/neoforge/client/IArmPoseTransformer;)V",
            "parameters": {
                // Point to class where the proxy is located
                // Should be separate as this is a client only class
                "class": "example/examplemod/client/MyClientEnumParams",
                // The field name of the enum proxy
                "field": "CUSTOM_ARM_POSE"
            }
        }
    ]
}
```

```java
// Create the enum parameters
public class MyClientEnumParams {
    public static final EnumProxy<HumanoidModel.ArmPose> CUSTOM_ARM_POSE = new EnumProxy<>(
        HumanoidModel.ArmPose.class,
        // Whether the pose uses both arms
        false,
        // The pose transformer
        (IArmPoseTransformer) MyClientEnumParams::applyCustomModelPose
    );

    private static void applyCustomModelPose(
        HumanoidModel<?> model, HumanoidRenderState state, HumanoidArm arm
    ) {
        // Apply model transforms here
        // ...
    }
}

// In some client only class
public static final HumanoidModel.ArmPose EXAMPLE_POSE = HumanoidModel.ArmPose.valueOf("EXAMPLEMOD_ARM_POSE");
```

然后，通过 `IClientItemExtensions#getArmPose` 设置手臂姿势：

```java
public class ConsumableClientItemExtensions implements IClientItemExtensions {

    // ...

    @Override
    public HumanoidModel.ArmPose getArmPose(
        LivingEntity entity, InteractionHand hand, ItemStack stack
    ) {
        // We first need to check if the item is being used and has our animation
        if (
            entity.isUsingItem() && entity.getUseItemRemainingTicks() > 0
            && entity.getUsedItemHand() == hand
            && itemInHand.getUseAnimation() == EXAMPLE_ANIMATION
        ) {
            // Return pose to apply
            return EXAMPLE_POSE;
        }

        // Otherwise return null
        return null;
    }
}
```

### 在实体上覆盖声音

有时，一个实体可能想在消耗物品时播放不同的声音。在这种情况下，[`LivingEntity`][livingentity] 实例可以实现 `Consumable.OverrideConsumeSound` 并让 `getConsumeSound` 返回他们希望其实体播放的 `SoundEvent`。

```java
public class MyEntity extends LivingEntity implements Consumable.OverrideConsumeSound {
  
    // ...

    @Override
    public SoundEvent getConsumeSound(ItemStack stack) {
        // Return the sound to play
    }
}
```

## `ConsumableListener`

虽然消耗品和消耗后应用的效果很有用，但有时效果的属性需要作为其他[数据组件][datacomponents]对外可用。例如，猫和狼也吃[食物][food]并查询其营养值，或者带有药水内容的物品会查询其颜色用于渲染。在这些情况下，数据组件会实现 `ConsumableListener` 以提供消耗逻辑。

`ConsumableListener` 只有一个方法：`#onConsume`，它接收当前的世界、消耗物品的实体、被消耗的物品以及物品上的 `Consumable` 实例。当物品完全被消耗时，`#onConsume` 会在 `Item#finishUsingItem` 期间被调用。

添加你自己的 `ConsumableListener` 只需[注册一个新的数据组件][datacompreg]并实现 `ConsumableListener`。

```java
public record MyConsumableListener() implements ConsumableListener {

    @Override
    public void onConsume(
        Level level, LivingEntity entity, ItemStack stack, Consumable consumable
    ) {
        // Do things here
    }
}
```

### 食物

食物是饥饿系统中的一种 `ConsumableListener`。食物物品的所有功能都已在 `Item` 类中处理，因此只需将 `FoodProperties` 添加到 `DataComponents#FOOD` 并附带一个可消耗品即可。有一个名为 `food` 的辅助方法，它接收 `FoodProperties` 和 `Consumable` 对象，如果未指定，则使用 `Consumables#DEFAULT_FOOD`。

`FoodProperties` 可以通过直接调用记录类构造函数或通过 `new FoodProperties.Builder()` 创建，完成后再调用 `build`：

- `nutrition` - 设置恢复多少饥饿值。以半个饥饿点为单位计算，例如，Minecraft 的牛排恢复 8 点饥饿值。
- `saturationModifier` - 用于计算吃下该食物时恢复的[饱和度值][hunger]的饱和度修饰符。计算公式为 `min(2 * nutrition * saturationModifier, playerNutrition)`，这意味着使用 `0.5` 会使有效的饱和度值与营养值相同。
- `alwaysEdible` - 该物品是否总是可以食用，即使饥饿条已满。默认为 `false`，金苹果和其他提供超出填饱肚子之外奖励的物品为 `true`。

```java
// Assume there is some DeferredRegister.Items ITEMS
public static final DeferredItem<Item> FOOD = ITEMS.registerSimpleItem(
    "food",
    new Item.Properties().food(
        new FoodProperties.Builder()
            // Heals 1.5 hearts
            .nutrition(3)
            // Carrot is 0.3
            // Raw Cod is 0.1
            // Cooked Chicken is 0.6
            // Cooked Beef is 0.8
            // Golden Aple is 1.2
            .saturationModifier(0.3f)
            // When set, the food can alway be eaten even with
            //  a full hunger bar.
            .alwaysEdible()
    )
);
```

如需示例，或查看 Minecraft 使用的各种值，请查看 `Foods` 类。

要获取物品的 `FoodProperties`，请调用 `ItemStack.get(DataComponents.FOOD)`。这可能返回 null，因为并非所有物品都是可食用的。要确定物品是否可食用，请对 `getFoodProperties` 调用的结果进行 null 检查。

### 药水内容物

通过 `PotionContents` 获取的[药水][potions]内容物是另一个 `ConsumableListener`，其效果在消耗时应用。它们包含一个可选的要应用的药水、一个可选的药水颜色染色、一个与药水一同应用的自定义 [`MobEffectInstance`][mobeffectinstance] 列表，以及一个在获取物品堆名称时使用的可选翻译键。如果不是 `PotionItem` 的子类型，modder 需要重写 `Item#getName`。

[animation]: #itemuseanimation
[consumeeffect]: #consumeeffect
[datacomponent]: datacomponents.md
[datacompreg]: datacomponents.md#creating-custom-data-components
[extensibleenum]: ../advanced/extensibleenums.md
[food]: #food
[hunger]: https://minecraft.wiki/w/Hunger#Mechanics
[item]: index.md
[livingentity]: ../entities/livingentity.md
[modbus]: ../concepts/events.md#event-buses
[mobeffectinstance]: mobeffects.md#mobeffectinstances
[particles]: ../resources/client/particles.md
[potions]: mobeffects.md#potions
[sound]: ../resources/client/sounds.md#creating-soundevents
[registering]: ../concepts/registries.md#methods-for-registering