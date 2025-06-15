# Designing Logs for Better Observability and Faster Troubleshooting

## System Architecture

The overall system architecture is composed of three main parts:
![logging-system-diagram.png](pics/logging-system-diagram.png)

**Part 1: Upstream Services and Snapshot Generation**  
This layer includes “Service 1”, “Service 2”, “Service 3”, and “Service 4”, all responsible for handling end-user requests. These requests typically represent business transactions, such as purchasing a book or subscribing to a service.  
Each transaction is sent to the **Transaction Snapshot Service** for further processing via ActiveMQ, using the message type `"TransactionGenerated"`.  
Processed transaction data — such as `accountId`, `transactionType`, `transactionDate`, `revenueAmount`, and `transactionPreference` — is then streamed to the downstream processor via Kafka.

**Part 2: Transaction Snapshot Processor**  
This microservice listens to Kafka topics, ingests transaction snapshots, parses them, and inserts the parsed data into the downstream service’s database.

**Part 3: Downstream Services**  
This layer consists of the data store where the **Data Ingestion Service** writes incoming transaction data. It also supports the web UI that end users interact with.

## Logging Format Design Principles

When designing your logging format, focus on the system's key operations, anticipate potential issues, and think carefully about what needs to be monitored.

### A Counterexample

I’ve seen cases where engineers, without a clear design principle, ended up adding verbose, low-value logs, similar to local debugging outputs, such as:

| Time | Message |
| --- | --- |
| 04/19/2025 18:06:47.152 | Transaction Event execution time: 20 |
| 04/19/2025 18:06:47.152 | Time taken to make insertion: 17 |
| 04/19/2025 18:06:47.147 | Time taken to make db insertion: 11 TxnSource : ORDER |
| 04/19/2025 18:06:47.135 | Time taken to getConnection: 3 |
| 04/19/2025 18:06:47.135 | after getting db connection  |
| ... | ... |

**Problems with this approach:**
- Floods the system with excessive logs
- Degrades query performance
- Makes it harder to efficiently locate relevant information during incidents

---

### Key Logging Design Principles

- **Monitoring requirements should drive the structure and content of your logs:**  
  Otherwise, it becomes extremely difficult to leverage them effectively during troubleshooting.

- **Use a consistent log format:**  
  Ensure fields like event code, event duration, labels, and log levels have consistent meanings across all modules. Otherwise, during an incident, you'll waste time searching through unfamiliar or inconsistent logs.

- **Leverage index fields for faster querying:**  
  Especially when using tools like Kibana, good indexing is critical for loading dashboards efficiently. Relying solely on full-text search over several months of data can easily cause timeouts.

## Key Problems and Questions to Address

The **core guarantee** of this system is that every transaction generated upstream must be reliably persisted in the downstream database — **without data loss**.

This leads to two critical questions:
1. **How can we detect if data loss has occurred?**
2. **If a customer reports missing data, how can we quickly identify at which stage the data was lost?**

Given the system’s asynchronous nature, we must monitor:
- Overall (end-to-end) latency
- Stage-by-stage latency
- Error occurrences and types

Additionally, we need to organize errors smartly to decide on proper responses.

## Designing a Logging Format to Solve These Problems

### 1. How can we detect if data loss has occurred?

- Log the transaction ID immediately when a transaction is generated.
- Log the transaction ID again after successful insertion into the database.  
  By comparing the two sets of IDs, we can easily identify any lost transactions.

### 2. How can we quickly identify at which stage data was lost?

- At each processing stage, log:
    - The transaction ID
    - The processing outcome (success/failure)
- Implement a **Trace ID** that spans all stages to chain related activities together.  
  With a Trace ID, we can filter logs to easily see the full lifecycle of a transaction.

### 3. How can we monitor system latency (overall and per stage)?

- **Within Transaction Snapshot Service:**  
  Break down snapshot generation into phases (e.g., context preparation, discount calculation, revenue calculation, Kafka publish).  
  Use a `ThreadLocal` to capture latency per phase.

- **End-to-End Latency:**  
  Capture the timestamp when a transaction is generated and when it is inserted downstream. The difference gives the overall latency.

### 4. How should we monitor and organize errors?

Errors typically fall into three categories:
- **Neglectable:** Caused by user misconfiguration; no action needed.
- **Retriable:** Temporary issues like DB lock contention; can be automatically retried.
- **Non-retriable:** Critical issues requiring engineering investigation.

Standardize error logs with fields like `log.level` and `event.severity`:
- **Non-retriable error:** `level=error`, `event.severity=error (3)`
- **Retriable error:** `level=error`, `event.severity=warn (4)`
- **Neglectable error:** `level=warn`, `event.severity=notice (5)`


## Final Thoughts

Designing good logging is not just about "what happened" — it's about enabling observability, rapid troubleshooting, and ultimately ensuring system reliability.  
Start by defining **what you need to monitor** — then build your logs around that.
