# 元件网络模块全量重构方案

> 范围：`com.ytdd9527.networksexpansion.implementation.machines.ae`（87 文件）→ `machines.cellnet`
> 状态：待开发者确认 → 拆 tasks → 分阶段实施
> 授权特许：允许改 Slimefun 物品 ID / 类名 / 包名 / 语言键 / PDC 键 / 配置键 / 数据库表名；**不做任何旧数据兼容**，测试期旧档物品与数据作废。
> 命名总纲：**剔除全部 "AE" 痕迹**（包名/类名/ID/语言键/配置键/玩家可见文案），统一走项目原生风格（Network/Cell/Drive/Chain 语系）。

## 0. 需求提示词（定稿）

> 全量重写 ae 模块并更名为 cellnet。功能零回退，以 CONTEXT.md §7 与游戏内验证为准。
> 不可触碰：CONTEXT.md §2.1 LogiTech2 反射协议、§2.2 五大约束、§5 硬规则、§6 已定稿性能决策（异步 ticker + runTask、批量抓取、轮转预算、粒子零方块访问、5 秒工作节奏）。已删除功能（堆叠机器/生成器/AeLineProfiler/排班制）禁止复活。
> 目标：包按领域重排并改名 cellnet；**所有含 AE/ae 的命名全部替换**；消灭复制粘贴（表驱动）；lore 全量重写（作者口吻、简洁、操作高亮、不堆对仗）；拆 AEAssemblyDrive；**注释全部清零——类头、方法级、行内注释一律不写**（覆盖 CONTEXT 注释人设，P6 同步写回 CONTEXT.md 与宪法）。
> 每阶段 `gradlew.bat clean build` 通过 + 功能清单验收后才进下一阶段。

## 0.1 红线（本方案全程有效，违反即返工）

1. **禁止任何内部注释**：类头、方法级、行内注释一律不写；既有注释全部清零（P4 执行），新代码零注释。例外：无。
2. 禁止触碰 §0 列出的已验证约束与已定稿性能决策。
3. 禁止复活已删除功能（堆叠机器/生成器/AeLineProfiler/排班制）。
4. 禁止写旧数据迁移/兼容层。
5. 模块外文件只准引用更新与纯追加（§3.5 清单内），禁止改其逻辑。

P6 收尾时把红线 1 写回 CONTEXT.md 与宪法（替换原"注释人设"条目）。

## 1. 现状诊断

| # | 问题 | 证据 |
| --- | --- | --- |
| 1 | "AE" 是外来词（借自 AE2），与本项目 Networks 语系格格不入 | 全模块命名/文案 |
| 2 | `assembler/`（灌装机）与 `assembly/`（装配产线）名字近义易混 | 两包并列 |
| 3 | `common/`（9）与 `util/`（3）职责重叠 | ItemKey 在 common，NumberFormat 在 util |
| 4 | 15 档元件 × 4 处手写复制（ItemStacks/Items/Recipes/SetupUtil） | SetupUtil.java:165-180 |
| 5 | 15 段同构 lore 手写 yml | zh-CN.yml:4577-4719 |
| 6 | AEAssemblyDrive 1772 行巨型类 | CONTEXT §7 已立项待拆 |
| 7 | 挡位阶梯不规律 | GEAR_LADDER 4倍/2倍/盒格梗混搭 |
| 8 | Manager 后缀泛滥 | drive/、storage/ |
| 9 | 文档漂移：04-ae-system.md 记 api/ 7 文件，实际 2 个 | AECellApi 等已删 |

## 2. 目标结构树（`machines/cellnet/`）

