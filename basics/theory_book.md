# Order Books: A Full Reference

## What An Order Book Is

An order book is the continuously maintained, live record of every outstanding buy and sell instruction for a financial instrument. The instrument can be anything that gets traded on an organized venue: equities, futures, options, bonds, foreign exchange pairs, cryptocurrency pairs, interest rate swaps traded on a swap execution facility, prediction market contracts, or any other standardized product with a price and a buyer and a seller on the opposite side.

The order book is not a static snapshot. It changes with every new event: an order is submitted, an order is canceled, two orders match and create a trade, an order expires because its time-in-force runs out. Every one of those events is a mutation to the book's state.

The purpose of the order book is to support price discovery, which means allowing buyers and sellers to find each other and agree on a price. Before electronic order books, this happened in physical trading pits where traders shouted prices at each other. Electronic books formalized the process into a deterministic rule system. The rules are explicit: orders have defined types, they follow defined priority, and the matching engine applies those rules consistently to every order.

An order book is therefore two things at once. It is a data structure that stores resting orders. It is also a rule engine that governs what happens when new orders arrive, how trades are generated, how orders are removed, and how the state of the book is communicated to the outside world.

When you understand order books well, you understand the mechanical foundation on which all financial trading is built. Every equity trade on a major exchange, every futures contract executed on a derivatives market, every cryptocurrency transaction on a centralized exchange flows through logic that is some version of what this document describes.

---

## The Structure Of The Book

Every order book has two sides: the bid side and the ask side.

The bid side holds buy orders. Every order on the bid side represents a participant who wants to purchase some quantity of the instrument. The bid side is sorted from the highest price at the top to the lowest price at the bottom. The ordering reflects aggressiveness: a buyer offering more money is closer to being willing to trade. The highest bid is therefore the most competitive buy offer and sits at the top of the bid side. If you are a seller looking at the bid side, you would prefer to sell to the buyer offering the most money, which is why the highest bid gets priority.

The ask side holds sell orders. Every order on the ask side represents a participant who wants to sell some quantity of the instrument. The ask side is sorted from the lowest price at the top to the highest price at the bottom. A seller willing to sell for less is more competitive from a buyer's perspective. The lowest ask sits at the top of the ask side and is the first sell offer a buyer will encounter. If you are a buyer looking at the ask side, you want to pay as little as possible, so the lowest ask gets priority.

These two facts — bids sorted descending, asks sorted ascending — mean the two sides converge toward the middle. The best bid and best ask are the prices closest to each other. If they ever touch or cross, a trade can happen.

A concrete example with actual numbers helps:

```text
Bid side                    Ask side

Price     Quantity           Price     Quantity
50.05     800                50.07     1,200
50.04     500                50.08     600
50.03     1,100              50.09     400
50.02     300                50.10     2,000
50.01     900                50.11     750
```

The best bid in this book is `50.05`. The best ask is `50.07`. There is a gap of `0.02` between them. No trade can happen at this moment because no buyer is willing to pay `50.07` and no seller is willing to accept `50.05`. The market is in equilibrium. The moment a buyer submits an order at `50.07` or a seller submits an order at `50.05`, a trade happens.

---

## The Best Bid, Best Ask, And Spread

The best bid is the single highest price that any buyer in the book is currently willing to pay. It is also called the top of book bid or simply the bid. The best ask is the single lowest price that any seller in the book is currently willing to accept. It is also called the top of book offer or simply the offer.

Together, the best bid and best ask are called the `top of book` or the `inside market`. This is the most important information in the book for most participants because it tells you immediately: what is the cheapest price you can buy at right now, and what is the best price you can sell at right now.

The spread is the arithmetic difference between the best ask and the best bid:

```text
spread = best ask - best bid
```

The spread is always a non-negative number in a properly functioning book. If the best ask were lower than the best bid, that would mean a buyer is willing to pay more than a seller is asking, and a trade should have already happened. A non-zero spread means the book has not yet found a willing cross.

The spread has enormous practical significance. It is the immediate cost of executing a round trip: buying at the ask and immediately selling at the bid. If the spread is `0.02`, every participant who buys immediately at the ask and sells immediately at the bid loses `0.02` per unit regardless of any directional movement.

Market makers, who are participants that continuously post both bids and asks, earn the spread over time by buying at the bid and selling at the ask repeatedly. Their risk is that the price moves against their resting orders before they can trade out. The spread compensates them for that risk and for the service they provide by making the market liquid.

Spreads are determined by supply and demand for liquidity. In a heavily traded instrument with many participants competing to post bids and asks, the spread compresses because competition forces market makers to post tighter prices to get fills. In an illiquid instrument with few participants, the spread widens because there is less competition and the risk of holding positions is higher.

---

## Price Levels And Queues

A price level is the collection of all resting orders on one side of the book at exactly the same price. Price levels are the internal building blocks of the order book.

At a given price, multiple orders may exist. The orders at that price are organized into a queue based on their arrival time, with earlier orders placed ahead of later orders. This queue is the mechanism through which time priority is enforced.

Consider this example of the ask side:

```text
Price     Queue at that price (in arrival order)

50.07     [Order A: 300, Order B: 200, Order C: 700]
50.08     [Order D: 400, Order E: 600]
50.09     [Order F: 1,000]
```

Order A arrived first at `50.07` and is at the front of the queue. Order C arrived last at `50.07` and is at the back. If a buy order comes in that can trade at `50.07`, it fills against Order A first, then Order B, then Order C, in that exact sequence.

This matters because two traders who both post a sell at `50.07` are not equal from the matching engine's perspective. The one who arrived first will fill first. Arriving early is advantageous because early placement earns you a higher probability of execution.

The total displayed quantity at a price level is the sum of all individual order quantities at that level. In the example above, the displayed quantity at `50.07` is `300 + 200 + 700 = 1,200`. Market data consumers see `50.07 x 1,200`. They do not necessarily see the individual order breakdown unless they have access to order-level data.

---

## Liquidity And Depth

Liquidity and depth are related but distinct concepts.

Liquidity is the general capacity of the market to absorb trading without significant price impact. A highly liquid market is one where you can buy or sell large quantities at prices close to the current best bid or ask. A liquid market is also one that recovers quickly after a large trade pushes prices in one direction.

Depth describes specifically how much quantity is available at price levels near the top of the book. A book with deep depth has large quantities posted at prices just above and below the current best ask and bid. A book with shallow depth has only small quantities available near the top, which means even a moderately sized order can consume the visible liquidity and force the price to move.

To understand the difference between a tight spread and actual liquidity, consider these two scenarios.

Scenario one:

```text
Bid side                    Ask side

Price     Quantity           Price     Quantity
99.99     10                 100.00    10
99.98     10                 100.01    10
99.97     10                 100.02    10
```

The spread is `0.01`, which looks excellent. But a buyer who wants `500` units will immediately exhaust the first several levels. They will pay `100.00`, `100.01`, `100.02`, and continue walking up the ask side until their order is filled. Their average execution price will be far above `100.00`. This is a thin market despite the tight spread.

Scenario two:

```text
Bid side                    Ask side

Price     Quantity           Price     Quantity
99.99     10,000             100.00    8,000
99.98     12,000             100.01    9,500
99.97     15,000             100.02    11,000
```

The spread is also `0.01`. But now a buyer who wants `500` units consumes a tiny fraction of the first level at `100.00`. The price barely moves. This is a deep, liquid market.

