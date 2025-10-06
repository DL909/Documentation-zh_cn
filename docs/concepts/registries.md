---
sidebar_position: 1
---
# 注册表

注册是将模组的对象（例如[物品][item]、[方块][block]、实体等）告知游戏的过程。注册事物很重要，因为没有注册，游戏将根本不知道这些对象，这会导致无法解释的行为和崩溃。

简单来说，注册表是一个映射（map）的包装器，它将注册名（下文会介绍）映射到已注册的对象，这些对象通常被称为注册表条目。注册名在同一个注册表内必须是唯一的，但同一个注册名可以存在于多个注册表中。最常见的例子是，方块（在 `BLOCKS` 注册表中）有一个物品形式，并且在 `ITEMS` 注册表中使用相同的注册名。

每个已注册的对象都有一个唯一的名称，称为其注册名。该名称由一个 [`ResourceLocation`][resloc] 表示。例如，泥土方块的注册名是 `minecraft:dirt`，僵尸的注册名是 `minecraft:zombie`。模组添加的对象当然不会使用 `minecraft` 命名空间；而是会使用它们的模组 ID。

## 原版 vs. 模组

为了理解 NeoForge 注册表系统中的一些设计决策，我们首先来看看 Minecraft 是如何做到这一点的。我们将以方块注册表为例，因为大多数其他注册表的运作方式都相同。

注册表通常注册[单例][singleton]。这意味着所有注册表条目都只存在一次。例如，你在整个游戏中看到的所有石头方块实际上都是同一个石头方块，只是被显示了很多次。如果你需要石头方块，你可以通过引用已注册的方块实例来获取它。

Minecraft 在 `Blocks` 类中注册所有方块。通过 `register` 方法，`Registry#register()` 被调用，其中 `BuiltInRegistries.BLOCK` 处的方块注册表是第一个参数。在所有方块都注册完毕后，Minecraft 会基于方块列表执行各种检查，例如验证所有方块都已加载模型的自我检查。

这一切之所以能正常工作，主要是因为 `Blocks` 类被 Minecraft 足够早地加载了。模组不会被 Minecraft 自动进行类加载，因此需要变通方法。

## 注册方法

NeoForge 提供了两种注册对象的方式：`DeferredRegister` 类和 `RegisterEvent`。请注意，前者是后者的包装器，为了防止出错，推荐使用前者。

### `DeferredRegister`

我们首先创建我们的 `DeferredRegister`：

```java
public static final DeferredRegister<Block> BLOCKS = DeferredRegister.create(
        // The registry we want to use.
        // Minecraft's registries can be found in BuiltInRegistries, NeoForge's registries can be found in NeoForgeRegistries.
        // Mods may also add their own registries, refer to the individual mod's documentation or source code for where to find them.
        BuiltInRegistries.BLOCKS,
        // Our mod id.
        ExampleMod.MOD_ID
);
```

然后，我们可以使用以下方法之一将我们的注册表条目添加为静态 final 字段（关于在 `new Block()` 中添加什么参数，请参阅[关于方块的文章][block]）：

```java
public static final DeferredHolder<Block, Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
        // Our registry name.
        "example_block",
        // A supplier of the object we want to register.
        () -> new Block(...)
);

public static final DeferredHolder<Block, SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
        // Our registry name.
        "example_block",
        // A function creating the object we want to register
        // given its registry name as a ResourceLocation.
        registryName -> new SlabBlock(...)
);
```

`DeferredHolder<R, T extends R>` 类持有我们的对象。类型参数 `R` 是我们注册到的注册表的类型（在我们的例子中是 `Block`）。类型参数 `T` 是我们 supplier 的类型。由于我们在第一个例子中直接注册了一个 `Block`，我们提供 `Block` 作为第二个参数。如果我们要注册一个 `Block` 的子类的对象，例如 `SlabBlock`（如第二个例子所示），我们会在这里提供 `SlabBlock`。

`DeferredHolder<R, T extends R>` 是 `Supplier<T>` 的一个子类。为了在需要时获取我们注册的对象，我们可以调用 `DeferredHolder#get()`。`DeferredHolder` 继承自 `Supplier` 这一事实也允许我们将 `Supplier` 用作字段的类型。这样，上面的代码块就变成了下面这样：

