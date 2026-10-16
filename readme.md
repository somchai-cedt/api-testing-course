# Auntie Som Lab — automated Bruno API audit

An importable Bruno collection for the supplied Auntie Som Noodle Stall Flask API. It checks business rules with executable JavaScript tests, extracts the login token automatically, calculates expected prices independently, and checks inventory before and after order attempts. **No changes were made to `app.py`.**

## Verified result

Tested on 2026-10-16 with Bruno CLI **4.2.1**, Python **3.12**, and Flask **3.1.3**, using the backend from `api-testing-course-main.zip`.

- 23 requests executed; no request setup or transport errors.
- 9 requests passed all their checks; 14 requests had failing checks.
- **50 of 79 JavaScript test checks passed; 29 failed.**
- Bruno exited with code **1**, correctly reporting the broken business rules.

## Bugs discovered and how the tests detect them

| Bug | Observed behavior | Automated detection |
| --- | --- | --- |
| B01: Incorrect order total | Two Tom Yum Noodles cost 50 each, but POST returns **95**, and GET persists **95**, instead of **100**. Source subtracts an unexplained 5. | Requests **08–09** compare both POST and GET totals against `menu.price * quantity`, calculated before ordering. They do not use the incorrect POST total as the expected GET total. |
| B02: Missing order returns success status | GET of `ORD-does-not-exist-lab` returns **200** with `{"error":"Order not found"}`. | Request **11** requires **404** and an error object with no order fields. |
| B03: Zero, negative, and fractional quantities accepted | Quantity **0** creates an order with total **-5**; **-1** creates an order and increases stock; **0.5** creates an order and makes stock fractional. | Requests **12–14** require a validation error and no order ID/created status. They also compare the complete menu before and after each attempt. The zero test detects order creation even though stock does not change. |
| B04: Invalid quantity types cause server errors | String `"1"`, missing quantity, and `null` produce **500** HTML error pages. | Requests **15–17** require **400 or 422**, a JSON error object, and unchanged stock. A 500 cannot pass as an acceptable rejection. |
| B05: Invalid request shapes cause server errors | Array `itemId: []` and array JSON body `[]` produce **500**. | Requests **19–20** require a validation error and unchanged inventory. |
| B06: Bearer authentication syntax is not enforced | `Authorization: <valid token>` is accepted without the required `Bearer ` prefix, creating an order and reducing stock. This requires a real issued token; random tokens remain rejected. | Request **21** deliberately uses the saved token without the prefix, requires **401**, rejects an order-shaped response, and verifies that stock did not change. |
| B07: Stock limit is not enforced | The code checks only whether stock is already zero or negative. It accepts a quantity larger than remaining stock, then makes stock negative. | Request **22** reads live stock and sends `floor(stock) + 1`. It requires **400 or 409**, an error object, and unchanged inventory. Request **23** separately checks integer stock and nonnegative stock. |

In the recorded sequential run, item 2 starts with stock **1**. The buggy negative request raises it to **2**, the fractional request reduces it to **1.5**, the malformed auth request reduces it to **0.5**, and the overstock request sends quantity **1**, leaving **-0.5**. Each mutation test compares its own live before/after snapshots, so later failures do not depend on assuming earlier invalid requests were harmless.

Passing controls demonstrate that login extraction, valid bearer auth, bad-password rejection, unknown-user rejection, missing-auth rejection, random-token rejection, unknown-item rejection, exact stock deduction, missing-item rejection, and already-sold-out rejection work. The valid order request passes its creation and stock checks while failing its independent price check.

The provided reference example also shows a total of 45 for an item priced at 50. This audit treats menu price multiplied by quantity as the intended rule because no discount is specified. A documented discount would require revising that expectation.

## Files

```text
Auntie-Som-Lab/
  bruno.json                  # Open this folder as a Bruno collection
  01-Login.bru ... 23-Final-inventory-integrity.bru
  environments/
    local.bru                 # Ready to use; ignored by Git
README.md
.gitignore
```

## Environment and script behavior

| Variable | Default / purpose |
| --- | --- |
| `baseUrl` | `http://localhost:5000`; every request and scripted menu lookup uses it. The helper overrides it for its local server. |
| `username`, `password` | Public lab credentials `student` / `hungry`. |
| `accessToken` | Blank initially. Automatically set by the login post-response script and used in subsequent bearer headers. Cleared by request 23. |
| `invalidToken` | Deliberately invalid value used for the negative auth test. |
| `primaryItemId`, `edgeItemId` | `1`, `2`; the first is used for the valid exact-stock order, the second for input and overstock cases. |
| `validQuantity` | `2`, matching fresh stock for item 1. |
| `expectedPrimaryPrice`, `expectedEdgePrice` | `50`, `45`, checked against the advertised fixture. |

`orderId`, expected total, request quantity, and before-menu snapshots are runtime variables. The token is an **environment variable**, as required. A snapshot is taken before each order attempt, and an additional scripted GET verifies inventory afterward. Setup guards report missing tokens or stale fixtures as setup errors rather than silently skipping tests.

Bruno CLI 4.x persists environment writes. Keep the generated **local.bru** ignored by Git; commit only the blank **local.example.bru** template. If a run is interrupted before request 23, clear the token in your local environment before sharing anything. The supplied `.gitignore` excludes local environments and new reports. The sanitized verified reports in `evidence/` are intentionally included.

Negative validation policies accept the documented auth/item statuses and reasonable unspecified validation statuses: missing/invalid auth **401**, unknown item **404**, invalid quantity or malformed JSON shape **400/422**, insufficient stock **400/409**, missing order **404**. Successful creation accepts **200/201**. Error bodies must be objects with a nonempty string `error` and no created order. If a future API defines a different error schema, update these contract assertions intentionally.

This suite does not wait an hour to verify actual token expiry, simulate concurrent orders, or assert ownership on GET orders; the supplied reference does not require GET authentication. Advertising `expiresIn: 3600` is checked, but that does not prove expiry enforcement.