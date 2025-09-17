### Glossary

| Term | Meaning |
|---|---|
| KVault | Kamino vault program that invests a single-asset pool into KLend reserves and issues shares. |
| VaultState | Zero-copy account storing admin keys, token/shares mints, accounting, fee accruals, and allocation strategy. |
| VaultAllocation | Per-reserve allocation entry: weight, token cap, cToken allocation, target token amount, last invest slot. |
| Shares | SPL token representing pro-rata claim on vault AUM minus pending fees. |
| AUM | Assets Under Management. For KVault: token_available + invested_total − pending_fees. |
| Crank funds | Small buffer accumulated per-reserve to compensate rounding during invest/withdraw. |
| KLend | Kamino lending program implementing reserves, obligations, lending market, and risk controls. |
| Reserve | Lending reserve account holding liquidity/collateral state, fees, price, and configuration. |
| Collateral (cTokens) | Token minted when depositing liquidity into a reserve; redeemable for underlying liquidity via exchange rate. |
| CollateralExchangeRate | Mapping between liquidity and collateral; supports floor/ceil/fraction conversions. |
| Obligation | Per-user position tracking deposited collaterals, borrows, LTVs, and optional orders for deleveraging. |
| LTV | Loan-to-Value ratio; borrow-factor-adjusted debt value divided by deposited value. |
| Allowed/Unhealthy Borrow Value | Thresholds determining max borrow and liquidation condition per obligation. |
| Liquidation Bonus | Discount awarded to liquidator; function of unhealthy factor, reserve caps, and emode caps. |
| Flash Loan | Flash borrow/repay within same transaction; enforced by instruction-couple checks and CPI prohibitions. |
| Referral Fees | Portion of variable interest allocated to referrers; tracked per reserve and token state. |
| Borrow Rate Curve | Piecewise-linear mapping from utilization to borrow rate. |
| Utilization | Borrowed / Total Supply of a reserve. |
| Scope | Oracle aggregator copying and validating prices from multiple sources into unified feeds. |
| Confidence interval | Tolerance check ensuring deviation is below a fraction of price; factor or bps form. |
| KFarms | Staking and rewards program with warmup/cooldown, locking modes, penalties, and delegated staking. |
| RewardPerShare | Accumulated rewards per stake unit; drives user reward tally updates. |
| Warmup/Cooldown | Delayed activation of stake and delayed withdrawal of unstaked amounts, respectively. |
| Locking Mode | None/WithExpiry/Continuous; controls penalty assessment on early withdrawals. |
| PDA | Program Derived Address; deterministic addresses controlled by program signer seeds. |
| LMA | Lending Market Authority PDA for KLend. |
| Base Vault Authority | KVault PDA used to sign CPIs into KLend. |
| TWAP/EMA | Time-weighted or exponential moving average used for smoothing oracle prices. |
| Elevation Group | Risk grouping in KLend that changes borrow factors and liquidation logic. |
| Emergency Mode | Configuration gating for KLend to disable operations in emergencies. |
| Withdrawal Caps | Per-interval caps for deposits/borrows withdrawals tracked with timestamps. |
| Token-2022 Extensions | SPL token features beyond classic SPL; only a strict subset is allowed in Kamino flows. |
| DatedPrice | Scope price with value, exponent, and unix_timestamp used by consumers. |
| Program Data | Upgradeable program’s `program_data` account holding metadata and upgrade authority. |