```
machines/cellnet/
├── cell/                     元件域
│   ├── CellTier.java               【新】档位唯一事实来源（序号→ID/材质/容量/配方/升级材料）
│   ├── StorageCell.java            原 AEStorageCell（实例由 CellTier 表生成）
│   ├── VoidCell.java / VoidCellHandle.java / VoidCellSupport.java
│   ├── CellHandle.java / CellUseHandler.java
│   ├── ledger/CellLedger.java + CellPersistence.java（逻辑不动）
│   ├── rule/CellAcceptRules.java
│   └── menu/                       CellMenu / VoidCellMenu / CellMenuListener / CellLore
├── drive/                    驱动器域
│   ├── CellDrive.java              原 AEDrive
│   ├── EnderDrive.java             原 AEEnderDrive
│   ├── DriveStorage.java           原 AEDriveStorage
│   ├── DriveOwnership.java / DriveGuide.java / DriveCache.java（原 AENetworkCache）
│   ├── DriveCellSlots.java         原 AEDriveCellManager
│   ├── CellUniqueness.java         原 AECellUniquenessManager
│   ├── WhitelistStore.java         原 AEDriveWhitelistManager
│   ├── DriveMonitor.java           原 AEDriveManager（物品 ID → DRIVE_MONITOR）
│   ├── ender/                      EnderChannelController + ChannelConfigurator（原 AEEnderConfigurator）
│   └── menu/                       DriveWhitelistMenu + DriveSlotGuard + DriveUniquenessListener
│                                   + EnderVoidGuardListener + DriveCellOpenListener + DriveWhitelistListener
├── assembly/                 装配产线
│   ├── AssemblyDrive.java          拆分后：方块壳 + tick 调度 + 槽位（≤300 行）
│   ├── AssemblyRound.java          【新】craftRound/craftSlot/probeBatchCapacity/fetchMaterials
│   ├── AssemblyMonitorBridge.java  【新】监控器远程操作静态桥（自带归属校验）
│   ├── AssemblyMonitor.java / AssemblyWorkshop.java / AssemblyCard.java
│   ├── core/                       GearCore + OverclockCore + SmartCore
│   └── recipe/AssemblyCardRecipes.java
├── filler/                   灌装机
│   ├── ContainerFiller.java        原 AEStorageAssembler（逻辑不动）
│   ├── FillStrategies.java / FillerPdc.java
├── chain/                    链式传输（原 line/）
│   ├── AbstractChainMachine.java   原 AbstractAELineMachine
│   ├── ChainGuiBase.java           原 AELineGuiBase
│   ├── ChainGrabber.java / ChainPusher.java / ChainTransceiver.java（原 GrabPusher）
│   ├── ChainModule.java / ChainModuleItem.java / ChainInventoryGuard.java
│   ├── ChainRangeParticles.java / ChainTargetCache.java（逻辑不动）
├── collect/                  收纳书：CollectBook / CollectListener / CollectService
├── converter/                CellConverter / CellCleaner / CellQuantumConverter
├── ui/                       【合并】GUI 公共件
│   ├── Icons.java（原 AEIcons）/ BrowseUi.java / CellUi.java / CellMenuCommon.java / CellSlotUi.java
├── support/                  【合并 common+util】纯工具
│   ├── ItemKey.java / ItemHashMap.java / NumberFormat.java / ItemSearch.java
│   ├── GhostItems.java / NetworkUtil.java / ChatInput.java / Limits.java / RecipeLegitimacy.java
├── storage/                  持久化子系统
│   ├── CellStorageDatabase.java    原 AEStorageDatabase
│   ├── dao/CellDao.java            原 AEStorageCellController
│   ├── util/SerializeUtils.java    原 AESerializeUtils
│   └── connection/journal/schema/template/write/ 内部结构不动
├── listener/                 ExplosiveToolListener / NetworkCacheInvalidationListener / PersistenceCleanupListener
└── api/                      DriveApi.java / DriveType.java
```

注：核心共享工具 `com.balugaq.netex.utils.LineOperationUtil` 被本模块与核心链式机器共用，**不改名**（模块外，且非 AE 命名）。

## 3. 重命名总表

### 3.1 物品 ID（前缀 `NTW_EXPANSION_` 不变，剔除 AE）

