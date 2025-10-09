---
sidebar_position: 5
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 盔甲

盔甲是[物品][item]，其主要用途是利用各种抗性和效果保护[`LivingEntity`][livingentity]免受伤害。许多模组都添加了新的盔甲套装（例如铜甲）。

## 自定义盔甲套装

一套供人形生物使用的盔甲通常由四件物品组成：头部的头盔、胸部的胸甲、腿部的护腿和脚部的靴子。此外，还有专为狼、马和羊驼设计的盔甲，它们被装备在动物专用的“身体”盔甲槽中。所有这些物品通常通过七个[数据组件][datacomponents]来实现：

- `DataComponents#MAX_DAMAGE` 和 `#DAMAGE` 用于耐久度
- `#MAX_STACK_SIZE` 用于将堆叠大小设置为 `1`
- `#REPAIRABLE` 用于在铁砧中修复盔甲
- `#ENCHANTABLE` 用于最大[附魔][enchantment]值
- `#ATTRIBUTE_MODIFIERS` 用于盔甲、盔甲韧性和击退抗性
- `#EQUIPPABLE` 用于实体如何装备此物品。

通常，每件盔甲都使用 `Item.Properties#humanoidArmor`（用于人形生物）、`wolfArmor`（用于狼）和 `horseArmor`（用于马）进行设置。它们都使用 `ArmorMaterial` 结合人形生物的 `ArmorType` 来设置组件。参考值可以在 `ArmorMaterials` 中找到。本例使用铜盔甲材质，您可以根据需要调整其值。

```java
public static final ArmorMaterial COPPER_ARMOR_MATERIAL = new ArmorMaterial(
    // The durability multiplier of the armor material.
    // ArmorType have different unit durabilities that the multiplier is applied to:
    // - HELMET: 11
    // - CHESTPLATE: 16
    // - LEGGINGS: 15
    // - BOOTS: 13
    // - BODY: 16
    15,
    // Determines the defense value (or the number of half-armors on the bar).
    // Based on ArmorType.
    Util.make(new EnumMap<>(ArmorType.class), map -> {
        map.put(ArmorItem.Type.BOOTS, 2);
        map.put(ArmorItem.Type.LEGGINGS, 4);
        map.put(ArmorItem.Type.CHESTPLATE, 6);
        map.put(ArmorItem.Type.HELMET, 2);
        map.put(ArmorItem.Type.BODY, 4);
    }),
    // Determines the enchantability of the armor. This represents how good the enchantments on this armor will be.
    // Gold uses 25; we put copper slightly below that.
    20,
    // Determines the sound played when equipping this armor.
    // This is wrapped with a Holder.
    SoundEvents.ARMOR_EQUIP_GENERIC,
     // Returns the toughness value of the armor. The toughness value is an additional value included in
    // damage calculation, for more information, refer to the Minecraft Wiki's article on armor mechanics:
    // https://minecraft.wiki/w/Armor#Armor_toughness
    // Only diamond and netherite have values greater than 0 here, so we just return 0.
    0,
    // Returns the knockback resistance value of the armor. While wearing this armor, the player is
    // immune to knockback to some degree. If the player has a total knockback resistance value of 1 or greater
    // from all armor pieces combined, they will not take any knockback at all.
    // Only netherite has values greater than 0 here, so we just return 0.
    0,
    // The tag that determines what items can repair this armor.
    Tags.Items.INGOTS_COPPER,
    // The resource key of the EquipmentClientInfo JSON discussed below
    // Points to assets/examplemod/equipment/copper.json
    ResourceKey.create(EquipmentAssets.ROOT_ID, ResourceLocation.fromNamespaceAndPath("examplemod", "copper"))
);
```

现在我们有了 `ArmorMaterial`，我们可以用它来[注册][registering]盔甲：

