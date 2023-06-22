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

For the merchant, this design combines flexibility and security.
On the one hand, the merchant retains control over their website's look and feel: when each iframe contains only one form control and no other visible elements, the merchant can treat the iframe as if it was a form control when it comes its to properties like display behavior, dimensions, or position.
On the other hand, cross-origin iframes isolate the sensitive payment information from the merchant's infrastructure, which makes it easier for the merchant to comply with the [Payment Card Industry Data Security Standard (PCI DSS)](https://www.pcisecuritystandards.org/): Section 2.2.3 of the [best-practices supplement](https://listings.pcisecuritystandards.org/pdfs/best_practices_securing_ecommerce.pdf) states that
> a merchant implementing an e-commerce solution that uses iFrames [...] may be eligible to assess its compliance using [...] the smallest possible subset of PCI DSS requirements, because most of the PCI DSS requirements are outsourced to the PSP.

From the browser perspective, this means that there are common and legitimate use-cases of multi-frame forms.
This raises questions about the security model of  cross-origin iframes.

For example, when the user attempts to autofill the above credit card form, the `account` field should likely not be filled, for otherwise `https://ads.example` could inject fields to steal credit card data (e.g., if the ads service was compromised).
On the other hand, filling credit card data into `https://psp.example` is apparently desired.

This example illustrates that the same-origin policy is a solid baseline for autofilling across frames, but does not provide sufficient granularity for the browser to differentiate between untrusted and trusted (for the purposes of Autofill) frames.

## Proposal

We propose a new policy-controlled feature `shared-autofill` by which a parent document can designate the trustworthiness of child documents as far as Autofill is concerned.
The browser shall treat this feature as a necessary condition for autofilling across origins.

**Terminology.**
* An *autofill* is a single operation that fills multiple form controls of the [fully active descendant of a top-level traversable with user attention](https://html.spec.whatwg.org/multipage/interaction.html#fully-active-descendant-of-a-top-level-traversable-with-user-attention).
* An *autofill's origin* is null if no element is [focused](https://html.spec.whatwg.org/interaction.html#focused); otherwise it is the [focused](https://html.spec.whatwg.org/interaction.html#focused) element's [node document](https://dom.spec.whatwg.org/#concept-node-document)'s [origin](https://dom.spec.whatwg.org/#concept-document-origin).

In our above example, when the user focuses the cardholder-name field and autofills the form, the autofill's origin is `https://merchant.example`.
The semantics of `shared-autofill` shall be as follows:

**Definition.**
An autofill *A* may fill a form control *FC* only if
* *A* and *FC*'s document have the [same origin](https://html.spec.whatwg.org/multipage/origin.html#same-origin) or
* `shared-autofill` is [enabled in *FC*'s document for *FC*'s document's origin](https://w3c.github.io/webappsec-permissions-policy/#algo-is-feature-enabled).

The default allowlist of `shared-autofill` shall be `'self'`.

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

The `shared-autofill` feature is enabled (only) in the top-level document (due to the default allowlist `'self'`) and the two PSP documents (by the [inherited policy](https://w3c.github.io/webappsec-permissions-policy/#algo-define-inherited-policy-in-container) whose target list [defaults to `'src'`](https://w3c.github.io/webappsec-permissions-policy/#iframe-allow-attribute)).
The following table illustrates which fields may be autofilled depending on the autofill's origin:

| Origin                     | `name`   | `num`    | `exp`    | `cvc`    | `account` |
|----------------------------|:--------:|:--------:|:--------:|:--------:|:---------:|
| `https://merchant.example` | &#10004; | &#10004; | &#10004; | &#10004; | &#10006;  |
| `https://psp.example`      | &#10004; | &#10004; | &#10004; | &#10004; | &#10006;  |
| `https://ads.example`      | &#10004; | &#10004; | &#10004; | &#10004; | &#10004;  |

In more detail: if an autofill's origin is ...
* `https://merchant.example`, then
  - `name` and `exp` may be autofilled because of the same-origin clause;
  - `num` and `cvc` may be autofilled because their inherited policies enable `shared-autofill` on `https://psp.example`;
  - `account` must not be autofilled because the inherited policy disallows `shared-autofill` and it's cross-origin;
* `https://psp.example`, then
  - `name` and `exp` may be autofilled because the default allowlist enables `shared-autofill` in the top-level document;
  - `num` and `cvc` may be autofilled because of the same-origin clause;
  - `account` must not be autofilled because the inherited policy disallows `shared-autofill` and it's cross-origin;
* `https://ads.example`, then
  - `name` and `exp` may be autofilled because the default allowlist enables `shared-autofill` in the top-level document;
  - `num` and `cvc` may be autofilled because their inherited policies enable `shared-autofill` on `https://psp.example`;
  - `account` may be autofilled because of the same-origin clause.

Since `shared-autofill` is only a necessary condition, the browser can impose further restrictions on when to autofill a form control in a cross-origin frame.
For example, it may limit filling highly sensitive data like credit card numbers to documents whose origin is a descendant of the autofill's origin.
Also, the user agent may just decide not to autofill across frames altogether.

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
The attacker controls the parent document.
They trick the user into filling and submitting a form in a descendant frame.

The attacker tricks the user into autofilling a, say, cardholder-name field. The attacker has embedded an invisible iframe that contains a cross-origin payment form and has enabled `shared-autofill` in that child document.
Due to `shared-autofill`, the autofill in the parent document may fill in the attacker's payment form.
The attacker then click-jacks the user into submitting the payment form.

The attack relies on `shared-autofill` because the attacker cannot access the payment form directly.
However, the attacker can likely also click-jack the user into autofilling the form in the iframe directly – entirely without `shared-autofill`.
Also, the attacker arguably has much simpler means to steal the payment information by manipulating the parent document directly.

**Case 2.**
The attacker controls a document where `shared-autofill` is enabled.
They steal autofilled values.

The attacker adds a, say, payment form to the document they control and where `shared-autofill` is enabled.
They speculate on an autofill happening in another document.
Due to `shared-autofill`, the autofill in that document may also fill the attacker-controlled payment form.

The attack relies on `shared-autofill` because the attacker cannot access the document where the autofill happened directly.
Unless the attacker-controlled frame is the main frame, this requires a dangerous use of `shared-autofill` in the parent frame of the attacker-controlled frame.

**Mitigation.**
A parent document's policy can effectively override any malicious `allow="shared-autofill"` attributes (because the document policy [takes precedence](https://w3c.github.io/webappsec-permissions-policy/#algo-define-inherited-policy-in-container) over the container policy).
For example, the HTTP header `Permissions-Policy: shared-autofill=(self "https://psp.example")` implicitly prevents all origins except for `https://psp.example` from inheriting shared-autofill.
This mitigates both cases except the scenario of Case 2 where the attacker controls the main frame.

## Demo

Chrome 93.0.4577.0 or later includes an experimental implementation when run with the flags `--enable-features=AutofillAcrossIframes,AutofillSharedAutofill`.
There's a [test page](https://schwering.github.io/shared-autofill/form.html) and a [video of the user experience](https://www.youtube.com/watch?v=YDH19FygOa4).