| 现 ID | 新 ID |
| --- | --- |
| `AE_STORAGE_CELL_L1..L15` | `CELL_1..15` |
| `AE_STORAGE_CELL_UNLIMITED` | `CELL_INFINITY` |
| `AE_VOID_CELL` | `VOID_CELL` |
| `AE_DRIVE` | `CELL_DRIVE` |
| `AE_ENDER_DRIVE` | `ENDER_DRIVE` |
| `AE_DRIVE_MANAGER` | `DRIVE_MONITOR` |
| `AE_ASSEMBLY_DRIVE / _MONITOR / _WORKSHOP / _CARD` | `ASSEMBLY_DRIVE / ASSEMBLY_MONITOR / ASSEMBLY_WORKSHOP / ASSEMBLY_CARD` |
| `AE_OVERCLOCK_CORE / AE_SMART_CORE` | `OVERCLOCK_CORE / SMART_CORE` |
| `AE_STORAGE_ASSEMBLER` | `CONTAINER_FILLER` |
| `AE_CELL_CLEANER / AE_CELL_CONVERTER` | `CELL_CLEANER / CELL_CONVERTER` |
| `AE_ENDER_CONFIGURATOR` | `CHANNEL_CONFIGURATOR` |
| `AE_LINE_GRABBER / _PUSHER / _GRAB_PUSHER` | `CHAIN_GRABBER / CHAIN_PUSHER / CHAIN_TRANSCEIVER` |
| `AE_MODULE_RANGE / _CAPACITY / _MODE / _VANILLA / _MULTI_DIRECTION` | `CHAIN_MODULE_RANGE / CHAIN_MODULE_CAPACITY / CHAIN_MODULE_MODE / CHAIN_MODULE_VANILLA / CHAIN_MODULE_MULTI_DIRECTION` |
| `AE_COLLECT_BOOK` | `COLLECT_BOOK` |

### 3.2 玩家可见中文名（去 "AE " 前缀）

`AE 驱动器`→`元件驱动器`、`AE 存储元件`→`存储元件`、`AE 无限元件`→`无限元件`、`AE 虚空元件`→`虚空元件`、`AE 末影驱动器`→`末影驱动器`、`AE 驱动器管理器`→`驱动器监视器`、`AE 装配驱动器/监控器/工坊/卡`→`装配驱动器/装配监控器/装配工坊/装配卡`、`AE 装配超频核心/智能核心`→`超频核心/智能核心`、`AE 自动灌装机`→`自动灌装机`、`AE 元件清理台/转运机`→`元件清理台/元件转运机`、`AE 频道配置器`→`频道配置器`、`AE 线性抓取器/推送器/抓取推送器`→`链式抓取器/链式推送器/链式收发器`、`AE 模块`→`链式模块`、`AE 网络收纳书`→`网络收纳书`。

### 3.3 类名规则

- **AE 前缀全部删除**（包名已表域）；原 `AEDrive`→`CellDrive`，`NetworkRoot` 等核心类的引用同步更新（仅 import 与类名）
- 去 Manager 后缀：`AECellUniquenessManager`→`CellUniqueness`、`AEDriveWhitelistManager`→`WhitelistStore`、`AEDriveCellManager`→`DriveCellSlots`、`ChatInputManager`→`ChatInput`
- `Ae` 驼峰一律消灭；菜单 `*Menu`、守卫 `*Guard`/`*Listener`、工具无后缀
- ConnectionManager/SchemaManager 属 storage 内部生命周期管理，保留

### 3.4 语言键 / 配置键 / 数据键

| 类别 | 现 | 新 |
| --- | --- | --- |
| 语言键 | `messages.ae.*` / `icons.ae-*` | `messages.cellnet.*` / `icons.cellnet-*` |
| 配置键 | `ae-storage.*` / `ae.max-item-types` / `ae.assembly.*` / `ae-collect.enabled` | `cellnet-storage.*` / `cellnet.max-item-types` / `assembly.*` / `collect-book.enabled` |
| PDC 键（核心 Keys 类） | `AE_CELL_UUID` 等 `AE_*` 全族 | `CELL_UUID` 等去 AE 同名 |
| 方块数据键 | `ae_ender_channel` / `ae_assembly_enabled` / `ae_assembly_name` / `ae_storage_assembler_*` | `ender_channel` / `assembly_enabled` / `assembly_name` / `container_filler_*` |
| 槽位标记 | `networks:ae_converter_cell_slot_marker` 等 | 去 ae 同名 |
| 数据库 | `data/ae_storage.db`，表 `ae_*` | `data/cellnet_storage.db`，表去 `ae_` 前缀（`cell_items`/`cell_meta`/`item_templates`/`journal`/`journal_archive`/`schema_info`） |
| 末影频道表 | `ae_ender_channels` | `ender_channels` |

