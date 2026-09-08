# 大乱斗模式（Brawl Mode）+ Max随机

在上游 `zyz9408/perfect-player` 基础上新增的自定义内容。分支：`brawl`（从 `base` = pinned commit `efca6764` 拉出）。

## ⚡ 随机组建到指定总评

建球员界面「🎲 随机球队 / 👥 更换球员」旁边新增 **一个数字输入框（默认 97）+ 紫粉色按钮「⚡ 随机到该总评」**。

在框里填任意目标总评（40~99），点按钮：

1. 扫描全部 30 队的建球员候选池（现役 + 历史惊喜卡）
2. **阶段 A（冲高）**：13 项属性各自反复随机「抽球队 → 抽一批 5 人」，每项留下调整后最高的；算总评，没到目标就整套重刷（最多 400 次），还不到就分阶段放宽「跨位置衰减」（40% 后下限 0.90，70% 后完全无视），把总评顶到能达到的最高值
3. **阶段 B（降配）**：如果最高值比目标还高（填 85、75 这种就会），按「权重和为 1」的原理把 13 项统一下调 `最高OVR − 目标` 点，再为每一项随机搜一名「该属性 ≈ 目标值」的真实球员，落到目标 ±1
4. 弹窗汇报：
   - `✅ OVR 97（目标 97，已达标）` — 直接冲到
   - `🎯 OVR 85（目标 85，已按目标降配；可冲到 97）` — 走了降配
   - `⚠️ OVR 96（目标 99，数据上限）` — 目标太高，尽力了
   - 下面逐项列出：等级、值、来自哪支球队的哪名球员（★ 历史卡，⚡ 放宽了跨位置衰减，`(原 99×0.90)` 衰减明细）。控制台另有 `console.table`
5. 点「揭晓球员」进正常揭幕页

**实测**（都 < 0.3 秒，`hit=true`）：

| 位置 · 目标 | 结果 | 方式 |
|---|---|---|
| PG 97 | OVR 97 | 冲高（第 1 次） |
| SF 99 | OVR 99 | 冲高 + 放宽衰减（~280 次） |
| PG 85 / SG 75 / C 90 | OVR 84 / 75 / 90 | 降配 |
| PF 60 | OVR 62 | 降配（属性有下限，压不到 60） |

**默认值**：`assets/js/config.js` → `SIM_CONFIG.BUILD.MAX_RANDOM_TARGET_OVR`（输入框的初始值，默认 97）。输入框里改的目标只对当次生效。

**实现**：`nba-perfect-player.html` 内 `maxRandomBuild()`（读输入框）/ `_runMaxRandom(target)`（阶段 A+B）/ `_searchAttrValue()` / `_commitMaxRandom()` / `_showMaxRandomResult()` / `_collectMaxRandomPools()` / `getMaxRandomButtonHtml()`（`rerollTeamPlayers` 下方）。输入框+按钮注入在 `updateSlotButtons()` 和 `buildSlotHTML()`；`_maxRandomTargetInput` 保证按钮区重建时输入值不丢。

> 说明：候选池**始终**包含历史惊喜卡（不管是否大乱斗模式），这样高目标才稳。想「非大乱斗时只用现役球员」跟我说，改 `_collectMaxRandomPools` 一行即可。

---

## 🧭 双位置自建球员

选位置界面现在可以**选 1~2 个位置**（点第二个 = 副位置，再点一次取消，满 2 个再点别的会替换副位置）。

- 锁属性时的跨位置衰减，取「主位置」「副位置」里**衰减最小的那个** → 双位置基本等于在两个位置范围内自由抢属性，跨位置损失大幅减少，`⚡ 随机到该总评` 需要的抽取次数也明显下降。
  - 实测：纯 SF 冲 97 要 ~10 万次抽取 + 放宽衰减；**SF/PF 一次组建、~500 次抽取、OVR 98、无需放宽**。
