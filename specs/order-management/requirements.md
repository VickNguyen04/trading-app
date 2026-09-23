# Requirements — Order Management

**Purpose:** User stories and acceptance criteria for the Order Management module, derived from [`docs/order-management-domain-knowledge.md`](../../docs/order-management-domain-knowledge.md). Each story is traceable back to a source section (`Source: §X`); each acceptance-criteria scenario that encodes a numbered edge case is tagged inline (`Edge case §8.N`).

**Actors:** Trader, Risk Manager, System Admin, Market Data Feed (external), Matching Engine (internal), System (automated behavior with no human actor).

---

## Epic 1 — Place New Order

*Source: §3.1, §4*

### 1.1 Market Order

**Source:** §4.1

As a Trader, I want to place a market order, so that my order fills immediately at the best available price without specifying a price.

**Acceptance Criteria:**

- **Given** the market is open and a symbol is tradable, **when** I submit a market order, **then** the system routes it to the Matching Engine immediately and it fills at the best available price (subject to slippage).
- **Given** I submit a market order, **when** I attempt to set TIF to `OPG` or `CLS`, **then** the system accepts it only during the corresponding auction session; if unmatched in that session, the order is canceled.
- **Given** I submit a market order with TIF `IOC`, **when** the order is placed, **then** the system fills whatever quantity it can immediately and cancels the remainder.
- **Given** I submit a market order with TIF `FOK`, **when** the full quantity cannot be filled immediately, **then** the system cancels the entire order (no partial fill).
- **Given** it is after 4:00pm ET, **when** I submit a market order, **then** the system queues it for the next trading session instead of rejecting it (Edge case §8.7).

### 1.2 Limit Order

**Source:** §4.2

As a Trader, I want to place a limit order at a specified price, so that I control the maximum (buy) or minimum (sell) price I'm willing to trade at.

**Acceptance Criteria:**

- **Given** I submit a buy limit order, **when** the market price is at or below my limit price, **then** the order fills at my limit price or better.
- **Given** I submit a sell limit order, **when** the market price is at or above my limit price, **then** the order fills at my limit price or better.
- **Given** my limit price is immediately marketable, **when** I submit the order, **then** it fills right away; **given** it is not marketable, **when** I submit the order, **then** it rests on the order book.
- **Given** I submit a limit order with a price ≥ $1.00, **when** the price has more than 2 decimal digits, **then** the system rejects the order at input time (Edge case §8.10).
- **Given** I submit a limit order with a price < $1.00, **when** the price has more than 4 decimal digits, **then** the system rejects the order at input time (Edge case §8.10).
- **Given** a symbol has not yet begun trading (pre-IPO), **when** I attempt to submit any order type other than limit, **then** the system rejects it — only limit orders are accepted (Edge case §8.8).
- **Given** I want my order to be eligible during extended hours, **when** I submit it, **then** the system requires order type = limit **and** TIF = `DAY` or `GTC`; any other combination is rejected for extended-hours eligibility (Edge case §8.9).

### 1.3 Stop Order

**Source:** §4.3

As a Trader, I want to place a stop order, so that it automatically converts into a market order once the market reaches my stop price.

**Acceptance Criteria:**

- **Given** I submit a stop order with a stop price, **when** the market price touches the stop price, **then** the system converts it into a market order and routes it to the Matching Engine.
- **Given** a stop order has been triggered and converted, **when** it executes, **then** the resulting fill price is not guaranteed to equal the stop price (full slippage risk of a market order applies).
- **Given** I submit a stop order, **when** I attempt to set TIF to `IOC`, `FOK`, `OPG`, or `CLS`, **then** the system rejects the TIF — these apply only to Market and Limit orders.

### 1.4 Stop-Limit Order

**Source:** §4.4

As a Trader, I want to place a stop-limit order, so that once triggered it becomes a limit order instead of a market order, protecting me from unlimited slippage.

**Acceptance Criteria:**

