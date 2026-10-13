# 航海奇闻（Pirates Outlaws）手机版 · 官方 mod 支持版

> 本仓库提供一个**改造后的游戏本体 APK**，让手机版能够直接加载 **Steam 创意工坊的官方 PC mod**。
>
> **下载**：请到 [Releases](https://github.com/caffetsong/andriodPiratesOutlawsModed/releases) 页面下载
>
> **校验**（SHA-256）：
> ```
> 9c7514b2f9b8c484a00a77dba06f6e5d26bfc1c9b23e585583b3e87c7af5ad30
> ```

- 包名：`com.piratesoutlaws.roguelikecard.fabled.cn`（航海奇闻 1.97）
- 前提：**免 root**；因签名与原版不同，**必须先卸载官方版**再安装本版本

---

## 一、新增能力：加载官方 PC mod

原版手机版**完全没有 mod 入口** —— 但官方那套 mod 代码其实**一行不少地在包里**：`ModManager`、`ReadTable`、`FindMapping`、`ExtraAssetsPivot`，以及 `-ModGroup` 字符串常量全都保留着。

它用不了的原因只有一个：**没有任何东西给游戏传 `-ModGroup` 参数**，于是 `UtilsConfig.UseMod` 永远是 `false`，整条 mod 管线休眠。

本版本把这个参数喂进去，并修好官方加载器在安卓上的缺陷；其余**完全走官方逻辑**，一行游戏脚本都不改。

| 项 | 原版手机版 | 本版本 |
|---|---|---|
| 加载创意工坊 mod | ✗ 不可能 | ✓ 支持；`Table/`＋`Image/`＋`Localization/` 三种目录布局都能自动识别，**不需要 `mods.txt`** |
| ID 冲突 | — | ✓ 沿用官方的自动重映射（卡 9001+ / 角色 30001+ / 遗物 3001+ …），mod 作者无需关心 |
| 多 mod 共存 | — | ✓ 同名贴图冲突**自动隔离**：就地重命名，并同步改写该 mod 自己的表引用 |
| 卡牌名 / 技能描述 | — | ✓ mod 本地化正常合并进语言表 |
| 立绘 / 卡面 / 技能图标 | — | ✓ 正常绘制 |
| 存档 | 与原版共用 | ✓ **独立 mod 存档** `save1_mod_<组>.sav`，不污染原存档 |

---

## 二、顺手修掉的 4 个**游戏自身**的缺陷

这四条都不是改 mod 引入的，而是**「PC 上没事、安卓上必炸」**的代码 —— 也正是官方 mod 在手机版从来用不了的根本原因。

| # | 位置 | 症状 | 本版本的处理 |
|---|---|---|---|
| 1 | `System.String.Replace` | `ReadTable.Read` 用 `Name.Replace(Extension, "")` 算查表 key，而 52 个调用点传的都不带扩展名 → `Replace("")` 直接抛 `ArgumentException`：**只要装了任意一个 mod 就崩在启动画面** | 让空 `oldValue` 变成「无可替换」的 no-op |
| 2 | `ModManager.GetModPaths` | 它去枚举 `…/workshop/content/1046300/`（PC 创意工坊目录）。安卓上该路径不存在，枚举它**不是抛异常，而是直接 SIGSEGV**；崩溃发生在本地 mod 已入列**之后**，所以现象极具误导性：日志显示「已加载 1 个 mod」，实际一张表都没读进来 | 接管这个纯路径发现函数，自己拼出可用的 mod 目录 |
| 3 | `ModManager.LoadAllImages` | 同名贴图「先到先得」（`if (!modSpriteDic.ContainsKey(name))`），两个 mod 提供的同名状态图标会互相覆盖 | 冲突图**就地重命名**并改写该 mod 自己的引用，各留各的图 |
| 4 | `ModeView.Open` / `AdventureControl.EndGame` | 教程里点「放弃」走的是 `EndGame`，它**从不写** `Tutorial_AllFinish`；而主界面的 `ModeView.Open` 会把它置 1 却**没有落盘** → 回主界面后被存档冲回 0 → **再进冒险又从头进教程，出不来** | 在原函数返回后调用游戏自己的 `QuickSave`，让这个标记真正落盘 |

> 第 3 条在 PC 端同样存在（属游戏自身设计），本版本顺带修好了。

---

## 三、新增的可选功能

| 功能 | 说明 | 开关（默认） |
|---|---|---|
| **跳过洗牌动画** | 抽牌堆耗尽时的洗牌动画与 2 秒空等被去除，**但搬牌逻辑一行不改**（原方法照跑，只摘掉表现层），音效保留 | `<mod>/pomod.cfg` 或 `<files>/pomod.cfg` 里写 `skip_shuffle=1`（默认**关**） |
| **战斗日志** | `<files>/battle.log`：开战环境头 + 加/删卡 + 遗物 + 用卡 + 战斗数据读写 + 条件挂载，每行带**相对开战的秒数**且即时落盘 | `<files>/no_battle_log` 关闭（默认开） |
| **跨 mod 机制隔离** | 修正「别人家的技能在你的英雄身上执行」：实测用法师以外的英雄时，法师的元素结算链会运行，并把它自己的卡塞进你手里 | `<files>/no_mod_filter` 关闭（**默认开**） |
| **相册零污染** | mod 图片放在媒体扫描目录，却不会出现在相册里 | 自动 |

---

## 四、安装

### 1. 卸载官方版

签名不同，**不能覆盖安装**。卸载会清掉本地进度；在意的话先备份
`Android/data/com.piratesoutlaws.roguelikecard.fabled.cn/files/save1china.sav`。

### 2. 安装本版本

直接安装 `dist/pirates_outlaws_modded.apk`。

### 3. 放置 mod

把创意工坊的 mod 目录（形如 `Table/` + `Image/` + `Localization/`）放到下面**任一**位置，装载器会自动发现：

```
/sdcard/Android/media/com.piratesoutlaws.roguelikecard.fabled.cn/<mod 名>/
```



---

## 五、本版本**不包含**什么
- **不含任何 mod**。这个 APK 只是把游戏改造成「能加载 mod」，mod 文件需要另外放置。
- **不修改任何游戏数值、规则与平衡**。所有改动只涉及「让 mod 能被加载」与上述 4 个缺陷修复。
- 不含广告 / 内购相关的任何改动。

---

## 六、已知限制

1. **自签名**：无法覆盖安装官方版，也无法使用官方账号相关功能。
2. **手机版资源比 PC 少**：缺少 `event_images_6`、部分 `heroposNN` 与 `StreamingAssets/ExtraAssets`。引用了这些资源的 mod 会缺图。
3. **mod 作者可能硬编码运行时 ID**：实测某 mod 的开局事件里写着 `AddHideRelics("9112")` 这种**运行时编号**（正确写法是 `ModRelicID(9001)`）。在别人的 mod 组合下，这类硬编码可能指向完全不相干的内容 —— 属 mod 侧问题，建议反馈给作者。
4. **跨 mod 联动会被隔离**：本版本默认按「某个 mod 的技能只在它自己的角色身上执行」隔离，这会同时屏蔽作者有意设计的跨 mod 联动（目前未见此类设计）。

---
