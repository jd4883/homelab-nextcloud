# [nextcloud] — [Short title]

---

## TL;DR

| | |
|---|---|
| **What** | One-line summary of the change. |
| **Why safe** | One-line: why this is safe (e.g. soft affinity only, no breaking change). |
| **Proof** | One-line: how you verified (e.g. `helm template` OK). |

---

## Summary

| Icon | Change |
|------|--------|
| | **Change 1** — Short description. |

---

## Setup requirements (if any)

*(Omit if none.)*

---

## Render & validation

> **Command used:**  
> `helm template nextcloud ./helm -n nextcloud -f helm/values.yaml`

| Check | Result |
|-------|--------|
| `helm template ...` | OK / failed |

---

## Supporting evidence

<details>
<summary>Relevant snippet</summary>

```yaml
# Rendered YAML excerpt.
```

</details>

---

## Why this change is safe & correct

| Change | What we did | Why it's safe | Proof |
|--------|--------------|---------------|-------|
| | | | |

---

## Next steps

| Step | Action |
|------|--------|
| 1 | Merge; *(what happens next)*. |

---

**Checklist**

- [ ] Title starts with `[nextcloud]`.
- [ ] Chart version bumped in `helm/Chart.yaml` if applicable (semver).
- [ ] `helm dependency update` and `helm template` succeed in `helm/`.
