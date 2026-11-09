# longbridge-statement-skill

*[English](README.md) | 中文*

一个 Claude/agent **skill** + 独立 Python 工具，通过**长桥官方 CLI** 拉取**长桥证券（Longbridge）**
账户结单，生成可直接报税的 CSV —— 面向**个人所得税·境外所得（个税 境外所得）**、CRS 申报、券商盈亏对账。

是 **[futu-statement-skill](https://github.com/BH-M87/futu-statement-skill)** 的姊妹篇（另一家券商）。
两家券商的已实现盈亏合并后，才是最终的财产转让所得。

> 本仓库 fork 自 [BH-M87/longbridge-statement-skill](https://github.com/BH-M87/longbridge-statement-skill)
> （MIT），由 [eluo7](https://github.com/eluo7) 维护。新增了分配定性（股息 / 资本利得 / 资本返还）、
> 手工调整台账与按成交日归属。

## 为什么需要它

和多数券商一样，长桥结单：

- **没有「已实现盈亏」栏** —— 本工具用移动加权平均从 `stock_trades` 重建，并以**上年 12 月期末持仓**
  （`equity_holdings.cost_price`）做开仓成本，跨年持仓才不会算错；
- **分红在 `corps`（公司行动）里**，不是单独的「派息」栏；
- **利息在 `interests`** —— 是你付出的融资成本，不是收入。

方法见 [`SKILL.md`](SKILL.md)。

## 前置：长桥官方 CLI

本工具**只调用 `longbridge` 命令**，不碰你的任何凭证。**如果没装 CLI，脚本会询问是否帮你自动安装
（需你确认 y/N），装好后再询问是否帮你登录（`longbridge auth login`）**；你拒绝或非交互运行时则回退到
打印手动步骤。手动配置：

```bash
# 安装（按平台任选其一）
brew install --cask longbridge/tap/longbridge-terminal                                   # macOS
curl -sSL https://open.longbridge.com/longbridge/longbridge-terminal/install | sh        # macOS/Linux
scoop install https://open.longbridge.com/longbridge/longbridge-terminal/longbridge.json # Windows

# 登录认证（会打开浏览器走 OAuth）
longbridge auth login

# 验证能跑
longbridge statement --type monthly --format json
```

如果调用因未登录失败，脚本会提示你运行 `longbridge auth login`。文档：
https://open.longbridge.com/zh-CN/docs/cli/ 。本工具用的是 **CLI**（不是长桥 MCP）。
无需第三方 Python 包，Python 3.10+。

## 用法

```bash
python3 longbridge_tax.py --year 2025
```

输出默认落在 `out/<年份>/`（例如 `out/2025/`）；传 `-o 目录` 可覆盖基础目录，年份始终自动
嵌套在其下。**长桥的账号在 API 和对账单里都拿不到，只有在 CLI 登录时才知道**，所以脚本不会
按账号给输出命名，也**绝不要去猜账号**——猜错会把别人的报税数据贴错标签。CLI 也没有账户选择
参数，用的是 token 绑定的那个账户，所以只要确认 CLI 当前登录的就是你要的账户即可。只有当你
确实同时管理**多个**账户时，才需要分开：给每个账户各传一个 `-o`（例如 `-o out_acctA`），并在
两次运行之间切换 CLI 登录。

### 汇率：按法定时点动态取，不硬编码

**代码里没有任何内置汇率表。** 不传汇率参数时，脚本按法定时点从中国货币网
（`www.chinamoney.com.cn`）抓取人民银行授权公布的**人民币汇率中间价**，缓存在
`<输出目录>/<年份>/.fx_rates.json`，并把每个币种的汇率、发布日、口径、来源写进
`longbridge_<年>_汇率依据.csv`，便于报税留痕与复算。基准日若遇周末/节假日无发布，
取该日之前最近一个发布日，并同时记录基准日与实际采用日。

依据《个人所得税法》第十六条、《实施条例》第三十二条、财政部 税务总局公告
2020 年第 3 号第十二条：

| `--fx-basis` | 基准日 | 适用 |
|---|---|---|
| `year-end`（默认） | `--year` 当年 12-31（上一纳税年度最后一日） | 年度汇算清缴补缴税款 |
| `prior-month-end` | 办理申报的上一月最后一日，由 `--filing-date` 推算（默认今天） | 按次 / 按月申报 |

`--fx-as-of YYYY-MM-DD` 可直接指定日期。取不到官方数据时：
`--fx-rates-file rates.csv` 提供离线凭证（列 `date,currency,rate,source,note`），
或 `--fx-rate USD=7.10 --fx-rate HKD=0.90` 逐币种显式指定示例值（实际请填官方中间价；
`--rate` 是 HKD 简写）。
**显式汇率会跳过联网**，未列出的币种不折算并给出警告；`--no-fx` 只出原币报表。
取不到任何可信汇率时脚本**直接报错退出**，绝不猜一个数——汇率错了会静默算错整张申报表。

`--refresh` 同时强制重新抓取结单与汇率。脚本先按货币分桶，只有拿到对应汇率的币种
才填写人民币和应纳税额列。具体适用哪种口径，请与税务师确认。

### 输出（UTF-8-BOM，Excel 直接打开不乱码）

| 文件 | 内容 |
|---|---|
| `longbridge_<年>_成交明细.csv` | 股票/期权/基金成交；`清算金额` 已扣手续费 |
| `longbridge_<年>_股息利息现金流.csv` | 分红（corps）、融资利息、出入金 |
| `longbridge_<年>_已实现盈亏_按标的.csv` | 按标的已实现盈亏（移动加权平均） |
| `longbridge_<年>_账户净值.csv` | 每月资产总值（交叉校验） |
| `longbridge_<年>_税务汇总.csv` | 按货币分组的税务汇总：财产转让/分红/资本利得分配/资本返还备查/利息 + 应纳税额 |
| `longbridge_<年>_汇率依据.csv` | 本次折算所用的每个汇率：基准日、实际发布日、法定口径、数据来源 |
| `longbridge_<年>_分配明细.csv` | 每笔分配的性质（股息/资本利得/资本返还）、来源与应用方式 |
| `longbridge_<年>_次年成本加回.csv` | 仅资本利得分配：若券商已冲减成本，下一年用 `--opening-adjustments` 加回 |

脚本会打印已实现盈亏合计、分红、分配定性、利息，方便核对。

### 分配定性：股息 / 资本利得 / 资本返还

长桥把所有现金分配都叫「股息」，持仓的 `cost_price` 又是**摊薄成本**——任何分配都会让它下降，
所以「成本降了」**不能**证明是资本返还。三者的税务处理不同：

| 性质 | 税 | 成本基础 |
|---|---|---|
| `dividend` 股息 | 利息股息红利 20%，收到当年 | 不变 |
| `capital_gain` 资本利得 | 财产转让 20%，收到当年 | **不变** |
| `roc` 资本返还 | 不是收入 | **冲减**，递延到卖出 |

```bash
# 1) 先导出所有分配，character 留空待确认
python3 longbridge_tax.py --year 2025 --write-distribution-template
# 2) 按基金 tax supplement / 1042-S 填好 character（别用 cost_price 推），再运行
python3 longbridge_tax.py --year 2025 --distributions distributions_template.csv
# 严格模式：未定性的分配直接报错，而不是默认按股息
python3 longbridge_tax.py --year 2025 --distributions d.csv --on-unclassified-distribution error
```

匹配顺序：`(代码, 日期)` → `(代码, 金额)` → 仅当该标的当年只分配一次时的通配规则；有歧义直接报错。

### 手工调整：覆盖事件模型之外的情况

产品无限、应税事件有限。拆股、非现金分配、合并对价、券商重分类这类事件，不需要为每个产品
加分支，只要归入四类效果之一：

```csv
date,code,type,amount,currency,character,note
2025-05-01,ABC,quantity,2,USD,,2-for-1 split
2025-06-01,ABC,cost,120,USD,,non-cash distribution added to basis
2025-07-01,ABC,realized,-500,USD,,broker reclassification
2025-08-01,ABC,income,30,USD,dividend,special dividend
```

- `quantity`：amount=拆分比例（2 表示 2 拆 1），数量×比例、单位成本÷比例；
- `cost`：amount=总成本变动额（正负均可）；
- `realized`：直接计入财产转让所得；
- `income`：计入收入，`character` 为 `dividend`（默认）或 `capital_gain`。

规范的数据契约、分配决策树与失败模式见 [`docs/tax-treatment.md`](docs/tax-treatment.md)，
通用示例见 [`docs/examples.md`](docs/examples.md)（示例非穷举）。
资本利得分配会额外生成 `次年成本加回.csv`：如果券商冲减了成本，下一年运行时用
`--opening-adjustments` 指回该文件。

### 数据质量保护

- **缺月结单默认直接停止**（静默跳过会少报税），确认缺口后才用 `--allow-missing-months` 继续；
- **结单 key 前缀自动回退**：列表给的 `/statement_data/…` 失效时自动试 `/lb-statement/…`；
- **按成交日归属**：额外读取次年 1 月结单（12/30–31 的成交要到下月才出），并丢弃日期不在本年度的行，避免上年 12 月被重复计入。

## 已实现盈亏怎么算

按标的和货币分别做移动加权平均（长桥现金账户，主要做多）。`clear_amount` 带符号且已扣费；已实现只在**卖出**时
确认，年末仍持有的标的实现记 0（属未实现，不计税）。开仓成本用上年 12 月 `equity_holdings.cost_price`
带入。货币基金会显示但不并入已实现口径。**财产转让税须跨券商合并**——这里的单账户数要与（如）富途
在「财产转让所得」内相抵后再算。详见 [`SKILL.md`](SKILL.md)。

## 作为 Claude Code skill 使用

把本目录放进 skills 目录（如 `~/.claude/skills/longbridge-statement-tax/`）或用插件管理器安装。
当你让 Claude 算长桥盈亏/分红/税时，会自动加载 [`SKILL.md`](SKILL.md)。

## 隐私

仓库内**不含任何个人数据**。工具从你已认证的长桥 CLI 实时拉取、把 CSV 写到本地；`.gitignore`
已拦截 `*.csv`/`*.json`，避免误提交结果。

## 许可证

MIT
