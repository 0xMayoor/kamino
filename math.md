### Mathematical Analysis and Formulas

This document explains the math used across KVault, KLend, KFarms, and Scope. It details formulas, rounding, and invariants per file/function that perform numerical logic. Where handlers only marshal accounts, we reference the underlying math functions.

Note: Amounts are integers in lamports; fractional math uses `Fraction` or fixed-point decimals (`decimal_wad::Decimal`), with explicit floor/ceil/round operations to prevent drift.

## KVault Math

Files: `programs/kvault/src/state.rs`, `operations/vault_operations.rs`, `operations/klend_operations.rs`.

- Vault AUM:
  - `VaultState::compute_aum(invested_total)`:
    - AUM = token_available + invested_total − pending_fees
    - Require AUM ≥ 0; otherwise `AUMBelowPendingFees`.

- Target allocation computation:
  - `VaultState::refresh_target_allocations(invested)`:
    - Let total_tokens = compute_aum(invested.total)
    - If unallocated weight > 0, reserve `unallocated_target = total_tokens * unallocated_weight / (sum(weights)+unallocated_weight)`; cap by `unallocated_tokens_cap`. Decrease remaining tokens.
    - Iteratively distribute remaining tokens by proportional weights, capping each reserve at `token_allocation_cap`.
    - Set `token_target_allocation` per reserve (stored as Fraction bits).

- Shares issuance and redemption:
  - `get_shares_to_mint(holdings_aum, user_token_amount, shares_issued)`:
    - If `shares_issued == 0`, mint = `user_token_amount`.
    - Else require `holdings_aum > 0`. Compute:
      - shares_to_mint = floor(shares_issued * user_token_amount / ceil(holdings_aum))
  - `compute_amount_to_deposit_from_shares_to_mint(vault_total_shares, vault_total_holdings, shares_to_mint)`:
    - If zero shares, equals shares_to_mint; else `ceil(vault_total_holdings * shares_to_mint / vault_total_shares)`
  - `compute_user_total_received_on_withdraw(shares_issued, vault_total_holdings, shares_to_withdraw)`:
    - If full redemption, floor(vault_total_holdings), else floor(vault_total_holdings * shares_to_withdraw / shares_issued)
  - `calculate_shares_to_burn(amount_to_send_to_user, total_supply, total_sum, number_of_shares)`:
    - ceil(amount_to_send_to_user * total_supply / floor(total_sum)) bounded by `number_of_shares`.

- Withdraw math (`withdraw`):
  - Compute user’s entitled amount from AUM; split between available and invested (optional per-reserve path).
  - For invested leg: convert desired invested liquidity to collateral with `fraction_liquidity_to_collateral_ceil`, cap by owned cTokens, then compute expected liquidity received with `fraction_collateral_to_liquidity`; collect rounding error if fractional part lost > fractional part of payout.
  - Burn calculated shares, adjust accounting; any disinvest surplus stays in vault.

- Invest math (`invest`):
  - After fee charge and target refresh, compute difference between actual vs target token exposure for the chosen reserve. If positive, withdraw collateral; if negative, deposit liquidity up to available.
  - Convert liquidity<->collateral via reserve exchange rate; for Add: `liquidity = ceil(collateral_to_liquidity)`, withdraw from vault; for Subtract: `liquidity = floor(collateral_to_liquidity)`, deposit into vault.
  - Rounding loss compensated using `available_crank_funds` if possible.

- Fees (`charge_fees`):
  - `seconds_passed = timestamp - last_fee_charge_timestamp`.
  - Management fee: apply yearly bps pro-rata to `prev_aum`:
    - `mgmt_fee = mgmt_bps/10000 * seconds_passed / SECONDS_PER_YEAR` (as Fraction)
    - `mgmt_charge = prev_aum * mgmt_fee`
  - Performance fee: on positive earned interest `earned = new_aum - prev_aum`:
    - `perf_charge = performance_bps/10000 * earned`
  - `new_fees = min(mgmt_charge + perf_charge, new_aum)`; `pending_fees += new_fees`; `prev_aum = new_aum - new_fees`.

- CPI bridges (`operations/klend_operations.rs`): no intrinsic math beyond serialization and seed derivation; key numeric arguments are collateral/liquidity amounts passed through.

## KLend Math

Files: `state/reserve.rs`, `state/obligation.rs`, `utils/borrow_rate_curve.rs`, `state/liquidation_operations.rs`, `lending_market/flash_ixs.rs`, plus lending operations (function inventory via grep).

- Borrow rate curve (`utils/borrow_rate_curve.rs`):
  - Piecewise linear points `[(util_bps, rate_bps)]` with monotonic constraints; rate at utilization u:
    - Find segment [start,end] so `u in [u_s,u_e]`.
    - If exact endpoint, return rate_bps at point.
    - Else linear interpolation: `rate = (u - u_s) * slope_nom/slope_denom + offset`, where slope is per-bps difference; all values as `Fraction`.

- Reserve utilization (`ReserveLiquidity::utilization_rate`):
  - `util = borrowed / total_supply` with `total_supply = available + borrowed − protocol_fees − referrer_fees − pending_referrer_fees`.

- Interest accrual (`Reserve::accrue_interest` and `ReserveLiquidity::compound_interest`):
  - Borrow rate per slot from curve; approximate compounding over `slots_elapsed` using:
    - For small n: closed forms up to n=4; else truncated Taylor: `1 + base*n + base^2*n*(n-1)/2 + base^3*n*(n-1)*(n-2)/6` with `base = rate / SLOTS_PER_YEAR`.
  - New debt: `new_debt = prev_debt * compounded_interest_rate`
  - Host fixed rate separated: `fixed_host_fee = prev_debt*(compounded_fixed_rate) − prev_debt`
  - Variable protocol fee: `(new_debt − prev_debt − fixed_host_fee) * protocol_take_rate`
  - Referral: `absolute_referral_rate = protocol_take_rate * referral_rate`; `max_referrer_fees = net_new_variable_debt * absolute_referral_rate`
  - Update cumulative borrow index, pending referrer fees, accumulated protocol fees, borrowed amount.

- Collateral exchange rate (`CollateralExchangeRate`):
  - `fraction_collateral_to_liquidity = collateral * liquidity / collateral_supply` (BigFraction -> Fraction)
  - `fraction_liquidity_to_collateral = liquidity * collateral_supply / liquidity_total`
  - Integer helpers: `*_ceil` do ceil via adjusted numerator; `liquidity_to_collateral_fraction` maps u64 -> Fraction.
  - If zero supply or zero liquidity, uses `INITIAL_COLLATERAL_RATE`.

