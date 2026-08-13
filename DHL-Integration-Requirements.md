# DHL Express Integration Requirements
**Meeting Notes & Implementation Guide**

**Date:** August 13, 2026  
**Service:** STRATUS Returns Service  
**Region:** Global (US, Europe, APAC)  
**API:** MyDHL API v3.3.1

---

## 🎯 Meeting Takeaways - Key Requirements

Based on the meeting discussion, DHL Express integration requires implementation of **three core capabilities**:

### 1️⃣ **Tracking** 
### 2️⃣ **Token/Authentication**
### 3️⃣ **RMA (Return Merchandise Authorization)**

---

## 📋 Detailed Requirements

### 1. **Tracking** 🔍

#### **What DHL Provides:**
- **API Endpoint:** `GET /shipments/{shipmentTrackingNumber}/tracking`
- **Tracking URL Template:** `https://www.dhl.com/express/track?AWB={trackingNumber}`
- **Real-time Status Updates:** Via API polling or webhook

#### **What We Need to Implement:**
```java
// Store tracking information
returnDetailsDocument.setTrackingNumber(dhlResponse.getTrackingNumber());
returnDetailsDocument.setTrackingUrl("https://www.dhl.com/express/track?AWB=" + trackingNumber);
returnDetailsDocument.setShippingVendor("DHL");
```

#### **Data Flow:**
1. Create DHL shipment → Receive `shipmentTrackingNumber`
2. Store tracking number in MongoDB (`ReturnDetailsDocument.rma.shipments[].label.trackingNumber`)
3. Provide tracking URL to customer via API response
4. (Optional) Poll DHL Tracking API for status updates

#### **Response Structure:**
```json
{
  "shipments": [{
    "id": "1234567890",
    "status": {
      "statusCode": "delivered",
      "timestamp": "2026-08-13T10:30:00"
    },
    "events": [...]
  }]
}
```

---

### 2. **Token/Authentication** 🔐

#### **DHL Authentication Model:**
- **Type:** HTTP Basic Authentication (NOT OAuth like FedEx!)
- **Credentials Required:**
  - DHL Account Number
  - API Key (Username)
  - API Secret (Password)

#### **How to Obtain Credentials:**
1. Submit account information to DHL Express consultant:
   - Customer DHL Express Account(s)
   - Contact Name
   - Email
   - Phone
   - Address (Line, City, Postcode)
2. DHL will generate and email API credentials
3. Test in sandbox environment first
4. Move to production after validation

#### **Configuration Required:**
```properties
# application.properties
dhl.express.apiKey=${DHL_API_KEY}
dhl.express.apiSecret=${DHL_API_SECRET}
dhl.express.accountNumber=${DHL_ACCOUNT_NUMBER}
dhl.express.baseUrl=https://express.api.dhl.com/mydhlapi
dhl.express.testBaseUrl=https://express.api.dhl.com/mydhlapi/test
dhl.express.trackingUrl=https://www.dhl.com/express/track?AWB=
```

#### **Implementation Pattern:**
```java
// NO token holder/refresh needed - use Basic Auth per request
HttpHeaders headers = new HttpHeaders();
String auth = apiKey + ":" + apiSecret;
String encodedAuth = Base64.getEncoder().encodeToString(auth.getBytes(StandardCharsets.UTF_8));
headers.set("Authorization", "Basic " + encodedAuth);
headers.set("Content-Type", "application/json");
```

#### **Key Differences from FedEx:**
| Aspect | FedEx (Current) | DHL Express (New) |
|--------|-----------------|-------------------|
| **Auth Type** | OAuth2 Password Grant | HTTP Basic Auth |
| **Token Endpoint** | `/oauth2/token` | N/A |
| **Token Refresh** | Required (FedexTokenHolder) | Not needed |
| **Credentials** | ClientId, ClientSecret, Username, Password | API Key + Secret |
| **Header Format** | `Bearer <token>` | `Basic <base64(key:secret)>` |