```java
public static final Supplier<Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
        // Our registry name.
        "example_block",
        // A supplier of the object we want to register.
        () -> new Block(...)
);

public static final Supplier<SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
        // Our registry name.
        "example_block",
        // A function creating the object we want to register
        // given its registry name as a ResourceLocation.
        registryName -> new SlabBlock(...)
);
```

:::note
注意，有些地方明确要求一个 `Holder` 或 `DeferredHolder`，而不会只接受任何 `Supplier`。如果你需要这两种类型中的任何一种，最好根据需要将你的 `Supplier` 的类型改回 `Holder` 或 `DeferredHolder`。
:::

最后，由于整个系统是注册表事件的包装器，我们需要告诉 `DeferredRegister` 根据需要将自己附加到注册表事件上：

```java
//This is our mod constructor
public ExampleMod(IEventBus modBus) {
    //highlight-next-line
    ExampleBlocksClass.BLOCKS.register(modBus);
    //Other stuff here
}
```

:::info
对于方块、物品、数据组件和实体，存在 `DeferredRegister` 的提供了辅助方法的专门变体，它们分别是 [`DeferredRegister.Blocks`][defregblocks]、[`DeferredRegister.Items`][defregitems]、[`DeferredRegister.DataComponents`][defregcomp] 和 [`DeferredRegister.Entities`][defregentity]。
:::

### `RegisterEvent`

`RegisterEvent` 是注册对象的第二种方式。对于每个注册表，这个[事件][event]会在模组构造函数之后（因为 `DeferredRegister` 在构造函数中注册它们的内部事件处理器）和加载配置之前被触发。`RegisterEvent` 在模组事件总线上被触发。

```java
@SubscribeEvent // on the mod event bus
public static void register(RegisterEvent event) {
    event.register(
            // This is the registry key of the registry.
            // Get these from BuiltInRegistries for vanilla registries,
            // or from NeoForgeRegistries.Keys for NeoForge registries.
            BuiltInRegistries.BLOCKS,
            // Register your objects here.
            registry -> {
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_1"), new Block(...));
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_2"), new Block(...));
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_3"), new Block(...));
            }
    );
}
```

## 查询注册表

有时，你会发现自己处于这样一种情况：你想要通过给定的 ID 获取一个已注册的对象。或者，你想要获取某个已注册对象的 ID。由于注册表基本上是 ID（`ResourceLocation`）到不同对象的映射，即一个可逆映射，所以这两种操作都可行：

```java
BuiltInRegistries.BLOCKS.getValue(ResourceLocation.fromNamespaceAndPath("minecraft", "dirt")); // returns the dirt block
BuiltInRegistries.BLOCKS.getKey(Blocks.DIRT); // returns the resource location "minecraft:dirt"

// Assume that ExampleBlocksClass.EXAMPLE_BLOCK.get() is a Supplier<Block> with the id "yourmodid:example_block"
BuiltInRegistries.BLOCKS.getValue(ResourceLocation.fromNamespaceAndPath("yourmodid", "example_block")); // returns the example block
BuiltInRegistries.BLOCKS.getKey(ExampleBlocksClass.EXAMPLE_BLOCK.get()); // returns the resource location "yourmodid:example_block"
```

如果你只想检查一个对象是否存在，这也是可以的，不过只能通过键（key）来实现：

```java
BuiltInRegistries.BLOCKS.containsKey(ResourceLocation.fromNamespaceAndPath("minecraft", "dirt")); // true
BuiltInRegistries.BLOCKS.containsKey(ResourceLocation.fromNamespaceAndPath("create", "brass_ingot")); // true only if Create is installed
```

正如最后一个例子所示，这对于任何模组 ID 都是可行的，因此是检查来自另一个模组的某个物品是否存在的完美方式。

最后，我们还可以遍历一个注册表中的所有条目，可以遍历键（key），也可以遍历条目（条目使用 Java 的 `Map.Entry` 类型）：

```java
for (ResourceLocation id : BuiltInRegistries.BLOCKS.keySet()) {
    // ...
}
for (Map.Entry<ResourceKey<Block>, Block> entry : BuiltInRegistries.BLOCKS.entrySet()) {
    // ...
}
```

:::note
查询操作总是使用原版的 `Registry`，而不是 `DeferredRegister`。这是因为 `DeferredRegister` 仅仅是注册工具。
:::

:::danger
只有在注册完成后才能安全地使用查询操作。**不要在注册仍在进行时查询注册表！**
:::

## 自定义注册表

