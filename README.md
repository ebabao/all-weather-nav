# 全天候策略 v1.4 · 每日自动同步与部署（aw-nav）

参照 `ebabao/style-rotation-nav`（大类资产轮动策略每日净值看板）的架构，
把**策略模型接口**与**看板页面数据**做成每天自动同步、自动部署的闭环。

> **独立仓库部署**：全天候看板已从参考工程仓库中拆出，独立发布于本仓库的 GitHub Pages。
> 站点由 `deploy-pages` 工作流产物发布（仅含 `index.html` + `data/`），不依赖 main 分支目录树，
> 因此与其他仓库的清空式发布彻底隔离。

- 线上看板：`https://ebabao.github.io/all-weather-nav/`
- 数据接口：`data/meta.json`、`nav.json`、`signal.json`、`weights.json`、`rebalances.csv`、`monthly.csv`、`sync_audit.json`
- 参考工程：`https://github.com/ebabao/style-rotation-nav`（只读参照，未改动）
- 部署：推送即自动触发每日同步与 Pages 部署；`deploy_to_github.sh` / `.ps1` 为备用的手动推送脚本

## 架构

```
新浪行情(7 ETF) ──► sync_daily.py ──► data/*.json (策略模型接口)
                        │                    │
                        │ 复用 aw14/engine14  │ mirror_to_site() 实时目录映射
                        ▼                    ▼
              quick_run(theta_active)   site/data/*.json (线上发布目录)
                                             │ fetch(cache:no-store)
                                             ▼
                                      index.html (静态看板，不重建)
```

**关键解耦**：`index.html` 是纯静态文件，不含任何内嵌数据、不引用任何外部 CDN/字体/图片；
所有数值通过 `fetch('./data/*.json', {cache:'no-store'})` 读取。因此每日只需更新 `data/` 接口，
页面刷新即自动同步，无需重建 HTML —— 这就是「策略模型接口与看板页面数据自动每天同步」。

已验证：线上发布目录为**实时目录映射**，写入 `site/data/` 后对应公开 URL 立即返回新内容，
因此每日同步只需落盘接口，无需重新发布、无需重建页面。

## 每日执行链（两步 + 明确回退）

| 步骤 | 动作 | 失败处理 |
|---|---|---|
| Step 1 | 复用 `aw14/data14.py` 抓 7 只 ETF 前复权日线，重试 ≤2，硬超时 150s | 恢复仓库内上一版 parquet 快照，`source=cache_fallback`，**不编造数据**，继续用旧快照回放 |
| Step 2 | 复用 `aw14/engine14.py` 的 `quick_run(theta_active)` 全量回放，生成 7 个接口 | 接口校验（有限性/日期单调/长度一致/as_of 一致）失败则放弃写入，沿用上一版 |
| 校验 | 与 `model/state_snapshot.json` 官方指标对账，偏差以 pp 如实披露 | `reconcile.ok=false` 时页面标黄 |
| 幂等 | 数据指纹 + 参数指纹一致 → 跳过重写，仅追加一条 `skipped` 巡检记录 | 避免无新交易日时产生无意义写入 |
| 镜像 | `mirror_to_site()` 把 7 个接口 + 页面同步到 `site/`，线上立即生效 | `site/` 不存在时跳过，不影响主流程 |
| 提交 | `git diff --cached --quiet` 判断，有变化才 `chore(nav): 更新每日净值看板` | 保持参考工程的提交风格 |

运行指标全部写入 `data/sync_audit.json`：`run_id / scraped / new / md / status / source / retry / fallback / reconcile_pp`。

## 定时与兜底（两套并行）

**GitHub Actions 链路（已上线）**：
- `.github/workflows/daily_sync.yml`：cron `20 11 * * *`（UTC 11:20 = 北京 19:20，收盘后），
  抓数 → 回放 → 生成接口 → 提交 → `configure-pages` + `upload-pages-artifact(site/)` + `deploy-pages`。
  站点产物只包含页面与数据接口，脚本/模型文件不对外暴露。
