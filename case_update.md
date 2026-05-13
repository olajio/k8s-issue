Good choice — A gives Ariana more to work with and the four numbered questions make it easy for her to respond point-by-point. Here it is cleaned up and ready to paste into the case:

---

Hi Ariana,

Thanks for digging into this. Before getting to your command requests, I want to share some additional context that I think reframes the issue — it's broader than the single pod we initially flagged.

**What we're observing across the environment**

- The same behavior is occurring on **multiple pods**, not just `fixgw-tstvprod-5c46474fdb-4btpq`.
- On the affected nodes, **other pods continue ingesting logs normally** during the same windows when the affected pod goes silent — so the node, Filebeat DaemonSet, and the pipeline downstream are all healthy.
- The behavior is **intermittent**: sometimes Filebeat logs the JSON parse error and continues ingesting valid JSON from the pod; other times the same error appears to halt ingestion from that pod for several hours.
- Ingestion eventually resumes on its own — **no changes are made to the pod, the application, or Filebeat** when this happens.

**The parse error itself is expected**

The non-JSON messages (e.g. environment variable–style lines like `FIXPR_TSTVPROD_SVC_PORT_5100_TCP_PROTO=tcp`) are emitted periodically by the application and we expect `decode_json_fields` to drop them with an error like:

```
"error": {
  "data": "FIXPR_TSTVPROD_SVC_PORT_5100_TCP_PROTO=tcp",
  "field": "message",
  "message": "parsing input as JSON: invalid character 'F' looking for beginning of value",
  "type": "json"
}
```

That part is working as designed. The question is the **inconsistency in what happens next**: why does the same processor sometimes drop the offending message and keep going, and other times appear to stall valid JSON ingestion from the pod for hours?

**What we'd like your help understanding**

1. Is this a known issue or bug with `decode_json_fields` (or upstream Filebeat input handling) when malformed messages are interleaved with valid JSON?
2. Is there a documented or expected mechanism by which a parse failure could pause ingestion from a single pod / input while leaving the rest of the DaemonSet healthy?
3. Have other customers reported similar intermittent stalls that self-recover without intervention?
4. Given that ingestion resumes without any change on our side, what would you suggest we instrument to capture state *during* a stall?

**On the commands you requested**

The `kubectl describe`, `kubectl logs`, `kubectl get events`, and `kubectl get pod ... restartCount` commands require superadmin access in our cluster, which we don't have readily available. Given that the behavior is not specific to this one pod, would it be possible to drive the initial investigation from the **`filebeat_logging.yaml`** config I shared earlier and from the Filebeat-side logs / metrics? If you can identify specific Filebeat log lines or metrics that would correlate with the stall windows, we can pull those directly. If superadmin-level pod data ultimately becomes necessary, we can request that access — it'll just take a bit more coordination on our end.

Happy to jump on a call if it would speed this up.

Thanks,
Ola

---

Want me to tweak anything before you send — tone, add a specific time you're available for a call, or push harder on a particular question?
