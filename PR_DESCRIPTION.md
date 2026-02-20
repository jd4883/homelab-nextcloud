# ☁️ Nextcloud chart — pod anti-affinity for HA

---

## 🎯 TL;DR

| | |
|---|---|
| **What** | Add **podAntiAffinity** (soft, preferred) so Nextcloud pods prefer to run on different nodes. Keeps replicas separated for HA. |
| **Why safe** | Soft preference only; scheduler can still co-locate if needed. Same pattern as nginx, external-secrets, reloader. |
| **Proof** | `helm template` ✅; deployment spec shows `podAntiAffinity` with `topologyKey: kubernetes.io/hostname`. |

---

## 📦 Summary

| Icon | Change |
|------|--------|
| 🌊 | **Pod anti-affinity** — `nextcloud.affinity.podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution`: prefer not to schedule nextcloud pods on the same node (`kubernetes.io/hostname`). Label selector: `app.kubernetes.io/name: nextcloud`. |

---

## ✅ Render & validation

> **Command used:**  
> `helm template nextcloud ./helm -n nextcloud -f helm/values.yaml`

| Check | Result |
|-------|--------|
| `helm template ...` | ✅ OK, no errors |
| Deployment has podAntiAffinity | ✅ Rendered spec includes podAntiAffinity with topologyKey hostname |

---

## 📋 Supporting evidence

<details>
<summary>🌊 <b>Deployment affinity — podAntiAffinity in rendered manifest</b></summary>

```yaml
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - nextcloud
              topologyKey: kubernetes.io/hostname
            weight: 100
```

</details>

---

## 🛡️ Why this change is safe & correct

| Change | What we did | Why it's safe | Proof |
|--------|-------------|---------------|--------|
| Pod anti-affinity | Added `podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution` under `nextcloud.affinity`; topologyKey `kubernetes.io/hostname`; selector `app.kubernetes.io/name: nextcloud`. | **Soft** preference only; if the cluster has few nodes or other constraints, scheduler can still place two pods on the same node. Reduces single-node failure impact when multiple nodes exist. | Rendered deployment spec above. |

---

## 🚀 Next steps

| Step | Action |
|------|--------|
| 1️⃣ | Merge; on next sync/upgrade, new pods will prefer different nodes. |
| 2️⃣ | Optional: monitor scheduling (e.g. `kubectl get pods -n nextcloud -o wide`) to confirm spread. |