Depth is visually represented in Level 2 market data and in depth-of-market displays. Traders analyze depth to estimate how much a large order will move the market, whether large buyers or sellers are lurking at specific prices, and whether the book is balanced or skewed to one side.

---

## Core Components Of An Order Book System

Understanding the data structures and components that make up an order book system is important because it explains why certain operations are fast, why certain design choices are made, and what the system is capable of doing.

An `order` is the fundamental unit. Every order is a record with at minimum: a unique order identifier, the instrument symbol, the side (buy or sell), the order type (limit, market, stop, etc.), the quantity, the price (if applicable), the time-in-force, a timestamp of submission, a field for the current remaining quantity, and a status field. The order identifier must be globally unique among all active orders in the system, because it is the key used to look up and modify the order later.

The `bid book` and `ask book` are the two primary sorted data structures. Each side needs to support these operations efficiently: find the best price, iterate through orders at a price in arrival order, remove an order when it is filled or canceled, add a new order to the correct position. In practice, these structures are often implemented as sorted maps or sorted trees where each key is a price and each value is a queue of orders at that price.

A `price level` is the queue of orders at a single price. The queue enforces time priority. When a new order arrives at that price, it is appended to the end. When an order at the front fills, it is removed from the front. If an order in the middle gets canceled, it needs to be removed from its position in the queue.

An `order identity map`, sometimes called the order index or order store, is a hash map from order id to the order's location in the book. This component exists to support fast cancellation. Without it, canceling an order would require searching through price levels to find the order, which would scale poorly. With the identity map, cancellation is a constant-time lookup followed by a removal from the price level queue.

The `matching engine` is the component that processes incoming orders. It implements the logic: receive an order, determine if it can trade with anything currently on the opposite side, execute those trades, handle remaining quantity based on order type, add resting quantity to the book if applicable, update all relevant data structures, and emit events for every state change.

The `trade log` is an append-only record of every completed execution. Each trade record contains the price, the quantity, the timestamp, the aggressor order id (the incoming order), the passive order id (the resting order), and the side of the aggressor. The trade log is essential for settlement, audit, and reporting.

An `event stream` captures every state change in sequence. This is important for external consumers. If a market data subscriber receives book events in order, they can reconstruct the exact state of the book at any point in time by replaying those events forward from a known snapshot.

A `market data publisher` is responsible for broadcasting book state and trade information to external consumers. It translates internal events into market data messages (Level 1 quotes, Level 2 depth updates, trade prints) and sends them to connected subscribers.

A `session manager` handles the trading session lifecycle: pre-market, opening auction, continuous trading, closing auction, post-market. It controls when the matching engine accepts orders, what order types are valid in each phase, and what happens to resting orders at session boundaries.

---

## The Difference Between Orders And Trades

Orders and trades are fundamentally different things, and confusing them leads to misunderstanding how books work.

An order is an instruction. It is a message from a participant to the venue saying, "Here is what I want to do." Submitting an order does not guarantee anything happens. The order may rest in the book for a long time without trading. It may be rejected before it enters the book. It may partially fill and then wait. It may expire without ever trading. The order is an expression of intent.

A trade is an execution. A trade means two participants have been matched, quantities have been exchanged, and the transaction is complete. A trade happens only when an incoming order matches against a resting order on the opposite side.

Every trade requires two orders: the passive order that was already resting in the book, and the active order that just arrived and caused the match. The passive order is often called the maker. The active order is often called the taker. The trade records both.

This distinction becomes important in many contexts. In market data, an order book update (a bid or ask changing) is not a trade. A trade is a separate message entirely. A strategy that tracks book changes and trade prints is tracking two different event types that together describe what is happening in the market.

---

## Limit Orders

A limit order is an order to buy or sell a specific quantity at a specific price or better.

The word "better" has different meaning for each side. For a buy limit order, better means a lower price. The buyer sets a maximum and is happy to pay anything less. For a sell limit order, better means a higher price. The seller sets a minimum and is happy to receive anything more.

A buy limit order at `50.00` instructs the system: buy up to this quantity, but do not pay more than `50.00`. If the best ask is `49.95`, the order executes at `49.95` because that is a better price for the buyer than the limit. If the best ask is `50.05`, the order cannot execute immediately and will rest in the book.

A sell limit order at `50.00` instructs the system: sell up to this quantity, but do not accept less than `50.00`. If the best bid is `50.10`, the order executes at `50.10` because that is a better price for the seller. If the best bid is `49.90`, the order cannot execute immediately and will rest in the book.

Limit orders can be partially marketable. A buy limit order at `50.05` against a book where the best ask is `50.03` is marketable — it can trade. But maybe only `100` units are available at prices at or below `50.05`. If the order is for `500`, the first `100` execute immediately. The remaining `400` go to the bid side at `50.05` unless the order type prevents resting.

The marketability check is the gate. A buy limit order is marketable if the limit price is greater than or equal to the best ask. A sell limit order is marketable if the limit price is less than or equal to the best bid. If the order is not marketable, it cannot trade immediately and either rests or is discarded.

Limit orders are the building blocks of the order book. They are what create the bid and ask sides in the first place. Without people willing to post limit orders and wait for execution, there would be no book, no visible prices, and no liquidity for others to trade against.

---

## Market Orders

A market order is an instruction to buy or sell a specific quantity at whatever prices are currently available on the opposite side of the book. A market order does not specify a limit price. The trader is saying: fill my quantity immediately, and I accept whatever price the market gives me.

A market buy order executes against the ask side, starting at the lowest ask price and consuming quantity upward through price levels until the order is complete or liquidity runs out.

A market sell order executes against the bid side, starting at the highest bid price and consuming quantity downward through price levels.

Example ask side before a market buy:

```text
Price     Quantity
10.00     50
10.01     50
10.03     100
10.08     200
```

Market buy order: `120`

Execution sequence:
- Buy 50 at `10.00`, first level consumed
- Buy 50 at `10.01`, second level consumed
- Buy 20 at `10.03`, partial third level consumed

The order is filled. The remaining `80` at `10.03` and all of `10.08` are untouched.

The volume-weighted average price of this execution is:

```text
VWAP = (50 × 10.00 + 50 × 10.01 + 20 × 10.03) / 120
     = (500.00 + 500.50 + 200.60) / 120
     = 1201.10 / 120
     = 10.009
```

The trader received an average price of `10.009`, slightly above the original best ask of `10.00` because they had to walk up the book to fill their full quantity.

Slippage is the difference between the price that was at the top of the book when the order was submitted and the average price actually received. In this case, slippage is `10.009 - 10.00 = 0.009` per unit. Slippage is a cost that grows with order size, because larger orders consume more levels and push further into the book.

Market orders are risky in illiquid markets. If the ask side is thin and a large market buy arrives, it can consume every level on the ask side and execute at absurdly high prices. In some venues, market orders are protected by price bands or circuit breakers that prevent execution beyond a certain price range.

In some continuous order books, market orders never rest in the book. If a market buy exhausts all available asks, any remaining quantity is canceled. This is a deliberate design choice. Allowing market orders to rest in the book at an undefined price creates ambiguity.

---

## Stop Orders And Stop-Limit Orders

A stop order is a conditional order that becomes active only after the market price crosses a specified trigger price.

