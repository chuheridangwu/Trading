# HANDOFF · 操盘手交易日志

> 本文件是会话交接文档。新会话请先读此文件再开始工作。

## 项目简介

单文件交易日志 Web 应用 `index.html`（约 2500 行，无构建），深色 GitHub 风格。功能：交易日志记录、自动评测评分（R倍数/风险%/盈亏比/纪律）、仪表盘统计（资金曲线/R分布/策略拆分）、分批买入、持仓浮动盈亏、选币筛子、指标参考、进步指南、工具箱、数据管理（导入/导出 JSON/CSV）。

## 技术架构

- 单文件 `index.html`，双击即可运行（支持 Windows/macOS），纯前端无构建
- 数据存储：**Supabase 同步 + localStorage 兜底**双模，静默回退
  - Supabase URL: `https://rinxjijqpknwtwwqozru.supabase.co`（publishable key）
  - 表：`trading_entries`（key=entries_all）、`trading_checklist`（key=checklist_main），列 `id/data/updated_at`，upsert 写入
- localStorage key：
  - `tj_entries_v1`（主数据）、`tj_entries_v1_backup`（备份）
  - `tj_checklist_v2`（开仓清单）
  - `tj_live_prices_v1`（持仓浮动现价，独立于统计，不污染正式数据）
- CDN 依赖：`@supabase/supabase-js@2`（加载失败自动走本地模式，不报错）

## 数据模型（每条 entry）

```
id, date, market, symbol, direction(long/short), setup, timeframe,
tags[], entry, exit, size, stop, target, targets[], account, fees,
leverage, marginMode(isolated/cross), lots[], trend, regime,
entryReason, exitReason, status(open/closed),
emotionEntry/Hold/After, emotionNote, mistakes, lessons, redo,
selfScore{planExecution,entryTiming,exitTiming,riskMgmt,emotionCtrl}(1-5),
img, savedAt, eval{...}, notes[]
```

- `lots[]`：分批买入 `[{price,size}]`，保存时加权均价写入 `entry`、总量写入 `size`，每档存原值
- `eval` 保存时由 `evaluate(d)` 生成，核心：`pnl, rMult, riskPct, planRR, score, grade, issues, goods, warns, summary`

## 业务规则参数（改动前务必核对）

- 单笔风险：≤2% 合格 / >2% 警告 / >5% 严重（`riskPct = |entry-stop|*size/account`）
- 计划盈亏比：≥1:2 合格，<1:2 警告
- R 倍数：≤-1.5R 严重（移止损/扛单）、-1~-1.5R 亏损超标、≥1R 达标、≥2R 优秀
- 杠杆：现货填 1；≥20x 严重、>10x 警告；止损必须在估算爆仓价之内
- 自评 5 项各 1-5 分，均分×20 为评分基础，再按风险/盈亏比/R/复盘深度/情绪/杠杆扣分加
- 等级：A≥85、B≥70、C≥50、D<50
- 负面情绪项：FOMO追单、报复性、恐慌提前砍、死扛不认错、贪婪加仓
- R 分布分档（仪表盘）：≤-2、-2~-1、-1~0、0~1、1~2、2~3、≥3（负R红 / 0~1R黄 / ≥1R绿）
- 止损为表单必填项（强制，历史兼容）

## 已完成功能