- **Given** I submit a stop-limit order with a stop price and a limit price, **when** the market touches the stop price, **then** the order becomes "elected" and converts into a limit order at the specified limit price.
- **Given** an elected stop-limit order, **when** the market price is at or better than the limit price, **then** it fills like a normal limit order.
- **Given** an elected stop-limit order, **when** the price gaps through both the stop price and the limit price in the same move, **then** the order remains elected but is **not** executed — it rests on the book as a pending limit order and does **not** auto-cancel (Edge case §8.1).
- **Given** I submit a stop-limit order, **when** I attempt to set TIF to `IOC`, `FOK`, `OPG`, or `CLS`, **then** the system rejects the TIF — these apply only to Market and Limit orders.

### 1.5 Bracket Order

**Source:** §4.5

As a Trader, I want to place a bracket order (entry + take-profit + stop-loss), so that my exit is automatically managed once my entry fills.

**Acceptance Criteria:**

- **Given** I submit a bracket order, **when** the entry leg has not fully filled, **then** the take-profit and stop-loss legs remain inactive.
- **Given** the entry leg fills completely, **when** the fill is confirmed, **then** the system activates both the take-profit and stop-loss legs.
- **Given** both exit legs are active, **when** either the take-profit or the stop-loss fills, **then** the system automatically cancels the other exit leg.
- **Given** the take-profit leg fills only partially, **when** the partial fill is confirmed, **then** the system automatically reduces the stop-loss leg's quantity to match the remaining open position (Edge case §8.2).

### 1.6 OCO Order (One-Cancels-Other)

**Source:** §4.6

As a Trader who already holds a position, I want to place an OCO order (take-profit + stop-loss with no entry leg), so that I can manage my exit with two linked orders where only one can execute.

**Acceptance Criteria:**

- **Given** I already hold a position, **when** I submit an OCO order with a take-profit leg and a stop-loss leg, **then** both legs are immediately active (no entry leg required, unlike Bracket).
- **Given** both OCO legs are active, **when** one leg fills (fully or partially), **then** the system automatically cancels or adjusts the other leg consistently with the Bracket exit-leg behavior (§4.5).

### 1.7 OTO Order (One-Triggers-Other)

**Source:** §4.7

As a Trader, I want to place an OTO order (entry + a single exit leg), so that I get automatic exit management without needing both a take-profit and a stop-loss.

**Acceptance Criteria:**

- **Given** I submit an OTO order, **when** I configure it, **then** I can choose exactly one exit leg — either take-profit **or** stop-loss, not both.
- **Given** the entry leg fills completely, **when** the fill is confirmed, **then** the system activates the single configured exit leg.

### 1.8 Trailing Stop Order

**Source:** §4.8

As a Trader, I want to place a trailing stop order, so that my stop price automatically moves in my favor as the market moves, without me having to adjust it manually.

**Acceptance Criteria:**

- **Given** I place a trailing stop **sell** order with a trail amount, **when** the market price rises to a new high, **then** the system updates the High Water Mark (HWM) to that high and recalculates `stop_price = HWM − trail`.
- **Given** I place a trailing stop **buy** order with a trail amount, **when** the market price falls to a new low, **then** the system updates the HWM to that low and recalculates `stop_price = HWM + trail`.
- **Given** a trailing stop order is active, **when** the market moves against the favorable direction, **then** the HWM does **not** move backward — it only ever moves in the favorable direction.
- **Given** the market price touches the current `stop_price`, **when** that happens, **then** the order triggers per the behavior of a standard Stop Order (§4.3).

---

## Epic 2 — Modify Pending Order

*Source: §3.2, §7*

### 2.1 Modify a Pending Order

As a Trader, I want to modify an order that hasn't filled yet, so that I can adjust price/quantity without canceling and re-entering.

**Acceptance Criteria:**

- **Given** an order is in a non-terminal, non-pending state (e.g. `new`), **when** I submit a modify request, **then** the system transitions it to `pending_replace`.
- **Given** a modify request succeeds, **when** the replacement is applied, **then** the order transitions from `pending_replace` to `replaced`.
- **Given** an order is already in `pending_replace`, **when** I submit a second modify request before the first resolves, **then** the system rejects the second request to prevent conflicting concurrent updates (Edge case §8.3).
- **Given** an order is in a terminal state (`filled`, `canceled`, `expired`), **when** I attempt to modify it, **then** the system rejects the request — terminal states never transition further (§7).

---

## Epic 3 — Cancel Order

