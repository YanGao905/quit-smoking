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
- 欠款 = `max(0, count - quota) * price`（前 quota 根免费，之后每根 price 元）

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