A stop order itself does nothing while the market is above or below the trigger. It sits in a separate conditional order system, not in the main book. Once the trigger is breached, the stop order converts into an active order and is submitted to the matching engine.

A stop market sell order works as follows: the trader specifies a trigger price below the current market. If the market trades at or below that trigger, the stop converts into a market sell order and executes at whatever bids are available. This is commonly used to protect a long position. If the trader owns shares and wants to exit if the price falls significantly, a stop sell limits the downside.

Example:

```text
Current price: 100.00
Stop market sell 500, trigger 95.00
```

If the market trades at `94.95`, the trigger fires. The system submits a market sell for `500`. Execution happens immediately against the current bids at whatever prices are available.

A stop-limit order is a refined version. It has two prices: the trigger and the limit.

```text
Stop-limit sell 500
Trigger: 95.00
Limit:   94.50
```

When the trigger fires at or below `95.00`, the system submits a sell limit order at `94.50`, not a market order. This gives price protection. The order will not execute below `94.50`. However, if the market gaps past `94.50` in a fast move, the order may never fill at all. The trader trades execution certainty for price control.

A trailing stop is a variation where the trigger price is defined as an offset from the running high or low rather than an absolute price. For example, a trailing stop sell with a `2.00` trail fires when the price falls `2.00` below the highest price reached since the order was entered. This lets profits run while still providing protection on the downside.

Stop orders are not visible in the main order book before they trigger. They are invisible to other participants. This is intentional: if large stop orders were visible, other traders might try to push the price toward them to trigger them and benefit from the resulting liquidity.

---

## Iceberg Orders And Hidden Orders

An iceberg order is a limit order where only a small portion of the total quantity is displayed in the book at any one time. The visible portion is called the peak or display quantity. When the peak is filled, a new peak of the same size is automatically refreshed from the hidden reserve. This continues until the entire order is filled or the order is canceled.

Example:

```text
Iceberg sell: total 10,000, display 200
```

The book shows `200` at the iceberg's price. After that `200` trades, the book shows another `200`. The full `10,000` is never revealed. This is used by large traders who want to avoid revealing the full size of their position because a visible large order can move the market against them.

In most implementations, the refreshed display portion of an iceberg gets a new timestamp when it appears, so it goes to the back of the queue at that price level. This means iceberg orders trade at worse time priority compared to a regular limit order of the same total size, but they hide the total size in exchange.

A hidden order or reserve order goes further: the entire quantity is hidden from the book. No displayed size at all. These orders participate in matching but do not contribute to visible depth. In price-time priority books, hidden orders typically lose time priority to displayed orders at the same price because the display of liquidity is considered a service to the market.

---

## Price-Time Priority

Price-time priority, also called FIFO priority (First In First Out), is the most widely used matching rule in central limit order books worldwide.

The rule has two tiers. Price is the first tier. Time is the second tier and applies only when prices are equal.

In the price tier, for the ask side, a lower price always wins. A sell limit order at `50.00` is filled before a sell limit order at `50.01`, regardless of when either arrived. From a buyer's perspective, the cheaper seller is better, so the cheaper seller gets to trade first.

For the bid side, a higher price always wins. A buy limit order at `50.05` is filled before a buy limit order at `50.04`, regardless of when either arrived. From a seller's perspective, the buyer offering more is better.

In the time tier, when two orders are at the same price, the earlier order wins. If two sell orders are both at `50.00`, the one that arrived first is at the front of the queue for that price level. It fills before the later order.

Walk through a full example:

```text
State of the ask book before a buy order arrives:

Price     Order     Quantity     Arrival
50.00     A         300          08:01:00.000
50.00     B         500          08:01:00.050
50.00     C         200          08:01:00.120
50.01     D         400          08:00:59.000
50.01     E         600          08:01:01.000
```

Notice that Order D arrived at `08:00:59.000` before Orders A, B, and C. But D is at `50.01`, which is a worse ask price than `50.00`. Price priority puts D behind A, B, and C despite D's earlier arrival. Time priority never overrides price priority.

Now a buy order arrives:

```text
Buy 900 at 50.01
```

This order's limit is `50.01`, so it can trade against anything at `50.01` or below.

Matching proceeds:

- Fill 300 against A at `50.00` — A is fully filled, removed from book
- Fill 500 against B at `50.00` — B is fully filled, removed from book
- Fill 100 against C at `50.00` — C has 200, incoming order has 900, 800 filled so far, take 100 from C. C now has 100 remaining.

Incoming order is now fully filled (300 + 500 + 100 = 900). Matching stops.

Result: A and B are gone. C has 100 remaining at `50.00`. D and E are untouched.

The incoming buy order did not choose which orders to trade with. The book's priority rules made every decision. The buyer simply said "buy 900 at most 50.01" and the engine applied the rules.

---

## Why Price-Time Priority Exists And Its Implications

Price-time priority is not arbitrary. It was designed to create specific behavior and incentives.

Price priority exists because it channels competition in the right direction. Participants who want to trade have an incentive to offer the best price possible. A buyer who wants to be filled quickly bids higher. A seller who wants to be filled quickly asks lower. This competition narrows the spread over time, which benefits everyone by reducing transaction costs.

Time priority exists within a price level to reward early commitment. A participant who was willing to post a limit order and wait takes on risk. They do not know when or if they will fill. In exchange for that commitment, they are rewarded with priority over those who arrive later at the same price.

These combined incentives produce a market with tight spreads, deep liquidity, and fair rules. If time priority did not exist, participants at the same price would have no incentive to post early. If price priority did not exist, the market would not converge to competitive prices.

One important implication of time priority is queue position risk. At a busy price level, late arrivals may have many orders ahead of them in the queue. If the market moves away before their order fills, they lose their turn without trading. This creates a dilemma for passive traders: join the existing price level and wait in queue, or improve the price and jump ahead. This tension drives constant strategic decisions by market participants.

Another implication is that modifications to an order can affect priority. If a trader cancels an order and resubmits it at the same price, the resubmitted order goes to the back of the queue. The old position is lost.

---

## Alternative Priority Rules

Price-time priority is not the only matching algorithm used in financial markets. Different asset classes and trading products have evolved different approaches.

Pro-rata allocation is the primary alternative. Under pro-rata, when an incoming order matches at a price level, the fill is distributed among all resting orders at that price proportionally to their size. A resting order with 1,000 units receives a larger portion of the fill than a resting order with 100 units, regardless of arrival time.

Example:

```text
Resting asks at 50.00:
Order A: 500  (arrived first)
Order B: 1,500

Incoming buy: 400 at 50.00

Pure pro-rata allocation:
A's share: 500 / 2000 = 25%   →  fill 100
B's share: 1500 / 2000 = 75%  →  fill 300
```

Pro-rata is common in fixed-income futures markets such as the Eurodollar contract on the CME. It rewards size rather than speed. Participants with large orders receive larger allocations. This tends to attract large institutional participants who can post big orders.

The downside of pro-rata is that it creates incentives to post artificially large orders and then cancel most of them. A participant can post a large order to claim a large pro-rata share, then reduce the order after receiving a fill. This behavior is called quote stuffing and is controversial.

Hybrid priority systems combine price-time and pro-rata. A common hybrid: the first-arriving order at a price receives a guaranteed time-priority allocation (say, 40% of the fill), and the rest is distributed pro-rata among all orders at that price. This rewards both early commitment and size.

