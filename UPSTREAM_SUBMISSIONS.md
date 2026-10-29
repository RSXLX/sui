# Upstream submission drafts

These texts are ready to paste into MystenLabs/sui. Nothing in this file has been opened there. `Fixes #<issue>` is a placeholder: replace `<issue>` with the GitHub issue number after the issue is filed, then open the matching pull request.

Findings 1 and 2 both edit `format_balance`. They are separate branches off `main` because one changes overflow handling and the other changes the threshold comparison. They do not depend on each other. Merging both will conflict in `crates/sui/src/client_commands.rs` and needs a trivial resolution.

Fork pull requests, for reference only:

| Finding | Fork PR | Branch |
| --- | --- | --- |
| 1 | https://github.com/RSXLX/sui/pull/4 | `cursor/format-balance-high-decimals-5fcd` |
| 2 | https://github.com/RSXLX/sui/pull/5 | `cursor/format-balance-unit-threshold-5fcd` |
| 3 | https://github.com/RSXLX/sui/pull/6 | `cursor/multiaddr-reject-non-http-5fcd` |
| 4 | https://github.com/RSXLX/sui/pull/9 | `cursor/move-test-gas-units-5fcd` |
| 5 | https://github.com/RSXLX/sui/pull/7 | `cursor/coverage-native-nan-5fcd` |
| 6 | https://github.com/RSXLX/sui/pull/8 | `cursor/faucet-localhost-5fcd` |

## 1. CLI balance formatting panics when coin decimals are 30 or more

### Issue title

`sui client balance` panics when a coin has 30 or more decimals

### Issue body

`format_balance` in `crates/sui/src/client_commands.rs` unconditionally computes `10u128.pow(coin_decimals + 9)`. `u128::pow` panics when the result does not fit. `10^39` is the first power of ten that overflows `u128`, so `decimals >= 30` always panics. `coin::create_currency` and `coin_registry::new_currency` accept any `u8` decimals, and `sui client balance` / `sui client gas` pass that value through.

### Repro

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p sui --lib format_balance_does_not_panic_for_high_decimals -- --nocapture
```

Or call `format_balance(0, 30, 2, Some("ODD"))`.

### Expected

A formatted string, or a raw amount if the scale does not fit. For 0 units of a 30-decimal coin with symbol `ODD`, `0.00 ODD`. For a 39-decimal coin, the raw amount `0 ODD`, because `10^39` itself does not fit in `u128`.

### Actual

`attempt to exponentiate with overflow`. The panic happens even for a zero balance, because the powers are computed before the value is inspected.

### PR title

Keep CLI balance formatting from panicking on large coin decimals

### PR body

Fixes #<issue>

`10^(decimals + 9)` does not fit in `u128` when `decimals >= 30`, and `sui client balance` panics. Skip a K/M/B tier whose scale overflows. If `10^decimals` itself overflows, print the raw amount.

## 2. Exact thousand-coin balances stay on the lower display unit

### Issue title

Exact 1000 SUI displays as `1000.00 SUI` instead of `1.00K SUI`

### Issue body

`format_balance` switches to the K, M, or B suffix only when the raw amount is strictly greater than the threshold. For SUI (`decimals = 9`), 1000 whole SUI is exactly `10^(decimals + 3)` and fails that comparison, so it renders as `1000.00 SUI`. One extra mist renders as `1.00K SUI`. The same boundary exists at 1 million (`1000.00K` instead of `1.00M`) and 1 billion (`1000.00M` instead of `1.00B`).

### Repro

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p sui --lib format_balance_uses_higher_unit_at_exact_threshold -- --nocapture
```

### Expected

`1.00K SUI`, `1.00M SUI`, and `1.00B SUI`.

### Actual

`1000.00 SUI` for exactly 1000 SUI. The test fails on the first assertion; the M and B cases use the same `>` comparison.

### PR title

Use the higher unit at an exact thousand-coin balance

### PR body

Fixes #<issue>

Balance display used `>` for the K/M/B thresholds, so an amount exactly equal to 1000, 1_000_000, or 1_000_000_000 whole coins stayed on the lower unit. Compare with `>=`.

## 3. gRPC multiaddr parser accepts `/ws` as plaintext HTTP

### Issue title

gRPC multiaddr parser treats `/ws` and other non-HTTP protocols as HTTP

### Issue body

`parse_http_https` in `crates/mysten-network/src/multiaddr.rs` returns `Ok("http")` for every protocol other than `https`, and it consumes that protocol. `parse_end` then sees an empty remainder and accepts the address. `endpoint_from_multiaddr` therefore dials plaintext HTTP.

