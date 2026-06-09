# bLIP-XXXX: Cashback in BOLT 12 Invoices

```
bLIP: XXXX
Title: Cashback in BOLT 12 Invoices
Status: Draft
Author: Vincenzo Palazzo <vincenzopalazzodev@gmail.com>
Created: 2026-06-09
License: CC0
```

## Abstract

This bLIP defines a small set of experimental TLV fields that let a
merchant *declare*, inside a BOLT 12 `invoice`, a cashback (rebate) it
commits to giving the payer. The cashback can be a flat number of
millisatoshis, a proportion of `invoice_amount` (basis points), or both.
Because the invoice is signed by `invoice_node_id`, the declaration is
merchant-authenticated.

Two redemption models are supported, distinguished by
`invoice_cashback_mode`:

- **Instant (discount):** the payer's wallet deducts the cashback and pays
  the *net* amount; the merchant — being the recipient and the party that
  committed to the cashback — accepts the reduced amount for that invoice.
- **Deferred (payback):** the payer pays in full and the merchant later
  returns the cashback out of band, addressing the payer by their
  BOLT 12 Contact (bLIP 42) rather than via any field embedded in the
  invoice.

The fields are additive and odd-typed: a wallet that does not understand
them simply pays the invoice as a normal BOLT 12 invoice.

## Motivation

Cashback, rebates, and loyalty rewards drive consumer adoption in
card-based payments, but Lightning has no standard way for a merchant to
make a verifiable cashback commitment at the moment of payment.

BOLT 12 is the right place to express it: the `invoice` is the signed,
per-payment artifact carrying `invoice_amount`, against which a
proportional cashback resolves. A handful of named TLV fields let any
wallet show the user their rebate before they pay, retain a signed record
of the promise, and — for the instant model — settle the discounted amount
in a single payment with no second flow and no extra round trips.

Crucially, the payer's identity for any *deferred* payback is already
solved by BOLT 12 Contacts (bLIP 42): the merchant addresses the rebate to
the payer's contact. There is therefore no need to embed a redemption offer
in the invoice itself.

## Redemption Models

### Model A — Instant cashback (underpayment)

The merchant issues an invoice whose `invoice_amount` is the *gross* price
and attaches the cashback fields with `invoice_cashback_mode` = 1. A
supporting wallet computes:

```
net_msat = invoice_amount.msat - resolved_cashback_msat
```

and pays `net_msat` (sets `total_msat` to it). The merchant's node, which
issued and signed the commitment, accepts the payment even though it is
below the displayed `invoice_amount`.

This is sound within the protocol because the BOLT 4 final-node checks only
require `total_msat` to be consistent with what actually arrives
(`total_msat` == `amt_to_forward` for single-path; set total >= `total_msat`
for MPP). The "minimum amount it will accept" expressed by `invoice_amount`
is a *merchant-policy* value, not a BOLT 4-enforced floor — so the merchant
is free to accept less for an invoice on which it has committed cashback.

A non-supporting wallet doesn't know to deduct, so it pays the full gross
`invoice_amount` and simply forgoes the discount. The merchant accepts both
(anything `>= net_msat`).

### Model B — Deferred cashback (payback via BOLT 12 Contacts)

The merchant issues a normal invoice with `invoice_cashback_mode` = 0 (or
omitted) and the cashback fields as a signed promise. The payer pays in
full. Later, the merchant sends the cashback back to the payer, addressing
them by their BOLT 12 Contact (bLIP 42). The signed invoice + cashback TLVs
give the payer a record to reconcile the inbound rebate against.

This can be built **today** on top of bLIP 42 with no protocol changes — a
merchant simply decides to pay back a percentage of sats to a known
contact. It does not always work, though: it requires the payer to have
shared a contact, the payer to be reachable (online, with a usable route)
when the merchant pays back, the merchant to actually initiate the outbound
payment, and it carries a trust gap (the merchant can renege; the only
recourse is the signed commitment, off-protocol).

