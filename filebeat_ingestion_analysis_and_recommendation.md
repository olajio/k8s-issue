# Filebeat Ingestion Analysis and Safe Processor Recommendation

## 1) Analysis: Why some logs were ingested and some were not

I reconnected, read all 4 requested files, and compared the raw pod log against the Kibana export.

What I found:
1. The Filebeat config is parsing every container message as JSON.
2. The raw pod log file you shared contains only valid JSON log lines in the captured window.
3. The Kibana CSV contains two types of rows:
   - Normal parsed FIX logs (no error field).
   - JSON parse-error rows, where message is plain text (env vars, separators, file listings), not JSON.

Key evidence:
1. Raw pod log capture window is UTC 20:23:17 to 21:00:49.
2. Kibana CSV local timestamps span 06:31 to 17:20.
3. In Kibana CSV:
   - Total rows: 3464
   - Rows with parse errors: 1268
   - Rows without parse errors: 2196
4. Error clusters are at 06:31 and 07:45.
5. Clean ingestion resumes at 16:25 and continues through 17:20 (your observed pattern).

Why some logs are ingested and some are not:
1. Parsing behavior is the main differentiator:
   - Messages that are JSON-shaped get decoded and indexed cleanly.
   - Messages that are plain text trigger parsing input as JSON errors.
2. The “missing” perception is partly a time-window mismatch:
   - Raw pod file starts around 16:23 ET equivalent.
   - Clean Kibana block starts around 16:25 ET.
   - Most count differences are from logs before that 16:25 query start boundary plus edge effects.
3. There is no config rule dropping your app logs by content.
   - The only explicit drop rule is for calico-node container logs.

About the sample error:
1. It exactly matches this behavior: Filebeat attempted JSON decode on a non-JSON message and recorded an error with the original message data.

So the root cause is:
1. Unconditional JSON decode on message.
2. Mixed log formats from the pod lifecycle (structured JSON app logs plus plain text startup/runtime lines).

Recommended fix:
1. Apply JSON decoding only when message looks like JSON.
2. Keep non-JSON lines as plain messages instead of forcing JSON parse.
3. Optionally split inputs by container or stream if only one source is guaranteed JSON.
4. Re-run Kibana query with aligned time range and timezone when comparing against UTC pod dumps.

---

## 2) Follow-up: Safe Filebeat processor block + non-JSON examples

Use this processor block under your container input so JSON logs decode, while plain-text lines stay indexed as-is with no JSON parse errors.

```yaml
filebeat.inputs:
  - type: container
    paths:
      - /var/log/containers/*.log
    processors:
      - add_kubernetes_metadata:
          in_cluster: true
          host: ${NODE_NAME}
          matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

      # Keep original raw line for troubleshooting/comparison
      - copy_fields:
          fields:
            - from: message
              to: event.original
          fail_on_error: false
          ignore_missing: true

      # Decode only when the line looks like a JSON object
      - decode_json_fields:
          when:
            regexp:
              message: '^\s*\{.*\}\s*$'
          fields: ["message"]
          target: ""
          overwrite_keys: true
          add_error_key: false
          process_array: false
          max_depth: 3
```

Why this is safe:
1. Non-JSON lines do not hit decode_json_fields, so they are still indexed as plain message.
2. JSON app logs still decode into structured fields.
3. event.original preserves the unmodified line for audits and comparison.

A few non-JSON line examples from your data:
1. FIXPR_TSTVPROD_SVC_PORT_5100_TCP_PROTO=tcp
2. Files in directory: /home/hs-caviar-microservices/fixgateway/AWSSDK.SQS.dll
3. --------------------------------------
4. HOSTNAME=fixgw-tstvprod-5c46474fdb-4btpq
5. IMAGE_NAME=469620122115.dkr.ecr.us-east-2.amazonaws.com/core-shd-ci-us-east-2-ecr-hs-caviar-microservices:1.257.0

Optional stricter approach:
You can further restrict JSON decoding to only your target app container name, leaving all other containers untouched.
