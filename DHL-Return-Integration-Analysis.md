# DHL Return Integration Analysis — SRS Germany

**Application:** SRS (Starters Return Service)  
**Current region/carrier:** US / FedEx  
**Target region/carrier:** Germany / DHL Express  
**Reference document:** `FedEx-vs-DHL-Gap-Analysis-Complete.md`

---

## 1. Purpose

This document is a validated companion to the existing FedEx-vs-DHL gap analysis. The objective is to understand how the existing SRS FedEx return flow can be extended to support DHL Express for Germany.

The scope is specifically the **customer return flow**, not a new-order outbound shipment flow.

The analysis covers:

- Return shipment creation
- RMA handling
- Authentication
- Pickup
- Tracking
- Tracking URL
- Rating
- S4 callbacks/events
- Carrier/shipping-vendor mapping
- Data and code gaps
- Recommended integration approach

Where the existing repository/code does not provide enough information, the item is marked **Needs confirmation** rather than being assumed.

---

## 2. Existing SRS Business Flow

The existing SRS application handles return/replacement-related processing for the US region.

The conceptual flow is:

```text
Customer requests return
        ↓
SRS
        ↓
RMA generated / retrieved
        ↓
FedEx return shipment initiated
        ↓
FedEx tracking information
        ↓
SRS tracking/status handling
        ↓
S4 callbacks/events
        ↓
Return/shipping status consumed downstream
```

S4 receives shipment/return-related events such as initiated, picked up, received, delivered and other status events.

For Germany, DHL Express becomes the carrier:

```text
Customer
   ↓
SRS
   ↓
RMA
   ↓
DHL return shipment
   ↓
DHL tracking
   ↓
S4
```

The common SRS return/RMA business logic should ideally remain reusable while the carrier-specific FedEx integration is replaced or extended with DHL.

---

# 3. DHL API Capabilities Relevant to SRS

DHL MyDHL API provides multiple capabilities, including:

| Capability | Relevance |
|---|---|
| Shipment | Core capability to create the physical DHL shipment |
| Pickup | Used when DHL courier collection is required |
| Tracking | Used to retrieve shipment/tracking status |
| Rating / Capability | Used when rate/product/service selection is required |
| Address | Address validation/capability |
| Service Point | Physical DHL location/drop-off/pick-up functionality |
| Identifier | Identifier-related DHL operations |

Official DHL documentation:

https://developer.dhl.com/api-reference/dhl-express-mydhl-api

---

# 4. Shipment vs Return

## Is there a separate DHL Return Shipment API?

The MyDHL API does not expose a separate API named `CreateReturnShipment` in the API model being analysed.

The DHL **Shipment** capability creates the physical DHL shipment.

Therefore, for SRS:

```text
Business process:
RETURN

Carrier operation:
CREATE SHIPMENT
```

The same generic Shipment capability can be used for a return shipment.

## Outbound shipment

```text
HP / Warehouse
      ↓
     DHL
      ↓
   Customer
```

Typical logical roles:

```text
Shipper = HP
Receiver = Customer
```

## Return shipment

```text
Customer
      ↓
     DHL
      ↓
HP Return Center
```

Typical logical roles:

```text
Shipper/origin = Customer
Receiver/destination = HP return location
```

The exact DHL account/payer/origin/destination arrangement must be confirmed with the DHL Germany business setup.

### Important

Do not interpret the generic DHL Shipment API as meaning SRS must implement the new-order shipment business flow.

SRS is still implementing a **return**. DHL Shipment is simply the carrier operation used to create the physical return shipment.

---

# 5. Pickup vs Return

This is a critical distinction.

### Return

A return is the SRS business process:

> Customer wants to send a product back to HP.

### Shipment

The shipment is the physical carrier movement:

> Create a DHL shipment from the return origin to HP.

### Pickup

Pickup is a logistics operation:

> Ask DHL to send a courier to a location and collect the package.

Therefore:

**Pickup is NOT the same as Return.**

Pickup can be part of a return.

Example:

```text
Customer requests return
        ↓
SRS creates RMA
        ↓
SRS creates DHL shipment
        ↓
DHL Pickup requested
        ↓
DHL courier collects package
        ↓
DHL transports package
        ↓
HP
```

A return may also use a drop-off/service-point process instead of courier pickup.

## DHL Pickup

DHL provides a separate Pickup capability for creating, updating and cancelling pickup requests.

The Germany implementation must confirm whether the business requires:

1. DHL courier pickup from the customer's address, or
2. Customer drop-off at an accepted DHL location/service point.

The sample DHL shipment payload contains:

```json
"pickup": {
  "isRequested": false
}
```