```java
// ITEMS is a DeferredRegister.Items
public static final DeferredItem<Item> COPPER_HELMET = ITEMS.registerItem(
    "copper_helmet",
    props -> new Item(
        props.humanoidArmor(
            // The material to use.
            COPPER_ARMOR_MATERIAL,
            // The type of armor to create.
            ArmorType.HELMET
        )
    )
);

public static final DeferredItem<Item> COPPER_CHESTPLATE =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));
public static final DeferredItem<Item> COPPER_LEGGINGS =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));
public static final DeferredItem<Item> COPPER_BOOTS =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));

public static final DeferredItem<Item> COPPER_WOLF_ARMOR = ITEMS.registerItem(
    "copper_wolf_armor",
    props -> new Item(
        // The material to use.
        props.wolfArmor(COPPER_ARMOR_MATERIAL)
    )
);

public static final DeferredItem<Item> COPPER_HORSE_ARMOR =
    ITEMS.registerItem("copper_horse_armor", props -> new Item(props.horseArmor(...)));
```

如果你想从零开始创建盔甲或类似盔甲的物品，可以通过以下部分的组合来实现：

- 通过 `Item.Properties#component` 设置 `DataComponents#EQUIPPABLE` 来添加具有您自己要求的 `Equippable`。
- 通过 `Item.Properties#attributes` 向物品添加属性（例如盔甲、韧性、击退抗性）。
- 通过 `Item.Properties#durability` 添加物品耐久度。
- 通过 `Item.Properties#repariable` 允许物品被修复。
- 通过 `Item.Properties#enchantable` 允许物品被附魔。
- 将你的盔甲添加到 `minecraft:enchantable/*` `ItemTags` 中，以便你的物品可以应用某些附魔。

### `Equippable`

`Equippable` 是一个数据组件，包含实体如何装备此物品以及在游戏中如何处理渲染。这使得任何物品，无论是否被认为是“盔甲”，只要有此组件就可以被装备（例如，马鞍、羊驼上的地毯）。每个带有此组件的物品只能装备到一个 `EquipmentSlot` 中。

一个 `Equippable` 可以通过直接调用记录类构造函数或通过 `Equippable#builder` 创建，后者会为每个字段设置默认值，完成后再调用 `build`：

```java
// Assume there is some DeferredRegister.Items ITEMS
public static final DeferredItem<Item> EQUIPPABLE = ITEMS.registerSimpleItem(
    "equippable",
    new Item.Properties().component(
        DataComponents.EQUIPPABLE,
        // Sets the slot that this item can be equipped to.
        Equippable.builder(EquipmentSlot.HELMET)
            // Determines the sound played when equipping this item.
            // This is wrapped with a Holder.
            // Defaults to SoundEvents#ARMOR_EQUIP_GENERIC.
            .setEquipSound(SoundEvents.ARMOR_EQUIP_GENERIC)
            // The resource key of the EquipmentClientInfo JSON discussed below.
            // Points to assets/examplemod/equipment/equippable.json
            // When not set, does not render the equipment.
            .setAsset(ResourceKey.create(EquipmentAssets.ROOT_ID, ResourceLocation.fromNamespaceAndPath("examplemod", "equippable")))
            // The relative location of the texture to overlay on the player screen when wearing (e.g., pumpkin blur).
            // Points to assets/examplemod/textures/equippable.png
            // When not set, does not render an overlay.
            .setCameraOverlay(ResourceLocation.withDefaultNamespace("examplemod", "equippable"))
            // A HolderSet of entity types (direct or tag) that can equip this item.
            // When not set, any entity can equip this item.
            .setAllowedEntities(EntityType.ZOMBIE)
            // Whether the item can be equipped when dispensed from a dispenser.
            // Defaults to true.
            .setDispensable(true),
            // Whether the item can be swapped off the player during a quick equip.
            // Defaults to true.
            .setSwappable(false),
            // Whether the item should be damaged when attacked (for equipment typically).
            // Must also be a damageable item.
            // Defaults to true.
            .setDamageOnHurt(false)
            // Whether the item can be equipped onto another entity on interaction (e.g., right click).
            // Defaults to false.
            .setEquipOnInteract(true)
            // When true, an item with the SHEAR_REMOVE_ARMOR item ability can remove the equipped item.
            // Defaults to false.
            .setCanBeSheared(true)
            // The sound to play when shearing this equipped item.
            // This is wrapped with a holder.
            // Defaults to SoundEvents#SHEARS_SNIP.
            .setShearingSound(SoundEvents.SADDLE_UNEQUIP)
            .build()
    )
);
```

