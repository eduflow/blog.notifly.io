---
title: My gripe with Stripe
author: Malthe Jørgensen
---
_This post talks about [Eduflow](https://www.eduflow.com) and [Peergrade](https://www.peergrade.io) – our two main products._

Let's take a look at two fairly similar scenarios:

- I'm signing up for some service and my card is declined
- I'm signing up for some service and I fail the 3DSecure check

As a programmer – I know what you're saying – under the hood these two are of course very different.

But from the user's perspective, they were trying to pay and they failed to do so: potato, *potato3DSecure*. Unfortunately, if you're using Stripe, the outcome of these two failures are very different:

- If I'm creating a subscription in Stripe and I get a normal card decline, nothing happens. The user goes back to square one. No subscription, no nothing – as expected.
- If I'm creating a subscription in Stripe and I fail the 3DSecure verification – maybe I figured out that I didn't want to buy anyways, once I saw that scary verification screen [1]. In this case, Stripe will have created the subscription and is basically expecting the user to pay for it within the next 23 hours. Basically, there's a bunch of state, that both the user and the developer would expect to only have been created once the payment succeeded. ([Stripe docs](https://stripe.com/docs/billing/subscriptions/overview#subscription-lifecycle))

Now, that's a leaky abstraction if I ever saw one.

In the olden days (for us that was Stripe Checkout Legacy along with Stripe Billing), the payment flow between a web app and Stripe would be this:

Web app → Stripe → Web app

Principally, this is still possible if you use the new Stripe Checkout. But if you have anything but the most vanilla of setups – that will not work for you: tiered pricing, per-seat pricing, and metered billing are not available in Stripe Checkout as of Jan 14th 2021. Like many other SAAS businesses, we use tiered per-seat pricing in Eduflow.

So we need to use Stripe Elements, build our own form, and write a ton of Javascript boilerplate that is probably 90-95% identical to what other users of Stripe Elements are writing.

The flow now looks roughly like this:

Web app → Stripe → Web app → Stripe → Web app

Bear in mind that this is the flow in the frontend. There has always been a bit of backend setup, but the amount of code and complexity there has roughly stayed the same when comparing with the old setup.

The extra added round-trip to Stripe here is related to 3DSecure. The idea is – if you after the first round-trip have a PaymentIntent with `status == "requires_action"` it means 3DSecure is required for the payment. You should then call `stripe.confirmCardSetup()` in the frontend which initiates the second round-trip to Stripe.

As far as I can see from [the Stripe documentation](https://stripe.com/docs/billing/subscriptions/overview#requires-action) there is no other option than `stripe.confirmCardSetup()` when this happens, so why do I even have the choice to initiate it? If Stripe just called `stripe.confirmCardSetup()` itself we'd be back to good old happy-path:

Web app → Stripe → Web app (but still with a lot of extra boilerplate)

Yes, initiating 3DSecure myself allows me to warn the user about it first. But as a European I'm getting 3DSecure everywhere these days (probably because we've just entered 2021) – with no warning beforehand – so I think it would make more sense for Stripe to allow developers to stay on the happy-path instead of defaulting to a more complicated special-case with no opt-out.

That was the original value proposition of Stripe. The frontend code we have in Peergrade is these **23 lines of code** and it does tiered, per-seat pricing:

```jsx
<StripeCheckout
  name="Peergrade"
  description={plan.name}
  amount={billedNow * 100} // amount is in cents
  email={email}
  stripeKey={publicKey}
  billingAddress={true}
  zipCode={true}
  token={({ id, email }) =>
    createSubscription({
      stripeToken: id,
      stripeEmail: email,
      planType: plan.id,
      quantity,
    })
  }
>
  <input
    type="hidden"
    ref={(input) => {
      this._input = input
    }}
  />
</StripeCheckout>
```

Granted, this uses `react-stripe-checkout`, but that's just a very thin wrapper around `https://checkout.stripe.com/checkout.js`. 

In Eduflow, because we're forced to use Stripe Elements and it's interweaved throughout our subscriptions page, with `<CardNumberElement>`, `<CardExpiryElement>`, `<CardCvcElement>`
and calls to `stripe.createPaymentMethod()` and `stripe.confirmCardPayment()`.

Technically we should do that in Peergrade as well due to PSD2, but for now we will just let those payments fail and ask the customers to switch to Eduflow. It's too much hassle.

That old script – `checkout.js` – is really a one-line payment integration. Just include the script and you get the payment button! What happened Stripe?

I know what you're gonna say – PSD2/SCA/3DSecure happened. Yes, it did. But Stripe already needed to ask the bank whether to allow the transaction (going through VISA/Mastercard and a bunch of other steps behind the scenes). Now Stripe also needs to ask the customer as well through something like 3DSecure. `checkout.js` already shows a modal with a form for inputting name, address and credit card, so they can just reuse that modal to let the user input the 3DSecure verification as well.

Interestingly, if you're not creating a new subscription but rather upgrading an existing one you can use "[Pending updates](https://stripe.com/docs/billing/subscriptions/pending-updates)" to not have the subscription update until the transaction is fully confirmed with 3DSecure and everything. Why isn't that the default, and why isn't it available when the subscription is initially created?

[1] Everybody, [including Stripe](https://stripe.com/en-gb-dk/guides/strong-customer-authentication), expects 3DSecure to lower conversion rates, both because the extra step increases friction, but also because it will be unfamiliar and scary to some users. On a personal note, the first time I was asked to do 3DSecure probably 10 years ago or more, I declined. I never saw a 3DSecure prompt again until about 6 months ago. It just wasn't something I had heard about, and so it seemed scary that I need to receive a text message in order to do a credit card purchase. Over time, I expect customers will get used to it and conversion rates will normalize.

<!--
Card declines were already expected to be async in Stripe. The recommendation was to only provisioning whatever the user was paying for once your webhook was hit.

~~So here's my suggestion: The customer inputs their personal details and card details, and the web app sends that off to Stripe. Stripe checks with the bank whether the transaction is good to go. If the bank or some other security system replies "You need to do 3DSecure" – just do that – show the customer a spinner until the transaction (including 3DSecure) is finished. As a store/service/paid web app, I don't need to know about all that – it's part of the transaction.~~
-->