- `.github/workflows/watchdog.yml`：每 3 小时检查最近巡检记录，**超过 26 小时无同步即自动 `workflow_dispatch` 重启**
  `daily_sync.yml`（`mode=force`），与参考工程外部 bot 停更兜底同一思路。
- 手动重启：Actions → Run workflow，可选 `sync` / `retrain` / `force`、`no_refresh`。
- `on: push` 覆盖 `scripts/**`、`model/**` 与工作流自身变更，改工程即自动重跑并部署。

> 独立仓库 + artifact 发布后，站点不再依赖 main 分支目录树，也不再与参考工程共用发布源，
> 避免两个自动化互相清空覆盖导致页面 404。

## 参数配置分离与一键回退

`model/active_params.json` 为**生效参数**（15 维，遗传算法 164 个体 / 8 代训练所得，参数指纹 `70faa838df`）；
引擎中的 `THETA0` 保持默认基准不变。

> 删除 `model/active_params.json` 并重跑两步流水线，即回退到默认基准参数（与参考工程 `active_params.json` 同一做法）。

`--retrain` 为**按需**参数自进化入口（不在每日链上，避免参数漂移）：调用 `aw14/learn14.py` 的
`run_learning()` 重训并更新生效参数，失败则保留旧参数、不阻断同步。

## 本地复现

```bash
pip install -r requirements.txt
python3.11 scripts/build_page.py          # 生成静态 index.html（通常只需一次）
python3.11 scripts/sync_daily.py          # 完整每日同步（抓数 + 回放 + 接口 + 线上镜像）
python3.11 scripts/sync_daily.py --no-refresh   # 跳过抓数，用仓库内快照
python3.11 scripts/sync_daily.py --force        # 忽略幂等强制重写
python3.11 -m http.server 8000 --directory site   # 本地预览 http://localhost:8000
./deploy_to_github.sh ebabao/all-weather-nav   # 有写权限时推送并启用云端每日链
```

> 必须通过 HTTP 访问；浏览器直接打开 `file://` 无法读取 `data/` 目录，页面会给出明确提示。

## 目录

```
index.html                     静态看板（零外部依赖、零内嵌数据）
.nojekyll                      GitHub Pages 保留 data/ 目录
data/                          公开数据接口（每日同步产物）
site/                          线上发布目录（index.html + data/，实时映射）
model/
  active_params.json           生效参数（可删除回退）
  state_snapshot.json          v1.4 训练快照（官方指标，用于模型对账）
  all_weather_prototype.py     复用 v1.0 求解器
  aw/ aw12/ aw14/              复用现有数据层与引擎，未重写
  aw14_out/data/*.parquet      7 只 ETF 前复权价格/收益/可用性快照（抓数失败时的回退源）
scripts/
  sync_daily.py                每日同步主程序
  build_page.py                生成静态页面
.github/workflows/
  daily_sync.yml               每日定时同步 + 自动部署
  watchdog.yml                 停更巡检 + 自动重启
deploy_to_github.sh / .ps1     一键推送并启用云端每日链
requirements.txt
```

## 如实披露的边界

- 评估期自 **2019-12-06** 起（豆粕 ETF 上市后 7 只齐全），此前真实历史仅用于协方差预热；**代理格 0、无拼接、无编造**。
- 上市前段、QDII 场内溢价与汇率影响未剥离；回测不代表未来收益。
- 参数为训练数据调优所得，页面「模型对账」披露与 2026-11-08 训练快照的偏差；数据刷新后偏差为真实增量。
- 三项量化参考线（夏普 ≥1.5 / 回撤 ≤8% / 年化 ≥10%）按实际回放结果判定，未达成如实标注。
- `source=cache_fallback` 表示当日抓数失败、沿用上一版快照，页面顶部同步状态条会标黄提示。
- 当前线上站点为 GitHub Pages（`deploy-pages` 工作流产物）；`source=cache_fallback` 时页面顶部同步状态条标黄提示。

本仓库仅用于策略研究与工程演示，不构成投资建议。