Some markets use price-broker-time priority, which is like price-time but with a middle tier for certain privileged participants (such as designated market makers) who receive priority within a price level ahead of regular participants.

Lead Market Maker priority is another variant. Designated market makers at a venue may receive priority fills in exchange for obligations to quote tight markets continuously.

The choice of priority algorithm profoundly shapes who participates in a market, what strategies are profitable, and what the book looks like. Price-time markets tend to have very fast participants competing intensely on latency. Pro-rata markets tend to have participants competing on size.

---

## Maker And Taker

The terms maker and taker describe the two roles in every trade.

The maker is the resting order. It was already in the book when the trade happened. The maker provided liquidity to the market by posting a quote and waiting for someone to trade against it.

The taker is the incoming order. It arrived, matched against the resting order, and removed that liquidity from the book.

These roles determine fees at most exchanges. The maker-taker pricing model charges the taker a fee and pays the maker a rebate. The reasoning is that makers provide a service by making the market more liquid, while takers consume that service. Charging takers and rebating makers incentivizes participants to post resting orders, which tightens spreads and deepens the book.

Example fee structure:

```text
Maker rebate:  -0.002 (the maker receives $0.002 per share)
Taker fee:      0.003 (the taker pays $0.003 per share)
```

In this structure, the exchange nets `0.001` per share per trade. The maker nets a rebate.

Some venues use an inverted structure: they charge the maker and rebate the taker. This is designed to attract more aggressive order flow to the exchange.

Understanding maker-taker is important for strategy analysis. A strategy that always takes liquidity (market orders, IOC limit orders that cross the spread) pays taker fees on every trade. A strategy that provides liquidity earns maker rebates. The net cost of a strategy depends heavily on whether it is predominantly a maker or a taker.

One subtlety: a single limit order can be a maker for part of its execution and a taker for another part. If a limit order is marketable — meaning its price crosses the spread — it takes liquidity immediately against resting orders. If it is not fully filled and the remainder rests in the book, that remainder later acts as a maker when someone else trades against it. The two portions of the order are assessed different fees.

---

## Partial Fills

A partial fill occurs when only part of an order's quantity executes in a given matching cycle. The rest either rests in the book, is canceled, or is treated according to the order's time-in-force instructions.

Partial fills are not exceptional. They are the normal case for any order that is larger than the available resting quantity at the acceptable price or prices.

Example:

```text
Ask side at the moment a buy order arrives:
Price     Quantity
50.00     100
50.01     150

Incoming: Buy 400 at 50.02
```

The buy order can trade against both price levels because both are below its limit of `50.02`.

Execution:
- Fill 100 at `50.00`
- Fill 150 at `50.01`
- Total filled: 250. Remaining: 150.

The ask side is now empty up to and including `50.02`. The remaining `150` of the buy order now has nowhere to trade. Because it is a standard limit order, the remaining `150` becomes a resting bid at `50.02`.

The order's status is now "partially filled". It has traded `250` and has `150` resting.

The consequences of partial fills depend on what happens to remaining quantity:

If the order type is a regular limit order, remaining quantity rests in the book and waits for future orders.

If the order type is IOC, remaining quantity is immediately canceled after the matching attempt.

If the order type is FOK, a partial fill scenario should not arise because FOK orders are checked for full fillability before any execution happens. If they cannot fill completely, the entire order is rejected with no executions at all.

---

## Multi-Level Fills And Price Impact

An order that is large enough to consume all available quantity at one price level will continue matching at the next price level, and the next, until it is filled or its limit price is exhausted.

This behavior is called walking the book. It is the mechanism by which a large order has market impact: prices must move to accommodate the full quantity.

Example ask book before a large buy order:

```text
Price     Quantity
50.00     300
50.01     200
50.02     400
50.03     500
50.05     1,000
```

Incoming order: Buy 1,200 at 50.04

This buy order will trade at any price up to `50.04`. It cannot trade at `50.05` because that exceeds its limit.

Execution:
- Fill 300 at `50.00`
- Fill 200 at `50.01`
- Fill 400 at `50.02`
- Fill 300 at `50.03` (partial fill of the 500 at 50.03)
- Total: 1,200 filled. Matching stops.

Remaining at `50.03`: 200 left over from the original 500.

The order was completely filled across four price levels. The best ask was `50.00` before the order arrived. After matching, the best ask is `50.03` (with 200 remaining). The large buy order moved the best ask up by three ticks. This is price impact.

The volume-weighted average price of the execution:

```text
VWAP = (300 × 50.00 + 200 × 50.01 + 400 × 50.02 + 300 × 50.03) / 1,200
     = (15,000 + 10,002 + 20,008 + 15,009) / 1,200
     = 60,019 / 1,200
     = 50.016
```

The trader's average execution price was `50.016`, which is `0.016` above the best ask before the order was submitted. This is the price impact cost.

Price impact grows with order size relative to available liquidity. This is why large institutional traders often break orders into smaller pieces and execute them over time, rather than submitting one large order that walks the book.

---

## Time-In-Force: A Full Taxonomy

Time-in-force is the instruction that tells the matching engine how long an order is allowed to remain active. It is distinct from order type. Order type defines matching behavior. Time-in-force defines lifespan. Every active order combines both: a type that defines what it does and a time-in-force that defines how long it does it.

Understanding time-in-force is essential because it directly determines whether an unexecuted order persists across sessions, expires within a session, or is discarded the moment it cannot fill immediately.

### Day Order

A Day order is active only for the current trading session. If any portion of a Day order is not filled by the time the trading session closes, the remaining quantity is automatically canceled by the system.

Day is usually the default time-in-force for most order entry systems. When a trader places a limit order without specifying anything further, it is typically treated as a Day order.

The exact definition of a "day" can vary. On most equity exchanges, the regular trading session runs from market open to market close (9:30 AM to 4:00 PM Eastern Time on US exchanges, for example). A Day order placed at 10:00 AM that has not fully filled by 4:00 PM is automatically canceled. On exchanges that also support pre-market and post-market sessions, a Day order may or may not be active during those extended sessions, depending on the venue's rules.

Day orders are appropriate when a trader wants to participate in today's trading activity but does not want to leave a stale order in the book tomorrow. Circumstances change between sessions: news is released overnight, prices gap open, the trader's own position may change. A Day order prevents the scenario where a trader wakes up to a fill they no longer wanted based on conditions from the prior day.

### Good 'Til Canceled

Good 'Til Canceled, abbreviated GTC, is an order that persists across trading sessions indefinitely until it is either fully filled or explicitly canceled by the trader.

A GTC order placed on a Monday afternoon survives through the close, carries over into Tuesday, Wednesday, and subsequent sessions, and remains eligible to fill every day the market is open. There is no automatic expiration based on time.

This is useful when a trader has a specific target price in mind and is willing to wait however long it takes for the market to come to that price. A buyer who believes a stock will eventually fall to `45.00` can place a GTC buy limit at `45.00` and leave it. If the price reaches `45.00` next week, next month, or next quarter, the order may fill.

Despite the name, "Good 'Til Canceled" does not necessarily mean the order lives forever in all real-world implementations. Many brokers and exchanges impose a maximum lifetime on GTC orders, typically 30, 60, 90, or 180 days, after which the order is automatically canceled even if it has not filled. The broker's or exchange's specific rules determine the ceiling.