---

### 3. **RMA (Return Merchandise Authorization)** 📦

#### **What DHL Calls It:**
- DHL doesn't use the term "RMA" - they call it **"Shipment Creation"**
- RMA concept is handled through **customer references** in the shipment

#### **API Endpoint:**
```
POST https://express.api.dhl.com/mydhlapi/shipments
```

#### **Request Structure:**
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
    "description": "HP Printer Return",
    "incoterm": "DAP"
  },
  "outputImageProperties": {
    "imageOptions": [
      {
        "typeCode": "label",
        "templateName": "ECOM26_A4_001",
        "isRequested": true
      }
    ],
    "splitTransportAndWaybillDocLabels": false,
    "allDocumentsInOneImage": false,
    "splitDocumentsByPages": false,
    "splitInvoiceAndReceipt": false,
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

#### **Key Mappings:**

| Our Concept | DHL Field Path | Notes |
|-------------|----------------|-------|
| **RMA Number** | `customerReferences[typeCode='CU'].value` | Our `returnOrderId` |
| **Return Reason** | `content.description` | Text field |
| **Customer Email** | `customerDetails.shipperDetails.contactInformation.email` | Sender (customer) |
| **HP Address** | `customerDetails.receiverDetails` | Return destination |
| **Tracking Number** | Response: `shipmentTrackingNumber` | Returned by DHL |
| **Label PDF** | Response: `documents[].content` | Base64 encoded |

#### **Response Structure:**
```json
{
  "shipmentTrackingNumber": "1234567890",
  "trackingUrl": "https://mydhl.express.dhl/us/en/tracking.html#/details/1234567890",
  "packages": [
    {
      "referenceNumber": 1,
      "trackingNumber": "JD999999999999999999",
      "trackingUrl": "..."
    }
  ],
  "documents": [
    {
      "imageFormat": "PDF",
      "content": "JVBERi0xLjQKJ..." // Base64 encoded label
    }
  ],
  "shipmentDetails": [
    {
      "productCode": "P",
      "localProductCode": "P",
      "serviceHandlingFeatureCodes": ["AA"]
    }
  ]
}
```

---

## 🔧 Implementation Checklist

### **Phase 1: Configuration & Setup**
- [ ] Request DHL Express API credentials (submit account info to consultant)
- [ ] Receive API Key, Secret, and Account Number
- [ ] Add environment variables to deployment configs (`deploy/*/service.yaml`)
- [ ] Create `DhlConfig.java` properties class
- [ ] Test connectivity in sandbox environment

### **Phase 2: Core Implementation**
- [ ] Create `com.hp.tropos.clients.dhl.DhlClient.java`
  - [ ] Implement `createShipment()` method
  - [ ] Implement Basic Auth header construction
  - [ ] Handle DHL API response mapping
  - [ ] Add retry logic with `@Retryable`
- [ ] Create `com.hp.tropos.service.dhl.InitReturnDhlFlow.java`
  - [ ] Implement `InitReturnFlow` interface
  - [ ] Set `getFlowType()` to return `"DHL"`
  - [ ] Map internal model to DHL request format
- [ ] Create `DhlPayload.java` DTO model
- [ ] Add DHL-specific exceptions

### **Phase 3: Integration Points**
- [ ] Update `ReturnDetailsService.resolveInitReturnFlow()`
  - Already supports dynamic routing via LaunchDarkly flag
- [ ] Add LaunchDarkly flag value: `shipment-provider` = `"DHL"`
- [ ] Configure region-specific routing (US → FedEx, EU → DHL, etc.)
- [ ] Store `shippingVendor = "DHL"` in return document

### **Phase 4: Data Mapping**
- [ ] Map `returnOrderId` → `customerReferences[].value`
- [ ] Map customer details → `shipperDetails`
- [ ] Map HP return center address → `receiverDetails`
- [ ] Map items to packages (weight, dimensions, value)
- [ ] Handle customs declaration for international returns

### **Phase 5: Label & Tracking**
- [ ] Parse DHL response and extract `shipmentTrackingNumber`
- [ ] Store tracking number in `ReturnDetailsDocument.rma.shipments[].label.trackingNumber`
- [ ] Construct tracking URL: `https://www.dhl.com/express/track?AWB={trackingNumber}`
- [ ] Store Base64 label in response (if needed for display/download)

### **Phase 6: Testing**
- [ ] Unit tests for `DhlClient` and `InitReturnDhlFlow`
- [ ] Integration tests with DHL sandbox
- [ ] Test payload mapping and validation
- [ ] Test error handling (invalid address, service unavailable, etc.)
- [ ] Verify tracking number retrieval and URL generation

### **Phase 7: Deployment**
- [ ] Configure secrets in Kubernetes for each environment
  - `dev`, `stg`, `stg-west`, `pro`
- [ ] Update `deploy/*/service.yaml` with DHL env vars
- [ ] Set LaunchDarkly flag per environment/region
- [ ] Monitor logs for successful DHL API calls
- [ ] Validate end-to-end return creation flow

---

## 🌍 Regional Considerations

### **Geo-Specific Configuration:**
```yaml
# deploy/us/service.yaml (existing FedEx)
env:
  - name: SHIPPING_PROVIDER
    value: "Fedex"

# deploy/europe/service.yaml (new DHL)
env:
  - name: SHIPPING_PROVIDER
    value: "DHL"
  - name: DHL_API_KEY
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-eu
        key: apiKey
  - name: DHL_API_SECRET
    valueFrom:
      secretKeyRef:
        name: dhl-credentials-eu
        key: apiSecret
```

### **Multi-Vendor Support:**
The service already supports multiple shipping providers via the **Strategy Pattern**:
- `InitReturnFedexFlow` → US region
- `InitReturnDhlFlow` → EU/APAC regions (to be implemented)
- `InitReturnNoFlow` → Fallback/manual processing

LaunchDarkly flag can be configured per:
- **Region**
- **Account**
- **Customer segment**

---

## 📞 DHL Support Contacts

**API Documentation:**  
https://developer.dhl.com/api-reference/dhl-express-mydhl-api

**Support Email:**  
From meeting: Contact your DHL Express consultant (name/email provided separately)

**Developer Portal:**  
https://developer.dhl.com/

**Sandbox Environment:**  
https://express.api.dhl.com/mydhlapi/test

**Production Environment:**  
https://express.api.dhl.com/mydhlapi

---

## ⚠️ Known Limitations

### **DHL Express MyDHL API:**
1. **No dedicated RMA endpoint** - Use standard shipment creation
2. **Label-Free (QR Code) service** - Region-specific availability (check support matrix)
3. **Pickup requirements** - May need separate pickup booking for some regions
4. **Customs documentation** - Required for international returns (non-EU to EU, etc.)
5. **Rate shopping** - Use `/rates` endpoint separately if needed before shipment creation

### **Service Limitations:**
1. FedEx uses OAuth tokens; DHL uses Basic Auth - different auth flows
2. FedEx has dedicated RMA API; DHL uses generic shipment creation
3. Payload structure significantly different - full mapping layer needed

---

## 📊 Success Metrics

**Track these after implementation:**
- DHL API success rate (target: >99%)
- Average response time for shipment creation (target: <2s)
- Label generation success rate (target: 100%)
- Tracking number retrieval accuracy (target: 100%)
- Error rate by region/account

---

## 🔗 Related Documentation

- [FedEx vs DHL Comparison](./FedEx-vs-DHL-Comparison.md)
- [MyDHL API Specification v3.3.1](../../etc/docs/mydhl-api-spec.yaml) _(to be added)_
- [Return Flow Architecture](../etc/diagrams/standard-status-sequence-diagram.puml)

---

**Last Updated:** August 13, 2026  
**Owner:** Returns Team  
**Review Date:** Quarterly

