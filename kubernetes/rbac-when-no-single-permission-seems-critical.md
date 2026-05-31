# Kubernetes RBAC: When No Single Permission Seems Critical

> **Category:** Kubernetes · Cloud Security · Lateral Movement  
> **Level:** Advanced  

---

Kubernetes RBAC is one of those controls that organizations implement and then consider done. Roles are created, service accounts are bound, and the cluster moves on. What rarely happens is a periodic review of what each service account can actually do — and more importantly, what combinations of permissions create unintended paths.

## The Starting Point

A compromised pod in a low-criticality namespace. No root, no shell, no tools. Just `/bin/sh` and the standard service account token mounted at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

With that token and the cluster API endpoint (available via environment variables), it is possible to query the Kubernetes API directly without kubectl:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  -H "Authorization: Bearer $TOKEN" \
  https://${KUBERNETES_SERVICE_HOST}/api/v1/namespaces/
```

## Enumerating Permissions

The first step after confirming API access is understanding what the current service account can do. The `SelfSubjectRulesReview` API returns exactly that:

```bash
curl -s ... -X POST \
  -H "Content-Type: application/json" \
  -d '{"apiVersion":"authorization.k8s.io/v1","kind":"SelfSubjectRulesReview","spec":{"namespace":"app-dev"}}' \
  https://.../apis/authorization.k8s.io/v1/selfsubjectrulesreviews
```

In the exercise, the current SA could:
- List ConfigMaps and Deployments — expected
- **Create Jobs** — unexpected but seemed harmless

## The Chain

The dangerous part was not the `create Jobs` permission by itself. It was the combination:

1. The SA could create Jobs in the namespace
2. Jobs can specify a `serviceAccountName` — including other service accounts in the namespace
3. Another service account (`batch-runner`) had permission to list and read Secrets
4. `batch-runner` also had access to a ConfigMap in the production namespace

By creating a Job that ran under `batch-runner`'s identity, the `batch-runner` token was accessible in that Job's pod — and with it, all permissions that `batch-runner` had.

```json
{
  "spec": {
    "template": {
      "spec": {
        "serviceAccountName": "batch-runner",
        "containers": [{"name": "r", "image": "busybox",
          "command": ["sh", "-c", "cat /var/run/secrets/kubernetes.io/serviceaccount/token"]}]
      }
    }
  }
}
```

From there: list secrets → get registry credentials → pull private image → find embedded kubeconfig in image layer → access production namespace.

## What Made Each Step Possible

| Permission | Seemed harmless | Was actually |
|---|---|---|
| `create Jobs` | Just batch processing | Allows impersonating other service accounts |
| `batch-runner` reads secrets | Needed for app config | Exposed registry credentials |
| Registry access | Standard image pull | Exposed image history with embedded kubeconfig |
| kubeconfig in image layer | Already "deleted" | Still present in layer history |

## Fixing It

**Restrict Job creation to specific service accounts:**

Use an admission webhook or OPA/Kyverno policy to prevent Jobs from specifying arbitrary `serviceAccountName` values:

```yaml
# Kyverno policy example
spec:
  rules:
  - name: restrict-serviceaccount-in-jobs
    match:
      resources:
        kinds: ["Job"]
    validate:
      message: "Jobs must use the default service account"
      pattern:
        spec:
          template:
            spec:
              serviceAccountName: "default"
```

**Audit service account permissions regularly:**

```bash
kubectl auth can-i --list \
  --as=system:serviceaccount:app-dev:batch-runner \
  -n app-dev
```

Run this for every service account in every namespace. Automate it. Make the output part of your security review cadence.

**Scan images for embedded credentials:**

Layer history survives even when files are "deleted" in subsequent layers. Make image scanning with secret detection (`trufflehog`, `trivy`) a hard gate in CI.

## The Core Insight

In Kubernetes, the risk is not in individual permissions — it is in graphs. A read permission here, a create permission there, and a service account that was provisioned too broadly last quarter combine into a path from a low-criticality pod to production secrets.

The question to ask about every service account is not "what does it do?" but "what could someone do with this token if they had it?"

---

*Related: [Case 04 — Kubernetes RBAC and Lateral Movement Between Namespaces](https://github.com/juliomauro/sec-writeups/tree/main/cases/04-kubernetes-rbac)*
