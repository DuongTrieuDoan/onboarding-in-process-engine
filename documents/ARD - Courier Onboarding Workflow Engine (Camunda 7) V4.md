# **Architecture Requirements Document (ARD): Food Courier Onboarding Workflow Engine (V4)**

---

**Document ID:** ARD-CO-ENG-001-V4  
**Initiative:** Courier Onboarding with Process Engine (Camunda 7\)  
**Target Architecture:** Direct Microservice Invocation via SkipTheDishes Service Registry  
**Owner / Technical Lead:** Duong Doan / Onboarding Core Team  
**Status:** APPROVED FOR IMPLEMENTATION  
**Reference PRD:** PRD-CO-ENG-001-V10  
**Previous Version:** ARD-CO-ENG-001-V3

## **1\. Technical Architecture Overview**

---

This document specifies the technical integration topology, networking, and contract schemas for external task workers executing against SkipTheDishes internal staging/development backend microservices.

## **2\. Service Registry & Domain Routing Topology**

---

`┌────────────────────────────────────────────────────────────────────────┐`  
`│                      LOCAL WORKSTATION / VPN RUNTIME                   │`  
`│                                                                        │`  
`│   Camunda 7 Engine (Docker :9090)                                      │`  
`│   └── REST API: http://localhost:9090/engine-rest                      │`  
`│                                                                        │`  
`│   External Task Workers (Node.js / Quarkus Host Process)               │`  
`│   └── Long-Polling Client (/fetchAndLock)                              │`  
`└─────────────────────────────────┬──────────────────────────────────────┘`  
                                  `│ Direct HTTPS Calls over Corporate VPN`  
                                  `│ DNS: *.app.{{env}}.internal.skipthedishes.com`  
                                  `▼`  
`┌────────────────────────────────────────────────────────────────────────┐`  
`│                     DOWNSTREAM BACKEND SERVICES                        │`  
`│                                                                        │`  
`│   [Courier Delivery Zones]                                             │`  
`│   └── GET /zones/applicant-city/:cityId/zones                          │`  
`│                                                                        │`  
`│   [Courier Acquisition Service]                                        │`  
`│   ├── GET /applicants/:applicantId/background-check-new               │`  
`│   ├── GET /applicants/:applicantId/ack-bag-ownership                   │`  
`│   └── PUT /applicants/:applicantId/status                              │`  
`│                                                                        │`  
`│   [Document Management Service]                                        │`  
`│   ├── GET /documents/:documentId (Identity document)                   │`  
`│   └── GET /documents/:documentId (Insurance document)                  │`  
`│                                                                        │`  
`│   [Financial Management Service]                                       │`  
`│   └── GET /financial/couriers/uk/:applicantId/banking-information       │`  
`└────────────────────────────────────────────────────────────────────────┘`

## **3\. Microservice Schema Definitions & DTO Mappings**

### ---

**3.1 Courier Delivery Zones (CDZ)**

* **Endpoint:** GET http\://courier-delivery-zones.app.{{env}}.internal.skipthedishes.com/zones/applicant-city/:cityId/zones  
* **Response DTO:** Set\<ApplicantZone\>  
  `ApplicantZone {`  
      `public final String id;`  
      `public final String name;`  
      `public final String abbreviatedName;`  
      `public final String polygon;`  
      `public final Tenant tenant;`  
      `public final Set<ZoneVehicleType> allowedVehicleTypes;`  
      `public final String cityId;`  
      `public final String city;`  
  `}`

* **Worker Mapping:** Matches zoneId against ApplicantZone.id; verifies that declaredVehicle exists within allowedVehicleTypes; sets isZoneOpen=true and isVehicleSupported=true.

### **3.2 Courier Acquisition Service (BGC, Bag, Status)**

* **BGC Endpoint:** GET http\://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/background-check-new  
* **Response DTO:** BackgroundCheckList  
  `BackgroundCheckList {`  
      `public final List<BackgroundCheck> backgroundChecks;`  
  `}`  
  `BackgroundCheck {`  
      `public final String id;`  
      `public final String applicantId;`  
      `public final Provider provider;`  
      `public final BackgroundCheckStatus checkStatus;`  
      `public final List<BackgroundCheckReport> checkReports;`  
  `}`  
  `BackgroundCheckReport {`  
      `public final String reportId;`  
      `public final ReportType type;`  
      `public final BackgroundCheckReportStatus reportStatus;`  
      `public final Optional<ReportResult> result;`  
      `public final Optional<FinalStatus> finalDecision;`  
      `public final Optional<String> reason;`  
  `}`

* **Bag Ownership Endpoint:** GET http\://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/ack-bag-ownership  
  Returns raw boolean (true \= bag verified).  
* **Status Mutation Endpoint:** PUT http\://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/status  
  Used by both notify-surplus (sets status to WAITLIST) and activate-courier-profile (sets status to ACTIVE).

### **3.3 Document Management Service (ID & Insurance)**

* **Endpoint:** GET http\://document-management-service.app.{{env}}.internal.skipthedishes.com/documents/:documentId  
* **Response DTO:** Document  
  `Document(`  
      `String documentId,`  
      `DocumentName.Name documentName,`  
      `List<DocumentType> documentTypes,`  
      `DocumentStatus status,`  
      `Optional<String> rejectReason,`  
      `Optional<RejectReason> rejectReasonType,`  
      `Optional<Long> expiryDate,`  
      `Optional<Long> issueDate,`  
      `String metadata,`  
      `List<File> files,`  
      `Optional<Long> decisionTime,`  
      `Long uploadedTime,`  
      `Optional<ModifiedUser> modifiedUser,`  
      `Optional<String> suggestion`  
  `)`

* **Worker Mapping:** Checks DocumentStatus \== APPROVED. For insurance, validates that expiryDate is in the future. Throws BPMN error if status is REJECTED.

### **3.4 Financial Management Service (Banking)**

* **Endpoint:** GET http\://financial-management.app.{{env}}.internal.skipthedishes.com/financial/couriers/uk/:applicantId/banking-information  
* **Response Schema:**  
  `{`  
      `"name": "string",`  
      `"accountNumber": "string",`  
      `"branchSortCode": "string"`  
  `}`

* **Worker Mapping:** Validates format (6-digit sort code, 8-digit account number); sets bankStatus="APPROVED".

## **4\. Security, Networking & Environmental Configuration**

* ---

  **Environment Injection:** All endpoint templates use {{env}} interpolation (e.g., development, staging, production).  
* **Service Identification:** Direct service-to-service calls over internal DNS routes pass standard tracing headers:  
  `X-Caller-Service: camunda-onboarding-worker`  
  `X-Correlation-Id: <task.businessKey>`

* **Network Routing:** Host-level worker execution ensures requests route directly through the active AWS Client VPN to internal .skipthedishes.com domains without requiring Docker bridge DNS hacks.