```
  ┌────────┐                          ┌──────────┐
  │  Alice │                          │   Bob    │
  │ (payer)│                          │(merchant)│
  └───┬────┘                          └────┬─────┘
      │      invoice_request               │
      │───────────────────────────────────>│
      │   invoice (+ cashback TLVs, signed) │
      │<───────────────────────────────────│
      │                                     │
      │ Model A:  pay net = gross − cashback│
      │───────────────────────────────────>│   merchant accepts < gross
      │                                     │
      │ Model B:  pay gross in full         │
      │───────────────────────────────────>│
      │     …later, payback to Alice's      │
      │        BOLT 12 Contact (bLIP 42)    │
      │<────────────────────────────────────│
```

## Cashback TLV Fields

All fields use the invoice-specific experimental TLV range
(3000000000–3999999999). This range is *not* mirrored from the
`invoice_request` (the mirrored ranges are 0–159 and 1000000000–2999999999),
which is correct: the cashback is a unilateral merchant declaration the
payer did not request. All types are **odd**, so readers that do not
understand them ignore them.

### `invoice_cashback_amount`

1. type: 3000000001 (`invoice_cashback_amount`)
2. data:
    * [`tu64`:`msat`]

A flat cashback amount, in millisatoshis. MAY be combined with
`invoice_cashback_bps`.

### `invoice_cashback_bps`

1. type: 3000000003 (`invoice_cashback_bps`)
2. data:
    * [`tu32`:`basis_points`]

Cashback in basis points (1 bps = 0.01%) of `invoice_amount`.`msat`. The
resolved cashback is `floor(invoice_amount.msat * basis_points / 10000)`.

### `invoice_cashback_expiry`

1. type: 3000000005 (`invoice_cashback_expiry`)
2. data:
    * [`tu32`:`seconds_from_creation`]

For Model B, the number of seconds after `invoice_created_at` within which
the merchant commits to paying the cashback back. Informational for
Model A. If absent, merchant policy applies.

### `invoice_cashback_mode`

1. type: 3000000007 (`invoice_cashback_mode`)
2. data:
    * [`byte`:`mode`]

`0` = deferred payback via BOLT 12 Contacts (default if absent), `1` =
instant discount (payer pays the net amount). A wallet MUST NOT underpay
unless `mode` is `1`.

## Requirements

### The Merchant (Cashback Issuer)

A node that commits to cashback in an invoice:

- MUST set at least one of `invoice_cashback_amount` or
  `invoice_cashback_bps` if it sets any cashback field.
- MUST place all cashback fields in the inclusive range
  3000000000–3999999999 and MUST use odd type numbers.
- MUST NOT set cashback fields in any range mirrored from the
  `invoice_request` (0–159, 1000000000–2999999999).
- if it sets `invoice_cashback_bps`:
  - MUST resolve the cashback as
    `floor(invoice_amount.msat * basis_points / 10000)`.
- when both `invoice_cashback_amount` and `invoice_cashback_bps` are
  present:
  - MUST honor the larger of the two unless it documents a different
    combination policy out of band.
- if `invoice_cashback_mode` is `1` (instant):
  - MUST accept a settled payment whose total is `>= invoice_amount.msat −
    resolved_cashback_msat` for that invoice.
  - MUST NOT require a separate redemption flow.
- if `invoice_cashback_mode` is `0` or absent (deferred):
  - SHOULD address the payback to the payer's BOLT 12 Contact (bLIP 42).
  - SHOULD pay the cashback before `invoice_cashback_expiry` if set.
  - MUST be able to correlate the payback to the original payment (e.g.
    via `invoice_payment_hash` or a payer/payee proof).

### The Payer (Cashback Recipient)

A wallet that receives an invoice with cashback fields:

- if it recognizes the cashback TLV fields:
  - SHOULD display the resolved cashback to the user before confirming
    payment.
  - SHOULD retain the signed invoice as evidence of the commitment.
  - if `invoice_cashback_mode` is `1` (instant):
    - SHOULD set `total_msat` to `invoice_amount.msat −
      resolved_cashback_msat`.
    - MUST NOT underpay by more than `resolved_cashback_msat`.
  - if `invoice_cashback_mode` is `0` or absent (deferred):
    - MUST pay `invoice_amount` in full.
    - SHOULD ensure a BOLT 12 Contact is shared with the merchant so the
      payback can be addressed.
    - MUST NOT treat the cashback as received until the payback settles.
- if it does not recognize the cashback TLV fields:
  - MUST treat the message as a normal invoice (per the odd-type-is-ok
    rule) and pay `invoice_amount` in full.

## Relationship to BOLT 12 Contacts and Payee Proofs

Deferred cashback (Model B) reuses BOLT 12 Contacts (bLIP 42) as the payer
identifier, so no redemption offer needs to be embedded in the invoice. For
merchants that want a stronger guarantee that the redeeming/paid party is
the one entitled to the rebate, this bLIP composes with BOLT 12
payer/payee-proof work (cf. lightning/bolts#1295): the proof that the
original invoice was paid authorizes the payback. Absent a proof mechanism,
the merchant correlates `invoice_payment_hash` and retains per-payment
state.

## Backward Compatibility

Fully backward compatible. All new fields are in the BOLT 12
invoice-specific experimental range and use odd type numbers, so existing
implementations ignore them and pay the invoice in full. In Model A a
non-supporting wallet pays the gross `invoice_amount` (forgoing the
discount); the merchant accepts both gross and net. In Model B nothing
about the payment flow changes — the payback is an ordinary, separate
BOLT 12 payment to a contact.

## Rationale

**Why drop an embedded redemption offer?**
For deferred cashback the payer's destination is already solved by BOLT 12
Contacts (bLIP 42); embedding a per-invoice claim offer would duplicate
that and leak that the node runs a cashback program. The contact is the
identifier.

**Why allow underpayment for instant cashback?**
The merchant is the recipient and the committing party. `invoice_amount` is
defined as the minimum the merchant will accept, which is merchant policy,
not a BOLT 4-enforced floor; BOLT 4 only requires `total_msat` to be
consistent with the amount that arrives. So a merchant can safely accept
`net = gross − cashback` for an invoice it has flagged with cashback,
delivering the rebate atomically in a single payment with no second flow.
(Note this is the inverse of a payer trying to underpay against the
merchant's will, which BOLT 4 / merchant policy correctly rejects.)

**Why keep a gross `invoice_amount` plus a discount, rather than just
invoicing the net?**
Surfacing the gross price preserves the "list price + rebate" semantics
that make cashback legible to users and auditable against the signed
commitment. Setting `invoice_amount` directly to the net is the spec-cleanest
alternative and is noted in Open Questions.

**Why both a flat amount and a basis-points field?**
Flat suits fixed promotions; basis points suit proportional loyalty.
Allowing both with a defined resolution rule covers common programs.

## Open Questions

1. **Net-amount alternative**: For instant cashback, should the merchant
   instead set `invoice_amount` to the net and keep the cashback fields
   purely informational ("you saved X")? That avoids underpayment entirely
   at the cost of hiding the gross price.

2. **Combination policy**: When both flat and bps are present, is "the
   larger of the two" the right default, or the sum? Should it be an
   explicit flag?

3. **Trust model for deferred payback**: Model B works today but the
   merchant can renege. Should a payee proof be required to make the
   commitment enforceable, or is the signed invoice + reputation enough?

4. **Reachability for payback**: Deferred payback needs the payer reachable
   when the merchant pays. Should the bLIP recommend retry/expiry behavior,
   or leave it to wallet policy?

5. **Currency and caps**: Should there be a fiat-denominated cashback
   variant (mirroring `offer_currency`) and an `invoice_cashback_max` to
   bound proportional cashback on large invoices?

## Reference Implementation

*TODO: link to reference implementation once available.*

## Copyright

This bLIP is licensed under CC0.
