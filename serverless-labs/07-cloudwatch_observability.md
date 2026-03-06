# CloudWatch Native Observability — Dashboards, Log Insights & X-Ray Traces

Explore the observability data that already flows into CloudWatch from the AI document-processing pipeline: custom EMF metrics emitted by Powertools, structured log queries with Log Insights, service maps and end-to-end traces in X-Ray, and the pre-built alarms that guard the orchestrator Lambda.

---

## Table of Contents

1. [How CloudWatch Fits the Observability Picture](#1-how-cloudwatch-fits-the-observability-picture)
2. [What Is Already Instrumented](#2-what-is-already-instrumented)
3. [Part 1 — CloudWatch Metrics & Custom EMF Dashboard](#part-1--cloudwatch-metrics--custom-emf-dashboard)
4. [Part 2 — CloudWatch Log Insights](#part-2--cloudwatch-log-insights)
5. [Part 3 — AWS X-Ray Traces](#part-3--aws-x-ray-traces)
6. [Part 4 — CloudWatch Alarms](#part-4--cloudwatch-alarms)
7. [Part 5 — Composite Dashboard (putting it all together)](#part-5--composite-dashboard-putting-it-all-together)
8. [Troubleshooting](#troubleshooting)

---

## 1. How CloudWatch Fits the Observability Picture

Lab 06 built the **OpenSearch pipeline** (Logstash-equivalent log shipping, Elasticsearch-equivalent storage, Kibana-equivalent dashboards). CloudWatch provides a complementary, tighter AWS-native layer:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Orchestrator Lambda (OrchestratorContainer-dev)                        │
│                                                                         │
│  Powertools Logger  ──▶  CloudWatch Logs  ──▶  Log Insights queries     │
│  Powertools Metrics ──▶  CloudWatch Metrics  ──▶  Dashboards / Alarms   │
│  Powertools Tracer  ──▶  AWS X-Ray  ──▶  Service Map / Trace timeline   │
└─────────────────────────────────────────────────────────────────────────┘
```

| Concern | OpenSearch (Lab 06) | CloudWatch (this lab) |
|---------|--------------------|-----------------------|
| Log search | Full-text, faceted in Dashboards | SQL-like Log Insights queries |
| Metrics | Custom dashboards fed from log fields | Native Lambda metrics + EMF custom metrics |
| Tracing | Not covered | X-Ray service map + segment timeline |
| Alerting | OpenSearch alerting plugin | CloudWatch Alarms (already deployed) |
| Retention cost | OpenSearch cluster cost | Pay-per-query + storage tiers |

**Use CloudWatch when** you need sub-minute alarm response, native Lambda metrics, or distributed traces.
**Use OpenSearch when** you need ad-hoc full-text search across large log volumes or Kibana-style dashboards.

---

## 2. What Is Already Instrumented

The orchestrator Lambda (`services/ai-doc-processor/app/orchestrator/lambda_function.py`) is fully instrumented with AWS Lambda Powertools. No code changes are needed for this lab.

### Powertools clients

```python
logger  = Logger(service="ai-doc-processor", level=LOG_LEVEL)
metrics = Metrics(namespace="AIDocProcessor", service="ai-doc-processor")
tracer  = Tracer(service="ai-doc-processor")
```

### Handler decorators

```python
@logger.inject_lambda_context(log_event=False)   # adds request_id, cold_start, etc.
@metrics.log_metrics(capture_cold_start_metric=True)  # flushes EMF + captures ColdStart
@tracer.capture_lambda_handler                   # wraps handler in X-Ray segment
def lambda_handler(event, context): ...
```

### Custom metrics emitted (namespace: `AIDocProcessor`)

| Metric name | Unit | Emitted when |
|---|---|---|
| `ColdStart` | Count | Lambda initialises a new execution environment |
| `InvoicesReceived` | Count | S3 upload event received |
| `InvoicesProcessed` | Count | Pipeline completes successfully |
| `InvoiceProcessingErrors` | Count | Unhandled exception in pipeline |
| `TextractExtractionAttempts` | Count | `textract_extraction_agent` tool called |
| `InvoiceValidationAttempts` | Count | `validate_invoice_data` tool called |
| `SapPostingAttempts` | Count | `perform_invoice_posting_to_sap` tool called |
| `WhatsAppNotificationAttempts` | Count | `send_whatsapp_notification` tool called |

### Pre-deployed CloudWatch Alarms

| Alarm | Threshold | Period |
|---|---|---|
| `OrchestratorContainer-dev-Errors` | Errors ≥ 1 | 1 minute |
| `OrchestratorContainer-dev-Duration-p95` | p95 duration > 5 min (300 000 ms) | 5 minutes |
| `OrchestratorContainer-dev-Throttles` | Throttles ≥ 1 | 5 minutes |

---

## Part 1 — CloudWatch Metrics & Custom EMF Dashboard

### Step 1 — Enable Active X-Ray Tracing on the Lambda

> **Note:** This is needed before Part 3 (X-Ray). Do it first so traces accumulate while you build dashboards.

```bash
aws lambda update-function-configuration \
  --function-name OrchestratorContainer-dev \
  --tracing-config Mode=Active \
  --region ap-southeast-2
```

Confirm:
```bash
aws lambda get-function-configuration \
  --function-name OrchestratorContainer-dev \
  --region ap-southeast-2 \
  --query "TracingConfig"
```
Expected: `{"Mode": "Active"}`

---

### Step 2 — Generate some data

Trigger the Lambda a few times so metrics and traces are present.

```bash
# Option A — API Gateway (instant)
API_URL=$(aws cloudformation describe-stacks \
  --stack-name AIDocProcessorStack \
  --region ap-southeast-2 \
  --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" \
  --output text)

curl "$API_URL/items"
curl "$API_URL/items"
curl "$API_URL/items"

# Option B — S3 upload (triggers full invoice pipeline)
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name AIDocProcessorStack \
  --region ap-southeast-2 \
  --query "Stacks[0].Outputs[?contains(OutputKey,'Bucket')].OutputValue" \
  --output text)

echo "test invoice" > /tmp/invoice.txt
aws s3 cp /tmp/invoice.txt "s3://${BUCKET}/invoice-001.txt" --region ap-southeast-2
```

---

### Step 3 — Browse Lambda standard metrics

1. Open **CloudWatch → Metrics → All metrics**
2. Choose **Lambda → By Function Name**
3. Select `OrchestratorContainer-dev`
4. Tick these metrics:

   | Metric | Statistic to use |
   |---|---|
   | `Invocations` | Sum |
   | `Errors` | Sum |
   | `Duration` | p95 |
   | `Throttles` | Sum |
   | `ConcurrentExecutions` | Maximum |

5. Set the time range to **Last 1 hour**, period **1 minute**.

---

### Step 4 — Browse custom EMF metrics

1. Still in **CloudWatch → Metrics → All metrics**
2. Choose **Custom Namespaces → AIDocProcessor → Service**
3. Select `ai-doc-processor`
4. You will see all custom metrics from the table above.
5. Add `InvoicesReceived`, `InvoicesProcessed`, and `ColdStart` to the graph.

> **Why EMF?** Powertools writes metrics as Embedded Metric Format JSON to stdout. CloudWatch Logs automatically extracts them as real CloudWatch metrics — no agent, no extra API calls.

---

### Step 5 — Create a CloudWatch Dashboard

1. Navigate to **CloudWatch → Dashboards → Create dashboard**
2. Name it: `AIDocProcessor-dev`
3. Add the following widgets:

#### Widget 1 — Invocations & Errors (line chart)

- **Source:** Lambda → By Function Name → `OrchestratorContainer-dev`
- **Metrics:** `Invocations` (Sum), `Errors` (Sum)
- **Period:** 1 minute | **Title:** `Invocations vs Errors`

#### Widget 2 — Duration p95 (line chart)

- **Source:** Lambda → By Function Name → `OrchestratorContainer-dev`
- **Metric:** `Duration` — statistic **p95**
- **Period:** 5 minutes | **Title:** `Duration p95 (ms)`
- Add a horizontal annotation at **300 000 ms** (alarm threshold)

#### Widget 3 — Business pipeline metrics (bar chart)

- **Source:** AIDocProcessor → Service → `ai-doc-processor`
- **Metrics:** `InvoicesReceived`, `InvoicesProcessed`, `InvoiceProcessingErrors`
- **Period:** 5 minutes | **Title:** `Invoice Pipeline Throughput`

#### Widget 4 — Tool invocation counts (bar chart)

- **Source:** AIDocProcessor → Service → `ai-doc-processor`
- **Metrics:** `TextractExtractionAttempts`, `InvoiceValidationAttempts`, `SapPostingAttempts`, `WhatsAppNotificationAttempts`
- **Period:** 5 minutes | **Title:** `Agent Tool Invocations`

#### Widget 5 — Cold starts (number widget)

- **Source:** AIDocProcessor → Service → `ai-doc-processor`
- **Metric:** `ColdStart` — statistic **Sum**
- **Period:** 1 hour | **Title:** `Cold Starts (last hour)`

4. Click **Save dashboard**.

---

## Part 2 — CloudWatch Log Insights

Log Insights lets you run SQL-like queries against the structured JSON logs that Powertools writes to CloudWatch Logs.

### Navigate to Log Insights

**CloudWatch → Log Insights**
Log group: `/aws/lambda/OrchestratorContainer-dev`
Time range: **Last 1 hour**

---

### Query 1 — All structured log events

```sql
fields @timestamp, level, message, service, cold_start, xray_trace_id
| sort @timestamp desc
| limit 50
```

> Powertools injects `level`, `message`, `service`, `cold_start`, and `xray_trace_id` automatically into every log record.

---

### Query 2 — Errors only

```sql
fields @timestamp, level, message, error.type, error.value, bucket, key
| filter level = "ERROR"
| sort @timestamp desc
| limit 20
```

---

### Query 3 — Invoice processing latency (S3 triggers only)

```sql
fields @timestamp, message, @duration, bucket, key
| filter message = "Orchestration pipeline completed successfully"
| stats avg(@duration) as avg_ms,
        max(@duration) as max_ms,
        min(@duration) as min_ms,
        count() as total_runs
```

---

### Query 4 — Cold start frequency

```sql
fields @timestamp, cold_start, function_name, function_memory_size
| filter cold_start = 1
| stats count() as cold_starts by bin(1h)
```

---

### Query 5 — Tool invocation breakdown

```sql
fields @timestamp, message, tool, process_id
| filter ispresent(tool)
| stats count() as invocations by tool
| sort invocations desc
```

---

### Query 6 — Error rate over time

```sql
fields @timestamp, level
| stats sum(level = "ERROR") as errors,
        count() as total,
        sum(level = "ERROR") / count() * 100 as error_rate_pct
  by bin(5m)
| sort @timestamp asc
```

---

### Save a query

1. Run **Query 5** (tool invocation breakdown).
2. Click **Save** → name it `Tool Invocation Breakdown`.
3. Saved queries appear under **Saved queries** for re-use in dashboards.

### Add a Log Insights widget to the dashboard

1. Open `AIDocProcessor-dev` dashboard → **Add widget → Log table**
2. Select log group `/aws/lambda/OrchestratorContainer-dev`
3. Paste **Query 2** (errors only)
4. Title: `Recent Errors`
5. Save the dashboard.

---

## Part 3 — AWS X-Ray Traces

X-Ray captures the end-to-end journey of every Lambda invocation, broken into **segments** (Lambda runtime) and **subsegments** (annotated code blocks added by Powertools).

### Navigate to X-Ray

**CloudWatch → X-Ray traces → Service Map**
*(or search for "X-Ray" in the AWS Console top bar)*

> If the service map is empty, wait 1–2 minutes after triggering the Lambda and refresh. X-Ray ingests traces asynchronously.

---

### Step 1 — Explore the Service Map

The service map shows every AWS service the Lambda communicates with and the call latency between them.

Expected nodes for the AI Doc Processor:
```
  Client (API GW / S3)
       │
       ▼
  OrchestratorContainer-dev  [Lambda]
       │
       ├──▶  Amazon Bedrock
       │
       ├──▶  Amazon S3  (bucket reads)
       │
       └──▶  Amazon Textract  (if S3 trigger was used)
```

**What to look for:**
- Each node shows **Requests**, **Faults** (5xx), **Errors** (4xx), **Throttles**
- Edge labels show average response time between services
- Red/orange nodes indicate errors — click to drill into traces

---

### Step 2 — Open the Trace List

1. **CloudWatch → X-Ray traces → Traces**
2. Set time range: **Last 30 minutes**
3. Filter expression (optional):
   ```
   service("ai-doc-processor") AND duration > 1
   ```
4. You will see individual trace rows with:
   - **Trace ID** — unique identifier
   - **Duration** — total end-to-end time
   - **Status** — OK / Error / Fault
   - **Root cause** service

---

### Step 3 — Inspect a single trace

1. Click any trace row.
2. The **Segment Timeline** opens — a Gantt chart of every segment and subsegment.

Expected segments for an S3-triggered invocation:
```
▼ OrchestratorContainer-dev  [initialisation + handler]
   ├─ Initialization          [cold start only — Lambda runtime init]
   ├─ Invocation              [your handler code]
   │   ├─ lambda_handler      [@tracer.capture_lambda_handler subsegment]
   │   │   ├─ BedrockModel    [Bedrock API call — auto-traced by Powertools]
   │   │   └─ ...tool calls...
   └─ Overhead                [Lambda post-processing]
```

3. Click any subsegment to see:
   - **Metadata** — key-value data added by Powertools (`put_metadata`)
   - **Annotations** — indexed fields searchable across all traces
   - **Exception** — full traceback if the subsegment failed

---

### Step 4 — Filter traces by annotation

Powertools Tracer automatically annotates traces with `cold_start` and `service`.  You can search for them:

1. In the Traces view, click **Add filter**
2. Select **Annotation** → `cold_start` → `true`
3. This shows only cold-start invocations — useful for sizing memory/provisioned concurrency.

---

### Step 5 — X-Ray Analytics (advanced)

1. **CloudWatch → X-Ray traces → Analytics**
2. Set time range to **Last 1 hour**
3. Use the histogram to identify **latency outliers**:
   - Drag to select the slowest 10% of traces
   - The filter auto-updates to show only those traces
4. Click **Groups** to define a persistent filter:
   - Name: `SlowInvocations`
   - Filter: `duration > 30 AND service("ai-doc-processor")`

---

### Step 6 — Add an X-Ray widget to the CloudWatch Dashboard

1. Open `AIDocProcessor-dev` dashboard → **Add widget → Trace map**
2. Filter: `service("ai-doc-processor")`
3. Title: `X-Ray Service Map`
4. Save the dashboard.

---

## Part 4 — CloudWatch Alarms

Three alarms were deployed by CDK in `ai_doc_processor_stack.py`. This part shows how to test them and configure SNS notifications.

### View existing alarms

```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix "OrchestratorContainer-dev" \
  --region ap-southeast-2 \
  --query "MetricAlarms[*].[AlarmName,StateValue,ComparisonOperator,Threshold]" \
  --output table
```

Expected output:
```
-------------------------------------------------------------------------------------------
|                                     DescribeAlarms                                      |
+----------------------------------------------+-------+-------------------------------+-----------+
|  OrchestratorContainer-dev-Errors            |  OK   |  GreaterThanOrEqualToThreshold |  1.0      |
|  OrchestratorContainer-dev-Duration-p95      |  OK   |  GreaterThanOrEqualToThreshold |  300000.0 |
|  OrchestratorContainer-dev-Throttles         |  OK   |  GreaterThanOrEqualToThreshold |  1.0      |
+----------------------------------------------+-------+-------------------------------+-----------+
```

---

### Trigger the Errors alarm (test)

```bash
# Force the alarm to ALARM state for testing
aws cloudwatch set-alarm-state \
  --alarm-name "OrchestratorContainer-dev-Errors" \
  --state-value ALARM \
  --state-reason "Manual test trigger" \
  --region ap-southeast-2

# Check state
aws cloudwatch describe-alarms \
  --alarm-names "OrchestratorContainer-dev-Errors" \
  --region ap-southeast-2 \
  --query "MetricAlarms[0].StateValue"

# Reset to OK
aws cloudwatch set-alarm-state \
  --alarm-name "OrchestratorContainer-dev-Errors" \
  --state-value OK \
  --state-reason "Reset after test" \
  --region ap-southeast-2
```

---

### Add an SNS email notification

```bash
# 1. Create an SNS topic
TOPIC_ARN=$(aws sns create-topic \
  --name ai-doc-processor-alerts-dev \
  --region ap-southeast-2 \
  --query TopicArn --output text)

echo "Topic ARN: $TOPIC_ARN"

# 2. Subscribe your email
aws sns subscribe \
  --topic-arn "$TOPIC_ARN" \
  --protocol email \
  --notification-endpoint "you@example.com" \
  --region ap-southeast-2
# Confirm the subscription in your inbox

# 3. Attach the topic to all three alarms
for ALARM in \
  "OrchestratorContainer-dev-Errors" \
  "OrchestratorContainer-dev-Duration-p95" \
  "OrchestratorContainer-dev-Throttles"; do
  aws cloudwatch put-metric-alarm \
    --alarm-name "$ALARM" \
    --alarm-actions "$TOPIC_ARN" \
    --region ap-southeast-2
done
```

---

### View alarm history

```bash
aws cloudwatch describe-alarm-history \
  --alarm-name "OrchestratorContainer-dev-Errors" \
  --region ap-southeast-2 \
  --query "AlarmHistoryItems[*].[Timestamp,HistoryItemType,HistorySummary]" \
  --output table
```

---

## Part 5 — Composite Dashboard (putting it all together)

Your `AIDocProcessor-dev` dashboard should now look like this:

```
┌──────────────────────────┬──────────────────────────┐
│  Invocations vs Errors   │  Duration p95 (ms)        │
│  [Line chart]            │  [Line + annotation 5min] │
├──────────────────────────┼──────────────────────────┤
│  Invoice Pipeline        │  Agent Tool Invocations   │
│  Throughput [Bar chart]  │  [Bar chart]              │
├──────────────────────────┼──────────────────────────┤
│  Cold Starts [Number]    │  Alarm Status [Alarm]     │
├──────────────────────────┴──────────────────────────┤
│  Recent Errors           [Log table — Log Insights]  │
├─────────────────────────────────────────────────────┤
│  X-Ray Service Map       [Trace map widget]          │
└─────────────────────────────────────────────────────┘
```

### Add an Alarm Status widget

1. Open the dashboard → **Add widget → Alarm status**
2. Select all three `OrchestratorContainer-dev-*` alarms
3. Title: `Pipeline Health`
4. Save the dashboard.

### Set auto-refresh

1. Click the **Refresh** dropdown (top right of dashboard) → **10 seconds**
2. The dashboard now acts as a live operations screen.

### Share the dashboard

1. Click **Actions → Share dashboard**
2. Choose **Share with everyone in your account** (no sign-in required within the account)
3. Copy the public link.

---

## Troubleshooting

### X-Ray traces not appearing

| Symptom | Cause | Fix |
|---------|-------|-----|
| Service map is empty | Active tracing not enabled | Run Step 1 of Part 1 (update-function-configuration) |
| Traces appear but no subsegments | Powertools Tracer env vars not set | Confirm `POWERTOOLS_SERVICE_NAME` is set in Lambda env |
| `Subsegment … cannot end before` errors in logs | Concurrent async calls | Ensure Powertools version ≥ 2.x |

### Custom metrics not appearing in namespace AIDocProcessor

| Symptom | Cause | Fix |
|---------|-------|-----|
| Namespace missing | Lambda never completed successfully | Check Lambda logs for errors; trigger at least one successful invocation |
| Metrics appear in logs but not in CloudWatch | `cloudwatch:PutMetricData` permission missing | CDK already grants this; confirm IAM policy is attached |
| `ColdStart` is always 0 | Lambda is warm | Deploy a new version or wait for the execution environment to be recycled |

### Log Insights returns 0 results

| Symptom | Cause | Fix |
|---------|-------|-----|
| No results for any query | Wrong log group selected | Use `/aws/lambda/OrchestratorContainer-dev` (CDK-managed) |
| Logs exist but JSON fields missing | Logs not structured (plain print) | Confirm `logger.info(...)` calls are used, not `print()` |

---

## Summary

| What you built | AWS service | Powertools feature |
|---|---|---|
| Live Lambda metrics dashboard | CloudWatch Metrics + Dashboards | `Metrics` (EMF) |
| Structured log search & queries | CloudWatch Log Insights | `Logger` |
| End-to-end distributed traces | AWS X-Ray | `Tracer` |
| Automated alerting | CloudWatch Alarms | (native CDK construct) |
| Composite ops dashboard | CloudWatch Dashboards | All of the above |

The combination of **OpenSearch** (Lab 06) for log exploration and **CloudWatch** (this lab) for metrics, alarms, and traces gives you full observability coverage across the pipeline: you can spot a spike on the CloudWatch dashboard, drill into logs in OpenSearch, and correlate with the X-Ray trace — all within the same AWS account.