### 3.5 模块外牵连文件（仅引用更新/追加，不改逻辑）

`ExpansionItemStacks`、`ExpansionItems`、`ExpansionRecipes`、`SetupUtil`、`zh-CN.yml`、`Networks.java`（启动/关闭两处调用名）、`NetworkRoot`（import 与类名引用）、核心 `Keys` 类（PDC 键改名）、`ConfigManager`（getter 改名）、`ListenerManager`（监听器类名）、`Lang`（仅**新增** getItem args 重载）。

## 4. 挡位阶梯重构（已定数值）

```java
private static final long[] GEAR_LADDER = {1L, 4L, 8L, 16L, 32L, 64L, 128L, 256L, 512L, 1024L, 1728L, 3456L};
private static final int GEAR_MAX = 13;
```

- 规律：×2 直升到 1024，末两挡 1728/3456 是潜影盒 27/54 格彩蛋，保留
- 装配卡 PDC `AE_CRAFT_GEAR`→`CRAFT_GEAR` 存挡位序号，不做旧兼容（旧卡作废属预期）
- `gearMultiplier`/`applyGear`/`GEAR_LADDER.length` 相关判断逻辑不变，只换数组与上限
- 相关文案同步：`gear_level` 语言键、超频核心 lore

## 5. 表驱动改造（消灭复制粘贴）

### 5.1 CellTier（完整设计，照着写）

```java
package com.ytdd9527.networksexpansion.implementation.machines.cellnet.cell;

public enum CellTier {

    T1("1", "64", 64L, Material.MUSIC_DISC_11),
    T2("2", "256", 256L, Material.MUSIC_DISC_13),
    T3("3", "1K", 1_024L, Material.MUSIC_DISC_CAT),
    // … 容量与照现 AEStorageCellType LEVEL_0..14 逐档对应
    T15("15", "140.7T", 140_737_488_355_328L, Material.MUSIC_DISC_RELIC);

    private final String suffix;
    private final String label;
    private final long perTypeLimit;
    private final Material icon;

    public String id()           { return "NTW_EXPANSION_CELL_" + suffix; }
    public String label()        { return label; }
    public long perTypeLimit()   { return perTypeLimit; }
    public Material icon()       { return icon; }

    public SlimefunItemStack stack() {
        return Theme.themedSlimefunItemStack(
            Lang.getItem("NTW_EXPANSION_CELL", icon, label), Theme.MACHINE);
    }

    public ItemStack[] recipe() {
        // 照搬现 ExpansionRecipes.AE_STORAGE_CELL_L1..L15 对应档位内容
    }

    public ItemStack upgradeMaterial() {
        // 照搬现 AECellUpgradeMaterialRegistry 档位 → 量子存储映射
    }

    public static CellTier fromAmount(long maxAmount) {
        for (CellTier t : VALUES) if (t.perTypeLimit == maxAmount) return t;
        return null;
    }
    private static final CellTier[] VALUES = values();

    public static final class Unlimited {
        public static final String ID = "NTW_EXPANSION_CELL_INFINITY";
        public static final long PER_TYPE_LIMIT = Long.MAX_VALUE;
        // 无限档不参与升级/反查，stack/recipe 在此单写
    }
}
```

- `AEStorageCellType` 与 `AECellUpgradeMaterialRegistry` 两个文件删除，职能全部被本表吸收
- 元件物品实例不再散落在 `ExpansionItems`：注册与取用统一走 CellTier

### 5.2 表驱动注册（SetupUtil / ExpansionItems）

现状（ExpansionItems.java:1587 起，15 段复制 + SetupUtil.java:165-180 十五行）：

```java
public static final AEStorageCell AE_STORAGE_CELL_L1 = new AEStorageCell(
    group, ExpansionItemStacks.AE_STORAGE_CELL_L1, TYPE, ExpansionRecipes.AE_STORAGE_CELL_L1, 64);
// … 照抄 15 份
```