- Deposit/withdrawal on reserve (`compute_depositable_amount_and_minted_collateral`):
  - `collateral = liquidity_to_collateral(liquidity_amount)`; `liquidity_to_deposit = ceil(collateral_to_liquidity(collateral))` ≤ `liquidity_amount`.

- Borrowing (`Reserve::calculate_borrow`):
  - For `amount_to_borrow == u64::MAX`: compute max feasible borrow constrained by:
    - remaining borrow value: `max_bf_adjusted_debt_value * decimals / price / borrow_factor`
    - reserve remaining borrow and available liquidity
    - Apply fees (inclusive) to derive `receive_amount`.
  - Else: for a requested `receive_amount`, compute required `borrow_amount = receive + fees(exclusive)`; verify borrow-factor-adjusted value ≤ max.

- Repay (`Reserve::calculate_repay`):
  - If `u64::MAX`, settle all; else settle `min(requested, borrowed)`; repay_amount = ceil(settle).

- Obligation-level metrics (`obligation.rs`):
  - `loan_to_value = BF_adjusted_debt_value / deposited_value`.
  - `unhealthy_loan_to_value = unhealthy_borrow_value / deposited_value`.
  - Withdraw max value: if checking vs liquidation threshold uses `reserve_liq_threshold_pct`, else `reserve_max_ltv_pct`; compute `(highest_allowed - bf_adjusted_debt_value) * 100 / pct` with early returns for zero-
    limit or no headroom.
  - Accrue interest on per-liquidity basis with cumulative index ratio multiplication.

- Liquidations (`liquidation_operations.rs`):
  - Determine liquidation reason (LTV exceeded, auto-deleveraging, order execution), compute liquidation bonus rate:
    - Bad-debt path (no-bf LTV ≥ ~99%): bonus = max(min reserve bad-debt bonus, diff_to_bad_debt)
    - Otherwise unhealthy factor: `bonus = clamp(max(min_reserve_bonus, user_ltv - max_allowed_ltv), max_bonus)`; also bounded by diff_to_bad_debt and emode caps.
  - Max liquidatable borrowed amount: consider close factor, per-liquidity share, market cap.
  - Compute liquidation amounts:
    - `liquidation_ratio = debt_liq_amount / borrowed_amount`
    - `total_value_with_bonus = borrowed_value * liquidation_ratio * (1 + bonus)`
    - If value ≥ collateral value: settle all (full withdraw of deposited collateral on that leg); else proportionally withdraw `withdraw_pct = total_value_with_bonus / collateral_value`.
  - Protocol liquidation fee on bonus: `max(1, ceil((amount_liquidated − amount_liquidated/(1+bonus)) * protocol_fee_pct))`.

- Flash loans (`lending_market/flash_ixs.rs`):
  - Enforce same-tx borrowing/repayment pair, with matching liquidity amounts and exact accounts, and forbid CPI entry. No arithmetic fees shown in this file; fees are in `ReserveFees::calculate_flash_loan_fees`.

## KLend: lending_market/lending_operations.rs (per-handler)

- refresh_reserve: interest accrual; optional price update; saved-price age validation; last_update set.
- is_saved_price_age_valid: `now - last_price_ts < max_age_price_seconds`.
- is_price_refresh_needed: trigger threshold `= max_age * pct / 100`; compare against age.
- refresh_reserve_limit_timestamps: update deposit/borrow limit crossed timestamps.
- deposit_reserve_liquidity: amount>0; reserve not stale; compute depositable and minted collateral; enforce deposit limit; update withdrawal cap; deposit; mark stale.
- borrow_obligation_liquidity: amount>0; reserve/obligation freshness; market enabled; obligation healthy/non-obsolete; borrow limit and remaining value/capacity checks; calculate fees and borrow amount (inclusive/exclusive modes); update reserve and obligation debts; referral accrual; utilization gating; elevation trackers; invariants.
- deposit_obligation_collateral: amount>0; freshness; elevation rules; add/update collateral; elevation trackers if newly added; mark stale; invariants using exchange rate.
- withdraw_obligation_collateral: amount>0; freshness (ALL_CHECKS if borrows exist); compute allowed by max LTV/liquidation threshold; proportional amount logic; trackers on full withdraw; invariants.
- redeem_reserve_collateral: amount>0; freshness; exchange rate conversion; limit timestamp updates; optional withdrawal cap increment.
- redeem_fees: freshness; compute withdrawable protocol fees; update balances; mark stale.
- repay_obligation_liquidity: amount>0; freshness; accrue per-liquidity interest; compute settle and repay amounts; update caps; trackers; reserve repay and obligation decrease; invariants.

## KFarms Math

Files: `stake_operations.rs`, `farm_operations.rs`, `utils/math.rs`.

- Stake/amount conversions:
  - `convert_stake_to_amount = floor(stake * total_amount / total_stake)` (or ceil if `round_up`)
  - `convert_amount_to_stake = total_stake * amount / total_amount` when totals non-zero; else identity initialization.

- Pending/active stake transitions:
  - Add/remove pending stake adjusts both user and farm totals proportionally; activate pending moves pending → active 1:1.

- Unstake with penalties:
  - Early withdrawal penalty applied depending on `LockingMode` (WithExpiry/Continuous) using configured duration and timestamps; returns `(post_penalty_amount, penalty)`.

- Rewards issuance (`refresh_global_reward`):
  - For each reward index, if there is active stake:
    - `cumulative_amt = curve.get_cumulative_amount_issued_since_last_ts(last_ts, ts)`
    - If `RewardType::Proportional`, `reward_type_amt = cumulative_amt`; if `Constant`, multiply by `total_staked_amount`.
    - Adjust by `rps_decimals`: divide by `10^(rewards_per_second_decimals)`.
    - Optional oracle adjustment: if using Scope price, multiply by price.value and divide by `10^price.exp`.
    - Clamp by `rewards_available`, update `reward_per_share += issued / total_active_stake` (delegated vs non-delegated path).
  - User reward refresh: compute delta from `reward_per_share * user_stake − rewards_tally` and floor to u64.

- Farm vault proportional withdrawal:
  - Remove amounts from active/pending proportionally to requested withdraw: `removed_active = floor(active * req / total)` and similar for pending.