In systems design, handling GTC orders correctly at session boundaries requires the order to persist in storage across sessions, be reloaded at market open, and be properly tracked in the order identity map throughout its life. GTC orders can remain in the system for a very long time, which creates data management considerations.

### Good 'Til Date Or Time

Good 'Til Date (GTD), also called Good 'Til Time (GTT), is an order that persists until a specific timestamp provided by the trader. After that timestamp, any remaining quantity is automatically canceled.

Example:

```text
Sell 500 at 75.00, GTD 2026-05-15 12:00:00
```

This order remains active across sessions as needed until either it fills completely or the clock reaches `2026-05-15 12:00:00`. At that moment, if there is any remaining unfilled quantity, the system automatically expires the order.

GTD is useful for situations where a trade makes sense only within a specific time window. A trader might want an order active until the end of the week to capture any favorable movement, but not beyond that because an important market event (such as a central bank announcement) happens on the following Monday that could change the picture entirely.

GTD partial fills work the same way as any other partial fill. If `300` of the `500` fill before the expiration time, the trade record shows `300` executed. When the expiration timestamp arrives, only the remaining `200` are canceled. The already-executed `300` are not affected. Expiration never undoes a trade.

### Immediate Or Cancel

Immediate Or Cancel, abbreviated IOC, is a time-in-force that says: execute as much as possible right now, and immediately cancel any remaining unfilled quantity. The order makes no attempt to rest in the book.

IOC allows partial fills. If only part of the order can be filled immediately, that portion executes and the rest disappears.

Example:

```text
Ask book:
Price     Quantity
50.00     80
50.01     200

Incoming: Buy 300 at 50.00, IOC
```

The IOC order can trade at `50.00` only (because its limit is `50.00`). There are `80` units available. Fill `80`, then check: is there more at `50.00` or below? No. Cancel the remaining `220`.

If the limit had been `50.01`:

```text
Incoming: Buy 300 at 50.01, IOC
```

Then the order fills `80` at `50.00` and `200` at `50.01` for a total of `280`. The remaining `20` is canceled.

IOC is used when a trader wants to execute against available liquidity immediately but does not want to leave a resting order in the book. This is common in algorithmic trading where strategies do not want their intentions to be visible via a resting order, or where the strategy's logic will handle any unfilled quantity separately.

### Fill Or Kill

Fill Or Kill, abbreviated FOK, is a time-in-force that says: the entire order must execute immediately and completely, or the entire order is canceled with zero execution.

FOK is stricter than IOC. With IOC, partial fills are allowed. With FOK, the complete quantity must be available immediately, or no trade happens at all.

Before any execution occurs for an FOK order, the matching engine must determine whether the full quantity can be filled. If the available liquidity (at acceptable prices) is less than the requested quantity, the order is rejected in full.

Example:

```text
Ask book:
Price     Quantity
50.00     300
50.01     200

Incoming: Buy 600 at 50.01, FOK
```

The engine checks: can `600` be filled at `50.01` or below?

Available at `50.00`: 300. Available at `50.01`: 200. Total available at acceptable prices: 500. The order requires 600. The answer is no. The FOK order is canceled with zero execution.

Now with a larger book:

```text
Ask book:
Price     Quantity
50.00     300
50.01     400

Incoming: Buy 600 at 50.01, FOK
```

Available at `50.00`: 300. Available at `50.01`: 400. Total: 700. The order requires 600. The answer is yes. Execute: fill 300 at `50.00`, fill 300 at `50.01`. Done.

FOK is used when partial fills are operationally unacceptable. For example, a trader who needs to acquire exactly `600` shares as part of a larger strategy execution (perhaps to delta-hedge an options position precisely) cannot tolerate receiving `400`. It is all or nothing.

### All Or None

All Or None, abbreviated AON, is a time-in-force that requires the full quantity to fill, but unlike FOK, does not require it to happen immediately. An AON order may rest in the book and wait until enough liquidity is available to fill it completely.

Example:

```text
Buy 1,000 at 50.00, AON
```

This order will not partially fill `200` or `500`. It will only execute when there is enough available sell quantity at `50.00` or better to fill all `1,000` simultaneously.

AON orders are rare in modern electronic price-time priority systems because they create significant complexity in the matching algorithm. The matching engine must skip the AON order when there is insufficient liquidity and then come back to it when more liquidity arrives. This violates the simple queue structure of a FIFO book.

Additionally, AON orders create invisible conditional liquidity: the book may show `1,000` of bid interest at `50.00`, but sellers cannot actually trade small quantities against it. This can confuse market participants who see large bids or asks but find they cannot trade small amounts against them.

### At The Open

An At The Open order (sometimes written MOO for Market On Open or LOO for Limit On Open) is designed specifically for participation in the opening auction. It is only valid for that auction. If the order cannot be executed during the opening process, it is automatically canceled.

Opening auctions collect orders during a pre-market phase and then execute them at a single clearing price determined by an auction algorithm. At The Open orders participate in this process.

A Market On Open order accepts the opening auction clearing price, whatever it turns out to be. A Limit On Open order participates only if the clearing price is within the specified limit.

### At The Close

An At The Close order (sometimes written MOC for Market On Close or LOC for Limit On Close) is designed for participation in the closing auction. Like At The Open, it is valid only for the closing process and is canceled if it cannot execute.

Closing auctions are important because the closing price is a widely referenced benchmark. Index calculations, fund NAVs, options expirations, and many other financial instruments are priced off the official closing price. Participants who need to trade at or close to the official closing price use At The Close orders to participate in the closing auction.

---

## Order Lifecycle

Every order passes through a sequence of states from the moment it is submitted to the moment it is no longer active.

The first state is submission. The client, trader, or automated system sends an order message to the broker, gateway, or exchange. At this point, the order exists only in the submission queue.

The second state is validation. The system checks the order against a set of rules. These rules include: is the symbol valid and currently tradable? Is the side valid (buy or sell)? Is the quantity positive and within allowed limits? Is the price within tick size rules and reasonable price bands? Is the order type supported? Does the participant have the permissions required? Does the order pass pre-trade risk checks (position limits, notional value limits, fat-finger checks)?

If validation fails, the order is rejected. A rejected order generates a rejection message back to the submitter explaining the reason. A rejected order never touches the book.

If validation passes, the order is accepted. An accepted order transitions into the active state.

In the active state, the matching engine processes the order. For marketable orders, matching begins immediately. For non-marketable orders, the order is placed into the book.

During matching, the order may fill. A fill event is generated for each trade, containing the execution price, quantity, and the counterparty order id. If the entire quantity fills in one trade, the order is fully filled and complete. If part fills and part remains, the order is partially filled.

Partially filled orders in the book remain active in a partially-filled state. More fills can happen later as additional opposite-side orders arrive.

An active resting order can be canceled at any time. The cancellation request is identified by the order id. The matching engine finds the order, removes it from the book, and sends a cancellation acknowledgment. Cancellation applies only to the remaining quantity. Fills that already happened are unaffected.

An active resting order can also be replaced. A replacement is usually implemented as a cancel-and-resubmit internally, though the client sends it as a single modify instruction. The rules about what changes and what priority consequences follow are described in the Replacements section below.

An order may expire. Expiration happens automatically based on time-in-force. A Day order expires at session close. A GTD order expires at its specified timestamp. An IOC order effectively expires the moment its matching cycle ends without a full fill. Expired orders are removed from the book without any trades.

