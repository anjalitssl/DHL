# FedEx vs DHL Express Integration Comparison
**Shipping Provider Implementation Guide**

**Service:** STRATUS Returns Service  
**Date:** August 13, 2026

---

## 📋 Executive Summary

This document compares the **current FedEx implementation** with the **proposed DHL Express integration** for the STRATUS Returns Service. Both providers handle return shipment label generation, but with different API patterns and authentication mechanisms.

---

## 🔑 Authentication Comparison

### **FedEx (Current Implementation)**

#### **Type:** OAuth2 Password Grant Flow

```java
// Token Management Required
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

**Implementation:**
```java
// FedexTokenHolder.java - Manages token lifecycle
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

// FedexClient.java - Use token in requests
HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Bearer " + token);
```

**Credentials Required:**
- ✅ `FEDEX_CLIENT_ID`
- ✅ `FEDEX_CLIENT_SECRET`
- ✅ `FEDEX_ORG_NAME` (username)
- ✅ `FEDEX_PASSWORD`
- ✅ `FEDEX_XORG_NAME` (org identifier)

**Pros:**
- Secure token-based authentication
- Token can be reused for multiple requests
- Standard OAuth2 flow

**Cons:**
- Requires token management/refresh logic
- Additional complexity (FedexTokenHolder component)
- Token endpoint adds extra API call

---

### **DHL Express (Proposed Implementation)**

#### **Type:** HTTP Basic Authentication

```java
// No token management needed - authenticate per request
String auth = apiKey + ":" + apiSecret;
String encodedAuth = Base64.getEncoder()
    .encodeToString(auth.getBytes(StandardCharsets.UTF_8));

HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Basic " + encodedAuth);
headers.set("Content-Type", "application/json");
```

**Credentials Required:**
- ✅ `DHL_API_KEY` (username)
- ✅ `DHL_API_SECRET` (password)
- ✅ `DHL_ACCOUNT_NUMBER`

**Pros:**
- ✅ Simpler implementation (no token holder needed)
- ✅ No token expiry management
- ✅ Stateless authentication
- ✅ One less API call per operation

**Cons:**
- Credentials sent with every request (still secure over HTTPS)
- Base64 encoding required per request (negligible overhead)

---

## 🚀 API Endpoint Comparison

### **FedEx - RMA Creation**

**Endpoint:**
```
POST https://connect.supplychain.fedex.com/api/v2/rmas
```

**Purpose:**  
Dedicated **Return Merchandise Authorization** API specifically designed for returns.

**Request Payload:**
```json
{
  "rmaNumber": "RET-20260813-001",
  "customer": {
    "emailAddress": "customer@example.com",
    "firstName": "John",
    "lastName": "Doe"
  },
  "shipToAddressId": 12345,
  "orders": [
    {
      "orderNumber": "FO-123456",
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

**Response:**
```json
{
  "success": "true",
  "rma": {
    "shipments": [
      {
        "label": {
          "trackingNumber": "738979304299",
          "labelData": "JVBERi0xLjQK..." // Base64 PDF
        },
        "orders": [...]
      }
    ]
  },
  "requestIdentifier": "req-abc123",
  "transactionDate": "2026-08-13T10:30:00Z"
}
```

**Key Features:**
- ✅ RMA-specific endpoint
- ✅ Uses address lookup by `shipToAddressId` (configured in FedEx portal)
- ✅ Simplified customer object
- ✅ Return reason built into API

**Implementation:**
```java
// FedexClient.java
@Retryable(retryFor = HttpServerErrorException.class)
public ReturnDetailsDocument createRmaS4(ReturnDetailsDocument doc) {
    FedexPayload payload = new FedexPayload();
    payload.setRmaNumber(doc.getRetReturnNumber());
    payload.setCustomer(buildCustomer(doc));
    payload.setShipToAddressId(configuredAddressId);
    payload.setOrders(buildOrders(doc));
    
    return restTemplate.postForObject(
        fedexConfig.getCreateRmaUrl(), 
        payload, 
        ReturnDetailsDocument.class
    );
}
```

---

### **DHL Express - Shipment Creation**

**Endpoint:**
```
POST https://express.api.dhl.com/mydhlapi/shipments
```

**Purpose:**  
Generic **shipment creation** API. Returns are handled as regular shipments with appropriate customer references.

**Request Payload:**
```json
{
  "plannedShippingDateAndTime": "2026-08-13T14:00:00GMT+01:00",
  "pickup": {
    "isRequested": true
  },
  "productCode": "P",
  "accounts": [
    {
      "typeCode": "shipper",
      "number": "123456789"
    }
  ],
  "customerDetails": {
    "shipperDetails": {
      "postalAddress": {
        "cityName": "Austin",
        "countryCode": "US",
        "postalCode": "78701",
        "addressLine1": "123 Main St"
      },
      "contactInformation": {
        "email": "customer@example.com",
        "phone": "+1234567890",
        "fullName": "John Doe"
      }
    },
    "receiverDetails": {
      "postalAddress": {
        "cityName": "Memphis",
        "countryCode": "US",
        "postalCode": "38101",
        "addressLine1": "DHL Return Center"
      },
      "contactInformation": {
        "email": "returns@hp.com",
        "phone": "+1987654321",
        "fullName": "HP Returns"
      }
    }
  },
  "content": {
    "packages": [
      {
        "weight": 2.5,
        "dimensions": {
          "length": 25,
          "width": 20,
          "height": 15
        }
      }
    ],
    "isCustomsDeclarable": true,
    "declaredValue": 299.99,
    "declaredValueCurrency": "USD",
    "description": "HP Printer Return - Damaged",
    "incoterm": "DAP"
  },
  "outputImageProperties": {
    "imageOptions": [
      {
        "typeCode": "label",
        "isRequested": true
      }
    ],
    "encodingFormat": "pdf"
  },
  "customerReferences": [
    {
      "value": "RET-20260813-001",
      "typeCode": "CU"
    }
  ]
}
```

**Response:**
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

**Key Features:**
- ✅ More detailed address requirements (full address, not lookup ID)
- ✅ Flexible customer references for RMA number
- ✅ Supports multiple packages per shipment
- ✅ Customs declaration built-in
- ✅ Return reason goes in `content.description`

**Proposed Implementation:**
```java
// DhlClient.java
@Retryable(retryFor = HttpServerErrorException.class)
public ReturnDetailsDocument createShipment(ReturnDetailsDocument doc) {
    DhlShipmentRequest request = new DhlShipmentRequest();
    request.setCustomerDetails(buildCustomerDetails(doc));
    request.setContent(buildContent(doc));
    request.setCustomerReferences(buildReferences(doc));
    
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", "Basic " + getBasicAuth());
    
    HttpEntity<DhlShipmentRequest> entity = 
        new HttpEntity<>(request, headers);
    
    return restTemplate.postForObject(
        dhlConfig.getBaseUrl() + "/shipments",
        entity,
        DhlShipmentResponse.class
    );
}
```

---

## 📦 Data Mapping Comparison

| Field | FedEx Structure | DHL Structure |
|-------|-----------------|---------------|
| **RMA/Return ID** | `rmaNumber` (root level) | `customerReferences[typeCode='CU'].value` |
| **Customer Email** | `customer.emailAddress` | `customerDetails.shipperDetails.contactInformation.email` |
| **Customer Name** | `customer.firstName`, `customer.lastName` | `customerDetails.shipperDetails.contactInformation.fullName` |
| **HP Return Address** | `shipToAddressId` (lookup) | `customerDetails.receiverDetails` (full object) |
| **Order ID** | `orders[].orderNumber` | Passed as custom reference |
| **Items** | `orders[].items[]` (SKU, quantity) | `content.packages[]` (weight, dimensions) |
| **Return Reason** | `returnItemInfo.returnReason` | `content.description` |
| **Tracking Number** | `rma.shipments[].label.trackingNumber` | `shipmentTrackingNumber` |
| **Label PDF** | `rma.shipments[].label.labelData` | `documents[].content` |

---

## 🔍 Tracking Comparison

### **FedEx Tracking**

**Tracking Number Format:** Numeric, 12 digits  
**Example:** `738979304299`

**Tracking URL:**
```
https://www.fedex.com/fedextrack/?tracknumbers=738979304299
```

**API Endpoint:**  
❌ Not currently used (tracking handled by S4/GEKKO events)

**Implementation:**
```java
// Stored in response
payload.setTrackingUrl(
    fedexConfig.getTrackingUrl() + trackingNumber
);
```

---

### **DHL Express Tracking**

**Tracking Number Format:** Alphanumeric, 10 digits  
**Example:** `1234567890` or `JD999999999999999999` (package level)

**Tracking URL:**
```
https://www.dhl.com/express/track?AWB=1234567890
```

**API Endpoint:**
```
GET https://express.api.dhl.com/mydhlapi/shipments/{trackingNumber}/tracking
```

**Proposed Implementation:**
```java
// DhlClient.java
public TrackingInfo getTracking(String trackingNumber) {
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", "Basic " + getBasicAuth());
    
    HttpEntity<?> entity = new HttpEntity<>(headers);
    
    return restTemplate.exchange(
        dhlConfig.getBaseUrl() + "/shipments/" + trackingNumber + "/tracking",
        HttpMethod.GET,
        entity,
        DhlTrackingResponse.class
    ).getBody();
}

// Store in return document
payload.setTrackingUrl(
    "https://www.dhl.com/express/track?AWB=" + trackingNumber
);
```

---

## 🏗️ Architecture Comparison

### **FedEx Implementation (Current)**

```
┌─────────────────────────────────────────────────────┐
│ ReturnDetailsController                              │
│   POST /v1/internal/initiate                        │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ ReturnDetailsService                                 │
│   - sendReturn()                                    │
│   - resolveInitReturnFlow() ◄─── LaunchDarkly flag │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ InitReturnFedexFlow (implements InitReturnFlow)    │
│   - getFlowType() → "Fedex"                         │
│   - initReturn()                                    │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ FedexClient                                          │
│   - createRma()                                     │
│   - createRmaS4()                                   │
│   - fedexCall()                                     │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ FedexTokenHolder                                     │
│   - getFedExAuthToken()                             │
│   - refreshToken()                                  │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
     ┌──────────────────────┐
     │ FedEx RMA API        │
     │ OAuth + REST         │
     └──────────────────────┘
```

**Components:**
- `FedexConfig.java` - Properties
- `FedexClient.java` - API client
- `FedexTokenHolder.java` - Token manager
- `FedexPayload.java` - Request DTO
- `InitReturnFedexFlow.java` - Flow implementation
- `FedexException` classes - Error handling

---

### **DHL Implementation (Proposed)**

```
┌─────────────────────────────────────────────────────┐
│ ReturnDetailsController                              │
│   POST /v1/internal/initiate                        │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ ReturnDetailsService                                 │
│   - sendReturn()                                    │
│   - resolveInitReturnFlow() ◄─── LaunchDarkly flag │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ InitReturnDhlFlow (implements InitReturnFlow)      │
│   - getFlowType() → "DHL"                           │
│   - initReturn()                                    │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ DhlClient                                            │
│   - createShipment()                                │
│   - getTracking()                                   │
└────────────────┬────────────────────────────────────┘
                 │
                 │ (No token holder needed!)
                 │
                 ▼
     ┌──────────────────────┐
     │ DHL Express API      │
     │ Basic Auth + REST    │
     └──────────────────────┘
```

**Components to Create:**
- ✅ `DhlConfig.java` - Properties
- ✅ `DhlClient.java` - API client (NO token holder)
- ✅ `DhlShipmentRequest.java` - Request DTO
- ✅ `DhlShipmentResponse.java` - Response DTO
- ✅ `InitReturnDhlFlow.java` - Flow implementation
- ✅ `DhlException` classes - Error handling

**Key Difference:**  
❌ **NO** `DhlTokenHolder` needed → simpler architecture!

---

## 🔒 Security Comparison

| Aspect | FedEx | DHL |
|--------|-------|-----|
| **Credentials in Transit** | Token (OAuth) | Base64 Basic Auth |
| **Credentials at Rest** | K8s Secrets | K8s Secrets |
| **Token Lifetime** | 1 hour (refresh needed) | N/A (stateless) |
| **Token Storage** | In-memory (FedexTokenHolder) | Not needed |
| **HTTPS Required** | Yes | Yes |
| **Credential Rotation** | Supported | Supported |

**Both are secure** over HTTPS. DHL's approach is simpler but equally secure.

---

## ⚙️ Configuration Comparison

### **FedEx Environment Variables**

```yaml
# deploy/stg-west/service.yaml (current)
env:
  - name: FEDEX_XORG_NAME
    valueFrom:
      secretKeyRef:
        name: fedex-credentials-stratus-returns-stg-us-west-2
        key: xOrgName
  - name: FEDEX_ORG_NAME
    valueFrom:
      secretKeyRef:
        name: fedex-credentials-stratus-returns-stg-us-west-2
        key: orgName
  - name: FEDEX_PASSWORD
    valueFrom:
      secretKeyRef:
        name: fedex-credentials-stratus-returns-stg-us-west-2
        key: password
  - name: FEDEX_CLIENT_ID
    valueFrom:
      secretKeyRef:
        name: fedex-credentials-stratus-returns-stg-us-west-2
        key: clientId
  - name: FEDEX_CLIENT_SECRET
    valueFrom:
      secretKeyRef:
        name: fedex-credentials-stratus-returns-stg-us-west-2
        key: clientSecret
```

**application.properties:**
```properties
fedex.contentType=application/x-www-form-urlencoded
fedex.xOrgName=${FEDEX_XORG_NAME}
fedex.orgName=${FEDEX_ORG_NAME}
fedex.accessTokenUri=https://connect.supplychain.fedex.com/api/fsc/oauth2/token
fedex.grantType=password
fedex.scope=Fulfillment_Returns
fedex.password=${FEDEX_PASSWORD}
fedex.clientId=${FEDEX_CLIENT_ID}
fedex.clientSecret=${FEDEX_CLIENT_SECRET}
fedex.trackingUrl=https://www.fedex.com/fedextrack/?tracknumbers=
fedex.createRmaUrl=https://connect.supplychain.fedex.com/api/v2/rmas
```

---

### **DHL Environment Variables (Proposed)**

```yaml
# deploy/europe/service.yaml (new)
env:
  - name: DHL_API_KEY
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-stratus-returns-europe
        key: apiKey
  - name: DHL_API_SECRET
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-stratus-returns-europe
        key: apiSecret
  - name: DHL_ACCOUNT_NUMBER
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-stratus-returns-europe
        key: accountNumber
```

**application.properties:**
```properties
dhl.express.apiKey=${DHL_API_KEY}
dhl.express.apiSecret=${DHL_API_SECRET}
dhl.express.accountNumber=${DHL_ACCOUNT_NUMBER}
dhl.express.baseUrl=https://express.api.dhl.com/mydhlapi
dhl.express.testBaseUrl=https://express.api.dhl.com/mydhlapi/test
dhl.express.trackingUrl=https://www.dhl.com/express/track?AWB=
```

**Simpler:** Only 3 credentials vs 5 for FedEx!

---

## 🧪 Testing Comparison

### **FedEx Testing**

**Sandbox Details:**
- No public sandbox documented in codebase
- Uses production credentials with test data
- Attribution flags used for prod testing bypass

**Current Test Approach:**
```java
// InitReturnFedexFlowTest.java
@Test
void testInitReturnSuccess() {
    when(fedexClient.createRma(any(), any()))
        .thenReturn(mockResponse);
    when(fedexConfig.getTrackingUrl())
        .thenReturn("test");
    
    ReturnDetailsDocument result = 
        initReturnFedexFlow.initReturn(doc);
    
    assertEquals("true", result.getSuccess());
}
```

---

### **DHL Testing (Proposed)**

**Sandbox Environment:**
```
https://express.api.dhl.com/mydhlapi/test
```

**Test Strategy:**
```java
// InitReturnDhlFlowTest.java
@Test
void testInitReturnSuccess() {
    when(dhlClient.createShipment(any()))
        .thenReturn(mockDhlResponse);
    when(dhlConfig.getTrackingUrl())
        .thenReturn("https://www.dhl.com/express/track?AWB=");
    
    ReturnDetailsDocument result = 
        initReturnDhlFlow.initReturn(doc);
    
    assertNotNull(result.getRma().getShipments());
    assertTrue(result.getTrackingNumber().length() == 10);
}
```

**DHL provides better sandbox support!**

---

## 📊 Feature Matrix

| Feature | FedEx | DHL | Notes |
|---------|-------|-----|-------|
| **RMA Creation** | ✅ Dedicated API | ✅ Via Shipment API | DHL uses generic endpoint |
| **Tracking API** | ⚠️ Available (not used) | ✅ Available | Can implement polling |
| **Label Generation** | ✅ PDF/ZPL | ✅ PDF/ZPL/PNG | DHL has more formats |
| **Pickup Booking** | ✅ Included in RMA | ✅ Separate or in request | DHL more flexible |
| **Return Reason** | ✅ Structured field | ⚠️ Free text | FedEx more structured |
| **Address Lookup** | ✅ Via shipToAddressId | ❌ Full address required | FedEx simpler |
| **Multi-Package** | ✅ Supported | ✅ Supported | Both support |
| **Customs Declaration** | ⚠️ Limited | ✅ Full support | DHL better for intl |
| **Label-Free/QR Code** | ❌ Not supported | ✅ Supported | DHL modern feature |
| **Rate Shopping** | ❌ RMA endpoint only | ✅ Separate /rates endpoint | DHL can quote first |

---

## 🌍 Regional Support

### **FedEx**
- ✅ **US Region:** Primary use case (currently implemented)
- ✅ **International:** Supported but not actively used
- Return center: Memphis, TN

### **DHL Express**
- ✅ **Europe:** Strong presence (target for new integration)
- ✅ **APAC:** Available
- ✅ **Americas:** Available but FedEx preferred
- Multiple return centers per region

---

## 💰 Cost Considerations

*Note: Actual costs not in scope of this technical comparison*

**FedEx:**
- Per-RMA charges
- Billed to FedEx account directly

**DHL:**
- Per-shipment charges
- Billed to DHL Express account
- Potential volume discounts for enterprise customers

---

## 🚦 Migration Strategy

### **Option 1: Region-Based Routing** *(Recommended)*

```java
// LaunchDarkly configuration
{
  "shipment-provider": {
    "US": "Fedex",
    "EU": "DHL",
    "APAC": "DHL",
    "default": "Fedex"
  }
}
```

**Implementation:**
- Keep FedEx for US customers
- Add DHL for EU/APAC customers
- LaunchDarkly flag determines routing per region

---

### **Option 2: Account-Based Routing**

```java
// Based on customer account configuration
if (account.getPreferredCarrier() == "DHL") {
    return InitReturnDhlFlow;
} else {
    return InitReturnFedexFlow;
}
```

---

### **Option 3: Phased Migration**

1. **Phase 1:** DHL in test environments only
2. **Phase 2:** DHL for EU pilot customers
3. **Phase 3:** Full EU migration to DHL
4. **Phase 4:** APAC migration
5. **Future:** Keep both based on strategic needs

---

## ✅ Implementation Effort Estimate

### **FedEx (Reference - Already Done)**

| Component | Effort | Complexity |
|-----------|--------|------------|
| Configuration | 1 day | Low |
| Token Management | 3 days | Medium |
| API Client | 5 days | Medium |
| Flow Implementation | 2 days | Low |
| Testing | 3 days | Medium |
| **Total** | **~14 days** | **Medium** |

---

### **DHL (Proposed)**

| Component | Effort | Complexity | Notes |
|-----------|--------|------------|-------|
| Configuration | 1 day | Low | Simpler than FedEx |
| ~~Token Management~~ | 0 days | N/A | **Not needed!** |
| API Client | 4 days | Medium | More complex payload |
| Flow Implementation | 2 days | Low | Similar to FedEx |
| Data Mapping Layer | 3 days | Medium | More fields to map |
| Testing | 3 days | Medium | Sandbox available |
| Documentation | 1 day | Low | |
| **Total** | **~14 days** | **Medium** | Similar effort |

**Key Insight:** Despite different auth, total effort is comparable. DHL saves time on token management but needs more work on data mapping.

---

## 🔑 Key Takeaways

### **Use FedEx When:**
- ✅ US-based returns
- ✅ Simpler address management needed (address lookup by ID)
- ✅ Integrated with existing FedEx logistics

### **Use DHL When:**
- ✅ European/APAC returns
- ✅ International customs requirements
- ✅ Label-Free/QR code capabilities needed
- ✅ Simpler authentication preferred

### **Both Support:**
- ✅ Return label generation
- ✅ Tracking numbers
- ✅ PDF labels
- ✅ Multi-package shipments
- ✅ Retry logic
- ✅ Error handling

---

## 📝 Decision Log

**Date:** August 13, 2026

**Decision:** Implement DHL Express integration for EU region

**Rationale:**
1. FedEx stronger in US; DHL stronger in EU
2. Customer demand for European return processing
3. DHL API simpler authentication model
4. Both APIs mature and well-documented
5. Multi-vendor support already architected (InitReturnFlow interface)

**Next Steps:**
1. Request DHL Express API credentials
2. Implement `InitReturnDhlFlow` and `DhlClient`
3. Test in sandbox environment
4. Deploy to EU staging
5. Monitor and iterate

---

## 📚 References

**FedEx:**
- API Docs: Internal FedEx Supply Chain portal
- Config: `application.properties` (lines 163-176)
- Implementation: `com.hp.tropos.clients.fedex.*`

**DHL Express:**
- API Docs: https://developer.dhl.com/api-reference/dhl-express-mydhl-api
- Version: MyDHL API v3.3.1
- Implementation: To be created in `com.hp.tropos.clients.dhl.*`

**Service Architecture:**
- Interface: `com.hp.tropos.service.InitReturnFlow`
- Service: `com.hp.tropos.service.ReturnDetailsService`
- Controller: `com.hp.tropos.api.ReturnDetailsController`

---

**Document Owner:** Returns Team  
**Last Updated:** August 13, 2026  
**Review Cycle:** Quarterly or when adding new shipping providers