- 总评（`calcOVR`）和赛季里的角色/首发判定仍按**主位置**算，副位置只影响「抢属性」这一步。
- 揭幕页、建球员页会显示「控球后卫 / 得分后卫」这样的双位置标签。
- 实现：`getUserPositions()` / `getPosPenalty()`（支持数组/斜杠串，取 min 衰减）/ `togglePositionSelect()` / `getUserPositionLabel()`；`STATE.position2` / `STATE.finalPosition2`。

## 🧲 自由球员加盟倾向（按球队总评）

休赛期分配自由球员时，不再单纯按上赛季战绩由弱到强分配，而是按「加盟意向分」：

- **球队越强（roster 前 9 人平均 OVR 越高），越能吸引自由球员**；强援（OVR ≥ 86）对强队的偏好更极端，且强队即使该位置已经很满也能签下（抱团）。
- **太弱的球队（比联盟均值弱 ≥ 3.5 强度点）会被强援直接拒绝**（80% 概率）。
- 边缘球员（低 OVR）仍会被弱队的空位轻微吸引，保证垫底队也能补满人。
- 每支球队每个休赛期最多签 1 名强援，避免一年之内五星全挤一队；多个赛季累积下来强队滚雪球成超级球队。

实测（60 次单强援 / 15 轮混合池）：

| 自由球员档次 | 平均落位（0=联盟最强，29=最弱） |
|---|---|
| 强援 OVR 90+ | **~6.5**（前几名球队） |
| 中产 OVR 79 | ~12 |
| 边缘 OVR 69 | ~16 |

**参数**：`assets/js/config.js` → `SIM_CONFIG.FREE_AGENCY`（`ENABLED` / `AFFINITY_WEIGHT` / `STAR_OVR` / `STAR_REJECT_GAP` / `STAR_REJECT_CHANCE` / `RANDOM_JITTER` / `TEAM_STRENGTH_TOP_N`）。`ENABLED:false` 回到原版逻辑。

实现：`getFaTeamStrength()` + `assignFreeAgents()` 里的 `faAffinity()` / `faRejectsTeam()` / 每球员按意向重排 `orderedTeams`。

## 大乱斗：做了两件事

### 1. 把 `SIM_CONFIG` 抽成独立文件

- `assets/js/hupu/script-05-2678-qlg35lrc-upload-1783494754597-24.js` → **`assets/js/config.js`**（`git mv`，内容不变）
- `nba-perfect-player.html` 里的 `<script src>` 同步改掉
- 末尾新增 `window.SIM_CONFIG = SIM_CONFIG`，发布后可直接在浏览器控制台改参数
- `index.html` 改成一个跳转到 `nba-perfect-player.html` 的壳（原本是一份 2 万多行的旧版游戏副本，避免双份维护；`nba-perfect-player.html` 为唯一正式文件）

### 2. 大乱斗模式

**入口**（二选一）：
- 首页多出一张卡「🔥 大乱斗模式」，点它进入
- 或直接开 `nba-perfect-player.html?brawl=1`

**效果**：
- **每年休赛期的新秀里，按比例替换成历史球星**（韦德、科比、伯德、贾巴尔……），带真实姓名、位置、13 项属性、巅峰 OVR 和头像
- **允许同一个球员的不同时期版本同场竞技**：
  - 历史卡池抽完会自动重洗再来一轮，所以同一张卡可以在不同年份、不同球队反复出现 —— 你可能同时看到「湖人的詹姆斯」和「骑士的詹姆斯」
  - 数据集里本身带多时代版本的球星（詹姆斯 ×3、科怀 ×3、巴克利 ×2…），各版本会加时代后缀区分，如「查尔斯-巴克利·96版」「卢卡-东契奇·22版」
  - 每个历史新秀内部用合成主键 `HistRookie_N`，引擎不会因为重名把他们合并
- 普通「生涯模式」完全不受影响

## 可调参数（`assets/js/config.js` → `SIM_CONFIG.BRAWL`）

