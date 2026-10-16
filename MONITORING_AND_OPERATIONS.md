# Monitoring and Operations (Design Level)

This document explains the observability philosophy, metrics selection, critical log management, and alerting strategy for the **Todo Summary Assistant**.

---

## 1. Which Metrics We Monitor and Why

We apply Google's **Four Golden Signals** and the **RED Method** (Rate, Errors, Duration):

| Metric | Source | Why We Monitor It |
| :--- | :--- | :--- |
| **Request Rate (RPS)** | `http_server_requests_seconds_count` | Measures traffic volume and sudden traffic spikes or unexpected drops. |
| **Error Rate (HTTP 5xx / 4xx)** | `http_server_requests_seconds_count{status=~"5.."}` | Direct indicator of user-facing bugs, database failures, or external API outages. |
| **Latency (P95 / P99)** | `http_server_requests_seconds_bucket` | Measures user experience; 95% of users receive responses faster than this threshold. |
| **JVM Memory & GC** | `jvm_memory_used_bytes`, `jvm_gc_pause_seconds` | Detects memory leaks and performance degradations before the app encounters Out-Of-Memory (OOM). |
| **Database Pool (HikariCP)** | `hikaricp_connections_active`, `hikaricp_connections_idle` | Detects connection starvation or slow queries holding connections. |
| **Host CPU & Memory** | `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes` | Detects resource exhaustion on the underlying EC2 virtual machine. |
| **Disk Space Free** | `node_filesystem_avail_bytes` | Ensures system has capacity for Docker images, logs, and database files. |

---

## 2. Which Logs Are Critical

Not all logs require engineer attention. Critical logs include:

1. **Application Exceptions (`ERROR` & `FATAL`)**:
   * Database connection errors (`CommunicationsException`, `CannotCreateTransactionException`).
   * Third-party integration failures (Cohere API 401/429/500 errors, Slack Webhook failures).
2. **Security & Authentication Events**:
   * Repeated unauthorized access attempts (HTTP 401/403).
   * SSH login failures on the EC2 host (`/var/log/auth.log`).
3. **Container & System Crashes**:
   * Linux Kernel Out-Of-Memory killer invocations (`dmesg | grep -i oom`).
   * Docker daemon errors and container abnormal exit codes (`exit code != 0`).

---

## 3. Which Alerts Matter vs. Which Should NOT Alert (Avoiding Noise)

To prevent **alert fatigue**, alerts must be actionable and tied to real business impact.

### ✅ Meaningful Alerts (Page the Engineer):
1. **`ServiceDown`**: The application or health endpoint is down for > 1 minute.
2. **`HighHttpErrorRate`**: More than 5% of requests return 5xx errors over 2 minutes (indicates broken features).
3. **`LowDiskSpace`**: Root filesystem has less than 15% free space (prevents catastrophic server lockup).
4. **`HighLatency`**: P95 latency exceeds 1.5 seconds for 5 minutes (user experience degraded).

### ❌ What Should NOT Alert (Informational Only):
1. **Single 4xx client errors**: Bad requests (`400 Bad Request` or `404 Not Found`) are normal client behavior.
2. **Brief CPU spikes**: A 10-second CPU spike during application startup or garbage collection is normal; only alert if CPU remains high for > 5 continuous minutes.
3. **Routine container restarts during deployment**: Controlled restarts triggered by CI/CD should be muted during scheduled deployment windows.

---

## 4. How Operational Issues Are Detected Early

1. **Synthetic Health Probes**:
   * Prometheus polls `/actuator/health` every 15 seconds. If any dependency (database or disk) reports `DOWN`, alerts fire immediately before users complain.
2. **P95 Latency Drift**:
   * Slow database queries or API throttles show up in latency histograms before outright crashing the service.
3. **Connection Pool Saturation**:
   * If HikariCP active connections approach 90% of max pool size, a warning fires before requests begin timing out.
4. **Log Trend Analysis**:
   * Aggregating error log spikes allows catching subtle issues early before total service disruption.