- 仪表盘：统计卡片（总数/盈亏/胜率/平均R/评分/最佳最差）、资金曲线（含最大回撤）、近期日志、当前持仓
- R 倍数分布直方图（`renderRDist`）、策略绩效拆分（`renderSetupBreakdown`，按 setup 分组，仅统计已平仓）
- 持仓浮动盈亏（`updateLive`）：输入现价算浮动 PnL/R/距入场%/距止损%/距目标%，存 `livePrices`，平仓自动清除，不计入统计
- 分批买入/补仓（`renderLots`/`syncLotsToEntry`）：多档买入价+数量（或买入金额=币值），自动算加权均价与总数量，锁定 entry/size 字段；详情页展示每档
- **买入金额→数量**（`syncAmtToSize`，新建日志）：只填入场价 + 买入金额 U（**币值/名义**），自动算 `数量=金额÷价、保证金=金额÷杠杆`，实时联动 f_entry/f_leverage/f_amt；有分批买入档位时自动让位
- **加仓爆仓模拟器**（工具箱 `computeSim`）：账户总资金/杠杆/当前价/逐仓or全仓 + 多档「买入价+买入金额U」→ 实时算每档数量、保证金、加权均价、逐仓各档爆仓价、全仓综合爆仓价（`加权均价−(钱包余额−MMR总额)÷总数量`），含保证金占比与爆仓距离警告、全仓「不会强平」判定
- **2026-08 已移除**：仪表盘『震荡幅度计算器 calcSwing / 仓位计算器 calcPosition』、工具箱『智能仓位控制 calcSmartSize』及其 sz* JS 已整体删除（UI+JS+init 调用一并清理，避免残留元素引用抛错）。导航顺序改为：仪表盘→工具箱→新建日志→日志列表→指标参考→选币筛子→进步指南→数据管理。所需算法仍保留在加仓爆仓模拟器（`computeSim`）与新建日志实时测算（`liveCalc`）中
- 新建/编辑/列表/详情/笔记/导入导出/清单/筛子/指南/工具箱
- 自动评测引擎（`evaluate`）：扣分项 + 改进建议 + 总结语

## 核心公式速查（币安 U 本位算法）

> **语义约定（用户已确认，2026-08）**：模拟器/新建日志/智能仓位控制里的「投入金额 / 买入金额 U」一律指**买入的币值（名义价值）**，不是保证金、不是投入本金。固定按此，不提供切换。
> 保证金 = 金额 ÷ 杠杆；数量 = 金额 ÷ 买入价。金额栏的填写量决定持仓名义价值，与买入价无关。

- 名义/币值换算：`数量 = 金额(U) ÷ 买入价`；`保证金 = 金额 ÷ 杠杆`
- **维持保证金率 MMR**：按名义价值阶梯（<5万=0.4%、≤25万=0.5%、≤100万=1%、≤500万=2.5%、≤2500万=5%…），整仓统一取档 `MM_TIERS`
- **逐仓强平价**（含 MMR + 手续费，多头）：`入场价 × (1 − 1/杠杆 + MMR + 0.05%)`；空头：`入场价 × (1 + 1/杠杆 − MMR − 0.05%)`。每档独立，最多亏该档保证金；**只由杠杆与 MMR 决定，与仓位大小无关**
- **全仓强平价**：触发条件 `钱包余额 + 浮动盈亏 = 维持保证金总额`，反推
  - 多头：`加权均价 − (钱包余额 − 维持保证金总额) ÷ 总数量`
  - 空头：`加权均价 + (钱包余额 − 维持保证金总额) ÷ 总数量`
  - 担保池 = **钱包全部余额**，非占用保证金；破产价（亏光）略低于强平价
  - 若结果为负（担保远大于仓位），界面显示「不会强平」
- 固定风险仓位：`数量 = 账户 × 风险% ÷ |入场−止损|`
- 止损有效性（多头）：止损必须 > 逐仓强平价，否则先强平后止损
- 币安按**标记价格**实时估算强平价，工具为预估值；账户有其他持仓/挂单/资金费率时需重算

## 踩过的坑