This only means that the supplied sample does not request pickup. It does not mean DHL pickup is unavailable.

---

# 6. RMA Analysis

## What is RMA?

RMA means **Return Merchandise Authorization**.

In SRS, the RMA is the business identifier for the return.

It should not be confused with:

- DHL authentication credentials
- DHL access token
- DHL tracking number
- DHL shipment/waybill number

Conceptually:

```text
SRS RMA
   ↓
Business return
   ↓
DHL shipment
   ↓
DHL tracking number
```

## Does DHL support RMA?

Yes.

DHL's reference-data documentation explicitly defines:

```text
RMA = RMA Number
```

DHL also defines other reference types, including:

```text
CU  = Consignor reference number
OID = Order Number
PON = Purchase Order Number
RTL = Return Leg waybill number
SWN = Original Waybill number (Return)
```

Official DHL reference data:

https://developer.dhl.com/sites/default/files/2025-10/MyDHL%20API%20Reference%20data%20guide%20%283.1.0%29.pdf

### Important correction

Do not assume:

```text
RMA = CU
```

`CU` is a **Consignor reference number**.

DHL explicitly has:

```text
RMA = RMA Number
```

Therefore the preferred mapping is:

```text
SRS RMA
    ↓
DHL shipment/customer reference
    ↓
typeCode = RMA
```

The exact REST JSON path for the reference must be confirmed against the final MyDHL v3.3.1 OpenAPI schema being used by the implementation.

## RMA relationship with tracking

The preferred relationship is:

```text
RMA123456
    ↓
DHL Shipment
    ↓
DHL Tracking Number 1234567890
    ↓
Tracking events
```

The RMA remains the business identifier.

The DHL tracking number identifies the physical shipment.

---

# 7. Authentication

## Existing FedEx

The existing repository analysis should be used to determine the exact FedEx authentication mechanism, including:

- Token endpoint
- Client credentials
- Access token
- Token refresh
- HTTP headers
- Token storage
- Interceptors/filters
- Secret configuration

Do not assume the FedEx mechanism without checking the code.

## DHL MyDHL

DHL MyDHL API uses **pre-emptive HTTP Basic Authentication** with DHL-provided credentials.

Conceptually:

```text
SRS
 ↓
DHL credentials
 ↓
HTTP Basic Authentication
 ↓
MyDHL API
```

Therefore the DHL implementation should not create an OAuth token-refresh component simply because the existing FedEx implementation has one.

The concepts are separate:

```text
DHL credentials
     ↓
Authentication

RMA
     ↓
Business return identifier

DHL tracking number
     ↓
Physical shipment identifier
```

---

# 8. Tracking

DHL provides a dedicated Tracking capability.

The tracking operation is used after a DHL shipment/tracking identifier exists.

Conceptually:

```text
DHL Create Shipment
        ↓
DHL shipment/tracking number
        ↓
DHL Tracking API
        ↓
Tracking events/status
```

There is not a separate business concept that requires a different tracking API simply because the shipment is a return.

Both outbound and return shipments are DHL shipments.

## Tracking request concept

Example:

```http
GET /shipments/{shipmentTrackingNumber}/tracking
```

The exact endpoint/path should be taken from the final MyDHL v3.3.1 OpenAPI specification supplied for the implementation.

## Tracking response concept

The response contains shipment/tracking information and tracking events/statuses.

Conceptually:

```json
{
  "shipmentTrackingNumber": "DHL_TRACKING_NUMBER",
  "status": "IN_TRANSIT",
  "events": [
    {
      "date": "2026-08-20",
      "time": "14:30",
      "location": "Berlin",
      "description": "Shipment picked up"
    }
  ]
}
```

This is a conceptual example only. Do not implement this exact response structure without validating the actual DHL schema.

---

# 9. Tracking Number vs Tracking URL

These are separate.

### Tracking number

The DHL shipment/waybill identifier returned/associated with the physical shipment.

### Tracking API

The API SRS uses to retrieve tracking information.

### Customer-facing tracking URL

The browser URL a customer/downstream system can use to view tracking.

Do not assume that the DHL API endpoint and the customer-facing URL are the same.

The implementation must determine who currently creates the FedEx tracking URL:

1. FedEx returns it
2. SRS constructs it
3. S4 constructs it
4. Another downstream system constructs it

Then the same ownership should be determined for DHL.

### Important

Do not hardcode a DHL tracking URL until the customer-facing URL format is confirmed.

Also do not assume a fixed DHL tracking-number length.

Treat the carrier tracking identifier as an opaque carrier identifier unless the S4/downstream contract requires a specific validation.

---

# 10. DHL Product Code

