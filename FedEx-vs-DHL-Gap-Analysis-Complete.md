# FedEx vs DHL Express - Complete Gap Analysis for Germany Expansion
**SRS (Stratus Returns Service) - Germany Region Implementation**

**Date:** August 13, 2026  
**Analyst:** Anjali Taluri Sri Sai Latha  
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
3. [Three Critical Requirements Analysis](#three-critical-requirements)
4. [Field-by-Field Mapping Comparison](#field-mapping)
5. [Gap Analysis & Missing Fields](#gap-analysis)
6. [S4 Integration Impact](#s4-integration)
7. [Implementation Recommendations](#implementation-recommendations)
8. [Code Changes Required](#code-changes)

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
      "content": "JVBERi0xLjQK..." // Base64 label
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

**From DHL Documentation (provided screenshots):**

| Endpoint Type | Test URL | Production URL |
|---------------|----------|----------------|
| All shipment statuses | `https://express.api.dhl.com/mydhlapi/test/shipments/{trackingNumber}/tracking` | `https://express.api.dhl.com/mydhlapi/shipments/{trackingNumber}/tracking` |
| Shipment with last checkpoint | `https://express.api.dhl.com/mydhlapi/test/shipments/{trackingNumber}/tracking?trackingView=last-checkpoint` | `https://express.api.dhl.com/mydhlapi/shipments/{trackingNumber}/tracking?trackingView=last-checkpoint` |
| All checkpoints with remarks | `https://express.api.dhl.com/mydhlapi/test/tracking?shipmentTrackingNumber={trackingNumber}&trackingView=all-checkpoints-with-remarks` | `https://express.api.dhl.com/mydhlapi/tracking?shipmentTrackingNumber={trackingNumber}&trackingView=all-checkpoints-with-remarks` |

**DHL Tracking Event Codes (from provided image):**

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

**RMA Location:** Inside `customerReferences` array

```json
{
  "customerReferences": [
    {
      "value": "RET-20260813-001",
      "typeCode": "CU"
    }
  ]
}
```

**From DHL API Documentation:**
> "Reference of your shipment or individual package. This is usually the order number from your system. CU = Consignor's reference number, the value is a constant."

**⚠️ Critical Understanding:**

| Concept | Description | Location | Example |
|---------|-------------|----------|---------|
| **RMA** | Business return identifier | `customerReferences[typeCode='CU'].value` | `"RET-20260813-001"` |
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
        private String typeCode;   // "CU"
    }
}

// DhlClient.java
public DhlShipmentRequest buildRequest(ReturnDetailsDocument doc) {
    DhlShipmentRequest request = new DhlShipmentRequest();
    
    // Add RMA as customer reference
    CustomerReference rma = new CustomerReference();
    rma.setValue(doc.getRetReturnNumber());
    rma.setTypeCode("CU");
    request.getCustomerReferences().add(rma);
    
    // ... rest of mapping
}
```

---

## 4. Field-by-Field Mapping Comparison

### **Root Level Fields**

| Field Purpose | FedEx Field | FedEx Type | DHL Express Field | DHL Type | Mapping Complexity |
|---------------|-------------|------------|-------------------|----------|--------------------|
| **RMA/Return ID** | `rmaNumber` | String (max 20) | `customerReferences[typeCode='CU'].value` | String | 🟡 Medium - Different structure |
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

## 6. S4 Integration Impact

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

**✅ GOOD NEWS:** Field `carrierName` exists!

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

## 7. Implementation Recommendations

### **7.1 Architecture Pattern - Multi-Carrier Design**

**Keep existing pattern:** Strategy Pattern with `InitReturnFlow` interface

```
InitReturnFlow (interface)
    ├── InitReturnFedexFlow (USA)
    ├── InitReturnDhlFlow (Germany)  ← NEW
    └── InitReturnNoFlow (Fallback)
```

**Router Logic (already exists):**
```java
// ReturnDetailsService.java - line 241
private ReturnDetailsDocument resolveInitReturnFlow(ReturnDetailsDocument payload) {
    for (InitReturnFlow initReturnFlow : initReturnFlows) {
        if (initReturnFlow.getFlowType().equalsIgnoreCase(
                launchDarklyClient.getShipmentProviderFlag())) {
            return initReturnFlow.initReturn(payload);
        }
    }
    return payload;
}
```

**LaunchDarkly Configuration:**
```json
{
  "shipment-provider": {
    "US": "Fedex",
    "DE": "DHL",
    "default": "Fedex"
  }
}
```

### **7.2 DHL Product Code Determination**

**From provided Global Product Codes table:**

| Use Case | Product Code | Product Name | Content | Transport Type |
|----------|-------------|--------------|---------|----------------|
| **Germany Domestic** | `N` | EXPRESS DOMESTIC | Y-Doc | Land within country |
| **EU Cross-Border** | `U` | EXPRESS WORLDWIDE | Y-Doc | Air within EU |
| **Non-EU to EU** | `P` | EXPRESS WORLDWIDE | N-Non Doc | Air outside EU |

**Determination Logic:**
```java
public String determineProductCode(String originCountry, String destCountry) {
    // Germany returns → HP warehouse in Germany
    if ("DE".equals(originCountry) && "DE".equals(destCountry)) {
        return "N";  // EXPRESS DOMESTIC
    }
    
    // EU country → Germany
    if (isEUCountry(originCountry) && "DE".equals(destCountry)) {
        return "U";  // EXPRESS WORLDWIDE (EU)
    }
    
    // Non-EU → Germany
    return "P";  // EXPRESS WORLDWIDE (non-EU)
}
```

### **7.3 Address Configuration**

**Create new MongoDB config for DHL receiver addresses:**

```json
{
  "key": "dhlReceiverAddress_DE",
  "value": {
    "postalAddress": {
      "cityName": "Bonn",
      "countryCode": "DE",
      "postalCode": "53113",
      "addressLine1": "Charles-de-Gaulle-Str. 20"
    },
    "contactInformation": {
      "companyName": "HP Deutschland GmbH",
      "fullName": "HP Returns Department",
      "email": "returns.de@hp.com",
      "phone": "+4922899360"
    }
  }
}
```

### **7.4 Weight & Dimensions Strategy**

**Option 1: Add to Item Model (Preferred)**
```java
@Document(collection = "returnDetails")
public class Item {
    private String sku;
    private int quantity;
    private ReturnState status;
    private String returnReason;
    
    // NEW fields
    private Double weight;  // in kg or lbs
    private Dimensions dimensions;
}

@Data
public class Dimensions {
    private Integer length;  // cm or inches
    private Integer width;
    private Integer height;
}
```

**Option 2: SKU-Based Lookup (Fallback)**
```java
public PackageDimensions getPackageDimensions(String sku) {
    // Call product catalog service
    ProductInfo product = productCatalog.getBySku(sku);
    return new PackageDimensions(
        product.getWeight(),
        product.getLength(),
        product.getWidth(),
        product.getHeight()
    );
}
```

**Option 3: Default Values (Quick Start)**
```java
// For testing/MVP
private static final double DEFAULT_WEIGHT = 2.5;  // kg
private static final int DEFAULT_LENGTH = 30;  // cm
private static final int DEFAULT_WIDTH = 20;
private static final int DEFAULT_HEIGHT = 15;
```

### **7.5 QR Code / Label-Free Service**

**From DHL Email:**
> "The Label Free service must first be enabled on your API account."

**Service Codes:**
- `PZ` - Label Free service
- `PT` - Data Staging 03 (stores data for 3 months)

**Implementation:**
```java
// For Label-Free returns
if (launchDarklyClient.isLabelFreeEnabled()) {
    request.getValueAddedServices().add(
        new ValueAddedService("PZ")
    );
    request.getValueAddedServices().add(
        new ValueAddedService("PT")
    );
    
    // Request QR code
    request.getOutputImageProperties()
        .getImageOptions().add(
            new ImageOption("qr-code", true)
        );
}
```

---

## 8. Code Changes Required

### **8.1 New Java Classes**

#### **1. DhlConfig.java**
```java
package com.hp.tropos.config.properties;

@ConfigurationProperties(prefix = "dhl.express")
@Getter
@Setter
public class DhlConfig {
    private String apiKey;
    private String apiSecret;
    private String accountNumber;
    private String baseUrl;
    private String testBaseUrl;
    private String trackingUrl;
}
```

#### **2. DhlClient.java**
```java
package com.hp.tropos.clients.dhl;

@Slf4j
@Component
public class DhlClient {
    private final DhlConfig dhlConfig;
    private final RestTemplate restTemplate;
    private final ReturnConfigRepository returnConfigRepository;
    private final LaunchDarklyClient launchDarklyClient;
    
    @Retryable(retryFor = HttpServerErrorException.class)
    public ReturnDetailsDocument createShipment(
            ReturnDetailsDocument doc, 
            AttributionFlag attributionFlag) {
        
        DhlShipmentRequest request = buildRequest(doc);
        HttpHeaders headers = buildAuthHeaders();
        
        HttpEntity<DhlShipmentRequest> entity = 
            new HttpEntity<>(request, headers);
        
        DhlShipmentResponse response = restTemplate.postForObject(
            dhlConfig.getBaseUrl() + "/shipments",
            entity,
            DhlShipmentResponse.class
        );
        
        return mapResponse(response, doc);
    }
    
    private HttpHeaders buildAuthHeaders() {
        String auth = dhlConfig.getApiKey() + ":" + dhlConfig.getApiSecret();
        String encodedAuth = Base64.getEncoder()
            .encodeToString(auth.getBytes(StandardCharsets.UTF_8));
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Basic " + encodedAuth);
        headers.set("Content-Type", "application/json");
        return headers;
    }
    
    private DhlShipmentRequest buildRequest(ReturnDetailsDocument doc) {
        // Map fields from doc to DHL request
        // See detailed mapping below
    }
}
```

#### **3. InitReturnDhlFlow.java**
```java
package com.hp.tropos.service.dhl;

@Slf4j
@Service
@Transactional
public class InitReturnDhlFlow implements InitReturnFlow {
    
    private final DhlClient dhlClient;
    private final AttributionIdService attributionService;
    
    @Override
    public ReturnDetailsDocument initReturn(ReturnDetailsDocument payload) {
        AttributionFlag attributionFlag = attributionService
            .getAttributionFlag(payload.getOrderAttributionId());
        
        ReturnDetailsDocument response = dhlClient.createShipment(
            payload, 
            attributionFlag
        );
        
        if (response != null && response.isSuccess()) {
            payload.setRma(response.getRma());
            payload.setTrackingUrl(buildTrackingUrl(
                response.getRma().getShipments().get(0)
                    .getLabel().getTrackingNumber()
            ));
            // Set history, etc.
            return payload;
        }
        
        throw new ShipmentProviderException("DHL response was not found");
    }
    
    @Override
    public String getFlowType() {
        return "DHL";
    }
    
    private String buildTrackingUrl(String trackingNumber) {
        return "https://www.dhl.com/express/track?AWB=" + trackingNumber;
    }
}
```

#### **4. DHL Request/Response DTOs**

```java
@Data
public class DhlShipmentRequest {
    private String productCode;
    private String plannedShippingDateAndTime;
    private Pickup pickup;
    private List<Account> accounts;
    private List<CustomerReference> customerReferences;
    private OutputImageProperties outputImageProperties;
    private CustomerDetails customerDetails;
    private Content content;
    private List<ValueAddedService> valueAddedServices;
    
    @Data
    public static class CustomerReference {
        private String value;
        private String typeCode;
    }
    
    @Data
    public static class CustomerDetails {
        private ShipperDetails shipperDetails;
        private ReceiverDetails receiverDetails;
    }
    
    @Data
    public static class Content {
        private String unitOfMeasurement;
        private Boolean isCustomsDeclarable;
        private String incoterm;
        private String description;
        private List<Package> packages;
    }
    
    @Data
    public static class Package {
        private Double weight;
        private Dimensions dimensions;
    }
    
    // ... other nested classes
}

@Data
public class DhlShipmentResponse {
    private String shipmentTrackingNumber;
    private String trackingUrl;
    private List<PackageResponse> packages;
    private List<Document> documents;
}
```

### **8.2 Configuration Updates**

#### **application.properties**
```properties
# DHL Express Configuration
dhl.express.apiKey=${DHL_API_KEY}
dhl.express.apiSecret=${DHL_API_SECRET}
dhl.express.accountNumber=${DHL_ACCOUNT_NUMBER}
dhl.express.baseUrl=https://express.api.dhl.com/mydhlapi
dhl.express.testBaseUrl=https://express.api.dhl.com/mydhlapi/test
dhl.express.trackingUrl=https://www.dhl.com/express/track?AWB=
```

#### **deploy/germany/service.yaml**
```yaml
env:
  - name: SHIPPING_PROVIDER
    value: "DHL"
  - name: DHL_API_KEY
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-germany
        key: apiKey
  - name: DHL_API_SECRET
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-germany
        key: apiSecret
  - name: DHL_ACCOUNT_NUMBER
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-germany
        key: accountNumber
```

### **8.3 MongoDB Configuration Documents**

```javascript
// DHL Receiver Address (HP Return Center in Germany)
db.returnConfig.insertOne({
  "key": "dhlReceiverAddress_DE",
  "value": {
    "postalAddress": {
      "cityName": "Bonn",
      "countryCode": "DE",
      "postalCode": "53113",
      "addressLine1": "Charles-de-Gaulle-Str. 20"
    },
    "contactInformation": {
      "companyName": "HP Deutschland GmbH",
      "fullName": "HP Returns Department",
      "email": "returns.de@hp.com",
      "phone": "+4922899360"
    }
  }
});

// DHL Product Code Config
db.returnConfig.insertOne({
  "key": "dhlProductCode_DE_DOMESTIC",
  "value": "N"  // EXPRESS DOMESTIC
});

db.returnConfig.insertOne({
  "key": "dhlIncoterm",
  "value": "DAP"  // Delivered At Place
});
```

### **8.4 Item Model Updates (Optional but Recommended)**

```java
@Document(collection = "returnDetails")
public class Item {
    // Existing fields
    private String sku;
    private int quantity;
    private ReturnState status;
    private String returnReason;
    private String serialNumber;
    private LocalDateTime timeStamp;
    private boolean waybillGenerated;
    
    // NEW fields for DHL
    private Double weight;  // in kg
    private Dimensions dimensions;  // L x W x H in cm
}

@Data
public class Dimensions {
    private Integer length;
    private Integer width;
    private Integer height;
}
```

---

## Summary of Critical Gaps

### **🔴 HIGH Priority Gaps**

1. **Package Weight & Dimensions**
   - Status: ❌ NOT in current Item model
   - Impact: Cannot create DHL shipment without this
   - Solution: Add fields OR use defaults OR fetch from catalog

2. **Full HP Receiver Address**
   - Status: ⚠️ Only have addressId lookup
   - Impact: DHL requires full address object
   - Solution: Create config with complete address

3. **Customer Phone Number**
   - Status: ❌ NOT captured in current flow
   - Impact: Required by DHL
   - Solution: Add to user model OR make optional

4. **DHL Product Code Determination**
   - Status: ❌ No logic exists
   - Impact: Required field
   - Solution: Add determination logic based on origin/destination

5. **S4 Carrier Field Validation**
   - Status: ⚠️ Unknown if "DHL" accepted
   - Impact: S4 integration may fail
   - Solution: Validate with S4 team

### **🟡 MEDIUM Priority Gaps**

6. **Incoterm Configuration**
   - Status: ❌ Not configured
   - Impact: Required for DHL
   - Solution: Add config (likely "DAP")

7. **Customs Declaration Logic**
   - Status: ❌ No logic exists
   - Impact: Required for international returns
   - Solution: Add country-based determination

8. **Pickup Request Logic**
   - Status: ❌ Not implemented
   - Impact: May be required for some regions
   - Solution: Add business logic

### **🟢 LOW Priority Gaps**

9. **QR Code / Label-Free Service**
   - Status: ❌ Not implemented
   - Impact: Optional feature
   - Solution: Implement if needed

10. **Multiple Package Handling**
    - Status: ⚠️ Current model is item-based
    - Impact: DHL model is package-based
    - Solution: Aggregate items into packages

---

## Next Steps & Action Items

### **Phase 1: Validation & Preparation (Week 1)**
1. ✅ **Request DHL API Credentials**
   - Submit account info to DHL consultant
   - Receive API Key, Secret, Account Number

2. ✅ **Validate S4 Integration**
   - Confirm `carrierName` accepts "DHL"
   - Confirm tracking number format accepted
   - Confirm tracking URL field exists

3. ✅ **Add Weight/Dimensions Fields**
   - Update Item model
   - Update database schema
   - Update UI/API to capture data

4. ✅ **Create DHL Receiver Address Config**
   - Get official HP Germany return center address
   - Add to MongoDB returnConfig

### **Phase 2: Core Implementation (Weeks 2-3)**
5. ✅ **Create DHL Client Classes**
   - DhlConfig.java
   - DhlClient.java
   - DhlShipmentRequest/Response DTOs
   - InitReturnDhlFlow.java

6. ✅ **Implement Field Mapping**
   - Map RMA to customerReferences
   - Map customer to shipperDetails
   - Map HP address to receiverDetails
   - Map items to packages

7. ✅ **Add Product Code Determination Logic**
   - Based on origin/destination country
   - Configuration per region

8. ✅ **Update LaunchDarkly Configuration**
   - Add "DHL" option
   - Configure per region

### **Phase 3: Testing (Week 4)**
9. ✅ **Unit Tests**
   - DhlClient tests
   - InitReturnDhlFlow tests
   - Field mapping tests

10. ✅ **Integration Tests with DHL Sandbox**
    - Test shipment creation
    - Test tracking retrieval
    - Test error handling

11. ✅ **S4 Integration Testing**
    - Test with DHL carrier name
    - Validate payload acceptance
    - Test end-to-end flow

### **Phase 4: Deployment (Week 5)**
12. ✅ **Deploy to DEV**
    - Configure secrets
    - Test with real credentials
    - Validate flows

13. ✅ **Deploy to STG**
    - Smoke testing
    - Performance testing

14. ✅ **Deploy to PROD (Germany)**
    - Phased rollout
    - Monitor metrics
    - Rollback plan ready

---

## Appendix A: DHL Product Codes Reference

From provided image - **Global Product Codes:**

### **Most Used DHL Express Products**

| Code | Name | Content Code | Doc/Non-Doc | Transport Type |
|------|------|--------------|-------------|----------------|
| **N** | EXPRESS DOMESTIC | DOM | Y-Doc | Land within country |
| **W** | ECONOMY SELECT | ESU | Y-Doc | Land within EU |
| **H** | ECONOMY SELECT | ESI | N-Non Doc | Land outside EU |
| **U** | EXPRESS WORLDWIDE | ECX | Y-Doc | Air within EU |
| **P** | EXPRESS WORLDWIDE | WPX | N-Non Doc | Air outside EU |

### **Other Air Transport Products**

| Code | Name | Content Code | Doc/Non-Doc | Transport Type |
|------|------|--------------|-------------|----------------|
| D | EXPRESS WORLDWIDE | DOX | Y-Doc | Air within EU (documents) |
| K | EXPRESS 9:00 | TDK | Y-Doc | Air within EU |
| E | EXPRESS 9:00 | TDE | N-Non Doc | Air outside EU |
| L | EXPRESS 10:30 | TDL | Y-Doc | Air within EU |
| M | EXPRESS 10:30 | TDM | N-Non Doc | Air outside EU |
| T | EXPRESS 12:00 | TDT | Y-Doc | Air within EU |
| Y | EXPRESS 12:00 | TDY | N-Non Doc | Air outside EU |

### **Other Land Transport Products**

| Code | Name | Content Code | Doc/Non-Doc | Transport Type |
|------|------|--------------|-------------|----------------|
| I | EXPRESS DOMESTIC 9:00 | DOK | Y-Doc | Land within country |
| O | EXPRESS DOMESTIC 10:30 | DOL | Y-Doc | Land within country |
| 1 | EXPRESS DOMESTIC 12:00 | DOT | Y-Doc | Land within country |

### **Other Products**

| Code | Name | Content Code | Doc/Non-Doc | Transport Type |
|------|------|--------------|-------------|----------------|
| B | EXPRESS BREAKBULK | BBX | Y-Doc | Air within EU |
| 7 | EXPRESS EASY | XED | Y-Doc | Air within EU |
| 8 | EXPRESS EASY | XEP | N-Non Doc | Air outside EU |
| C | MEDICAL EXPRESS | CMX | Y-Doc | Air within EU |
| Q | MEDICAL EXPRESS | WMX | N-Non Doc | Air outside EU |

---

## Appendix B: Country Postcode Formats

From provided image - sample formats:

| Country Code | Country Name | Postcode Format | Significant Figures |
|--------------|--------------|-----------------|---------------------|
| DE | GERMANY | 99999 | 5 |
| US | UNITED STATES | 99999 | 5 |
| GB | UNITED KINGDOM | AA99 9AA | Variable |
| FR | FRANCE | 99999 | 5 |
| IT | ITALY | 99999 | 5 |
| ES | SPAIN | 99999 | 5 |
| NL | NETHERLANDS | 9999 AA | 6 |
| BE | BELGIUM | 9999 | 4 |
| AT | AUSTRIA | 9999 | 4 |
| CZ | CZECH REPUBLIC | 999 99 | 6 |
| CA | CANADA | A9A 9A9 | 7 |

---

## Document Change Log

| Date | Author | Changes |
|------|--------|---------|
| Aug 13, 2026 | Anjali Taluri | Initial comprehensive analysis |

---

**End of Document**

