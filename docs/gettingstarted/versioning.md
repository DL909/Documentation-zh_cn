# 版本控制

本文将详细介绍版本控制在 Minecraft 和 NeoForge 中的工作方式，并为模组的版本控制提供一些建议。

## Minecraft

Minecraft 使用[语义化版本控制][semver]。语义化版本控制，简称 "semver"，其格式为 `主版本号.次版本号.补丁版本号`。因此，例如，Minecraft 1.20.2 的主版本号为 1，次版本号为 20，补丁版本号为 2。

自 2011 年 Minecraft 1.0 推出以来，Minecraft 一直使用 `1` 作为主版本号。在此之前，版本控制方案经常变更，曾有过像 `a1.1` (Alpha 1.1)、`b1.7.3` (Beta 1.7.3) 这样的版本，甚至还有完全不遵循明确版本控制方案的 `infdev` 版本。由于主版本号 `1` 已经维持了十多年，再加上 Minecraft 2 是个内部笑话，普遍认为这个主版本号不太可能再改变了。

### 快照版 (Snapshots)

快照版偏离了标准的 semver 方案。它们的标签格式为 `YYwWWa`，其中 `YY` 代表年份的后两位数字（例如 `23`），`WW` 代表该年的第几周（例如 `01`）。因此，例如，快照版 `23w01a` 是 2023 年第一周发布的快照。

`a` 后缀的存在是为了应对同一周发布两个快照的情况（此时第二个快照的名称可能会是 `23w01b`）。Mojang 过去偶尔会这样做。另外，这个后缀也被用于像 `20w14infinite` 这样的快照，这是[2020 年的无限维度愚人节玩笑][infinite]。

### 预发布版 (Pre-releases) 和发布候选版 (Release Candidates)

当一个快照周期即将结束时，Mojang 会开始发布所谓的预发布版。预发布版被认为是某个版本的功能完整版，并且只专注于修复漏洞。它们使用该版本的 semver 标记，后跟 `-preX`。因此，例如，1.20.2 的第一个预发布版被命名为 `1.20.2-pre1`。通常会有多个预发布版，相应地会以 `-pre2`、`-pre3` 等后缀命名。

类似地，当预发布周期完成时，Mojang 会发布发布候选版 1（在版本后加上 `-rc1`，例如 `1.20.2-rc1`）。Mojang 的目标是发布一个如果没有进一步漏洞就可以发布的候选版本。然而，如果出现意外的漏洞，也可能会有 `-rc2`、`-rc3` 等版本，与预发布版类似。

## NeoForge

NeoForge 使用一种经过调整的 semver 系统：主版本号是 Minecraft 的次版本号，次版本号是 Minecraft 的补丁版本号，而补丁版本号是“真正的” NeoForge 版本号。因此，例如，NeoForge 20.2.59 是 Minecraft 1.20.2 的第 60 个版本（从 0 开始计数）。开头的 `1` 被省略了，因为它不太可能改变，原因见[上文][minecraft]。

NeoForge 的一些地方也使用[Maven 版本范围][mvr]，例如 [`neoforge.mods.toml`][neoforgemodstoml] 文件中的 Minecraft 和 NeoForge 版本范围。这些范围与 semver 大部分兼容，但并非完全兼容（例如，它不考虑 `pre` 标签）。

## 模组

没有绝对最佳的版本控制系统。不同的开发风格、项目范围等都会影响使用何种版本控制系统的决定。有时，版本控制系统也可以结合使用。本节旨在概述一些常用的版本控制系统，并提供现实生活中的例子。

通常，一个模组的文件名看起来像 `modid-<version>.jar`。因此，如果我们的模组 ID 是 `examplemod`，版本是 `1.2.3`，我们的模组文件就会被命名为 `examplemod-1.2.3.jar`。

:::note
版本控制系统是建议，而非严格执行的规则。这尤其体现在何时以及如何更改（“提升”）版本上。如果你想使用不同的版本控制系统，没有人会阻止你。
:::

### 语义化版本控制

语义化版本控制（"semver"）由三部分组成：`主版本号.次版本号.补丁版本号`。当代码库有重大变更时，主版本号会提升，这通常与重大的新功能和漏洞修复相关。当引入次要功能时，次版本号会提升，而当更新只包含漏洞修复时，会进行补丁版本号的提升。

普遍认为，任何 `0.x.x` 版本都是开发版本，在第一个（完整）版本发布时，版本应提升至 `1.0.0`。

“次版本号用于新功能，补丁版本号用于漏洞修复”的规则在实践中常常被忽略。一个流行的例子就是 Minecraft 本身，它通过次版本号实现重大功能更新，通过补丁版本号实现次要功能更新，而漏洞修复则在快照版中进行（见上文）。

根据模组更新的频率，这些数字可能或大或小。例如，[精致装饰(Supplementaries)][supplementaries] 的版本是 `2.6.31`（在撰写本文时）。三位数甚至四位数的数字，尤其是在 `patch` 部分，是完全可能的。

### “简化版”和“扩展版” Semver

