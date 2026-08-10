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

- 仪表盘：统计卡片（总数/盈亏/胜率/平均R/评分/最佳最差）、资金曲线（含最大回撤）、近期日志、当前持仓、震荡幅度/仓位计算器
- R 倍数分布直方图（`renderRDist`）、策略绩效拆分（`renderSetupBreakdown`，按 setup 分组，仅统计已平仓）
- 持仓浮动盈亏（`updateLive`）：输入现价算浮动 PnL/R/距入场%/距止损%/距目标%，存 `livePrices`，平仓自动清除，不计入统计
- 分批买入/补仓（`renderLots`/`syncLotsToEntry`）：多档买入价+数量，自动算加权均价与总数量，锁定 entry/size 字段；详情页展示每档
- 新建/编辑/列表/详情/笔记/导入导出/清单/筛子/指南/工具箱
- 自动评测引擎（`evaluate`）：扣分项 + 改进建议 + 总结语

## 踩过的坑

1. **Supabase publishable key 写入权限依赖 RLS**：key 暴露在前端，若 RLS 未锁好，任何人可覆盖/清空数据。未验证，待办。
2. **入场价/仓位是派生字段**：分批买入时会锁定 `f_entry`/`f_size`（readOnly），清空买入档位才会解锁手填。
3. **浮点显示**：加权均价用 `String(+avg.toFixed(8))` 去尾零；盈亏显示用 `toLocaleString`（min2/max8 小数位）。
4. **输入框 onchange 重建 DOM**：`renderLots`/`renderTargets` 每次变更重建列表会失焦，沿用既有模式，勿改。
5. **表单 reset 不清 readOnly**：`resetForm` 里必须靠 `renderLots()`→`syncLotsToEntry()` 重置 readOnly，不能依赖 form.reset()。
6. **工作区曾混入未提交的历史改动**（margin-mode 按钮组、param-group 表单分组样式），提交前先 `git status` 区分，勿误提交无关文件。
7. 数据加载兼容：旧数据无 `eval`/`lots`/`targets` 字段，所有读取需 `?.`/判空兜底。

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
