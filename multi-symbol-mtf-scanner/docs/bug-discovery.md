# The Bug That Changed Everything

## Summary
Early versions of this strategy showed a Profit Factor of **1.62** on ASELS — looked great on paper. However, when I added proper state management to the code, the PF dropped to **0.27**. 

This document explains the investigation that proved the strategy was a trend-follower without any actual risk management working.

## Symptoms
- Backtest showed +12.24% return with PF 1.62 on ASELS.
- Trades only exited on "reverse signals" (MA crossunder).
- TP and SL levels appeared in charts but never executed in the Strategy Tester.

## Root Cause
The strategy used `var` variables to hold state (entry price, SL, TP) across bars. On the exact bar the signal fired:
1. `v_entry` and `v_sl` were calculated and set.
2. `strategy.entry()` queued the order.
3. The line `if strategy.position_size == 0` evaluated to **TRUE** (because the order had not filled yet on that specific tick).
4. As a result, `v_entry` and `v_sl` were immediately reset to `na`.

On the next bar, the position opened, but because the stop-loss variable was `na`, the `strategy.exit()` command executed without a stop-loss. Trades were left unprotected and only closed on a reverse signal.

## The Fix
I changed the reset condition to only trigger when a position transitions from open to closed (using the history reference `[1]`):

    bool position_just_closed = strategy.position_size == 0 and strategy.position_size[1] > 0
    if position_just_closed
        v_entry := na
        v_sl    := na

## Aftermath
- **Before fix:** PF 1.62 (Fake — unprotected trades allowed trends to run indefinitely).
- **After fix:** PF 0.27 (Real — stop-losses were finally executing).

This discovery proved that the strategy's edge didn't come from a strict SL/TP setup. It was a pure trend-follower.

This bug led directly to the v1.0 exit optimization test, which proved that removing the TP completely and letting the SL + Reverse Signal handle exits restored the edge (avg PF 1.08 across 5 symbols).