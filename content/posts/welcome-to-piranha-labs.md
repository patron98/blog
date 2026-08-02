---
title: "Welcome to Piranha Labs: an €80 Computer, a Spare MacBook, and a Question"
date: 2026-08-02T08:00:00Z
tags: ["meta", "trading", "self-hosted", "introduction"]
summary: "The origin story of this blog: a self-hosted algorithmic trading lab built on a Raspberry Pi and a leftover MacBook, run under one rule — the data decides, not the vibes. Start here."
---

Every trading-bot story on the internet follows the same arc: someone
builds a bot, shows you a green equity curve, and sells you a course.
This blog is the other story — the one where we actually try to find out,
with instruments instead of vibes, whether a self-built system has an
edge at all. Spoiler: we don't know yet. That's the point.

## The question

Can a self-hosted system, built and operated by one person (plus an AI
pair-engineer doing the heavy lifting), find and *prove* a real trading
edge — before risking a single euro of real money?

Not "can it make money in a backtest" — anyone can make a backtest sing.
Proven, as in: measured live, on demo money, with the same discipline
you'd want from a medical trial. Regime boundaries respected. Leaky
features hunted down. Thresholds justified by calibration curves instead
of gut feeling. If the answer turns out to be *no edge*, that's a valid
result and this blog will say so.

## The lab

Everything runs on hardware most people would call e-waste-adjacent:

- **A Raspberry Pi 5** (€80) runs the Java trading bot, Postgres,
  Prometheus, Grafana, the CI runners that build our images, and — since
  recently — a media server, because the Pi was clearly not busy enough.
- **A 2019 MacBook** with a cooling pad runs the Python ML engine:
  per-symbol LightGBM models, a continuous training loop, and an Optuna
  optimizer that runs walk-forward parameter studies every night.
- **Six rule-based strategies** generate signals; an **ML gate** decides
  which signals deserve an order. Two strategies currently trade for
  real (demo-real); four run in "shadow mode" — every signal is judged
  and journalled, but no order is placed. The shadows exist purely to
  feed the measurement.

The whole thing journals obsessively: every decision, every rejection,
every fill, every outcome — including the outcomes of trades we *didn't*
take. That shadow ledger turned out to be the most valuable instrument in
the lab, which is a story of its own
([we wrote it up](/blog/posts/honest-features-silent-bot/)).

## The rules

1. **Demo money until the data says otherwise.** The bot graduates to
   real money when the measurement proves an edge — not when a good week
   feels convincing.
2. **Honest measurement beats profit.** If a fix makes the metrics worse
   but truer, the fix ships. We've already lived through this one: we
   repaired two data leaks and our own bot went silent for three days.
   Worth it.
3. **Everything is observable.** If a component can fail silently, it
   gets an alert. Our alert list has grown mostly by being burned:
   models that decay, strategies that go quiet, backups that "succeed"
   without backing anything up.
4. **The failures get published.** Postmortems are the most useful
   things engineers write and the rarest things trading people publish.
   We intend to be the exception.

## What's here so far

- [We Fixed Our ML Features and Our Trading Bot Went Silent for Three
  Days](/blog/posts/honest-features-silent-bot/) — the leak-fix
  postmortem, and why every threshold in your system is secretly
  calibrated to the feature regime you set it in.
- [Our CI Failed in 2 Seconds With No Error — So We Moved It to a
  Raspberry Pi](/blog/posts/ci-moved-to-a-raspberry-pi/) — how emulated
  ARM builds quietly ate our CI budget, and the €80 fix.

Coming when the data ripens: the shadow lab (measuring the trades you
didn't take), model-decay detection that only judges the model actually
in production, and — eventually — the calibration curve that will decide
whether this bot ever touches real money.

*Nothing here is financial advice. The piranha is small, the tank is
demo, and the only thing we're certain of so far is that honest
measurement is harder than trading.*
