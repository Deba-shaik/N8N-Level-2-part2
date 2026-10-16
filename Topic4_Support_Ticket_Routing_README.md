# Topic 4 — Support Ticket Priority Routing & Conditional Logic

**Program:** Tayana Academy — n8n Hands-On Intermediate Assessment  
**Workflow:** `Topic 4 -lev 2- Support Ticket Priority Routing`  
**Platform:** n8n  
**Documentation date:** 16 October 2026  
**Implementation basis:** *Implementation Plan: N8N Hands-On Intermediate Assessment, Topic 4* (`top imp.pdf`), with the actual configuration evidenced by supplied screenshots.

## 1. Objective

Automate support-ticket intake, validation, priority-based routing, and simulated notification dispatch. Valid tickets are directed to one of four priority queues (Low, Medium, High, Urgent), while unrecognized priorities go to a Fallback/Manual Triage queue. Incomplete tickets are rejected and logged rather than dispatched. A separate mock notification workflow returns a simulated delivery acknowledgment, and an audit node formats a dispatch record.

**Important:** This assessment uses a **mock notification server**. A response status of `DELIVERED` is a simulated acknowledgment, **not proof that a real email was sent**.

## 2. Architecture

```text
WebhookTicketIntake (POST /support-ticket)
    |
Code ValidateAndSanitize
    |
IF HasRequiredFields
    |-- False --> Log RejectedTicket
    |
    `-- True --> Switch RouteByPriority
                    |-- Output 0: Low ----> Set Email Low -----|
                    |-- Output 1: Medium -> Set Email Medium --|
                    |-- Output 2: High ---> Set Email High ----|--> HTTP MockDispatchNotification
                    |-- Output 3: Urgent -> Set Email Urgent --|             |
                    `-- Fallback --------> Set Email Fallback-|             v
                                                                    Code BuildRoutingAudit
```

**Separate mock server:** `Webhook (POST /mock/notifications/dispatch)` → `Respond to Webhook (JSON, HTTP 200)`.

**Connection check:** The audit node produced a successful output in a supplied screenshot. The final overview screenshot appears to place `Code BuildRoutingAudit` to the left of the HTTP node, so **verify the actual canvas connection is HTTP Request → Code BuildRoutingAudit** before exporting the workflow.

## 3. Nodes and responsibilities

| Node | Purpose |
|---|---|
| `WebhookTicketIntake` | Accepts incoming support-ticket JSON via HTTP POST. |
| `Code ValidateAndSanitize` | Normalizes the ticket fields and checks required data; emits `isValid`, `missingFields`, and a structured `ticket`. |
| `IF HasRequiredFields` | Routes valid tickets to the Switch; invalid tickets to rejection logging. |
| `Log RejectedTicket` | Records `TICKET_DROPPED_INCOMPLETE`, missing fields, and timestamp. |
| `Switch RouteByPriority` | Routes Low, Medium, High, Urgent, and unrecognized priorities to separate outputs. |
| `Set Email Low` | Builds recipient, queue, subject, body, and priority for General Tier 0. |
| `Set Email Medium` | Builds the corresponding Tier 1 notification. |
| `Set Email High` | Builds the Tier 2 technical escalation notification. |
| `Set Email Urgent` | Builds the on-call escalation notification. |
| `Set Email Fallback` | Builds the manual-triage notification for unexpected priority values. |
| `HTTP MockDispatchNotification` | Sends a JSON POST to the published mock server. |
| `Code BuildRoutingAudit` | Produces an audit event containing dispatch ID, queue, recipient, mock delivery status, and timestamp. |

## 4. Priority routing table

| Incoming priority | Target queue | Recipient | Evidence status |
|---|---|---|---|
| Low | General Tier 0 Queue | `support-general@example.com` | Set-node output screenshot provided |
| Medium | Tier 1 Support Queue | `support-tier1@example.com` | Set-node output screenshot provided |
| High | Tier 2 Technical Escalations | `support-tier2@example.com` | Branch connected; no individual execution screenshot provided |
| Urgent | On-Call Urgent Page Desk | `oncall-escalation@example.com` | Branch connected; no individual execution screenshot provided |
| Critical / unrecognized | Manual Triage Desk | `support-triage@example.com` | Fallback set-node, mock HTTP and audit outputs provided |
| Missing required field | Rejection log | No dispatch | Rejection output screenshot provided |

The example addresses are **mock assessment recipients**, not verified live support mailboxes.

## 5. Mock notification server

The separate workflow **Mock Notification Server** was shown as **Published** and successfully executed, with green checks on both `Webhook` and `Respond to Webhook`.

**Production mock endpoint:**

```text
POST https://n8n.tayanasolutions.com/webhook/mock/notifications/dispatch
```

The HTTP Request node uses:

- Method: `POST`
- Authentication: `None` (assessment mock only)
- Send Body: `ON`
- Body Content Type: `JSON`
- Specify Body: `Using JSON`

The request maps `recipientEmail`, `queue`, `subject`, `body`, and `priority` from the selected `Set Email` branch. The mock server responds with `dispatchId`, `recipient`, `subject`, `routedQueue`, `status`, and `sentAt`.

Example n8n expression for the HTTP Request JSON field:

```javascript
{{ JSON.stringify({
  recipientEmail: $json.recipientEmail,
  queue: $json.queue,
  subject: $json.subject,
  body: $json.body,
  priority: $json.priority
}) }}
```

