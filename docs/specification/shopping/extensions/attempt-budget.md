<!--
   Copyright 2026 UCP Authors

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Attempt Budget Extension

## Overview

Idempotency stops a retried request from having its effect twice, but a new
Complete Checkout needs a fresh idempotency key even when the payload is
unchanged (see [Idempotency](../checkout/index.md)). Nothing in the core
checkout capability therefore bounds how many times a Platform may resubmit
payment on one checkout.

The Attempt Budget extension lets a Business cap the number of **failed**
Complete Checkout attempts per checkout session, report the counter, and refuse
further completions before they reach payment processing. The refusal uses the
existing `requires_escalation` status and `continue_url`, so the Buyer finishes
the purchase and a looping Platform stops.

The counter is a safety rail for well-behaved loops, not a fraud decision, and
it does not replace the Business's own cross-session velocity controls.

## Discovery

<!-- ucp:example schema=profile def=business_schema extract=$.capabilities target=$.ucp.capabilities -->

```json
{
  "capabilities": {
    "dev.ucp.shopping.attempt_budget": [
      {
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/attempt-budget",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/attempt_budget.json",
        "extends": "dev.ucp.shopping.checkout",
        "config": { "complete": { "max_failed": 3 } }
      }
    ]
  }
}
```

## Schema

### Business configuration

{{ extension_schema_fields('attempt_budget.json#/$defs/business_config', 'shopping/extensions/attempt-budget') }}

### Attempts (response only)

{{ extension_schema_fields('attempt_budget.json#/$defs/attempts', 'shopping/extensions/attempt-budget') }}

{{ extension_schema_fields('attempt_budget.json#/$defs/attempt_counter', 'shopping/extensions/attempt-budget') }}

## Counting

A **failed completion attempt** is a Complete Checkout operation that:

1. is not an idempotent replay of an earlier request (same idempotency key,
    identical body),
2. was forwarded to payment processing, and
3. ended in `payment_failed`, or a handler-specific decline that the Business
    maps to it.

These are **not** counted: a replay; a request rejected before payment
processing (validation, signature, stale session, `complete_in_progress`); an
authentication or Buyer-action outcome that is not a failure (a challenge
returned as `requires_escalation` or an Action); and the budget-exhausted
refusal itself.

## Exhaustion

When `used` equals `limit`, the Business refuses Complete Checkout without
calling payment processing:

<!-- ucp:example schema=shopping/attempt_budget def=dev.ucp.shopping.checkout op=read -->

```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "success", "payment_handlers": {} },
  "id": "chk_123",
  "status": "requires_escalation",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_1",
      "item": { "id": "item_roses", "title": "Red Roses", "price": 2000 },
      "quantity": 1,
      "totals": [
        { "type": "subtotal", "amount": 2000 },
        { "type": "total", "amount": 2000 }
      ]
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 2000 },
    { "type": "total", "amount": 2000 }
  ],
  "links": [],
  "continue_url": "https://shop.example/checkout/chk_123/resume",
  "attempts": { "complete": { "used": 3, "limit": 3 } },
  "messages": [
    {
      "type": "error",
      "code": "attempt_limit_reached",
      "severity": "requires_buyer_review",
      "path": "$.attempts.complete",
      "content": "Too many failed payment attempts. Continue with the buyer to finish this order."
    }
  ]
}
```

## Normative requirements

1. A Business that advertises this capability **MUST** count failed completion
    attempts per checkout session as defined above, and **MUST** apply the count
    atomically so that concurrent Complete Checkout requests cannot exceed
    `limit` payment-processing attempts.
2. The Business **MUST** include `attempts.complete.used` and
    `attempts.complete.limit` in every checkout response for the session once
    the capability is negotiated, **MUST** keep `limit` constant for the life of
    the session, and **MUST** ignore any `attempts` value received in a request.
3. When `used` equals `limit`, a Complete Checkout request **MUST NOT** be
    forwarded to payment processing. The Business **MUST** respond with the
    checkout in `status: requires_escalation`, a `continue_url`, and an error
    message with `code: attempt_limit_reached`,
    `severity: requires_buyer_review` and `path: $.attempts.complete`.
4. A replay of an earlier request (same idempotency key, identical body)
    **MUST** return the original result and **MUST NOT** change `used`.
5. After exhaustion the Business **MUST** still honor Get Checkout and Cancel
    Checkout, and **MUST NOT** return the session to `ready_for_complete` as the
    result of a Platform-driven Update. The session proceeds only through the
    Buyer at `continue_url`, or by cancellation or expiry.
6. A Platform that negotiated the capability **SHOULD** stop submitting
    completions when `used + 1 == limit` unless the Buyer has just supplied a
    different instrument or asked to retry. It **MUST** treat
    `attempt_limit_reached` as a request for Buyer handoff and **MUST NOT** work
    around it by creating a new checkout for the same intent without the Buyer.
7. A Business **SHOULD** also apply its own cross-session velocity controls.
8. The refusal and the `attempts` object **MUST NOT** carry decline detail
    beyond what the failed attempt already returned.

The `severity` is `requires_buyer_review`, not `recoverable`: the
[standard errors table](../checkout/index.md#standard-errors) recommends
`recoverable` for standard errors, and this one is the exception because no
Platform-side action can recover it.
