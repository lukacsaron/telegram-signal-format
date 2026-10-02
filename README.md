# Telegram signal format

A Telegram trading signal is a plain-text message that names a direction and an instrument, and usually adds an entry price or zone, a stop loss, and one or more take-profit levels. There is no standard for it. Every channel writes its own way, and the software that reads them has to cope with all of it.

This repository documents what those messages look like in practice, drawn from live channels, and what a reader has to handle to parse them correctly. It is published by the team behind [TTMT – Telegram to MetaTrader](https://telegramtometatrader.com/?utm_source=github&utm_medium=owned&utm_campaign=signal-format), which executes signals on MetaTrader 4 and MetaTrader 5 accounts. It is written for anyone building something similar, whether or not they use our product.

The maintained version of this document, with examples rendered and kept current, is the [signal format guide in the TTMT docs](https://docs.telegramtometatrader.com/reference/signal-format?utm_source=github&utm_medium=owned&utm_campaign=signal-format). Machine-readable fixtures for every layout below live in [telegram-signal-test-corpus](https://github.com/lukacsaron/telegram-signal-test-corpus).

## Contents

- [Why publish a format nobody follows](#why-publish-a-format-nobody-follows)
- [The minimum](#the-minimum)
- [Layouts seen in the wild](#layouts-seen-in-the-wild)
- [What a reader must tolerate](#what-a-reader-must-tolerate)
- [The two things that break](#the-two-things-that-break)
- [Managing a trade from the channel](#managing-a-trade-from-the-channel)
- [Alert first, details later](#alert-first-details-later)
- [Checklist for channel owners](#checklist-for-channel-owners)

## Why publish a format nobody follows

The absence of a standard is the actual problem, and the cost lands on subscribers rather than on channels.

Measured across just under 72,000 new-trade signals in the 90 days to September 2026: **0.14% failed for a reason the channel owner controlled.** Roughly ten percent failed for reasons entirely on the subscriber's side, such as their own daily-loss limit, a broker connection still syncing, no free margin, or a symbol they had filtered out. The reader is rarely the bottleneck. Knowing which failures belong to whom is worth more than another regular expression.

## The minimum

A message is a trade instruction if it carries two fields:

| Field | Required | Notes |
|---|---|---|
| Direction | yes | `BUY`, `SELL`, `LONG`, `SHORT`, and the obvious variants |
| Symbol | yes | `XAUUSD`, `XAU USD`, `GOLD`, `#XAUUSD`, `XAUUSDSELL` with no space |
| Entry | no | a single price or a zone; if absent, the trade is at market |
| Stop loss | no | inferred from the channel's history when absent, which is a guess |
| Take profits | no | TP1 to TP6, in the order written |

`Sell gold now` is a valid signal. It is also a signal that executes against an inferred stop, which is the one thing a channel can remove for free.

The shortest message worth recommending:

```
SELL XAUUSD 4626-4629
SL 4632
TP1 4623
TP2 4620
```

## Layouts seen in the wild

All five are real shapes from channels running between 94% and 100% execution.

**Zone with labelled blocks**

```
SELL XAUUSD @ 4626-4629

SL 4632

TP1 4623
TP2 4620
```

**Labelled fields**

```
BUY XAUUSD

Entry: 4345.16

SL: 4330.16
TP1: 4350.16
TP2: 4355.16
TP3: 4385.16
```

**Bare inline, repeated target keyword**

```
XAUUSD BUY 4329

TP 4334
TP 4339
TP 4344
TP 4350 OPEN

SL 4320
```

Repeating a bare `TP` is fine. Targets are taken in the order written, so the first is TP1.

**Named zone on its own line**

```
GOLD BUY SETUP

Gold Buy Zone 4438 - 4433

SL: 4428

TP1: 4443
TP2: 4448
TP3: 4453
TP4: Hold
```

**Direction first, price block below**

```
XAUUSD (Gold)
BUY NOW
4346-4344

TP 1: 4350
TP 2: 4354
TP 3: 4360
Stop Loss: 4342
```

## What a reader must tolerate

| Input | Expected behaviour |
|---|---|
| Emoji anywhere, including inside the numbers | Ignored for parsing |
| `Stop Lose`, `Stoploss`, `S/L`, `SL :` | All read as the stop |
| A numbered or titled prefix such as `3. GOLD BUY SETUP` | Skipped |
| `TP4: Hold`, `TP OPEN`, `Hold for more` | A runner. No numeric target invented |
| `4438 - 4433`, `4438/4433`, `@ 4438-4433` | All an entry zone |
| A zone written high to low or low to high | Order does not matter |
| Mixed languages in one message | Values still extracted |
| Risk boilerplate at the end of every post | Ignored |
| More than six targets | First six used, message not rejected |

## The two things that break

**1. The entry zone crowds the stop.** When an entry is a range, orders spread across it, and the deepest sits at the edge nearest the stop. If that edge lands within about 5 pips of the stop, the deepest orders have almost no room before they are stopped out. The distance is measured from the zone edge closest to the stop: for a buy, the bottom of the zone minus the stop; for a sell, the stop minus the top of the zone.

A reader has three options when it sees this: shorten the zone so the deepest order keeps a usable distance, fall back to a single entry, or refuse the signal. Filling the full zone as written leaves the deepest orders a few pips from the stop.

This is subtler than it looks, because subscribers widen entry zones in their own settings. A gap that is safe as posted can be eaten by somebody else's configuration. The practical rule for a channel: keep the far edge of the zone at least as far from the stop as the zone is wide.

**2. The message arrives after the price has gone.** If the market has already traded through the stop by the time the message is read, the position should not open. Opening it means entering an already-losing trade against targets anchored to a price that no longer exists. Tight scalps on gold are where this bites.

## Managing a trade from the channel

No command syntax is needed or desirable. These all executed as written:

```
Move sl to Be
Set Breakeven at lower Part
TP2 hit, close your profits and adjust your SL to entry
Close worst ones and keep best ones running after making them risk free
Lock in profits from worst entries
```

**Use Telegram's reply function.** Over the same 90 days, close instructions posted as a reply to the original signal executed at **25.5%**, against **15.1%** for the identical instruction posted standalone. Modifications ran **59.5%** against **52.7%**. A reply removes the guesswork about which trade is meant, and it is one extra tap.

## Alert first, details later

More than half the channels in TTMT's public directory post an alert and fill in the numbers afterwards:

```
GOLD BUY NOW
```

then, as a reply:

```
Entry 4438-4433
SL 4428
TP1 4443
TP2 4448
```

A reader needs a mode for this: open on the alert, then apply the numbers as a modification when they arrive. The pattern that causes trouble is a channel switching between modes without warning.

## Checklist for channel owners

- Direction and symbol in every signal
- A stop in every signal, so nothing is inferred
- Stop on the correct side: below entry for a buy, above for a sell
- Targets nearest first, six at most
- One trade per message
- Entry zone no wider than the gap between its far edge and the stop
- Reply to your own signal for every breakeven, partial, or close
- The same layout every time
- Post the level before the commentary

## Contributing

Pull requests with real message shapes are welcome, with the channel name removed. Two rules: no screenshots of private channels, and no performance claims about any channel.

## Related

- [TTMT signal format guide](https://docs.telegramtometatrader.com/reference/signal-format?utm_source=github&utm_medium=owned&utm_campaign=signal-format), the maintained version of this document
- [Execution modes](https://docs.telegramtometatrader.com/channels/execution-modes?utm_source=github&utm_medium=owned&utm_campaign=signal-format), how alert-first channels are handled
- [For signal providers](https://telegramtometatrader.com/for-signal-providers?utm_source=github&utm_medium=owned&utm_campaign=signal-format), the same advice written for channel owners
- [telegram-signal-test-corpus](https://github.com/lukacsaron/telegram-signal-test-corpus), fixtures for the layouts above

## Licence and publisher

The text is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Quote it, fork it, or build on it with attribution.

Published by jazzrabbit OÜ (registry code 16489902, Estonia), the company that operates TTMT – Telegram to MetaTrader. Nothing here is financial advice.