1. **Supabase publishable key 写入权限依赖 RLS**：key 暴露在前端，若 RLS 未锁好，任何人可覆盖/清空数据。未验证，待办。
2. **入场价/仓位是派生字段**：分批买入时会锁定 `f_entry`/`f_size`（readOnly），清空买入档位才会解锁手填。
3. **浮点显示**：加权均价用 `String(+avg.toFixed(8))` 去尾零；盈亏显示用 `toLocaleString`（min2/max8 小数位）。
4. **输入框 onchange 重建 DOM**：`renderLots`/`renderTargets`/`renderSimLots` 每次变更重建列表会失焦，沿用既有模式，勿改。
5. **表单 reset 不清 readOnly**：`resetForm` 里必须靠 `renderLots()`→`syncLotsToEntry()` 重置 readOnly，不能依赖 form.reset()。
6. **工作区曾混入未提交的历史改动**（margin-mode 按钮组、param-group 表单分组样式），提交前先 `git status` 区分，勿误提交无关文件。
7. 数据加载兼容：旧数据无 `eval`/`lots`/`targets` 字段，所有读取需 `?.`/判空兜底。
8. **全仓强平价模型（币安式）**：担保池 = 钱包全部余额（用户填的账户资金），触发条件 `钱包余额+浮动盈亏=维持保证金总额`，即 `强平价=avg−(余额−MMR总额)/Q`；MMR 按名义阶梯自动取档。曾踩坑：误用"总投入保证金"当担保品导致强平价过近（10000U 账户 3×200U 20x → 错误 0.714，正确全仓 0.128、逐仓 0.62/0.67/0.93）。未填余额时按占用保证金计算并在界面标注更保守。
9. **分批买入金额/数量互斥**：档位内改『数量』会删除该档 `amt`（数量手动优先）；改『金额』自动重算数量。`amt` 不持久化，编辑时反推 `金额=数量×价÷杠杆`。
10. **函数定义顺序**：`renderSimLots`/`computeSim` 在脚本加载末尾立即执行（无档直接 return），依赖 `priceFmt` 先于其定义（在「价格格式化」区），勿把其前移。
11. **币安强平价是参考值**：用标记价格算，含维持保证金率(MMR 阶梯 0.4%~15%)、taker 手续费(0.05%)近似，忽略资金费率/精确强平手续费；全仓若有其他持仓/挂单会改变担保池。界面已加说明。
12. **模拟器金额语义曾是"保证金"（踩坑）**：旧版 `数量=金额×杠杆÷价` 把输入当占用保证金，导致"每档买 200U"实际变成名义 4000U（20x），数量与保证金被放大。用户确认 200U=买入币值后已改 `数量=金额÷价、保证金=金额÷杠杆`，全仓强平价随之变远（10000U 账户 3×200U 20x 全仓不会强平、逐仓 0.62/0.67/0.93）。改此语义务必同步 simHint/sim 表格列/stat 文案。
13. **语义统一范围**：除模拟器外，新建日志的 `syncAmtToSize`/`renderLots`/`syncLotsToEntry`/回填恢复、`liveCalc`/`evaluate` 的爆仓公式、智能仓位 `calcSmartSize` 也都存在同款"金额=保证金"旧语义与缺 MMR/手续费、负强平价被硬夹 0.000001 的问题，已一并统一为名义币值 + 币安算法（含 `不会强平` 显示）。改任意一处金额/爆仓逻辑，其余三处要同步核对。
14. **空单全仓强平价**：空单强平价恒在均价上方、钱包越大越远但永远是真实数值（币安显示如 14188U 钱包 3×200U 做空 20x ≈ 17.7），不能像多单那样判"不会强平"（只有多单才会因担保远超仓位出现负强平价）。`noLiq=liqCross<=0` 只对多单成立。

## 下一步计划（待办，按优先级）

- [ ] **Supabase RLS 检查与加固**（最高优先，安全）
- [ ] 纪律评分：计划目标 vs 实际出场对照，从 checklist 勾选变成自动对照
- [ ] 自动定时导出本地备份（当前仅手动）
- [ ] 按方向（多/空）与按币种拆分统计
- [ ] PWA 支持，手机端随时记录

## 验证方式

```bash
# 提取内联 JS 并做语法检查（每次改动后必跑）
awk '/^<script>$/{f=1;next} /^<\/script>/{f=0} f' index.html > /tmp/tj_check.js && node --check /tmp/tj_check.js
```

浏览器验证：双击 `index.html`，重点回归：新建日志（含分批买入/止盈目标/自评）、仪表盘各图表、持仓浮动盈亏、编辑回填。

## Git 工作流（约定）

- remote: `https://github.com/chuheridangwu/Trading.git`（branch: main）
- 每次修改完成后：语法检查 → `git add` 相关文件 → 中文简短 commit（如「新增分批买入功能」）→ `git push origin main`
- 除非用户明确要求，不添加注释；提交前检查 `git status` 避免混入无关改动