- Math helpers: `u64_mul_div(a,b,c)` using 128-bit intermediate; `ten_pow` table; `full_decimal_mul_div(a,b,c)` via U256.

## Scope Math

Files: `utils/math.rs` in Scope.

- Price conversions:
  - sqrt price Q64.64 to price: `price = (sqrt_price^2 >> 64)` with decimal normalization.
  - Map x64 price to `(value, exp)` by choosing exp so integer part fits within 64 bits; compute value accordingly.
  - Convert lamports price to token price adjusting for decimals difference.
  - `u64_div_to_price(n, d)`: choose exp near denominator scale, compute scaled integer division to retain precision.

- Confidence checks:
  - Factor form: ensure `price > deviation * tolerance` after aligning exponents.
  - Decimal form (bps): ensure `price * confidence_bps > deviation * FULL_BPS`.

- Misc:
  - `mul_bps(amount, bps) = amount * bps / FULL_BPS`.
  - Slot/time conversions using `DEFAULT_MS_PER_SLOT`.
  - Normalize rate decimals by scaling up/down using 10^diff.

## Rounding and Saturation Rules

- Floor is used when converting fractional amounts to integer token transfers to avoid over-crediting users; ceil is used to ensure sufficient collateral when redeeming for a target liquidity.
- Saturating ops (e.g., `saturating_sub`) protect from negative underflows when accounting for fees or interest.
- Minimum fees often enforced at 1 unit to avoid zero-fee edge cases.

## Verification Guidance

- For each handler in KLend listed by name in `lending_operations.rs` (see grep inventory), the numerical logic ultimately invokes the documented math above: reserve interest accrual, exchange rate conversions, LTV checks, fees, and caps. To verify a specific handler:
  1) Locate the call path to `Reserve::{deposit, withdraw, borrow, repay, redeem_fees}` and `Obligation` updates.
  2) Confirm price freshness and elevation-group checks; then apply formulas herein.

- For KVault, all amount computations funnel through `vault_operations.rs::common` and `state.rs` methods listed; CPI edges do not change math, only where amounts are applied.

- For KFarms, arithmetic is localized to `stake_operations.rs` and `farm_operations.rs` as shown.

This completes the deep mathematical analysis for the opened files and referenced functions, covering deposit, withdraw, borrow, repay, collateral exchange, yield, liquidation, flash loans, fees, rewards, and oracle math.

## Handler → Math Coverage Matrix (Traceability)

- KLend/handlers
  - borrow_obligation_liquidity → lending_operations::borrow_obligation_liquidity; Reserve::borrow; Obligation::find_or_add_liquidity/borrow; fees (ReserveFees); utilization gating; token_transfer::borrow_obligation_liquidity_transfer
  - deposit_reserve_liquidity → lending_operations::deposit_reserve_liquidity; Reserve::deposit_liquidity; exchange rate; caps; token_transfer::deposit_reserve_liquidity_transfer
  - deposit_obligation_collateral → lending_operations::deposit_obligation_collateral; ObligationCollateral::deposit; elevation trackers; invariants
  - withdraw_obligation_collateral → lending_operations::withdraw_obligation_collateral; Obligation::withdraw; max_withdraw_value; invariants
  - redeem_reserve_collateral → lending_operations::redeem_reserve_collateral; Reserve::redeem_collateral; caps
  - repay_obligation_liquidity → lending_operations::repay_obligation_liquidity; Reserve::repay; Obligation::repay; settle vs repay_amount; invariants
  - redeem_fees / withdraw_protocol_fees → Reserve::calculate_redeem_fees / ::redeem_fees; market-level accounting
  - flash_borrow_reserve_liquidity / flash_repay_reserve_liquidity → flash_ixs checks; ReserveFees::calculate_flash_loan_fees; Reserve::borrow/repay symmetrical
  - liquidate_obligation_and_redeem_reserve_collateral → liquidation_operations::{calculate_liquidation, bonus, protocol_fee}; exchange conversions
  - refresh_reserve(_batch) → accrue_interest; price age; last_update
  - refresh_obligation(_farms_for_reserve) → recompute market values and tiers; invariants
  - set_obligation_order → order_operations; ranges and interpolation
  - request_elevation_group → elevation group constraints and trackers
  - socialize_loss → proportional reduction across depositors

- KFarms/handlers
  - stake/unstake → stake_operations::{add_active_stake, remove_active_stake, penalties}; update tallies
  - harvest_reward → reward per share delta; treasury fee split
  - refresh_farm/user_state → refresh_global_rewards; user_refresh_all_rewards; pending activation/withdrawal migrations
  - withdraw_from_farm_vault → proportional split active/pending via u64_mul_div

- KVault/handlers
  - deposit/withdraw → vault_operations::deposit/withdraw; shares math; fees charging; rounding loss handling
  - invest → target allocations; exchange conversions and rounding; crank funds use
  - fee-related → charge_fees path; give_up_pending_fees adjustments

### Line-by-line Math Commentary: KLend Borrow Handler

- handler_borrow_obligation_liquidity.rs → lending_operations::borrow_obligation_liquidity
  - Lines 58-61: Preconditions; sets up reserve/market/obligation; clock for slots used in staleness and rate comp.
  - Lines 70-73: `deposit_reserves_iter` constructed for valuation; used to recompute invested values and tiers (affects remaining borrow value).
  - Lines 74-95: Optional referrer; if present, validate token mint matches reserve liquidity mint, and that the obligation’s referrer equals provided account; this gates the referral branch in fee computation (mathematically enabling `absolute_referral_rate`).
  - Lines 97-100: Snapshot reserve balances for postconditions; deltas must match `(borrow_fee + receive_amount)` as Subtractive.
  - Lines 101-114: Core call computes:
    - `borrow_amount_f` (Fraction) subject to min of remaining borrow capacity/value and liquidity, plus fees; see math (calculate_borrow) for inclusive/exclusive handling.
    - `receive_amount = floor(borrow_amount - protocol_fee - referrer_fee)`; must be > 0.
  - Lines 118-131: If `borrow_fee > 0` transfer origination fee; fee rounding occurs upstream in `calculate_borrow_fees` (`max(1)` enforcement), so integer token transfer matches floored fee.
  - Lines 133-142: Transfer of `receive_amount` from reserve to user; mints obey `decimals` and use `transfer_checked` semantics.
  - Lines 144-151: Post delta checks ensure `available_amount` reduced by `borrow_fee + receive_amount` and vault balance moved accordingly; the math matches Reserve::borrow which floors to u64 and adds fractional debt to `borrowed_amount_sf`.

