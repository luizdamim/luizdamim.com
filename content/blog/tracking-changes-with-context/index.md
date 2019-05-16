---
title: Tracking changes with context
date: "2019-05-01"
description: "What, who and why of keeping track of data changes"
# tags: ["elixir", "phoenix", "ecto"]
---

Imagine an online shop who sells... anything, really. Users place orders buying products they want. But sometimes _they regret_ their decision ~~me 2min after buying something~~. Sometimes there's a _problem_ with the _payment gateway_. Sometimes you have a problem in your warehouse and the products are _out of stock_. Sometimes you can't sell because you can't _ship to the user's address_ or any other constraint that can't be verified at checkout time (please please PLEASE verify everything you can at checkout time, everyone gets happy :)).

All these possibilities result in the order cancellation. Behind the scene the application updates the order's `status` field from `processing` to `cancelled` and hopefully logs the date/time. That's it. Right? RIGHT?

## Sometimes knowing _What_ changed is not enough. It's essential to know _Why_ it changed.

If you don't know **why** something happened you are losing insight about your application, making you unable to act upon it.

---

**Example 1: User doesn't have sufficient funds**

Payment declined because the user doesn't have sufficient funds. The application cancels the order automatically.

The fields `gateway` and `reason` could be set by your application's payment workflow and `error_code` returned by the gateway processor.

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

_Although in this example_ `actor_id` _is_ `null`, _I usually have an application user that is assigned to every automated action in the application._

**Example 2: Can't ship to the user's address**

User changed their shipping address after placing the order and for whatever reason you can't ship there.

Here `reason` and `description` were informed by an employee after manually analyzing the problem.

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

_These are contrived examples. There are better ways to handle these cases than cancelling the order._

---

On the next post I'll show one way you can do it in Elixir using _Ecto_ and _Phoenix_.

_Many thanks to Allan Jorge, @gustavoaguiar and @tyurok from_ [_Elixir Brazil OffTopic @ Telegram_](https://t.me/elixirbr_offtopic) _for helping me overcome my writing block / fear of writing this first post._