自定义注册表允许你指定额外的系统，让你的模组的附属模组可以接入。例如，如果你的模组要添加法术，你可以将法术做成一个注册表，从而允许其他模组向你的模组添加法术，而你无需做任何额外的事情。它还允许你自动执行一些操作，比如同步条目。

让我们从创建[注册表键][resourcekey]和注册表本身开始：

```java
// We use spells as an example for the registry here, without any details about what a spell actually is (as it doesn't matter).
// Of course, all mentions of spells can and should be replaced with whatever your registry actually is.
public static final ResourceKey<Registry<Spell>> SPELL_REGISTRY_KEY = ResourceKey.createRegistryKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "spells"));
public static final Registry<YourRegistryContents> SPELL_REGISTRY = new RegistryBuilder<>(SPELL_REGISTRY_KEY)
        // If you want to enable integer id syncing, for networking.
        // These should only be used in networking contexts, for example in packets or purely networking-related NBT data.
        .sync(true)
        // The default key. Similar to minecraft:air for blocks. This is optional.
        .defaultKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "empty"))
        // Effectively limits the max count. Generally discouraged, but may make sense in settings such as networking.
        .maxId(256)
        // Build the registry.
        .create();
```

然后，通过在 `NewRegistryEvent` 中将它们注册到根注册表来告诉游戏该注册表的存在：

```java
@SubscribeEvent // on the mod event bus
public static void registerRegistries(NewRegistryEvent event) {
    event.register(SPELL_REGISTRY);
}
```

你现在可以像处理任何其他注册表一样，通过 `DeferredRegister` 和 `RegisterEvent` 来注册新的注册表内容：

```java
public static final DeferredRegister<Spell> SPELLS = DeferredRegister.create(SPELL_REGISTRY, "yourmodid");
public static final Supplier<Spell> EXAMPLE_SPELL = SPELLS.register("example_spell", () -> new Spell(...));

// Alternatively:
@SubscribeEvent // on the mod event bus
public static void register(RegisterEvent event) {
    event.register(SPELL_REGISTRY_KEY, registry -> {
        registry.register(ResourceLocation.fromNamespaceAndPath("yourmodid", "example_spell"), () -> new Spell(...));
    });
}
```

## 数据包注册表

数据包注册表（也称为动态注册表，或根据其主要用例称为世界生成注册表）是一种特殊的注册表，它在世界加载时从[数据包][datapack] JSON 文件中加载数据（因此得名），而不是在游戏启动时加载。默认的数据包注册表最主要地包括大多数世界生成注册表，以及其他少数几个。

数据包注册表允许其内容在 JSON 文件中指定。这意味着不需要代码（如果你不想自己编写 JSON 文件，那么除了[数据生成][datagen]之外）。每个数据包注册表都与其关联一个 [`Codec`][codec]，用于序列化，并且每个注册表的 ID 决定了其数据包路径：

- Minecraft 的数据包注册表使用格式 `data/yourmodid/registrypath`（例如 `data/yourmodid/worldgen/biome`，其中 `worldgen/biome` 是注册表路径）。
- 所有其他数据包注册表（NeoForge 或模组的）使用格式 `data/yourmodid/registrynamespace/registrypath`（例如 `data/yourmodid/neoforge/biome_modifier`，其中 `neoforge` 是注册表命名空间，`biome_modifier` 是注册表路径）。

数据包注册表可以从 `RegistryAccess` 中获取。如果在服务端，可以通过调用 `ServerLevel#registryAccess()` 来检索这个 `RegistryAccess`；如果在客户端，则通过 `Minecraft.getInstance().getConnection()#registryAccess()`（后者只有在你实际连接到一个世界时才有效，否则连接将为 null）。这些调用的结果可以像任何其他注册表一样使用，以获取特定元素或遍历内容。

### 自定义数据包注册表

自定义数据包注册表不需要构造一个 `Registry`。相反，它们只需要一个注册表键和至少一个 [`Codec`][codec] 来（反）序列化其内容。重申之前的法术例子，将我们的法术注册表注册为数据包注册表看起来像这样：