Refer back to sections “Borrowing” and “Interest accrual” for precise formulas used by `CalculateBorrowResult` and debt updates.

## Additional Handler Math Details (KLend)

- Deposit Obligation Collateral (handler_deposit_obligation_collateral)
  - Preconditions: Reserve freshness; elevation-group gating when `disable_usage_as_coll_outside_emode > 0` and obligation outside emode with debt.
  - Math effect: Increase `ObligationCollateral.deposited_amount` by `collateral_amount` and mark last_update stale; update elevation trackers if newly added reserve. Market value accrues from `exchange_rate.fraction_collateral_to_liquidity(collateral_amount)` times price (applied during obligation refresh).
  - Invariants: `post_deposit_obligation_invariants(liq_value_from_collat, obligation, reserve, pre_collateral_mv, min_net_value)`.

- Withdraw Obligation Collateral (handler_withdraw_obligation_collateral)
  - If no borrows: `withdraw_amount = min(requested or u64::MAX → all, deposited_amount)`.
  - Else compute `max_withdraw_value`: see Obligation::max_withdraw_value using either `reserve_max_ltv_pct` or `reserve_liq_threshold_pct` per `LtvMaxWithdrawalCheck`.
    - For arbitrary `withdraw_amount`, compute `withdraw_value = collateral_value * withdraw_amount / deposited_amount` and ensure it does not exceed `max_withdraw_value` (strict or non-strict for liquidation threshold path).
  - Token movement equals `withdraw_amount` of cTokens; LTV recomputed after; if full withdrawal and no other positions, trackers update and account can close.
  - Invariants: `post_withdraw_obligation_invariants(fraction_collateral_to_liquidity(withdraw_amount), ... reserve_liq_threshold, prev_collateral_mv, min_net_value)`.

- Refresh Reserve / Refresh Reserves Batch
  - Interest accrual over `slots_elapsed` with `approximate_compounded_interest` (see earlier section). Updates cumulative borrow index and borrowed amount, protocol and referral fee accumulators.
  - Price update logic: fetch if `is_price_refresh_needed` (age threshold) and adapters validate. Sets `market_price_sf` and `market_price_last_updated_ts`.
  - Limit timestamps updated when deposit/borrow limits are crossed. Emits price for observability.

- Repay Obligation Liquidity
  - Accrue per-liquidity interest to align borrowed amount with cumulative index.
  - Compute `settle_amount = min(requested (as Fraction), borrowed)`; `repay_amount = ceil(settle)` for integer transfer; token transfer moves `repay_amount`; reserve.available += `repay_amount`; borrowed decreases by `settle_amount` in Fraction.
  - Invariants: `post_repay_obligation_invariants(settle_amount, obligation, repay_reserve, borrow_market_value, min_net_value)`.

- Liquidate Obligation and Redeem Reserve Collateral
  - Select liquidation parameters (user_ltv, bonus) per reason (LTV exceeded, autodeleverage, order execution), possibly using `max_allowed_ltv_override_pct` (owner-only and staging gated).
  - Compute `debt_liq_amount_f` bounded by close factor, market cap, and leg-specific constraints. Calculate repay_amount = ceil(settle) based on the chosen leg; derive `withdraw_collateral_amount` and optional redeem `withdraw_liquidity_amount` using exchange rate.
  - Protocol liquidation fee: `protocol_fee = ceil((withdraw_liquidity - withdraw_liquidity/(1+bonus)) * protocol_fee_pct)`; transferred from liquidator liquidity to reserve fee receiver.
  - Signed invariants when repay and withdraw happen in the same reserve vault: `LendingAction::SubstractiveSigned(withdraw_liquidity - repay_amount)`.

- Request Elevation Group
  - No arithmetic change; builds iterators for deposits/borrows/referrer states; validates remaining account count; calls `request_elevation_group`, which enforces borrowing enablement and capacity within the elevation group (see lending_operations and reserve.borrow_factor_f math).

- Set Obligation Order
  - Order fields (min/max bonus bps, condition/opportunity) are validated; math impact is deferred until liquidation execution (interpolation and capping vs diff-to-bad-debt).

- Update Reserve Config
  - Accrue interest before changing config; if `skip_config_integrity_validation`, config changes allowed only if reserve unused (available > min_initial_deposit? false) and both deposit/borrow limits set to 0; otherwise the integrity validator runs (ensuring e.g., LTV < liquidation threshold, bonus bps bounds, etc.).

- Update Global Config
  - No numeric transformations beyond per-mode (de)serialization.

- Update Lending Market
  - Validations per mode (bool, ranges, non-zero, utf8 name). Elevation group fields validated by `validate_new_elevation_group`:
    - `ltv_pct <= liquidation_threshold_pct < 100`, `max_liquidation_bonus_bps <= FULL_BPS`, `id < MAX_NUM_ELEVATION_GROUPS`, non-empty debt reserve/max collateral when id != NONE, and `liquidation_threshold * (1 + max_bonus) <= 100%`.

- Withdraw Referrer Fees
  - `withdraw_amount = min(referrer_token_state.amount_unclaimed, reserve.liquidity.accumulated_referrer_fees, reserve.available_amount)` floored to u64; reduces reserve.available; decrements referrer unclaimed and reserve accumulated referrer fees accordingly (see state::reserve methods).

- Withdraw Protocol Fees
  - `amount = min(requested, fee_vault.amount)` transferred from reserve fee vault to the global fee collector ATA.

- Combo: Deposit + Collateral (deposit_reserve_liquidity_and_obligation_collateral)
  - `collateral_amount = liquidity_to_collateral(liquidity_deposited)` via current exchange rate; then collateral deposited into obligation; AUM and LTV adjust on next obligation refresh.

- Combo: Withdraw + Redeem (withdraw_obligation_collateral_and_redeem_reserve_collateral)
  - `withdraw_obligation_amount` computed per LTV check; then `withdraw_liquidity_amount = collateral_to_liquidity(withdraw_obligation_amount)`; transfer both legs accordingly; LTV recomputed.