The terminal states are fully filled, canceled, rejected, and expired. Once an order reaches a terminal state, it cannot trade, be modified, or be canceled again.

Tracking order states correctly is critical for both the exchange system and for clients. A mismatch between what the exchange thinks the order state is and what the client thinks it is leads to errors: double-fills, missed fills, accidental open positions.

---

## Cancellations In Depth

A cancellation is a request to remove a resting order's remaining quantity from the book.

Cancellations are common. In many high-frequency trading environments, the ratio of cancellations to fills is extremely high. Participants may post and cancel thousands of orders for every trade that actually executes. They do this to adjust their quoted prices as market conditions change.

For a cancellation to succeed, the order must be active and identified by its order id. The matching engine receives the cancellation request, looks up the order in the order identity map, locates the order's price level, removes the order from its queue position within that level, and updates the book.

If removing the order leaves a price level empty (no more orders at that price on that side), the price level itself is removed. This keeps the book clean and ensures that the best bid or ask is always a price level with actual resting quantity.

The matching engine must handle the case where a cancellation arrives at the same moment as a matching event that would fill the order. These are called late cancels or cancels-after-fill. The system must process events in strict sequence and determine: did the fill happen first or did the cancel arrive first? Whatever happened first takes precedence. If the fill happened first, the cancel is rejected or acknowledged with zero remaining quantity to cancel.

Cancellations never generate trade events. They only update the book's visible depth.

One subtle point: a cancellation can change the top of book without any trade occurring. If the only bid at the best bid price is canceled, the best bid drops to the next available price level. External market data consumers see a book update that looks like a price move, but no trade happened. Confusing cancellations with trades is a common error in market data analysis.

---

## Replacements And Modifications

A replacement or modification changes the parameters of an existing resting order. It is typically implemented as an atomic cancel-and-resubmit: the old order is removed from the book and a new order is created with the updated parameters.

Common modifications include changing the price, changing the quantity, or changing the time-in-force.

Changing the price almost always results in the loss of time priority. When an order moves to a new price level, it arrives at that level as a new entry and takes the last position in that level's queue. It does not carry over its old position. This is correct behavior: the order is now competing at a different price, and other orders already at that price arrived earlier.

Changing quantity is more nuanced. Reducing quantity is generally considered non-aggressive: the trader is asking for less, not making a more aggressive bid or offer. Many venues allow quantity reductions to preserve time priority. The order stays in the same queue position, just with a smaller size. Increasing quantity is considered aggressive and usually causes the order to lose priority, because the trader is now committing more capital and the size increase is effectively a new, larger order.

Some venues implement a free-to-reduce policy: any quantity reduction is permitted without priority loss up to but not including a cancel-and-resubmit. If the quantity goes to zero, that is effectively a cancellation, not a reduction.

The practical consequence of priority loss on modification is significant. A trader who has been patiently waiting at the front of a queue at a price level will not cancel-and-resubmit lightly, because doing so sends them to the back. This is why modifications that change price typically cause meaningful queue position degradation.

---

## Matching Flow: Buy Order In Full Detail

When a buy order arrives at the matching engine, the following sequence occurs:

The engine first determines whether the order can match at all. For a limit buy order, the order is marketable if the limit price is greater than or equal to the current best ask. For a market buy order, any available ask is acceptable.

If the order is not marketable (the limit price is below the best ask), no matching occurs. The order is placed into the bid side at its price, in the correct queue position at that price level. The book is updated and the event is published.

If the order is marketable, matching begins. The engine accesses the lowest ask price. This is the best ask. The engine takes the first order in that price level's queue. This is the oldest order at the best ask price.

The engine computes the trade quantity: the minimum of the incoming order's remaining quantity and the resting order's remaining quantity.