The sample DHL payload supplied to the team uses:

```json
"productCode": "U"
```

The supplied DHL documentation identifies `U` as an EXPRESS WORLDWIDE product code in the relevant product-code reference.

However:

**Do not automatically implement `U` for Germany.**

The correct product must be confirmed based on:

- Germany origin/destination
- DHL account
- Required service level
- Shipment type
- Business agreement
- Package/content characteristics

The sample payload is sample/test data and should not be copied directly into production.

---

# 11. Shipment Request Mapping

The supplied DHL sample contains concepts such as:

```json
{
  "plannedShippingDateAndTime": "...",
  "pickup": {
    "isRequested": false
  },
  "productCode": "U",
  "accounts": [
    {
      "number": "DHL_ACCOUNT",
      "typeCode": "shipper"
    }
  ],
  "customerReferences": [
    {
      "value": "REFERENCE",
      "typeCode": "CU"
    }
  ],
  "outputImageProperties": {
    "encodingFormat": "pdf"
  },
  "customerDetails": {
    "shipperDetails": {},
    "receiverDetails": {}
  },
  "content": {
    "packages": []
  }
}
```

For SRS return processing, the conceptual mapping is:

| SRS concept | DHL concept |
|---|---|
| Return/RMA | DHL `RMA` reference |
| Customer/origin | `shipperDetails` |
| HP return destination | `receiverDetails` |
| Carrier service | `productCode` |
| DHL account | `accounts` |
| Pickup required | `pickup.isRequested` / Pickup capability |
| Package | `content.packages` |
| Label | `outputImageProperties` / shipment document options |
| Tracking | Shipment response + Tracking service |

The exact mandatory fields must be confirmed against the final API schema and Germany shipment scenario.

---

# 12. Important Data Gap: Weight and Dimensions

The existing FedEx return model described in the analysis contains item/SKU/quantity information.

DHL shipment creation requires physical shipment/package information for the applicable shipment scenario.

Therefore investigate where SRS can obtain:

- Weight
- Length
- Width
- Height
- Package count
- Package contents/description
- Customs information where required

Do not immediately add these fields to the SRS item model.

First determine whether they already exist in:

- Order data
- Product catalog
- Warehouse/fulfillment data
- Existing shipment model
- Another downstream service

---

# 13. Rating

DHL provides Rating/Capability functionality.

However, Rating is only required for SRS if the business flow actually needs rate/product selection.

### Rating is relevant if:

- SRS currently performs FedEx rating
- Germany needs dynamic DHL service selection
- The business needs rate calculation before shipment creation

### Rating may be outside scope if:

- Germany has already selected a fixed DHL product/service
- SRS only needs to create the return shipment and track it

Therefore check the existing FedEx code first.

Do not add DHL Rating simply because DHL provided a Rating URL.

---

# 14. S4 Callback/Event Analysis

The existing analysis identified shipment information similar to:

```json
{
  "shipmentDetails": {
    "carrierTrackingNumber": "738979304299",
    "carrierName": "Fedex",
    "carrierTrackingURL": "https://..."
  }
}
```

For Germany, conceptually:

```json
{
  "shipmentDetails": {
    "carrierTrackingNumber": "<DHL_TRACKING_NUMBER>",
    "carrierName": "DHL",
    "carrierTrackingURL": "<DHL_TRACKING_URL>"
  }
}
```

But the following must be confirmed:

| Question | Status |
|---|---|
| Does S4 have carrier field? | Confirm from existing payload |
| Can carrier value be `DHL`? | Needs S4 confirmation |
| Is carrier an enum? | Needs S4 confirmation |
| Does S4 validate tracking number? | Needs confirmation |
| Does S4 construct tracking URL? | Needs confirmation |
| Does S4 simply propagate tracking URL? | Needs confirmation |
| Do downstream consumers hardcode FedEx? | Needs confirmation |
| Is a schema change required? | Needs confirmation |

### Critical S4 question

> If SRS sends `carrierName = DHL`, DHL tracking number and DHL tracking URL through the existing S4 event contract, will S4 and all downstream consumers accept and propagate these values without a contract change?

---

# 15. DHL Event to SRS Status Mapping

DHL tracking events include carrier-specific event codes.

Examples from the supplied DHL material include:

| DHL code | Meaning |
|---|---|
| SD | Shipment information received |
| SA | Shipment acceptance |
| PU | Shipment pickup |
| AF | Arrived facility |
| PL | Processed at location |
| WC | With delivering courier |
| OK | Delivery |
| DD | Delivered damaged |
| ND | Not delivered |
| RD | Refused delivery |
| RT | Returned to consignor |
| OH | On hold |
| IC | In clearance processing |
| CR | Clearance release |