重写后：

```java
// CellTier 内
private StorageCell instance;

public StorageCell register(ItemGroup group) {
    instance = new StorageCell(group, stack(), RecipeType.ENHANCED_CRAFTING_TABLE, recipe(), perTypeLimit);
    instance.register(Networks.getInstance());
    return instance;
}

public StorageCell instance() { return instance; }
```

```java
// SetupUtil.setupItem() 内，一行替代十五行
ExpansionItemsMenus.SUB_MENU_CELLNET.addTo(Arrays.stream(CellTier.values())
        .map(t -> t.register(group)).toArray(SlimefunItem[]::new));
```

`ExpansionItems.AE_STORAGE_CELL_L1..L15` 十五个字段删除，外部取用改 `CellTier.T3.instance()`；`ExpansionItemStacks`/`ExpansionRecipes` 中十五组常量同步删除。

### 5.3 lore 模板化 + Lang 重载

zh-CN.yml 只留模板：

```yaml
NTW_EXPANSION_CELL:
  name: "存储元件 <MACHINE>({0})"
  lore:
    - "<passive>每种物品占一格 · 每格上限 <aqua>{0}"
    - ""
    - "<click_info>右键<passive> 查看内容 · 升级格数"
    - "<passive>放入<gold>元件驱动器<passive>接入网络"

NTW_EXPANSION_CELL_INFINITY:
  name: "无限元件 <MACHINE>(∞)"
  lore:
    - "<passive>每格数量无限 · 格数无限 · 无需升级"
    - ""
    - "<click_info>右键<passive> 查看内容"
    - "<passive>放入<gold>元件驱动器<passive>接入网络"
```

Lang 新增重载（纯追加，不动既有方法）：

```java
public static SlimefunItemStack getItem(String id, Material material, Object... args) {
    SlimefunItemStack stack = get().getItem(id, material);
    if (args.length > 0) {
        ItemMeta meta = stack.getItemMeta();
        meta.setDisplayName(MessageFormat.format(meta.getDisplayName(), args));
        List<String> lore = meta.getLore();
        if (lore != null) lore.replaceAll(line -> MessageFormat.format(line, args));
        stack.setItemMeta(meta);
    }
    return stack;
}
```

CellTier.stack() 里 `Lang.getItem("NTW_EXPANSION_CELL", icon, label)` 一次调用出成品。
**注意**：WikiUtils 若按 `items.<物品ID>` 逐键取文案，模板键会让 15 档元件缺条目——实施时先查 WikiUtils 取数路径，必要时改吃生成后的 SlimefunItemStack。

## 6. Lore 全量重写规范（作者口吻）

结构定式（≤6 行）：功能一句（平实说人话，不硬凑对仗）→ 关键数值（`<aqua>`）→ 空行 → 操作（`<click_info>` 高亮按键）→ 警告（`<error>`，有才写）。

禁止：四字对仗硬凑、说明书句式、颜色堆砌（≤4 种主题标签）、暴露实现细节（几挡几槽这类内部数字不进 lore）。

示例（定稿）：

```yaml
NTW_EXPANSION_ASSEMBLY_DRIVE:
  name: "装配驱动器 <MACHINE>"
  lore:
    - "<passive>插入装配卡自动生产 · 装满即停"
    - ""
    - "<click_info>左键/右键<passive> 升降挡"
    - "<click_info>Shift+左键<passive> 切模式 · <click_info>Shift+右键<passive> 定计划"

NTW_EXPANSION_CHAIN_GRABBER:
  name: "链式抓取器 <MACHINE>"
  lore:
    - "<passive>沿设定方向把容器物品抓进网络"
    - ""
    - "<click_info>手持物品点距离槽<passive> 数量即距离"
    - "<passive>插入<gold>链式模块<passive>扩展能力"
```

范围：`items.NTW_EXPANSION_*`（本模块全键）+ `messages.cellnet.*` + `icons.cellnet-*`；改完全文扫描非法 `<>` 标签（CONTEXT §5.4）。

