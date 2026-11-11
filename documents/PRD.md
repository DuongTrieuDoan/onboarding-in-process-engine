# **Product Requirements Document (PRD): Food Courier Onboarding Workflow Engine (V10)**

---

**Document ID:** PRD-CO-ENG-001-V10  
**Initiative:** Courier Onboarding with Process Engine (Camunda 7\)  
**Target Market Launch:** United Kingdom (UK Baseline Demo)  
**Owner / Author:** Duong Doan / Onboarding Core Team  
**Status:** UPDATED \- EXACT BACKEND SERVICE CONTRACTS & APPLICANT USER GATES  
**Target Platform:** SkipTheDishes / One Logistics Courier Acquisition & Activation  
**Previous Version:** PRD-CO-ENG-001-V9

## **1\. Executive Summary & Context**

---

The Courier Onboarding Workflow Engine orchestrates the end-to-end UK courier acquisition and compliance journey using Camunda 7\. Following recent architectural updates, the workflow enforces applicant-driven User Tasks for manual submissions, direct machine-to-machine backend service invocations, and precise technical alignment with SkipTheDishes internal microservice endpoints.

## **2\. Integrated Downstream Service Landscape**

---

All external task workers communicate directly with internal services across the corporate VPN without user Keycloak JWT session tokens:

* **Courier Delivery Zones Service:** courier-delivery-zones.app.{{env}}.internal.skipthedishes.com  
* **Courier Acquisition Service:** courier-acquisition.app.{{env}}.internal.skipthedishes.com  
* **Document Management Service:** document-management-service.app.{{env}}.internal.skipthedishes.com  
* **Financial Management Service:** financial-management.app.{{env}}.internal.skipthedishes.com

## **3\. End-to-End Worker Topic & Backend Contract Matrix**

---

| Topic Name | BPMN Stage / Trigger | HTTP Method & Downstream URL | Payload Contract / Expected DTO | Engine Result Mapping   |
| :---- | :---- | :---- | :---- | :---- |
| fetch-zone-data | Process Start (Automated) | GET http\://courier-delivery-zones.app.{{env}}.internal.skipthedishes.com/zones/applicant-city/:cityId/zones | Set\<ApplicantZone\> (contains id, allowedVehicleTypes, cityId) | Sets isZoneOpen, isVehicleSupported, and capacityStatus for DMN gating |
| notify-surplus | Zone DMN Evaluates to "SURPLUS" | PUT http\://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/status | Status update payload (e.g., WAITLIST or SURPLUS) | Terminates process instance at EndEvent\_Surplus |
| ocr-document-verification | Applicant completes "Upload Government ID" User Task | GET http\://document-management-service.app.{{env}}.internal.skipthedishes.com/documents/:documentId | Document (fields: documentId, status, rejectReason, files) | Sets idStatus="APPROVED" or throws BpmnError("ID\_MANUAL\_REVIEW\_REQUIRED") |
| sterling-bgc-check | Applicant completes "BGC Submission" User Task | GET http\://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/background-check-new | BackgroundCheckList (contains checkReports, reportStatus, finalDecision) | Sets bgcChStatus="CLEAR" and bgcRtwStatus="CLEAR" or throws BpmnError("BGC\_REJECTED") |
| verify-insurance | Applicant completes "Upload Commercial Insurance" User Task | GET http\://document-management-service.app.{{env}}.internal.skipthedishes.com/documents/:documentId | Document (fields: documentName, status, expiryDate) | Sets insuranceStatus="APPROVED" or throws BpmnError("INSURANCE\_REJECTED") |
| verify-thermal-bag | Applicant completes "Update Bag condition" User Task | GET http\://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/ack-bag-ownership | boolean (true/false) | Sets bagStatus="APPROVED" when ownership is acknowledged |
| verify-bank-details | Applicant completes "Update Bank Details" User Task | GET http\://financial-management.app.{{env}}.internal.skipthedishes.com/financial/couriers/uk/:applicantId/banking-information | JSON Object (name, accountNumber, branchSortCode) | Sets bankStatus="APPROVED" upon successful schema validation |
| activate-courier-profile | Activation Gate Pass \+ Path Routing | PUT http\://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/status | Status update payload: {"status": "ACTIVE"} | Finalizes onboarding and advances to EndEvent\_CourierActive |
| notify-sla-breach | 2-hour SLA Timer Fires on LiveOps Audit Queue | Internal Ops Webhook / PagerDuty Dispatch | SLA notification event | Alerts operational leads without interrupting the review task |

## **4\. Functional Requirements & Process Execution**

### ---

**FR-1: Zone Eligibility & Capacity Verification**

Upon registration, the process invokes the Delivery Zones Service via /zones/applicant-city/:cityId/zones to match the courier's requested zone against active geographic polygons and allowed vehicle types (e.g., Car, Motorcycle, Bicycle, E-Bike). If capacity is full or the zone is closed, courier-acquisition is updated via PUT /applicants/:applicantId/status to mark the courier as waitlisted.

### **FR-2: Sequential Applicant Entry and Automated Verification**

To support manual operational entry during local demonstration, the Parallel Requirements Engine routes tokens to User Tasks in Camunda Tasklist. Once an applicant submits their details:

1. **Identity:** Worker queries Document Management Service (/documents/:documentId). If DocumentStatus \!= APPROVED, the error boundary triggers manual LiveOps review or re-upload.  
2. **Background Check:** Worker queries /applicants/:applicantId/background-check-new. If Sterling reports adverse findings, isolated remediation allows re-submission.  
3. **Insurance:** Worker validates the insurance document metadata and expiration date.  
4. **Bag Ownership:** Worker verifies bag ownership acknowledgement via ack-bag-ownership boolean query.  
5. **Bank Details:** Worker verifies UK account details (sort code and account number) via Financial Management Service.

### **FR-3: Activation Gating**

Once all five parallel streams complete, DMN Decision decision\_activation\_gate validates compliance criteria. Approved applicants undergo the 70/30 split. Direct and audited applicants culminate in a call to PUT /applicants/:applicantId/status to transition the applicant to ACTIVE status.