Do not assume a one-to-one mapping to SRS statuses.

Recommended architecture:

```text
DHL event
   ↓
DHL adapter/status mapper
   ↓
Common SRS status
   ↓
S4 event
```

---

# 16. End-to-End Germany Return Flow

```text
Customer requests return
        ↓
SRS
        ↓
Generate/retrieve RMA
        ↓
Determine Germany → DHL
        ↓
Build DHL Shipment Request
        │
        ├── RMA reference
        ├── Customer/origin address
        ├── HP destination
        ├── DHL account
        ├── Product/service code
        ├── Package information
        └── Customs information if applicable
        ↓
DHL Shipment API
        ↓
DHL shipment/tracking identifier
        ↓
Associate:
RMA ↔ DHL tracking number
        ↓
Optional DHL Pickup
        ↓
DHL transports package
        ↓
DHL Tracking
        ↓
DHL tracking event
        ↓
SRS maps DHL event
        ↓
S4 callback/event
        ↓
S4/downstream systems
        ↓
HP receives return
```

---

# 17. FedEx → DHL API Mapping

| Existing SRS/FedEx function | DHL equivalent | Status |
|---|---|---|
| Return shipment creation | DHL Shipment | Confirmed capability |
| RMA | DHL reference type `RMA` | Confirmed capability |
| Authentication | MyDHL BasicAuth | Confirmed |
| Pickup | DHL Pickup | Confirmed capability |
| Tracking | DHL Tracking | Confirmed capability |
| Tracking URL | DHL customer-facing tracking URL | Needs confirmation |
| Rating | DHL Rating | Conditional |
| Carrier value | `DHL` | Needs S4 confirmation |
| Status mapping | DHL events → SRS statuses | Design required |

---

# 18. Field-Level Gap Analysis

| Purpose | Existing SRS/FedEx | DHL | Action |
|---|---|---|---|
| Return identifier | RMA | Reference type `RMA` | Validate exact REST mapping |
| Carrier | FEDEX | DHL | Make carrier configurable |
| Shipment | FedEx return shipment | DHL Shipment | Add DHL adapter |
| Authentication | Existing FedEx auth | BasicAuth | Add DHL credential configuration |
| Tracking | FedEx tracking | DHL Tracking | Add DHL tracking client |
| Tracking number | FedEx tracking number | DHL shipment/waybill identifier | Common carrier field if compatible |
| Tracking URL | FedEx URL | DHL URL | Confirm ownership/format |
| Pickup | Existing FedEx behavior | DHL Pickup | Confirm business requirement |
| Rating | Existing/unknown | DHL Rating | Only if required |
| Product | FedEx service | DHL product code | Business confirmation |
| Weight | Existing/unknown | DHL package weight | Find source |
| Dimensions | Existing/unknown | DHL package dimensions | Find source |
| Customs | Existing/unknown | DHL customs fields | Confirm Germany requirement |
| S4 carrier | FEDEX | DHL | Validate S4 enum/contract |

---

# 19. Recommended Architecture

Do not simply replace:

```text
FEDEX
```

with:

```text
DHL
```

Instead use a carrier-specific abstraction:

```text
                 SRS Return Flow
                       ↓
              Carrier abstraction
                 /           \
                /             \
            FedEx             DHL
              ↓                ↓
        FedEx client       DHL client
              ↓                ↓
          FedEx API         MyDHL API
```

Common SRS logic:

- Return request
- RMA
- Return state
- Persistence
- Common event handling
- S4 orchestration

Carrier-specific logic:

- Authentication
- Shipment request
- Pickup
- Tracking
- Tracking URL
- Status mapping
- Carrier references

This supports:

```text
US       → FedEx
Germany  → DHL
Future   → Another carrier
```

---

# 20. Code Investigation Checklist

Before implementation, identify the actual code locations for:

## RMA

- Where RMA is generated
- Which class/method generates it
- Where it is stored
- How it is passed to FedEx
- How it is associated with tracking
- Whether “RMA token” is actually a business identifier or authentication token

## FedEx

- FedEx client class
- Return API method
- Request DTO
- Response DTO
- Authentication class
- Token handling
- Tracking method
- Tracking URL handling
- Error/retry logic

## DHL

Determine where to implement:

- DHL credentials
- Shipment request/response
- RMA reference
- Pickup
- Tracking
- DHL status mapping

## S4

Find:

- S4 event method
- Payload DTO
- Carrier field
- Tracking number field
- Tracking URL field
- RMA field
- Status/event field
- Any FEDEX hardcoding
- Downstream assumptions

---

