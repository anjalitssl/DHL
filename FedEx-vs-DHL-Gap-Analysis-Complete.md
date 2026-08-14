# FedEx vs DHL Express - Complete Gap Analysis for Germany Expansion
**SRS (Stratus Returns Service) - Germany Region Implementation**


**Current Implementation:** FedEx (USA)  
**Target Implementation:** DHL Express MyDHL API v3.3.1 (Germany)

---

## Executive Summary

This document provides a comprehensive gap analysis between the current FedEx RMA implementation (USA) and the proposed DHL Express MyDHL API integration (Germany) for the SRS (Stratus Returns Service). 

### Key Requirements from Meeting (3 Critical Areas):

1. **Tracking URL** - How tracking will work with DHL
2. **Token/Authentication** - DHL's authentication mechanism
3. **RMA (Return Merchandise Authorization)** - How RMA is handled in DHL

### Critical Finding:

**DHL Express MyDHL API does NOT have a dedicated "Return API"**. Returns are handled as **standard shipments** with **reversed shipper/receiver roles**:
- **Customer** = Shipper (origin)
- **HP/Warehouse** = Receiver (destination)

---

## Table of Contents

1. [Current FedEx Implementation Analysis](#current-fedex-implementation)
2. [DHL Express API Structure Analysis](#dhl-express-api-structure)
3. [Three Critical Requirements Analysis](#three-critical-requirements) — Tracking URL, Authentication, RMA
4. [Field-by-Field Mapping Comparison](#field-mapping)
5. [Gap Analysis & Missing Fields](#gap-analysis)
6. [CONFIRMED vs NEEDS CONFIRMATION (Critical Validation Items)](#confirmed-vs-needs-confirmation)
7. [S4 Integration Impact](#s4-integration) / [7.1 RMA Codebase Investigation](#rma-codebase-investigation) / Label Analysis — FedEx vs DHL (labelURL, Base64, persistence options)
8. [FINAL AUDIT PASS — Additional Mandatory Analysis Areas](#final-audit-pass) — 8.1 Carrier Selection/Architecture, 8.2 Rating/Capability, 8.3 Germany Label Free/QR, 8.4 Pickup/Drop-off/Service Point, 8.5 Customs, 8.6 Error Handling/Retry/Idempotency, 8.7 Configuration/Secrets, 8.8 Observability, 8.9 Backward Compatibility, 8.10 Testing Impact
9. [FINAL Consolidated Deliverables](#final-consolidated-deliverables) — 9.1 Final Gap Table, 9.2 Final Implementation Impact, 9.3 Final Open Questions (owner-based), 9.4 Final End-to-End Germany Flow, 9.5 Contradiction Resolution Log, 9.6 FINAL CONFIDENCE/READINESS

**Scope note:** This analysis covers the **Germany region only**. The existing US FedEx flow is NOT being replaced — Sections 8.1 and 8.9 explicitly confirm the additive, backward-compatible design (`InitReturnFedexFlow` stays untouched; a new `InitReturnDhlFlow` is added alongside it).

---

## 1. Current FedEx Implementation Analysis

### **FedEx API Endpoint**
```
POST https://connect.supplychain.fedex.com/api/v2/rmas
```

### **FedEx Request Payload Structure**

```json
{
  "rmaNumber": "RET-20260813-001",
  "customer": {
    "firstName": "John",
    "lastName": "Doe",
    "emailAddress": "customer@example.com"
  },
  "shipToAddressId": 123,
  "emailNotification": true,
  "orders": [
    {
      "orderNumber": "FO-123456789",
      "orderDate": "2026-08-01",
      "items": [
        {
          "sku": "2K4V0A#ABA",
          "quantity": 1,
          "returnItemInfo": {
            "returnReason": "Damaged"
          }
        }
      ]
    }
  ]
}
```

### **FedEx Response Payload**

```json
{
  "success": "true",
  "requestIdentifier": "req-abc123",
  "transactionDate": "2026-08-13T10:30:00Z",
  "rma": {
    "rmaNumber": "38186",
    "shipments": [
      {
        "label": {
          "trackingNumber": "794672887344",
          "labelURL": "https://api-sandbox.supplychain.fedex.com/..."
        }
      }
    ]
  }
}
```

### **FedEx Key Characteristics**

| Aspect | FedEx Implementation |
|--------|---------------------|
| **API Type** | Dedicated RMA/Return API |
| **Authentication** | OAuth2 Password Grant Flow |
| **Token Management** | Required - FedexTokenHolder class |
| **RMA Field** | Root-level `rmaNumber` |
| **Address** | Lookup by `shipToAddressId` (configured) |
| **Items** | SKU + quantity based |
| **Tracking** | Numeric 12-digit tracking number |

### **Current Code Flow**

```
ReturnDetailsController.sendReturn()
    ↓
ReturnDetailsService.sendReturn()
    ↓
resolveInitReturnFlow() [LaunchDarkly: "Fedex"]
    ↓
InitReturnFedexFlow.initReturn()
    ↓
FedexClient.createRmaS4()
    ↓
POST to FedEx RMA API
    ↓
Store tracking number & RMA
```

---

## 2. DHL Express API Structure Analysis

### **DHL Express API Endpoint**
```
POST https://express.api.dhl.com/mydhlapi/shipments
```

⚠️ **Critical:** DHL uses **Shipment API** (not RMA API) for returns!

### **DHL Express Request Payload Structure (from provided screenshots)**

```json
{
  "productCode": "U",
  "plannedShippingDateAndTime": "2026-08-05T16:00:00GMT+01:00",
  "pickup": {
    "isRequested": false
  },
  "accounts": [
    {
      "number": "304xxxxxx",
      "typeCode": "shipper"
    }
  ],
  "customerReferences": [
    {
      "value": "20219876",
      "typeCode": "CU"
    }
  ],
  "outputImageProperties": {
    "encodingFormat": "pdf",
    "imageOptions": [
      {
        "typeCode": "qr-code",
        "isRequested": true
      }
    ]
  },
  "customerDetails": {
    "shipperDetails": {
      "postalAddress": {
        "cityName": "Prague",
        "countryCode": "CZ",
        "postalCode": "15000",
        "addressLine1": "Vaclavske namesti"
      },
      "contactInformation": {
        "phone": "+420220300111",
        "companyName": "DHL",
        "fullName": "DHL Express",
        "email": "dhl@dhl.com"
      }
    },
    "receiverDetails": {
      "postalAddress": {
        "cityName": "Barcelona",
        "countryCode": "ES",
        "postalCode": "08001",
        "addressLine1": "Rbelot"
      },
      "contactInformation": {
        "phone": "0795632404",
        "companyName": "Test Dhlapi",
        "fullName": "Test Dhlapi",
        "email": "test@dhlapi.com"
      }
    }
  },
  "content": {
    "unitOfMeasurement": "metric",
    "isCustomsDeclarable": false,
    "incoterm": "DAP",
    "description": "Testing Equipment",
    "packages": [
      {
        "weight": 1.552,
        "dimensions": {
          "length": 20,
          "width": 12,
          "height": 8
        }
      }
    ]
  },
  "valueAddedServices": [
    {
      "serviceCode": "PZ"
    },
    {
      "serviceCode": "PT"
    }
  ]
}
```

### **DHL Express Response Payload Structure**

```json
{
  "shipmentTrackingNumber": "1234567890",
  "trackingUrl": "https://mydhl.express.dhl/...",
  "packages": [
    {
      "referenceNumber": 1,
      "trackingNumber": "JD999999999999999999"
    }
  ],
  "documents": [
    {
      "imageFormat": "PDF",
      "content": "JVBERi0xLjQK..."
    }
  ]
}
```

### **DHL Express Key Characteristics**

| Aspect | DHL Express Implementation |
|--------|---------------------------|
| **API Type** | Generic Shipment API (no dedicated return API) |
| **Authentication** | HTTP Basic Auth (simpler!) |
| **Token Management** | NOT required - credentials per request |
| **RMA Field** | `customerReferences[typeCode='CU'].value` |
| **Address** | Full address object required (no lookup) |
| **Items** | Package-based (weight + dimensions) |
| **Tracking** | Alphanumeric 10-digit tracking number |

---

## 3. Three Critical Requirements Analysis

### **Requirement 1: Tracking URL**

#### **FedEx Tracking:**

**Tracking Number Format:** 12-digit numeric  
**Example:** `794672887344`

**Tracking URL:**
```
https://www.fedex.com/fedextrack/?tracknumbers=794672887344
```

**Current Code:**
```java
// FedexClient.java - line 48
payload.setTrackingUrl(
    fedexConfig.getTrackingUrl() + trackingNumber
);
```

**Storage:**
```java
returnDetailsDocument.rma.shipments[0].label.trackingNumber = "794672887344"
```

---

#### **DHL Tracking:**

**Tracking Number Format:** 10-digit alphanumeric  
**Example:** `1234567890` or `JD999999999999999999` (package level)

**Tracking URL:**
```
https://www.dhl.com/express/track?AWB=1234567890
```

**From DHL Documentation :**

| Endpoint Type | Test URL | Production URL |
|---------------|----------|----------------|
| All shipment statuses | `https://express.api.dhl.com/mydhlapi/test/shipments/{trackingNumber}/tracking` | `https://express.api.dhl.com/mydhlapi/shipments/{trackingNumber}/tracking` |
| Shipment with last checkpoint | `https://express.api.dhl.com/mydhlapi/test/shipments/{trackingNumber}/tracking?trackingView=last-checkpoint` | `https://express.api.dhl.com/mydhlapi/shipments/{trackingNumber}/tracking?trackingView=last-checkpoint` |
| All checkpoints with remarks | `https://express.api.dhl.com/mydhlapi/test/tracking?shipmentTrackingNumber={trackingNumber}&trackingView=all-checkpoints-with-remarks` | `https://express.api.dhl.com/mydhlapi/tracking?shipmentTrackingNumber={trackingNumber}&trackingView=all-checkpoints-with-remarks` |

**DHL Tracking Event Codes :**

| Code | Description |
|------|-------------|
| AD | Agreed delivery |
| AF | Arrived facility |
| AR | Arrival in delivery facility |
| BA | Bad address |
| BN | Customer broker notified |
| BR | Broker release |
| CA | Closed on arrival |
| CC | Awaiting cnee collection |
| CD | Controllable clearance delay |
| CM | Customer moved |
| CR | Clearance release |
| CS | Closed shipment |
| DD | Delivered damaged |
| DF | Depot Facility |
| DS | Destroyed / disposal |
| EM | Electronic Data Merge |
| FD | Forward destination (DD's expected) |
| HP | Held for payment |
| IC | In clearance processing |
| MC | Miscode |
| MD | Missed delivery cycle |
| MS | Mis-sort |
| ND | Not delivered |
| NH | Not home |
| OH | On hold |
| **OK** | **Delivery** ✅ |
| PD | Partial delivery |
| PL | Processed at location |
| PU | Shipment pick up |
| PY | Payment |
| RD | Refused delivery |
| RR | Response received |
| RT | Returned to consignor |
| SA | Shipment acceptance |
| SC | State changed |
| SD | Shipment information received |
| SM | Scheduled movement |
| SS | Shipment stopped |
| TP | Forwarded to 3rd party - no DD's |
| TR | Record of transfer |
| UD | Uncontrollable clearance delay |
| WC | With delivering courier |

**⚠️ Gap Identified:**
- FedEx tracking URL format ≠ DHL tracking URL format
- DHL has more detailed checkpoint codes
- Need to store correct tracking URL per carrier

**Code Change Required:**
```java
// New: DhlClient.java
public String buildTrackingUrl(String trackingNumber) {
    return "https://www.dhl.com/express/track?AWB=" + trackingNumber;
}

// Store in return document
payload.setTrackingUrl(dhlClient.buildTrackingUrl(trackingNumber));
payload.setShippingVendor("DHL");  // Critical for S4!
```

---

### **Requirement 2: Token/Authentication**

#### **FedEx Authentication (Current):**

**Type:** OAuth2 Password Grant Flow

**Token Endpoint:**
```
POST https://connect.supplychain.fedex.com/api/fsc/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
&scope=Fulfillment_Returns
&username={ORG_NAME}
&password={PASSWORD}
&client_id={CLIENT_ID}
&client_secret={CLIENT_SECRET}
```

**Token Response:**
```json
{
  "access_token": "eyJhbGciOiJS...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Current Implementation:**
```java
// FedexTokenHolder.java
public class FedexTokenHolder {
    private String token;
    private LocalDateTime expiryTime;
    
    public synchronized String getFedExAuthToken() {
        if (isTokenExpired()) {
            refreshToken();
        }
        return token;
    }
}

// FedexClient.java
HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Bearer " + token);
```

**Credentials Required (5):**
- `FEDEX_CLIENT_ID`
- `FEDEX_CLIENT_SECRET`
- `FEDEX_ORG_NAME`
- `FEDEX_PASSWORD`
- `FEDEX_XORG_NAME`

---

#### **DHL Authentication (Target):**

**Type:** HTTP Basic Authentication

⚠️ **Critical:** **NO TOKEN MANAGEMENT NEEDED!**

**Authentication per Request:**
```java
String auth = apiKey + ":" + apiSecret;
String encodedAuth = Base64.getEncoder()
    .encodeToString(auth.getBytes(StandardCharsets.UTF_8));

HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Basic " + encodedAuth);
headers.set("Content-Type", "application/json");
```

**Credentials Required (3 - from DHL email):**
- `DHL_API_KEY` (username)
- `DHL_API_SECRET` (password)
- `DHL_ACCOUNT_NUMBER`

**How to Obtain (from DHL email):**
> "To connect to the API, you need a DHL Express account, endpoint, and API credentials. Please send us the following information so we can generate and send the API credentials to your email."

**Required Information:**
- Customer DHL Express Accounts
- Customer Contact Name
- Customer Email
- Customer Telephone
- Customer Address Line
- Customer City
- Customer Postcode

**⚠️ Major Simplification:**
- ❌ No `FedexTokenHolder` equivalent needed
- ❌ No token refresh logic
- ❌ No token expiry handling
- ✅ Simpler implementation
- ✅ Stateless authentication

**Code Implementation:**
```java
// DhlConfig.java
@ConfigurationProperties(prefix = "dhl.express")
public class DhlConfig {
    private String apiKey;
    private String apiSecret;
    private String accountNumber;
    private String baseUrl;
    private String trackingUrl;
}

// DhlClient.java
public class DhlClient {
    private final DhlConfig dhlConfig;
    
    private HttpHeaders buildAuthHeaders() {
        String auth = dhlConfig.getApiKey() + ":" + dhlConfig.getApiSecret();
        String encodedAuth = Base64.getEncoder()
            .encodeToString(auth.getBytes(StandardCharsets.UTF_8));
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Basic " + encodedAuth);
        headers.set("Content-Type", "application/json");
        return headers;
    }
}
```

---

### **Requirement 3: RMA (Return Merchandise Authorization)**

#### **FedEx RMA Handling (Current):**

**RMA Location:** Root level of payload
```json
{
  "rmaNumber": "RET-20260813-001"
}
```

**Source Code:**
```java
// FedexClient.java - line 152
payload.setRmaNumber(returnDetailsDocument.getRetReturnNumber());

// Validation
if (payload.getRmaNumber().length() > 20) {
    throw new RmaNumberLengthException(
        "RmaNumber length should not be greater than 20!"
    );
}
```

**Storage:**
```java
returnDetailsDocument.setRetReturnNumber("RET-20260813-001")
```

**FedEx Response:**
```json
{
  "rma": {
    "rmaNumber": "38186",
    "shipments": [...]
  }
}
```

---

#### **DHL RMA Handling (Target):**


```json
{
  "customerReferences": [
    {
      "value": "RET-20260813-001",
      "typeCode": "RMA"
    }
  ]
}
```

**From DHL API Documentation:**
> **IMPORTANT CORRECTION:** DHL explicitly defines `RMA = RMA Number` as a reference type. Do NOT assume `CU = Consignor reference number` is used for RMA. CU is a different reference type. Use `typeCode="RMA"` for return identifiers.

**⚠️ Critical Understanding:**

| Concept | Description | Location | Example |
|---------|-------------|----------|---------|
| **RMA** | Business return identifier | `customerReferences[typeCode='RMA'].value` | `"RET-20260813-001"` |
| **DHL Tracking Number** | Physical shipment identifier | Response: `shipmentTrackingNumber` | `"1234567890"` |
| **DHL API Credentials** | Authentication | Header: `Authorization: Basic <base64>` | N/A |

**These are 3 SEPARATE concepts - do NOT confuse!**

**Code Implementation:**
```java
// DhlShipmentRequest.java
@Data
public class DhlShipmentRequest {
    private List<CustomerReference> customerReferences;
    
    @Data
    public static class CustomerReference {
        private String value;      // RMA number
        private String typeCode;   // "RMA" (NOT "CU" - CU is Consignor reference)
    }
}

// DhlClient.java
public DhlShipmentRequest buildRequest(ReturnDetailsDocument doc) {
    DhlShipmentRequest request = new DhlShipmentRequest();
    
    // Add RMA as customer reference
    CustomerReference rma = new CustomerReference();
    rma.setValue(doc.getRetReturnNumber());
    rma.setTypeCode("RMA");  // CRITICAL: Use "RMA" not "CU" per DHL spec
    request.getCustomerReferences().add(rma);
    
    // ... rest of mapping
}
```

---

## 4. Field-by-Field Mapping Comparison

### **Root Level Fields**

| Field Purpose | FedEx Field | FedEx Type | DHL Express Field | DHL Type | Mapping Complexity |
|---------------|-------------|------------|-------------------|----------|--------------------|
| **RMA/Return ID** | `rmaNumber` | String (max 20) | `customerReferences[typeCode='RMA'].value` | String | 🟡 Medium - Different structure |
| **Product/Service** | N/A (implicit) | N/A | `productCode` | String | 🔴 High - New required field |
| **Shipping Date** | N/A (implicit) | N/A | `plannedShippingDateAndTime` | ISO DateTime | 🔴 High - New required field |
| **Pickup Request** | N/A (implicit) | N/A | `pickup.isRequested` | Boolean | 🟢 Low - Simple boolean |
| **Account Info** | N/A (from config) | N/A | `accounts[].number`, `accounts[].typeCode` | Array | 🟡 Medium - New structure |
| **Email Notification** | `emailNotification` | Boolean | Not direct equivalent | N/A | 🔴 High - Different approach |
| **Output Format** | N/A (implicit PDF) | N/A | `outputImageProperties` | Object | 🟡 Medium - New structure |

### **Customer/Address Mapping**

| Field Purpose | FedEx Field | DHL Express Field | Gap Analysis |
|---------------|-------------|-------------------|--------------|
| **Customer Email** | `customer.emailAddress` | `customerDetails.shipperDetails.contactInformation.email` | Path more nested |
| **Customer Name** | `customer.firstName` + `customer.lastName` | `customerDetails.shipperDetails.contactInformation.fullName` | Single field vs two fields |
| **Customer Phone** | Not in current payload | `customerDetails.shipperDetails.contactInformation.phone` | **MISSING in FedEx** |
| **Customer Company** | Not in current payload | `customerDetails.shipperDetails.contactInformation.companyName` | **MISSING in FedEx** |
| **HP Return Address** | `shipToAddressId` (lookup ID) | `customerDetails.receiverDetails.postalAddress.*` (full object) | **Major difference** |
| **City** | Via address lookup | `postalAddress.cityName` | Need to provide explicitly |
| **Country** | Via address lookup | `postalAddress.countryCode` (ISO 2-char) | Need to provide explicitly |
| **Postal Code** | Via address lookup | `postalAddress.postalCode` | Need to provide explicitly |
| **Address Line** | Via address lookup | `postalAddress.addressLine1` | Need to provide explicitly |

**⚠️ CRITICAL GAP: FedEx uses address LOOKUP (`shipToAddressId: 123`), DHL requires FULL ADDRESS**

**Current FedEx Address Handling:**
```java
// FedexClient.java - lines 241-242
ReturnConfigDocument address = returnConfigRepository
    .getByKey("s4ShipToAddressId")
    .orElseThrow(() -> new FedexPreconditionFailedException(
        "Shipment address not found!"
    ));

Integer addressId = (Integer) address.getValue().get(0).get("value");
payload.setShipToAddressId(addressId);  // Just an ID!
```

**Required DHL Address Handling:**
```java
// Need FULL address from config
ReturnConfigDocument receiverAddress = returnConfigRepository
    .getByKey("dhlReceiverAddress")
    .orElseThrow(...);

ReceiverDetails receiver = new ReceiverDetails();
receiver.getPostalAddress().setCityName("Memphis");
receiver.getPostalAddress().setCountryCode("US");
receiver.getPostalAddress().setPostalCode("38101");
receiver.getPostalAddress().setAddressLine1("DHL Return Center");
receiver.getContactInformation().setCompanyName("HP Returns");
receiver.getContactInformation().setEmail("returns@hp.com");
receiver.getContactInformation().setPhone("+1234567890");
receiver.getContactInformation().setFullName("HP Returns Department");
```

### **Order/Items Mapping**

| Field Purpose | FedEx Structure | DHL Express Structure | Gap |
|---------------|-----------------|----------------------|-----|
| **Order Number** | `orders[].orderNumber` | Custom reference or package customer reference | Can map |
| **Order Date** | `orders[].orderDate` | Not directly in shipment | Minor |
| **Items** | `orders[].items[]` (SKU-based) | `content.packages[]` (physical packages) | **Major difference** |
| **SKU** | `items[].sku` | Not in DHL shipment (can add as reference) | Different model |
| **Quantity** | `items[].quantity` | Implicit in package count | Different model |
| **Return Reason** | `items[].returnItemInfo.returnReason` | `content.description` (text field) | Can map to description |

**⚠️ MAJOR STRUCTURAL DIFFERENCE:**

**FedEx Model:** Item-centric (SKU + quantity)
```json
{
  "orders": [{
    "items": [
      {"sku": "2K4V0A#ABA", "quantity": 1, "returnItemInfo": {"returnReason": "Damaged"}}
    ]
  }]
}
```

**DHL Model:** Package-centric (weight + dimensions)
```json
{
  "content": {
    "description": "HP Printer Return - Damaged",
    "packages": [
      {
        "weight": 2.5,
        "dimensions": {"length": 25, "width": 20, "height": 15}
      }
    ]
  }
}
```

**Code Impact:**
```java
// FedEx (current)
returnDetailsDocument.getItems().stream()
    .filter(item -> item.getStatus() == ReturnState.processing)
    .forEach(item -> {
        Map<String, Object> items = new HashMap<>();
        items.put("sku", item.getSku());
        items.put("quantity", item.getQuantity());
        itemArray.add(items);
    });

// DHL (needed)
List<Package> packages = new ArrayList<>();
returnDetailsDocument.getItems().stream()
    .filter(item -> item.getStatus() == ReturnState.processing)
    .forEach(item -> {
        Package pkg = new Package();
        pkg.setWeight(item.getWeight());  // ⚠️ Need to add weight field!
        pkg.getDimensions().setLength(item.getLength());  // ⚠️ Need dimension fields!
        pkg.getDimensions().setWidth(item.getWidth());
        pkg.getDimensions().setHeight(item.getHeight());
        packages.add(pkg);
    });
```

**⚠️ CRITICAL MISSING DATA:**
- Current `Item` model does NOT have weight/dimensions
- Need to either:
  1. Add weight/dimensions to Item model
  2. Use default/estimated values per SKU
  3. Fetch from product catalog

---

## 5. Gap Analysis & Missing Fields

### **Fields Present in FedEx but NOT in DHL**

| FedEx Field | Purpose | Can It Be Removed? | Impact |
|-------------|---------|-------------------|--------|
| `emailNotification` | Enable/disable email | Yes | DHL handles differently |
| `shipToAddressId` | Address lookup ID | Yes | DHL uses full address |
| `orders[].orderDate` | Order date | Yes | Can put in description/reference |
| `items[].sku` | Product SKU | No | Need as reference or description |
| `items[].quantity` | Item quantity | No | Affects package count |

### **Fields Present in DHL but NOT in FedEx**

| DHL Field | Required? | Purpose | Source | Impact |
|-----------|-----------|---------|--------|--------|
| `productCode` | ✅ Yes | DHL service type (U, P, N, etc.) | Config/determination | 🔴 **HIGH** - Need to determine |
| `plannedShippingDateAndTime` | ✅ Yes | When shipment ready | System date + time | 🟡 Medium |
| `pickup.isRequested` | ✅ Yes | Request pickup | Business logic | 🟡 Medium |
| `accounts[].number` | ✅ Yes | DHL account number | Config | 🟢 Low - from credentials |
| `content.packages[].weight` | ✅ Yes | Package weight | **MISSING** | 🔴 **HIGH** |
| `content.packages[].dimensions` | ✅ Yes | L x W x H | **MISSING** | 🔴 **HIGH** |
| `content.unitOfMeasurement` | ✅ Yes | "metric" or "imperial" | Config | 🟢 Low |
| `content.isCustomsDeclarable` | ✅ Yes | Customs needed? | Country logic | 🟡 Medium |
| `content.incoterm` | ✅ Yes | Trade terms (DAP, DDP, etc.) | Config | 🟡 Medium |
| `customerDetails.shipperDetails.contactInformation.phone` | ✅ Yes | Customer phone | **MISSING** | 🔴 **HIGH** |
| `customerDetails.shipperDetails.contactInformation.companyName` | No | Company name | Not captured | 🟡 Medium |
| `customerDetails.receiverDetails.*` | ✅ Yes | Full HP address | Need config | 🔴 **HIGH** |
| `outputImageProperties` | ✅ Yes | Label format | Config | 🟢 Low |
| `valueAddedServices` | No | Optional services (QR, etc.) | Config | 🟢 Low |

### **Critical Missing Data in Current System**

| Missing Data | Where Needed | Current Availability | Solution |
|--------------|--------------|---------------------|----------|
| **Package Weight** | `content.packages[].weight` | ❌ Not in Item model | Add field OR use default/catalog |
| **Package Dimensions** | `content.packages[].dimensions` | ❌ Not in Item model | Add field OR use default/catalog |
| **Customer Phone** | `shipperDetails.contactInformation.phone` | ❌ Not captured | Add to user model OR make optional |
| **HP Return Address** | `receiverDetails` full object | ⚠️ Only have ID | Create config with full address |
| **DHL Product Code** | `productCode` | ❌ Not determined | Add business logic to determine |
| **Incoterm** | `content.incoterm` | ❌ Not configured | Add config (likely "DAP") |

---

## 6. CONFIRMED vs NEEDS CONFIRMATION (Critical Validation Items)

### **Reference Source:** Official DHL MyDHL API Reference data guide & implementation analysis

This section tracks which statements are CONFIRMED from official documentation vs which NEED business/technical confirmation before implementation.

### **CONFIRMED Items (Based on Official DHL Documentation):**

| Item | Statement | Source |
|---|---|---|
| ✅ | DHL provides Shipment API capability | Official DHL MyDHL API v3.3.1 |
| ✅ | DHL provides Pickup API capability | Official DHL MyDHL API v3.3.1 |
| ✅ | DHL provides Tracking API capability | Official DHL MyDHL API v3.3.1 |
| ✅ | DHL provides Rating/Capability API | Official DHL MyDHL API v3.3.1 |
| ✅ | MyDHL uses HTTP Basic Authentication | Official DHL MyDHL API v3.3.1 |
| ✅ | DHL supports `RMA` as RMA Number reference type | DHL Reference data guide (2025-10) |
| ✅ | `CU` means Consignor reference number (NOT RMA) | DHL Reference data guide (2025-10) |
| ✅ | Pickup is different from Return business process | Business logic distinction |
| ✅ | DHL has NO separate return-specific Shipment endpoint | Official API analysis |
| ✅ | Return uses generic Shipment API with reversed roles | Architecture design pattern |

### **NEEDS CONFIRMATION Items (Business/Technical Decision Points):**

| # | Item | Status | Who Confirms | Impact |
|---|---|---|---|---|
| 1 | Customer must always be shipper in return shipment | ⏳ PENDING | DHL Team | HIGH |
| 2 | HP/Warehouse must always be receiver | ⏳ PENDING | Business | HIGH |
| 3 | Product code `N` is correct for Germany domestic returns | ⏳ PENDING | DHL Team | HIGH |
| 4 | Product code `U` is correct for Germany EU returns | ⏳ PENDING | DHL Team | HIGH |
| 5 | Exact customer-facing DHL tracking URL format | ⏳ PENDING | DHL Team | MEDIUM |
| 6 | S4 system accepts `"DHL"` as carrier value | ⏳ PENDING | S4 Team | **CRITICAL** |
| 7 | S4 validates tracking number format | ⏳ PENDING | S4 Team | MEDIUM |
| 8 | S4 constructs tracking URL OR accepts from SRS | ⏳ PENDING | S4 Team | MEDIUM |
| 9 | Courier pickup required for Germany returns | ⏳ PENDING | Business | MEDIUM |
| 10 | Weight and dimensions available in current model | ⏳ PENDING | Product Team | **CRITICAL** |
| 11 | Customer phone number is mandatory for DHL | ⏳ PENDING | DHL Team | MEDIUM |
| 12 | Any downstream systems hardcoded for FEDEX | ⏳ PENDING | Architecture | MEDIUM |

### **DO NOT ASSUME Items (Treat as Variable):**

| ⚠️ Item | Reason |
|---|---|
| DHL tracking number has fixed length | DHL spec doesn't guarantee length - treat as opaque identifier |
| Product code is same across all regions | Confirm for each region |
| Tracking URL format is same as FedEx | DHL format is different |
| Weight/dimensions use same units as FedEx | Confirm metric vs imperial with DHL |
| Pickup API vs Shipment API usage | Business must determine |

### **Action Items:**

🔴 **CRITICAL - Before Engineering Starts:**
- [ ] Confirm RMA typeCode = "RMA" (NOT "CU")
- [ ] Confirm S4 accepts "DHL" carrier value
- [ ] Identify weight/dimensions source or default strategy
- [ ] Confirm required DHL product code(s)

🟡 **Important - Week 1:**
- [ ] Validate all NEEDS CONFIRMATION items from table above
- [ ] Complete code investigation checklist
- [ ] Get S4 contract/schema documentation

---

## 7. S4 Integration Impact

### **From Call Transcript:**
> Kumar: "Just find out when we are triggering that event or API call to S4 to initiate the shipment, right? What payload we are passing? Okay, so even if we are not using this DHL shipment directly or S4 need to implement that, but we need to pass that information to S4 in that case. What would be the difference in our payload that we pass to S4?"

### **Current S4 Payload Analysis**

**Key Question:** Does current S4 payload contain `shippingVendor`/`carrier` field?

**From Replacement Shipment Events (sample data):**
```json
{
  "shipmentDetails": {
    "carrierTrackingNumber": "738979304299",
    "carrierName": "Fedex",
    "carrierTrackingURL": "https://www.fedex.com/fedextrack/?trknbr=738979304299"
  }
}
```

**:** Field `carrierName` exists!

### **Required S4 Payload Changes**

| Field | Current (FedEx) | Needed (DHL) | Change Required? |
|-------|-----------------|--------------|------------------|
| `carrierName` | `"Fedex"` | `"DHL"` | ✅ Yes - dynamic value |
| `carrierTrackingNumber` | `"738979304299"` (12 digits) | `"1234567890"` (10 digits) | ✅ Yes - different format |
| `carrierTrackingURL` | `"https://www.fedex.com/fedextrack/?trknbr=..."` | `"https://www.dhl.com/express/track?AWB=..."` | ✅ Yes - different URL |

### **S4 Integration Questions to Validate**

1. ✅ **Does S4 accept dynamic carrier names?**
   - If S4 has enum/validation for `carrierName`, need to add "DHL"
   
2. ✅ **Does S4 validate tracking number format?**
   - FedEx: 12-digit numeric
   - DHL: 10-digit alphanumeric
   - May need relaxed validation

3. ✅ **Does S4 construct tracking URL or accept from SRS?**
   - If SRS provides → just change URL format
   - If S4 constructs → S4 needs DHL URL template

4. ✅ **Are there downstream consumers of `carrierName`?**
   - Check if any system hardcodes "Fedex"
   - Check if any system needs DHL-specific handling

### **Proposed S4 Payload Structure**

```json
{
  "shipmentDetails": {
    "carrierName": "DHL",
    "carrierTrackingNumber": "1234567890",
    "carrierTrackingURL": "https://www.dhl.com/express/track?AWB=1234567890",
    "shippingVendor": "DHL",
    "region": "Germany"
  }
}
```

**Code Change:**
```java
// ReturnDetailsService.java
payload.setShippingVendor(launchDarklyClient.getShipmentProviderFlag());

// When sending to S4
s4Payload.setCarrierName(payload.getShippingVendor());  // "DHL"
s4Payload.setCarrierTrackingNumber(payload.getRma().getShipments().get(0).getLabel().getTrackingNumber());
s4Payload.setCarrierTrackingURL(payload.getTrackingUrl());
```

---


---

## 7.1 RMA CODEBASE INVESTIGATION - COMPLETED ANALYSIS

This section documents findings from analyzing the actual SRS codebase to understand how RMA is currently implemented.

### **7.1.1 Where is RMA currently generated?**

**Finding:** RMA is **NOT generated** in SRS codebase. It's **received from external source** (customer/API request).

**Evidence:**
- **File:** `src/main/java/com/hp/tropos/repository/model/ReturnDetailsDocument.java`
- **Line:** 119-120
- **Code:**
```java
@JsonProperty("retReturnNumber")
private String retReturnNumber;
```

**Explanation:** The field `retReturnNumber` receives the RMA value from the incoming API request. It's stored but not generated by SRS.

---

### **7.1.2 Which class/method creates the RMA identifier?**

**Finding:** RMA is **NOT created** by any SRS method. It's an input field.

**Storage:** `ReturnDetailsDocument.retReturnNumber`

**Validation occurs in:** `FedexClient.createRmaS4()` method
- **File:** `src/main/java/com/hp/tropos/clients/fedex/FedexClient.java`
- **Lines:** 152-155
- **Code:**
```java
payload.setRmaNumber(returnDetailsDocument.getRetReturnNumber());
if (payload.getRmaNumber().length() > 20) {
    throw new RmaNumberLengthException("RmaNumber length should not be greater than 20!");
}
```

**Pattern:** Validation happens during FedEx payload construction, not during RMA creation.

---

### **7.1.3 How is RMA stored in MongoDB ReturnDetailsDocument?**

**Finding:** RMA stored in TWO places in codebase:

**Primary Storage:**
- **Field Name:** `retReturnNumber`
- **Location:** `ReturnDetailsDocument.java:119-120`
- **Data Type:** String
- **Indexed:** NO (unlike `returnOrderId` which is unique-indexed at line 77)
- **JSON Property:** `"retReturnNumber"`

**Secondary Storage (FedEx Response):**
- **Field Name:** `rma` (nested object)
- **Location:** `ReturnDetailsDocument.java:67-68`
- **Type:** `Rma` class object
- **Contains:** `shipments[]` collection with tracking numbers

**MongoDB Collection:**
```java
@Document("returnDetails")  // Collection name: "returnDetails"
public class ReturnDetailsDocument {
    @JsonProperty("retReturnNumber")
    private String retReturnNumber;
    
    @JsonProperty("rma")
    private Rma rma;  // Contains FedEx response with rmaNumber + shipments
}
```

**Structure in MongoDB:**
```json
{
  "_id": "...",
  "retReturnNumber": "RET-20260813-001",      // Input RMA
  "rma": {
    "rmaNumber": "38186",                      // FedEx-generated RMA
    "shipments": [
      {
        "label": {
          "trackingNumber": "794672887344"     // FedEx tracking
        }
      }
    ]
  }
}
```

**Key Distinction:**
- `retReturnNumber` = Business RMA identifier (input)
- `rma.rmaNumber` = FedEx-generated RMA (response)
- These are TWO DIFFERENT VALUES

---

### **7.1.4 How is RMA passed to FedEx API client?**

**Finding:** RMA is passed via FedexPayload object during createRmaS4() call.

**Method:** `FedexClient.createRmaS4()` (Line 128)

**Step-by-step execution:**

1. **Input:** `ReturnDetailsDocument returnDetailsDocument` received as parameter

2. **Line 151-152:** Create payload and extract RMA:
```java
FedexPayload payload = new FedexPayload();
payload.setRmaNumber(returnDetailsDocument.getRetReturnNumber());
```

3. **Line 153-155:** Validate RMA length (max 20 chars):
```java
if (payload.getRmaNumber().length() > 20) {
    throw new RmaNumberLengthException("RmaNumber length should not be greater than 20!");
}
```

4. **Line 156:** Fill config values (customer, address):
```java
fillWithConfigValuesS4(payload, returnDetailsDocument);
```

5. **Line 159-181:** Build items and orders array

6. **Line 208:** Serialize FedexPayload to JSON:
```java
String json = objectMapper.writeValueAsString(payload);
```

7. **Line 209-210:** Send to FedEx API:
```java
HttpEntity<?> entity = new HttpEntity<>(json, headers);
ReturnDetailsDocument response = restTemplate.postForObject(
    fedexConfig.getCreateRmaUrl(), entity, ReturnDetailsDocument.class);
```

**FedEx Payload Structure (FedexPayload class):**
```json
{
  "rmaNumber": "RET-20260813-001",          // ← RMA at root level
  "customer": {...},
  "shipToAddressId": 123,
  "orders": [...]
}
```

**Key Point:** RMA is a **root-level field** in FedEx payload structure.

---

### **7.1.5 How is RMA associated with FedEx tracking number?**

**Finding:** RMA and tracking number are stored in SEPARATE object hierarchies, associated through response mapping.

**Class Hierarchy:**
```java
// ReturnDetailsDocument contains:
private Rma rma;

// Rma class contains:
class Rma {
    private String rmaNumber;              // FedEx-returned RMA
    private ArrayList<RmaShipment> shipments;  // Contains tracking
}

// RmaShipment contains:
class RmaShipment {
    private RmaShippmentLabel label;
}

// RmaShippmentLabel contains:
class RmaShippmentLabel {
    private String trackingNumber;         // "794672887344"
}
```

**File Locations:**
- `src/main/java/com/hp/tropos/repository/model/Rma.java`
- `src/main/java/com/hp/tropos/repository/model/RmaShipment.java`
- `src/main/java/com/hp/tropos/repository/model/RmaShippmentLabel.java`

**Association Flow:**
```
Input RMA: ReturnDetailsDocument.retReturnNumber
    ↓
Send to FedEx API
    ↓
FedEx returns: { "rma": { "rmaNumber": "...", "shipments": [...] } }
    ↓
Response mapped to: ReturnDetailsDocument.rma
    ↓
Tracking at: ReturnDetailsDocument.rma.shipments[0].label.trackingNumber
```

---

### **7.1.6 Is "RMA" term used as business identifier or authentication token?**

**Finding:** RMA is EXCLUSIVELY a **BUSINESS IDENTIFIER**, NOT an authentication token.

**Evidence:**

1. **Stored as data field** (Line 119):
```java
@JsonProperty("retReturnNumber")
private String retReturnNumber;  // ← Data field
```

2. **Passed in payload** (Line 152):
```java
payload.setRmaNumber(returnDetailsDocument.getRetReturnNumber());  // ← Data
```

3. **Validated as business value** (Line 153-154):
```java
if (payload.getRmaNumber().length() > 20) {
    throw new RmaNumberLengthException(...);  // ← Business validation
}
```

4. **NOT in authentication headers** (Lines 86-88):
```java
headers.set("Authorization", "Bearer " + token);  // ← TOKEN here
// RMA is in payload body, NOT headers
```

5. **Complete separation from auth** (Line 81):
```java
String token = fedexTokenHolder.getFedExAuthToken();  // ← Auth token
payload.setRmaNumber(returnDetailsDocument.getRetReturnNumber());  // ← Business ID
```

**Distinct Concepts:**
- **Authentication Token** = for API authentication (header)
- **RMA** = business return identifier (payload data)

**Conclusion:** ✅ RMA is 100% a business identifier with NO authentication role.

---

### **7.1.7 Can same RMA model be reused for DHL with just different carrier?**

**Finding:** ✅ **YES - RMA model is FULLY REUSABLE for DHL**

**Evidence:**

1. **RMA is carrier-agnostic in storage:**
```java
@JsonProperty("retReturnNumber")
private String retReturnNumber;  // ← Both carriers use this
```

2. **RMA value doesn't change** - Only payload placement changes:

**FedEx (Current):**
```json
{
  "rmaNumber": "RET-20260813-001"  // Root level
}
```

**DHL (Proposed):**
```json
{
  "customerReferences": [
    {
      "value": "RET-20260813-001",  // SAME value
      "typeCode": "RMA"
    }
  ]
}
```

3. **DHL Implementation (only change needed):**
```java
// DHL implementation
String rmaValue = doc.getRetReturnNumber();  // ← Same retrieval
CustomerReference rmaRef = new CustomerReference();
rmaRef.setValue(rmaValue);         // ← Same value
rmaRef.setTypeCode("RMA");         // ← Different location only
request.getCustomerReferences().add(rmaRef);
```

**Reusability Assessment:**

| Aspect | Reusable? | Why |
|---|---|---|
| **Value** | ✅ YES | Same `retReturnNumber` field |
| **Storage** | ✅ YES | Same MongoDB field |
| **Retrieval** | ✅ YES | Same `getRetReturnNumber()` |
| **Validation** | ✅ YES | Same business rules |
| **Response storage** | ✅ YES | Same `Rma` object |
| **Payload structure** | ❌ NO | Root vs array |

**Conclusion:** ✅ RMA model is 95% reusable. ONLY payload building logic differs.

---

### **7.1.8 Summary Table - RMA Codebase Findings**

| Question | Finding | Location |
|---|---|---|
| **Generated?** | NO - Input from request | API input |
| **Created by?** | Not created (stored only) | `ReturnDetailsDocument.java:119` |
| **MongoDB Storage** | `retReturnNumber` + `rma` object | Lines 67-68, 119-120 |
| **Passed to FedEx** | Via `FedexPayload.setRmaNumber()` | `FedexClient.java:152` |
| **With Tracking?** | Via `rma.shipments[].label` | `Rma.java` hierarchy |
| **Business ID?** | ✅ YES (not auth token) | Line 81-152 comparison |
| **Reusable for DHL?** | ✅ YES (95% reusable) | Both use `retReturnNumber` |

---

### **7.2 RMA Investigation Checklist**

- [x] Where is RMA currently generated? → NOT generated, received as input
- [x] Which class/method creates it? → No creation, stored in ReturnDetailsDocument
- [x] How stored in MongoDB? → `retReturnNumber` field + `rma` object
- [x] Passed to FedEx? → Via FedexPayload in createRmaS4()
- [x] Associated with tracking? → Via Rma.shipments[].label.trackingNumber
- [x] Business ID or token? → BUSINESS IDENTIFIER
- [x] Reusable for DHL? → YES - 95% reusable

---

## Label Analysis — FedEx vs DHL

**Date Added:** August 14, 2026
**Scope:** SHIPPING LABEL portion only, for the customer to HP/Germany-warehouse RETURN shipment.
**Method:** Direct codebase inspection (file/line evidence below) + DHL documentation information supplied for this analysis (no DHL website/API access was performed or claimed).

> **IMPORTANT NOTE ON SOURCES:** Everywhere this section describes DHL behavior, it is based on "DHL documentation information supplied for this analysis" (the sample payload/response fields given in the task prompt). No DHL developer portal or website was accessed. All FedEx / SRS behavior below is CONFIRMED FROM CODE with exact file/line references.

### 1. Current FedEx Label Flow — Narrative

1. `POST /v1/internal/initiate` -> `ReturnDetailsController` -> `ReturnDetailsService.sendReturn()` builds a `ReturnDetailsDocument` and calls `resolveInitReturnFlow(body)`.
2. `resolveInitReturnFlow()` picks the `InitReturnFlow` bean whose `getFlowType()` matches the LaunchDarkly flag `getShipmentProviderFlag()`. For US/FedEx this resolves to `InitReturnFedexFlow` (`getFlowType()` returns `"Fedex"`).
3. `InitReturnFedexFlow.initReturn()` calls `FedexClient.createRma()`, which POSTs to FedEx (`fedexConfig.getCreateRmaUrl()`) and deserializes the response directly into a `ReturnDetailsDocument` (same class used for the internal model — FedEx's response shape is mapped 1:1 onto our own document model).
4. FedEx's response contains `rma.shipments[0].label.labelURL` and `rma.shipments[0].label.trackingNumber`. Both land in `RmaShippmentLabel` (nested under `Rma` -> `RmaShipment` -> `RmaShippmentLabel`).
5. `InitReturnFedexFlow` copies the whole `rma` object onto the payload (`payload.setRma(...)`) and separately constructs `trackingUrl` by string-concatenating the FedEx tracking-number with a configured URL prefix (`fedex.trackingUrl` property). `labelURL` is copied along as part of `rma` but is never read/used by any other code path.
6. The full `ReturnDetailsDocument` (including the embedded `rma.shipments[0].label.labelURL` and `trackingNumber`) is persisted to MongoDB via `returnDetailsRepository.save(body)`.
7. Later, when FedEx RMA/waybill creation happens via the S4-triggered path (`ReturnDetailsService.handleProcessingStatus()` -> `FedexClient.createRmaS4()`), the tracking number is copied onto each `Item.trackingNumber` field by the static helper `ReturnDetailsService.setFedexTrackingNumber()`. `labelURL` is NOT copied to `Item` — there is no `Item.labelURL` field at all.
8. Outbound notifications about the return/shipment go to the Stratus Event Manager (EMS), not directly to "S4" as an HTTP client from SRS's perspective — `MessagingService.getEventAttributes()` builds an `EventAttributes` DTO containing `trackingNumber`, `shippingVendor` (=`shipmentProvider`), and `trackingUrl`. `EventAttributes` has NO `labelURL` field. This event is what ultimately reaches downstream consumers (S4/SAP is one of the consumers per the diagrams in this doc, eClaims, and other listeners (see `docs/Delinquent-Writeoff_Sequence-Diagram.puml`).
9. Inbound S4 status callbacks (`ReturnStatusS4`, XML->object bound at `POST /v1/return-details/status`) carry `shippingInfo[].trackingNumber` and `shippingInfo[].shippingVendor` but no label field at all — S4 sends tracking status back to SRS; it does not exchange label content with SRS in either direction.
10. **CORRECTION (added after re-verification, Aug 14 2026):** `labelURL` is NOT fully "dead" as originally stated below — `ReturnDetailsController.getReturnDetailsOrderNumber()` (`GET v1/internal/return-details/{returnOrderId}`, `ReturnDetailsController.java:95-111`) returns the **raw `ReturnDetailsDocument`** (`ResponseEntity<ReturnDetailsDocument>`) as its JSON response body. Since `rma` is a field on that document, `labelURL` (nested at `rma.shipments[0].label.labelURL`) IS serialized out to any caller of this endpoint (protected only by `AuthScopes.READ`). This is a real, existing exposure path — any internal API consumer (a portal, admin tool, or another microservice with a READ-scoped token) that calls this endpoint today receives `labelURL` in the response, even though nothing inside SRS itself reads it further. By contrast, the two other read endpoints that return `List<ReturnResponse>` (`v1/return-details/tenants/{tenantId}`, `v1/internal/return-details/subscriptions/{subscriptionId}`) use the `ReturnResponse` DTO (`api/dto/ReturnResponse.java`), which has NO `rma`/`labelURL` field — so `labelURL` only leaks out through the single-record `GET .../{returnOrderId}` endpoint, not the list endpoints.

**Conclusion (CONFIRMED FROM CODE, updated): `labelURL` is received from FedEx and stored in MongoDB as a field of the persisted document. It is never read, transformed, downloaded, or forwarded to EMS/S4 by any SRS business logic — but it IS exposed verbatim to any caller of `GET v1/internal/return-details/{returnOrderId}` as part of the full document response.** Only `trackingNumber` and `trackingUrl` are actively used/propagated by SRS's own logic; `labelURL` is passively readable via that one GET endpoint. This materially changes Open Question #1 below from "cannot be answered from this repository alone" to "SRS itself provides a code path that would let an external caller retrieve it — whether anything actually calls that endpoint for this purpose still needs team confirmation."

### 2. Exact Code Flow (class -> method -> file:line)

| Step | Class | Method | File | Line(s) |
|---|---|---|---|---|
| 1 | ReturnDetailsController | sendReturn() | src/main/java/com/hp/tropos/api/ReturnDetailsController.java | ~59-71 |
| 2 | ReturnDetailsService | sendReturn() | src/main/java/com/hp/tropos/service/ReturnDetailsService.java | 195-224 |
| 3 | ReturnDetailsService | resolveInitReturnFlow() | src/main/java/com/hp/tropos/service/ReturnDetailsService.java | 242-249 |
| 4 | InitReturnFedexFlow | initReturn() | src/main/java/com/hp/tropos/service/fedex/InitReturnFedexFlow.java | 33-62 |
| 5 | FedexClient | createRma() | src/main/java/com/hp/tropos/clients/fedex/FedexClient.java | 71-125 |
| 6 | Rma / RmaShipment / RmaShippmentLabel | data model, response deserialization target | src/main/java/com/hp/tropos/repository/model/{Rma,RmaShipment,RmaShippmentLabel}.java | full files (34, 40, 24 lines) |
| 7 | InitReturnFedexFlow | initReturn() — tracking URL construction | src/main/java/com/hp/tropos/service/fedex/InitReturnFedexFlow.java | line 51 |
| 8 | ReturnDetailsDocument | field rma, trackingUrl | src/main/java/com/hp/tropos/repository/model/ReturnDetailsDocument.java | 67-68 (rma), 116-117 (trackingUrl) |
| 9 | ReturnDetailsRepository (Spring Data) | save() (via MongoRepository) | called from ReturnDetailsService.sendReturn() | line 211 |
| 10 | ReturnDetailsService | handleProcessingStatus() | src/main/java/com/hp/tropos/service/ReturnDetailsService.java | 1392-1424 |
| 11 | ReturnDetailsService | setFedexTrackingNumber() (static) | src/main/java/com/hp/tropos/service/ReturnDetailsService.java | 1426-1439 |
| 12 | Item | field trackingNumber (no label field) | src/main/java/com/hp/tropos/repository/model/Item.java | line 74-75 |
| 13 | MessagingService | getEventAttributes() | src/main/java/com/hp/tropos/service/MessagingService.java | 56-102 (label never referenced; tracking at 66-71) |
| 14 | EventAttributes | DTO — no label field | src/main/java/com/hp/tropos/clients/stratus/eventsmanager/dto/EventAttributes.java | full file (27 lines) |
| 15 | MessagingService | publish() -> publishToEventManager() | src/main/java/com/hp/tropos/service/MessagingService.java | 133-192 |
| 16 | ReturnStatusS4 (inbound S4 callback model) | ShippingInfo inner class — no label field | src/main/java/com/hp/tropos/service/model/ReturnStatusS4.java | 109-118 |
| 17 | FedexConfig | trackingUrl property holder | src/main/java/com/hp/tropos/config/properties/FedexConfig.java | line 31 |
| 18 | application.properties | fedex.trackingUrl value | src/main/resources/application.properties | line 174 |
| 19 | ReturnDetailsRepository | Mongo query referencing rma.shipment.label (existence check only) | src/main/java/com/hp/tropos/repository/ReturnDetailsRepository.java | 92-108 |

### 3. FedEx Request/Response Mapping (exact)

**Response deserialization target:** FedEx's HTTP response body is deserialized straight into `ReturnDetailsDocument.class` (see `FedexClient.createRma()` line 99: `restTemplate.postForObject(fedexConfig.getCreateRmaUrl(), entity, ReturnDetailsDocument.class)`). There is no separate "FedexResponseMapper" class — SRS reuses its own persisted-entity class as the FedEx DTO. This is architecturally significant: FedEx's shape (`rma.shipments[].label.{labelURL,trackingNumber}`) is baked directly into the internal document model.

Exact JSON path (confirmed from `RmaShippmentLabel.java` `@JsonProperty` annotations and the sample payload committed to `etc/postman/Stratus Returns Service.postman_collection.json`):
```
rma.rmaNumber                              (String)
rma.shipments[0].label.id                  (String)
rma.shipments[0].label.labelURL            (String, URL) - @JsonProperty("labelURL"), RmaShippmentLabel.java:18-19
rma.shipments[0].label.trackingNumber      (String)      - @JsonProperty("trackingNumber"), RmaShippmentLabel.java:21-22
```
Real example from the Postman collection (representative of what FedEx actually returns in this integration):
```json
"label": {
    "id": "108988",
    "labelURL": "https://api-sandbox.supplychain.fedex.com/api/sandbox/v1/labels/108988",
    "trackingNumber": "794672887344"
}
```

### 4. Current Label Persistence (CONFIRMED FROM CODE)

- Model: `RmaShippmentLabel.labelURL` (String) — nested inside `RmaShipment.label` — nested inside `Rma.shipments[]` — nested inside `ReturnDetailsDocument.rma`.
- Persistence: the entire `ReturnDetailsDocument` (a `@Document("returnDetails")` Spring Data Mongo entity, `ReturnDetailsDocument.java:34`) is saved via `returnDetailsRepository.save(body)` (`ReturnDetailsService.java:211`). Because `rma` is a field on this document, `labelURL` is persisted to MongoDB as a plain string URL, nested under `returnDetails.rma.shipments[0].label.labelURL`.
- The field is not indexed (unlike `returnOrderId` at `ReturnDetailsDocument.java:76-78` which has `@Indexed(unique = true)`).
- No download/decoding logic exists anywhere in `src/main`. A repository-wide search for `Base64`, `InputStream`, `byte[]`, and `MultipartFile` in `src/main/java/com/hp/tropos/**` returned zero matches. SRS never fetches the FedEx label PDF; it only stores the URL string as-is.
- Persistence category (per Part 3 options): **A — stored directly as a URL.** SRS does NOT download and store binary/Base64 content (option B), and does not persist the label to a separate object/document store (option C).
- The only place `label` is referenced outside the model classes is a MongoDB existence-check query in `ReturnDetailsRepository.java:100-108`, which checks `{ 'rma.shipment.label': null }` as part of a retry-eligibility filter for the undelivered-fulfillment background job — this checks for label presence, not its content, and the query key itself (`rma.shipment.label`, singular "shipment") does not even match the actual persisted field path (`rma.shipments[]`, plural array) — this looks like a pre-existing minor inconsistency/bug in that query, unrelated to DHL but worth flagging.

### 5. Current S4/EMS Label Handling (CONFIRMED FROM CODE)

There is no single "S4 client" that SRS calls directly for outbound events; instead:

- Outbound: `MessagingService.getEventAttributes()` (`MessagingService.java:56-102`) builds `EventAttributes` with only these shipment-related fields: `trackingNumber` (line 67, sourced from `rma.shipments[0].label.trackingNumber`), `shippingVendor` (line 70, sourced from `returnData.getShipmentProvider()`), `trackingUrl` (line 71, sourced from `returnData.getTrackingUrl()`). `labelURL`/label content is never added to this DTO. This event is published to the Stratus Event Manager (EMS) via `StratusEventManagerClient` (`MessagingService.java:190-191`), which is the mechanism by which downstream systems (including, per the sequence diagrams in this repo, S4/SAP) learn about return/shipment status.
- Inbound: `ReturnStatusS4` (`src/main/java/com/hp/tropos/service/model/ReturnStatusS4.java`) is the XML/JSON model bound from S4's status callback (`POST /v1/return-details/status`, handled by `ReturnDetailsService.handleS4ReturnStatus()`). Its `ShippingInfo` inner class (lines 109-118) has only `shippingVendor`, `trackingNumber`, `partSerialNumber`, `partLineNum` — no label field. S4 sends tracking/status info to SRS; it does not send or receive label documents.

**Direct answers (Part 4):**
1. Does S4/EMS payload contain labelURL? **NO** (confirmed — `EventAttributes.java` has no such field).
2. Does the current contract support inline Base64 document content? **NO** — no such field exists, and no code anywhere serializes/attaches binary content to an event.
3. Is label information sent to S4/EMS at all? **NO.**
4. Which downstream system consumes tracking info? Per code + diagrams: Stratus Event Manager (EMS) -> downstream consumers including S4/SAP, eClaims, and other listeners (see `docs/Delinquent-Writeoff_Sequence-Diagram.puml`).
5. Does any downstream service assume label is a URL? Cannot be confirmed from this codebase — SRS itself never forwards the label at all, so there is no code-level assumption to point to. (NEEDS TEAM CONFIRMATION with the S4/EMS/eClaims teams.)
6. FedEx-specific assumptions? Yes — `InitReturnFedexFlow.java:51` hardcodes FedEx's tracking-URL-prefix config (`fedexConfig.getTrackingUrl()`) and assumes `rma.getShipments().get(0)` always exists with a non-null `label` (no null-check).
7. Would sending DHL Base64 content break the current contract? Not applicable to the current contract, because the current contract never sends label content in the first place. The only real compatibility question is around `trackingNumber` and `trackingUrl`, both of which are plain strings and DHL's equivalents (`shipmentTrackingNumber` and a DHL tracking URL) are also plain strings, so those two fields are structurally compatible.

### 6. DHL Label Behavior (per documentation information supplied for this analysis — NOT independently verified against DHL's live API/docs)

DHL documentation information supplied for this analysis states:
- DHL's MyDHL Shipment API response contains `shipmentTrackingNumber` (top-level) and a `documents[]` array.
- Each entry in `documents[]` can carry the label/waybill content directly in `documents[].content`.
- `content` "can be Base64 encoded."
- The request can specify `outputImageProperties.encodingFormat = "pdf"` to influence the format of the returned document.
- There is no dedicated "Create Return Shipment" endpoint — a return is just a shipment with shipper/receiver roles reversed.
- DHL documentation information supplied for this analysis also gives a sample DHL tracking URL: `https://www.dhl.com/en/express/tracking.html?AWB=<number>` (flagged explicitly in the prompt as needing validation against the real DHL contract before production use).

### 7. FedEx to DHL Response Mapping (Label-Specific)

| Field | FedEx (CONFIRMED FROM CODE) | DHL (per supplied documentation) | Mapping Impact |
|---|---|---|---|
| Tracking number | rma.shipments[0].label.trackingNumber -> RmaShippmentLabel.trackingNumber | shipmentTrackingNumber (top-level) | Straightforward value mapping; DHL is NOT nested the same way — requires new extraction logic, not a drop-in replacement of RmaShippmentLabel. |
| Label/document | rma.shipments[0].label.labelURL (a live URL string) -> RmaShippmentLabel.labelURL | documents[typeCode="label"].content (Base64-encoded string, inline in the response body) | Fundamentally different delivery model: URL-by-reference vs. content-by-value. This is the single biggest architectural gap. |
| Tracking URL | Constructed by SRS: fedexConfig.getTrackingUrl() + trackingNumber (InitReturnFedexFlow.java:51), using fedex.trackingUrl property (application.properties:174) | Not returned by DHL's shipment response per the supplied documentation; a sample public tracking URL format was supplied (https://www.dhl.com/en/express/tracking.html?AWB=<number>) but this is documentation/sample info only, needs validation | Same SRS-side construction pattern (config-prefix + tracking number) can likely be reused for DHL, but the actual URL template must be confirmed with DHL/team before production. |
| Output format control | Not present in FedEx flow (FedEx's format is whatever labelURL resolves to; SRS never inspects it) | Request has outputImageProperties.encodingFormat = "pdf" — DHL requires the requester to ask for a format | New request-building responsibility that has no FedEx analogue in the current code. |

### 8. Label Format Comparison

- FedEx (CONFIRMED FROM CODE): SRS has zero opinion or logic about label format. `labelURL` is stored and never opened, inspected, downloaded, or converted anywhere in `src/main`. There is no MIME-type handling, no `.pdf` filename check, no ZPL/PNG-specific logic, and no printer-integration code found in this repository. SRS today is a pure pass-through of a URL string it never dereferences.
- DHL (per supplied documentation): `outputImageProperties.encodingFormat = "pdf"` implies DHL can be explicitly asked to return PDF; content arrives as Base64 inline data rather than a fetchable link.
- Direct answers (Part 7):
  1. Is PDF already "supported" by SRS? Trivially yes in the sense that SRS never touches the byte content either way — there is no code path that would reject or mishandle a PDF; but there is also no code path that consumes it.
  2. Is conversion required? Not for SRS's own logic (it doesn't process the bytes today) — but if downstream/EMS/S4 or any UI expects a clickable URL, then yes, some conversion/storage step would be needed to turn DHL's Base64 payload into a URL, if URL-based consumption is required downstream.
  3. Does downstream S4/EMS support PDF? Cannot be confirmed from code — the current EMS EventAttributes DTO has no field for label content or PDF at all. NEEDS TEAM CONFIRMATION.
  4. Does any warehouse/printing system require ZPL? No evidence found in this repository either way. NEEDS TEAM CONFIRMATION.
  5. Is the current FedEx label consumed by a browser or printer? Cannot be determined from this codebase — SRS never opens the URL, so any browser/printer consumption must happen entirely outside SRS. NEEDS TEAM CONFIRMATION on who actually consumes rma.shipments[0].label.labelURL from the database.
  6. Is there code that assumes a URL ending in .pdf? No — no filename/extension-based logic was found anywhere in src/main.
  7. Is there code that downloads the URL before processing? No — confirmed no HTTP GET, no InputStream, no file I/O tied to labelURL anywhere in src/main.

### 9. Base64 vs URL Analysis

- FedEx model = pull (SRS/consumer fetches bytes later via labelURL, at their leisure, subject to FedEx's URL validity/auth/expiry — none of which SRS has any code to manage today).
- DHL model (per supplied documentation) = push (bytes are embedded directly in the synchronous API response as Base64 in documents[].content).
- SRS's current persistence model (a MongoDB document with a labelURL string field) can trivially store a Base64 string too — MongoDB has no schema constraint here, so there is no database-schema blocker to storing DHL's Base64 content as-is if the team decides to keep it inline. However, storing large Base64 PDF blobs directly inside the main returnDetails document (which is read/written frequently across the whole return lifecycle) is a design concern (document bloat, read/write performance, MongoDB 16MB document size limit) — this is an architectural judgment call, not a hard code blocker.
- No existing SRS code reads, decodes, or re-encodes Base64 for labels. Any Base64 handling for DHL would be entirely new code (not a modification of existing decode logic, since none exists).

### 10. Tracking Number Mapping

- FedEx: rma.shipments[0].label.trackingNumber -> copied in two places:
  - InitReturnFedexFlow.java:51 (embedded implicitly as part of payload.setRma(...) a few lines earlier, and explicitly used to build trackingUrl)
  - ReturnDetailsService.setFedexTrackingNumber() (ReturnDetailsService.java:1426-1439) — copies rmaShipment.getLabel().getTrackingNumber() onto each matching Item.trackingNumber (by SKU match).
- DHL (per supplied documentation): shipmentTrackingNumber is a top-level field on the shipment response, not nested under a label object.
- Can the same internal model store both? The final storage target (Item.trackingNumber, a plain String field — Item.java:74-75) is carrier-agnostic and can store a DHL tracking number with no schema change. However, the intermediate extraction path (fedexResponse.getRma().getShipments().get(0).getLabel().getTrackingNumber()) is FedEx-response-shape-specific and hardcoded to the Rma/RmaShipment/RmaShippmentLabel class hierarchy. DHL's flat shipmentTrackingNumber cannot be read through this same object graph without new extraction code (this is a code change, not a data-model blocker).
- Are tracking number and label always generated together? In the FedEx model, yes structurally — both live under the same label object in the same response, generated in the same API call. Per DHL's supplied documentation, they are not necessarily co-located: shipmentTrackingNumber is top-level while the label lives inside documents[] — still produced by the same shipment-creation call, but structurally decoupled in the response shape.

### 11. Tracking URL Analysis

- CONFIRMED FROM CODE: The FedEx tracking URL is constructed by SRS, not returned by FedEx. Evidence:
  - FedexConfig.java:31 — private String trackingUrl; (a Spring @ConfigurationProperties(prefix = "fedex") field)
  - application.properties:174 — fedex.trackingUrl=https://www.fedex.com/fedextrack/?tracknumbers=
  - InitReturnFedexFlow.java:51 — payload.setTrackingUrl(fedexClient.getFedexConfig().getTrackingUrl() + payload.getRma().getShipments().get(0).getLabel().getTrackingNumber()); — simple string concatenation of the configured prefix + the FedEx tracking number.
- It is not hardcoded in Java (it's externalized to application.properties), not generated by S4, and not generated by a separate utility class — it's inline string concatenation inside InitReturnFedexFlow.
- For DHL: the exact same pattern (a configured URL-prefix property + tracking number concatenation) could structurally be reused for a new InitReturnDhlFlow equivalent, using a new dhl.trackingUrl (or similar) property. The DHL URL format supplied in this prompt (https://www.dhl.com/en/express/tracking.html?AWB=<number>) is documentation/sample information only and must be validated against DHL's actual contract before being hardcoded into a config value.

### 12. S4/EMS Compatibility Summary

| Question | Answer | Basis |
|---|---|---|
| Does current contract carry labelURL/label content today? | No | EventAttributes.java has no such field (CONFIRMED FROM CODE) |
| Would adding DHL's Base64 label break the existing contract? | No, because there is nothing to break — no field exists for it today, so DHL doesn't corrupt anything existing; it's a net-new capability, not a modification | CONFIRMED FROM CODE |
| Does current contract carry trackingNumber/trackingUrl/shippingVendor? | Yes (EventAttributes.trackingNumber, .trackingUrl, .shippingVendor) | CONFIRMED FROM CODE |
| Can DHL's shipmentTrackingNumber flow through the same EventAttributes fields? | Yes, structurally — both are plain strings | CONFIRMED FROM CODE (field types), extraction logic still needs to change (Section 10) |
| Does downstream (S4/eClaims/etc.) need the label at all today? | Unknown from this codebase — since SRS never forwards it, the question of downstream necessity cannot be answered from code alone | NEEDS TEAM CONFIRMATION |

### 13. Gap Analysis Table

| Area | Existing FedEx | DHL (per supplied docs) | Code Location | Gap | Recommended Approach |
|---|---|---|---|---|---|
| Tracking number | rma.shipments[0].label.trackingNumber | shipmentTrackingNumber (top-level) | RmaShippmentLabel.java:21-22; ReturnDetailsService.java:1426-1439 | New extraction path required (different nesting) | Add DHL-specific extraction logic that writes into the same Item.trackingNumber / EventAttributes.trackingNumber targets |
| Label URL | labelURL (live fetchable URL) | Not provided as URL by DHL (per supplied docs) | RmaShippmentLabel.java:18-19 | DHL has no direct equivalent field | Do not attempt to map 1:1; treat as a different concept (see "Label document" row) |
| Label document | N/A — SRS never processes label bytes | documents[] array, possibly multiple documents/types | New concept | Entire capability doesn't exist in SRS today | New model/DTO needed only if SRS decides to actively handle DHL documents (not required by current contract) |
| Label content | N/A | documents[].content | N/A | No existing field to store this in Rma/RmaShippmentLabel | If needed, add a new field (e.g., to a DHL-specific model), NOT to RmaShippmentLabel (which is FedEx-shaped) |
| Base64 | Never used | Content "can be Base64 encoded" per supplied docs | N/A (no Base64 code found in src/main) | No decode/encode utility exists in SRS | New code required if Base64 must be processed (decode, store, or re-expose) |
| PDF | Never inspected by SRS | outputImageProperties.encodingFormat="pdf" (requestable) | N/A | No format-negotiation logic exists today | If format matters downstream, add explicit format request/validation for DHL calls |
| ZPL | No evidence of use anywhere in repo | Not mentioned in supplied DHL info | N/A | Unknown requirement | NEEDS TEAM CONFIRMATION (warehouse/printer requirements) |
| Document type | FedEx: implicitly PDF (SRS agnostic) | DHL: explicit typeCode per document (e.g. "label") per supplied docs | N/A | SRS has no typeCode concept today | If multiple DHL document types are returned, add a typeCode filter when selecting which document is "the label" |
| Label storage | Stored as URL string, nested in rma.shipments[0].label.labelURL | Not defined by current SRS model | ReturnDetailsDocument.java:67-68 (rma field) | RmaShippmentLabel is FedEx-specific class | Do not force DHL data into RmaShippmentLabel; needs its own model if persisted |
| Label persistence | Full label object persisted as part of ReturnDetailsDocument in MongoDB | N/A today | ReturnDetailsRepository.save(), ReturnDetailsService.java:211 | No schema blocker (MongoDB is schemaless per-field), but document-size/perf is a consideration for large Base64 blobs | Prefer NOT persisting large Base64 content inline in the main document; if persistence is required, evaluate a separate collection/object store |
| S4/EMS payload | EventAttributes: trackingNumber, shippingVendor, trackingUrl only | Same fields structurally compatible; no label field exists in contract today | EventAttributes.java; MessagingService.java:56-102 | No gap for tracking fields; label was never in scope of this contract | No contract change needed unless business requires label delivery to downstream — TEAM DECISION |
| Downstream consumer | EMS -> (per diagrams) S4/SAP, eClaims, etc. | Same downstream topology assumed (not FedEx/DHL-specific) | MessagingService.publish() | Consumer identity/expectations for label are unknown | NEEDS TEAM CONFIRMATION with S4/EMS/eClaims owners |
| Tracking URL | SRS-constructed: config prefix + tracking number (InitReturnFedexFlow.java:51) | Documentation/sample URL given, not confirmed | FedexConfig.java:31, application.properties:174 | Need a validated DHL tracking URL template | Reuse the same config-prefix + concatenation pattern once DHL URL format is confirmed by DHL/team |
| Carrier | shipmentProvider field, driven by LaunchDarkly flag (getShipmentProviderFlag()) | Would be a new flag value (e.g. "DHL") | ReturnDetailsService.java:195; InitReturnFedexFlow.java:66 (getFlowType() returns "Fedex") | Need a new InitReturnFlow implementation with getFlowType() returning e.g. "DHL" | Follow the existing InitReturnFlow strategy-pattern precedent (analysis only — do not implement per task scope) |
| RMA | rmaNumber root-level field in FedEx payload; retReturnNumber internal field | customerReferences[typeCode="RMA"].value per earlier section of this document | FedexClient.java:152; ReturnDetailsDocument.java:119-120 | Different payload placement only; value itself is reusable | Carry retReturnNumber into DHL's customerReferences array with typeCode="RMA" |
| Error handling | FedexUnauthorizedException, FedexPreconditionFailedException, FedexClientException, RmaNumberLengthException — all FedEx-specific | Not defined by supplied DHL docs | src/main/java/com/hp/tropos/clients/fedex/*.java | No DHL-specific exception types exist | NEEDS TEAM CONFIRMATION on DHL error contract; new exception types would likely be needed, but this is implementation, out of scope for this analysis |

### 14. Recommended Integration Approach (Analysis Only — Not Implemented)

Evaluating the three options  against the actual codebase evidence above:

- Option A (SRS stores Base64/binary, existing SRS consumers read the document): Does not fit — there are no existing SRS consumers of labelURL content to begin with (Section 4/8 confirmed zero download/decode code). There is nothing for a Base64 field to be "read by" inside SRS today.
- Option B (SRS decodes/stores PDF, generates an accessible URL, existing S4/downstream contract continues to use URL): This assumes downstream currently consumes a label URL from S4/EMS — but Section 5/12 confirmed the current EMS/S4 contract (EventAttributes) has no label field at all, so there is no existing "URL contract" to preserve for the label specifically (only trackingUrl exists, which is a different concept — the shipment tracking page, not the label document). If some other, out-of-repository consumer reads labelURL directly from MongoDB (unconfirmed — see open question), Option B would be the only way to keep that consumer working unmodified.
- Option C (SRS sends Base64 directly downstream): Would require adding a new field to EventAttributes and possibly to whatever schema S4/EMS itself expects — a downstream contract change, not just an SRS-side change.

Given the actual codebase (labelURL is stored-but-unused, and never forwarded downstream today): the lowest-risk, most contract-compatible starting point is to treat trackingNumber and trackingUrl as the only fields that must reliably map from DHL -> SRS -> EMS (since these are the only fields with real, active consumers in the code). The labelURL/label-document question is not actually a live production dependency in the current codebase — before deciding between Option A/B/C, the team must first confirm (see Open Questions) whether any system outside this repository reads rma.shipments[0].label.labelURL directly from the returnDetails MongoDB collection. If no such consumer exists, the safest approach is to not persist DHL's Base64 label inline in the main returnDetails document at all, and instead treat label handling as a separate, to-be-designed capability — decoupled from the tracking-number/tracking-URL migration, which is comparatively low-risk and can be reused with only new extraction-mapping code (Section 10/11).

### 15. Open Questions for Team

1. Does any system outside SRS  actually call `GET v1/internal/return-details/{returnOrderId}` and read/use `rma.shipments[0].label.labelURL` from that response, or query the returnDetails MongoDB collection directly? SRS itself is now CONFIRMED to expose this field via that endpoint (see Section 1, item 10, and Section 17 below); what's still unconfirmed is whether any real consumer relies on it.
2. Does S4 or EMS (or any listener behind EMS) require the actual label document/PDF at all, or only tracking number + tracking URL (which is all the current contract carries)?
3. If the label is required downstream, should DHL's Base64 content be (a) decoded and stored as a URL-accessible object (Option B), (b) passed through as Base64 (Option C), or (c) not persisted by SRS at all and left to a separate label-service?
4. Is DHL's sample tracking URL (https://www.dhl.com/en/express/tracking.html?AWB=<number>) the correct, stable, production customer-facing tracking URL? (Explicitly flagged in the supplied documentation as needing validation.)
5. Can documents[] contain more than one document (e.g., commercial invoice + label)? If so, what typeCode reliably identifies "the label" vs. other documents? (Not answerable from the supplied documentation snippet alone.)
6. Does the DHL shipment-creation call always return a label in the same synchronous response, or can label generation be asynchronous/delayed? (Not stated in supplied documentation.)
7. Are there MongoDB document-size concerns if large Base64 PDFs were stored inline in returnDetails (16MB BSON document limit)? Does the team have an existing object-storage pattern already used elsewhere in this codebase (e.g., see reconextReportsBucketName/reconextReportsFolder in FedexConfig.java:35-37) that could be reused for DHL labels?
8. Should a new ReturnDetailsDocument/EMS contract field be added for label content/URL, and if so, who owns approval of that downstream schema change (S4 team, EMS team, or both)?
9. Is ReturnDetailsRepository.java:100-101's query key 'rma.shipment.label' (singular "shipment") versus the actual persisted field path rma.shipments[] (plural array) an existing bug, or intentional? (Flagged as an aside finding, not DHL-related, but relevant to any future label-based Mongo query design for DHL.)

### 16a. Verification Pass — Three Specific Checks Re-Confirmed Against Code

This sub-section documents a targeted re-verification of three specific claims made above, done by re-inspecting the actual source files (not re-deriving from scratch).

**Check 1 — Where does the FedEx `labelURL` actually go after SRS receives it?**
- Confirmed path: FedEx response → `ReturnDetailsDocument.rma.shipments[0].label.labelURL` → persisted to MongoDB via `returnDetailsRepository.save(body)` → **also returned verbatim in the JSON body of `GET v1/internal/return-details/{returnOrderId}`** (`ReturnDetailsController.java:95-111`, `ResponseEntity<ReturnDetailsDocument>`), because that endpoint returns the entire persisted document, not a filtered DTO.
- Checked and ruled out as exposure paths: `GET v1/return-details/tenants/{tenantId}` and `GET v1/internal/return-details/subscriptions/{subscriptionId}` both return `List<ReturnResponse>` (`api/dto/ReturnResponse.java`), and `ReturnResponse` has no `rma`/`labelURL` field (only `trackingNumber`, `trackingUrl`, `shippingVendor`, etc.) — confirmed by reading the full 93-line DTO. So `labelURL` does NOT leak through the tenant/subscription list endpoints, only through the single-record internal GET-by-returnOrderId endpoint.
- No other reader of `rma`/`label` was found beyond the one Mongo existence-check query already noted in Section 4 (`ReturnDetailsRepository.java:100-108`, presence-only, not content).
- **Revised answer:** `labelURL` is not fully inert — it is persisted, and then re-exposed as-is (still just a URL string) to any caller of that one internal GET endpoint. There is no evidence in this repo of what (if anything) calls that endpoint specifically to read the label, so "is it actually used by a consumer" remains a team question — but "is there a code path that could expose it" is now answered YES, not "no such path exists."

**Check 2 — Does S4 expect a URL, or can it accept document content?**
- Re-searched the whole repo for any S4-side API contract (OpenAPI/Swagger/WSDL/JSON-schema) that isn't just SRS's own outbound DTO. None exists in this repository — the only two S4-facing contracts in-repo are (a) SRS's own outbound `EventAttributes` DTO (no label field, string-typed `trackingNumber`/`trackingUrl`) and (b) SRS's own inbound `ReturnStatusS4`/`ShippingInfo` model for S4's status callback (also no label field, string-typed fields only).
- Also checked a structurally similar, unrelated flow for corroboration: the Gekko→Solace `REPLACEMENT_SHIPMENT_TRACK_N_TRACE` event (`src/test/resources/SolaceSampleEvent/ReplacementShipmentTrackNTraceEvent.json`, consumed — not sent to S4 — by `GekkoTransformServiceUnitTest`/Jolt specs) carries `carrierTrackingURL` as a plain string, reinforcing that everywhere in this codebase carrier tracking links are modeled as URL strings, never as binary/Base64 content. This is circumstantial (a different, inbound, replacement-flow pipeline), not direct proof of what S4 itself can accept for a *label document* — it does not resolve the open question.
- **Answer unchanged, still NEEDS TEAM CONFIRMATION with the S4 team:** nothing in this repository defines what S4's own receiving system supports for label delivery (URL vs. inline document). SRS's own contract simply never sends label data either way, so there's no in-repo evidence either confirming or ruling out S4 accepting inline document content.

**Check 3 — Can the existing SRS model store DHL's Base64 PDF?**
- Re-checked `RmaShippmentLabel.java` (the class holding `labelURL`) for any length/format constraints: no `@Size`, `@Pattern`, `@Length`, or `maxLength` annotations found anywhere in that class or elsewhere in `repository/model` for this field (confirmed via repo-wide search — zero matches).
- `labelURL` is a plain `String` with only a `@JsonProperty("labelURL")` annotation — no validation constrains its content or length.
- MongoDB itself imposes no per-field schema/length constraint (schemaless per-field); the only real limit is the **16MB BSON document size cap** on the whole `returnDetails` document, which matters because `rma` lives inside that same document, not a separate collection.
- **Answer unchanged and reconfirmed:** Yes, the current model can technically store a Base64 string in `labelURL` (or a new sibling field) with zero code-level or database-schema changes required. The only real concern is architectural (document bloat / read-write performance / proximity to the 16MB limit for a frequently-read-and-written document), not a hard blocker.

### 16. Exact Code References (Consolidated)

- File: src/main/java/com/hp/tropos/repository/model/RmaShippmentLabel.java — Full class. Fields: id, labelURL (@JsonProperty("labelURL"), line 18-19), trackingNumber (line 21-22).
- File: src/main/java/com/hp/tropos/repository/model/RmaShipment.java — Field label of type RmaShippmentLabel (line 35-36).
- File: src/main/java/com/hp/tropos/repository/model/Rma.java — Field shipments (ArrayList<RmaShipment>, line 31-32).
- File: src/main/java/com/hp/tropos/repository/model/ReturnDetailsDocument.java — Field rma (line 67-68, type Rma); field trackingUrl (line 116-117, type String).
- File: src/main/java/com/hp/tropos/clients/fedex/FedexClient.java — Method createRma(ReturnDetailsDocument, AttributionFlag) (line 71-125): deserializes FedEx response into ReturnDetailsDocument (line 99). Method createRmaS4(...) (line 128-192). Method fedexCall(FedexPayload) (line 194-236). Method createAttributionRes(...) (line 289-314) — builds a synthetic Rma/RmaShipment/RmaShippmentLabel for test/prod-testing mode, setting only trackingNumber (line 307), never labelURL.
- File: src/main/java/com/hp/tropos/service/fedex/InitReturnFedexFlow.java — Method initReturn(ReturnDetailsDocument) (line 33-62). Line 47: payload.setRma(returnDetailsDocument.getRma()) (this is the only place labelURL gets copied — implicitly, as part of the whole rma object). Line 51: tracking URL construction reading .getLabel().getTrackingNumber() (NOT .getLabelURL()).
- File: src/main/java/com/hp/tropos/service/ReturnDetailsService.java — Method sendReturn() (line ~146-224). Method resolveInitReturnFlow() (line 242-249). Method handleProcessingStatus() (line 1392-1424). Static method setFedexTrackingNumber(ReturnDetailsDocument, ReturnDetailsDocument) (line 1426-1439) — reads rmaShipment.getLabel().getTrackingNumber() (line 1433), never reads labelURL.
- File: src/main/java/com/hp/tropos/repository/model/Item.java — Field trackingNumber (line 74-75). No labelURL field exists on Item.
- File: src/main/java/com/hp/tropos/service/MessagingService.java — Method getEventAttributes(ReturnDetailsDocument) (line 56-102). Line 66-68: reads returnData.getRma().getShipments().get(0).getLabel().getTrackingNumber() into attributes.setTrackingNumber(...). Line 70: attributes.setShippingVendor(returnData.getShipmentProvider()). Line 71: attributes.setTrackingUrl(returnData.getTrackingUrl()). labelURL is never referenced in this method.
- File: src/main/java/com/hp/tropos/clients/stratus/eventsmanager/dto/EventAttributes.java — Full class (27 lines). Fields: subscriptionId, fulfillmentOrderId, returnOrderId, status, trackingNumber, shippingVendor, trackingUrl, invoiceNumber, invoiceCurrency, invoiceTotal, parts. No label-related field.
- File: src/main/java/com/hp/tropos/service/model/ReturnStatusS4.java — Inner class ShippingInfo (line 109-118): fields shippingVendor, trackingNumber, partSerialNumber, partLineNum. No label-related field.
- File: src/main/java/com/hp/tropos/config/properties/FedexConfig.java — Field trackingUrl (line 31), createRmaUrl (line 33).
- File: src/main/resources/application.properties — Line 174: fedex.trackingUrl=https://www.fedex.com/fedextrack/?tracknumbers=.
- File: src/main/java/com/hp/tropos/repository/ReturnDetailsRepository.java — Method findAllDocumentsByUndeliveredNull(Date, Pageable) (line 92-108): Mongo @Query referencing 'rma.shipment.label': null (line 101) as part of a retry-eligibility filter.
- File: etc/postman/Stratus Returns Service.postman_collection.json — Contains a full real-shape example FedEx response payload showing label.id, label.labelURL, label.trackingNumber together.

---

## FINAL OUTPUT — Label-Specific Analysis Summary

### A. Executive Summary

SRS's current FedEx integration receives a label as a URL (rma.shipments[0].label.labelURL) nested inside the same response object it uses as its internal persisted document (ReturnDetailsDocument). This URL is saved to MongoDB as part of the whole document but is never read, downloaded, or forwarded by any other code in the repository — only the sibling field trackingNumber (and the SRS-constructed trackingUrl) are actively propagated to Item.trackingNumber and to the downstream Event Manager (EventAttributes). DHL, per the documentation information supplied for this analysis, does not return a label URL at all — it returns shipmentTrackingNumber (top-level) plus a documents[] array whose content is (potentially Base64-encoded) inline document data, with format requested via outputImageProperties.encodingFormat. This is a fundamentally different delivery model (push/inline vs. pull/URL) for the label specifically, while the tracking-number and tracking-URL concepts are structurally compatible and low-risk to migrate.

### B. Existing FedEx Code Flow

ReturnDetailsController.sendReturn() -> ReturnDetailsService.sendReturn() (ReturnDetailsService.java:195-224) -> resolveInitReturnFlow() (line 242-249) -> InitReturnFedexFlow.initReturn() (InitReturnFedexFlow.java:33-62) -> FedexClient.createRma() (FedexClient.java:71-125), response deserialized directly into ReturnDetailsDocument -> label lands in ReturnDetailsDocument.rma.shipments[0].label (RmaShippmentLabel, fields labelURL + trackingNumber) -> InitReturnFedexFlow.java:51 builds trackingUrl from trackingNumber only -> returnDetailsRepository.save(body) (ReturnDetailsService.java:211) persists the whole document including labelURL -> later, ReturnDetailsService.setFedexTrackingNumber() (line 1426-1439) copies trackingNumber (not labelURL) onto Item.trackingNumber -> MessagingService.getEventAttributes() (line 56-102) copies trackingNumber, shippingVendor, trackingUrl (not labelURL) into EventAttributes -> published to EMS (MessagingService.publish(), line 133-192) -> downstream consumers (S4/SAP, eClaims, etc., per repo diagrams).

### C. DHL Label Model (per supplied documentation only)

Shipment creation response -> shipmentTrackingNumber (top-level string) + documents[] (array of document objects) -> each document may have content (possibly Base64) and a typeCode. Format of content can be influenced via request field outputImageProperties.encodingFormat (e.g., "pdf").

### D. FedEx vs DHL Label Mapping

See the consolidated table in "13. Gap Analysis Table" above — key rows: Tracking number (straightforward remap, different nesting), Label URL vs Label document/content (no direct equivalent — different delivery model), Tracking URL (SRS-constructed pattern reusable, but DHL URL template unconfirmed).

### E. S4/EMS Impact

The current S4/EMS contract (EventAttributes.java) does not carry label information today (confirmed — only trackingNumber, shippingVendor, trackingUrl). Therefore DHL's Base64 label cannot break an existing contract that doesn't include it, but delivering DHL's label to S4/EMS would require a new field and a downstream schema change, which is outside SRS's unilateral control. The tracking-number/tracking-URL fields are structurally compatible with DHL's flat response shape, pending new extraction-mapping code.

### F. Code Changes Potentially Required (NOT implemented — identification only)

- A DHL-specific response/label extraction path (distinct from the FedEx-shaped Rma/RmaShipment/RmaShippmentLabel classes), since DHL's shipmentTrackingNumber is top-level and documents[] has a different shape entirely.
- A decision-driven addition to EventAttributes (and the corresponding S4/EMS downstream schema) only if the team confirms label content/URL must be forwarded downstream (Open Question #2).
- A DHL tracking-URL configuration value (parallel to fedex.trackingUrl), once the real DHL tracking URL template is confirmed (Open Question #4).
- Possibly a new persistence strategy (e.g., object storage) if large Base64 label content should not live inline in the main returnDetails MongoDB document (Open Question #7).
- A new InitReturnFlow implementation (e.g. InitReturnDhlFlow) following the existing strategy-pattern precedent, for orchestration — but this is broader than label handling alone and is noted here only for completeness.

### G. Open Questions for Team

See full numbered list in Section 15 above. Most critical for label planning specifically: (1) whether any external consumer reads labelURL directly from MongoDB today, (2) whether S4/EMS actually needs the label content at all, and (5) how to reliably identify "the label" among potentially multiple documents[] entries from DHL.

### H. Final Recommendation

Based strictly on the actual codebase: do not assume label-URL parity is required. Because labelURL is confirmed to be inert (stored but unused/unforwarded) in the current system, the team should first validate Open Question #1 (any hidden external consumer of labelURL from MongoDB) before committing to Option A, B, or C. If no external consumer exists, the lowest-risk path is to (i) migrate tracking-number and tracking-URL mapping first (low risk, high code-reuse, per Sections 10-11), and (ii) treat DHL label/document handling as a separate, explicitly-scoped follow-up decision — not something to force into the existing FedEx-shaped RmaShippmentLabel model, and not something to persist inline in the main returnDetails document without a size/performance review, given DHL's content is (per supplied documentation) inline Base64 rather than a lightweight URL reference like FedEx's.

---

# 8. FINAL AUDIT PASS — Additional Mandatory Analysis Areas 

This section was added during a FINAL, CODE-VERIFIED audit of this document. Everything above this point (Sections 1-7 and the Label Analysis/FINAL OUTPUT block) was reviewed and preserved unchanged — it remains accurate and is not duplicated here. This section fills the specific gaps identified during the audit: Carrier Selection Architecture, Rating, Germany Label Free/QR, Pickup/Drop-off/Service Point, Customs, Error Handling/Retry/Idempotency, Configuration/Secrets, Observability, Backward Compatibility, and Testing Impact — none of which had a dedicated section in the document prior to this pass (confirmed by a repository-wide text search for "Label Free", "Rating", "Pickup", "Customs", "Idempotency", "Observability", "Backward Compat", "Testing Impact", "InitReturnDhlFlow", "Service Point", and "QR", all returning zero prior matches in this file).


```

- `InitReturnFlow` (`src/main/java/com/hp/tropos/service/InitReturnFlow.java`) is a plain interface: `initReturn(ReturnDetailsDocument)` + `getFlowType()`.
- Spring auto-collects **all** beans implementing `InitReturnFlow` into the injected `List<InitReturnFlow> initReturnFlows` — this is a textbook strategy pattern, already multi-bean-capable with zero changes needed to the loop itself.
- `InitReturnFedexFlow` (`src/main/java/com/hp/tropos/service/fedex/InitReturnFedexFlow.java:65-66`) hardcodes `getFlowType() → "Fedex"` — this is the ONLY place the literal string `"Fedex"` is used as a selector; everywhere else `shipmentProvider`/`shippingVendor` is a passthrough plain `String`, not an enum.
- `InitReturnNoFlow` (`src/main/java/com/hp/tropos/service/InitReturnNoFlow.java`) also exists as a no-op fallback implementation of the same interface — confirming this is a genuine, already-proven multi-implementation strategy pattern in production use today, not a theoretical one.

**Safe Germany design (PROPOSED DESIGN, directly following the existing precedent):**

```text
InitReturnFlow (existing interface, unchanged)
  ├── InitReturnFedexFlow   (existing, UNCHANGED — getFlowType() → "Fedex")
  ├── InitReturnNoFlow      (existing, UNCHANGED — no-op fallback)
  └── InitReturnDhlFlow     (NEW — getFlowType() → "DHL")
```

Adding `InitReturnDhlFlow` requires **zero modification** to `resolveInitReturnFlow()`, `InitReturnFedexFlow`, or `InitReturnNoFlow` — the loop already iterates over an injected collection and matches by string equality against the LaunchDarkly flag value. This is the safest possible Germany design because it is purely additive: the existing FedEx bean's `getFlowType()` value, registration, and behavior are untouched, and carrier selection remains a simple LaunchDarkly-flag-driven value (e.g., targeting Germany traffic to a `"DHL"` flag value) rather than a code branch.

**Do NOT:** replace `InitReturnFedexFlow`'s logic, rename its `getFlowType()` value, or introduce a conditional branch inside a single shared flow class — any of these would risk regressing the existing US FedEx flow, and none are necessary given the loop already supports multiple beans.

## 8.2 Rating / Capability (CONFIRMED FROM CODE — NOT PRESENT TODAY)

**Does the existing FedEx SRS return flow perform rating?** A full search of `FedexClient.java`, `FedexPayload`, and the FedEx request/response structures documented in Sections 1-4 above shows **no rating, rate-shopping, or service-level-selection method or field anywhere in the current FedEx integration.** FedEx's returned RMA implicitly uses whatever service level FedEx's own account configuration applies — SRS does not request or compare rates before creating the RMA.

**Conclusion:** Because rating does not exist in the current flow, DHL's Rating/Capability API (confirmed to exist as a DHL capability in Section 6's CONFIRMED table) should **NOT** be built for Germany merely because DHL exposes it. Whether Germany requires:
- a **fixed** DHL product/service (mirroring today's "whatever FedEx gives us" pattern), or
- **dynamic** rate/service selection (a net-new capability with no FedEx precedent to reuse)

is **NEEDS BUSINESS CONFIRMATION** — owner: Business/Product. Until confirmed, the default, lowest-risk assumption consistent with current behavior is a **fixed** product code per Section 3's Requirement discussion (see also the `productCode` NEEDS CONFIRMATION entries #3/#4 in Section 6).

## 8.3 Germany Label Free / QR Code (CONFIRMED FROM DHL MATERIAL + explicit gaps)

The DHL sample payload already reproduced in Section 2 of this document contains direct evidence of Label Free-related fields that were not previously analyzed:

```json
"outputImageProperties": {
  "encodingFormat": "pdf",
  "imageOptions": [
    { "typeCode": "qr-code", "isRequested": true }
  ]
},
...
"valueAddedServices": [
  { "serviceCode": "PZ" },
  { "serviceCode": "PT" }
]
```

**What this confirms (CONFIRMED FROM DHL MATERIAL — the sample payload itself):**
- The DHL Shipment request can request a `qr-code` image alongside/instead of the PDF label via `outputImageProperties.imageOptions[typeCode="qr-code"]`.
- `valueAddedServices` with `serviceCode: "PZ"` and `serviceCode: "PT"` are present in the sample — these correspond to the PZ (Label Free-related) and PT (Data Staging-related) service codes referenced in the DHL team information supplied for this project.
- The QR code, per the DHL team information supplied separately for this analysis, is returned Base64-encoded in PNG format.

**What is explicitly NOT established (NEEDS BUSINESS/DHL CONFIRMATION):**
- Whether `PZ`/`PT`/Label Free/QR is actually **required** for this SRS Germany return journey, or whether the sample payload merely demonstrates that the *capability* exists. The sample payload is a capability demonstration, not a confirmed SRS business requirement — this document must not conflate the two.
- Whether Label Free needs to be separately **enabled on the DHL account** before `PZ`/`qr-code` requests will succeed (per DHL team information — this is an account-provisioning dependency, not just an API-request parameter).
- Whether LLT (service-content information) applies to this specific return scenario — not present anywhere in the supplied sample payload or DHL team information reviewed for this analysis.
- Whether the Germany return journey should avoid physical label printing entirely (Label Free/QR only) or should default to the conventional PDF label shown in the same sample (`outputImageProperties.encodingFormat: "pdf"`) — both are requested **simultaneously** in the sample, which itself does not resolve which one SRS's actual customer journey needs.

**Explicit disambiguation :**

| Concept | What it is |
|---|---|
| QR code (`imageOptions[typeCode="qr-code"]`) | A Base64 PNG image, an *alternative/supplement* to the printed label, used at drop-off/Service Point |
| `shipmentTrackingNumber` | The carrier's shipment identifier — NOT the QR code |
| PDF/ZPL label (`documents[].content`) | The conventional printed shipping document — NOT the QR code |
| `customerReferences[typeCode="RMA"]` | The SRS business return identifier — NOT any DHL-generated artifact |

**Conclusion:** Does SRS actually need Label Free for Germany? **This repository/business material cannot answer that question — NEEDS BUSINESS/DHL CONFIRMATION.** The sample payload proves DHL supports requesting both a QR code and a PDF label in the same call; it does not prove which one (or both) this specific SRS Germany project must implement. Owner: DHL team (account enablement mechanics) + Business/Product (customer journey decision).

## 8.4 Pickup / Drop-off / Service Point (CONFIRMED FROM DHL MATERIAL + explicit gaps)

The sample payload's `"pickup": { "isRequested": false }` (Section 2) only proves that the **sample** does not request pickup — it does not prove pickup is unavailable or unnecessary for Germany.

**Germany return logistics options, all structurally possible per the sample/DHL capability set, none yet confirmed required:**

1. Courier pickup (`pickup.isRequested: true`) + conventional PDF label
2. Courier pickup + Label Free/QR (no printing needed even with a courier)
3. Customer drop-off at a DHL Service Point + conventional PDF label
4. Customer drop-off at a DHL Service Point + Label Free/QR (fully paperless customer journey)

**Business decision required (NEEDS BUSINESS CONFIRMATION, owner: Business/Product):** which of these combinations SRS must support at Germany launch. This directly determines: (a) whether `pickup.isRequested` needs to be a per-return configurable value rather than a hardcoded `false`, and (b) whether the Label Free/QR capability from Section 8.3 must be built at all. Do not assume pickup is required — the current sample explicitly sets it to `false`, and no business requirement in this repository states otherwise.

## 8.5 Customs — Germany Scope Only (CONFIRMED FROM DHL MATERIAL + explicit gap)

The sample payload (Section 2) includes `"isCustomsDeclarable": false` and `"incoterm": "DAP"` — both CONFIRMED FROM DHL MATERIAL as fields that exist in the DHL request schema, but neither confirms which Germany origin/destination scenario this project actually supports.

**Scenarios to distinguish (NOT built globally, only where evidence supports it):**
- Germany → Germany (fully domestic return)
- EU country → Germany (cross-border within EU, customs-declarable status likely still `false` for intra-EU per common carrier practice, but not confirmed here)
- Non-EU country → Germany (customs-declarable likely `true`, additional customs fields required)

**This repository contains no evidence proving which of these scenarios is in scope** — the FedEx flow analyzed throughout this document is a US-domestic flow with no customs handling of any kind, so there is no existing SRS precedent to extend. The sample's `isCustomsDeclarable: false` is consistent with *either* a Germany-domestic *or* an intra-EU scenario, and does not by itself resolve the question.

**Conclusion:** Origin scope is **NEEDS BUSINESS CONFIRMATION** — owner: Business/Product. If Germany-domestic-only is confirmed, no additional customs fields are required beyond what the sample already shows (`isCustomsDeclarable: false`, `incoterm: "DAP"`). If any cross-border scenario is confirmed in scope, the exact mandatory customs fields (commercial invoice data, HS codes, customs value, etc.) must be re-derived from DHL for that specific route — this document does not invent a customs field list without that confirmation.

## 8.6 Error Handling / Retry / Idempotency (CONFIRMED FROM CODE + explicit DHL gap)

**Existing FedEx behavior (CONFIRMED FROM CODE):**
- `FedexClient.createRma()` / `createRmaS4()` are annotated `@Retryable(retryFor = HttpServerErrorException.class, backoff = @Backoff(delay = 100))` — retries occur **only** on 5xx server errors, with a 100ms fixed backoff delay (Spring Retry's default max attempts, since none is overridden).
- `HttpClientErrorException` (any 4xx) is caught, optionally logged (gated by a LaunchDarkly flag), and **re-thrown** — there is no dedicated 400/401/403/429 handling branch; all 4xx statuses fall into the same generic catch block.
- A blank/empty FedEx auth token results in an explicit `FedexUnauthorizedException`, thrown immediately with **no retry**.
- `RmaNumberLengthException` is thrown if the RMA exceeds 20 characters — a business-rule validation failure, not a network/HTTP failure.
- `FedexPreconditionFailedException` is thrown when required return-config data (`s4Customer`/`s4ShipToAddressId`) is missing from `returnConfigRepository`.
- **Duplicate-prevention for the async waybill-creation job path:** the scheduled job that eventually calls `FedexClient.createRmaS4()` filters candidate items by `status == processing && !waybillGenerated`, and only flips `waybillGenerated = true` after a successful (non-`"false"`-success) response; on any exception, it increments a retry counter and computes an **exponential backoff** wait time before the next attempt, deferring via a `nextRetryDate` field. This is the primary existing application-level safeguard against re-processing the same return repeatedly — it is not a DHL-side/FedEx-side idempotency key, it is an SRS-side "don't retry immediately, and don't reprocess something already marked done" pattern.
- No explicit request-correlation-ID header, and no FedEx-side idempotency-key mechanism, was found anywhere in `FedexClient`.

**Critical DHL scenario (per task requirement):** SRS sends Create Shipment → DHL successfully creates the shipment → SRS times out before receiving the response → SRS retries → **risk of a duplicate DHL shipment.**

- **Does DHL provide idempotency support?** NOT established from the DHL sample payload or DHL team information reviewed for this analysis — no idempotency-key field or duplicate-detection behavior is documented in the material available to this repository. **NEEDS DHL TEAM CONFIRMATION.**
- **Can an SRS business key (the RMA/`customerReferences[typeCode="RMA"]` value) safely prevent duplicates?** Only if DHL's API supports querying/deduplicating shipments by a caller-supplied reference before creating a new one — this is **not confirmed** by the material reviewed. **NEEDS DHL TEAM CONFIRMATION.**
- **Is reconciliation required?** If DHL has no idempotency support, a reconciliation step (query DHL by RMA reference before retrying, after a timeout) would be required — this is **PROPOSED DESIGN**, with no existing FedEx precedent to reuse directly (FedEx's own retry is scoped to 5xx only, and this repository shows no evidence FedEx has ever needed reconciliation logic for a "success then timeout" scenario specifically).

**HTTP status handling — existing FedEx vs proposed DHL (do not invent beyond this):**

| Status | Existing FedEx (CONFIRMED FROM CODE) | Proposed DHL handling |
|---|---|---|
| 400 | Generic `HttpClientErrorException` catch, re-thrown, no retry | PROPOSED: same — non-retryable, DHL-specific bad-request exception |
| 401/403 | Blank token → `FedexUnauthorizedException`, immediate failure, no retry | PROPOSED: DHL Basic Auth failures should similarly fail fast, non-retried |
| 429 | No explicit 429 branch found — falls into the generic client-error catch | NEEDS DHL/ARCHITECTURE CONFIRMATION — whether 429 should be retried with backoff is a new decision, not inherited from FedEx |
| 5xx | `@Retryable(retryFor = HttpServerErrorException.class)`, fixed 100ms backoff | PROPOSED: same retry-annotation pattern reusable for a DHL client method |
| Network timeout | No explicit timeout-specific handling found (relies on `RestTemplate` defaults) | NEEDS ARCHITECTURE CONFIRMATION — must be designed deliberately given the duplicate-shipment risk above, not inherited silently |
| Invalid address / invalid product / invalid package data | Not applicable to the current FedEx request shape (no address/package-level validation exists in FedEx's model) | NEEDS DHL MATERIAL CONFIRMATION — exact DHL error codes/response shape for these cases were not part of the material reviewed for this analysis |

Do not invent retry behavior beyond what is stated above — anything not explicitly confirmed here is PROPOSED DESIGN pending DHL/architecture sign-off.

## 8.7 Configuration / Secrets (dedicated section; CONFIRMED FROM CODE + PROPOSED DESIGN)

**Existing FedEx configuration (CONFIRMED FROM CODE, `FedexConfig`, `@ConfigurationProperties(prefix="fedex")`):** `contentType`, `xOrgName`, `orgName`, `accessTokenUri`, `grantType`, `scope`, `password`, `clientId`, `clientSecret`, `trackingUrl`, `createRmaUrl`, plus unrelated Reconext-report fields (`reconextReportsBucketName`, `reconextReportsFolder`, `reconextProcessedReportsFolder`) that must not be confused with carrier config. The FedEx **username** is sourced from a LaunchDarkly flag (`getFedExUserNameFlag()`), not from `FedexConfig` — an operational detail specific to FedEx's current credential management, not necessarily a pattern DHL needs to copy.

**Proposed DHL configuration (PROPOSED DESIGN — new, parallel `DhlConfig` class, not reusing `FedexConfig`):**

```properties
# Illustrative property names only — NOT actual secrets, NOT yet wired into code
dhl.baseUrl=
dhl.accountNumber=
dhl.apiKey=
dhl.apiSecret=
dhl.productCode=
dhl.trackingUrl=
dhl.labelFreeEnabled=
dhl.connectTimeoutMs=
dhl.readTimeoutMs=
dhl.retry.maxAttempts=
dhl.retry.backoffMs=
```

**Secret-management impact:**
- No actual DHL credential values should ever be committed to source. This repository's `deploy/*/service.yaml` files already reference FedEx and Gekko secrets via Kubernetes secret refs (e.g., existing `gekko-credentials-*` naming pattern per environment) rather than plaintext — a parallel `dhl-credentials-*` secret per environment (dev/pie/pro/stg/stg-west) is the PROPOSED, consistent approach.
- Credential **provisioning** itself (per the DHL team information already summarized in Section 3, Requirement 2) requires supplying DHL with: DHL Express account, customer contact name, email, telephone, address, city, postcode — an operational/business dependency that must be completed before any `dhl.*` secret value exists to configure, separate from the SRS code/config change itself.
- Whether Germany DHL uses a single account across all non-prod environments or per-environment DHL sandbox/test accounts is **NEEDS CONFIRMATION** — owner: DHL team + DevOps.

## 8.8 Observability

**Existing FedEx logging (CONFIRMED FROM CODE):** `FedexClient` logs the outbound request URL and full request/response payload (gated by LaunchDarkly flags `logResponseSuccessPayload()`/`logResponseErrorPayload()`), the HTTP status code on error, and the exception message. A dedicated `logObfuscateCustomer()` method explicitly redacts customer PII (e.g., email) before logging the outbound FedEx request.

**Proposed for DHL (PROPOSED DESIGN, modeled directly on the existing FedEx pattern):**
- Log: request correlation identifier (if DHL provides one), the RMA/`retReturnNumber`, the DHL shipment/waybill number once returned, the API operation name (create shipment vs. tracking query), response HTTP status, retry attempt count, failure reason, and call latency.
- **Do NOT log:** DHL Basic Auth credentials, the full Base64 label/QR content, or unredacted customer PII — reuse the existing `logObfuscateCustomer()`-style redaction pattern for any DHL request logging, consistent with current FedEx practice.

## 8.9 Backward Compatibility

**Proof the proposed Germany design does not break existing US FedEx behavior (CONFIRMED FROM CODE, by construction of the additive design in Section 8.1):**
- `resolveInitReturnFlow()` iterates over **all** registered `InitReturnFlow` beans and matches by string equality — adding a new `InitReturnDhlFlow` bean does not alter `InitReturnFedexFlow`'s bean registration, its `"Fedex"` flow-type string, or its behavior in any way.
- `FedexClient`, `FedexConfig`, `Rma`/`RmaShipment`/`RmaShippmentLabel`, and all FedEx-specific exception classes require **zero modification** to support DHL, provided a new, separate DHL client/config/model set is added alongside them (per Section 7's `DhlClient`/`DhlConfig`/`DhlShipmentRequest` proposals) rather than generalizing/merging them into the FedEx-shaped classes.
- `shipmentProvider`/`shippingVendor` is already a plain, non-enum `String` throughout (`ReturnDetailsDocument`, `EventAttributes`) — introducing the value `"DHL"` alongside `"Fedex"` requires no schema/type change.
- `Item.trackingNumber` is already carrier-neutral (`String`) — no schema change required to store a DHL tracking number.

**Regression risks to watch for (PROPOSED — risk list, not observed failures):**
- Any change to the LaunchDarkly flag's existing `"Fedex"` value (as opposed to purely adding a new `"DHL"` value/targeting rule) would be a regression risk — DHL must be introduced additively, never by altering the existing flag value.
- Repurposing `RmaShippmentLabel` or `FedexPayload` for DHL instead of adding new DHL-specific classes would risk breaking FedEx's exact JSON (de)serialization shape.
- If `MessagingService.getEventAttributes()`-equivalent tracking-number extraction is ever modified to be carrier-aware, the existing FedEx-specific extraction path must remain intact for FedEx-flagged returns — a carrier-aware branch, not a full rewrite, is the safe approach.

## 8.10 Testing Impact

| Test area | Type | Notes |
|---|---|---|
| DHL flow selection (`InitReturnDhlFlow.getFlowType() == "DHL"`, correctly selected by `resolveInitReturnFlow()`) | Unit | Mirror existing `InitReturnFedexFlowTest` pattern |
| FedEx regression (`resolveInitReturnFlow()` still selects FedEx when flag = "Fedex", unaffected by the new DHL bean's presence) | Unit | Add an explicit multi-bean-context test if none exists today |
| DHL request mapping (RMA → `customerReferences[typeCode="RMA"]`, shipper/receiver, packages, etc.) | Unit | Field-by-field, once mandatory DHL fields are finalized per Section 4/5 |
| Tracking number extraction (`shipmentTrackingNumber` → `Item.trackingNumber`) | Unit | New extraction path, distinct from FedEx's nested `rma.shipments[0].label.trackingNumber` read |
| Tracking URL construction | Unit | Once `dhl.trackingUrl` format is confirmed |
| Label parsing (PDF Base64, ZPL Base64) | Unit | Only if Section "Label Analysis" persistence decision requires SRS to process label content at all |
| Label Free QR Base64 PNG handling | Unit | Only if Section 8.3 confirms Label Free is in scope |
| DHL status → SRS status mapping | Unit | Recommended once a DHL-event-to-`ReturnState` table is business-confirmed (no such mapping exists in this document prior to this audit pass — see Open Questions in Section 8.13) |
| Pickup true/false | Unit | Covers both `pickup.isRequested` values per Section 8.4 business decision |
| Error handling (400/401/403/429/5xx/timeout/invalid address/invalid product/invalid package) | Unit + Integration | Mirror `FedexClient`'s existing exception-handling coverage pattern |
| Duplicate/retry scenario (Section 8.6) | Integration | Requires a DHL sandbox or a mock able to simulate a timeout-after-success scenario |
| EMS event generation (DHL values flow correctly into the existing `EventAttributes`-equivalent contract) | Unit | Extend existing messaging-service test coverage for a DHL-flagged document |
| Contract testing | Contract | Recommended against the actual MyDHL v3.3.1 OpenAPI schema before production cutover — not yet available in this repository |

---

# 9. FINAL Consolidated Deliverables

## 9.1 Final Consolidated Gap Analysis Table

| Area | Current FedEx behavior | DHL Germany behavior | Code evidence | DHL evidence | Gap | Required change | Status | Owner |
|---|---|---|---|---|---|---|---|---|
| Carrier selection | Strategy pattern, `InitReturnFlow` beans matched by `getFlowType()` vs LaunchDarkly flag | Add `InitReturnDhlFlow`, `getFlowType()→"DHL"` | `ReturnDetailsService.java:242-249`, `InitReturnFedexFlow.java:65-66` | N/A | New bean + new flag value | Add `InitReturnDhlFlow` + `DhlClient` | PROPOSED | SRS team |
| RMA | Business identifier, received not generated, root-level `rmaNumber` in request | `customerReferences[typeCode="RMA"]` | Section 7.1 (full) | DHL reference-data guide | Placement only, value reusable (95%) | New mapping code in `DhlClient` | CONFIRMED (mapping approach) | SRS team |
| Authentication | OAuth2 password grant, Bearer token, `FedexTokenHolder` | HTTP Basic Auth, stateless, no token holder needed | Section 3, Requirement 2 | DHL email + API docs | Different mechanism entirely, simpler for DHL | New `DhlConfig`/`DhlClient` auth, not reusing `FedexTokenHolder` | CONFIRMED (both sides) | DHL team |
| Tracking number | `rma.shipments[0].label.trackingNumber` (nested, 12-digit numeric) | `shipmentTrackingNumber` (top-level, 10-digit alphanumeric) | Section 3, Requirement 1 | DHL sample payload | Different nesting AND different format/length — treat as opaque | New extraction path into existing carrier-neutral `Item.trackingNumber` | CONFIRMED (fields compatible) | SRS team |
| Tracking URL | SRS-constructed: `fedex.trackingUrl` + trackingNumber | Same construction pattern proposed with `dhl.trackingUrl`; exact format given in Section 3 test/prod tables | Section 3, Requirement 1 | DHL sample tracking URL + endpoint table | New config value | Add `dhl.trackingUrl` config | CONFIRMED (URL format) / NEEDS CONFIRMATION (is it the correct customer-facing URL) | DHL team |
| Label | `labelURL` (pull, URL-by-reference), stored, unused by SRS logic, but exposed via internal GET endpoint | Base64 content (push, by value) in `documents[]`, PDF or ZPL | Label Analysis section (full) | DHL sample payload | Entirely different delivery model | New DHL-specific document handling, only if business confirms a need | NEEDS CONFIRMATION (is label persistence needed at all) | Business/Product |
| Label Free / QR | N/A — no equivalent concept in FedEx flow | `imageOptions[typeCode="qr-code"]`, `valueAddedServices[PZ,PT]`, Base64 PNG, account-enablement required | N/A | Sample payload + DHL team information (Section 8.3) | Entire new capability, not yet scoped | Build only if confirmed required | NEEDS CONFIRMATION | DHL team + Business |
| Product code | N/A (FedEx has no explicit product/service selection) | Sample uses `U`; domestic vs EU vs non-EU code NOT established | Section 4 | DHL sample payload | Must confirm correct product for each route | Confirm before implementation | NEEDS CONFIRMATION | DHL team + Business |
| Weight/dimensions | Not present in `Item` model | Required for `content.packages[]` | Section 4 (full) | DHL sample payload | Data source not identified in this repository | Investigate order/fulfillment/product-catalog systems before adding fields to `Item` | NEEDS CONFIRMATION (data source) | SRS team + Business |
| Rating | Not present in current FedEx flow (no rating/rate-shopping code found) | DHL exposes Rating/Capability but may not be needed if a fixed product/service is used | Section 8.2 | DHL capability docs | Only relevant if dynamic selection required | Do not build unless confirmed | NEEDS CONFIRMATION | Business/Product |
| Pickup/drop-off | Not evidenced as a distinct capability in current FedEx flow | `pickup.isRequested` / Service Point; 4-option business decision (Section 8.4) | N/A | DHL sample payload | Business decision on logistics model | Build only the confirmed combination(s) | NEEDS CONFIRMATION | Business/Product |
| Customs | Not present (US domestic flow) | Only relevant if origin scope crosses a customs border | N/A | Sample payload (`isCustomsDeclarable`, `incoterm`) | Origin scope unconfirmed | Build only if scope requires it | NEEDS CONFIRMATION | Business/Product |
| S4/EMS event contract | `EventAttributes`-equivalent: trackingNumber, shippingVendor, trackingUrl; carrier is plain String | Same fields, new String value `"DHL"` | Label Analysis §5/§12 | N/A | No structural gap for tracking fields; label field would be net-new if ever required | None required for tracking/URL; new field only if label forwarding is confirmed needed | CONFIRMED (structural compatibility) / NEEDS CONFIRMATION (S4-side acceptance of "DHL" value) | S4/EMS team |
| Idempotency/retry | `@Retryable` on 5xx only; app-level `waybillGenerated` flag + exponential backoff | Duplicate-shipment risk on timeout-then-retry; DHL idempotency support unconfirmed | Section 8.6 | Not established in supplied DHL material | Need duplicate-prevention strategy | Confirm DHL idempotency support; design reconciliation if absent | NEEDS CONFIRMATION | DHL team + Architecture |
| Configuration/secrets | `FedexConfig` (`fedex.*`), k8s secret refs per environment | New `DhlConfig` (`dhl.*`), new k8s secret refs | Section 8.7 | DHL credential-provisioning requirements | New config class + secrets per environment | Add `DhlConfig` + secrets | PROPOSED | SRS team + DevOps |

## 9.2 Final Implementation Impact

**A. Files/classes that DEFINITELY require modification** (evidence-based — none, beyond LaunchDarkly configuration, since the existing `resolveInitReturnFlow()` loop and `InitReturnFedexFlow` need no code change to support an additive DHL bean):
- LaunchDarkly flag configuration for the shipment-provider flag — a new targeting value (`"DHL"`) must be added, a config change rather than a Java source file.

**B. Files/classes that MAY require modification** (conditional on business decisions):
- The `MessagingService`/`EventAttributes`-equivalent tracking-number extraction, only if a carrier-aware branch is needed distinct from the FedEx-specific nested read.
- `ReturnDetailsDocument`-equivalent model, only if new DHL-specific fields (Label Free QR content, DHL-specific reference IDs) are confirmed necessary.
- `Item` model, only if weight/dimensions/package data is confirmed to not exist elsewhere and must be added here (Section 4) — NOT to be modified reflexively.

**C. New classes/DTOs/config that MAY be required:**
- `InitReturnDhlFlow` (implements `InitReturnFlow`, `getFlowType() → "DHL"`)
- `DhlClient`, `DhlConfig`, `DhlShipmentRequest` (already sketched in Section 3/7 of this document)
- Optional: DHL-specific label/document model (only if label persistence is confirmed necessary)
- Optional: DHL status-mapper component (only if SRS must actively map DHL tracking events, as opposed to relying solely on the existing S4 callback)
- Optional: new DHL-specific exception types, parallel to existing `FedexUnauthorizedException`/`FedexClientException`/`FedexPreconditionFailedException`/`RmaNumberLengthException`

**D. Existing classes that should remain UNTOUCHED:**
- `FedexClient`, `FedexTokenHolder`/token logic, `FedexConfig`, `Rma`/`RmaShipment`/`RmaShippmentLabel`, `InitReturnFedexFlow`, and all FedEx-specific exception classes.

**E. S4/EMS contract changes:** None required for tracking number, tracking URL, or carrier-name propagation (all already carrier-neutral plain Strings, per Label Analysis §5/§12). A new field is only required if label content/URL forwarding to EMS/S4 is confirmed as a business requirement.

**F. Database/model changes:** None required to store a DHL tracking number or carrier name. Possibly required: new fields for DHL-specific label/document content (only if confirmed necessary) and for weight/dimensions/package data (only if confirmed no existing source and Germany genuinely requires SRS to supply these values).

**G. Configuration/secret changes:** New `dhl.*` properties (Section 8.7) and new environment-specific secret references, parallel to existing `fedex.*`/`gekko-credentials-*` patterns.

**H. Test changes:** See Section 8.10 in full.

**I. External dependencies:** DHL Express account + API credentials (provisioning dependency per DHL team information); possibly a DHL sandbox/mock server for contract testing (not yet available in this repository).

**J. Business decisions still blocking implementation:**
1. Germany return logistics model (pickup vs. drop-off vs. Label Free — Section 8.4)
2. Whether Label Free/QR is required at all (Section 8.3)
3. Correct DHL product code(s) for domestic vs. EU vs. non-EU Germany return scenarios (Section 4/6)
4. Customs scope (Germany-domestic only vs. EU/non-EU origin — Section 8.5)
5. Whether DHL label content/document must be persisted or forwarded downstream at all (Label Analysis section)
6. Shipper/receiver/payer/account-typeCode configuration for a Germany return (Executive Summary + Section 4 address mapping)
7. Whether SRS must actively map DHL tracking events into a common status, or whether S4 remains the sole status-transition source
8. DHL idempotency/duplicate-shipment handling strategy (Section 8.6)

## 9.3 Final Open Questions — Owner Based

**DHL TEAM**
1. Does the MyDHL API instance provisioned for this project support idempotent shipment creation, or must SRS implement its own reconciliation-before-retry logic to avoid duplicate shipments on timeout (Section 8.6)?
2. Is PZ/Label Free/QR required for the Germany SRS customer-return journey, or should SRS generate a conventional PDF/ZPL return label (Section 8.3)?
3. Is `productCode = "U"` (per the sample) or `"N"` actually correct for a Germany-domestic customer return, and does the correct product differ by origin (EU vs. non-EU) or DHL account/service agreement (Section 4, NEEDS CONFIRMATION items #3/#4)?
4. What HTTP status/response body does DHL return specifically for invalid address, invalid product code, and invalid package data (Section 8.6)?
5. Is `https://www.dhl.com/express/track?AWB=<number>` the correct, stable, production customer-facing tracking URL for this account/region (Section 3, Requirement 1)?
6. Can `documents[]` contain more than one document per shipment response, and if so, what field reliably identifies "the label" vs. other documents?

**S4/EMS TEAM**
7. Does S4 validate or enum-constrain the `carrierName`/`shippingVendor` value? Will `"DHL"` be accepted without an S4-side contract or configuration change (Section 6, NEEDS CONFIRMATION #6)?
8. Does S4/EMS need the DHL label document/content at all, or is tracking number + tracking URL sufficient (matching what is actually used today for FedEx, per the Label Analysis section)?
9. Does S4 validate the tracking-number format/length? FedEx's is 12-digit numeric; DHL's is 10-digit alphanumeric (or longer at the package level, e.g. `JD999999999999999999`) — will relaxed validation be needed (Section 6, NEEDS CONFIRMATION #7)?

**BUSINESS/PRODUCT**
10. Which of the four Germany return logistics combinations (courier pickup + label, courier pickup + Label Free, drop-off + label, drop-off + Label Free — Section 8.4) must SRS support at launch?
11. Is the Germany DHL project scoped to Germany-domestic returns only, or must it also support EU-country-to-Germany or non-EU-country-to-Germany returns (Section 8.5)?
12. For a Germany customer return, should the DHL account holder/payer always be HP, or can/should the customer ever be the payer (Executive Summary shipper/receiver roles)?
13. Is a customer phone number mandatory for the DHL shipment request, and if current SRS/FedEx flow does not capture it today, where should it come from (Section 6, NEEDS CONFIRMATION #11)?

**ARCHITECTURE**
14. Should `shippingVendor`/`carrierName` be promoted from a plain `String` to a shared enum now that a second carrier is being introduced, given both directions of the S4/EMS contract currently leave it unconstrained?
15. Should DHL label persistence (if required) be inline Base64, object storage + URL, or a dedicated DHL document model (per the Label Analysis section's Options A-D)?
16. Should a DHL-specific reconciliation/idempotency-check step be built proactively, or should SRS accept the same retry-on-5xx-only risk profile currently accepted for FedEx (Section 8.6)?

**SRS TEAM**
17. Where, if anywhere, can weight/dimensions/package-count/description data for a return shipment be sourced today (order data, fulfillment data, product catalog, or another integration) — confirm the exact gap identified in Section 4/6?
18. Does the existing `resolveInitReturnFlow()` bean-selection loop have any test coverage that assumes exactly one `InitReturnFlow` bean exists, which would need updating once a second (`InitReturnDhlFlow`) bean is registered (Section 8.1)?

## 9.4 Final End-to-End Germany Flow (corrected against repository evidence)

```text
Customer requests return
        ↓
ReturnDetailsController.sendReturn() → ReturnDetailsService.sendReturn()
        ↓
RMA (business identifier — received/stored as retReturnNumber, not authenticated; Section 7.1)
        ↓
resolveInitReturnFlow() — country/carrier selection via LaunchDarkly flag (Section 8.1)
        ↓
        ├── "Fedex" → InitReturnFedexFlow → FedexClient (UNCHANGED)
        └── "DHL"   → InitReturnDhlFlow (NEW) → DhlClient (NEW)
                          ↓
                    DHL HTTP Basic Auth (per DHL email/API docs — Section 3, Requirement 2)
                          ↓
                    Map SRS return → DHL shipment request
                       (RMA → customerReferences[typeCode="RMA"] — Section 7.1;
                        shipper/receiver, account/payer — NEEDS CONFIRMATION per Executive Summary;
                        product code — NEEDS CONFIRMATION per Section 4;
                        weight/dimensions — NEEDS CONFIRMATION per Section 4;
                        pickup.isRequested — NEEDS BUSINESS DECISION per Section 8.4;
                        customs fields — only if Section 8.5 scope requires them)
                          ↓
                    Create DHL shipment (generic Shipment API — no dedicated Return endpoint exists, per Executive Summary)
                          ↓
                    shipmentTrackingNumber (waybill) returned
                          ↓
                    Label (PDF/ZPL Base64) OR Label-Free QR (Base64 PNG) — per Section 8.3/8.4 business decision
                          ↓
                    Persist ONLY what is confirmed necessary:
                       - trackingNumber → Item.trackingNumber (carrier-neutral, no schema change needed)
                       - trackingUrl → carrier-neutral field (no schema change needed)
                       - label/QR content → ONLY if Label Analysis section/Section 8.3 confirms a persistence need
        ↓
Publish to EMS-equivalent event contract (trackingNumber, shippingVendor="DHL", trackingUrl) — Label Analysis §5/§12
        ↓
Downstream consumers: S4/SAP, eClaims, other listeners (per existing topology — UNCHANGED)
        ↓
Inbound: S4 status callback → existing status-handling path (UNCHANGED mechanism)
        ↓
Common SRS status transition — OR, only if Section 8.6/status-mapping scope is confirmed, a new DHL tracking-event
   → DHL status mapper → common status path, run in parallel to/instead of the existing S4 callback
```

**Key correction vs. a naive "EMS → S4/downstream consumers" one-directional template:** this repository's evidence (Label Analysis §5, Section 7) shows SRS publishes outbound status **to** an event-manager layer (consumed by S4 downstream), and separately receives an explicit inbound status **callback** from S4. Both directions must be accounted for in the DHL design, not just the outbound one.

## 9.5 Final Quality Check — Contradiction Resolution Log

| Check | Resolution |
|---|---|
| RMA = CU vs RMA = RMA | No contradiction — Section 3 Requirement 3 and Section 6's CONFIRMED table consistently state `RMA = RMA Number` (correct, `typeCode="RMA"`) and explicitly warn `CU = Consignor reference number` is a **different** reference type. The Section 2 sample payload's use of `typeCode: "CU"` for a generic reference is clearly marked in Section 3 as something to correct to `typeCode: "RMA"` for the actual RMA value — not a contradiction, but the exact reason this document calls out the correction. |
| DHL label only PDF vs PDF/ZPL | No contradiction — the Label Analysis section and DHL team information both state PDF **or** ZPL (Base64), never PDF-only. |
| SRS calls S4 directly vs SRS publishes to EMS | Resolved in the Label Analysis section (§5/§8/§12): SRS publishes outbound events to an event-manager layer; S4 is a downstream consumer per diagrams; S4 only reaches SRS via an explicit inbound callback. Section 7's "S4 Integration Impact" heading refers to this same downstream-consumer relationship, not a direct outbound SRS→S4 call — no contradiction once both sections are read together, but this final section makes the distinction explicit for any reader who only reads Section 7 in isolation. |
| labelURL unused vs labelURL persisted/exposed by internal GET | Reconciled in the Label Analysis section (§1 item 10, §16a Check 1): both are true and not contradictory — `labelURL` is persisted and exposed via one specific internal GET endpoint, but never read/used by SRS's own business logic beyond that. |
| DHL tracking URL vs customer-facing tracking URL | Kept distinct — Section 3 Requirement 1 provides both the DHL API tracking-query endpoint table (test/production URLs for querying tracking data) and a separate customer-facing browser URL (`https://www.dhl.com/express/track?AWB=...`); Section 9.3 Q5 correctly flags the latter as still NEEDS CONFIRMATION. |
| Pickup = Return | Explicitly rejected in this document's Section 8.4 addition and consistent with the Executive Summary's return/shipment distinction — not treated as equivalent anywhere. |
| productCode U treated as confirmed | Consistently marked NEEDS CONFIRMATION in Section 4, Section 6 (items #3/#4), and Section 9.3 Q3 — never upgraded to CONFIRMED anywhere in this document. |
| customer always shipper treated as confirmed | Consistently marked NEEDS CONFIRMATION in Section 6 (items #1/#2) and Section 9.3 Q12 — never upgraded to CONFIRMED. |
| DHL Base64 assumed required downstream | Corrected in the Label Analysis section and Section 9.1/9.2: explicitly NOT assumed required; current EMS-equivalent contract has no label field, so nothing is broken, and forwarding is only a NEEDS CONFIRMATION item. |
| Label Free treated as mandatory without evidence | Corrected in Section 8.3: explicitly stated as a DHL CAPABILITY demonstrated in the sample payload, not a confirmed SRS BUSINESS REQUIREMENT, with an explicit "cannot be answered from this repository" conclusion. |

No historical code evidence was deleted during this audit pass — all prior file/line references, code snippets, and tables from Sections 1-7 and the Label Analysis/FINAL OUTPUT block remain exactly as originally written.

## 9.6 FINAL CONFIDENCE / READINESS

**CONFIRMED FROM CODE**
- Carrier selection mechanism (`resolveInitReturnFlow()`, `getFlowType()`, LaunchDarkly-driven, multi-bean strategy pattern already proven with `InitReturnFedexFlow`/`InitReturnNoFlow`)
- RMA is a received (not generated), plain business identifier, not an auth token (Section 7.1, exhaustive)
- FedEx authentication mechanism (OAuth2 password grant, `FedexTokenHolder`, 5 required credentials)
- FedEx label is URL-by-reference (`labelURL`), persisted but functionally unused beyond one internal GET-endpoint exposure (Label Analysis)
- FedEx tracking number/URL extraction and propagation paths, exactly, with file/line evidence (Sections 1, 3)
- SRS's outbound event path is an EMS-equivalent publish, not a direct S4 call; S4 reaches SRS only via an inbound callback (Label Analysis §5/§8/§12)
- `Item.trackingNumber` and carrier/shippingVendor fields are plain, carrier-neutral, non-enum Strings
- Existing FedEx retry behavior (5xx only, fixed 100ms backoff) and the async waybill job's exponential-backoff/flag-based idempotency pattern (Section 8.6)
- Weight/dimensions/package data do NOT exist in the `Item` model today (Section 4)

**CONFIRMED FROM DHL** (material/team information supplied for this project)
- DHL Shipment, Pickup, Tracking, Rating/Capability API surfaces exist; no dedicated Return-Shipment endpoint (Section 6)
- DHL reference-data distinguishes `RMA` (RMA Number) from `CU` (Consignor reference number)
- DHL supports label as PDF or ZPL, Base64-encoded, via `documents[]`, plus a QR-code image option and `PZ`/`PT` value-added services in the same sample request (Section 8.3)
- DHL requires a DHL Express account + credentials, provisioned with specific contact/address information (Section 3, Requirement 2)
- HTTP Basic Auth, stateless, no token-refresh lifecycle needed (a genuine simplification vs. FedEx)

**PROPOSED DESIGN**
- New `InitReturnDhlFlow` / `DhlClient` / `DhlConfig`, additive to (not replacing) FedEx equivalents
- Reuse of the existing config-prefix + tracking-number string-concatenation pattern for a DHL tracking URL
- A dedicated DHL document model if label persistence is ever required
- A DHL status-mapper component, only if scope confirms SRS must actively consume DHL tracking events

**OPEN BUSINESS DECISIONS**
- Germany return logistics model (pickup vs. drop-off vs. Label Free)
- Whether Label Free/QR is required at all
- Correct DHL product code(s) per route
- Customs scope (domestic-only vs. EU/non-EU origin)
- Whether DHL label content must be persisted/forwarded downstream
- Shipper/receiver/payer/account-typeCode configuration

**OPEN TECHNICAL DECISIONS**
- DHL idempotency/duplicate-shipment prevention strategy
- Whether `shippingVendor`/`carrierName` should become an enum
- Whether SRS should actively map DHL tracking events into a common status, or rely solely on the existing S4 callback

**BLOCKERS BEFORE DEVELOPMENT**
1. DHL account provisioning (credentials, endpoint) must be completed by the DHL team before any live integration testing is possible.
2. Product code and customs scope must be confirmed by DHL/Business before the shipment-request mapping can be finalized.
3. The Germany logistics model (pickup/drop-off/Label Free) must be confirmed by Business before pickup/Label-Free code paths are built.
4. DHL idempotency behavior must be confirmed before the duplicate-shipment risk (Section 8.6) can be considered mitigated.

 

