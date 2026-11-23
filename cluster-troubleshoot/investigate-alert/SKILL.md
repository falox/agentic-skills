---
name: investigate-alert
description: Investigate reported OpenShift alerts, explain their root causes, and recommend remediation when action is needed.
---

# ENVIRONMENT

- Platform: OpenShift Container Platform (OCP). Use OpenShift-specific resources (ClusterOperator, ClusterVersion, MachineConfigPool, Route, etc.) alongside standard Kubernetes ones.
- You run commands against the live cluster.
- Available CLIs: oc, kubectl, jq, wget, openssl, skopeo, python3
- Networking diagnostics: iproute (ip, ss), bind-utils (dig, nslookup, host), net-tools (netstat, ifconfig), tcpdump, lsof
- Process and system: procps-ng (ps, top, free), strace, findutils, file, diffutils
- Container images: skopeo
- General: git, less, vim, tar, gzip, unzip, diff
- Writable directories: /home/agent, /tmp/agent-workspace. The root filesystem is read-only.

# RESPONSE RULES

- Support each finding with cluster evidence. Name the resource and its namespace when applicable. Include a timestamp or an exact error message when it helps explain the cause.
- If multiple causes exist, list them numbered with supporting evidence.
- If inconclusive, say so and suggest what additional access or data would help narrow it down. Never fabricate information.
- In diagnosis mode, stay focused on the reported issue — don't surface unrelated errors.
- CRITICAL: The output remediation plan and options MUST address only the root cause of the specific alert or issue being analyzed. Never include secondary issues, unrelated findings, or general recommendations discovered during investigation.
- No URLs unless from command output or provided context.


# Investigate an alert

Investigate all reported alerts in one read-only run. Use the alert labels to find the affected resources. Group instances only when the evidence points to the same cause.

Start with the affected resource. For example, a full database connection pool can explain API errors. If the fix depends on which client holds the connections, identify that client. In the RCA, explain how the responsible resource caused the alert and cite the observation that links the two. Stop once the evidence supports that explanation and the decision on remediation.

When a change is needed, say which resource to change, how to change it, and how to verify recovery. Give the best supported option first. One option is enough, and each option must respect the user's constraints. When the evidence shows that no change is needed, explain why and return no remediation options.

## If you need metrics

Query the Thanos Querier route in `openshift-monitoring`. Run the whole example in one call. Replace the example expression with your PromQL query.

```bash
set -euo pipefail
host=$(oc -n openshift-monitoring get route thanos-querier -o jsonpath='{.spec.host}')
token=$(oc whoami -t)
query='ALERTS{alertname="KubePodCrashLooping",alertstate="firing"}'
encoded=$(jq -rn --arg q "$query" '$q | @uri')
wget -nv -O - --header="Authorization: Bearer $token" \
  "https://$host/api/v1/query?query=$encoded" |
  jq -e 'if .status == "success" then .data.result else error(.error // "Query failed") end'
```

For this alert query, an empty result means no firing instance matched at query time. If the request fails, the alert state is unknown.

If you need the firing instances for an earlier incident, query `ALERTS` at that time. Add `&time=<unix-seconds>` to the query URL, using the reported time in seconds since the Unix epoch. Use the returned labels to identify the affected resources, even if the alert has since cleared.

If the trigger is unclear, find the rule by name in the relevant `PrometheusRule` resource. Check the expression, the `for` duration if set, and the annotations. Query the expression over the incident window if the timing matters to the diagnosis. If the rule has changed since the incident, use the version from that time if available.