*Source: §3.3, §7*

### 3.1 Cancel an Order

As a Trader, I want to cancel an order that hasn't filled yet, so that I can withdraw it from the market when I change my mind.

**Acceptance Criteria:**

- **Given** an order is in a cancelable, non-terminal state, **when** I submit a cancel request, **then** the system transitions it toward `canceled` (via `pending_cancel` where applicable).
- **Given** an order is currently in `pending_replace`, **when** I submit a cancel request, **then** the system **rejects** it — cancel is not permitted while a replace is in flight (Edge case §8.3, invalid transition per §7).
- **Given** a buy order is canceled before it fills, **when** the cancellation is confirmed, **then** the system releases the locked buying power immediately (§6.2).
- **Given** a sell-long or buy-to-cover order is canceled before it fills, **when** the cancellation is confirmed, **then** no buying-power release is needed since those only affect buying power upon execution, not placement (§6.2).
- **Given** an order is in a terminal state (`filled`, `canceled`, `expired`), **when** I attempt to cancel it, **then** the system rejects the request (§7).

---

## Epic 4 — View Order Status

*Source: §3.4, §7*

### 4.1 View Current Order Status

As a Trader, I want to view the current status of my order, so that I know whether it's working, filled, or done.

**Acceptance Criteria:**

- **Given** I request an order's status, **when** the system responds, **then** the status is one of: `new`, `partially_filled`, `filled`, `done_for_day`, `canceled`, `expired`, `replaced`, `pending_cancel`, `pending_replace`.
- **Given** an order's status is `filled`, `canceled`, or `expired`, **when** I check it later, **then** its status never changes further — these are terminal states (§7).
- **Given** an order is `partially_filled`, **when** I view it, **then** the system shows both the filled quantity and the remaining open quantity.

---

## Epic 5 — View Order History

*Source: §3.5*

### 5.1 View Order History

As a Trader, I want to view my historical orders, so that I can review past trading activity.

**Acceptance Criteria:**

- **Given** I have placed orders in the past, **when** I request my order history, **then** the system returns orders across all statuses, including terminal ones (`filled`, `canceled`, `expired`).
- **Given** I request order history, **when** the results are returned, **then** each order includes its type, TIF, quantities, prices, and final/current status.

---

## Epic 6 — System Auto-Cancels Expired Orders

*Source: §3.6, §5*

### 6.1 DAY Orders Expire at End of Session

As the System, I want to automatically cancel unfilled DAY orders at end of session, so that traders don't carry unintended exposure overnight.

**Acceptance Criteria:**

- **Given** a DAY order has not fully filled by session close, **when** the session ends, **then** the system automatically cancels the remaining open quantity.

### 6.2 GTC Orders Expire After 90 Days

As the System, I want to automatically expire GTC orders after 90 days, so that stale resting orders don't remain indefinitely.

**Acceptance Criteria:**

- **Given** a GTC order has not filled or been canceled, **when** 90 days elapse from placement, **then** the system automatically transitions it to `expired`.

### 6.3 OPG / CLS Orders Cancel if Unmatched in Auction

As the System, I want to cancel OPG/CLS orders that don't match during their target auction, so that they don't linger outside their intended execution window.

**Acceptance Criteria:**

- **Given** an order has TIF `OPG`, **when** the opening auction concludes without a match, **then** the system cancels the order.
- **Given** an order has TIF `CLS`, **when** the closing auction concludes without a match, **then** the system cancels the order.

### 6.4 IOC / FOK Orders Resolve Immediately on Placement

As the System, I want to immediately fill-or-cancel IOC/FOK orders on placement, so that traders never have unfilled IOC/FOK quantity resting on the book.

**Acceptance Criteria:**

- **Given** an order has TIF `IOC`, **when** it is placed, **then** the system fills whatever is immediately available and cancels the unfilled remainder right away.
- **Given** an order has TIF `FOK`, **when** the full quantity cannot be filled immediately, **then** the system cancels the entire order with zero fill.
- **Given** an IOC order is placed, **when** there appears to be sufficient liquidity on paper, **then** the order may still be fully canceled without a fill due to execution-time conditions (Edge case §8.4).

---

## Epic 7 — Risk Manager Blocks Orders Exceeding Limits