## 装备资源

现在我们在游戏中有了一些盔甲，但如果我们尝试穿上它，什么都不会渲染，因为我们从未指定如何渲染装备。为此，我们需要在 `Equippable#assetId` 指定的位置创建一个 `EquipmentClientInfo` JSON，该位置相对于[资源包][respack]的 `equipment` 文件夹（`assets` 文件夹）。`EquipmentClientInfo` 指定了用于渲染每个图层的相关纹理。

`EquipmentClientInfo` 在功能上是一个从 `EquipmentClientInfo.LayerType` 到要应用的 `EquipmentClientInfo.Layer` 列表的映射。

`LayerType` 可以被认为是一组用于某个实例渲染的纹理。例如，`LayerType#HUMANOID` 被 `HumanoidArmorLayer` 用于渲染人形实体上的头部、胸部和脚部；`LayerType#WOLF_BODY` 被 `WolfArmorLayer` 用于渲染身体盔甲。如果它们属于同一种可穿戴设备，比如铜甲，可以将它们组合成一个装备信息 JSON。

`LayerType` 映射到一些要应用和按顺序渲染纹理的 `Layer` 列表。一个 `Layer` 实际上代表一个要渲染的单个纹理。第一个参数代表纹理的位置，相对于 `textures/entity/equipment`。

第二个参数是一个可选参数，指示[纹理是否可以被染色][tinting]为一个 `EquipmentClientInfo.Dyeable`。`Dyeable` 对象持有一个整数，如果存在，则表示用于给纹理染色的默认 RGB 颜色。如果此可选参数不存在，则使用纯白色。

:::warning
为了使物品应用非未染色以外的颜色，该物品必须在 [`ItemTags#DYEABLE`][tag] 中，并且 `DataComponents#DYED_COLOR` 组件必须设置为某个 RGB 值。
:::

第三个参数是一个布尔值，指示是否应使用渲染期间提供的纹理，而不是 `Layer` 中定义的纹理。这方面的一个例子是玩家的自定义披风或自定义鞘翅纹理。