```java
public static final ResourceKey<Registry<Spell>> SPELL_REGISTRY_KEY = ResourceKey.createRegistryKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "spells"));

@SubscribeEvent // on the mod event bus
public static void registerDatapackRegistries(DataPackRegistryEvent.NewRegistry event) {
    event.dataPackRegistry(
            // The registry key.
            SPELL_REGISTRY_KEY,
            // The codec of the registry contents.
            Spell.CODEC,
            // The network codec of the registry contents. Often identical to the normal codec.
            // May be a reduced variant of the normal codec that omits data that is not needed on the client.
            // May be null. If null, registry entries will not be synced to the client at all.
            // May be omitted, which is functionally identical to passing null (a method overload
            // with two parameters is called that passes null to the normal three parameter method).
            Spell.CODEC,
            // A consumer which configures the constructed registry via the RegistryBuilder.
            // May be omitted, which is functionally identical to passing builder -> {}.
            builder -> builder.maxId(256)
    );
}
```

### 数据包注册表的数据生成

由于手动编写所有 JSON 文件既繁琐又容易出错，NeoForge 提供了一个[数据提供器][datagenindex]来为你生成 JSON 文件。这对于内置的和自己的数据包注册表都适用。

首先，我们创建一个 `RegistrySetBuilder` 并向其添加我们的条目（一个 `RegistrySetBuilder` 可以包含多个注册表的条目）：

```java
new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        // Register configured features through the bootstrap context (see below)
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        // Register placed features through the bootstrap context (see below)
    });
```

`bootstrap` lambda 参数是我们实际用来注册对象的。它的类型是 `BootstrapContext`。要注册一个对象，我们像这样调用它的 `#register` 方法：

```java
// The resource key of our object.
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_CONFIGURED_FEATURE = ResourceKey.create(
    Registries.CONFIGURED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_configured_feature")
);

new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        bootstrap.register(
            // The resource key of our configured feature.
            EXAMPLE_CONFIGURED_FEATURE,
            // The actual configured feature.
            new ConfiguredFeature<>(Feature.ORE, new OreConfiguration(...))
        );
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        // ...
    });
```

如果需要，`BootstrapContext` 也可以用来查找另一个注册表中的条目：

```java
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_CONFIGURED_FEATURE = ResourceKey.create(
    Registries.CONFIGURED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_configured_feature")
);
public static final ResourceKey<PlacedFeature> EXAMPLE_PLACED_FEATURE = ResourceKey.create(
    Registries.PLACED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_placed_feature")
);

new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        bootstrap.register(EXAMPLE_CONFIGURED_FEATURE, ...);
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        HolderGetter<ConfiguredFeature<?, ?>> otherRegistry = bootstrap.lookup(Registries.CONFIGURED_FEATURE);
        bootstrap.register(EXAMPLE_PLACED_FEATURE, new PlacedFeature(
            otherRegistry.getOrThrow(EXAMPLE_CONFIGURED_FEATURE), // Get the configured feature
            List.of() // No-op when placement happens - replace with whatever your placement parameters are
        ));
    });
```

最后，我们在一个实际的数据提供器中使用我们的 `RegistrySetBuilder`，并将该数据提供器注册到事件中：

```java
@SubscribeEvent // on the mod event bus
public static void onGatherData(GatherDataEvent.Client event) {
    // Adds the generated registry objects to the current lookup provider for use
    // in other datagen.
    this.createDatapackRegistryObjects(
        // Our registry set builder to generate the data from.
        new RegistrySetBuilder().add(...),
        // (Optional) A biconsumer that takes in any conditions to load the object
        // associated with the resource key
        conditions -> {
            conditions.accept(resourceKey, condition);
        },
        // (Optional) A set of mod ids we are generating the entries for
        // By default, supplies the mod id of the current mod container.
        Set.of("yourmodid")
    );

    // You can use the lookup provider with your generated entries by either calling one
    // of the `#create*` methods or grabbing the actual lookup via `#getLookupProvider`
    // ...
}
```

[block]: ../blocks/index.md
[blockentity]: ../blockentities/index.md
[codec]: ../datastorage/codecs.md
[datagen]: #data-generation-for-datapack-registries
[datagenindex]: ../resources/index.md#data-generation
[datapack]: ../resources/index.md#data
[defregblocks]: ../blocks/index.md#deferredregisterblocks-helpers
[defregcomp]: ../items/datacomponents.md#creating-custom-data-components
[defregentity]: ../entities/index.md#entitytype
[defregitems]: ../items/index.md#deferredregisteritems
[event]: events.md
[item]: ../items/index.md
[resloc]: ../misc/resourcelocation.md
[resourcekey]: ../misc/resourcelocation.md#resourcekeys
[singleton]: https://en.wikipedia.org/wiki/Singleton_pattern