- Combo: Deposit and Withdraw (deposit_and_withdraw)
  - Capture `initial_ltv`; perform deposit leg (liquidity -> collateral -> obligation), refresh reserve/obligation, then withdrawal with liquidation-threshold path; refresh again and enforce `post_deposit_and_withdraw_obligation_enforcements(obligation, withdraw_reserve, market, initial_ltv)` to ensure LTV movement and state consistency.

- Init Handlers
  - init_lending_market: initialize owner, bump, and quote currency; no math beyond assignment.
  - init_reserve: set initial `available_amount = min_initial_deposit_amount`, initial collateral supply equal to min initial, Hidden status; performs an initial liquidity transfer to supply vault.
  - init_obligation: zero-initialize arrays, set owner, referrer (from user metadata), tag and timestamp/slot.

## KLend: Farms/Referrer/Init Math & Effects

- InitFarmsForReserve / InitObligationFarmsForReserve
  - Pure wiring via CPIs to KFarms; no arithmetic state changes beyond storing farm pubkeys on `Reserve` or initializing farmer state. Rewards accounting math happens in KFarms.

- InitReferrerStateAndShortUrl / DeleteReferrerStateAndShortUrl
  - No numeric math; enforces ASCII-alphanumeric (underscore/hyphen) constraint on short URLs; state closes return lamports via account constraints.

- InitReferrerTokenState
  - Initializes `ReferrerTokenState` with zero unclaimed/cumulative amounts and reserve mint; math applies later during interest accrual and withdrawals (see Reserve::compound_interest and withdraw_referrer_fees).

- InitGlobalConfig / InitLendingMarket / InitReserve / InitObligation / InitUserMetadata
  - Initialization writes only; no numeric transformations except setting `min_initial_deposit_amount` for reserve starting balances and collateral supply. All subsequent math follows the earlier sections.

## KFarms Math (handlers)

- Stake
  - Input amount possibly set to `user_ata.amount` if `u64::MAX` sentinel.
  - farm_operations::stake computes:
    - If warmup: add_pending_deposit_stake; else add_active_stake; both use Decimal math: `convert_amount_to_stake = total_stake * amount / total_amount` with precise scaling.
    - Updates reward tallies on active stake increase: `new_tally = old + added_shares * reward_per_share` (Decimal scaled u128).
  - Token transfer: amount_to_stake from user_ata → farm_vault.

- Unstake
  - farm_operations::unstake computes `amount_to_unstake` in tokens from Decimal shares using `convert_stake_to_amount = floor(stake * total_amount / total_stake)`; applies early withdrawal penalty per locking mode:
    - WithExpiry/Continuous: penalty = mul_bps(amount, locking_early_withdrawal_penalty_bps) gated by time.
  - Adds pending_withdrawal stake; no immediate token transfer here; actual withdrawal happens via withdraw_unstaked_deposits.

- HarvestReward
  - farm_operations::harvest:
    - refresh reward index; compute delta reward for user = floor((rps_user_new - rps_user_old)); enforce min claim duration.
    - Split treasury fee: `reward_treasury = u64_mul_div(reward, treasury_fee_bps, BPS_DIV_FACTOR)`; `reward_user = reward - reward_treasury`.
  - Two token transfers from rewards_vault: to user and to treasury vault.

- WithdrawFromFarmVault
  - Proportional removal: `removed_active = floor(total_active_amount * req / vault_amount)` and same for pending.
  - final_amount_to_withdraw = removed_active + removed_pending.
  - Token transfer from farm_vault to withdrawer.

## KVault Math (handlers)

- Deposit
  - vault_operations::deposit performs fee charging, AUM computation, and derives `shares_to_mint`:
    - If `shares_issued == 0`: `shares_to_mint = user_token_amount`.
    - Else: `shares_to_mint = floor(shares_issued * user_token_amount / ceil(AUM))` where `AUM = token_available + invested_total − pending_fees`.
  - `user_tokens_to_deposit` computed via `compute_amount_to_deposit_from_shares_to_mint` (ceil variant to ensure sufficient deposit for target shares).
  - `crank_funds_to_deposit = num_reserve * crank_fund_fee_per_reserve` is added to the vault for rounding coverage.
  - Postconditions ensure minted shares changed `shares_issued` exactly and user/token vault balances reflect `token_to_deposit + crank_funds`.

- Withdraw (available-only or invested path)
  - Determine `total_for_user = floor(AUM * shares_to_withdraw / shares_issued)` (or `floor(AUM)` if withdrawing full supply).
  - Split into `available_to_send_to_user = min(token_available, total_for_user)` and the remainder sourced via invested path.
  - If invested path provided for a specific reserve: compute `invested_liquidity_to_send_to_user_f = min(invested_in_reserve, total_for_user - available)` then compute ctokens to burn: `ceil(liquidity_to_collateral(invested_liq_floor))`; cap by available ctokens.
  - Recompute liquidity from ctokens burned: floor/ceil as appropriate; rounding loss computed and optionally covered by `available_crank_funds`.
  - Shares to burn: `ceil(amount_to_send_to_user * total_supply / floor(AUM))` clamped by user’s shares.
  - Postconditions: vault/user balances and reserve supply deltas match `WithdrawEffects` fields.

- CPI aspects
  - Reserves freshly refreshed before math; redeem leg uses CPI with `base_vault_authority` seeds.
  - Rounding handled by transferring `invested_liquidity_to_send_to_user` and compensating via crank funds if possible.

### KFarms math — delegated set_stake and user initialization

- set_stake (delegated farms)
  - Preconditions: total_pending_stake_scaled == 0, total_pending_amount == 0, deposit_warmup_period == 0, withdrawal_cooldown_period == 0; and total_active_stake_scaled == total_staked_amount (scaled equality).
  - Let current_stake_amount = floor(user_state.active_stake_scaled) as u64; new_stake is a u64.
  - If equal, no-op.
  - Otherwise define diff = |new_stake − current_stake_amount| and op ∈ {+ , −} by direction. If increasing:
    - initialize_reward_ts_if_needed, set user_state.last_stake_ts = ts.
    - Enforce deposit cap: can_accept_deposit(diff).
  - Updates:
    - farm_state.total_active_stake_scaled op= diff (as u128)
    - farm_state.total_staked_amount op= diff (as u64)
    - user_state.active_stake_scaled op= diff (as u128)
    - For each reward i: rewards_tally_scaled[i] = reward_per_share_scaled[i] * new_stake
  - Rewards math basis (scaled Decimal D): reward_due_i = floor((rps_i * new_stake) − rewards_tally_prev_i); then rewards_tally_i := rewards_tally_prev_i + reward_due_i.

