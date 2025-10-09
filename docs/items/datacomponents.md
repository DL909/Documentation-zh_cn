---
sidebar_position: 2
---

# 数据组件

数据组件是用于在 `ItemStack` 上存储数据的映射中的键值对。每一份数据，例如烟花爆炸或工具，都作为实际对象存储在物品堆上，使得这些值可见且可操作，而无需动态转换通用的编码实例（例如，`CompoundTag`，`JsonElement`）。

## `DataComponentType`

每个数据组件都有一个关联的 `DataComponentType<T>`，其中 `T` 是组件值的类型。`DataComponentType` 代表一个引用已存储组件值的键，以及一些用于处理读写到磁盘和网络的编解码器（如果需要）。

现有组件的列表可以在 `DataComponents` 中找到。

### 创建自定义数据组件

与 `DataComponentType` 关联的组件值必须实现 `hashCode` 和 `equals`，并且在存储时应被视为**不可变的**。

:::note
组件值可以很容易地使用记录类（record）来实现。记录类的字段是不可变的，并且实现了 `hashCode` 和 `equals`。
:::

```java
// A record example
public record ExampleRecord(int value1, boolean value2) {}

// A class example
public class ExampleClass {

    private final int value1;
    // Can be mutable, but care needs to be taken when using
    private boolean value2;

    public ExampleClass(int value1, boolean value2) {
        this.value1 = value1;
        this.value2 = value2;
    }

    @Override
    public int hashCode() {
        return Objects.hash(this.value1, this.value2);
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == this) {
            return true;
        } else {
            return obj instanceof ExampleClass ex
                && this.value1 == ex.value1
                && this.value2 == ex.value2;
        }
    }
}
```

一个标准的 `DataComponentType` 可以通过 `DataComponentType#builder` 创建，并使用 `DataComponentType.Builder#build` 构建。构建器包含三个设置：`persistent`、`networkSynchronized`、`cacheEncoding`。

`persistent` 指定了用于将组件值读写到磁盘的[`Codec`][codec]。`networkSynchronized` 指定了用于在网络上传输组件值的 `StreamCodec`。如果没有指定 `networkSynchronized`，那么 `persistent` 中提供的 `Codec` 将被包装并用作[`StreamCodec`][streamcodec]。

:::warning
构建器中必须提供 `persistent` 或 `networkSynchronized`；否则，将抛出 `NullPointerException`。如果没有数据需要通过网络发送，则将 `networkSynchronized` 设置为 `StreamCodec#unit`，并提供默认的组件值。
:::

`cacheEncoding` 会缓存 `Codec` 的编码结果，这样如果组件值没有改变，任何后续的编码都会使用缓存的值。只有当组件值预计很少或从不改变时才应使用此选项。

`DataComponentType` 是注册表对象，必须被[注册][registered]。

```java
// Using ExampleRecord(int, boolean)
// Only one Codec and/or StreamCodec should be used below
// Multiple are provided for an example

// Basic codec
public static final Codec<ExampleRecord> BASIC_CODEC = RecordCodecBuilder.create(instance ->
    instance.group(
        Codec.INT.fieldOf("value1").forGetter(ExampleRecord::value1),
        Codec.BOOL.fieldOf("value2").forGetter(ExampleRecord::value2)
    ).apply(instance, ExampleRecord::new)
);
public static final StreamCodec<ByteBuf, ExampleRecord> BASIC_STREAM_CODEC = StreamCodec.composite(
    ByteBufCodecs.INT, ExampleRecord::value1,
    ByteBufCodecs.BOOL, ExampleRecord::value2,
    ExampleRecord::new
);

// Unit stream codec if nothing should be sent across the network
public static final StreamCodec<ByteBuf, ExampleRecord> UNIT_STREAM_CODEC = StreamCodec.unit(new ExampleRecord(0, false));


// In another class
// The specialized DeferredRegister.DataComponents simplifies data component registration and avoids some generic inference issues with the `DataComponentType.Builder` within a `Supplier`
public static final DeferredRegister.DataComponents REGISTRAR = DeferredRegister.createDataComponents(Registries.DATA_COMPONENT_TYPE, "examplemod");

public static final Supplier<DataComponentType<ExampleRecord>> BASIC_EXAMPLE = REGISTRAR.registerComponentType(
    "basic",
    builder -> builder
        // The codec to read/write the data to disk
        .persistent(BASIC_CODEC)
        // The codec to read/write the data across the network
        .networkSynchronized(BASIC_STREAM_CODEC)
);

/// Component will not be saved to disk
public static final Supplier<DataComponentType<ExampleRecord>> TRANSIENT_EXAMPLE = REGISTRAR.registerComponentType(
    "transient",
    builder -> builder.networkSynchronized(BASIC_STREAM_CODEC)
);

// No data will be synced across the network
public static final Supplier<DataComponentType<ExampleRecord>> NO_NETWORK_EXAMPLE = REGISTRAR.registerComponentType(
   "no_network",
   builder -> builder
        .persistent(BASIC_CODEC)
        // Note we use a unit stream codec here
        .networkSynchronized(UNIT_STREAM_CODEC)
);
```

