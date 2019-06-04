---
title: Reorganizing your (Phoenix) Contexts as Use Cases
date: "2019-06-03"
description: "Extract your contexts' actions into Use Cases, identifying execution paths and making requirements explicit."
tags: ["elixir", "phoenix", "ecto"]
---

Contexts are a great concept to help you identify well defined boundaries between the different areas of your application. There is nothing new or specific about Phoenix Contexts, it's just a subset - identification and organization - of [Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html), defined by Eric Evans in the excellent [Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215/ref=nav_signin?crid=2VP1T4HLLN3YK&keywords=domain+driven+design&qid=1559488367&s=gateway&sprefix=domain+driven+d%2Caps%2C273&sr=8-1&).

![Contexts example](contexts.png "Contexts example")

In this post I'll show you another way to organize your Contexts by extracting _actions_ into their own **Use Cases** modules to convey better clarity and make requirements explicit.

## What is a Use Case?

> A use case is a written description of how users will perform tasks on your website. It outlines, from a user’s point of view, a system’s behavior as it responds to a request. Each use case is represented as a sequence of simple steps, beginning with a user's goal and ending when that goal is fulfilled. - [Usability.gov](https://www.usability.gov/how-to-and-tools/methods/use-cases.html)

In other words, a Use Case is an action that some user/actor performs on your application, like a button click in the UI, a command entered via CLI or even a response from a external API. It is something that the actor does and expects a side effect: a data change, sending an email, a background calculation.

Phoenix already creates three generic _use cases_ when you use its generator to create or update a context: `create_*/1`, `update_*/2` and `delete_*/1` functions. In a `Sales` context with `Client` and `Order`, Phoenix defines the following functions:

- `create_client/1`
- `update_client/2`
- `delete_client/1`
- `create_order/1`
- `update_order/2`
- `delete_order/1`

Now add order items, payments, addresses and the concept of checkout to the context. Also add the other necessary accompanying functions like data validation, policies verification, external data fetching, email sending and [event logging](/blog/tracking-changes-with-context-using-phoenix-and-ecto), and the number of functions will increase drastically.

![Scumbag brain](scumbagbrain.jpg "Scumbag brain")

There's nothing technically wrong defining all these functions inside a single module in Elixir, but as contexts grows larger you start to have too many functions related to each other but loosely related to the parent concept. The cognitive overhead to build a mental state of all these functions is too much.

## Writing a Use Case

Steps for writing a Use Case:

1. identify the actions a user can perform
2. extract each action in its own Use Case module
3. ...
4. PROFIT!

No joke, that's it. Here's an example for an order cancellation:

```elixir
defmodule AlchemyReaction.Sales.CancelOrderManually do
  @moduledoc """
  Cancels an order with the given reason and notifies the client via email.
  """
  use Ecto.Schema
  import Ecto.Changeset
  alias Ecto.Multi

  alias AlchemyReaction.Sales.Order

  embedded_schema do
    field :reason
  end

  def changeset(attrs) do
    %__MODULE__{}
    |> cast(attrs, [:reason])
    |> validate_required([:reason])
    |> validate_length(:reason, min: 10)
  end

  def call(%Ecto.Changeset{valid?: true} = changeset, %{order: order, actor: actor} = _deps) do
    data = apply_changes(changeset)

    Multi.new()
    |> Multi.update(:order, Order.changeset(order, %{status: "cancelled"}))
    |> Multi.run(:audit, &cancel_order_audit(&1, &2, data.reason, actor))
    |> Multi.run(:email, &notify_client(&1, &2, actor))
  end

  defp cancel_order_audit(_repo, %{order: order}, reason, actor) do
    # ...
  end

  defp notify_client(_repo, %{order: order}, actor) do
    # ...
  end
end
```

The key benefits of writing Use Cases are:

**Code organization and clarity**