`/ip4/127.0.0.1/tcp/80/ws` becomes `http://127.0.0.1:80`. `/dns/example.com/tcp/443/ws` becomes `http://example.com:443`. Omitting the scheme (`/ip4/127.0.0.1/tcp/80`) should still default to HTTP.

### Repro

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p mysten-network --lib rejects_non_http -- --nocapture
```

### Expected

`parse_ip4` and `parse_dns` return an error for a `/ws` suffix. An address with no scheme still parses as HTTP.

### Actual

`Ok((127.0.0.1:80, "http"))` and `Ok(("example.com", 443, "http"))`.

### PR title

Reject non-HTTP protocols when parsing gRPC multiaddrs

### PR body

Fixes #<issue>

`parse_http_https` treated every non-`https` protocol as HTTP and consumed it. A missing scheme still defaults to HTTP. Any other protocol, including `/ws`, is now an error.

## 4. `sui move test` gas statistic mixes MIST and gas units

### Issue title

`sui move test` reports almost the entire gas budget as gas used before any bytecode runs

### Issue body

`SuiVMTestSetup::used_gas` in `crates/sui-move/src/unit_test.rs` subtracts `GasStatus::remaining_gas()` from `execution_bound`. `execution_bound` is the MIST budget passed to `SuiGasStatus::new`. `remaining_gas()` is in gas units (`budget / gas_price`). The Move unit-test gas price is 500, so an unused meter reports `budget - budget / 500`.

The "Gas Used" column and the statistics CSV both use this value. Pass/fail is unchanged. This is not the dry-run `gasUsed` versus `gasBudget` behavior tracked separately.

### Repro

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p sui-move --lib fresh_meter_reports_zero_gas_used -- --nocapture
```

The test builds a meter with a `1_000_000_000` MIST budget and does not execute bytecode.

### Expected

`0`.

### Actual

`998000000`, which is `1_000_000_000 - 1_000_000_000 / 500`.

### PR title

Report Move unit-test gas from the meter

### PR body

Fixes #<issue>

`used_gas` subtracted remaining gas units from a MIST budget, so an unused meter looked almost fully spent. Report `gas_used_pre_gas_price()` instead.

## 5. Move coverage prints NaN for native functions

### Issue title

`sui move coverage summary --summarize-functions` prints `NaN` for native functions

### Issue body

`FunctionSummary::percent_coverage` divides `covered as f64` by `total as f64`. Native functions have no bytecode, so both values are 0 and the result is `NaN`. `summarize_human` prints that percentage for every function when `--summarize-functions` is set. The CSV summary already skips native functions. The package footer in `format_human_summary` has the same `0/0` when a package has no bytecode instructions.

### Repro

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p move-coverage --lib native_function_summary_does_not_report_nan -- --nocapture
```

Or run `sui move coverage summary --summarize-functions` on a package that contains a `native fun`.

### Expected

Native functions are omitted, or shown as `n/a`. The text does not contain `NaN`.

### Actual

```text
	native fun hash
		total: 0
		covered: 0
		% coverage: NaN
```

### PR title

Skip native functions in Move coverage percentages

### PR body

Fixes #<issue>

Native functions have no bytecode, so coverage divided by zero and printed `NaN`. Omit them from the human summary, matching CSV, and do not divide when the total instruction count is zero.

## 6. `sui client faucet` rejects a `localhost` RPC URL

### Issue title

`sui client faucet` does not recognize `http://localhost:9000` as localnet

### Issue body

`find_faucet_url` in `crates/sui/src/client_commands.rs` chooses the faucet from the active RPC host. Localnet is recognized only when the host equals `127.0.0.1` (`SUI_LOCAL_NETWORK_URL`) or `0.0.0.0` (`SUI_LOCAL_NETWORK_URL_0`). The hostname `localhost` matches neither, so the command bails out and asks for an explicit faucet URL.

### Repro

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p sui --lib find_faucet_url_accepts_localhost_hostname -- --nocapture
```

Or set the active RPC to `http://localhost:9000` and run `sui client faucet` without `--url`.

### Expected

`http://127.0.0.1:9123/v2/gas`, the same URL used for `http://127.0.0.1:9000`.

### Actual

`Cannot recognize the active network. Please provide the gas faucet full URL.`

### PR title

Treat localhost as localnet when choosing the faucet

### PR body

Fixes #<issue>

`find_faucet_url` compared the RPC host only with `127.0.0.1` and `0.0.0.0`. Accept the hostname `localhost` as localnet and return `http://127.0.0.1:9123/v2/gas`.
