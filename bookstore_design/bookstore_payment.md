# Payment with a 3P Service like Stripe

- [Payment with a 3P Service like Stripe](#payment-with-a-3p-service-like-stripe)
  - [Requirements and User Experience Context](#requirements-and-user-experience-context)
  - [One-time Payment Method](#one-time-payment-method)
    - [The Problem: Handling Raw Credit Card Data is Difficult and Risky](#the-problem-handling-raw-credit-card-data-is-difficult-and-risky)
    - [How the Payment Token Flow Works](#how-the-payment-token-flow-works)
  - [Use Stored Payment Methods](#use-stored-payment-methods)
    - [The Concept: `Customer` and `PaymentMethod` Objects](#the-concept-customer-and-paymentmethod-objects)
    - [Workflow for a First-Time Buyer Saving a Card](#workflow-for-a-first-time-buyer-saving-a-card)
    - [Workflow for a Returning Customer](#workflow-for-a-returning-customer)

## Requirements and User Experience Context

This document details the technical design for the payment processing stage of the bookstore's checkout flow. The complete end-to-end user experience for checkout is defined in [`bookstore_analytics.md`](./bookstore_analytics.md). This design specifically covers the implementation of **Step 3: Payment Information** from that user flow.

The key user experience requirements that this design addresses are:
* Allowing a user to pay by entering new credit card information into a secure, embedded form.
* Allowing a returning customer to select a previously saved payment method for a faster checkout.
* Providing an option for the user to save their new card details for future purchases.

The following sections explain the tokenization strategy and data models used to meet these requirements while ensuring maximum security and minimizing PCI DSS compliance scope.

---

## One-time Payment Method

A **payment token** is a secure, single-use, non-sensitive string of characters that represents a customer's raw credit card information. It acts as a substitute or a stand-in for the actual card details (the 16-digit number, CVV, and expiration date).

The entire purpose of using a payment token is to **avoid ever letting sensitive credit card data touch your own servers**. This dramatically enhances security and reduces your compliance burden.

### The Problem: Handling Raw Credit Card Data is Difficult and Risky

If your server directly handles a customer's credit card number, you are subject to a strict and complex set of security standards called the **Payment Card Industry Data Security Standard (PCI DSS)**.

* **Complexity:** Achieving and maintaining PCI compliance is extremely onerous and expensive. It involves rigorous network security, strict access controls, regular vulnerability scans, penetration testing, and extensive logging.
* **Risk:** If your systems are breached and you lose customer credit card data, the financial and reputational damage to your business can be catastrophic, including massive fines and the loss of the ability to process payments.

Modern payment gateways like **Stripe** or **Braintree** provide a "tokenization" system so you can avoid this entire problem.

### How the Payment Token Flow Works

Here is the step-by-step process in our bookstore architecture, using Stripe as the example:

1.  **Secure Frontend Fields (Stripe.js):**
    * On your checkout page, you don't use standard HTML `<input>` fields for the credit card number and CVV.
    * Instead, you include Stripe's JavaScript library (`Stripe.js`). This library creates secure `<iframe>` elements on your page that are hosted directly by Stripe.
    * The customer sees a seamless form, but they are actually typing their sensitive card details directly into fields controlled by Stripe's secure servers, not yours.

2.  **Client-Side Token Creation:**
    * When the customer clicks "Place Order", your frontend JavaScript **does not** read the credit card number.
    * Instead, it calls a function in the `Stripe.js` library (e.g., `stripe.createToken()`).
    * The `Stripe.js` library takes the card details the user entered and sends them directly from the user's browser to Stripe's servers over a secure connection. **This data completely bypasses your server.**

3.  **Stripe Sends Back a Token:**
    * Stripe's servers receive the raw card data, validate it, and securely store it in their PCI-compliant vault.
    * Stripe then generates a unique, single-use token (it looks like `tok_1MguA52eZvKYlo2Chg8G4bT1`).
    * This token is sent back to the user's browser.

4.  **Your Server Receives the Safe Token:**
    * Now, your frontend JavaScript takes this safe, non-sensitive token—along with the rest of the order information (cart contents, shipping address)—and sends it to your backend (`Order Service`).
    * This is the first time any representation of the payment information has touched your application.

5.  **Your Server Uses the Token to Charge:**
    * Your `Order Service` (specifically, the `Payment Service` in our design) receives the token `tok_...`.
    * It then makes a secure, server-to-server API call to Stripe, telling it to "create a charge using this token."
    * Stripe receives the request, looks up the token in its system, finds the real credit card details associated with it, and processes the payment.

By using this flow, the payment token acts as a secure proxy, allowing you to charge your customer without ever having the liability of handling their raw card information.

## Use Stored Payment Methods

The single-use token flow is perfect for guest checkouts or first-time buyers, but forcing repeat customers to enter their card information every time would be inconvenient. The solution is an extension of the tokenization concept, designed specifically for **saving cards on file** securely. The principle remains the same: your server never touches the raw credit card number.

Here’s how it works, using Stripe's model as an example:

### The Concept: `Customer` and `PaymentMethod` Objects

Instead of a single-use `Token`, payment gateways provide objects for long-term storage of customer information. In Stripe, these are:

1.  **The `Customer` Object:** You create a `Customer` object in Stripe that corresponds to a user in your site's database. This object acts as a container for that user's saved payment details and other information. You save the Stripe `Customer` ID (e.g., `cus_Abc123...`) in your own `Users` table.
2.  **The `PaymentMethod` Object:** This represents a customer's saved card. It is also a secure, non-sensitive identifier (e.g., `pm_Def456...`) that you can safely handle.

### Workflow for a First-Time Buyer Saving a Card

Let's say a new user is checking out and ticks a box that says, "Save this card for future purchases."

1.  **Secure Frontend Entry:** The user enters their card details into the secure Stripe Elements `<iframe>`, just like before.
2.  **Create a `PaymentMethod`:** When the user submits, your frontend JavaScript calls a function like `stripe.createPaymentMethod()`. This sends the card details directly to Stripe and returns a permanent `PaymentMethod` ID (`pm_...`) to the browser.
3.  **Create a Stripe `Customer`:** Your frontend sends the `pm_...` ID to your backend. Your backend server then tells Stripe:
    * "Create a new `Customer` for my user with this email address." Stripe returns a `Customer` ID (`cus_...`), which you **save in your database** associated with your user's account.
    * "Now, attach this `PaymentMethod` (`pm_...`) to this new `Customer` (`cus_...`)."
4.  **Process the Current Charge:** To pay for the current order, your backend tells Stripe, "Charge `Customer` `cus_...` using their saved `PaymentMethod` `pm_...`."

### Workflow for a Returning Customer

This is where the user experience becomes seamless.

1.  **Load Checkout Page:** The user logs in and proceeds to checkout.
2.  **Fetch Saved Cards:** Your backend retrieves the Stripe `Customer` ID (`cus_...`) from your database for the logged-in user. It then asks Stripe, "Please give me the list of saved cards for `Customer` `cus_...`."
3.  **Display Safe Details:** Stripe returns a list of `PaymentMethod` objects. This list **does not contain card numbers**. It contains safe information you can display to the user, such as "Visa ending in 4242, expires 12/2028".
4.  **One-Click Purchase:** The user sees their saved card(s), selects one, and clicks "Confirm Purchase."
5.  **Charge the Saved Card:** Your frontend simply sends the chosen `PaymentMethod` ID (`pm_...`) to your backend. Your backend then tells Stripe, "Charge `Customer` `cus_...` using their saved `PaymentMethod` `pm_...`."

No card entry or tokenization is needed for this subsequent purchase.

**Summary of Benefits:**

* **User Convenience:** Customers get a fast, one-click checkout experience, which is proven to increase conversion rates.
* **Persistent Security:** Your application **still never sees or stores a raw credit card number**. You only store and handle the safe, non-sensitive `Customer` and `PaymentMethod` IDs provided by the payment gateway, keeping your PCI compliance scope minimal.