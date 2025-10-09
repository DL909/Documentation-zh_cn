---
sidebar_position: 6
---
# 生物效果与药水

状态效果，有时被称为药水效果，在代码中称为 `MobEffect`，是每刻都会影响一个[`LivingEntity`][livingentity] 的效果。本文将解释如何使用它们，效果和药水之间的区别，以及如何添加您自己的效果。

## 术语

- `MobEffect` 每时每刻都会影响一个实体。和[方块][block]或[物品][item]一样，`MobEffect` 是注册表对象，意味着它们必须被[注册][registration]并且是单例。
    - **瞬间生物效果** 是一种特殊的生物效果，设计为施加一刻。原版有两种瞬间效果：瞬间治疗和瞬间伤害。
- `MobEffectInstance` 是一个 `MobEffect` 的实例，具有持续时间、增幅和其他一些属性（见下文）。`MobEffectInstance` 之于 `MobEffect` 就如同[`ItemStack`][itemstack]之于 `Item`。
- `Potion` 是一个 `MobEffectInstance` 的集合。原版主要将药水用于四种药水物品（继续阅读），但是，它们可以随意应用于任何物品。物品本身决定是否以及如何使用其上设置的药水。
- **药水物品** 是指打算在其上设置药水的物品。这是一个非正式术语，原版的 `PotionItem` 类与此无关（它指的是“普通”药水物品）。Minecraft 目前有四种药水物品：药水、喷溅药水、滞留药水和药箭；不过模组可能会添加更多。

## `MobEffect`

要创建你自己的 `MobEffect`，请扩展 `MobEffect` 类：

```java
public class MyMobEffect extends MobEffect {
    public MyMobEffect(MobEffectCategory category, int color) {
        super(category, color);
    }
  
    @Override
    public boolean applyEffectTick(ServerLevel level, LivingEntity entity, int amplifier) {
        // Apply your effect logic here.

        // If this returns false when shouldApplyEffectTickThisTick returns true, the effect will immediately be removed
        return true;
    }
  
    // Whether the effect should apply this tick. Used e.g. by the Regeneration effect that only applies
    // once every x ticks, depending on the tick count and amplifier.
    @Override
    public boolean shouldApplyEffectTickThisTick(int tickCount, int amplifier) {
        return tickCount % 2 == 0; // replace this with whatever check you want
    }
  
    // Utility method that is called when the effect is first added to the entity.
    // This does not get called again until all instances of this effect have been removed from the entity.
    @Override
    public void onEffectAdded(LivingEntity entity, int amplifier) {
        super.onEffectAdded(entity, amplifier);
    }

    // Utility method that is called when the effect is added to the entity.
    // This gets called every time this effect is added to the entity.
    @Override
    public void onEffectStarted(LivingEntity entity, int amplifier) {
    }
}
```

与所有注册表对象一样，`MobEffect` 必须被[注册][registration]，如下所示：

```java
// MOB_EFFECTS is a DeferredRegister<MobEffect>
public static final Holder<MobEffect> MY_MOB_EFFECT = MOB_EFFECTS.register("my_mob_effect", () -> new MyMobEffect(
        //Can be either BENEFICIAL, NEUTRAL or HARMFUL. Used to determine the potion tooltip color of this effect.
        MobEffectCategory.BENEFICIAL,
        //The color of the effect particles in RGB format.
        0xffffff
));
```

`MobEffect` 类还提供了向受影响实体添加[属性修饰符][attributemodifier]的默认功能，并在效果过期或通过其他方式移除时移除它们。例如，速度效果会为移动速度添加一个属性修饰符。效果属性修饰符的添加方式如下：

```java
public static final Holder<MobEffect> MY_MOB_EFFECT = MOB_EFFECTS.register("my_mob_effect", () -> new MyMobEffect(...)
        .addAttributeModifier(Attributes.ATTACK_DAMAGE, ResourceLocation.fromNamespaceAndPath("examplemod", "effect.strength"), 2.0, AttributeModifier.Operation.ADD_VALUE)
);
```

### `InstantenousMobEffect`

如果你想创建一个瞬间效果，你可以使用辅助类 `InstantenousMobEffect` 而不是常规的 `MobEffect` 类，就像这样：

```java
public class MyMobEffect extends InstantenousMobEffect {
    public MyMobEffect(MobEffectCategory category, int color) {
        super(category, color);
    }

    @Override
    public void applyEffectTick(ServerLevel level, LivingEntity entity, int amplifier) {
        // Apply your effect logic here.
    }
}
```

然后，像平常一样[注册][registration]你的效果。

### 事件

许多效果的逻辑是在其他地方应用的。例如，漂浮效果是在生物实体的移动处理器中应用的。对于模组添加的 `MobEffect`，将它们应用在[事件处理器][events]中通常是有意义的。NeoForge 还提供了一些与效果相关的事件：

