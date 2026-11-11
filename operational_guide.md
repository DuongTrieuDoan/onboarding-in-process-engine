# End-to-End Operational Guide: Running the UK Courier Onboarding Workflow

This guide walks you through deploying, initiating, operating, and verifying the complete UK Courier Onboarding process (`courier-onboarding-uk`) using Camunda 7 and the standalone host worker.

## 1. Prerequisites & Environment Setup

Ensure the following tools and services are available before running the process:

* **Docker:** Engine running locally.
* **Camunda 7 Platform:** Deployed in Docker, exposing port `9090` (e.g., `http://localhost:9090/engine-rest`).
* **Node.js:** Runtime installed (v18+ or v20+) with `"type": "module"` enabled in `package.json`.
* **Camunda Modeler:** Desktop application (v5.20+) for diagram inspection or manual deployment.
* **VPN Connection:** Corporate AWS Client VPN active if querying live downstream staging endpoints.

---

## 2. Artifact Deployment

The engine requires three primary definition files to be deployed simultaneously under the same deployment boundary:

1. `courier-onboarding-uk.bpmn` (Process Definition)
2. `zone-vehicle-eligibility.dmn` (Decision: `decision_zone_vehicle_eligibility`)
3. `activation-gate.dmn` (Decision: `decision_activation_gate`)

### Deploying via cURL

Run the multi-part upload command from your terminal:

```bash
curl -X POST "http://localhost:9090/engine-rest/deployment/create" \
  -H "accept: application/json" \
  -F "deployment-name=courier-onboarding-uk-v1" \
  -F "enable-duplicate-filtering=true" \
  -F "deploy-changed-only=true" \
  -F "data=@courier-onboarding-uk.bpmn" \
  -F "data=@zone-vehicle-eligibility.dmn" \
  -F "data=@activation-gate.dmn"
```

Verify the deployment output returns HTTP `200 OK` and includes entries in `deployedProcessDefinitions` and `deployedDecisionDefinitions`.

---

## 3. Starting the Background Worker

Start the external task worker on your host machine to handle automatic validations:

```bash
cd /path/to/camunda-workers
node workers.js
```

Verify that the terminal output confirms active polling:

```text
[Worker] Started. Polling Camunda engine at: http://localhost:9090/engine-rest
[Worker] Target backend environment: staging
```

Leave this terminal open throughout the execution of the process.

---

## 4. Initiating a Process Instance (Stage 1 Entry)

Trigger a new courier onboarding instance by executing a `POST` request against the process definition start endpoint. At process start, provide only the foundational applicant and routing parameters:

```bash
curl -X POST "http://localhost:9090/engine-rest/process-definition/key/courier-onboarding-uk/start" \
  -H "Content-Type: application/json" \
  -d '{
    "businessKey": "COURIER-UK-DEMO-001",
    "variables": {
      "courierId": { "value": "app_demo_101", "type": "String" },
      "cityId": { "value": "london", "type": "String" },
      "zoneId": { "value": "zone-london-central", "type": "String" },
      "declaredVehicle": { "value": "CAR", "type": "String" }
    }
  }'
```

---

## 5. End-to-End Workflow Stages

### Stage 1: Zone & Vehicle Evaluation

1. **Task:** `Task_FetchZoneData` (Topic: `fetch-zone-data`).
2. **Execution:** The worker queries the Delivery Zones service (`/zones/applicant-city/:cityId/zones`) and injects `isZoneOpen`, `isVehicleSupported`, and `capacityStatus`.
3. **Decision:** `decision_zone_vehicle_eligibility` evaluates eligibility:
   * **If Surplus/Waitlist:** Token routes to `notify-surplus` and terminates at `EndEvent_Surplus`.
   * **If Eligible:** Advances to `update-zone-vehicle` to persist zone selections, then enters the parallel requirements engine.

---

### Stage 2: Parallel Applicant Requirements Engine & Variable Supplementation

The execution token splits across 5 parallel branches at `Gateway_Fork_Requirements`. Each branch pauses on an applicant User Task waiting for manual submission.

This is the stage where document references and the audit score are supplemented into the process context:
* `"idDocumentId": "doc_id_9901"`
* `"insuranceDocId": "doc_ins_4402"`
* `"randomActivationScore": 45`

#### Option A: Completing & Supplementing Variables via Camunda Tasklist (UI)