You can know, or at least have an idea, what each context do just by looking at the file structure. You can also group the Use Cases in [sub contexts](/blog/tracking-changes-with-context-using-phoenix-and-ecto) folders on larger contexts for better organization. In this case the `CancelOrderManually` would be nested in the sub context `order` and named as `AlchemyReaction.Sales.Orders.CancelOrderManually`.

**You make the requirements explicit**

It's clear that a reason, the order being cancelled and the actor who is doing it are neededed to manually cancel the order. Here I'm passing the `order` and the `actor` as dependencies to the `call/2` function because I'm loading them outside, mainly for checking permissions, but you could have the `order_id` and `actor_id` defined in the schema and load them inside the use case.

Keep in mind that requirements varies depending on the execution context. A reason is required to manually cancel an order by an operator in the customer support, but it may not be required by the analist in the fraud detection department. This is a terrible example, but I hope I can make you through.

Also note that these requirements are strictly related to what is needed to execute the Use Case, there's nothing to do with roles and permissions. Both the operator in the customer support and the analyst in the fraud detection have permissions to manually cancel an order.

**Use the Use Case to build your forms**

The changeset serves as the data structure to build your forms the same way you would use an `Order` schema in a traditional way (you will need to set the `as` namespace and the submission `method` manually as the `form_for/x` helper can't infer from the struct).

You are decoupling the requirements to execute the use case from the underlying implementation, as oftentimes a change can span multiple schemas.

**It's easy to reuse and compose Use Cases**

You must always return an `Ecto.Multi` from the `call/2` function. Always is a strong word, but I really mean ALWAYS!

> Ecto.Multi makes it possible to pack operations that should be performed in a single database transaction and gives a way to introspect the queued operations without actually performing them. Each operation is given a name that is unique and will identify its result in case of success or failure. - [Ecto.Multi docs](https://hexdocs.pm/ecto/Ecto.Multi.html)

By always returning a multi you can reuse other Use Cases, composing them together by appending/merging them into the parent multi. Think of cancelling an order where you also have to cancel the payment, put back the items in stock and maybe scheduling an email for inviting the user to buy again in 2 weeks. Each of these use cases can be executed by themselves giving other contexts.

I use `Ecto.Multi` even when there's no database operation. The cost of opening/close an unutilized transaction is negligible.

`Ecto.Multi` is love. `Ecto.Multi` is life. :heart: :heart: :heart:

**Don’t make your Use Cases generic or (overly) configurable**

One of the key benefits of a Use Case is that it reflects a single, defined, finite execution path in the application. If you try to make it generic enough to handle multiple use cases, then you lose the benefit of clarity and context.

Having a flag to indicate that an email notification should be sent after performing the use case is ok. Having multiple options and knobs resulting in multiple execution paths and outputs is a red flag.

## Identifying a Use Case

This is the hardest part. It's a constant process of mapping and understanding [why and how data changes](/blog/tracking-changes-with-context) by analyzing the day to day of the application's users and the corner cases that surface from time to time.

![USE THE VERBS, LUKE](use-the-verbs.jpg "USE THE VERBS, LUKE")

Pay attention to the [language](https://martinfowler.com/bliki/UbiquitousLanguage.html) they use, specially the verbs. Verbs denote actions and actions are what you're interested in.

It's ok if you don't get it right the first (or second, or third) time. But it's important you keep refining...

Some other examples of Use Cases:

- `Sales` context: `ShipOrder`, `RefundOrder`, `ChangeShippingAddress`, `CancelOrderViaPaymentGateway` (naming is hard);
- `Accounts` context: `ChangePrimaryEmail`, `ChangePassword`, `DeactivateAccount`, `ConfirmEmailAddress`, `SetPreferredPaymentMethod`, `SetPreferredShippingAddress`;

## Conclusion

It may seem cumbersome to write this much code to do the same thing as the traditional way. I don't have any metrics, but my feeling is that it's not much more, it could even be the same. You're not just typing code out, you're mapping every operation and process of the application, identifying and documenting every execution path.

I've been using this concept of Use Cases with great success for multiple years, in applications written in Elixir, Ruby and even PHP.