- initialize_user
  - Delegated=false branch enforces payer == delegatee == authority == owner.
  - Delegated=true branch enforces authority ∈ {delegate_authority, second_delegated_authority}.
  - Sets PDA bump and delegatee; initializes user state; reward math unaffected yet.

### KVault math — withdraw_pending_fees and give_up_pending_fees

- withdraw_pending_fees
  - Inputs: reserve allocation; effects = { available_to_send_to_user, invested_to_disinvest_ctokens, invested_liquidity_to_send_to_user, invested_liquidity_to_disinvest } returned by vault_operations::withdraw_pending_fees.
  - Step 1: If invested_to_disinvest_ctokens > 0, redeem cTokens via CPI to KLend. Let Δvault_tokens = token_vault_after_redeem − token_vault_before.
  - Constraint: Δvault_tokens ≥ invested_liquidity_to_send_to_user.
  - Step 2: Transfer to admin ATA: transfer_amount = available_to_send_to_user + invested_liquidity_to_send_to_user.
  - Post-checks ensure VaultAndUserBalances deltas match effects and reserve supply/cToken vault movements are consistent.
  - Accounting identities:
    - token_vault_after = token_vault_before − transfer_amount + Δvault_tokens
    - ctoken_vault_after = ctoken_vault_before − invested_to_disinvest_ctokens

- give_up_pending_fees
  - Given max_amount_to_give_up, vault_operations::give_up_pending_fee computes forgiveness up to cap across reserves using current slot/timestamp.
  - Reduces pending fees liability: pending_fees_after = max(0, pending_fees_before − forgiven_amount), where forgiven_amount ≤ max_amount_to_give_up.
  - No token transfers; affects fee accrual state and future withdrawals.

### KFarms math — rewards lifecycle and refresh/treasury/vault flows

- initialize_reward
  - Sets rewards vault/mint/decimals; no arithmetic besides initializing counters.

- add_reward(amount, reward_index)
  - Let reward_amount = clamp_by_policy(amount, scope_price) returned by farm_operations::add_reward.
  - Token-2022 transfer_checked moves reward_amount to rewards vault with mint decimals.
  - Global: rewards_available[reward_index] += reward_amount.

- withdraw_reward(amount, reward_index)
  - farm_operations::withdraw_reward picks reward_amount ≤ min(requested, rewards_available[ri]).
  - Transfer out reward_amount from rewards vault; rewards_available[ri] -= reward_amount.

- reward_user_once(reward_index, amount)
  - Directly increments user_state.rewards_issued_unclaimed[ri] += amount and decrements farm_state.reward_infos[ri].rewards_issued_unclaimed accordingly (per farm_operations).

- refresh_global_rewards / refresh_user_state
  - Global: reward_per_share_scaled_i advances using emission rate and elapsed time Δt: rps_i_new = rps_i_old + floor(Δissued_scaled / total_active_stake_scaled).
  - User: reward_i = floor((rps_i_new * user_active_stake_scaled) − rewards_tally_scaled_i_prev); tally_i := tally_i_prev + reward_i.

- withdraw_unstaked_deposits
  - Moves pending_withdrawal_unstake to user ATA; amount_to_withdraw derived by prior unstake effects, obeying warmup/cooldown timers.

- deposit_to_farm_vault(amount)
  - Increases farm_vault token balance by amount; internal counters updated via farm_operations::deposit_to_farm_vault.

- withdraw_treasury(amount)
  - Checks available = reward_treasury_vault.amount; requires amount > 0 and ≤ available; transfers exactly amount.

- transfer_ownership
  - Unstake all from old user: compute WithdrawEffects { amount_to_withdraw } equal to prior active stake amount considering penalties=0 (enforced by cooldown/locking = 0).
  - Stake same amount_to_withdraw for new user; require equality of stake and withdrawn to preserve total; keeps conservation: sum stakes conserved.

### KVault math — config/allocation/init

- update_vault_config(entry, data)
  - First computes holdings and charges fees to snapshot state at pre-change values.
  - Field updates:
    - PerformanceFeeBps: require ≤ FULL_BPS.
    - ManagementFeeBps: require ≤ MAX_MGMT_FEE_BPS.
    - MinWithdrawAmount: require ≤ UPPER_LIMIT_MIN_WITHDRAW_AMOUNT.
    - Name: bytes -> fixed array via slice_to_array_padded.
    - PendingVaultAdmin, LookupTable, Farm, AllocationAdmin: set pubkeys.
    - MinDepositAmount/MinInvestAmount/MinInvestDelaySlots/UnallocatedWeight/UnallocatedTokensCap: set numerics.

- update_reserve_allocation(target_weight, cap)
  - Admin rules: insert requires vault admin; update allows allocation admin or vault admin.
  - Require reserve.liquidity.mint_pubkey == vault.token_mint (same base token).
  - Upsert allocation with ctoken_vault PDA bump and provided weight/cap.

- remove_allocation
  - Removes the reserve from internal allocation map; must be existing and removable per VaultState checks.

- init_vault
  - Initializes vault fields; calls vault_operations::initialize with token and shares decimals and current timestamp.
  - Performs an initial deposit of INITIAL_DEPOSIT_AMOUNT to set non-degenerate share price.
  - DepositEffects: shares_to_mint = ceil(INITIAL_DEPOSIT_AMOUNT * total_shares / AUM_before) with bootstrap logic inside; token_to_deposit plus crank_funds_to_deposit are transferred from admin.

### KLend math — lending_market modules

- config_items validators
  - check_bool: value ∈ {0,1}.
  - check_not_zero: value ≠ 0.
  - check_not_negative: value ≥ 0 (for signed-converted types).
  - check_in_range[a,b]: enforce a ≤ value ≤ b; used by check_valid_pct (0..=100) and check_valid_bps (0..=FULL_BPS).
  - check_gte(min), check_lte(max): order constraints.
  - representing_u8_enum: domain = image(TryFrom<u8>), else InvalidConfig.
  - Rendering does not affect math; as_fraction formats Fraction::from_bits(value).

- ix_utils CPI depth
  - Let H = get_stack_height(). For same-program call, require H ≤ TRANSACTION_LEVEL_STACK_HEIGHT.
  - For non-self calls, if program_id ∉ CPI_WHITELISTED_ACCOUNTS -> forbidden; else allowed only if H ≤ TRANSACTION_LEVEL_STACK_HEIGHT + whitelist_level.
  - Flash path: stricter; any non-self or H > TRANSACTION_LEVEL_STACK_HEIGHT returns forbidden.

