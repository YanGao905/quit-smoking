# 戒烟账本（Quit Ledger）— 项目备忘

单文件手机端 H5 应用，帮双方记录抽烟数量并互相监督结算。

- 线上地址：https://yangao905.github.io/quit-smoking/
- 仓库：https://github.com/YanGao905/quit-smoking （公开）
- 技术：纯单文件 `index.html`（内联 CSS/JS，无构建、无依赖）

## 数据与同步

- 本地缓存：`localStorage` key `quit-smoking-v1`
- 云同步：Supabase（`vkctifsaunpatmwxyuuk.supabase.co`），`ledger` 表单行共享 JSON（`id='shared'`）
  - 浏览器端用 publishable key（`sb_publishable_...`，可公开）
  - 策略：整块 last-write-wins，去抖推送（600ms）+ 每 4s 轮询 + 切前台/可见性变化时拉取
- 双方共用同一本账本，一方改动几秒内对方可见
- 顶部同步状态灯（胶囊标签）：🟢 SYNCED / 🟡 SYNCING（脉冲）/ 🔴 OFFLINE

## 金额规则

- 每条记录快照存 `count / quota / price`
- 欠款 = `max(0, count - quota) * price`（前 quota 根免费，之后每根 price）
- 货币符号：JS 常量 `const CUR='$';`（当前美元 USD）。改币种改这一处即可；HTML 里另有 3 处硬编码符号（`todayDue` / `totalDueMini` / `totalBig` 的 `<small>`）需同步改。

## 结构：两个 tab，按角色分工

**核心理念**：抽烟的人和收钱的人看不同的东西。真实使用场景是「过了半夜一次性记昨天抽了多少根」，不是实时累加今天。

- **Log tab（`page-log`）—— 给戒烟本人，只讲香烟、不谈钱**
  - 顶部大卡固定对准**昨天**（`logDate=defaultLogDate()`，每次 `renderLog` 重算；无左右切换，看/改别的天用下面的 Recent days 列表）；点大数字 `openSheet(logDate)` 打字；`−/+/+5` = `addLog()`。
  - `renderCatchup()`：更早没记的日子提示补登（点一下把该日载入大卡）。
  - `renderWeekStats()`：近 7 天香烟总数 + 日均。
  - `drawCig()`：每天香烟曲线（7 天视图标数字，见下）。
  - `renderLogRows()`：**可编辑**的「Recent days」列表（`#logRows`），只显示香烟根数 + 铅笔，点行 `openSheet(d)` 改。**修改功能集中在这里。**
- **Money tab（`page-money`）—— 给收钱的人，只讲钱、只读**
  - 总欠款卡 + `drawMoney()` 累计欠款曲线（全时段 `seriesForRange(0)`）+ 规则设置。
  - `renderMoney()` 的日账单（`#billRows`）**只读**：日期 + 当天欠款，无铅笔、不可点（防止收钱的人改抽烟数）。

导航只有 2 个（`pages={log:'page-log',money:'page-money'}`）。

## 功能行为（易忘的产品细节）

- **补登/改**：全在 Log tab —— 大卡切日子、或点 Recent days 行开 `openSheet`。Money tab 不能改。
- **曲线数字标注**（`drawCig()`）：每天抽烟数量标在点上，超标 `--over`(红)、达标 `--ink`(黑)。只在 `if(data.length<=7)`（7 天视图）显示；30 天/All 不标。
- **庆祝弹窗**（`maybeCelebrate()`）：连续达标满 **7 天**才弹（`Math.floor(streak/7)*7`，阈值 `>=7`）；断 streak 后重新连满会再弹。改阈值只改 `/7` 和 `>=7`。
  注意：Log 卡片上的小 `streak-badge` 是另一元素，≥2 天就显示，与弹窗无关。
- **今天种子**：`ensureTodaySeed()` 仍会把今天预填到额度（保留旧行为、不改数据）；Log 默认记昨天，今天照旧自动出现在列表/曲线里（额度值、$0）。

## 本地测试技巧（不污染线上账本）

线上是双方共用的真实账本，**严禁写测试数据**。要看多天数据/触发弹窗等效果时，
复制 `index.html` 到临时文件，做两处改造后本地 `python3 -m http.server` 预览：
1. 把启动处 `cloudInit();` 换成 `renderAll();maybeCelebrate();`（跳过联网）
2. 在 `let db=load();` 后注入构造好的 `db.records`（含目标日期/连续天数），并 stub `cloudPush` 为 no-op

## 部署（重要：当前很慢，有更快方案）

当前用的是 **GitHub Pages 老 Jekyll 构建**，每次改动要 5~10 分钟，还偶发报错。
必须保留根目录 `.nojekyll`（内联 JS 里的 `{{`/`{%` 会被 Jekyll 误解析导致构建失败）。

```bash
cd /tmp/quit-smoking-deploy   # 或重新 clone 仓库
git add index.html
git commit -m "..."
git push origin main
# 然后等 GitHub Pages 构建（慢）
```

### ⚡ 一劳永逸提速：切到 GitHub Actions 部署（30~60s 稳定）

被阻塞点：git 令牌缺 `workflow` 权限，push 不了 `.github/workflows/*.yml`（GitHub 直接拒绝）。
用户只需在自己电脑上跑一次授权命令，之后就能改用 Actions 直接上传静态文件、绕过 Jekyll：

```bash
gh auth refresh -s workflow -h github.com
```

授权后：加一个上传静态站点的 workflow（`actions/upload-pages-artifact` + `actions/deploy-pages`），
并把 Pages 构建方式切成 workflow：`gh api -X PUT repos/YanGao905/quit-smoking/pages -f build_type=workflow`。

## 坑

1. 保留 `.nojekyll`，否则 Jekyll 构建必失败
2. 不要往线上共享账本写测试数据（会覆盖用户真实数据，last-write-wins）
3. Pages 老构建偶发报错：报错就重新触发 `gh api -X POST repos/YanGao905/quit-smoking/pages/builds`
