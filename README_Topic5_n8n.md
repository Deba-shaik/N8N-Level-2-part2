# Topic 5 — Loop Processing, Pagination & Rate-Limited Batching

## Project overview
This n8n assessment demonstrates paginated retrieval from a mock users API, accumulation of records, processing in controlled batches, simulated email requests, failure classification, a rate-limit pause, and a final execution summary.

**Implementation status:** Completed and successfully executed in the provided screenshots. The email requests are simulations; no actual email delivery is claimed.

## Verified execution results

| Metric | Result |
|---|---:|
| Total users processed | 23 |
| Simulated sends classified SENT | 22 |
| Records classified FAILED | 1 |
| Processing batches | 5 |
| Pagination HTTP requests | 4 |
| Final status | COMPLETED |

The invalid test email is assigned to User 13 (`bad-email-format`). Batch size is 5, yielding batches of 5, 5, 5, 5 and 3. The configured Wait node uses a 2-second interval; exact elapsed intervals are not established by the supplied screenshots.

## Architecture

### Workflow A: Mock Paginated Users API

`Webhook (GET) → Generate_23_Users (Code) → Respond to Webhook`

- Webhook path: `mock/api/users`
- Production endpoint: `https://n8n.tayanasolutions.com/webhook/mock/api/users`
- Query parameters: `page` and `limit`
- Mock dataset: 23 users, with User 13 deliberately having an invalid email address.
- Page size 10 yields page counts of 10, 10, 3, then 0.
- The published mock API workflow was shown with all three nodes executed successfully.

### Workflow B: Topic 5 main workflow

`Manual Trigger → Set Config → Code_InitPaging → HTTP_FetchPage → IF_HasMorePages`

- **True (more users):** `Code_AccumulatePage → HTTP_FetchPage` (next page)
- **False (no users):** `Code_PrepareDataset → Loop_Over_Items`
- **Loop output:** `HTTP_SendSimulatedEmail → Code_ClassifyResult → Wait_RateLimitPause → Loop_Over_Items`
- **Done output:** `Code_BuildExecutionSummary`

## Node descriptions

| Node | Purpose |
|---|---|
| Manual Trigger | Starts a manual test execution |
| Set Config | Stores page size, batch size and wait settings |
| Code_InitPaging | Initializes pagination state |
| HTTP_FetchPage | Retrieves a page from the mock users API |
| IF_HasMorePages | Tests whether the API page contains users |
| Code_AccumulatePage | Collects users and advances the page counter |
| Code_PrepareDataset | Converts accumulated users into individual n8n items |
| Loop_Over_Items | Processes the dataset in batches of 5 |
| HTTP_SendSimulatedEmail | POSTs simulated email data to JSONPlaceholder |
| Code_ClassifyResult | Classifies records as SENT or FAILED, including invalid email format |
| Wait_RateLimitPause | Pauses between loop iterations; configured for 2 seconds |
| Code_BuildExecutionSummary | Reports processed, sent, failed, batches and completion status |

## How to run

1. Open the **Mock Paginated Users API** workflow and ensure its published production webhook is available.
2. Optionally check its pages using the commands below.
3. Open **Topic 5 - Loop Processing, Pagination & Rate-Limited Batching**.
4. Verify that `HTTP_FetchPage` uses the production webhook URL and sends `page` and `limit` query parameters.
5. Verify the configured batch size is 5 and the Wait interval is 2 seconds.
6. Run the main workflow from **Manual Trigger**.
7. Open `Code_BuildExecutionSummary` and confirm the output metrics.

### Optional Windows CMD API checks

```bat
curl "https://n8n.tayanasolutions.com/webhook/mock/api/users?page=1&limit=10"
curl "https://n8n.tayanasolutions.com/webhook/mock/api/users?page=2&limit=10"
curl "https://n8n.tayanasolutions.com/webhook/mock/api/users?page=3&limit=10"
curl "https://n8n.tayanasolutions.com/webhook/mock/api/users?page=4&limit=10"
```

Expected user counts are 10, 10, 3 and 0 respectively.

## Example final output

```json
{
  "totalProcessed": 23,
  "sent": 22,
  "failed": 1,
  "totalBatches": 5,
  "status": "COMPLETED"
}
```

The actual final output also includes a `completedAt` timestamp.

## Validation and evidence

- Main workflow screenshot: all nodes green, 4 pagination executions, 23 records passed through processing, 5 batch-processing iterations.
- Mock API screenshot: published workflow with Webhook, Generate_23_Users and Respond to Webhook successfully executed.
- Final summary screenshot: 23 processed, 22 sent, 1 failed, 5 batches, COMPLETED.
- The test simulates an invalid email failure while allowing other records to continue.

## Limitations

- JSONPlaceholder accepts simulated POST requests; `SENT` means successful simulation/classification, **not** a delivered email.
- Execution timing screenshots are needed to independently prove the precise rate-limit delay.
- Workflow JSON exports should be submitted alongside this README and the evidence report when required by the assessment.

## Suggested submission files

- `README_Topic5_n8n.md` (this file)
- `Topic5_n8n_Evidence_Report.pdf`
- `Topic5_n8n_Evidence_Report.md`
- Exported JSON for both n8n workflows
- Workflow and final-summary screenshots