- withdrawal_cap_operations
  - Types: WithdrawalCaps { current_total: i128, config_capacity: i128, last_interval_start_timestamp: u64, config_interval_length_seconds: u64 }.
  - add_to_withdrawal_accum(caps, req, t): check_and_update(Add); sub_from_withdrawal_accum: check_and_update(Remove).
  - If interval length > 0:
    - If t − last_start ≥ interval_length: reset current_total := 0; last_start := t.
    - For Add: require current_total + req ≤ config_capacity and capacity ≥ 0.
    - Update counter with checked add/sub; overflow -> MathOverflow/IntegerOverflow.
  - If interval length = 0:
    - Update counter with saturating add/sub.
  - Invariant per interval: 0 ≤ current_total ≤ config_capacity (when interval_length > 0). Monotonicity: last_start ≤ t.

### Scope price math — mappings, refresh, TWAP/EMA, divergence

- DatedPrice and scaling
  - Price(value, exp) represents value × 10^{-exp}. To rescale to decimals d: if exp > d then scaled = value / 10^{exp-d} else scaled = value × 10^{d-exp}.
  - Comparisons normalize exponents; if exponent gap > MAX_SAFE_EXP_DIFF, ordering short-circuits conservatively to avoid overflow.

- Reference price divergence
  - Let p = curr_price (Decimal), r = ref_price (Decimal). Accept if |p − r| ≤ r × 5%.
  - Equivalent inequality: 100 × |p − r| ≤ 5 × r.

- Refresh execution constraints
  - For a refresh transaction at instruction index i: program_id at i must equal Scope program id; get_stack_height() ≤ TRANSACTION_LEVEL_STACK_HEIGHT; for all j < i, instruction[j].program_id == ComputeBudget111... . Violation -> RefreshInCPI/UnexpectedIxs.

- TWAP enablement and source
  - Mapping toggles twap_enabled[i] ∈ {0,1}; twap_source[i] indexes source price. When enabled, each refresh updates EMA and stores EmaTwap in OracleTwaps.

- EMA/TWAP formula and sampling
  - Period T = 3600s (1h). Adjusted smoothing alpha based on Δt = current_ts − last_ts: alpha = 1 if Δt ≥ T; error if Δt < T/120; else alpha = 2 / (1 + T/Δt).
  - Update rule: EMA_new = alpha × price + (1 − alpha) × EMA_prev; values represented with Decimal scaled integers.
  - Sample tracker is a 64-bit ring over window T. Erase points outside window; set current point; invariants:
    - 0 ≤ samples_in_window = popcount(tracker) ≤ 64
    - MIN_SAMPLES_IN_PERIOD ≤ samples_in_window
    - First and last subperiods among N=3 subperiods each have ≥ 1 sample.

- Reset TWAP
  - Resets EMA state and tracker for index; post-condition: last_update_slot = 0; samples_in_window = 0; next update seeds EMA with first price.

### Scope oracle math — Pyth Lazer, Chainlink, MostRecentOf, Capped/Floored, Jupiter LP

- Pyth Lazer
  - External verification: signature and message verified by Pyth Lazer program; Scope trusts result.
  - update_price maps payload i -> DatedPrice(price, slot, ts) with mapping-specific generic params; then optional TWAP update and 5% reference divergence check.

- Chainlink (Streams)
  - Verified report returned as return_data; decoded by variant V3/V7/V8/V9/V10 depending on OracleType.
  - Each update_price_vX enforces report freshness, correct aggregator/config, scales to Price(value,exp). After write, optional TWAP and 5% reference divergence check.

- MostRecentOf
  - Inputs: source_entries[4], max_divergence_bps, sources_max_age_s.
  - For each source k: enforce now − ts_k ≤ sources_max_age_s; track P_min = min_k Price_k, P_max = max_k Price_k; choose DatedPrice with maximum ts.
  - Divergence constraint: let s = Decimal(P_min), spread = Decimal(P_max) − s. Enforce spread ≤ s × (max_divergence_bps / FULL_BPS). If violated, reject.

- Capped/Floored
  - Start from source DatedPrice P_s. Optional caps: P_cap, P_floor.
  - If both and P_cap < P_floor -> reject.
  - Final Price P = min(P_s, P_cap) if cap exists, then P = max(P, P_floor) if floor exists. Timestamps retained from source.

- Jupiter LP
  - No recompute: price = AUM_usd / supply, with decimals 6 required on LP mint.
  - Recompute path:
    - For each custody j with dated price (price_j, ts_j, slot_j):
      - If custody.is_stable: token_amount_usd_j = asset_amount_to_usd(price_j, owned_j, decimals_j); trader_short_profits_j = 0.
      - Else: compute global_short_pnl at price scaled to 6 decimals; if trader_has_profit then trader_short_profits += pnl else pool_amount_usd += pnl; add guaranteed_usd; compute net_assets_usd = asset_amount_to_usd(price_j, owned_j − locked_j, decimals_j); pool_amount_usd += net_assets_usd.
      - Track oldest price age: oldest_ts = min(oldest_ts, ts_j), oldest_slot likewise.
    - LP value (pool value) = pool_amount_usd − trader_short_profits.
    - price_dec = LP_value / lp_token_supply; result Price = Decimal->Price; DatedPrice uses oldest_ts/slot.
  - Scaling helper: asset_amount_to_usd(price: Price(value, exp), amount, token_decimals) computes
    - If exp + token_decimals > 6: (value × amount) / 10^{exp + token_decimals − 6}
    - Else: (value × amount) × 10^{6 − (exp + token_decimals)}

### Scope key oracle math — audit essentials

- Pyth (on-chain and EMA)
  - Staleness: accept agg.pub_slot within 10 min else fallback prev_slot within 10 min else reject.
  - Require exponent ≤ 0; value = |price| (u64).
  - Confidence check: check_confidence_interval(value, exp, conf, exp, ORACLE_CONFIDENCE_FACTOR), typically conf ≤ value / k.
  - EMA variant: use EMA price/conf; same confidence and staleness (10 min) over publish_time.

- Pyth Pull (and EMA)
  - VerificationLevel = Full; use signed message data; staleness gate up to caller; estimate slot from publish_time.
  - Confidence check identical via legacy Price struct (value, exponent, conf).