有时，semver 也可以只看到两个数字。这是一种“简化版”的 semver，或称“两段式”semver。它们的版本号只有一个 `主版本号.次版本号` 方案。这通常被一些小型模组使用，这些模组只添加了少量简单的对象，因此很少需要更新（除了 Minecraft 版本更新），常常永久停留在 `1.0` 版本。

“扩展版”semver，或称“四段式”semver，有四个数字（比如 `1.0.0.0`）。根据模组的不同，格式可能是 `主版本号.接口版本号.次版本号.补丁版本号`，或 `主版本号.次版本号.补丁版本号.热更新版本号`，或者完全不同——没有标准的做法。

对于 `主版本号.接口版本号.次版本号.补丁版本号`，`major` 版本与 `api` 版本是解耦的。这意味着 `major`（功能）部分和 `api` 部分可以独立提升。这通常被那些为其他模组开发者提供 API 的模组使用。例如，[通用机械(Mekanism)][mekanism] 目前的版本是 10.4.5.19（在撰写本文时）。

对于 `主版本号.次版本号.补丁版本号.热更新版本号`，补丁级别被分成了两部分。这是 [机械动力(Create)][create] 模组使用的方法，其目前版本为 0.5.1f（在撰写本文时）。请注意，Create 用一个字母而不是第四个数字来表示热修复，以保持与常规 semver 的兼容性。

:::info
简化版 semver、扩展版 semver、两段式 semver 和四段式 semver 并非任何形式的官方术语或标准化格式。
:::

### Alpha, Beta, Release

和 Minecraft 本身一样，模组开发也常常采用软件工程中经典的 `alpha`/`beta`/`release` 阶段，其中 `alpha` 表示不稳定/实验性版本（有时也称为 `experimental` 或 `snapshot`），`beta` 表示半稳定版本，而 `release` 表示稳定版本（有时被称为 `stable` 而非 `release`）。

一些模组使用其主版本号来表示 Minecraft 版本的更新。一个例子是 [JEI物品管理器(Just Enough Items)][jei]，它使用 `13.x.x.x` 对应 Minecraft 1.19.2，`14.x.x.x` 对应 1.19.4，`15.x.x.x` 对应 1.20.1（没有 1.19.3 和 1.20.0 的版本）。其他模组则将标签附加到模组名称后，例如 [模拟殖民地(Minecolonies)][minecolonies] 模组，在撰写本文时其版本为 `1.1.328-BETA`。

### 包含 Minecraft 版本

在文件名中包含模组对应的 Minecraft 版本是很常见的做法。这使得最终用户可以更容易地找出模组适用于哪个 Minecraft 版本。常见的位置是在模组版本之前或之后，前者比后者更普遍。例如，JEI 版本 `16.0.0.28`（撰写本文时的最新版）对应 1.20.2，文件名可能变为 `jei-1.20.2-16.0.0.28` 或 `jei-16.0.0.28-1.20.2`。

### 包含模组加载器

你可能知道，NeoForge 并不是唯一的模组加载器，许多模组开发者会在多个平台上进行开发。因此，需要一种方法来区分同一模组、同一版本但用于不同模组加载器的两个文件。

通常，这是通过在名称的某个位置包含模组加载器来实现的。`jei-neoforge-1.20.2-16.0.0.28`、`jei-1.20.2-neoforge-16.0.0.28` 或 `jei-1.20.2-16.0.0.28-neoforge` 都是有效的做法。对于其他模组加载器，`neoforge` 部分将被替换为 `forge`、`fabric`、`quilt` 或你可能与 NeoForge 并行开发的其他任何模组加载器。

### 关于 Maven 的说明

Maven，这个用于依赖托管的系统，使用一种在某些细节上与 semver 不同的版本控制系统（尽管 `major.minor.patch` 的基本模式保持不变）。相关的[Maven 版本范围（MVR）][mvr]系统在 NeoForge 的一些地方被使用（见[上文][neoforge]）。在选择你的版本控制方案时，你应该确保它与 MVR 兼容，否则，其他模组将无法依赖你模组的特定版本！

[create]: https://www.curseforge.com/minecraft/mc-mods/create
[infinite]: https://zh.minecraft.wiki/w/20w14infinite
[jei]: https://www.curseforge.com/minecraft/mc-mods/jei
[mekanism]: https://www.curseforge.com/minecraft/mc-mods/mekanism
[minecolonies]: https://www.curseforge.com/minecraft/mc-mods/minecolonies
[minecraft]: #minecraft
[neoforgemodstoml]: modfiles.md#neoforgemodstoml
[mvr]: https://maven.apache.org/enforcer/enforcer-rules/versionRanges.html
[mvr]: https://maven.apache.org/ref/3.5.2/maven-artifact/apidocs/org/apache/maven/artifact/versioning/ComparableVersion.html
[neoforge]: #neoforge
[pre]: #pre-releases
[rc]: #release-candidates
[semver]: https://semver.org/
[supplementaries]: https://www.curseforge.com/minecraft/mc-mods/supplementaries