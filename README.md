# Policy-controlled feature `shared-autofill`

We propose a new policy-controlled feature `shared-autofill` for safe autofilling of cross-origin forms.

## Motivation

**Autofill.**
Autofill is a browser feature that assists the user with filling forms.
While there is no clear-cut definition, the feature typically activates when the user focuses a text field and suggests values for the field and other, semantically related fields.

For example, consider a credit card form.
When the user focuses the cardholder-name field, Autofill may offer to fill also the associated credit card number field, expiration date, and/or CVC.

**Multi-frame forms.**
In some forms, the control elements are distributed across multiple frames.
This is particularly common with payment forms, where the individual form controls are embedded via individual iframes from a third-party Payment Service Provider (PSP), like in this pseudo-code example:

```
<!-- Top-level document URL: https://merchant.example/... -->
<form>
  Cardholder name:    <input id="name">
  Credit card number: <iframe src="https://psp.example/..."><input id="num"></iframe>
  Expiration date:    <input id="exp">
  CVC:                <iframe src="https://psp.example/..."><input id="cvc"></iframe>
                      <iframe src="https://ads.example/..."><input id="account"></iframe>
</form>
```

For the merchant, this design combines security and flexibility: the cross-origin iframes isolate the sensitive payment information from the merchant's infrastructure (which helps them with [PCI DSS](https://www.pcisecuritystandards.org/) compliance), yet the merchant can arrange and style the fields in their website's look and feel.

From the browser perspective, this means that there are common and legitimate use-cases of multi-frame forms.
This raises questions about the security model of  cross-origin iframes.

For example, when the user attempts to autofill the above credit card form, the `account` field should likely not be filled, for otherwise `https://ads.example` could inject fields to steal credit card data (e.g., if the ads service was compromised).
On the other hand, filling credit card data into `https://psp.example` is apparently desired.

This example illustrates that the same-origin policy is a solid baseline for autofilling across frames, but does not provide sufficient granularity for the browser to differentiate between untrusted and trusted (for the purposes of Autofill) frames.

## Proposal

We propose a new policy-controlled feature `shared-autofill` by which a parent frame can designate the trustworthiness of iframes as far as Autofill is concerned.
The browser shall treat this feature as a necessary condition for autofilling across origins.