## 7. AssemblyDrive 三刀拆分（原 AEAssemblyDrive 1772 行）

纯搬运，方法体一行不改，拆完逐方法 diff：

```java
// AssemblyDrive.java —— 方块壳（≤300 行）
public class AssemblyDrive extends NetworkObject {
    // 槽位常量：RECIPE_SLOTS / TASK_DISPLAY_SLOTS / TOGGLE / UPGRADE / SMART / CLEAR / STATUS
    // PDC 键、方块数据键（assembly_enabled / assembly_name）
    // tick 调度（异步探测 + runTask 回主线程，1s/轮，退避逻辑原样）
    // 菜单渲染与点击 handler
}

// AssemblyRound.java —— 合成引擎（包私有，纯逻辑）
final class AssemblyRound {
    static void craftRound(...);      // 整轮聚合（原 :885）
    static void craftSlot(...);       // 单产线（原 :552）
    static long probeBatchCapacity(...);  // 容量探测（原 :938/:574）
    static void fetchMaterials(...);  // withVoidSuspended 取料（原路径不动）
    static void settlePlanProgress(...);
}

// AssemblyMonitorBridge.java —— 监控器静态桥（public，自带归属校验）
public final class AssemblyMonitorBridge {
    public static DriveOverview readOverview(...);
    public static List<MonitorLine> readLines(...);
    public static void changeGear(...);       // 原 changeGearFromMonitor
    public static void cycleLine(...);        // 原 cycleLineFromMonitor
    public static void requestQuantity(...);  // 原 requestQuantityFromMonitor
    public static void setName(...);          // 原 setNameFromMonitor
    public static void handleTargetInput(...);
}
```

拆分后 `AssemblyMonitor` 只依赖 `AssemblyMonitorBridge`，`AssemblyRound` 不 import 任何菜单类。

## 8. 性能建议（只列新增）

1. 表驱动附带：档位反查 O(1) 枚举数组（`CellTier.VALUES`）
2. lore 模板按 tier 缓存生成结果，拿物品不再走 yml 拼接
3. `NetworkUtil.findRoot` 缓存键 hashCode→精确 Location（同 2.1.169 第④刀修法）
4. 枪毙项：任何"快照+定时同步"替代实读实写（约束 1）；排班制（已判定负面优化）

## 9. 防负面优化验收线

- 功能基线（P0 冻结）：圆石进网 / 元件存取与升级 / 保留位与满载清空 / 末影频道共享 / 白名单三道门 / 灌装机四容器 / 装配三模式十三挡 / 链式三机五模式 / 收纳书双模式 / 转换清理 / 监控器远程操作 / 幽灵物品四出口防刷 / 关服落盘重启恢复
- 性能基线：spark 主线程 Networks 占比 ≤0.33% 量级；BOTH.grab ≤1835µs
- 每阶段 `gradlew.bat clean build` + version 递增

## 10. 实施阶段

| 阶段 | 内容 | 风险 |
| --- | --- | --- |
| P0 | 快照源码到 bak/，冻结功能清单 + spark 基线 | 零 |
| P1 | CellTier 表 + 表驱动注册 + lore 模板化 + 全部 ID 改名 | 低 |
| P2 | 挡位阶梯 12 挡 + lore/文案全量重写（去 AE 中文名） | 低 |
| P3 | 包重命名 cellnet + 结构重排 + 类重命名（纯移动零逻辑） | 低 |
| P4 | ui/support 合并、Manager 收敛、重复守卫/分页收口 + **注释全部清零** | 中 |
| P5 | AssemblyDrive 三刀拆分 | 中 |
| P6 | PDC 键/配置键/数据库表改名 + 验收：build + 功能清单逐项游戏内过 + spark 对比 + 同步 CONTEXT.md（注释人设条目改为零注释）与 04-ae-system.md（改名 04-cellnet-system.md） | 中 |

每阶段完成暂停等开发者确认。

## 11. 交付物

- 重构后代码 + 语言文件
- 玩家功能使用指南：`specs/ae-rework/player-guide.md`（随 lore 定稿同步更新，可直接用于更新公告）
