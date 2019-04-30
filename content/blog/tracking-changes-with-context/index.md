---
title: Tracking changes with context
date: "2019-04-30"
description: "The what, which, why and who of keeping track of data changes"
# tags: ["elixir", "phoenix", "ecto"]
---

Imagine an online shop who sells... anything, really. Users place orders buying products they want. But sometimes _they regret_ their decision ~~me 2min after buying something~~. Sometimes there's a _problem_ with the _payment gateway_. Sometimes you have a problem in your warehouse and the products are _out of stock_. Sometimes you just can't sell because you can't _ship to the user's address_ or any other constraint that can't be verified at checkout time (please please PLEASE verify everything you can at checkout time, everyone gets happy :)).

All these possibilities result in the order cancellation. Behind the scenes the application changes the order's `status` field from `processing` to `cancelled` and hopefully logs the date/time. That's it. Right? RIGHT?

## Sometimes knowing _Which_ data changed isn't enough. It's important to know _Why_ it changed.

If you only know that some data changed you're losing insight about your application. Not knowing **why** it changed you can't act upon it. You lose context.

By treating your changes as events you can have any level of granularity you need not only to know what changed and who changed, but most importantly **why it changed**.

**Example 1:**

```json
{
  "event": "order_cancelled",
  "id": 42,
  "actor_id": null,
  "metadata": {
    "reason": "payment_declined",
    "error_code": "INSUFFICIENT_FUNDS",
    "gateway": "IPayU"
    ...
  }
}
```

**Example 2:**

```json
{
  "event": "order_cancelled",
  "id": 42,
  "actor_id": 19,
  "metadata": {
    "reason": "shipping_address",
    "description": "User changed his address, can't ship to the new one."
    ...
  }
}
```

On my next post I'll show you one way we can do it in Elixir using _Ecto Changesets_ and _Phoenix **Contexts**_.

_Many thanks to Allan Jorge, @gustavoaguiar and @tyurok from [Elixir Brazil [OffTopic] @ Telegram](https://t.me/elixirbr_offtopic) for helping me overcome my writing block / fear of writing this first post._