*Source: §3.7*

### 7.1 Risk Manager Intervenes on Orders Exceeding Exposure Limits

As a Risk Manager, I want to block or intervene on orders that exceed exposure limits, so that the firm's risk stays within acceptable bounds.

**Acceptance Criteria:**

- **Given** an order would exceed a defined exposure limit, **when** it is submitted, **then** the Risk Manager has the ability to block it before it reaches the Matching Engine.

> **⚠️ Needs clarification:** The domain source (§9, Open Questions) does not specify how Risk Manager override actually works — what limits are checked, at what stage (pre-trade vs. post-trade), or what the intervention UX/API looks like. No further AC can be written until this is confirmed with a stakeholder; do not treat the single AC above as complete.

---

## Epic 8 — Buying Power Management

*Source: §6 (cross-cutting; applies across Epics 1–3)*

### 8.1 Lock Buying Power on Order Placement

**Source:** §6.1

As the System, I want to lock the order's value against available buying power the moment an order is placed, so that a trader cannot over-commit funds across multiple simultaneous orders.

**Acceptance Criteria:**

- **Given** a trader places an order that opens a position, **when** the order is accepted, **then** the system immediately deducts (locks) the order's value from available buying power — this is a temporary hold, not an actual cash debit.

### 8.2 Release Buying Power on Cancel or Execution

**Source:** §6.2

As the System, I want to release locked buying power at the correct point in the order lifecycle, so that traders regain capacity as soon as it's actually freed up.

**Acceptance Criteria:**

- **Given** a buy order is canceled before it fills, **when** the cancellation is confirmed, **then** the system releases the locked buying power immediately.
- **Given** a sell-long or buy-to-cover order is placed, **when** it is merely `new` (not yet executed), **then** no buying power is released; **when** it later executes, **then** buying power is released/adjusted only at that point.

### 8.3 Far-Side and Time-of-Day Pricing

**Source:** §6.3, §6.4

As the System, I want to price buying-power impact using the least favorable (far side) price and the correct rule for the current time window, so that buying power locks are always conservative enough to cover execution risk.

**Acceptance Criteria:**

- **Given** a buy order, **when** the system calculates its buying-power impact, **then** it uses the ASK price.
- **Given** a sell order, **when** the system calculates its buying-power impact, **then** it uses the BID price.
- **Given** it is regular trading hours, **when** pricing an order, **then** the system uses the far side of NBBO.
- **Given** it is extended hours, **when** pricing an order, **then** the system uses the midpoint of Bid/Ask.
- **Given** it is fully outside trading hours, **when** pricing an order, **then** the system uses the last traded price.

### 8.4 Short-Sell Buying Power Buffer

**Source:** §6.5

As the System, I want to apply a conservative buffer and continuous recalculation to short-sell buying power, so that the firm is protected against the unlimited loss potential of short positions.

**Acceptance Criteria:**

- **Given** a trader submits a short-sell order, **when** the system calculates required buying power, **then** it computes `MAX(limit_price, 1.03 × current_ask) × quantity` — a 3% buffer above the current ask (or the limit price if higher).
- **Given** an open short-sell order, **when** the current market price changes, **then** the system continuously recalculates the locked buying power based on live price — it is not fixed at the price when the order was placed.
- **Given** the market price for a shorted symbol rises, **when** the system recalculates, **then** the locked buying-power requirement increases accordingly, which can trigger a margin call (Edge case §8.5).
- **Given** a short position incurs a real trading loss, **when** that loss is realized, **then** it can exceed the amount that was originally locked at order placement — the initial lock is not a loss cap (Edge case §8.6).

---

## Open Questions

*Carried forward from source §9 — unresolved as of this writing.*

1. Does the system need to support all 8 order types at MVP, or start with Market/Limit/Stop/Stop-Limit only?
2. Is extended-hours trading support required in the first phase?
3. Asset scope: equities only, or crypto/options from day one?
4. How exactly does Risk Manager override work (see Epic 7 clarification note)?
5. What specific compliance/regulatory requirements must be satisfied?

**Assumption:** All business rules in the source domain document are based on the Alpaca Markets API reference and require confirmation with actual stakeholders before being treated as final.
