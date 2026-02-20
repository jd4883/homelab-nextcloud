# Nextcloud (Kubernetes)

Helm chart in **helm/**; Argo CD uses path **helm**. This chart wraps the [official Nextcloud Helm chart](https://github.com/nextcloud/helm) with **External Secrets** (1Password) and configures **external PostgreSQL** and **external Redis**. Deploy into the **nextcloud** namespace. **Auth:** basic (Nextcloud login); OIDC is a future step—**nginx OIDC is bypassed** for this ingress so requests go straight to Nextcloud. **TLS** by nginx; **persistence** via existing PVCs `nextcloud-html` and `nextcloud-data` (RWX for HA and to share data path with e.g. Immich). **HA:** 2 replicas, HPA 2–4, PDB minAvailable 1, sticky sessions. Image: fixed tag (32.0.6-apache).

---

## Pre-deploy summary

### DNS names (ingress + trusted domains)

| Host |
|------|
| nextcloud.expectedbehaviors.com |
| cloud.expectedbehaviors.com |
| files.expectedbehaviors.com |
| sync.expectedbehaviors.com |
| drive.expectedbehaviors.com |
| nc.expectedbehaviors.com |

Create CNAMEs (or A/AAAA) and ensure external-dns / cert-manager can manage them. All point at the same Nextcloud ingress.

### Secrets you need (1Password → External Secrets)

The chart assumes **external Postgres and Redis are available** and you **pre-stage these secrets** in 1Password. External Secrets Operator (sync wave -1) creates the K8s secrets. Use a **ClusterSecretStore** (e.g. `onepassword-connect`). Default 1Password **item titles** and **field labels** are set in values as below.

| K8s secret (target) | 1Password item (title) | Field labels (exact) | Purpose |
|---------------------|-------------------------|----------------------|---------|
| nextcloud-admin | **nextcloud-admin** | `username`, `password`, `smtp-username`, `smtp-password`, `smtp-host` | Admin login; SMTP fields can be placeholders when using nextcloud-smtp. |
| nextcloud-db | **postgresql** | `username`, `password` | Postgres user/password for database **nextcloud** (DB name fixed in values). Optional: `host`. |
| nextcloud-redis | **redis** | `password` | Redis auth (shared Redis item). |
| nextcloud-smtp | **gmail-ticket-system** | `username` (e.g. expectedbehaviors@gmail.com), `nextcloud_app_password`, `host` (e.g. smtp.gmail.com) | Gmail SMTP; enabled by default, injected via extraEnv. |

Mail “From” is from values: **fromAddress** + **domain** (e.g. `expectedbehaviors` + `gmail.com`).

### Postgres and Redis password limits

| | Max length | Restrictions |
|--|------------|--------------|
| **PostgreSQL** | **65,535 bytes** (PG 14+); **1,000 bytes** (PG 11–13) | Avoid `;` and `=` in passwords if using connection strings (client parsing). No inherent special-char limit. |
| **Redis** | **512 characters** | In config files, quotes and backslashes can be interpreted; prefer alphanumeric or URL-encode if using special chars. |

---

## Secrets to populate (1Password) — detail

Use the **exact 1Password field labels** in the “Secrets you need” table above. The ExternalSecret templates map those labels into K8s secret keys the chart expects (e.g. 1Password `username` → K8s key `nextcloud-username`). Override mappings via `externalSecrets.<name>.data.<key>.property` in values if needed.

**Gmail SMTP (gmail-ticket-system)**  
SMTP is enabled by default from 1Password item **gmail-ticket-system**: field `username` (e.g. expectedbehaviors@gmail.com), `nextcloud_app_password`, and `host` (e.g. smtp.gmail.com). extraEnv injects these from the nextcloud-smtp secret; Reloader includes nextcloud-smtp.

## Chart contents

- **Nextcloud app:** Upstream `nextcloud/nextcloud`, ingress, persistence (existingClaims), cronjob, security context. **Resources:** requests 500m/512Mi, limits 1 CPU/1Gi; cronjob 100m/128Mi–200m/256Mi.
- **HA:** 2 replicas, **HPA** min 2 / max 4 (CPU 70%, memory 80%), **PDB** minAvailable 1. **Ingress:** sticky sessions (cookie, persistent) for multi-replica. **Affinity:** soft preference for nodes with `node-role=database` (for when you add dedicated DB nodes).
- **Secrets:** External Secrets (sync wave -1) from 1Password: nextcloud-admin, nextcloud-db, nextcloud-redis, nextcloud-smtp. ClusterSecretStore required.
- **External DB/Redis:** Chart does not deploy them; use Postgres cluster in namespace **postgresql** and Redis in **redis**. One operator per cluster; create DB `nextcloud` and user, then point this chart at the cluster endpoints.
- **Reloader:** Pods reload when the four secrets change.

---

## 1. 1Password items (reference)

Item **titles** must match `externalSecrets.<name>.itemTitle` (defaults: nextcloud-admin, nextcloud-db, nextcloud-redis; optional nextcloud-smtp). Use the **field labels** from “Secrets you need” (username, password, etc.). The chart’s database name is fixed to **nextcloud** in values; only credentials (and optionally Postgres host) come from secrets.

---

## 2. values.yaml: what to replace

- **externalSecrets:** Set `itemTitle` per secret to match your 1Password item titles; set `secretStoreRef.name` if not `onepassword-connect`. Enable/disable with `externalSecrets.<name>.enabled`.
- **externalDatabase.host:** Default assumes a **Postgres cluster** (e.g. 3 nodes via operator) in namespace **postgresql**; use the cluster **read-write** endpoint (e.g. `postgresql-rw.postgresql.svc.cluster.local` for CloudNative-PG).
- **externalRedis.host:** Default assumes **Redis** in namespace **redis** (e.g. 3-node cluster shared across services). Use the service your operator exposes (e.g. `redis-master.redis.svc.cluster.local`).
- **nextcloud.nextcloud.host** and **ingress hosts:** Set to your primary domain; **trustedDomains** must include all ingress hosts (e.g. `nextcloud.expectedbehaviors.com`, `cloud.expectedbehaviors.com`, `files.expectedbehaviors.com`, `sync.expectedbehaviors.com`). TLS is not set in the chart (nginx handles it).
- **Persistence:** Chart uses **existingClaim** only. Create PVCs **nextcloud-html** and **nextcloud-data** (e.g. in Longhorn) in the nextcloud namespace and bind them before deploying; the chart will not create PVCs.

---

## 3. Argo CD

Argo CD is configured **only** in the central config: **`homelab/helm/core/argocd/terraform/argocd-config/configs/config.yaml`**. The **nextcloud** project and its Application entry are there; the application block is **commented out** until Postgres and Redis (and 1Password items) are ready. When ready, uncomment that block and apply the Argo CD config (e.g. via Terraform). There is no Application manifest in this repo.

---

## 4. When ready to deploy

1. Deploy PostgreSQL in namespace **postgresql** and Redis in namespace **redis** (or set `externalDatabase.host` / `externalRedis.host` to your service FQDNs).
2. Create database `nextcloud` and a user; add credentials to the **postgresql** 1Password item.
3. Add Redis password to the **redis** 1Password item if required.
4. Create **nextcloud-admin**, **postgresql**, **redis**, and **gmail-ticket-system** 1Password items with the exact field labels from “Secrets you need” (username, password, etc.).
5. Confirm `externalDatabase.host` and `externalRedis.host` in values (defaults assume namespaces `postgresql` and `redis`).
6. Uncomment the nextcloud application block in **config.yaml** (see §3), apply the Argo CD config, then sync—or run `helm upgrade --install nextcloud ./helm -n nextcloud -f helm/values.yaml` manually.

Render and validate:

```bash
helm template nextcloud ./helm -n nextcloud -f helm/values.yaml
```

---

## 5. Postgres and Redis: one operator, many consumers

- **Do you need a Helm chart for Postgres and for Redis?** Yes — but **one chart per operator** (e.g. one CloudNative-PG or Zalando Postgres operator, one Redis operator or Bitnami Redis chart). You do **not** instantiate the operator per app: deploy the operator once per cluster (or namespace), then create **resources** (Postgres clusters/databases, Redis instances) that multiple apps use.
- **One Redis for multiple use cases?** Yes. A single Redis instance can serve many apps. Redis has logical databases 0–15; you can use one index per app if the app supports it. Nextcloud uses the default index; point it at your shared Redis and set `externalRedis.host`. Put the (shared or per-app) password in the **nextcloud-redis** 1Password item.
- **One Postgres for multiple apps?** Yes. Create one Postgres cluster (or instance), then create a **database** `nextcloud` and a **user** with access to it. Store that user’s username/password in **nextcloud-db**. Database name is fixed to `nextcloud` in chart values.
- **Host from secret:** Optionally store Postgres host in the **nextcloud-db** item (field `host`) and set `externalSecrets.db.data.dbHost: { property: "host" }` and `existingSecret.hostKey: db-host`. Redis host is always from values (`externalRedis.host`).

---

## 6. Security and reliability

- **Chart-level:** `readOnlyRootFilesystem`, `runAsNonRoot`, fixed image tag, 1Password for all secrets. Reverse-proxy config for overwrite URL/protocol.
- **SMTP:** Prefer port **587** with **STARTTLS** (explicit upgrade from plain to TLS; modern standard and better security). Port 465 = implicit TLS (also fine). Default in this chart: 587 + STARTTLS. Gmail supports both.
- **Other improvements:** Enable 2FA and brute-force protection in the Nextcloud admin UI (Security & setup warnings). Rely on Postgres/Redis and Longhorn for durability; DB backups to be added to the chart later.
- **HA:** 2 replicas; HPA 2–4; PDB minAvailable 1; RWX for `nextcloud-data` (and `nextcloud-html`); sticky sessions. Same data path can be shared with Immich.

---

## 7. Legacy config and External Secrets vs mounts

**Old docker-kubernetes-terraform / jd4883 nextcloud:** The previous stack used file-based secrets (env vars from mounted secret files). We do **not** use External Secrets to template out **file mounts** (e.g. config.php or nginx config from 1Password). All sensitive values are injected as **K8s secrets** → env/secretKeyRef; the chart’s config snippets (overwrite.config.php, trustedDomains, etc.) stay in values. The old `config.php` / nginx / php.ini content is reflected as: trusted_domains in values, redis/db from externalRedis/externalDatabase + secrets, overwrite.cli.url in configs. No additional ExternalSecret-backed mounts are required.

**Do not** copy `instanceid`, `passwordsalt`, or `secret` from any old config — a new install generates those.

## 8. Nginx, PHP, and config (included)

- **Ingress:** `proxy-body-size: 16G`; long timeouts; `proxy-request-buffering: off`. **HA:** `affinity: cookie` and `affinity-mode: persistent` for sticky sessions across replicas. Carddav/caldav redirects and deny rules in `server-snippet`; TLS by cluster nginx.
- **PHP:** `upload_max_filesize=16G`, `post_max_size=16G`, `memory_limit=512M`, `max_execution_time=3600` (via `phpConfigs`).
- **overwrite.cli.url:** Set in `nextcloud.configs.overwrite.config.php`.

## 9. Auth: basic now, OIDC later

- **Current:** **Basic auth** only—users log in with Nextcloud’s built-in login (admin from 1Password nextcloud-admin). **Nginx OIDC is bypassed** for this ingress (no proxy auth in front of Nextcloud).
- **Future (OIDC):** When you want OIDC (e.g. GitHub): (1) **Nextcloud Social login** app—configure GitHub as OIDC in Nextcloud admin; or (2) **nginx OIDC** in front—enable global or route-specific OIDC on the ingress so auth happens at the proxy. Use the credentials you generated for either approach. Document your choice in runbooks.

## 10. Backups

- **DB backups** will be built into this Helm chart in a later iteration (e.g. CronJob or sidecar using the Postgres operator’s backup or `pg_dump`). For now, rely on your Postgres operator’s backup story or volume snapshots.

## 11. Apps: default list and how to add more

- **Default apps:** The chart defines **defaultApps** in values (`calendar`, `contacts`, `mail`, `notes`, `twofactor_totp`). A **post-install Helm hook Job** (see **templates/jobs.yaml**) runs after install/upgrade and installs each app via `occ app:install`. Edit `nextcloud.defaultApps` in values to add or remove apps; the Job runs on the next install/upgrade.
- **Adding more apps later:** (1) **Web UI:** Settings → Apps → enable/install. (2) **CLI:** `kubectl exec deploy/nextcloud -n nextcloud -- sudo -u www-data php occ app:install <appid>`. App IDs: `occ app:list` or the Apps page.

## 12. Baking in config from jd4883/nextcloud (git@github.com:jd4883/nextcloud.git)

We bake static config from your old repo into the chart via **values** (ConfigMap-style content), not as separate file mounts from 1Password. **Credentials in that repo are considered burned;** replace them with injection from 1Password (already done for admin, DB, Redis, SMTP). No secrets from the repo are mounted; only non-secret options are merged into the chart.

**How to do it:**

1. **Clone the repo:** `git clone git@github.com:jd4883/nextcloud.git` (or use a local copy).
2. **Strip credentials and machine-specifics:** Remove or do not copy: `instanceid`, `passwordsalt`, `secret`, `dbpassword` / `dbuser` / `dbhost` (we inject these), any SMTP passwords, and any other secrets. Replace those with placeholders or omit; the chart and External Secrets provide the real values.
3. **Map content into the chart:**
   - **config.php** (or equivalent): Add any **non-secret** `$CONFIG` keys as extra entries under `nextcloud.nextcloud.configs` in **helm/values.yaml**. Each file is a snippet; use a unique key (e.g. `custom.config.php` or `legacy-snippet.config.php`) and paste the PHP array entries you want. Nextcloud merges multiple config files.
   - **php.ini** (or container PHP overrides): Add directives under `nextcloud.phpConfigs` in values (e.g. `zz-legacy.ini`) so they are applied in the container.
   - **nginx** (if the repo had container nginx): We use ingress nginx, so only **location/server snippets** that still apply (e.g. extra rewrites, headers) should be merged into the ingress `server-snippet` in values. Any repo-specific nginx config that is obsolete for ingress can be skipped.
4. **Optional ConfigMap for many files:** If the repo has many static files you want mounted (e.g. multiple config snippets), create a ConfigMap from the repo (with credentials stripped) and we can add support for mounting it (e.g. `extraVolumes` / `extraVolumeMounts` if the official chart supports it, or a small wrapper). For a few snippets, using `configs` and `phpConfigs` in values is enough.

**Exact information we need from you (provide in a message, doc, or tarball with secrets redacted):**

| # | We need | Format / example |
|---|--------|-------------------|
| 1 | **Complete list of every file in the repo** | One line per file: path and filename from repo root (e.g. `config/config.php`, `nginx/site-confs/default`, `php.ini`). |
| 2 | **List of every config key that is a credential or secret** | Key names only (e.g. `instanceid`, `passwordsalt`, `secret`, `dbpassword`, `dbuser`, `mail_smtppassword`). We will not bake these; we use 1Password. |
| 3 | **Single config file or multiple fragments?** | Answer in one sentence: e.g. “One config.php” or “Three fragments: config.php, custom.config.php, overwrite.config.php”. |
| 4 | **List of any special or app-specific settings to replicate** | e.g. custom apps path, overwrite URLs, memcache backend, or other `$CONFIG` / php.ini / nginx options you want in the chart. |

Without (1)–(4) we cannot map the repo into the chart’s `configs`, `phpConfigs`, and ingress snippets. With them we can add the exact keys and optionally a ConfigMap for many files.

## 13. CI (GitHub Actions)

Same pattern as the atlantis chart: **Release on merge to main** (bump patch, tag, GitHub Release) and **Release notes from PR** (OpenAI summary). Paths under **helm/** only; add **OPENAI_API_KEY** in repo Secrets for release notes. See **.github/workflows/README.md**.