- Switchboard V2
  - Require latest_confirmed_round.num_success ≥ min_oracle_results.
  - Price scaling: if scale > 10 then exp=10, value=floor(mantissa / 10^{scale-10}); else exp=scale, value=mantissa.
  - Confidence check with std_deviation mantissa/scale against ORACLE_CONFIDENCE_FACTOR.
  - Timestamps from round_open_{slot,timestamp}.

- Switchboard On-Demand
  - Similar scaling with cap 15; validate std_dev; last_updated_slot from sample; unix_timestamp ≈ now − slots_to_secs(now_slot − sample_slot).

- Orca Whirlpool / Raydium v3 (CFMM)
  - Input sqrt_price Q64.64 (Whirlpool) or Q64.64-like: raw sqrt(P) where P=price(B/A) in base units.
  - Price(A->B) = adjust_decimals( (sqrt_price^2) / 2^128, decA, decB ); invert for B->A.
  - adjust_decimals handles 10^{decB-decA} factor to return Price(value, exp).

- Meteora DLMM (bin-based CLMM)
  - Raw price q = get_x64_price_from_id(active_id, bin_step) as Q64.64; if inverted path, use (1<<128)/q.
  - Convert q to lamport price; then scale to token decimals to produce Price(value, exp).

- cTokens (Solend-like)
  - Accrue interest: slots_elapsed = now_slot − last_update.slot; compounded_rate = (1 + r_slot)^{slots_elapsed}; update cumulative_borrow_rate, borrowed, protocol_fees.
  - Collateral exchange rate: if mint_total_supply==0 or total_liquidity==0 -> initial WAD; else rate = supply_collateral / total_liquidity.
  - Scaled rate value = collateral_to_liquidity(10^{15}); output Price(value, exp=15).

### KLend internals — interest, borrow curve, fees, liquidation

- Borrow curve and utilization
  - utilization u = borrowed / total_supply, where total_supply = available + borrowed − protocol_fees (Decimal/Fraction exact).
  - Piecewise linear borrow rate r(u):
    - If u < u_opt or u_opt = 100%: r = min_rate + (optimal_rate − min_rate) × (u / u_opt)
    - Else: r = optimal_rate + (max_rate − optimal_rate) × ((u − u_opt) / (1 − u_opt))

- Accrual (per reserve)
  - slots_elapsed = now_slot − last_update.slot; if 0 → no-op.
  - slot_rate = r / SLOTS_PER_YEAR.
  - compounded = (1 + slot_rate)^{slots_elapsed}.
  - Update borrowed: B' = B × compounded.
  - Net_new_debt = B' − B.
  - Fees:
    - protocol_take_rate p_take (percent of interest): protocol_fees += Net_new_debt × p_take.
    - host_fixed_interest_rate h_fix (bps): liquidity.compound_interest receives this and accumulates host as fixed interest cut; referral_fee_bps ref similarly.
  - cumulative_borrow_rate_wads *= compounded.
  - available remains; market_price updated elsewhere during refresh.

- Collateral exchange rate
  - If supply_collateral == 0 or total_liquidity == 0: initial WAD (INITIAL_COLLATERAL_RATE).
  - Else: rate = supply_collateral / total_liquidity.
  - liquidity_from_collateral = floor(collateral / rate). collateral_from_liquidity = ceil(liquidity × rate).

- Liquidation math (selected)
  - Borrowed value V_b = liquidity.market_value(). Collateral value V_c from collateral.reserve.
  - Compute liquidation params: user LTV, liquidation bonus b (as Fraction), reason.
  - If V_b < min_full_liquidation_threshold and reason not order: require repay_amount ≥ total borrowed; debt_liq = total borrowed.
  - Else: debt_liq = min(max_liquidatable_borrowed_amount, provided_amount).
    - max_liquidatable_borrowed_amount uses close factor CF:
      - If user_ltv > insolvency_risk_unhealthy_ltv → CF=1 else CF=close_factor_pct.
      - MV caps: min(CF × total_debt_mv, mv_of_this_liquidity, market_max_debt_mv_at_once).
      - Convert back to amount via ratio: debt_liq = borrowed_amount × (MV_cap / mv_of_this_liquidity).
  - liquidation_ratio α = debt_liq / borrowed_amount.
  - Total value with bonus: V_settle = V_b × α × (1 + b).
  - calculate_liquidation_amounts splits V_settle into (settle_amount, repay_amount, withdraw_amount) with roundings matching exchange rates.

### KVault internals — AUM, fees, allocation cadence

- AUM
  - AUM(current) = available_tokens + Σ_i liquidity_i (from each allocation via exchange rate), as Fraction. Pending fees are excluded when computing prev_aum and fee accrual basis.

- Shares issuance/redemption
  - First deposit: shares_to_mint = user_amount.
  - Else: shares_to_mint = floor(shares_issued × user_amount / ceil(AUM)).
  - On withdraw, user entitlement E = floor(number_of_shares × AUM / shares_issued). Burn shares according to theoretical amount sent to user (available + invested path) to maintain proportionality.

- Fees (continuous-like, discrete slots)
  - Management fee: mgmt_yearly_bps over elapsed seconds on prev_aum:
    - mgmt_fee_factor = mgmt_bps × seconds_passed / SECONDS_PER_YEAR.
    - mgmt_charge = prev_aum × mgmt_fee_factor.
  - Performance fee: perf_bps × max(AUM − prev_aum, 0).
  - pending_fees += min(mgmt_charge + perf_charge, AUM).
  - prev_aum := AUM − new_fees; last_fee_charge_timestamp := now.

- Allocation and invest cadence
  - For each reserve allocation: target_weight and cap.
  - Gate: current_slot ≥ last_invest_slot + min_invest_delay_slots.
  - Compute invested totals; refresh_target_allocations; per-reserve difference from target determines InvestingDirection Add/Subtract.
  - Add: deposit liquidity to cTokens (collateral_amount from liquidity via exchange rate); Subtract: redeem cTokens (ceil to cover required liquidity); track rounding_loss and recover via crank funds in handler.

- Rounding loss handling on withdraw
  - If fractional liquidity from ctokens redemption has larger fractional part than user fractional entitlement, subtract 1 lamport from user amount to avoid vault loss (liquidity_rounding_error = 1), ensuring conservation and preventing fee leakage.