**Terminology.**
* An *autofill* is a single operation that fills multiple form controls of the [fully active documents](https://html.spec.whatwg.org/multipage/browsers.html#fully-active).
* An *autofill's origin* is the [origin of the document](https://dom.spec.whatwg.org/#concept-document-origin) that [has focus](https://html.spec.whatwg.org/multipage/interaction.html#has-focus-steps).

In our above example, when the user focuses the cardholder-name field and autofills the form, the autofill's origin is `https://merchant.example`.
The semantics of `shared-autofill` shall be as follows:

**Definition.**
An autofill *A* may fill a form control *FC* only if
* *A* and *FC*'s document have the [same origin](https://html.spec.whatwg.org/multipage/origin.html#same-origin) or
* `shared-autofill` is [enabled in *FC*'s document for *FC*'s document's origin](https://w3c.github.io/webappsec-permissions-policy/#algo-is-feature-enabled).

The default allowlist of `shared-autofill` shall be `self`.

# Discussion

Using `shared-autofill`, a website can discriminate between trusted and untrusted cross-origin iframes.
Let us validate the definition with the above example, now assuming the merchant has annotated the PSP-hosted iframes with `shared-autofill`:

```
<!-- Top-level document URL: https://merchant.example/... -->
<form>
  Cardholder name:    <input id="name">
  Credit card number: <iframe src="https://psp.example/..." allow="shared-autofill"><input id="num"></iframe>
  Expiration date:    <input id="exp">
  CVC:                <iframe src="https://psp.example/..." allow="shared-autofill"><input id="cvc"></iframe>
                      <iframe src="https://ads.example/..."><input id="account"></iframe>
</form>
```

The `shared-autofill` feature is enabled (only) in the top-level document (due to the default allowlist `self`) and the two PSP documents (by the [inherited policy](https://w3c.github.io/webappsec-permissions-policy/#algo-define-inherited-policy-in-container) whose target list [defaults to `src`](https://w3c.github.io/webappsec-permissions-policy/#declared-origin)).
The following table illustrates which fields may be autofilled depending on the autofill's origin:

| Origin                     | `name`   | `num`    | `exp`    | `cvc`    | `account` |
|----------------------------|:--------:|:--------:|:--------:|:--------:|:---------:|
| `https://merchant.example` | &#10004; | &#10004; | &#10004; | &#10004; | &#10006;  |
| `https://psp.example`      | &#10004; | &#10004; | &#10004; | &#10004; | &#10006;  |
| `https://ads.example`      | &#10004; | &#10006; | &#10004; | &#10006; | &#10004;  |

In more detail: if an autofill's origin is ...
* `https://merchant.example`, then
  - `name` and `exp` may be autofilled because of the same-origin clause
  - `num` and `cvc` may be autofilled because their inherited policies enable `shared-autofill` on `https://psp.example`
  - `account` must not be autofilled because the inherited policy disallows `shared-autofill` and they're cross-origin
  because either `shared-autofill`
* `https://psp.example`, then
  - `name` and `exp` may be autofilled because the default allowlist enables `shared-autofill` in the top-level document
  - `num` and `cvc` may be autofilled because of the same-origin clause
  - `account` must not be autofilled because the inherited policy disallows `shared-autofill` and they're cross-origin
* `https://ads.example`, then
  - `name` and `exp` may be autofilled because the default allowlist enables `shared-autofill` in the top-level document
  - `num` and `cvc` must not be autofilled because the inherited policy disallows `shared-autofill` and they're cross-origin
  - `account` may be autofilled because of the same-origin clause

Since `shared-autofill` is only a necessary condition, the browser can impose further restrictions on when to autofill a form control in a cross-origin frame.
For example, it may exclude credit card numbers from cross-origin autofills, or it may decide not to autofill across frames altogether.

Note that the semantics of `shared-autofill` is based only on the focused document.
It thus abstracts from the typical interaction flow that begins with the user focussing a field.
For example, the definition is also compatible with a flow where the browser prompts the user when it has detected a fillable form.

## Attack vectors

Malicious use of `shared-autofill` could expose autofill data to untrusted cross-origin frames.
An attacker could use this, for example, to steal filled credit card data or to execute a payment in a child frame.
This section assesses the scenarios and mitigations.

Such an attack presupposes that the parent document explicitly enables `shared-autofill` in its cross-origin child frames.
We consider two cases:

**Case 1.**
The attacker controls the parent frame.

Then, they could make use of `shared-autofill` to trick the user, for example, into executing a payment: the attacker embeds an invisible iframe that loads a cross-origin payment form, and tricks the user into autofilling a, say, cardholder-name field in the parent frame.
Due to `shared-autofill`, this also fills the payment form in the attacker's child frame.
Next, the attacker click-jacks the user into submitting the form.

However, it's likely that the attacker could also click-jack the user into autofilling the form in the iframe directly – entirely without `shared-autofill`.
Also, an attacker who controls the parent frame arguably has much simpler means to steal the payment information by manipulating the parent frame directly.

**Case 2.**
The attacker only controls the child frame.

If the parent document enabled `shared-autofill` in that child document, the attacker could add a payment form to the child document and steal any values autofilled into these fields.

In this case, `shared-autofill` is a key component of the attack.
However, it fundamentally relies incorrect and dangerous use of `shared-autofill` on the parent document's end.

**Mitigation.**
In either case, the parent document's policy could effectively override any malicious `allow="shared-autofill"` attributes (because the document policy [takes precedence](https://w3c.github.io/webappsec-permissions-policy/#algo-define-inherited-policy-in-container) over the container policy).
For example, the HTTP header `Permissions-Policy: shared-autofill=(self "https://psp.example")` implicitly prevents all origins except for `https://psp.example` from inheriting shared-autofill.

## Demo

Chrome 93.0.4577.0 or later includes an experimental implementation when run with the flags `--enable-features=AutofillAcrossIframes,AutofillSharedAutofill`.
There's a [test page](https://schwering.github.io/shared-autofill/form.html) and a [video of the user experience](https://drive.google.com/file/d/1ToX_Q3QdQW1fGqjBVYaBq9Z1zZMTF77N/view?usp=sharing).

