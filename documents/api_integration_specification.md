# SkipTheDishes Courier Onboarding: Downstream API Integration Specification

This document defines the downstream REST API integration specifications for all external tasks and workers orchestrated by the Camunda 7 process engine (`courier-onboarding-uk`).

---

## 1. Global Infrastructure & Environment Routing

All service endpoints resolve via the SkipTheDishes internal service registry across the corporate VPN:

```
http://<service-name>.app.{{env}}.internal.skipthedishes.com
```

* `{{env}}` represents the runtime target environment (e.g., `development`, `staging`, `production`).
* Host-level workers connect directly without requiring user session token passthrough.
* Standard request headers for tracing:
  * `X-Caller-Service: camunda-onboarding-worker`
  * `X-Correlation-Id: <businessKey>`

---

## 2. Topic to Endpoint Mapping Matrix

| External Task Topic | HTTP Method | Target Downstream Endpoint | DTO / Contract Return Type | Supporting Types / Schemas |
| :--- | :---: | :--- | :--- | :--- |
| `fetch-zone-data` | `GET` | `http://courier-delivery-zones.app.{{env}}.internal.skipthedishes.com/zones/applicant-city/:cityId/zones` | `Set<ApplicantZone>` | `ApplicantZone`, `ZoneVehicleType` |
| `sterling-bgc-check` | `GET` | `http://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/background-check-new` | `BackgroundCheckList` | `BackgroundCheck`, `BackgroundCheckReport` |
| `document-verification` | `GET` | `http://document-management-service.app.{{env}}.internal.skipthedishes.com/documents/:documentId` | `Document` | `DocumentStatus`, `DocumentType` |
| `verify-insurance` | `GET` | `http://document-management-service.app.{{env}}.internal.skipthedishes.com/documents/:documentId` | `Document` | `DocumentStatus`, `DocumentType` |
| `verify-thermal-bag` | `GET` | `http://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/ack-bag-ownership` | `boolean` | `true` (approved) / `false` (pending) |
| `verify-bank-details` | `GET` | `http://financial-management.app.{{env}}.internal.skipthedishes.com/financial/couriers/uk/:applicantId/banking-information` | `BankingInformation` | `name`, `accountNumber`, `branchSortCode` |
| `activate-courier-profile` | `PUT` | `http://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/status` | `StatusMutationResponse` | `{"status": "ACTIVE"}` |
| `notify-surplus` | `POST` | `http://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/city-zones` | `ApplicantCityZones` | `DeliveryZoneWithCity` |
| *(notify-surplus vehicle)* | `POST` | `http://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/update-selected-vehicle` | `SelectedVehicle` | `Vehicle`, `MotorVehicleType` |
| `update-zone-vehicle` | `POST` | `http://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/city-zones` | `ApplicantCityZones` | `DeliveryZoneWithCity` |
| *(update vehicle)* | `POST` | `http://courier-acquisition.app.{{env}}.internal.skipthedishes.com/applicants/:applicantId/update-selected-vehicle` | `SelectedVehicle` | `Vehicle`, `MotorVehicleType` |

---

## 3. Data Transfer Objects (DTOs) & Models

### 3.1 Courier Delivery Zones (CDZ)

#### `ApplicantZone`
```java
public class ApplicantZone {
    public final String id;
    public final String name;
    public final String abbreviatedName;
    public final String polygon;
    public final Tenant tenant;
    public final Set<ZoneVehicleType> allowedVehicleTypes;
    public final String cityId;
    public final String city;
}
```

---

### 3.2 Courier Acquisition Service

#### `BackgroundCheckList` & `BackgroundCheck`
```java
public class BackgroundCheckList {
    public final List<BackgroundCheck> backgroundChecks;
}

public class BackgroundCheck {
    public final String id;
    public final String applicantId;
    public final Provider provider;
    public final Optional<String> referenceApplicantId;
    public final Optional<String> referenceCheckId;
    public final BackgroundCheckStatus checkStatus;
    public final Optional<Long> emailSentDate;
    public final Optional<Long> dataReceivedDate;
    public final Optional<Long> checkExecutedDate;
    public final Optional<Long> completedDate;
    public final Optional<Long> finalDecisionDate;
    public final List<BackgroundCheckReport> checkReports;
}
```

#### `ApplicantCityZones` & `DeliveryZoneWithCity`
```java
public class ApplicantCityZones {
    public List<DeliveryZoneWithCity> zones;
}

public class DeliveryZoneWithCity {
    public final String id;
    public final Set<ZoneVehicleType> vehicleTypes;
    public final String city;
}
```

#### `SelectedVehicle`
```java
public class SelectedVehicle {
    public final Vehicle vehicle;
    public final Optional<MotorVehicleType> motorVehicleType;
}

public enum Vehicle {
    CAR,
    BICYCLE
}

public enum MotorVehicleType {
    MOTORBIKE,
    MOTOR_VEHICLE
}
```

---

### 3.3 Document Management Service (DMS)

#### `Document`
```java
public class Document {
    public final String documentId;
    public final DocumentName.Name documentName;
    public final List<DocumentType> documentTypes;
    public final DocumentStatus status;
    public final Optional<String> rejectReason;
    public final Optional<RejectReason> rejectReasonType;
    public final Optional<Long> expiryDate;
    public final Optional<Long> issueDate;
    public final String metadata;
    public final List<File> files;
    public final Optional<Long> decisionTime;
    public final Long uploadedTime;
    public final Optional<ModifiedUser> modifiedUser;
    public final Optional<String> suggestion;
}
```

---

### 3.4 Financial Management Service

#### Banking Information Schema (JSON)
```json
{
  "name": "string",
  "accountNumber": "string",
  "branchSortCode": "string"
}
```

---

## 4. Camunda Variable Translation Rules

| Target Service / Field | Expected Value | Process Variable Set |
| :--- | :--- | :--- |
| `ApplicantZone.allowedVehicleTypes` | Contains declared vehicle | `isVehicleSupported = true` |
| `Document.status` | `APPROVED` | `idStatus = "APPROVED"` or `insuranceStatus = "APPROVED"` |
| `Document.status` | `REJECTED` | Throws `BpmnError("ID_MANUAL_REVIEW_REQUIRED")` or `BpmnError("INSURANCE_REJECTED")` |
| `BackgroundCheck.checkStatus` | `COMPLETED` | `bgcChStatus = "CLEAR"`, `bgcRtwStatus = "CLEAR"` |
| `BackgroundCheck.checkStatus` | Adverse findings | Throws `BpmnError("BGC_REJECTED")` |
| `ack-bag-ownership` | `true` | `bagStatus = "APPROVED"` |
| Banking fields valid | Sort code & account number present | `bankStatus = "APPROVED"` |