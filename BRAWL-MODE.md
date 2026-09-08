# 大乱斗模式（Brawl Mode）+ Max随机

在上游 `zyz9408/perfect-player` 基础上新增的自定义内容。分支：`brawl`（从 `base` = pinned commit `efca6764` 拉出）。

## ⚡ Max随机（建球员一键最优）

建球员界面「🎲 随机球队 / 👥 更换球员」旁边新增紫粉色按钮 **「⚡ Max随机」**。

点一下就会：
1. 自动扫描全部 30 队的建球员候选池（现役 + 历史惊喜卡）
2. 对 13 项属性各自反复随机「抽球队 → 抽一批 5 人」，每项都留下调整后最高的一个
3. 算总评；没到目标（默认 **OVR ≥ 97**，见 `SIM_CONFIG.BUILD.MAX_RANDOM_TARGET_OVR`）就整套重刷，最多 400 次
4. 还刷不到就分阶段放宽「跨位置衰减」（40% 后下限 0.90，70% 后完全无视），确保能冲上目标
5. 弹窗汇报结果：最终 OVR、是否达标、随机了多少次、以及每项属性来自哪支球队的哪名球员（★ = 历史卡，⚡ = 该项放宽了跨位置衰减，`(原 99×0.90)` = 衰减明细）。控制台另有 `console.table` 完整表格
6. 点「揭晓球员」进入正常的揭幕页

实测：PG / SG / PF / C 基本第 1 次随机就能到 97~98；SF 因为权重分散在内防/篮板，会触发放宽后到 97。整个过程 < 1 秒。

**参数**：`assets/js/config.js` → `SIM_CONFIG.BUILD.MAX_RANDOM_TARGET_OVR`（默认 97，范围 60~99）。控制台可热改，例如 `SIM_CONFIG.BUILD.MAX_RANDOM_TARGET_OVR = 99`。

**实现**：`nba-perfect-player.html` 内 `maxRandomBuild()` / `_runMaxRandom()` / `_showMaxRandomResult()` / `_collectMaxRandomPools()` / `getMaxRandomButtonHtml()`（就在 `rerollTeamPlayers` 下方）。按钮注入在 `updateSlotButtons()` 和 `buildSlotHTML()`。

> 说明：Max随机的候选池**始终**包含历史惊喜卡（不管是否大乱斗模式），这样才好稳定冲到 97。想只用现役球员来 Max随机，跟我说一声改一行即可。

---

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