A trade is generated with the trade price (the resting order's price), the trade quantity, the incoming order as aggressor, and the resting order as passive.

Both orders' remaining quantities are reduced by the trade quantity.

If the resting order's remaining quantity reaches zero, it is fully filled. It is removed from the price level queue. If the price level is now empty, the price level is removed from the ask side. The best ask advances to the next price level.

If the incoming order's remaining quantity reaches zero, it is fully filled. Matching stops.

If neither has reached zero after the trade, and the incoming order's limit price still allows trading at the current best ask, matching continues with the next order in the queue at the same price level.

This loop continues until one of the following conditions ends matching: the incoming order is fully filled, no more asks exist at acceptable prices, or the order type prevents further matching (for example, an IOC order stops after the initial matching cycle even if more opportunities exist).

After the matching loop ends, the outcome for the incoming order's remaining quantity is determined by its order type and time-in-force. If it is a resting-eligible order (standard limit, GTC limit, GTD limit, Day limit), the remaining quantity is placed on the bid side at the limit price. If it is IOC or FOK (where FOK was already evaluated before this point), the remaining quantity is canceled.

---

## Matching Flow: Sell Order In Full Detail

The sell order matching flow mirrors the buy order flow exactly, with directions reversed.

A sell order matches against the bid side. The engine checks whether the sell order's limit price is less than or equal to the best bid. For market sell orders, any available bid is acceptable.

If not marketable, the order rests on the ask side at its price.

If marketable, the engine accesses the highest bid. This is the best bid. The engine takes the first order in that price level's queue (the oldest bid at the best bid price).

The trade quantity is the minimum of the incoming sell order's remaining quantity and the resting bid's remaining quantity. A trade is generated at the resting bid's price.

Quantities are updated. Fully filled resting bids are removed. Empty price levels are removed from the bid side.

The loop continues as long as the sell order has remaining quantity and acceptable bids exist.

After matching, remaining sell quantity rests on the ask side or is canceled depending on order type.

---

## Execution Price

Execution price is the price at which a trade occurs. In a continuous book, the convention in most markets is that the execution price equals the resting order's price, not the incoming order's price.

This means: the price that was already in the book before the trade wins.

Example:

```text
Resting ask: Sell 100 at 50.00
Incoming buy: Buy 100 at 50.05
```

The buyer's limit is `50.05`. The seller's price is `50.00`. The trade executes at `50.00`. The buyer gets price improvement: they were willing to pay up to `50.05` but only paid `50.00`. The seller executes at their stated price of `50.00`.

This convention makes sense. The resting seller posted a specific price and committed to it. The incoming buyer arrived later and is the aggressor. The resting order's terms stand.

The buyer benefits from price improvement whenever their limit price exceeds the available resting ask prices. A marketable buy limit order at `50.10` against asks of `50.00`, `50.02`, and `50.04` executes at those prices, not at `50.10`. The buyer pays the actual available prices, which are all better than their limit.

In auctions, the pricing rule is different. All participants in a clearing auction execute at the single clearing price, regardless of what price they specified in their orders. A buy limit at `50.05` and a buy limit at `50.20` both execute at the same clearing price if both are marketable at that price.

In midpoint execution (such as dark pools that match at the midpoint of the spread), the execution price is the midpoint of the current best bid and best ask. Neither resting order's price wins; a neutral reference price is used.

---

## Crossed And Locked Books

A locked book is when the best bid equals the best ask. No trade has happened yet at that price, but a buy and a sell are quoting the same price.

Example:

```text
Best bid: 50.00
Best ask: 50.00
```

In theory, these two orders should trade. In a single venue's matching engine, they would. But in fragmented markets where the same instrument trades on multiple exchanges simultaneously, a locked book can appear in consolidated market data. The best bid might be from Exchange A and the best ask from Exchange B. Both quoting `50.00`. No individual exchange's matching engine sees both quotes, so no trade occurs at the consolidated level. This represents an arbitrage opportunity.

A crossed book is when the best bid exceeds the best ask:

```text
Best bid: 50.05
Best ask: 50.02
```

This should not exist in a healthy continuous single-venue book. A crossing would have been matched immediately when one of those orders arrived. If it appears, something is wrong: market data is delayed or out of sequence, a trading halt has occurred, or the feed is showing stale data.

In multi-venue consolidated data, crossed markets can appear briefly due to quote changes arriving out of order or simultaneous latency differences. They are quickly arbed away by participants trading between venues.

---

## Tick Size And Price Granularity

Tick size is the minimum allowed increment between two adjacent valid prices.

If the tick size is `0.01`, the valid price grid is `50.00, 50.01, 50.02, ...`. An order submitted at `50.005` would be rejected or rounded to the nearest valid price depending on venue rules.

Tick size has deep effects on market behavior.

A larger tick size forces more orders to queue at the same price because there are fewer distinct prices available. This means time priority becomes more important: since you cannot improve price by a tiny increment, the advantage goes to participants who arrived earlier at the existing price. In a market with a tick size of `0.25`, a participant who wants to be ahead of others at `50.00` cannot bid `50.001`. They must jump to `50.25`. This is a larger price concession and may not be worth making just for priority.

A smaller tick size creates more price levels and allows traders to compete more finely. A trader can improve price by a tiny increment to gain priority. This can reduce average queue lengths at any given price but also increases the total number of price levels the engine must manage.

Tick size also affects the minimum possible spread. A spread cannot be smaller than one tick. In an equity with a tick size of `0.01`, the minimum spread is `0.01`. A very tight spread relative to tick size indicates intense competition.

Different asset classes have different tick size regimes. US equities generally trade at one-cent increments. Some options and futures have larger ticks. Some FX products have sub-pip tick sizes.

---

## Lot Size And Quantity Constraints

Lot size is the minimum quantity unit in which an instrument can be traded. In some markets, orders must be submitted in multiples of the lot size.

In US equities, any quantity of whole shares is generally valid. In some futures markets, contracts come in fixed sizes and you can only trade whole contracts. In OTC bond markets, quantities are often specified in face value with minimum denomination requirements.

If the lot size is `100`, then orders for `100, 200, 300, ...` are valid. An order for `150` would be rejected.

Fractional trading is increasingly available in retail brokerage contexts, where customers can buy fractions of a share. This requires special handling at the broker level because exchanges typically only accept whole shares.

The matching engine must check the quantity against lot size rules during validation. An order that violates lot size requirements is rejected before it touches the book.

Odd lots, which are quantities smaller than the standard lot size, are handled differently on many venues. They may not interact with the main book at all, or they may be routed to alternative execution mechanisms.

---

## Types Of Order Books

### Central Limit Order Book

The central limit order book, or CLOB, is the dominant market structure in electronic trading worldwide. It is central in the sense that all participants send orders to the same place. It is a limit order book because the resting orders that make up its depth are limit orders.

In a CLOB, any participant can post a bid, any participant can post an ask, and any incoming order is matched against the best available resting orders according to the venue's priority rules. There is no single dealer or counterparty. The matching is anonymous (or anonymized post-trade in some systems). The rules are consistent for all participants.

The CLOB's strength is transparency and fairness. Everyone can see the resting orders (in the lit version of the book), and the rules are known in advance. Its weakness is that large orders can move the market by making their intentions visible before execution.

### Level 1 Book

Level 1 book data is the most basic view of the market. It contains the top of book only: best bid price, best bid quantity, best ask price, best ask quantity, last trade price, and last trade quantity.

Level 1 data is sufficient to monitor the current quoted market and see where the last trade happened. It is adequate for most passive investors and for strategies that trade at the current best prices and do not need to model the deeper book.

Level 1 data does not tell you how much liquidity is available beyond the first level, or how large an order would need to be to move the price. Those questions require Level 2 data.

### Level 2 Book

Level 2 book data shows multiple price levels on each side. Instead of just the best bid and ask, you see depth: how much quantity is available at each of the next several price levels.

Level 2 typically shows aggregated quantity at each price. The data shows `50.00 x 500` meaning there are `500` units resting at `50.00` in total, but not necessarily how many individual orders make up that `500`.

Level 2 is useful for understanding market depth, estimating market impact, and seeing where large supply or demand is concentrated. A very large bid at a specific price acts as a support level: it represents a large buyer that would absorb selling. Whether that bid is real and will hold is a separate question.

Level 2 data is available on most major exchanges as a market data subscription. Retail traders often have access to some levels of depth through their brokerages.

### Level 3 Book

Level 3 book data, also called market-by-order or order-by-order data, shows every individual resting order with its price, quantity, and arrival sequence.

Instead of showing `50.00 x 500`, Level 3 shows that the `500` at `50.00` consists of Order A for `200`, Order B for `150`, and Order C for `150`, in that order of priority.

Level 3 data allows you to reconstruct the precise state of the book, calculate queue position for each order, and observe order-level events (additions, cancellations, modifications, fills) as they happen.

This data is valuable for research, strategy development, and understanding microstructure. It is usually more expensive than Level 1 or Level 2 because it is more granular and generates far more data volume. High-frequency trading firms often use Level 3 data or proprietary co-located feeds that are equivalent.

### Lit Book

A lit book is one where orders are displayed transparently to market participants. Bids and asks, with their prices and quantities, are visible before execution. The vast majority of equity exchange books are lit.

Lit books support price discovery. When everyone can see the bids and asks, information is distributed. Participants compete by posting better prices, tightening spreads, and adding depth. The visible book provides a reference point for all trading decisions.

The downside of a lit book is information leakage. If you need to buy `100,000` shares and you post a large visible bid, sophisticated participants can observe your interest and may adjust their behavior. Market makers may widen spreads or reduce depth in anticipation of your order's impact. Other traders may front-run by buying ahead of you and selling to you at a higher price.

### Dark Book And Dark Pools

A dark book or dark pool is a trading venue where orders are not displayed before execution. Participants submit orders, the system matches buyers and sellers, and trades happen, but the resting orders themselves are not visible to anyone.

Dark pools are used primarily by institutional investors who need to trade large quantities without revealing their intentions. If a pension fund needs to sell `5 million` shares of a stock, posting that sell interest visibly on an exchange would immediately push the price down. By routing to a dark pool, the fund can potentially execute without adversely impacting the price.

Dark pools can match at different reference prices: the midpoint of the current lit best bid and ask, the last trade price, or a negotiated price. The terms vary by venue.

The tradeoff of dark execution is uncertainty about fill. Because orders are not visible, you do not know if matching will occur or when. Lit books provide more predictability via visible depth. In dark venues, you submit and wait to see if a match materializes.

Regulatory frameworks in various jurisdictions limit or monitor dark pool trading because excessive dark trading can reduce lit book quality and overall price transparency.

### Continuous Book

A continuous book matches orders in real time throughout the trading session. Orders arrive asynchronously, and each new order is processed as it is received. The matching engine works event by event, in sequence.

Continuous trading is the standard mode for most electronic exchanges during regular trading hours. Every new order submitted is immediately evaluated against the current book. If it can trade, it does so. If it cannot, it rests or is discarded.

The continuous book is deterministic and fair in a time-sequential sense: every event is processed in the order it arrived. Participants compete on the speed and quality of their orders.

### Auction Book

An auction book collects orders over a designated time period and then executes all eligible orders at a single clearing price.

This is fundamentally different from continuous trading. In a continuous book, the first order to cross the spread gets executed immediately. In an auction book, everyone who submitted orders during the collection window participates in a batch process.

The clearing price is usually determined by a price maximization formula: the price at which the largest possible volume can be traded. Sometimes additional tie-breaking rules apply when multiple prices could clear the same volume.

Auctions are used at market open, market close, after trading halts, and sometimes in the middle of the session (intraday auctions). Opening auctions are important because they incorporate all the information from overnight news and establish the opening price. Closing auctions are important because the closing price is used as a reference benchmark for many financial products.

In an auction, participants have time to observe the indicative clearing price as orders are submitted, which can attract more orders and improve price discovery. The final clearing price is determined when the auction ends.

### Pro-Rata Book

A pro-rata book uses a different priority algorithm for fills at the same price. Rather than awarding fills to the oldest order first, pro-rata distributes fills in proportion to each order's size relative to the total resting quantity at that price.

Full example with numbers:

```text
At price 50.00:
Order A: 500   (25% of total)
Order B: 1,000 (50% of total)
Order C: 500   (25% of total)
Total:   2,000

Incoming sell: 400
```

Pure pro-rata allocation:
- A receives 25% of 400 = 100
- B receives 50% of 400 = 200
- C receives 25% of 400 = 100

This contrasts with price-time, where A would receive all `400` (since A arrived first), B and C would receive nothing.

Pro-rata markets tend to attract participants who compete on size rather than speed. A large order receives a proportionally larger allocation. This incentivizes posting large orders, which can provide deep liquidity. But it also incentivizes posting artificially large orders to claim a larger slice of fills, then canceling the excess after execution.

Eurodollar futures on the CME have historically used pro-rata allocation. SOFR futures (the Eurodollar replacement) also use pro-rata.

### Hybrid Book

A hybrid book mixes multiple priority rules. Common combinations include price-time-pro-rata (giving part of the fill to the first-arriving order by time priority and distributing the rest pro-rata), or continuous-and-auction (running continuous sessions during the day with auction opens and closes).

Some hybrid systems also distinguish between displayed and hidden orders in their priority. A displayed order at a price may receive priority over a hidden order at the same price, even if the hidden order arrived earlier.

The CME's Globex matching algorithm for some products uses a hybrid called FIFO with LMM. The LMM (Lead Market Maker) receives a guaranteed allocation of a certain percentage of every trade at their quoted price, and the rest is FIFO. This rewards the LMM for continuously quoting tight markets.

---

## Market Microstructure And Price Discovery

Market microstructure is the study of how trading is organized, how prices are formed, and how participants interact within the mechanics of trading.

Price discovery is the process by which the market aggregates information and arrives at a price. In an order book, price discovery happens through the submissions of bids and asks. Each participant who submits an order is expressing a view about value. An informed buyer who believes the instrument is worth more than its current price will bid above the current best bid. This moves the book and signals to the market that there is someone who values the instrument higher. Sellers adjust their asks in response.

Adverse selection is a key concept. When a market maker posts a bid at `50.00`, they do not know who will sell to them. If the seller is uninformed (just needs cash, rebalancing a portfolio), the market maker buys at `50.00` and is fine. But if the seller is informed (knows something that implies the instrument is actually worth `49.00`), the market maker bought at `50.00` and now holds an asset worth less. This is adverse selection: the market maker was selected against by an informed participant.

Because of adverse selection risk, market makers widen their spreads as uncertainty increases. The wider the spread, the more buffer a market maker has against being adversely selected. In times of high uncertainty (around earnings announcements, economic data releases, geopolitical events), spreads widen because the probability of trading against an informed participant is higher.

The spread can therefore be decomposed conceptually into components: the order processing cost (the cost of running the matching engine and market making infrastructure), the inventory cost (the risk of holding an unbalanced position), and the adverse selection cost (the cost of being on the wrong side of an informed trade).

Queue position in a price-time book determines adverse selection exposure. A market maker who is first in the queue at the best bid will be the first to buy when a large seller arrives. If that large seller is informed, being first in the queue is a disadvantage. Being at the back of the queue means you trade only after the best ask has already been hit multiple times, which may signal something about the trade's nature.

---

## Common Order Book Events And Their Meanings

An order book's state changes through a defined set of event types. Understanding these events is important for reconstructing book state from market data feeds.

An `add` event means a new resting order has entered the book. The book's depth at a specific price increases by the order's quantity.

A `trade` event means an execution occurred. The resting order (passive side) has reduced its quantity or been removed. The book depth decreases.

A `cancel` event means a resting order's remaining quantity was removed by explicit request. Book depth decreases without any trade.

A `reduce` event (sometimes called a partial cancel) means a resting order's quantity was reduced but not to zero. The order remains in the book with a smaller size.

A `replace` event means an order was modified. Depending on the type of modification and the venue, this may change price, quantity, or both.

An `expire` event means an order was removed automatically due to time-in-force. This happens at session close for Day orders, at the specified timestamp for GTD orders, and at the end of the matching cycle for IOC orders.

A `reject` event means an order was not accepted. No book change occurred.

A `snapshot` event provides a complete view of the book at a specific point in time. It is used to initialize a subscriber's local copy of the book when they first connect to a market data feed.

If you receive events in strict sequence and apply them correctly to a snapshot, you can maintain an accurate local copy of the book at all times. This is what every trading system, data vendor, and analytics platform does internally.

A critical operational requirement is sequence number tracking. Every event in a market data feed is typically assigned a sequence number. A subscriber who receives sequence 1001 and then sequence 1003 has missed sequence 1002. The local book state is now incorrect. The subscriber must detect the gap and request a re-snapshot before relying on the data again.

---

## Important Edge Cases

An empty book on one side means no resting bids or no resting asks. A market sell order cannot execute if there are no bids. In most systems, the market order is rejected or held depending on the venue's policy.

An order with quantity zero should be rejected at validation. It is a meaningless instruction.

An order with a price that violates tick size rules should be rejected. For example, if the tick size is `0.01`, a price of `50.003` is invalid.

A cancellation for an order id that does not exist in the active order map should generate a rejection or an error response. It should not silently succeed because the client needs to know their instruction had no effect.

A duplicate order id is invalid. If a new order arrives with an order id that already exists in the active order map, it must be rejected. Each active order must have a unique identifier.

A fully filled order must be removed from the order identity map. If the system keeps fully filled orders in the map, it creates incorrect responses to cancellation requests and consumes memory unnecessarily.

A cancel instruction for a fully filled order should receive a response indicating the order is already completed. The cancel should not appear to succeed.

Race conditions between fills and cancels must be handled with strict event ordering. If a fill and a cancel arrive nearly simultaneously, the system must process them in the order they were received. Whichever arrives first takes effect. If the fill arrives first, the cancel applies to the remaining quantity. If the cancel arrives first, the fill does not happen.

An order that is canceled or expired must never trade after that point. This seems obvious but requires careful implementation: once an order leaves the active book, no matching should reference it.

---