## 组件映射

所有的数据组件都存储在 `DataComponentMap` 中，使用 `DataComponentType` 作为键，对象作为值。`DataComponentMap` 的功能类似于一个只读的 `Map`。因此，有 `#get` 方法可以根据其 `DataComponentType` 获取一个条目，或者如果不存在则提供一个默认值（通过 `#getOrDefault`）。

```java
// For some DataComponentMap map

// Will get dye color if component is present
// Otherwise null
@Nullable
DyeColor color = map.get(DataComponents.BASE_COLOR);
```

### `PatchedDataComponentMap`

由于默认的 `DataComponentMap` 只提供基于读的操作方法，因此写操作由其子类 `PatchedDataComponentMap` 支持。这包括 `#set` 一个组件的值或者完全 `#remove` 它。

`PatchedDataComponentMap` 使用一个原型（prototype）和一个补丁映射（patch map）来存储更改。原型是一个 `DataComponentMap`，它包含此映射应具有的默认组件及其值。补丁映射是一个从 `DataComponentType` 到 `Optional` 值的映射，它包含对默认组件所做的更改。

```java
// For some PatchedDataComponentMap map

// Sets the base color to white
map.set(DataComponents.BASE_COLOR, DyeColor.WHITE);

// Removes the base color by
// - Removing the patch if no default is provided
// - Setting an empty optional if there is a default
map.remove(DataComponents.BASE_COLOR);
```

:::danger
原型和补丁映射都是 `PatchedDataComponentMap` 哈希码的一部分。因此，映射中的任何组件值都应被视为**不可变的**。在修改数据组件的值后，请务必调用 `#set` 或其下文讨论的关联方法之一。
:::

## 组件持有者

所有可以持有数据组件的实例都实现了 `DataComponentHolder`。`DataComponentHolder` 实际上是 `DataComponentMap` 中只读方法的委托。

```java
// For some ItemStack stack

// Delegates to 'DataComponentMap#get'
@Nullable
DyeColor color = stack.get(DataComponents.BASE_COLOR);
```

### `MutableDataComponentHolder`

`MutableDataComponentHolder` 是 NeoForge 提供的一个接口，用于支持对组件映射进行写操作的方法。Vanilla 和 NeoForge 中的所有实现都使用 `PatchedDataComponentMap` 来存储数据组件，因此 `#set` 和 `#remove` 方法也有同名的委托。

此外，`MutableDataComponentHolder` 还提供了一个 `#update` 方法，该方法处理获取组件值（如果未设置则获取提供的默认值）、对值进行操作，然后将其设置回映射中。操作符可以是一个 `UnaryOperator`，它接收组件值并返回组件值；也可以是一个 `BiFunction`，它接收组件值和另一个对象并返回组件值。

```java
// For some ItemStack stack

FireworkExplosion explosion = stack.get(DataComponents.FIREWORK_EXPLOSION);

// Modifying the component value
explosion = explosion.withFadeColors(new IntArrayList(new int[] {1, 2, 3}));

// Since we modified the component value, 'set' should be called afterward
stack.set(DataComponents.FIREWORK_EXPLOSION, explosion);

// Update the component value (calls 'set' internally)
stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    // Default value if no component value is present
    FireworkExplosion.DEFAULT,
    // Return a new FireworkExplosion to set
    explosion -> explosion.withFadeColors(new IntArrayList(new int[] {4, 5, 6}))
);

stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    // Default value if no component value is present
    FireworkExplosion.DEFAULT,
    // An object that is supplied to the function
    new IntArrayList(new int[] {7, 8, 9}),
    // Return a new FireworkExplosion to set
    FireworkExplosion::withFadeColors
);
```

## 向物品添加默认数据组件

虽然数据组件存储在 `ItemStack` 上，但可以在 `Item` 上设置一个默认组件的映射，以便在构造 `ItemStack` 时作为原型传递给它。可以通过 `Item.Properties#component` 将组件添加到 `Item` 中。

