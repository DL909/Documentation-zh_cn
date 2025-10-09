---
sidebar_position: 2
---
# 数据与网络

一个没有数据的实体是相当无用的，因此，在实体上存储数据至关重要。所有实体都会存储一些默认数据，例如它们的类型和位置。本文将解释如何添加您自己的数据，以及如何同步这些数据。

添加数据最简单的方法是在你的 `Entity` 类中添加一个字段。然后你可以随心所欲地与这些数据交互。然而，一旦你需要同步这些数据，这很快就会变得非常烦人。这是因为大多数实体逻辑仅在服务器上运行，并且只是偶尔（取决于 [`EntityType`][entitytype] 的 `clientUpdateInterval` 值）才会向客户端发送更新；这也是当服务器的 tick 速度太慢时，实体容易出现明显“延迟”的原因。

因此，原版引入了一些系统来帮助解决这个问题，每个系统都有其特定的用途。在必要时，你也可以选择[发送自定义数据][custom]。

## `SynchedEntityData`

`SynchedEntityData` 是一个用于在运行时存储值并通过网络同步它们的系统。它分为三个类：

- `EntityDataSerializer` 基本上是 [`StreamCodec`][streamcodec] 的包装器。
    - Minecraft 使用一个硬编码的序列化器映射。NeoForge 将此映射转换为一个注册表，这意味着如果你想添加新的 `EntityDataSerializer`，它们必须通过[注册][registration]来添加。
    - Minecraft 在 `EntityDataSerializers` 类中定义了各种默认的 `EntityDataSerializer`。
- `EntityDataAccessor` 由实体持有，用于获取和设置数据值。
- `SynchedEntityData` 本身持有实体所有的 `EntityDataAccessor`，并按需自动调用 `EntityDataSerializer` 来同步值。

首先，在您的实体类中创建一个 `EntityDataAccessor`：

```java
public class MyEntity extends Entity {
    // The generic type must match the one of the second parameter below.
    public static final EntityDataAccessor<Integer> MY_DATA =
        SynchedEntityData.defineId(
            // The class of the entity.
            MyEntity.class,
            // The entity data accessor type.
            EntityDataSerializers.INT
        );
}
```

:::danger
虽然编译器会允许你在 `SynchedEntityData#defineId()` 中使用非所属类作为第一个参数，但这样做会并且将会导致难以调试的问题，因此应不惜一切代价避免。（这也包括通过 mixin 或类似方法添加字段。）
:::

然后我们必须在 `defineSynchedData` 方法中定义默认值，如下所示：

```java
public class MyEntity extends Entity {
    public static final EntityDataAccessor<Integer> MY_DATA = SynchedEntityData.defineId(MyEntity.class, EntityDataSerializers.INT);

    @Override
    protected void defineSynchedData(SynchedEntityData.Builder builder) {
        // Our default value is zero.
        builder.define(MY_DATA, 0);
    }
}
```

最后，我们可以像这样获取和设置实体数据（假设我们在 `MyEntity` 的一个方法中）：

```java
int data = this.getEntityData().get(MY_DATA);
this.getEntityData().set(MY_DATA, 1);
```

## `readAdditionalSaveData` 和 `addAdditionalSaveData`

这两个方法用于从磁盘读取和写入数据。它们通过从/向一个[值I/O][valueio]加载/保存你的值来工作，就像这样：

```java
// Assume that an `int data` exists in the class.
@Override
protected void readAdditionalSaveData(ValueInput input) {
    this.data = input.getIntOr("my_data", 0);
}

@Override
protected void addAdditionalSaveData(ValueOutput output) {
    output.putInt("my_data", this.data);
}
```

## 自定义生成数据

在某些情况下，当实体在客户端生成时，需要一些自定义数据，但这些数据不会随时间改变。在这种情况下，你可以在你的实体上实现 `IEntityWithComplexSpawn` 接口，并使用它的两个方法 `#writeSpawnData` 和 `#readSpawnData` 来向/从网络缓冲区写入/读取数据：

```java
@Override
public void writeSpawnData(RegistryFriendlyByteBuf buf) {
    buf.writeInt(1234);
}

@Override
public void readSpawnData(RegistryFriendlyByteBuf buf) {
    int i = buf.readInt();
}
```

此外，你可以在生成时发送自己的数据包。为此，请重写 `IEntityExtension#sendPairingData` 并在那里发送你的数据包，就像其他任何数据包一样：

```java
@Override
public void sendPairingData(ServerPlayer player, Consumer<CustomPacketPayload> packetConsumer) {
    // Call super for some base functionality.
    super.sendPairingData(player, packetConsumer);
    // Add your own packets.
    packetConsumer.accept(new MyPacket(...));
}
```

有关自定义网络数据包的更多信息，请参阅[网络文章][networking]。

## 数据附件

实体已被修改以扩展 `AttachmentHolder`，因此支持通过[数据附件][attachment]进行数据存储。其主要用途是在不属于您自己的实体（即由 Minecraft 或其他模组添加的实体）上定义自定义数据。更多信息请参见链接的文章。

## 自定义网络消息

为了同步，您也总是可以选择在需要时使用自定义数据包来发送额外信息。更多信息请参阅[网络文章][networking]。

[attachment]: ../datastorage/attachments.md
[custom]: #custom-network-messages
[entitytype]: index.md#entitytype
[networking]: ../networking/index.md
[registration]: ../concepts/registries.md#methods-for-registering
[streamcodec]: ../networking/streamcodecs.md
[valueio]: ../datastorage/valueio.md