1. Navigate to `http://localhost:9090/camunda/app/tasklist/` and log in (`demo` / `demo`).
2. Open your task filter (e.g., **All Tasks**).
3. **Upload Government ID (`Task_ID_Remediation`):**
   * Click **Claim**.
   * Under the **Form** / **Add Variable** tab, add:
     * Name: `idDocumentId`, Type: `String`, Value: `doc_id_9901`
   * Click **Complete**. The worker will automatically trigger `document-verification` using this document ID.
4. **Upload Commercial Insurance (`Task_Insurance_Remediation`):**
   * Click **Claim**.
   * Add:
     * Name: `insuranceDocId`, Type: `String`, Value: `doc_ins_4402`
   * Click **Complete**. The worker will automatically trigger `verify-insurance` using this document ID.
5. **BGC Submission (`Task_BGC_Remediation`):**
   * Click **Claim**.
   * (Optional) Supplement `randomActivationScore` here:
     * Name: `randomActivationScore`, Type: `Integer`, Value: `45`
   * Click **Complete**. The worker triggers `sterling-bgc-check`.
6. **Update Bag condition (`Activity_1ah8sz4`):**
   * Click **Claim** and **Complete**. The worker triggers `verify-thermal-bag`.
7. **Update Bank Details (`Activity_1d83ijo`):**
   * Click **Claim** and **Complete**. The worker triggers `verify-bank-details`.

#### Option B: Completing & Supplementing Variables via REST API

If automating Stage 2 or testing via terminal:

1. Query pending task IDs for this instance:
   ```bash
   curl -s "http://localhost:9090/engine-rest/task?processInstanceBusinessKey=COURIER-UK-DEMO-001" | jq '.[].id, .[].name'
   ```

2. Complete **Upload Government ID** with `idDocumentId`:
   ```bash
   curl -X POST "http://localhost:9090/engine-rest/task/<ID_TASK_ID>/complete" \
     -H "Content-Type: application/json" \
     -d '{
       "variables": {
         "idDocumentId": { "value": "doc_id_9901", "type": "String" }
       }
     }'
   ```

3. Complete **Upload Commercial Insurance** with `insuranceDocId`:
   ```bash
   curl -X POST "http://localhost:9090/engine-rest/task/<INSURANCE_TASK_ID>/complete" \
     -H "Content-Type: application/json" \
     -d '{
       "variables": {
         "insuranceDocId": { "value": "doc_ins_4402", "type": "String" }
       }
     }'
   ```

4. Complete **BGC Submission** with `randomActivationScore`:
   ```bash
   curl -X POST "http://localhost:9090/engine-rest/task/<BGC_TASK_ID>/complete" \
     -H "Content-Type: application/json" \
     -d '{
       "variables": {
         "randomActivationScore": { "value": 45, "type": "Integer" }
       }
     }'
   ```

5. Complete the remaining tasks (`Activity_1ah8sz4` and `Activity_1d83ijo`) with empty variable payloads `{}`.

---

### Stage 3: Activation Gate & Audit Routing

Once all 5 tracks complete and join at `Gateway_Join_Requirements`, the token enters `decision_activation_gate`:

1. **Compliance Check:** Evaluates `idStatus`, `insuranceStatus`, `bgcChStatus`, `bgcRtwStatus`, and `bagStatus`.
   * If any mandatory check is blocked or unapproved, routes to `Task_Activation_Remediation`.
2. **70/30 Routing:**
   * **Auto-Activation (`randomActivationScore <= 70`):** Because `randomActivationScore` was supplemented as `45`, the gateway automatically selects `Flow_Auto_Activate` straight to `Task_ActivateCourier`.
   * **LiveOps Audit Queue (`randomActivationScore > 70`):** If a score $>70$ is provided, the token routes to `Task_LiveOps_ActivationQueue` assigned to `liveops-agents` with an active 2-hour SLA timer (`Boundary_SLA_2Hours`).

---

### Stage 4: Profile Activation

1. **Task:** `Task_ActivateCourier` (Topic: `activate-courier-profile`).
2. **Execution:** The worker issues a `PUT /applicants/:applicantId/status` request with payload `{"status": "ACTIVE"}`.
3. **End Event:** Token terminates at `EndEvent_CourierActive` (*Courier Activated*).

---

## 6. Verifying Process Completion

To verify the instance completed successfully:

1. **Cockpit UI:** Open `http://localhost:9090/camunda/app/cockpit/default/#/processes`, select `courier-onboarding-uk`, and check the **History** tab to see all completed activities.
2. **REST API History:**
   ```bash
   curl -s "http://localhost:9090/engine-rest/history/process-instance?processInstanceBusinessKey=COURIER-UK-DEMO-001" | jq
   ```