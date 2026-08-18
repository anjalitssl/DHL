# DHL Express Germany Expansion — Consolidated Gap & Readiness Analysis
**SRS (Stratus Returns Service) — FedEx (USA, current) vs. DHL Express (Germany, target)**

**Jira:** STFSVC-2656 — Geo Support: FedEx Payload → Country-Specific Configuration (Germany, Netherlands, France, Spain, Italy, UK, Canada)
**Last consolidated:** August 18, 2026
**Supersedes:** `FedEx-vs-DHL-Gap-Analysis-Complete.md` + `RAID-DHL-Return-Label-Dependencies.md` (content merged, deduplicated, and reorganized below — nothing substantive removed, only consolidated).

---

## 1. Executive Summary

SRS is expanding returns support to Germany using **DHL Express**, alongside the existing **FedEx** flow (USA). This document is the single source of truth for that gap analysis: what's confirmed from code, what's confirmed from DHL material, and what is still open — by owner.

**Critical finding:** DHL Express MyDHL API has **no dedicated "Return API."** Returns are handled as a standard **Shipment**, with shipper/receiver roles reversed:
- **Customer = Shipper** (origin)
- **HP/Warehouse = Receiver** (destination)

**Scope:** Germany only. The existing US FedEx flow is **not replaced** — this is an additive change (`InitReturnFedexFlow` stays untouched; a new `InitReturnDhlFlow` is added alongside it, per the existing strategy-pattern architecture — see §6).

**Three critical requirements driving this analysis:** (1) Tracking URL, (2) Token/Authentication, (3) RMA handling — see §4.

---

## 2. At-a-Glance: FedEx (Current) vs. DHL (Target)