| 参数 | 默认 | 说明 |
|---|---|---|
| `ENABLED` | `false` | 由模式卡 / `?brawl=1` 打开，也可手动设 |
| `ROOKIE_HISTORICAL_RATIO` | `0.6` | 每名新秀有多大概率是历史球星（0~1）。调到 0.9 就几乎全是传奇 |
| `BUILD_HISTORICAL_DRAW_CHANCE` | `0.75` | 建球员阶段每轮出现历史惊喜卡的概率（普通模式是 0.20） |
| `ALLOW_DUPLICATE_IDENTITIES` | `true` | 允许同名/同时代版本并存（当前实现始终允许，保留开关备用） |
| `HISTORICAL_ROOKIE_AGE` | `'young'` | `'young'` = 一律 19~21 岁重新出道；`'authentic'` = 按 OVR 粗略给 22~27 岁 |
| `ERA_LABEL_SUFFIX` | `true` | 显示名后加「·03版」这类时代后缀（仅对有真实时代信息的卡生效） |
| `HISTORICAL_OVR_SCALE` | `1.0` | 历史新秀 OVR 相对巅峰值的缩放，想削弱设 0.9 左右 |
| `HISTORICAL_OVR_MIN` / `MAX` | `62` / `99` | OVR 上下限 |

> 控制台热改示例：`SIM_CONFIG.BRAWL.ROOKIE_HISTORICAL_RATIO = 0.95`

## 实现位置（都在 `nba-perfect-player.html` 同一个主 `<script>` 里）

| 改动 | 位置 |
|---|---|
| `getHistoricalSurpriseDrawChance()` —— 建球员阶段历史卡概率改为读配置 | `drawBuildPlayers` 上方 |
| `collectBrawlHistoricalPool()` / `drawBrawlHistoricalCard()` / `brawlEraLabel()` / `brawlEraSuffix()` / `makeHistoricalRookie()` | `generateRookie` 上方 |
| `generateRookie()` 顶部按 `ROOKIE_HISTORICAL_RATIO` 注入历史新秀 | `generateRookie` 内 |
| 首页「大乱斗」模式卡 + `?brawl=1` | `renderModeSelect` 内 |

历史球员数据直接复用建球员阶段已经转换好的 `window.PERFECT_PLAYER_HISTORICAL_SURPRISE_DATA`（150 名，来自 `assets/data/perfect-player-pool.json` 的 `historicalPlayers`，已是 13 属性游戏格式、头像齐全）。没有新增数据文件。

## 已验证

在本地静态服务器（`python3 -m http.server`）用浏览器控制台测过：
- `config.js` 正常加载，`window.SIM_CONFIG.BRAWL` 可读写，无报错
- 首页两张模式卡都渲染；点「大乱斗」→ 进入建角色流程（不再是「开发中」）
- `?brawl=1` 能自动开启 brawl
- 40 名新秀里约 27 名历史球星（≈ ratio 0.6）；属性/位置/OVR/时代后缀正确
- 200 次抽取里出现詹姆斯 ×3 等重复身份 —— 多时代并存生效
- `evolveLeague()` 连跑两个赛季无异常
- 普通模式下 `generateRookie()` 100 次 0 个历史球员，建球员历史卡概率仍为 0.20 —— 无回归

## 已知边界 / 后续可做

- 第一年联盟里历史新秀不多（大多数球队名单已满 18 人，只有空缺才补新秀）；随着老将逐年退役，占比会明显上升。想第一年就爆改，可另外在 `evolveLeague` 里主动替换部分现役球员（本次未做，避免动名单规模逻辑）
- 巅峰表里约 130 张卡的 `_sourceLabel` 是打包产物「1957-58」，非真实时代，已被时代后缀逻辑过滤；真正多时代的是那 ~16 名巨星
- 历史新秀头像走 `photoLocal` / `photoUrl`，个别历史卡可能是灰底占位图
