---
sidebar_position: 4
---
# 属性

属性是[生物实体][livingentity]的特殊字段，它们决定了诸如最大生命值、速度或护甲等基本属性。所有属性都存储为 double 值并自动同步。原版提供了广泛的默认属性，你也可以添加自己的属性。

由于历史实现的原因，并非所有属性都对所有实体起作用。例如，恶魂会忽略飞行速度，而跳跃力量只影响马，不影响玩家。

## 内置属性

### Minecraft

以下属性位于 `minecraft` 命名空间下，其代码内值可在 `Attributes` 类中找到。

| 名称                             | 代码内                             | 范围          | 默认值 | 用途                                                                                                                                                                 |
|----------------------------------|----------------------------------|----------------|---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `armor`                          | `ARMOR`                          | `[0,30]`       | 0             | 实体的护甲值。值为 1 意味着热键栏上方半个胸甲图标。                                                                                                                    |
| `armor_toughness`                | `ARMOR_TOUGHNESS`                | `[0,20]`       | 0             | 实体的护甲韧性值。更多信息请参阅 [Minecraft Wiki][wiki] 上的[护甲韧性][toughness]。                                                                                  |
| `attack_damage`                  | `ATTACK_DAMAGE`                  | `[0,2048]`     | 2             | 实体造成的基础攻击伤害，不包括任何武器或类似物品。                                                                                                                       |
| `attack_knockback`               | `ATTACK_KNOCKBACK`               | `[0,5]`        | 0             | 实体造成的额外击退。击退还有一个未由此属性表示的基础强度。                                                                                                            |
| `attack_speed`                   | `ATTACK_SPEED`                   | `[0,1024]`     | 4             | 实体的攻击冷却时间。数值越高意味着冷却时间越长，设置为 0 可有效重新启用 1.9 以前的战斗机制。                                                                       |
| `block_break_speed`              | `BLOCK_BREAK_SPEED`              | `[0,1024]`     | 1             | 实体挖掘方块的速度，作为乘法修正值。更多信息请参阅[挖掘速度][miningspeed]。                                                                             |
| `block_interaction_range`        | `BLOCK_INTERACTION_RANGE`        | `[0,64]`       | 4.5           | 实体可以与方块交互的范围，单位为方块。                                                                                                                                  |
| `burning_time`                   | `BURNING_TIME`                   | `[0,1024]`     | 1             | 实体被点燃时燃烧时间的倍率。                                                                                                                                          |
| `explosion_knockback_resistance` | `EXPLOSION_KNOCKBACK_RESISTANCE` | `[0,1]`        | 0             | 实体的爆炸击退抗性。这是一个百分比值，即 0 是无抗性，0.5 是半抗性，1 是完全抗性。                                                                                          |
| `entity_interaction_range`       | `ENTITY_INTERACTION_RANGE`       | `[0,64]`       | 3             | 实体可以与其他实体交互的范围，单位为方块。                                                                                                                              |
| `fall_damage_multiplier`         | `FALL_DAMAGE_MULTIPLIER`         | `[0,100]`      | 1             | 实体受到的摔落伤害的倍率。                                                                                                                                              |
| `flying_speed`                   | `FLYING_SPEED`                   | `[0,1024]`     | 0.4           | 飞行速度的倍率。并非所有飞行实体都实际使用此属性，例如恶魂会忽略它。                                                                                                        |
| `follow_range`                   | `FOLLOW_RANGE`                   | `[0,2048]`     | 32            | 实体将以玩家为目标/跟随玩家的距离，单位为方块。                                                                                                                               |
| `gravity`                        | `GRAVITY`                        | `[1,1]`        | 0.08          | 实体受到的重力影响，单位为方块/刻²。                                                                                                                                        |
| `jump_strength`                  | `JUMP_STRENGTH`                  | `[0,32]`       | 0.42          | 实体的跳跃力量。数值越高意味着跳得越高。                                                                                                                                |
| `knockback_resistance`           | `KNOCKBACK_RESISTANCE`           | `[0,1]`        | 0             | 实体的击退抗性。这是一个百分比值，即 0 是无抗性，0.5 是半抗性，1 是完全抗性。                                                                                              |
| `luck`                           | `LUCK`                           | `[-1024,1024]` | 0             | 实体的幸运值。在掷[战利品表][loottables]时使用此值，以提供额外掷骰机会或以其他方式修改结果物品的品质。                                                                     |
| `max_absorption`                 | `MAX_ABSORPTION`                 | `[0,2048]`     | 0             | 实体的最大吸收（黄色心）。值为 1 意味着半颗心。                                                                                                                          |
| `max_health`                     | `MAX_HEALTH`                     | `[1,1024]`     | 20            | 实体的最大生命值。值为 1 意味着半颗心。                                                                                                                                  |
| `mining_efficiency`              | `MINING_EFFICIENCY`              | `[0,1024]`     | 0             | 实体挖掘方块的速度，作为加法修正值，仅当使用正确的工具时有效。更多信息请参阅[挖掘速度][miningspeed]。                                                           |
| `movement_efficiency`            | `MOVEMENT_EFFICIENCY`            | `[0,1]`        | 0             | 当实体在有减速效果的方块（如灵魂沙）上行走时，施加于实体的线性插值移动速度加成。                                                                                        |
| `movement_speed`                 | `MOVEMENT_SPEED`                 | `[0,1024]`     | 0.7           | 实体的移动速度。数值越高意味着速度越快。                                                                                                                                |
| `oxygen_bonus`                   | `OXYGEN_BONUS`                   | `[0,1024]`     | 0             | 实体的氧气加成。此值越高，实体开始溺水所需时间越长。                                                                                                                    |
| `safe_fall_distance`             | `SAFE_FALL_DISTANCE`             | `[-1024,1024]` | 3             | 实体安全摔落的距离，即在此距离内不会受到摔落伤害。                                                                                                                        |
| `scale`                          | `SCALE`                          | `[0.0625,16]`  | 1             | 实体渲染时的缩放比例。                                                                                                                                                  |
| `sneaking_speed`                 | `SNEAKING_SPEED`                 | `[0,1]`        | 0.3           | 当实体潜行时，应用于实体的移动速度倍率。                                                                                                                                    |
| `spawn_reinforcements`           | `SPAWN_REINFORCEMENTS_CHANCE`    | `[0,1]`        | 0             | 僵尸生成其他僵尸的几率。这仅在困难难度下有效，因为僵尸增援在普通或更低难度下不会发生。                                                                                    |
| `step_height`                    | `STEP_HEIGHT`                    | `[0,10]`       | 0.6           | 实体的步高，单位为方块。如果此值为 1，玩家可以像走上台阶一样走上 1 方块高的平台。                                                                                           |
| `submerged_mining_speed`         | `SUBMERGED_MINING_SPEED`         | `[0,20]`       | 0.2           | 实体挖掘方块的速度，作为乘法修正值，仅当实体在水下时有效。更多信息请参阅[挖掘速度][miningspeed]。                                                              |
| `sweeping_damage_ratio`          | `SWEEPING_DAMAGE_RATIO`          | `[0,1]`        | 0             | 横扫攻击造成的伤害量，为主攻击的百分比。这是一个百分比值，即 0 是无伤害，0.5 是半伤害，1 是全伤害。                                                                            |
| `tempt_range`                    | `TEMPT_RANGE`                    | `[0,2048]`     | 10            | 使用物品引诱实体的范围。主要用于被动动物，例如牛或猪。                                                                                                                     |
| `water_movement_efficiency`      | `WATER_MOVEMENT_EFFICIENCY`      | `[0,1]`        | 0             | 当实体在水下时应用的移动速度倍率。                                                                                                                                     |
| `waypoint_transmit_range`        | `WAYPOINT_TRANSMIT_RANGE`        | `[0,60000000]` | 0             | 实体可以将其位置传输到某个路径点追踪器的范围。 |
| `waypoint_receive_range`         | `WAYPOINT_RECEIVE_RANGE`         | `[0,60000000]` | 0             | 实体可以接收另一个发射器的范围。                    |

:::warnings
Mojang 相对随意地设置了一些属性上限。这在护甲上尤其明显，其上限为 30。NeoForge 没有触及这些上限，但有模组可以改变它们。
:::

### NeoForge

以下属性位于 `neoforge` 命名空间下，其代码内值可在 `NeoForgeMod` 类中找到。

| 名称               | 代码内            | 范围      | 默认值 | 用途                                                                                                                                                |
|--------------------|--------------------|------------|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `creative_flight`  | `CREATIVE_FLIGHT`  | `[0,1]`    | 0             | 决定实体的创造模式飞行是否启用 (\> 0) 或禁用 (\<\= 0)。                                                                                                |
| `nametag_distance` | `NAMETAG_DISTANCE` | `[0,32]`   | 32            | 实体的名称标签可见的距离，单位为方块。                                                                                                                   |
| `swim_speed`       | `SWIM_SPEED`       | `[0,1024]` | 1             | 当实体在水下时应用的移动速度倍率。此倍率独立于 `minecraft:water_movement_efficiency` 应用。 |