```java
// For some DeferredRegister.Items REGISTRAR
public static final Item COMPONENT_EXAMPLE = REGISTRAR.register("component",
    // register is used over other overloads as the DataComponentType has not been registered yet
    registryName -> new Item(
        new Item.Properties()
        .setId(ResourceKey.create(Registries.ITEM, registryName))
        .component(BASIC_EXAMPLE.get(), new ExampleRecord(24, true))
    )
);
```

如果数据组件应该被添加到属于 Vanilla 或其他模组的现有物品上，那么应该在[**模组事件总线**][modbus]上监听 `ModifyDefaultComponentEvent` 事件。该事件提供了 `modify` 和 `modifyMatching` 方法，允许修改关联物品的 `DataComponentPatch.Builder`。构建器可以 `#set` 组件或 `#remove` 现有组件。

```java
@SubscribeEvent // on the mod event bus
public static void modifyComponents(ModifyDefaultComponentsEvent event) {
    // Sets the component on melon seeds
    event.modify(Items.MELON_SEEDS, builder ->
        builder.set(BASIC_EXAMPLE.get(), new ExampleRecord(10, false))
    );

    // Removes the component for any items that have a crafting remainder
    event.modifyMatching(
        item -> !item.getCraftingRemainder().isEmpty(),
        builder -> builder.remove(DataComponents.BUCKET_ENTITY_DATA)
    );
}
```

## 使用自定义组件持有者

要创建一个自定义的数据组件持有者，持有者对象只需实现 `MutableDataComponentHolder` 并实现缺失的方法。持有者对象必须包含一个表示 `PatchedDataComponentMap` 的字段来实现相关方法。

```java
public class ExampleHolder implements MutableDataComponentHolder {

    private int data;
    private final PatchedDataComponentMap components;

    // Overloads can be provided to supply the map itself
    public ExampleHolder() {
        this.data = 0;
        this.components = new PatchedDataComponentMap(DataComponentMap.EMPTY);
    }

    @Override
    public DataComponentMap getComponents() {
        return this.components;
    }

    @Nullable
    @Override
    public <T> T set(DataComponentType<? super T> componentType, @Nullable T value) {
        return this.components.set(componentType, value);
    }

    @Nullable
    @Override
    public <T> T remove(DataComponentType<? extends T> componentType) {
        return this.components.remove(componentType);
    }

    @Override
    public void applyComponents(DataComponentPatch patch) {
        this.components.applyPatch(patch);
    }

    @Override
    public void applyComponents(DataComponentMap components) {
        this.components.setAll(components);
    }

    // Other methods
}
```

### `DataComponentPatch` 和编解码器

为了将组件持久化到磁盘或通过网络发送信息，持有者可以发送整个 `DataComponentMap`。然而，这通常是一种信息浪费，因为任何默认值都将已经存在于数据发送到的地方。因此，我们使用 `DataComponentPatch` 来发送相关数据。`DataComponentPatch` 仅包含组件映射的补丁信息，不含任何默认值。然后，这些补丁将在接收方的位置应用于原型。

一个 `DataComponentPatch` 可以通过 `#patch` 从一个 `PatchedDataComponentMap` 创建。同样地，`PatchedDataComponentMap#fromPatch` 可以给定的原型 `DataComponentMap` 和一个 `DataComponentPatch` 来构造一个 `PatchedDataComponentMap`。

```java
public class ExampleHolder implements MutableDataComponentHolder {

    public static final Codec<ExampleHolder> CODEC = RecordCodecBuilder.create(instance ->
        instance.group(
            Codec.INT.fieldOf("data").forGetter(ExampleHolder::getData),
            DataCopmonentPatch.CODEC.optionalFieldOf("components", DataComponentPatch.EMPTY).forGetter(holder -> holder.components.asPatch())
        ).apply(instance, ExampleHolder::new)
    );

    public static final StreamCodec<RegistryFriendlyByteBuf, ExampleHolder> STREAM_CODEC = StreamCodec.composite(
        ByteBufCodecs.INT, ExampleHolder::getData,
        DataComponentPatch.STREAM_CODEC, holder -> holder.components.asPatch(),
        ExampleHolder::new
    );

    // ...

    public ExampleHolder(int data, DataComponentPatch patch) {
        this.data = data;
        this.components = PatchedDataComponentMap.fromPatch(
            // The prototype map to apply to
            DataComponentMap.EMPTY,
            // The associated patches
            patch
        );
    }

    // ...
}
```

[通过网络同步持有者数据][network]以及读写数据到磁盘必须手动完成。

[registered]: ../concepts/registries.md
[codec]: ../datastorage/codecs.md
[modbus]: ../concepts/events.md#event-buses
[network]: ../networking/payload.md
[streamcodec]: ../networking/streamcodecs.md