- `MobEffectEvent.Applicable` 在游戏检查 `MobEffectInstance` 是否可以应用于实体时触发。此事件可用于拒绝或强制将效果实例添加到目标。
- `MobEffectEvent.Added` 在 `MobEffectInstance` 添加到目标时触发。此事件包含有关目标上可能存在的先前 `MobEffectInstance` 的信息。
- `MobEffectEvent.Expired` 在 `MobEffectInstance` 过期时触发，即计时器归零时。
- `MobEffectEvent.Remove` 在效果通过除过期以外的方式（例如喝牛奶或通过命令）从实体中移除时触发。

## `MobEffectInstance`

简单来说，`MobEffectInstance` 就是应用到实体上的一个效果。创建一个 `MobEffectInstance` 是通过调用构造函数来完成的：

```java
MobEffectInstance instance = new MobEffectInstance(
        // The mob effect to use.
        MobEffects.REGENERATION,
        // The duration to use, in ticks. Defaults to 0 if not specified.
        500,
        // The amplifier to use. This is the "strength" of the effect, i.e. Strength I, Strength II, etc.
        // Must be between 0 and 255 (inclusive). Defaults to 0 if not specified.
        0,
        // Whether the effect is an "ambient" effect, meaning it is being applied by an ambient source,
        // of which Minecraft currently has the beacon and the conduit. Defaults to false if not specified.
        false,
        // Whether the effect is visible in the inventory. Defaults to true if not specified.
        true,
        // Whether an effect icon is visible in the top right corner. Defaults to true if not specified.
        true
);
```

有几个构造函数重载可用，分别省略了最后 1-5 个参数。

:::info
`MobEffectInstance` 是可变的。如果你需要一个副本，请调用 `new MobEffectInstance(oldInstance)`。
:::

### 使用 `MobEffectInstance`

一个 `MobEffectInstance` 可以像这样添加到 `LivingEntity` 中：

```java
MobEffectInstance instance = new MobEffectInstance(...);
livingEntity.addEffect(instance);
```

同样，`MobEffectInstance` 也可以从 `LivingEntity` 中移除。由于一个 `MobEffectInstance` 会覆盖实体上已存在的相同 `MobEffect` 的 `MobEffectInstance`，因此每个 `MobEffect` 和实体只能有一个 `MobEffectInstance`。因此，移除时只需指定 `MobEffect` 即可：

```java
livingEntity.removeEffect(MobEffects.REGENERATION);
```

:::info
`MobEffect` 只能应用于 `LivingEntity` 或其子类，即玩家和生物。像物品或投掷的雪球这样的东西不能被 `MobEffect` 影响。
:::

## `Potion`

`Potion` 是通过调用 `Potion` 的构造函数并传入你希望药水拥有的 `MobEffectInstance` 来创建的。例如：

```java
//POTIONS is a DeferredRegister<Potion>
public static final Holder<Potion> MY_POTION = POTIONS.register("my_potion", registryName -> new Potion(
    // The suffix applied to the potion
    registryName.getPath(),
    // The effects used by the potion
    new MobEffectInstance(MY_MOB_EFFECT, 3600)
));
```

药水的名称是第一个构造函数参数。它被用作翻译键的后缀；例如，原版中的长效和强效药水变体使用它来与它们的基础变体拥有相同的名称。

`new Potion` 的 `MobEffectInstance` 参数是一个可变参数。这意味着你可以向药水中添加任意多个效果。这也意味着可以创建空药水，即没有任何效果的药水。只需调用 `new Potion()` 即可！（顺便说一下，这就是原版添加 `awkward` 药水的方式。）

`PotionContents` 类提供了与药水物品相关的各种辅助方法。药水物品通过 `DataComponent#POTION_CONTENTS` 存储其 `PotionContents`。

### 酿造

现在你的药水已经添加，药水物品也对你的药水可用了。但是，在生存模式中没有办法获得你的药水，所以让我们来改变这一点！

药水传统上是在酿造台中制作的。不幸的是，Mojang 不提供对酿造配方的[数据包][datapack]支持，所以我们必须用稍微老式的方法，通过 `RegisterBrewingRecipesEvent` 事件用代码添加我们的配方。做法如下：

```java
@SubscribeEvent // on the game event bus
public static void registerBrewingRecipes(RegisterBrewingRecipesEvent event) {
    // Gets the builder to add recipes to
    PotionBrewing.Builder builder = event.getBuilder();

    // Will add brewing recipes for all container potions (e.g. potion, splash potion, lingering potion)
    builder.addMix(
        // The initial potion to apply to
        Potions.AWKWARD,
        // The brewing ingredient. This is the item at the top of the brewing stand.
        Items.FEATHER,
        // The resulting potion
        MY_POTION
    );
}
```

[attributemodifier]: ../entities/attributes.md#attribute-modifiers
[block]: ../blocks/index.md
[commonsetup]: ../concepts/events.md#event-buses
[datapack]: ../resources/index.md#data
[events]: ../concepts/events.md
[item]: index.md
[itemstack]: index.md#itemstacks
[livingentity]: ../entities/livingentity.md
[registration]: ../concepts/registries.md#methods-for-registering
[uuidgen]: https://www.uuidgenerator.net/version4