让我们为铜甲材料创建一个装备信息。我们还将假设每个图层都有两个纹理：一个用于实际的盔甲，另一个是覆盖并染色的。对于动物盔甲，我们将假设有一些动态纹理可以使用，可以传入。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// In assets/examplemod/equipment/copper.json
{
    // The layer map
    "layers": {
        // The serialized name of the EquipmentClientInfo.LayerType to apply.
        // For humanoid head, chest, and feet
        "humanoid": [
            // A list of layers to render in the order provided
            {
                // The relative texture of the armor
                // Points to assets/examplemod/textures/entity/equipment/humanoid/copper/outer.png
                "texture": "examplemod:copper/outer"
            },
            {
                // The overlay texture
                // Points to assets/examplemod/textures/entity/equipment/humanoid/copper/outer_overlay.png
                "texture": "examplemod:copper/outer_overlay",
                // When specified, allows the texture to be tinted the color in DataComponents#DYED_COLOR
                // Otherwise, cannot be tinted
                "dyeable": {
                    // An RGB value (always opaque color)
                    // 0x7683DE as decimal
                    // When not specified, set to 0 (meaning transparent or invisible)
                    "color_when_undyed": 7767006
                }
            }
        ],
        // For humanoid legs
        "humanoid_leggings": [
            {
                // Points to assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner.png
                "texture": "examplemod:copper/inner"
            },
            {
                // Points to assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner_overlay.png
                "texture": "examplemod:copper/inner_overlay",
                "dyeable": {
                    "color_when_undyed": 7767006
                }
            }
        ],
        // For wolf armor
        "wolf_body": [
            {
                // Points to assets/examplemod/textures/entity/equipment/wolf_body/copper/wolf.png
                "texture": "examplemod:copper/wolf",
                // When true, uses the texture passed into the layer renderer instead
                "use_player_texture": true
            }
        ],
        // For horse armor
        "horse_body": [
            {
                // Points to assets/examplemod/textures/entity/equipment/horse_body/copper/horse.png
                "texture": "examplemod:copper/horse",
                "use_player_texture": true
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="数据生成">

```java
public class MyEquipmentInfoProvider implements DataProvider {

    private final PackOutput.PathProvider path;

    public MyEquipmentInfoProvider(PackOutput output) {
        this.path = output.createPathProvider(PackOutput.Target.RESOURCE_PACK, "equipment");
    }

    private void add(BiConsumer<ResourceLocation, EquipmentClientInfo> registrar) {
        registrar.accept(
            // Must match Equippable#assetId
            ResourceLocation.fromNamespaceAndPath("examplemod", "copper"),
            EquipmentClientInfo.builder()
                // For humanoid head, chest, and feet
                .addLayers(
                    EquipmentClientInfo.LayerType.HUMANOID,
                    // Base texture
                    new EquipmentClientInfo.Layer(
                        // The relative texture of the armor
                        // Points to assets/examplemod/textures/entity/equipment/humanoid/copper/outer.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/outer"),
                        Optional.empty(),
                        false
                    ),
                    // Overlay texture
                    new EquipmentClientInfo.Layer(
                        // The overlay texture
                        // Points to assets/examplemod/textures/entity/equipment/humanoid/copper/outer_overlay.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/outer_overlay"),
                        // An RGB value (always opaque color)
                        // When not specified, set to 0 (meaning transparent or invisible)
                        Optional.of(new EquipmentClientInfo.Dyeable(Optional.of(0x7683DE))),
                        false
                    )
                )
                // For humanoid legs
                .addLayers(
                    EquipmentClientInfo.LayerType.HUMANOID_LEGGINGS,
                    new EquipmentClientInfo.Layer(
                        // Points to assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/inner"),
                        Optional.empty(),
                        false
                    ),
                    new EquipmentClientInfo.Layer(
                        // Points to assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner_overlay.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/inner_overlay"),
                        Optional.of(new EquipmentClientInfo.Dyeable(Optional.of(0x7683DE))),
                        false
                    )
                )
                // For wolf armor
                .addLayers(
                    EquipmentClientInfo.LayerType.WOLF_BODY,
                    // Base texture
                    new EquipmentClientInfo.Layer(
                        // Points to assets/examplemod/textures/entity/equipment/wolf_body/copper/wolf.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/wolf"),
                        Optional.empty(),
                        // When true, uses the texture passed into the layer renderer instead
                        true
                    )
                )
                // For horse armor
                .addLayers(
                    EquipmentClientInfo.LayerType.HORSE_BODY,
                    // Base texture
                    new EquipmentClientInfo.Layer(
                        // Points to assets/examplemod/textures/entity/equipment/horse_body/copper/horse.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/horse"),
                        Optional.empty(),
                        true
                    )
                )
                .build()
        );
    }

    @Override
    public CompletableFuture<?> run(CachedOutput cache) {
        Map<ResourceLocation, EquipmentClientInfo> map = new HashMap<>();
        this.add((name, info) -> {
            if (map.putIfAbsent(name, info) != null) {
                throw new IllegalStateException("Tried to register equipment client info twice for id: " + name);
            }
        });
        return DataProvider.saveAll(cache, EquipmentClientInfo.CODEC, this.pathProvider, map);
    }

    @Override
    public String getName() {
        return "Equipment Client Infos: " + MOD_ID;
    }
}

@SubscribeEvent // on the mod event bus
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(MyEquipmentInfoProvider::new);
}
```

</TabItem>
</Tabs>

## 装备渲染

装备信息通过 `EntityRenderer` 或其 `RenderLayer` 之一的渲染函数中的 `EquipmentLayerRenderer` 进行渲染。`EquipmentLayerRenderer` 作为渲染上下文的一部分，通过 `EntityRendererProvider.Context#getEquipmentRenderer` 获得。如果需要 `EquipmentClientInfo`，也可以通过 `EntityRendererProvider.Context#getEquipmentAssets` 获得。

默认情况下，以下图层会渲染关联的 `EquipmentClientInfo.LayerType`：

| `LayerType`             | `RenderLayer`          | 使用者                                                        |
|:-----------------------:|:----------------------:|:---------------------------------------------------------------|
| `HUMANOID`              | `HumanoidArmorLayer`   | 玩家、人形生物（例如僵尸、骷髅）、盔甲架 |
| `HUMANOID_LEGGINGS`     | `HumanoidArmorLayer`   | 玩家、人形生物（例如僵尸、骷髅）、盔甲架 |
| `WINGS`                 | `WingsLayer`           | 玩家、人形生物（例如僵尸、骷髅）、盔甲架 |
| `WOLF_BODY`             | `WolfArmorLayer`       | 狼                                                           |
| `HORSE_BODY`            | `HorseArmorLayer`      | 马                                                          |
| `LLAMA_BODY`            | `LlamaDecorLayer`      | 羊驼、行商羊驼                                            |
| `PIG_SADDLE`            | `SimpleEquipmentLayer` | 猪                                                            |
| `STRIDER_SADDLE`        | `SimpleEquipmentLayer` | 炽足兽                                                        |
| `CAMEL_SADDLE`          | `SimpleEquipmentLayer` | 骆驼                                                          |
| `HORSE_SADDLE`          | `SimpleEquipmentLayer` | 马                                                          |
| `DONKEY_SADDLE`         | `SimpleEquipmentLayer` | 驴                                                         |
| `MULE_SADDLE`           | `SimpleEquipmentLayer` | 骡                                                         |
| `ZOMBIE_HORSE_SADDLE`   | `SimpleEquipmentLayer` | 僵尸马                                                   |
| `SKELETON_HORSE_SADDLE` | `SimpleEquipmentLayer` | 骷髅马                                                 |
| `HAPPY_GHAST_BODY`      | `SimpleEquipmentLayer` | 快乐的恶魂                                                    |

`EquipmentLayerRenderer` 只有一个方法来渲染装备层，恰当地命名为 `renderLayers`：

```java
// In some render method where EquipmentLayerRenderer equipmentLayerRenderer is a field
this.equipmentLayerRenderer.renderLayers(
    // The layer type to render
    EquipmentClientInfo.LayerType.HUMANOID,
    // The resource key representing the EquipmentClientInfo JSON
    // This would be set in the `EQUIPPABLE` data component via `assetId`
    stack.get(DataComponents.EQUIPPABLE).assetId().orElseThrow(),
    // The model to apply the equipment info to
    // These are usually separate models from the entity model
    // and are separate ModelLayers linking to a LayerDefinition
    model,
    // The item stack representing the item being rendered as a model
    // This is only used to get the dyeable, foil, and armor trim information
    stack,
    // The pose stack used to render the model in the correct location
    poseStack,
    // The source of the buffers to get the vertex consumer of the render type
    bufferSource,
    // The packed light texture
    lighting,
    // An absolute path of the texture to render when use_player_texture is true for one of the layer if not null
    // Represents an absolute location within the assets folder
    ResourceLocation.fromNamespaceAndPath("examplemod", "textures/other_texture.png")
);
```

[item]: index.md
[datacomponents]: datacomponents.md
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[livingentity]: ../entities/livingentity.md
[registering]: ../concepts/registries.md#methods-for-registering
[rendering]: #equipment-rendering
[respack]: ../resources/index.md#assets
[tag]: ../resources/server/tags.md
[tinting]: ../resources/client/models/index.md#tinting