# 21. Confirmed vs Needs Confirmation

| Statement | Classification |
|---|---|
| DHL provides Shipment capability | CONFIRMED |
| DHL provides Pickup capability | CONFIRMED |
| DHL provides Tracking capability | CONFIRMED |
| DHL provides Rating/Capability | CONFIRMED |
| MyDHL uses Basic Authentication | CONFIRMED |
| DHL supports `RMA` as RMA Number reference | CONFIRMED |
| `CU` means Consignor reference number | CONFIRMED |
| Pickup is different from the business Return | CONFIRMED |
| DHL has a separate return-specific Shipment endpoint | NOT IDENTIFIED |
| Customer must always be shipper | NEEDS CONFIRMATION |
| HP must always be receiver | NEEDS CONFIRMATION |
| Product code `U` is correct for Germany | NEEDS CONFIRMATION |
| Exact customer-facing tracking URL | NEEDS CONFIRMATION |
| DHL tracking number has a fixed length | DO NOT ASSUME |
| S4 accepts `DHL` as carrier | NEEDS CONFIRMATION |
| S4 constructs tracking URL | NEEDS CONFIRMATION |
| Rating is required by SRS | NEEDS CONFIRMATION |
| Weight/dimensions must be added to current model | NEEDS CONFIRMATION |

---

# 22. Questions for Technical Discussion

## DHL

1. For Germany returns, should the customer be the shipment origin/shipper and HP the receiver?
2. Which DHL product code should be used?
3. Is courier pickup required?
4. Should pickup be requested through Shipment or the separate Pickup operation?
5. What exact REST field/path should contain the SRS RMA using `typeCode = RMA`?
6. What customer-facing DHL tracking URL should be sent to S4?
7. Which shipment fields are mandatory for the Germany return?
8. Where should weight and dimensions come from?
9. What customs information is required?
10. What payer/account configuration should be used?

## S4

11. Can `carrierName` be `DHL`?
12. Is the carrier field an enum?
13. Does S4 validate tracking number format?
14. Does S4 accept a DHL tracking URL?
15. Does S4 generate the tracking URL?
16. Are any downstream systems hardcoded for FedEx?
17. Can the existing S4 contract be reused?

## Architecture

18. Can existing SRS RMA logic remain unchanged?
19. Can a carrier-specific DHL adapter be added?
20. Can the existing persistence model store DHL tracking identifiers?
21. Can carrier selection be country/configuration based?

---

# 23. Final Recommendation

The Germany implementation should preserve the common SRS return/RMA flow and introduce DHL-specific carrier integration.

The recommended model is:

```text
SRS Return
   ↓
RMA remains common
   ↓
Country = Germany
   ↓
Carrier = DHL
   ↓
DHL BasicAuth
   ↓
DHL Shipment
   ↓
DHL tracking/waybill number
   ↓
Optional DHL Pickup
   ↓
DHL Tracking
   ↓
DHL status → SRS status
   ↓
S4
   ↓
carrier = DHL
tracking number = DHL identifier
tracking URL = confirmed DHL customer URL
```

The existing RMA should remain the business return identifier. DHL explicitly supports `RMA` as a reference type, so the existing SRS RMA can potentially be carried into DHL without creating a new DHL-specific business identifier. The exact REST mapping must be validated against the final MyDHL v3.3.1 schema.

Authentication is separate from RMA. DHL MyDHL uses BasicAuth credentials, so the FedEx OAuth/token lifecycle should not be copied into the DHL implementation.

Pickup is optional and depends on the Germany return business process. Tracking is a separate carrier capability after shipment creation. The customer-facing tracking URL must be confirmed rather than hardcoded from an assumed URL pattern.

S4 should preferably continue using the existing event contract, with the carrier value becoming `DHL`, but S4 and all downstream consumers must confirm that this value is accepted.

---

# 24. Official DHL References

DHL MyDHL API:

https://developer.dhl.com/api-reference/dhl-express-mydhl-api

DHL MyDHL API XML reference:

https://developer.dhl.com/api-reference/mydhl-api-xml-dhl-express

DHL MyDHL API Reference Data Guide:

https://developer.dhl.com/sites/default/files/2025-10/MyDHL%20API%20Reference%20data%20guide%20%283.1.0%29.pdf

---

## Source

This document is a companion to:

`FedEx-vs-DHL-Gap-Analysis-Complete.md`

It intentionally keeps the existing document unchanged and provides a cleaner analysis that separates:

- Confirmed DHL capabilities
- Existing SRS/FedEx behavior
- Proposed mapping
- Implementation gaps
- Items requiring confirmation

**Target:** SRS Germany DHL Express Return Integration