## 6. Audit implementation

`Code BuildRoutingAudit` runs once for each input item and uses the mock response to generate an audit record:

```javascript
const resp = $input.item.json;

return {
  json: {
    event: "TICKET_ROUTED_SUCCESSFULLY",
    dispatchId: resp.dispatchId,
    assignedQueue: resp.routedQueue,
    targetRecipient: resp.recipient,
    deliveryStatus: resp.status,
    timestamp: new Date().toISOString()
  }
};
```

**Observed Fallback audit output:**

```json
{
  "event": "TICKET_ROUTED_SUCCESSFULLY",
  "dispatchId": "disp_1792137941242",
  "assignedQueue": "Manual Triage Desk",
  "targetRecipient": "support-triage@example.com",
  "deliveryStatus": "DELIVERED"
}
```

The actual audit output also included a generated timestamp.

## 7. Testing and observed results

| Test | Expected behavior | Evidence / result |
|---|---|---|
| Low ticket | General Tier 0 payload | **Verified at Set Email Low node**; no separate end-to-end audit screenshot |
| Medium ticket | Tier 1 payload | **Verified at Set Email Medium node**; no separate end-to-end audit screenshot |
| High ticket | Tier 2 notification | **Pending end-to-end execution evidence** |
| Urgent ticket | On-call notification | **Pending end-to-end execution evidence** |
| Unrecognized `Critical` priority | Manual Triage Desk | **Verified:** Set Email Fallback → HTTP mock response → audit record |
| Empty `customerEmail` | Reject and log incomplete ticket | **Verified:** `isValid: false`, `missingFields: ["customerEmail"]`, `TICKET_DROPPED_INCOMPLETE` |
| Mock server POST | Return simulated dispatch acknowledgment | **Verified:** published mock server and successful webhook/response execution |

### Rejected-ticket example

The screenshot shows `IF HasRequiredFields` returned `isValid: false` because `customerEmail` was missing. `Log RejectedTicket` produced:

```json
{
  "event": "TICKET_DROPPED_INCOMPLETE",
  "missingFields": ["customerEmail"],
  "timestamp": "2026-10-16T08:08:04.564Z"
}
```

The invalid ticket was not routed to the notification dispatcher in that test.

## 8. How to run the assessment

1. Import or open the main Topic 4 workflow in n8n.
2. Ensure the separate **Mock Notification Server** workflow is published.
3. For editor testing, click **Listen for test event** on `WebhookTicketIntake`.
4. Send a JSON POST to the **test** endpoint. For production mode, publish the main workflow and use the production endpoint.
5. Inspect the Switch output, corresponding Set Email node, HTTP mock response, and audit node. For missing fields, inspect `Log RejectedTicket` instead.

Example Windows CMD test for Fallback:

```bat
curl.exe -X POST "https://n8n.tayanasolutions.com/webhook-test/support-ticket" -H "Content-Type: application/json" -d "{\"customerName\":\"Charlie Brown\",\"customerEmail\":\"charlie@peanuts.org\",\"subject\":\"Special Request\",\"description\":\"Custom billing question\",\"priority\":\"Critical\"}"
```

> The implementation-plan PDF contains **inconsistent webhook-path examples** (`/v1/tickets` and `/support-ticket`). This README uses `/support-ticket`, the path actually used in the supplied working test sequence. Confirm the live Webhook node's displayed URL before running commands.

## 9. Evidence inventory

The accompanying `Topic4_Support_Ticket_Routing_Evidence.pdf` documents these nine screenshot subjects:

1. Low-priority notification mapping and output.
2. Medium-priority notification mapping and output.
3. Fallback notification mapping and output.
4. Published Mock Notification Server with both nodes successful.
5. Dynamic `Respond to Webhook` expression.
6. Main workflow showing successful Fallback HTTP dispatch.
7. `Code BuildRoutingAudit` output.
8. Main workflow overview of invalid-ticket rejection.
9. `Log RejectedTicket` output showing missing customer email.

## 10. Risks, limitations, and final submission checks

- **No real email delivery:** The dispatcher returns simulated delivery responses. A live deployment would need an authorized email or helpdesk integration.
- **Webhook security:** The demonstrated webhook uses no authentication; production deployment should add a suitable authentication or request-signature mechanism.
- **Priority normalization:** Verify case-insensitive inputs such as `low`, `LOW`, and `Low` all map correctly.
- **Retry behavior:** The source implementation plan recommends `Retry on Fail`, 3 attempts, 3000 ms wait. The screenshots do **not** verify that setting; check it in the HTTP node before claiming completion.
- **Audit connection:** Confirm the HTTP node is actually upstream of the audit node in the final exported canvas.
- **Remaining evidence:** Capture individual end-to-end High and Urgent executions, and ideally full dispatch/audit outputs for Low and Medium.

## 11. Assessment outcome

The submitted screenshots support a functioning support-ticket validation and five-way routing design, a working simulated notification endpoint, a successful Fallback dispatch with an audit record, and correct rejection of an incomplete ticket. This is **not** a claim that every branch has passed a full end-to-end test or that live emails were sent.

---

**Reference:** Tayana Academy, *Implementation Plan: N8N Hands-On Intermediate Assessment — Topic 4: Support Ticket Priority Routing & Conditional Logic*, 10 pages, user-provided `top imp.pdf`.
