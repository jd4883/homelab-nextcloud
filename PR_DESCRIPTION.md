# ☁️ Nextcloud chart — STARTTLS, jobs.yaml, README clarity & jd4883 asks

---

## 🎯 TL;DR

| | |
|---|---|
| **What** | SMTP default 587/STARTTLS (preferred security). Install hook moved to **templates/jobs.yaml**. README §12: blunt table of exact info we need from jd4883/nextcloud. |
| **Why safe** | Config-only (port/secure); Job logic unchanged, file rename + label alignment; README is documentation only. |
| **Proof** | `helm template` ✅; Job renders from jobs.yaml; SMTP and README edits in repo. |

---

## 📦 Summary

| Icon | Change |
|------|--------|
| 🔐 | **SMTP:** Default port **587**, `secure: tls` (STARTTLS). README states 587/STARTTLS is preferred over 465/ssl. |
| 📄 | **Jobs:** Post-install install-default-apps Job moved from **install-default-apps-job.yaml** → **jobs.yaml**; label `app.kubernetes.io/component: install-default-apps`; aligned with other chart templates. |
| 📖 | **README §12:** Replaced vague “what to send” with a **table of exact asks**: (1) complete file list, (2) list of credential keys, (3) single vs multiple config, (4) special/app settings. No ambiguity. |

---

## ✅ Render & validation

> **Command used:**  
> `helm template nextcloud ./helm -n nextcloud -f helm/values.yaml`

| Check | Result |
|-------|--------|
| `helm dependency update helm` | ✅ OK |
| `helm template ...` | ✅ OK, no errors |
| Job from **jobs.yaml** | ✅ Renders `nextcloud-install-default-apps` with hook annotations and default apps loop |

---

## 📋 Supporting evidence

<details>
<summary>🔐 <b>SMTP 587 / STARTTLS in values</b></summary>

```yaml
# helm/values.yaml (excerpt)
    smtp:
      port: 587
      secure: tls   # STARTTLS (preferred over 465/ssl)
      authtype: LOGIN
```

</details>

<details>
<summary>📄 <b>Job from templates/jobs.yaml</b></summary>

```yaml
# Source: nextcloud/templates/jobs.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nextcloud-install-default-apps
  namespace: nextcloud
  labels:
    app.kubernetes.io/name: nextcloud
    app.kubernetes.io/instance: nextcloud
    app.kubernetes.io/component: install-default-apps
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-weight: "5"
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  ttlSecondsAfterFinished: 300
  backoffLimit: 3
  ...
      containers:
        - name: install-apps
          ...
            - |
              set -e
              for app in calendar contacts mail notes twofactor_totp; do
                echo "Installing app: $app"
                php /var/www/html/occ app:install "$app" || true
              done
```

</details>

<details>
<summary>📖 <b>README §12 — exact asks table</b></summary>

| # | We need | Format / example |
|---|--------|-------------------|
| 1 | **Complete list of every file in the repo** | One line per file: path and filename from repo root. |
| 2 | **List of every config key that is a credential or secret** | Key names only. We will not bake these; we use 1Password. |
| 3 | **Single config file or multiple fragments?** | Answer in one sentence. |
| 4 | **List of any special or app-specific settings to replicate** | e.g. custom apps path, overwrite URLs, memcache, etc. |

</details>

---

## 🛡️ Why each change is safe & correct

| # | Change | What we did | Why it's safe | Proof |
|---|--------|-------------|---------------|--------|
| 1 | SMTP 587/STARTTLS | values: `port: 587`, `secure: tls`; README recommends 587/STARTTLS. | Gmail supports 587; STARTTLS is standard; no secret or app logic change. | values + README §6. |
| 2 | jobs.yaml | Renamed **install-default-apps-job.yaml** → **jobs.yaml**; added component label `install-default-apps`. | Same Job spec; Helm treats any template file the same; labels align with HPA/PDB. | Rendered Job shows correct name and hook. |
| 3 | README §12 | Replaced prose with numbered table of exact asks (file list, credential keys, config layout, special settings). | Documentation only; unblocks jd4883 wiring. | README diff. |

---

## 🚀 Next steps

| Step | Action |
|------|--------|
| 1️⃣ | Merge; on upgrade, SMTP will use 587/STARTTLS (restart or rollout if needed for env to take effect). |
| 2️⃣ | Get (1)–(4) from jd4883/nextcloud and add configs/phpConfigs/ingress snippets. |
| 3️⃣ | Optional: add more Jobs to **jobs.yaml** (e.g. post-upgrade maintenance) under the same `{{- if }}` or new blocks. |