| Aspect | FedEx (USA, current) | DHL Express (Germany, target) |
|---|---|---|
| **API type** | Dedicated RMA/Return API (`POST /api/v2/rmas`) | Generic **Shipment API** (`POST https://express.api.dhl.com/mydhlapi/shipments`) — no dedicated return endpoint |
| **Authentication** | OAuth2 Password Grant, Bearer token, `FedexTokenHolder` (expiry/refresh) | HTTP Basic Auth — stateless, no token lifecycle |
| **Credentials** | 5: `FEDEX_CLIENT_ID`, `FEDEX_CLIENT_SECRET`, `FEDEX_ORG_NAME`, `FEDEX_PASSWORD`, `FEDEX_XORG_NAME` | 3: `DHL_API_KEY`, `DHL_API_SECRET`, `DHL_ACCOUNT_NUMBER` |
| **RMA field** | Root-level `rmaNumber` | `customerReferences[typeCode="RMA"].value` (NOT `typeCode="CU"` — that's a different reference type, Consignor reference) |
| **Address** | Lookup by `shipToAddressId` (config ID only) | Full address object required (`postalAddress`, `contactInformation`) |
| **Items model** | Item-centric: SKU + quantity | Package-centric: weight + dimensions |
| **Tracking number** | 12-digit numeric | 10-digit alphanumeric (top-level `shipmentTrackingNumber`) |
| **Label** | `labelURL` — pull model, fetchable URL | `documents[].content` — push model, inline Base64 (PDF/ZPL) |
| **Tracking URL** | SRS-constructed: `fedex.trackingUrl` + tracking number | Same pattern reusable; DHL's exact production URL format needs confirmation |
| **Code flow** | `ReturnDetailsController` → `ReturnDetailsService.sendReturn()` → `resolveInitReturnFlow()["Fedex"]` → `InitReturnFedexFlow` → `FedexClient.createRmaS4()` | Proposed: same loop → `["DHL"]` → new `InitReturnDhlFlow` → new `DhlClient` |

Full FedEx code evidence (file/line references, JSON payloads) is in **Appendix A**.

---

## 3. DHL Express API Structure

```
POST https://express.api.dhl.com/mydhlapi/shipments
```

**Sample request (from DHL-provided screenshots):**
```json
{
  "productCode": "U",
  "plannedShippingDateAndTime": "2026-08-05T16:00:00GMT+01:00",
  "pickup": { "isRequested": false },
  "accounts": [{ "number": "304xxxxxx", "typeCode": "shipper" }],
  "customerReferences": [{ "value": "20219876", "typeCode": "CU" }],
  "outputImageProperties": {
    "encodingFormat": "pdf",
    "imageOptions": [{ "typeCode": "qr-code", "isRequested": true }]
  },
  "customerDetails": {
    "shipperDetails": {
      "postalAddress": { "cityName": "Prague", "countryCode": "CZ", "postalCode": "15000", "addressLine1": "Vaclavske namesti" },
      "contactInformation": { "phone": "+420220300111", "companyName": "DHL", "fullName": "DHL Express", "email": "dhl@dhl.com" }
    },
    "receiverDetails": {
      "postalAddress": { "cityName": "Barcelona", "countryCode": "ES", "postalCode": "08001", "addressLine1": "Rbelot" },
      "contactInformation": { "phone": "0795632404", "companyName": "Test Dhlapi", "fullName": "Test Dhlapi", "email": "test@dhlapi.com" }
    }
  },
  "content": {
    "unitOfMeasurement": "metric",
    "isCustomsDeclarable": false,
    "incoterm": "DAP",
    "description": "Testing Equipment",
    "packages": [{ "weight": 1.552, "dimensions": { "length": 20, "width": 12, "height": 8 } }]
  },
  "valueAddedServices": [{ "serviceCode": "PZ" }, { "serviceCode": "PT" }]
}
```

**Sample response:**
```json
{
  "shipmentTrackingNumber": "1234567890",
  "trackingUrl": "https://mydhl.express.dhl/...",
  "packages": [{ "referenceNumber": 1, "trackingNumber": "JD999999999999999999" }],
  "documents": [{ "imageFormat": "PDF", "content": "JVBERi0xLjQK..." }]
}
```

> ⚠️ Note on the sample: it uses `customerReferences[typeCode="CU"]`. Per DHL's reference-data guide, **`CU` = Consignor reference (NOT the RMA)**. The actual RMA must use `typeCode="RMA"`. This correction is already reflected in §4 and the master gap table.

---

## 4. Three Critical Requirements

### 4.1 Tracking URL
- **FedEx:** 12-digit numeric (e.g. `794672887344`); URL = `fedexConfig.getTrackingUrl() + trackingNumber` (constructed by SRS, not returned by FedEx).
- **DHL:** 10-digit alphanumeric (or longer at package level, e.g. `JD999999999999999999`); customer-facing URL sample: `https://www.dhl.com/express/track?AWB=<number>` — **not yet validated as the correct production URL, flagged as an open item**.
- DHL also exposes dedicated tracking-query endpoints (test/prod) for status/checkpoint retrieval, with a much richer set of event codes than FedEx exposes today — see Appendix B for the full code table (`OK` = Delivery is the key success code; others cover customs holds, mis-sorts, refused delivery, etc.).
- **Gap:** URL template is carrier-specific; the same SRS-side "config prefix + tracking number" construction pattern is reusable, but DHL's config value must be confirmed before hardcoding.

### 4.2 Authentication
- **FedEx:** OAuth2 Password Grant, Bearer token via `FedexTokenHolder` (tracks expiry + refresh).
- **DHL:** HTTP Basic Auth (`Authorization: Basic <base64(apiKey:apiSecret)>`) — stateless, **no token management needed at all**. This is a genuine simplification vs. FedEx.
- Credential provisioning is a DHL-side business dependency: DHL requires HP to supply account name, contact, email, phone, address, city, postcode before they issue API credentials.

### 4.3 RMA (Return Merchandise Authorization)
- **FedEx:** root-level `rmaNumber`, sourced from `retReturnNumber`, validated ≤20 chars.
- **DHL:** `customerReferences[typeCode="RMA"].value` (array element, not root field).
- **Reusability: 95%.** RMA is received as input (not generated by SRS either way), stored the same way, retrieved the same way, validated the same way — **only the payload placement differs** (root field vs. array element). It is a plain business identifier with no authentication role for either carrier.

| Reusability check | FedEx → DHL |
|---|---|
| Value | ✅ Same `retReturnNumber` |
| Storage | ✅ Same MongoDB field |
| Retrieval | ✅ Same `getRetReturnNumber()` |
| Validation | ✅ Same business rules |
| Payload structure | ❌ Root-level (FedEx) vs. array/typeCode (DHL) — the only real difference |

---

## 5. Field-by-Field Mapping

### Root-level fields
| Purpose | FedEx | DHL | Gap |
|---|---|---|---|
| RMA/Return ID | `rmaNumber` (string, max 20) | `customerReferences[typeCode='RMA'].value` | Medium — different structure, same value |
| Product/service | N/A | `productCode` | High — new required field, correct value unconfirmed |
| Shipping date | N/A | `plannedShippingDateAndTime` | High — new required field |
| Pickup | N/A | `pickup.isRequested` | Medium — simple boolean, business decision needed (§8.4) |
| Account info | N/A (from config) | `accounts[].number/typeCode` | Medium — new structure |
| Email notification | `emailNotification` (boolean) | Shipment notification construct confirmed to exist (method + email) — exact field name/mapping TBD | Low-Medium — capability confirmed, only field mapping + account enablement open (§8) |
| Output format | N/A (implicit PDF) | `outputImageProperties` | Medium — new structure |

### Customer/address
| Purpose | FedEx | DHL | Gap |
|---|---|---|---|
| Email | `customer.emailAddress` | `shipperDetails.contactInformation.email` | Path more nested |
| Name | `firstName` + `lastName` | `contactInformation.fullName` | Two fields → one |
| Phone | **Not captured today** | `contactInformation.phone` | **Missing in FedEx model — must add or make optional** |
| Company | Not captured | `contactInformation.companyName` | Missing in FedEx model |
| HP return address | `shipToAddressId` (lookup ID only) | `receiverDetails.postalAddress.*` (full object) | **Major** — DHL needs full address, FedEx only needs an ID |

### Order/items
| Purpose | FedEx | DHL | Gap |
|---|---|---|---|
| Items | `orders[].items[]` — SKU + quantity | `content.packages[]` — weight + dimensions | **Major structural difference** — item-centric vs. package-centric |
| Weight/dimensions | **Not in `Item` model** | Required | **Critical missing data** — need new field, default/catalog lookup, or fetch elsewhere |
| Return reason | `items[].returnItemInfo.returnReason` | `content.description` (free text) | Can map |

---

## 6. Carrier Selection Architecture — Global Switch Today, Germany-Only Needs Enhancement

**Two distinct questions here — kept separate deliberately:**

### 6.1 Can a new carrier bean be added? — Yes, no architecture change needed
- `InitReturnFlow` (interface: `initReturn()` + `getFlowType()`) is a strategy pattern; Spring auto-collects **all** beans implementing it into `List<InitReturnFlow>`.
- `InitReturnFedexFlow.getFlowType()` returns `"Fedex"` — the only place that literal is used as a selector.
- `InitReturnNoFlow` already proves this is a genuine, proven multi-bean pattern in production, not theoretical.
- `resolveInitReturnFlow()` (`ReturnDetailsService.java:242-249`) loops over all beans and matches by string equality against `launchDarklyClient.getShipmentProviderFlag()`.
- **Conclusion:** adding `InitReturnDhlFlow` with `getFlowType() → "DHL"` requires **zero changes** to this loop. This part is low-risk and additive.

### 6.2 Can DHL be scoped to Germany only while FedEx serves everywhere else? — No, not without enhancement
This was a specific question raised on the architect review call. Confirmed from code:
- `getShipmentProviderFlag()` evaluates against a **single static `LDUser` singleton**, built once at startup (`LaunchDarklyConfig.getLDUser()` → `new LDUser(ldUserKey)` from one config value). This same object is reused for **every** flag check, for every return, for the life of the running instance.
- **No per-request attribute** (country, tenantId, subscriptionId, etc.) is ever attached to this context anywhere in the codebase (confirmed — zero matches for `country`/`Country`/`tenantId` in `src/main/java`; zero usage of LaunchDarkly's newer per-attribute `LDContext` model — this integration uses only the older single-key `LDUser`).
- SRS's own `ReturnDetailsDocument` model **also has no country field today** — `countryCode` only appears in inbound Solace event test fixtures and mock-service samples, never in the persisted return document.
- The selection **loop itself is not the blocker** (§6.1) — the gap is entirely in **how the flag's value is determined** (one global value for the whole app), not in how it's consumed.

**What "Germany only" would require:**
1. Either a per-request LaunchDarkly context carrying a country attribute + a targeting rule, **or** a simpler code-level `if` on country that bypasses LaunchDarkly targeting for this decision.
2. A reliable country value captured onto the return document (sourced from the inbound order/invoice event, which does carry `countryCode`, but it's not persisted onto the return document today).
3. Business confirmation of which address (customer/ship-from vs. warehouse/receiver) decides the carrier.
4. If the LD-context approach is chosen: review whether changing `LaunchDarklyClient`'s shared `ldUser` affects any of the ~30 *other* flags that also reuse the same static object today.

**Bottom line:** today's mechanism supports a **global** carrier switch (one value, whole fleet) with zero code change. It does **not** support "DHL for Germany, FedEx everywhere else" simultaneously — that is a genuine code gap requiring both an architecture decision and new data capture, not just a business decision.

### 6.3 Confirmed: No Inbound Carrier/Service Value From TAO or Any Downstream Caller Today
A direct question was raised on the architect review call: *who selects the carrier on the incoming request, what value is passed today (e.g. "FedEx Ground"), and does the downstream caller (TAO) need to send anything differently for Germany?* Confirmed from code:
- **No reference to "TAO" exists anywhere in `src/main`** — SRS's inbound "create return"/generate-label endpoint is `POST` on `ReturnDetailsController`, bound to `InitiateReturnRequest` (`api/dto/InitiateReturnRequest.java`).
- **`InitiateReturnRequest` has zero carrier/shipping fields.** Its full field set is: `returnOrderId`, `fulfillmentOrderId`, `userId`, `returnReason`, `eClaimsID`, `timeoutInDays`, `parts` (list), `orderAttributionId`, `isWriteOffReturn`. There is no `carrier`, `carrierName`, `shippingVendor`, `serviceType`, or `shipVia` field — **whoever calls this endpoint today (TAO or otherwise) cannot and does not influence carrier selection.**
- There is also **no hardcoded FedEx service-level string** (e.g. `"FedEx Ground"`) anywhere in `src/main` — FedEx's account/service level is implicit in the FedEx account configuration itself, not passed or selected in code (consistent with §9's Rating/Capability row).
- Carrier is instead resolved entirely server-side: `ReturnDetailsService.java:195` → `payload.setShipmentProvider(launchDarklyClient.getShipmentProviderFlag())` (global flag, default `"None"`) → `resolveInitReturnFlow()` (`ReturnDetailsService.java:242-249`) matches that value against each bean's `getFlowType()`.
- **What this means for Germany:** no inbound contract change is required for the downstream caller — **TAO (or whichever system calls this endpoint) does not need to send anything differently.** The only change needed is server-side: the LaunchDarkly flag value/targeting (see §6.2 for why global-only vs. Germany-only is still an open gap) and the new `InitReturnDhlFlow` bean's `getFlowType() → "DHL"`.
- This closes out §11 SRS Team item (carrier-selection request-field question) definitively: **confirmed no inbound field exists; nothing to change on the caller side.**

---

## 7. Label Handling — FedEx URL Model vs. DHL Inline Base64

### 7.1 Current FedEx behavior (confirmed from code)
- Label arrives as a **URL** (`labelURL`) inside `rma.shipments[0].label`, persisted to MongoDB as part of `ReturnDetailsDocument`.
- It is **never read, downloaded, or forwarded** by any SRS business logic. Only the sibling `trackingNumber` is actively used (copied to `Item.trackingNumber` and to the outbound EMS event).
- **One real exposure path exists:** `GET v1/internal/return-details/{returnOrderId}` returns the **full** `ReturnDetailsDocument` (not a filtered DTO), so `labelURL` is serialized out verbatim to any caller of that specific endpoint. It does **not** leak through the tenant/subscription list endpoints (those use `ReturnResponse`, which has no `rma`/`labelURL` field) or the outbound EMS event (`EventAttributes` has no label field at all).
- No Base64 encode/decode code exists anywhere in `src/main` today.

### 7.2 DHL behavior (per supplied documentation, not independently verified against DHL's live API)
- Response contains `shipmentTrackingNumber` (top-level, flat) + a `documents[]` array; each entry can carry label/waybill content directly in `documents[].content`, "can be Base64 encoded."
- Format is requestable via `outputImageProperties.encodingFormat` (e.g., `"pdf"`).
- This is a **fundamentally different delivery model**: FedEx = pull (fetch bytes later via URL), DHL = push (bytes embedded directly in the synchronous response).

### 7.3 The open question that gates everything else: does anyone actually consume `labelURL` today?
- **Confirmed:** the exposure path exists (§7.1). **Not confirmed:** whether anything actually calls that endpoint specifically to read the label, or queries MongoDB directly, outside this repository.
- No S4-side API contract exists in this repo to say what S4 itself supports for label delivery (URL vs. inline content).
- **This must be asked directly of the S4/EMS/eClaims teams** (see §11) before committing to a storage approach.

### 7.4 Storage options (decision pending on §7.3's answer)
| Option | Description | Fit |
|---|---|---|
| A — Store Base64 as-is | Simplest; MongoDB has no schema blocker | Risk: document bloat / read-write performance near the 16MB BSON document limit, since `rma` lives inside the same frequently-accessed `returnDetails` document |
| B — Decode & host as URL | Preserves a FedEx-like "URL" contract for any hidden consumer | Only needed if §7.3 finds a real consumer expecting a URL |
| C — Don't persist in SRS at all | Treat label handling as a separate, to-be-designed capability | **Recommended default** if §7.3 finds no consumer — lowest risk, decouples label work from the (low-risk, high-reuse) tracking-number/URL migration |

**Recommendation:** migrate tracking-number/tracking-URL first (low risk, reusable pattern per §4.1/§4.3) and treat label persistence as an explicitly separate, business-confirmed follow-up — do not force DHL's Base64 into the existing FedEx-shaped `RmaShippmentLabel` class.

---

## 8. Customer Email Notification — FedEx Direct-Email vs. DHL (Unconfirmed)

**Question raised on architect review call:** For FedEx, SRS implicitly tells FedEx to email the customer the return label/tracking info. Does DHL provide an equivalent, or is this a gap?

### 8.1 Confirmed FedEx behavior
- `FedexClient.createRmaS4()` sets `payload.setEmailNotification(launchDarklyClient.getEmailNotificationFlag())`. Combined with `customer.emailAddress` (already in the payload), this instructs **FedEx itself** to directly email the customer — a **carrier-side** email, separate from any HP-side communication.

### 8.2 Two separate channels exist today — only one is in question
- **Channel A** — FedEx directly emails the customer (carrier-triggered, entirely outside SRS's control).
- **Channel B** — SRS independently publishes `trackingUrl`/label/QR details via `publishStatusUpdate` → Event Mgr → Solace → EO/ACS/CDAX/downstream systems, which drives the **in-app/HP Smart App notification**. This channel is carrier-agnostic and **continues unchanged regardless of DHL's capability** — no work needed here for DHL parity.
- **The gap under discussion is Channel A only.**

### 8.3 Confirmed: DHL DOES have API-level notification capability — NOT a confirmed gap
- **Update (per DHL team confirmation):** DHL's MyDHL API provides **shipment email notification functionality** — the shipment request can carry a notification method + customer email address, giving DHL the ability to send a carrier-side shipment notification analogous to FedEx's `emailNotification` flag.
- **This should NOT be treated as a confirmed functional gap.** What remains open is narrower and purely mechanical:
  1. The exact REST field(s)/structure for the notification construct in the specific MyDHL contract HP is onboarding (name/shape not yet confirmed against our account's API version).
  2. Whether this notification capability is actually **enabled/configured** on HP's provisioned DHL account (API-level existence ≠ account-level activation).

### 8.4 Regardless of DHL Channel A outcome — Channel B is unaffected
- SRS's own downstream event flow (Channel B — `publishStatusUpdate` → Event Mgr → Solace → EO/ACS/CDAX → HP Smart App notification) **must continue unchanged**, independent of whether DHL's carrier-side notification ends up enabled. This guarantees the customer is notified through the existing HP/internal channel regardless of DHL account configuration status.
- **No gap to report to stakeholders at this time.** Once §11 DHL-Team item #1 (exact field mapping + account enablement) is confirmed, this closes out entirely — either DHL fires its own email too (redundancy case, §8.5) or it doesn't and Channel B alone covers it (current safety net either way).

### 8.5 Future consideration (explicitly not in scope now)
If DHL later adds Channel A support, there's a theoretical dual-notification/redundancy risk (customer gets both a DHL email and the SRS/HP Smart App notification for the same event). Flagged only as forward-looking — no action needed unless/until DHL's capability is confirmed.

**Status:** ✅ Not a confirmed gap — DHL API-level capability confirmed to exist. ⏳ Needs confirmation of exact REST field mapping + account-level enablement from DHL technical contact (see §11, DHL Team).

---

## 9. Other Cross-Cutting Gaps (Condensed)

| Area | FedEx today | DHL / Germany | Status |
|---|---|---|---|
| **Rating/Capability** | No rating/rate-shopping exists in the current flow at all — FedEx's account config implicitly determines service level | DHL exposes a Rating/Capability API, but nothing indicates Germany needs dynamic rate selection | Default assumption: **fixed** product code, matching today's "whatever FedEx gives us" pattern — NEEDS BUSINESS CONFIRMATION if dynamic selection is actually required |
| **Label Free / QR** | No equivalent concept exists in FedEx flow | Sample payload shows `imageOptions[typeCode="qr-code"]` + `valueAddedServices[PZ, PT]`, Base64 PNG QR, **requires separate DHL account enablement** | NEEDS CONFIRMATION whether this is a capability demo or an actual requirement for Germany |
| **Pickup / drop-off** | Not a distinct capability in FedEx flow | `pickup.isRequested` boolean; 4 possible combinations exist (courier+label, courier+QR, drop-off+label, drop-off+QR) | NEEDS BUSINESS DECISION on which combination(s) Germany must support at launch |
| **Customs** | Not present (US-domestic flow, no customs handling anywhere) | Sample shows `isCustomsDeclarable: false`, `incoterm: "DAP"` — consistent with domestic-only OR intra-EU, doesn't resolve which | NEEDS BUSINESS CONFIRMATION of origin scope (Germany-domestic only vs. EU/non-EU cross-border) before customs fields can be finalized |
| **Idempotency/retry** | Retries only on 5xx (100ms fixed backoff); 4xx fails fast, no retry; app-level `waybillGenerated` flag prevents reprocessing; no correlation-ID/idempotency-key mechanism exists | **Critical scenario:** SRS times out after DHL successfully creates a shipment → retry → risk of a **duplicate DHL shipment**. DHL idempotency support is not established from material reviewed | NEEDS DHL CONFIRMATION on idempotency support; if absent, needs a reconciliation-before-retry design |
| **Configuration/secrets** | `FedexConfig` (`fedex.*`), k8s secret refs per environment | New `DhlConfig` (`dhl.*`) needed, parallel secret refs; credential provisioning requires DHL account/contact info upfront | Straightforward addition, no blocker beyond DHL provisioning lead time |
| **Observability** | Logs outbound URL + full payload (gated by LD flags), redacts PII via `logObfuscateCustomer()` | Same dimensions apply: correlation ID, RMA, shipment ID, status, retry context, latency; same PII/credential exclusion rules | Straightforward, mirror existing pattern |
| **Backward compatibility** | `shipmentProvider`/`shippingVendor` already a plain non-enum `String`; `Item.trackingNumber` already carrier-neutral | Introducing `"DHL"` requires no schema/type change | Confirmed safe — **only regression risk is altering the existing `"Fedex"` flag value itself**, which must never happen; DHL must be added additively |

### ⚠️ Do NOT Assume (treat as variable, confirm before building)
| Assumption | Why it's risky |
|---|---|
| DHL tracking number has a fixed length | DHL spec doesn't guarantee length — treat as an opaque identifier, not a fixed-format string |
| Product code is the same across all regions | Must confirm per region/route (Germany domestic vs. EU vs. non-EU) |
| Tracking URL format is the same as FedEx's | DHL's format is different and not yet confirmed for production |
| Weight/dimensions use the same units as FedEx | Confirm metric vs. imperial with DHL — FedEx has no equivalent to compare against |
| Pickup API vs. Shipment API usage | Business must determine which DHL API/flag combination actually applies to returns |

---

## 9a. End-to-End Germany Flow (Proposed)

```text
Customer requests return
        │
ReturnDetailsController.sendReturn() → ReturnDetailsService.sendReturn()
        │
RMA (business identifier — received/stored as retReturnNumber; §4.3)
        │
resolveInitReturnFlow() — carrier selection via LaunchDarkly flag (§6)
        │
        ├── "Fedex" → InitReturnFedexFlow → FedexClient   (UNCHANGED)
        └── "DHL"   → InitReturnDhlFlow (NEW) → DhlClient (NEW)
                          │
                    DHL HTTP Basic Auth (§4.2)
                          │
                    Map SRS return → DHL shipment request
                       (RMA → customerReferences[typeCode="RMA"];
                        shipper/receiver, product code, weight/dimensions,
                        pickup.isRequested — all NEEDS CONFIRMATION, §11/§12)
                          │
                    Create DHL shipment (generic Shipment API — no dedicated Return endpoint)
                          │
                    shipmentTrackingNumber (waybill) returned
                          │
                    Label (PDF/ZPL Base64) OR Label-Free QR (Base64 PNG) — business decision, §9
                          │
                    Persist ONLY what is confirmed necessary:
                       - trackingNumber → Item.trackingNumber (carrier-neutral, no schema change)
                       - trackingUrl → carrier-neutral field (no schema change)
                       - label/QR content → ONLY if §7.4 confirms a persistence need
        │
Publish to EMS (trackingNumber, shippingVendor="DHL", trackingUrl) — §7.1/§9
        │
Downstream consumers: S4/SAP, eClaims, other listeners (UNCHANGED topology)
        │
Inbound: S4 status callback → existing status-handling path (UNCHANGED mechanism)
```
**Key point:** SRS both *publishes* outbound status to the event-manager layer (consumed by S4 downstream) *and* separately *receives* an inbound status callback from S4 — both directions must be designed for DHL, not just the outbound one.

---

## 10. Testing Impact

| Test area | Type | Notes |
|---|---|---|
| DHL flow selection (`getFlowType()=="DHL"` correctly picked) | Unit | Mirror `InitReturnFedexFlowTest` |
| FedEx regression (still selects FedEx, unaffected by new DHL bean) | Unit | Add explicit multi-bean test if none exists |
| DHL request mapping (RMA, shipper/receiver, packages) | Unit | Once mandatory fields finalized |
| Tracking number extraction (`shipmentTrackingNumber` → `Item.trackingNumber`) | Unit | New extraction path, distinct nesting from FedEx |
| Tracking URL construction | Unit | Once `dhl.trackingUrl` confirmed |
| Label parsing (PDF/ZPL Base64) | Unit | Only if §7.4 decision requires SRS to process label bytes |
| Label Free QR handling | Unit | Only if §9 Label Free is confirmed in scope |
| DHL status → SRS status mapping | Unit | Only if a DHL-event-to-status table is confirmed needed |
| Pickup true/false | Unit | Covers both values per §9 business decision |
| Error handling (400/401/403/429/5xx/timeout/invalid data) | Unit + Integration | Mirror `FedexClient`'s existing coverage |
| Duplicate/retry scenario | Integration | Requires DHL sandbox or timeout-simulating mock |
| EMS event generation with DHL values | Unit | Extend existing messaging-service coverage |
| Contract testing | Contract | Against actual MyDHL v3.3.1 OpenAPI schema before production — not yet available in this repo |

---

## 11. ALL OPEN QUESTIONS — By Owner (Single Consolidated Section)

> Every unresolved question from both source documents is captured here exactly once, grouped by who needs to answer it. Cross-references point back to the relevant section above for context.

### 🔵 DHL Team
1. Is `shipmentNotification` (or equivalent) available/enabled on HP's MyDHL API account to trigger a direct customer email, matching FedEx's `emailNotification` behavior? **DHL API-level capability already confirmed to exist** — remaining open items are (a) exact REST field mapping for HP's contract version, and (b) confirmation the feature is enabled on HP's provisioned account. (§8)
2. Does the provisioned MyDHL API instance support idempotent shipment creation, or must SRS build its own reconciliation-before-retry logic to avoid duplicate shipments on timeout? (§9)
3. Is `productCode = "U"` (per sample) or `"N"` correct for a Germany-domestic customer return — and does it differ by origin (EU vs. non-EU) or account/service agreement?
4. Must the customer always be the shipper, and HP/warehouse always the receiver, in every Germany return scenario?
5. Is Label Free/PZ/QR actually **required** for the Germany customer journey, or does the sample merely demonstrate the capability exists? Does Label Free need separate account enablement first? (§9)
6. What exact HTTP status/response body does DHL return for invalid address, invalid product code, and invalid package data?
7. Is `https://www.dhl.com/express/track?AWB=<number>` the correct, stable, **production** customer-facing tracking URL for this account/region? (§4.1)
8. Can `documents[]` contain more than one document per response (e.g., commercial invoice + label)? If so, what field reliably identifies "the label"?
9. Is a customer phone number mandatory for the DHL shipment request? (§5)
10. Does the DHL shipment-creation call always return the label synchronously, or can it be async/delayed?

### 🟢 S4 / EMS Team
11. Does S4 validate/enum-constrain `carrierName`/`shippingVendor`? Will `"DHL"` be accepted without an S4-side change?
12. Does S4 validate tracking-number format/length? FedEx is 12-digit numeric; DHL is 10-digit alphanumeric (or longer at package level) — will relaxed validation be needed?
13. Does S4 construct the tracking URL itself, or always accept it from SRS?
14. **Does S4/EMS (or any listener) actually need the label document/PDF at all**, or is tracking number + tracking URL sufficient (which is all today's contract carries)? (§7.3 — gates the entire label storage decision)
15. Does anything actually call `GET v1/internal/return-details/{returnOrderId}` or query MongoDB directly to read `labelURL` today? (§7.3)
16. Are there any downstream systems hardcoded to assume `"Fedex"` as the only carrier value?

### 🟡 Business / Product
17. Which of the 4 Germany logistics combinations (courier+label, courier+QR, drop-off+label, drop-off+QR) must SRS support at launch? (§9)
18. Is Germany DHL scoped to **domestic-only** returns, or must it also support EU-country-to-Germany / non-EU-country-to-Germany? (§9)
19. Should the DHL account holder/payer always be HP, or can the customer ever be the payer?
20. Is courier pickup required for Germany returns at all, or is drop-off-only acceptable?
21. If DHL's label must be persisted, should it be (a) Base64 as-is, (b) decoded to a hosted URL, or (c) not persisted by SRS at all? (§7.4)
22. **[Program-level]** Do we wait for HP Store's API to finish and reuse it for label generation, build a direct DHL integration, or use a hybrid/Stellar common-API layer? *(Build vs. Buy vs. Wait decision — RAID item)*
23. **[Program-level]** Are there regional behavior differences to account for across Netherlands, France, Spain, Italy, UK, and Canada (not just Germany)? *(RAID item — broader STFSVC-2656 scope)*

### 🟣 Architecture
24. Should `shippingVendor`/`carrierName` be promoted from a plain `String` to a shared enum now that a second carrier is being introduced?
25. Should DHL label persistence (if required) be inline Base64, object storage + URL, or a dedicated DHL document model? (§7.4)
26. Should a DHL-specific idempotency/reconciliation step be built proactively, or should SRS accept the same retry-on-5xx-only risk profile currently accepted for FedEx? (§9)
27. Which approach for Germany-only carrier scoping: per-request LaunchDarkly context + targeting rule, or a simpler code-level country check? (§6.2)
28. If the LaunchDarkly-context approach is chosen, would changing the shared `ldUser` to a per-request context affect any of the ~30 *other* flags currently reusing that same static object? (§6.2)
29. **[Program-level]** Can we use the **Stellar API** as the integration layer, or should ICS integrate directly with DHL? *(RAID item — integration design)*

### 🟠 SRS Team
30. Where, if anywhere, can weight/dimensions/package-count/description data for a return shipment be sourced today (order data, fulfillment data, product catalog, or another integration)? (§5)
31. Does the existing `resolveInitReturnFlow()` test coverage assume exactly one `InitReturnFlow` bean, requiring updates once `InitReturnDhlFlow` is added?
32. Which field should represent "country" for carrier-scoping purposes (ship-from/customer address vs. warehouse/receiver address), and how does it get captured onto the return document before `resolveInitReturnFlow()` runs? (§6.2)
32a. ~~Does TAO/downstream need to send a different carrier/service value for Germany?~~ **RESOLVED (§6.3):** No — `InitiateReturnRequest` has no carrier field today; the caller never selects the carrier, so no inbound contract change is needed for TAO.

### ⚪ Program / Ownership (RAID-tracked, non-technical)
33. **TPRA approval** — has HP already approved third-party integration with DHL, or does a TPRA request need to be created? *(Blocker severity: HIGH — cannot integrate with an external DHL API without this)*
34. **Ownership** — will the new DHL API be invoked by **TIO** (Transaction Integration Orchestrator) or directly by **ICS Returns Service**? *(Confirmed so far: Shipping labels will be created by ICS Return Service, which owns the DHL API integration and label-generation flow.)*

---

## 12. Master Gap & Readiness Table

| Area | FedEx (current) | DHL (Germany) | Gap | Required change | Status | Owner |
|---|---|---|---|---|---|---|
| Carrier selection (bean/flow) | Strategy pattern, beans matched by `getFlowType()` | Add `InitReturnDhlFlow`, `getFlowType()→"DHL"` | New bean + new flag value | Add `InitReturnDhlFlow` + `DhlClient` | PROPOSED | SRS team |
| Carrier selection **request contract** (TAO/downstream) | `InitiateReturnRequest` has no carrier field; caller never selects carrier | Same — no change to inbound contract needed | None | None — confirmed no caller-side change required (§6.3) | CONFIRMED (no gap) | N/A |
| Carrier selection **scope** (Germany-only) | One global static LD flag value, no per-request context | Need DHL-for-Germany-only, FedEx elsewhere | LD context has no attributes to target on; SRS model has no country field | Per-request LD context + targeting rule, **or** code-level country check; capture country onto the return document either way | NEEDS ENHANCEMENT + BUSINESS CONFIRMATION | SRS team + Architecture |
| RMA | Root-level `rmaNumber` | `customerReferences[typeCode="RMA"]` | Placement only, value 95% reusable | New mapping in `DhlClient` | CONFIRMED (mapping approach) | SRS team |
| Authentication | OAuth2 + Bearer token, `FedexTokenHolder` | HTTP Basic Auth, stateless | Different mechanism, simpler for DHL | New `DhlConfig`/`DhlClient` auth | CONFIRMED (both sides) | DHL team |
| Tracking number | Nested, 12-digit numeric | Top-level, 10-digit alphanumeric | Different nesting + format | New extraction path into existing `Item.trackingNumber` | CONFIRMED (fields compatible) | SRS team |
| Tracking URL | SRS-constructed: config prefix + number | Same pattern proposed; exact format TBD | New config value | Add `dhl.trackingUrl` | NEEDS CONFIRMATION (correct production URL) | DHL team |
| Address model | Lookup by ID only | Full address object required | Major — new data required | Populate `receiverDetails` fully | CONFIRMED gap | SRS team |
| Items/package model | Item-centric (SKU+qty) | Package-centric (weight+dimensions) | Major structural difference | New extraction/mapping logic | CONFIRMED gap | SRS team |
| Weight/dimensions | Not in `Item` model | Required | Critical missing data | Add field, default, or catalog lookup | NEEDS CONFIRMATION (data source) | SRS team + Business |
| Customer phone | Not captured | Required by DHL | Missing field | Add to user/return model or make optional | NEEDS CONFIRMATION | DHL team |
| Label | `labelURL` (pull, URL), stored but unused, exposed via one internal GET endpoint | `documents[].content` (push, Base64) | Entirely different delivery model | New DHL-specific handling, only if confirmed needed | NEEDS CONFIRMATION (is persistence needed at all — §7.3) | Business/Product |
| Label Free / QR | No equivalent | `imageOptions[qr-code]`, `valueAddedServices[PZ,PT]`, account-enablement required | Entire new capability | Build only if confirmed required | NEEDS CONFIRMATION | DHL team + Business |
| Email notification | `emailNotification` flag → carrier-side email | DHL MyDHL API confirmed to support shipment notification (method + email) — capability exists | Not a functional gap; only field-mapping/account-enablement confirmation needed | Map flag → DHL notification object once field mapping confirmed | NEEDS FIELD-MAPPING CONFIRMATION (not a gap) | DHL team |
| Product code | N/A (implicit) | `U` in sample; domestic/EU/non-EU code not established | Must confirm per route | Confirm before implementation | NEEDS CONFIRMATION | DHL team + Business |
| Rating | Not present | DHL exposes Rating API, may not be needed | Only relevant if dynamic selection required | Do not build unless confirmed | NEEDS CONFIRMATION | Business/Product |
| Pickup/drop-off | Not a distinct capability | `pickup.isRequested` + Service Point; 4-option decision | Business decision on logistics model | Build only confirmed combination(s) | NEEDS CONFIRMATION | Business/Product |
| Customs | Not present (US-domestic) | Only relevant if scope crosses a customs border | Origin scope unconfirmed | Build only if scope requires it | NEEDS CONFIRMATION | Business/Product |
| S4/EMS event contract | `trackingNumber`, `shippingVendor`, `trackingUrl` only, all plain strings | Same fields, new string value `"DHL"` | No structural gap for tracking; label field would be net-new | None required for tracking/URL; new field only if label forwarding confirmed | CONFIRMED (structural compatibility) / NEEDS CONFIRMATION (S4 accepts "DHL") | S4/EMS team |
| Idempotency/retry | Retries 5xx only; app-level `waybillGenerated` flag | Duplicate-shipment risk on timeout+retry; DHL idempotency unconfirmed | Need duplicate-prevention strategy | Confirm DHL support; design reconciliation if absent | NEEDS CONFIRMATION | DHL team + Architecture |
| Configuration/secrets | `FedexConfig` (`fedex.*`), k8s secret refs | New `DhlConfig` (`dhl.*`), new secret refs | New config class + secrets per environment | Add `DhlConfig` + secrets | PROPOSED | SRS team + DevOps |

---

## 13. Implementation Impact Summary

**A. Definitely requires modification:** LaunchDarkly flag configuration (new `"DHL"` targeting value) — a config change, not a code change, since the selection loop itself needs nothing.

**B. May require modification (conditional on open decisions above):**
- Tracking-number extraction logic (carrier-aware branch, only if needed)
- `ReturnDetailsDocument`-equivalent model (only if new DHL-specific fields are confirmed necessary)
- `Item` model (only if weight/dimensions must be added here — not reflexively)

**C. New classes/config likely required:**
- `InitReturnDhlFlow` (implements `InitReturnFlow`, `getFlowType() → "DHL"`)
- `DhlClient`, `DhlConfig`, `DhlShipmentRequest`
- Optional: DHL-specific label/document model, DHL status-mapper, DHL-specific exception types (parallel to `FedexUnauthorizedException` etc.)

**D. Must remain untouched:** `FedexClient`, `FedexTokenHolder`, `FedexConfig`, `Rma`/`RmaShipment`/`RmaShippmentLabel`, `InitReturnFedexFlow`, all FedEx-specific exceptions.

**E. S4/EMS contract:** No change required for tracking number/URL/carrier-name (already carrier-neutral plain strings). A new field is only needed if label forwarding is confirmed as a requirement.

**F. Database/model:** No change to store a DHL tracking number or carrier name. Possibly new fields for label content and/or weight/dimensions, only if confirmed necessary.

**G. External dependencies:** DHL Express account + API credentials (provisioning dependency, DHL-side lead time); possibly a DHL sandbox/mock for contract testing (not yet available in this repo); TPRA approval (§11, item 33).

---

## 14. Readiness Summary

**✅ CONFIRMED FROM CODE**
- Carrier-selection mechanism is a proven, multi-bean strategy pattern (§6.1)
- RMA is received (not generated), a plain business identifier, 95% reusable (§4.3)
- FedEx auth mechanism, label-as-URL model, and tracking extraction/propagation paths, all with file/line evidence (Appendix A)
- SRS's outbound path is an EMS publish, not a direct S4 call; S4 reaches SRS only via an inbound callback
- `Item.trackingNumber` and `shippingVendor` are carrier-neutral plain strings — no schema change needed for DHL values
- Weight/dimensions/package data do **not** exist in the `Item` model today

**✅ CONFIRMED FROM DHL MATERIAL**
- DHL Shipment/Pickup/Tracking/Rating APIs exist; no dedicated Return endpoint
- `RMA` vs. `CU` reference-type distinction
- Label as PDF or ZPL, Base64-encoded, via `documents[]`, plus QR-code + `PZ`/`PT` value-added services
- HTTP Basic Auth, no token lifecycle

**⏳ BLOCKERS BEFORE DEVELOPMENT STARTS**
1. DHL account provisioning (credentials, endpoint) — DHL-side lead time
2. Product code + customs scope confirmation (DHL/Business)
3. Germany logistics model confirmation (pickup/drop-off/Label Free) — Business
4. DHL idempotency behavior confirmation — otherwise duplicate-shipment risk is unmitigated
5. TPRA approval for external DHL integration (§11, item 33)
6. Ownership decision: TIO vs. ICS Returns Service (§11, item 34) — **Note:** label creation is already confirmed to sit with ICS Return Service

---

## Appendix A: FedEx Code Evidence (Reference Only)

Condensed file/line reference list — full narrative detail intentionally omitted here to keep this document presentable; all facts above were sourced from this evidence.

| # | Class / File | What it shows |
|---|---|---|
| 1 | `repository/model/RmaShippmentLabel.java:18-22` | Fields `labelURL`, `trackingNumber` — plain strings, no length/format constraints |
| 2 | `repository/model/RmaShipment.java:35-36` | Field `label` (`RmaShippmentLabel`) |
| 3 | `repository/model/Rma.java:31-32` | Field `shipments` (`ArrayList<RmaShipment>`) |
| 4 | `repository/model/ReturnDetailsDocument.java:67-68,116-117,119-120` | Fields `rma`, `trackingUrl`, `retReturnNumber` |
| 5 | `clients/fedex/FedexClient.java:71-236` | `createRma()`/`createRmaS4()`/`fedexCall()` — response deserialized directly into `ReturnDetailsDocument`; RMA validated ≤20 chars (line 152-155) |
| 6 | `service/fedex/InitReturnFedexFlow.java:33-66` | `initReturn()`; line 51 builds `trackingUrl` from tracking number (not `labelURL`); `getFlowType()→"Fedex"` (line 65-66) |
| 7 | `service/ReturnDetailsService.java:195,211,242-249,1426-1439` | Flag read, `save()`, `resolveInitReturnFlow()`, `setFedexTrackingNumber()` (copies tracking number only, never `labelURL`) |
| 8 | `repository/model/Item.java:74-75` | `trackingNumber` field — no `labelURL` field exists on `Item` |
| 9 | `service/MessagingService.java:56-102,133-192` | `getEventAttributes()` — only `trackingNumber`/`shippingVendor`/`trackingUrl`; `publish()` to EMS |
| 10 | `clients/stratus/eventsmanager/dto/EventAttributes.java` | Full 27-line DTO — no label field |
| 11 | `service/model/ReturnStatusS4.java:109-118` | Inbound S4 callback `ShippingInfo` — no label field |
| 12 | `config/properties/FedexConfig.java:31,33` | `trackingUrl`, `createRmaUrl` config properties |
| 13 | `application.properties:174` | `fedex.trackingUrl=https://www.fedex.com/fedextrack/?tracknumbers=` |
| 14 | `repository/ReturnDetailsRepository.java:92-108` | Mongo existence-check query `'rma.shipment.label': null` — note: singular "shipment" vs. actual plural `shipments[]` field path (pre-existing minor inconsistency, unrelated to DHL) |
| 15 | `api/ReturnDetailsController.java:95-111` | `GET v1/internal/return-details/{returnOrderId}` — returns full document, the one real `labelURL` exposure path |
| 16 | `clients/ldc/LaunchDarklyClient.java:57-59` | `getShipmentProviderFlag()` — evaluates against the shared static `ldUser` |
| 17 | `config/LaunchDarklyConfig.java:18-29` | `LDUser` built once at startup from a single config value — no per-request attributes |
| 18 | `etc/postman/Stratus Returns Service.postman_collection.json` | Real-shape FedEx response example (`label.id`, `label.labelURL`, `label.trackingNumber`) |

**FedEx credentials (full list):** `contentType`, `xOrgName`, `orgName`, `accessTokenUri`, `grantType`, `scope`, `password`, `clientId`, `clientSecret`, `trackingUrl`, `createRmaUrl` (`FedexConfig`, prefix `fedex.*`). Username is sourced from a LaunchDarkly flag, not `FedexConfig`.

**FedEx error handling:** retries only on 5xx (`@Retryable`, 100ms fixed backoff); 4xx caught and re-thrown, no retry; blank auth token → `FedexUnauthorizedException` (no retry); `RmaNumberLengthException` for RMA >20 chars; `FedexPreconditionFailedException` for missing return-config data. No correlation-ID or idempotency-key mechanism exists for FedEx either.

---

## Appendix B: DHL Tracking Event Codes (Reference)

| Code | Meaning | Code | Meaning | Code | Meaning |
|---|---|---|---|---|---|
| AD | Agreed delivery | FD | Forward destination | PY | Payment |
| AF | Arrived facility | HP | Held for payment | RD | Refused delivery |
| AR | Arrival in delivery facility | IC | In clearance processing | RR | Response received |
| BA | Bad address | MC | Miscode | RT | Returned to consignor |
| BN | Customer broker notified | MD | Missed delivery cycle | SA | Shipment acceptance |
| BR | Broker release | MS | Mis-sort | SC | State changed |
| CA | Closed on arrival | ND | Not delivered | SD | Shipment info received |
| CC | Awaiting cnee collection | NH | Not home | SM | Scheduled movement |
| CD | Controllable clearance delay | OH | On hold | SS | Shipment stopped |
| CM | Customer moved | **OK** | **Delivery ✅ (success code)** | TP | Forwarded to 3rd party |
| CR | Clearance release | PD | Partial delivery | TR | Record of transfer |
| CS | Closed shipment | PL | Processed at location | UD | Uncontrollable clearance delay |
| DD | Delivered damaged | PU | Shipment pick up | WC | With delivering courier |
| DF | Depot facility | | | | |
| DS | Destroyed/disposal | | | | |
| EM | Electronic data merge | | | | |

## Appendix C: Program Tracking Reference (RAID STFSVC-2656)

**Related documentation:** `SRS-DHL-Germany-Expansion-Analysis.md` (technical "how" analysis) · `DHL-Integration-Requirements.md` (API requirements/checklist) · `FedEx-vs-DHL-Comparison.md` (architecture diffs) — all in `docs/geo/`.

**Non-technical blocking items tracked on the RAID ticket (owners/dates as last recorded):**

| # | Item | Owner | Target | Status |
|---|---|---|---|---|
| 1 | Review DHL API specs, identify gaps/impacts | Hemantha Kumar Kandra | Aug 11, 2026 | ⏳ In progress at time of writing |
| 2 | Assess event changes needed for DHL return process | Hemantha Kumar Kandra | Aug 11, 2026 | ⏳ In progress |
| 3 | Resolve TPRA approval status (§11, item 33) | TBD | ASAP | ❌ Pending — **blocking** |
| 4 | Clarify ownership: TIO vs. ICS (§11, item 34) | Hemanth | ASAP | ❌ Pending — **blocking** |
| 5 | HP Store API build/wait/buy decision (§11, item 22) | TBD | TBD | ❌ Pending |
| 6 | Regional behavior analysis beyond Germany (§11, item 23) | TBD | TBD | ❌ Pending |
| 7 | Stellar API evaluation (§11, item 29) | TBD | TBD | ❌ Pending |

**Risk register (severity / likelihood / mitigation):**

| Risk | Severity | Likelihood | Mitigation |
|---|---|---|---|
| TPRA rejection/delay | 🔴 High | Medium | Start TPRA process immediately, in parallel with technical design |
| HP Store API delay | 🟡 Medium | High | Plan a direct DHL integration as a fallback, don't block on HP Store |
| Regional differences (NL/FR/ES/IT/UK/CA) | 🟡 Medium | Medium | Complete regional analysis early; don't assume Germany's answers apply elsewhere |
| Ownership conflict (TIO vs. ICS) | 🟡 Medium | Low | Schedule an explicit decision meeting; ICS is already confirmed to own label creation |
| Multi-country scope creep | 🟡 Medium | High | Phase rollout by country — Germany first, others as separately scoped follow-ups |

**Stakeholders (RAID ticket, for reference):** Mareena Narikuth (RAID owner/coordination), Bogdan Daniel Ciurezu (BRD/DHL coordination), Hemantha Kumar Kandra (technical review), Queralt Galin (DHL API spec distribution), Cristian Colonello, Chandrashekhar Khandke, Roger Bergada, Subhamita Abhyankar, Sandeep Kumar Verma, BABU SIVALINKI, Ana Rosa Villegas.

**Success criteria to unblock STFSVC-2656:** DHL API specs reviewed ✅ (done Aug 6, 2026) · TPRA approval obtained ❌ · Ownership (TIO vs. ICS) decided ❌ · Integration approach (Stellar vs. direct vs. HP Store) decided ❌ · Regional behavior differences documented ⏳ · Multi-carrier strategy confirmed ⏳.





