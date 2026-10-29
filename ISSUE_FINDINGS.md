# 审计发现

基线：本 fork 的 `main` 与 `MystenLabs/sui` 的 `main` 指向同一提交 `d16b6a12e734629d1dc523d758a4f2a73c30b8c8`（`grpc: opt-in address-alias checking for VerifySignature (#28471)`）。下面 5 条都在这个提交上用失败测试复现过。

这些测试断言的是正确行为，因此在当前代码上会失败。它们是复现，不是修复。

没有把共识关键路径上的安全漏洞写进这份公开草稿。这次审计没有发现需要单独走 Sui bug bounty / security policy 的问题。第 3 条是运维配置被错误解析，操作者自己写了非 HTTP 的 multiaddr，客户端却当成明文 HTTP；它不是远程利用，也不是共识问题。

重复检索范围：`gh search issues` / `gh search prs`，仓库 `MystenLabs/sui`。检索词包括 `format_balance`、`Gas Used`、`move unit test`、`multiaddr`、`parse_http_https`、`percent coverage`、`NaN`。没有找到描述同一缺陷的 open issue 或已合并修复。邻近但不同的条目写在每条的检索结果里。

## 1. `sui client` 在币精度大于等于 30 时 panic

- 严重程度：中
- 置信度：高
- 位置：`crates/sui/src/client_commands.rs` 第 3385–3389 行，`format_balance`

### 描述

`sui client balance` 和 `sui client gas` 用 `format_balance` 把原始余额格式化成带小数的字符串。函数无条件计算：

```rust
let billions = 10u128.pow(coin_decimals + 9);
```

`u128::pow` 在结果放不进 `u128` 时会 panic。`10^38` 仍能放下，`10^39` 不行，所以 `decimals >= 30` 一定会溢出。`coin::create_currency` 和 `coin_registry::new_currency` 都接受任意 `u8` 精度，链上可以存在这种币。查看余额时，精度来自币的 metadata（`pretty_print_balance`，约第 3278 行）。

### 复现

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p sui --lib format_balance_does_not_panic_for_high_decimals -- --nocapture
```

测试在 `crates/sui/src/client_commands.rs` 的 `format_balance_does_not_panic_for_high_decimals`。

### 实际与期望

实际：`attempt to exponentiate with overflow`，测试失败，信息是 `formatting a 30-decimal coin panicked`。

期望：返回可读的余额字符串，或者返回错误。不能因为某个币的 `decimals` 合法但偏大就让 CLI 进程崩溃。零余额也会触发，因为幂是在分支之前算的。

### 建议修复

用 `checked_pow`。溢出时退回原始整数加符号，不要再套 K/M/B。不要在进入分支之前就把四个阈值都算出来。

### 重复检索

`format_balance` 只命中已合并的实现 PR [#16477](https://github.com/MystenLabs/sui/pull/16477) 和 [#16686](https://github.com/MystenLabs/sui/pull/16686)，没有报告这个 panic。精度相关 issue（例如 #6379、#5893）讲的是创建币或钱包小数点，不是这次崩溃。

## 2. 余额恰好为 1000 / 100 万 / 10 亿个整币时，单位少进一档

- 严重程度：低
- 置信度：高
- 位置：`crates/sui/src/client_commands.rs` 第 3391–3405 行，`format_balance`

### 描述

单位切换用的是严格大于：

```rust
if value > billions { ... }
else if value > millions { ... }
else if value > thousands { ... }
```

SUI 的 `decimals` 是 9。1000 SUI 正好等于 `10^(9+3)` mist，比较失败，于是显示成 `1000.00 SUI`。再多 1 mist 就会跨过阈值，显示成 `1.00K SUI`。100 万和 10 亿整币同样落在 `1000.00K` / `1000.00M`，而不是 `1.00M` / `1.00B`。

### 复现

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p sui --lib format_balance_uses_higher_unit_at_exact_threshold -- --nocapture
```

### 实际与期望

实际：`left: "1000.00 SUI"`，`right: "1.00K SUI"`。断言在第一档就失败，M 和 B 走的是同一组 `>`。

期望：正好等于阈值时用更高一级单位，即 `1.00K SUI`、`1.00M SUI`、`1.00B SUI`。

### 建议修复

三处比较改成 `>=`。

### 重复检索

与第 1 条相同，没有 issue 描述这个边界。

## 3. gRPC multiaddr 把 `/ws` 等非 HTTP 协议当成明文 HTTP

- 严重程度：中
- 置信度：高
- 位置：`crates/mysten-network/src/multiaddr.rs` 第 326–334 行，`parse_http_https`；调用方是同文件的 `parse_ip4` / `parse_ip6` / `parse_dns`，以及 `crates/mysten-network/src/client.rs` 的 `endpoint_from_multiaddr`

### 描述

`parse_http_https` 只在协议是 `http` 或 `https` 时返回对应 scheme。其余情况，包括后面还有 `/ws` 这种协议，以及迭代器已经结束，都返回 `Ok("http")`，并且把那个协议消费掉。随后 `parse_end` 看到后面没有剩余组件，就认为地址合法。

因此 `/ip4/127.0.0.1/tcp/80/ws` 会变成 `http://127.0.0.1:80`，`/dns/example.com/tcp/443/ws` 会变成 `http://example.com:443`。省略 scheme 时默认 HTTP 是合理的；把一个明确写出来的其他协议当成 HTTP 不是。

