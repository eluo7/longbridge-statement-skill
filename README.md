# longbridge-statement-skill

*English | [中文](README.zh-CN.md)*

A Claude/agent **skill** + standalone Python tool that pulls **Longbridge (長橋證券)**
account statements via the **official Longbridge CLI** and builds tax-ready CSVs — for
**individual income tax on foreign income (个税 境外所得)**, CRS reporting, and broker P&L
reconciliation.

Companion to **[futu-statement-skill](https://github.com/BH-M87/futu-statement-skill)** (the
other broker). Combine both brokers' realized P&L for the final capital-gains figure.

> This repository is a fork of [BH-M87/longbridge-statement-skill](https://github.com/BH-M87/longbridge-statement-skill)
> (MIT), maintained by [eluo7](https://github.com/eluo7). It adds distribution characterisation
> (dividend / capital gain / return of capital), a manual-adjustment ledger and trade-date
> attribution.

## Why

Like most brokers, Longbridge statements have:

- **no realized-P&L section** — reconstructed here (average-cost) from `stock_trades`, seeded
  with the prior-December closing holdings so carried-in positions are costed correctly;
- **dividends inside `corps`** (corporate actions), not a "dividend" line;
- **interest in `interests`** — a financing cost you paid, not income;
- **every distribution labelled "dividend"** — but a year-end capital-gain distribution and a
  return of capital are not dividends, and the broker's 摊薄 `cost_price` cannot tell you which.
  Supply the character with `--distributions`; see [Distributions](#distributions-dividend-vs-capital-gain-vs-return-of-capital).

Method is documented in [`SKILL.md`](SKILL.md); the normative event/tax model and every data
contract are in [`docs/tax-treatment.md`](docs/tax-treatment.md), with generic worked examples
in [`docs/examples.md`](docs/examples.md).

## Prerequisite: official Longbridge CLI

This tool **shells out to `longbridge`** — it never handles your credentials. **If the CLI
isn't installed, the script offers to install it for you (after you confirm), then offers to
run `longbridge auth login`** — each step asks for y/N first (and falls back to printing the
manual steps if you decline or run it non-interactively). Manual setup:

```bash
# install (pick your platform)
brew install --cask longbridge/tap/longbridge-terminal                                   # macOS
curl -sSL https://open.longbridge.com/longbridge/longbridge-terminal/install | sh        # macOS/Linux
scoop install https://open.longbridge.com/longbridge/longbridge-terminal/longbridge.json # Windows

# authenticate (opens a browser, OAuth)
longbridge auth login

# verify it works
longbridge statement --type monthly --format json
```

If a CLI call fails because you're not logged in, the script points you to
`longbridge auth login`. Docs: https://open.longbridge.com/zh-CN/docs/cli/ . This uses the
**CLI** (not the Longbridge MCP). No third-party Python packages required. Python 3.10+.

## Usage

```bash
python3 longbridge_tax.py --year 2025
```

Output lands in `out/<year>/` by default (e.g. `out/2025/`); pass `-o DIR` to override the
base directory — the year is always auto-nested under it. The Longbridge account number is
**not** available from the API or the statements (it is known only at CLI login), so output
is not labelled by account. The CLI uses whichever account its token is bound to, so just
confirm the CLI is logged into the account you mean. If you run multiple accounts, give each
its own `-o` (e.g. `-o out_acctA`) and switch CLI logins between runs.

### FX: resolved by date, never hardcoded

There is **no built-in rate table**. With no FX flags the script fetches the official
PBOC/CFETS RMB central parity (人民币汇率中间价) for the statutory as-of date from
`www.chinamoney.com.cn`, caches it in `<outdir>/<year>/.fx_rates.json`, and writes an
audit trail to `longbridge_<YEAR>_汇率依据.csv`. When the as-of date has no publication
(weekend/holiday), the latest published day on or before it is used; both dates are reported.

The date follows 个人所得税法第十六条 / 实施条例第三十二条 / 财政部 税务总局公告
2020 年第 3 号第十二条: `--fx-basis year-end` (default) uses December 31 of `--year`
for 年度汇算清缴; `--fx-basis prior-month-end` uses the last day of the month before
`--filing-date` for 按次/按月申报. `--fx-as-of YYYY-MM-DD` overrides the date.

If the feed is unavailable, use `--fx-rates-file rates.csv`
(columns `date,currency,rate,source,note`) or pass `--fx-rate CCY=RATE` per currency
(`--rate` remains the HKD shorthand). Explicit rates short-circuit the network; currencies
you do not list are left unconverted with a warning. `--no-fx` produces an
original-currency-only report. With no authoritative rate available the run **fails
closed** instead of guessing. `--refresh` re-fetches both statements and rates. Confirm
which statutory basis applies to your filing with a tax adviser.

### Outputs (UTF-8-BOM, Excel-friendly)

| File | Contents |
|---|---|
| `longbridge_<YEAR>_成交明细.csv` | stock / option / fund trades; `清算金额` already net of fees |
| `longbridge_<YEAR>_股息利息现金流.csv` | dividends (corps), financing interest, withdrawals |
| `longbridge_<YEAR>_已实现盈亏_按标的.csv` | realized P&L per instrument (average-cost) |
| `longbridge_<YEAR>_账户净值.csv` | per-month asset total (cross-check) |
| `longbridge_<YEAR>_税务汇总.csv` | tax summary by currency — capital gains / dividends / capital-gain distributions / ROC memo / interest + tax due |
| `longbridge_<YEAR>_汇率依据.csv` | every FX rate used, with statutory as-of date, publication date, basis and source |
| `longbridge_<YEAR>_分配明细.csv` | every distribution with its character (dividend / capital gain / ROC) and source |
| `longbridge_<YEAR>_次年成本加回.csv` | capital-gain distributions only: input for next year's `--opening-adjustments` |

The script prints realized total, dividends, distribution characters and interest so you can
sanity-check.

### Distributions: dividend vs capital gain vs return of capital

Longbridge reports every cash distribution as a "dividend", and its holding `cost_price` is
**摊薄成本** — it drops on *any* distribution, so a falling cost is not evidence of return of
capital. The three characters are taxed differently:

| character | tax | cost basis |
|---|---|---|
| `dividend` | 利息股息红利 20 %, year received | unchanged |
| `capital_gain` | 财产转让 20 %, year received | **unchanged** |
| `roc` | not income | **reduced** (deferred into the sale) |

```bash
# 1) list every distribution with a blank character for review
python3 longbridge_tax.py --year 2025 --write-distribution-template
# 2) fill distributions_template.csv's `character` column (from the fund's tax supplement
#    or 1042-S — never from cost_price), then:
python3 longbridge_tax.py --year 2025 --distributions distributions_template.csv
# strict mode refuses to guess instead of defaulting unknowns to dividends:
python3 longbridge_tax.py --year 2025 --distributions d.csv --on-unclassified-distribution error
# long-tail corporate actions (splits, non-cash distributions, reclassifications):
python3 longbridge_tax.py --year 2025 --adjustments adjustments.csv
```

Matching is by `(code, date)`, then `(code, amount)`, then a wildcard only when the ticker
distributes once; ambiguous rules raise. For a capital-gain distribution the tool writes
`次年成本加回.csv`: if the broker reduced cost for it, run next year with
`--opening-adjustments <file>` to undo that.

### Data-quality guards

- **Missing months stop the run** by default (a silent skip understates tax). Override with
  `--allow-missing-months` only after checking the gap.
- **Statement key fallback**: the listed file_key sometimes has the wrong CDN prefix
  (`/statement_data/…` vs `/lb-statement/…`); the export retries the other prefix.
- **Trade-date attribution**: reads `<year+1>01` as well (30/31 Dec trades settle next month)
  and drops rows whose trade date is outside the year, so the prior December isn't counted
  twice.

## How realized P&L is computed

Average-cost per instrument and currency. `clear_amount` is signed and already net of fees;
realized P&L accrues only when a trade *reduces* the open position (selling a long, or buying
back a short), so whatever position is still open at year-end — long **or** short — contributes
0 (unrealized, not taxable). The prior-December `equity_holdings.cost_price` seeds carried-in
positions. A position going negative (a cash account can't truly short — this usually means
missing cost basis, e.g. shares transferred in or a sell recorded before its buy) is flagged
for review rather than booking the full sale proceeds as profit. Money-market funds are shown
but excluded from the realized headline.
**Capital-gains tax must be combined across brokers** — the standalone figure here nets
against e.g. Futu within `财产转让所得`. Details in [`SKILL.md`](SKILL.md).

## Use as a Claude Code skill

Drop this folder into your skills directory (e.g. `~/.claude/skills/longbridge-statement-tax/`)
or install via your plugin manager. Claude loads [`SKILL.md`](SKILL.md) when you ask it to
compute Longbridge P&L / dividends / tax.

## Privacy

This repo contains **no personal data**. It pulls live from your authenticated Longbridge
CLI and writes CSVs locally. `.gitignore` blocks `*.csv`/`*.json` so you can't accidentally
commit results.

## License

MIT