这是配置解析错误。操作者如果写成 `/ws` 或别的非 HTTP 后缀，连接会走明文 HTTP，而不是被拒绝。它不构成远程攻击面，不需要私下报 bounty。

### 复现

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p mysten-network --lib rejects_non_http -- --nocapture
```

测试：`parse_ip4_rejects_non_http_protocol`、`parse_dns_rejects_non_http_protocol`。

### 实际与期望

实际：

- `Ok((127.0.0.1:80, "http"))`
- `Ok(("example.com", 443, "http"))`

期望：`parse_ip4` / `parse_dns` 返回错误，指出后缀不是 `http` 或 `https`。没有 scheme 的 `/ip4/127.0.0.1/tcp/80` 可以继续默认 HTTP。

### 建议修复

`None` 保持 `Ok("http")`。任何其他 `Protocol` 返回 `Err`，不要消费后假装它是 HTTP。

### 重复检索

`multiaddr`、`parse_http_https` 没有对应 issue 或 PR。#20971（`network-address ignored`）是另一个配置项。

## 4. `sui move test` 的 “Gas Used” 把 MIST 预算和 gas 单位相减

- 严重程度：低
- 置信度：高
- 位置：`crates/sui-move/src/unit_test.rs` 第 196–201 行，`SuiVMTestSetup::used_gas`

### 描述

统计表和 CSV 的 `gas` 列来自 `used_gas`：

```rust
Gas::new(execution_bound)
    .checked_sub(gas_status.remaining_gas())
    .unwrap()
```

`execution_bound` 是传给 `SuiGasStatus::new` 的 gas budget，单位是 MIST。`GasStatus::remaining_gas` 返回的是 gas 单位：`budget / gas_price`（内部单位再除以 1000）。测试里 `TEST_GAS_PRICE` 是 500。什么都没执行时，报告值是 `budget - budget / 500`，而不是 0。

Move 自带的 `DefaultVMTestSetup` 用同一公式，但它的预算本身就是 gas 单位，所以那里减出来是对的。Sui 把 MIST 预算塞进了这个公式。

这只影响 `sui move test` 打印的 “Gas Used” 和 `--statistics` CSV，不改变测试通过与否。默认预算是 `max_tx_gas`（当前最大版本为 `50_000_000_000_000`），同一公式会让每个测试都显示大约 `4.99e13`，真实消耗只在个位量级上扰动这个数。

不要和 open issue #28448 混在一起。那条说的是 dry-run 的 `gasUsed` 被拿去当交易 `gasBudget` 后链上 `InsufficientGas`，路径不同。

### 复现

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p sui-move --lib fresh_meter_reports_zero_gas_used -- --nocapture
```

### 实际与期望

预算 `1_000_000_000` MIST（1 SUI），meter 刚创建、没有执行指令。

实际：`left: 998000000`，`right: 0`。`1_000_000_000 - 1_000_000_000/500 = 998_000_000`。

期望：0。

### 建议修复

用 `GasStatus::gas_used_pre_gas_price()`。如果这一列要显示 MIST，再乘 `gas_price`。不要用 MIST 预算减去剩余 gas 单位。

### 重复检索

`sui move test` 的 #22640、#22309 是别的失败（VM invariant、测试过滤器），都已关闭。#28448 仍开放，但是交易 gas 估算，不是这条统计。

## 5. `sui move coverage summary --summarize-functions` 对 native 函数打印 NaN

- 严重程度：低
- 置信度：高
- 位置：`external-crates/move/crates/move-coverage/src/summary.rs` 第 95–108 行和第 161 行，`ModuleSummary::summarize_human` 与 `FunctionSummary::percent_coverage`

### 描述

`--summarize-functions` 对模块里的每个函数都调用 `percent_coverage`：

```rust
(self.covered as f64) / (self.total as f64) * 100f64
```

native 函数没有字节码，`total` 和 `covered` 都是 0，结果是 `NaN`。同一函数在 CSV 路径里被 `!summary.fn_is_native` 滤掉了，人类可读路径没有这个过滤。模块合计在全是 0 时会打印 “No source code to compute coverage of in module”，但函数行仍然是 `% coverage: NaN`。

`sui move coverage summary --summarize-functions` 会走到这里（`external-crates/move/crates/move-cli/src/base/coverage.rs`）。任何带 `native fun` 的包，例如 framework 或 stdlib，都会打出 NaN。

同一文件的包级汇总 `format_human_summary`（`move-coverage/src/lib.rs` 约第 41–44 行）在指令总数为 0 时也会做 `0/0`，得到 `% Move Coverage: NaN`。这次的失败测试覆盖的是函数行。

### 复现

```bash
SUI_SKIP_SIMTESTS=1 cargo test -p move-coverage --lib native_function_summary_does_not_report_nan -- --nocapture
```

### 实际与期望

实际输出包含：

```text
	native fun hash
		total: 0
		covered: 0
		% coverage: NaN
>>> No source code to compute coverage of in module
```

期望：native 函数不出现在百分比里，或者打印 `n/a`。不能是 `NaN`。

### 建议修复

`total == 0` 时不要做除法。人类可读摘要与 CSV 一样跳过 native 函数，或者明确写成 `n/a`。包级汇总在 `total_instructions == 0` 时同样不要做除法。

### 重复检索

`percent coverage` 和 `NaN` 在 `MystenLabs/sui` 的 issue 里没有命中。
