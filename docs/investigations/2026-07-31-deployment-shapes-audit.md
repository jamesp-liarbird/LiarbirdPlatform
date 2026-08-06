# Deployment shapes audit — what is actually deployed vs stale

**Date:** 2026-07-31
**Scope:** every packaging/deployment "shape" of the Liarbird platform, cross-referenced against what real pipelines and environments consume.
**Repos examined:** LiarbirdServer (`9dd2ac96`, 2026-07-28), liarbird-infrastructure (`6caeb7c`, 2026-07-17), AgileFramework (`2fae214`, 2026-07-29).
**Method:** for each shape, three independent signals — (1) consumers: what pipelines, scripts, code, or docs reference it; (2) git recency; (3) planning status in AgileFramework (epics, sprint ledger, ADRs, known issues, GTM docs). A shape is only called *live* when a real pipeline or environment consumes it.

## TL;DR

Of the ~10 shapes on disk, **four are live**: GKE SaaS staging (fully automated), GKE SaaS prod (deliberately manual; externally live since 2026-06-28 with zero tenants), the VM appliance (real pipeline, feature-complete, not GA), and Docker Compose (the dev inner loop, but also the standing demo environment and the shape customer-facing docs point evaluators at). Everything else — compose-production, BYO-Kubernetes, the GitHub-Actions air-gap path, marketplace listings, Helm OCI chart publishing, and roughly eight Helm values overlays — is stale, parked, or planning-abandoned. Three probable defects surfaced as bycatch (§6).

## 1. Verdict summary

| Shape / artifact | Verdict | Last touched |
|---|---|---|
| `liarbird-helm` + `values-gke-saas` + `values-sni-ingress` + `values-gke-staging` → GKE staging | **Live, automated** | 2026-07-27 |
| `liarbird-helm` + `values-gke-saas` + `values-sni-ingress` + `values-gke-prod` → GKE prod | **Live, manual by design** | 2026-06-29 |
| `appliance/` (Packer + liarbirdctl + `appliance/charts/liarbird`) | **Live pipeline; product not GA** | 2026-07-26 |
| `docker-compose.yml` + `docker-compose.override.yml` | **Live** — dev inner loop, plus the standing demo host and the documented customer evaluation shape (§2.4) | 2026-07-22 |
| `release-manifest-dev.yaml` + `generate-dev-manifest.sh` | **Dev-only, healthy** | 2026-03-09 |
| `zitadel-init/` image + both chart Job templates | **Live** (shared image; appliance Job is a separate bash implementation) | 2026-07-23 |
| Air-gap Path B (`appliance/scripts/export-airgap-bundle.sh`) | **Maintained; no CI invoker** (ISSUE-347) | 2026-05-05 |
| `appliance/charts/liarbird/values-ha.yaml` | **Uncertain** — documented, no invoker; HA E2E deferred to M4 backlog | 2026-02-11 |
| `docker-compose.prod.yml` | **De facto dead** — no consumer, broken (ISSUE-387), no retirement decision | 2026-03-18 |
| `values-onprem.yaml` | **Orphan / probable defect** (§6.1) | 2026-05-03 |
| BYO-Kubernetes as a customer shape | **Abandoned** (ADR-M3-001) | — |
| Air-gap Path A (`airgap-bundle.yml`, `values-airgap.yaml`, `docs/deployment/airgap/`) | **Clearly stale** — depends on tags that don't exist; red since 2026-06-18 (ISSUE-409) | 2026-01-09 |
| `appliance/marketplace/` (AWS + Azure) | **Parked** — images boot-validated, listings drafted, never submitted; no wired build | 2026-02-24 |
| `helm-release.yml` (OCI chart publish) | **Never really exercised** — one `-test` tag ever; README advertises a different OCI path | 2025-12-18 |
| `publish-release.yml` | **Plausibly stale** — offers release types with no corresponding tags | 2025-12-18 |
| `values-gke.yaml` | **Superseded** by the saas/sni/staging layering | 2026-01-29 |
| `values-minimal.yaml`, `values-external-db.yaml`, `values-external-zitadel.yaml` | **Doc examples only** — no pipeline loads them | 2025-12-22 – 2026-01-08 |
| `values-gke-ingress.yaml`, `values-nginx-certmanager.yaml`, `values-local-zitadel-test.yaml` | **Clearly stale** — zero external references | 2025-12-22 – 2026-01-25 |
| `appliance/release-manifest.yaml` | **Stale placeholder** — fallback-only; omits forwardingrelay (§6.3) | 2026-02-19 |
| `appliance/release-manifest-staging.yaml` | **Clearly stale** — header claims consumers that don't exist | 2026-02-13 |
| `docker-compose.posthog.yml` | **Clearly stale** — zero references | 2025-12-30 |
| `EagerBeaver/docker-compose.neo4j.yml` | **Clearly stale** — legacy compose v1 syntax | 2025-10-02 |
| `appliance/liarbirdctl/k8s/liarbirdctl-service.yaml` | **Plausibly stale** — no consumer; liarbirdctl runs as host systemd | 2026-02-11 |

Note there are two independent Helm charts, not one chart with overlays: `liarbird-helm/` (chart `liarbird-platform`, umbrella with 9 `file://` subcharts, → SaaS/GKE) and `appliance/charts/liarbird` (chart `liarbird`, flat templates + ingress-nginx/CNPG/redis-operator dependencies, → appliance).

## 2. The four live shapes

### 2.1 GKE SaaS staging — the only fully-automated deploy

- Trigger `liarbird-server-staging` (push to `main`) is provisioned in Terraform: `liarbird-infrastructure/gke-saas/cloud_build.tf:98-130`, targeting cluster `liarbird-staging-cluster` in project `liarbirdplatform-staging` (`gke-saas/environments/staging.tfvars`).
- It runs `cloudbuild-staging.yaml:316-321`: `helm upgrade --install liarbird ./liarbird-helm --values values-gke-saas.yaml --values values-sni-ingress.yaml --values values-gke-staging.yaml`. This three-file layering is the only overlay set reachable from IaC-provisioned automation.
- Terraform itself installs only third-party components (cert-manager, ingress-nginx, kube-prometheus-stack, neo4j); the app deploy is entirely Cloud Build. Terraform CD (plan-on-PR / apply-on-merge / nightly drift) was validated on staging 2026-06-17 (`gke-saas/README.md:208-209`).
- Staging now runs `MULTI_TENANT_MODE=true` (validation report `AgileFramework/docs/validation/reporting/VR-2026-07-29-001.md`); the only tenant DB named anywhere is `tenant_demo_1_9b969a0e`.

### 2.2 GKE SaaS prod — live, manual by design, unfinished seams

- Prod infra (project `liarbirdplatform-prod`, cluster `liarbird-prod-cluster`, Cloud SQL REGIONAL HA) is provisioned; `liarbird.app` has been **externally live since 2026-06-28** — DNS, HTTPS, SSO (`AgileFramework/docs/prod-deployment/HANDOFF.md`).
- The app deploy path is deliberately manual: the prod project hosts no build triggers (`gke-saas/environments/prod.tfvars:157`, `enable_app_build_triggers = false`); the orchestrator hand-runs `cloudbuild-prod.yaml`, which layers `values-gke-prod.yaml` as the third overlay (`cloudbuild-prod.yaml:313-317`) and consumes images promoted by digest from the staging RC.
- Unfinished seams: prod cosign key is a placeholder that "does NOT exist yet" (`promotion/environments/prod.tfvars:50-55`); `infrastructure/prod-version.yaml` is pinned at the inert `0.0.0`; prod-version-watch off; agent-MSI serving off in both envs (`gke-saas/agent_msi.tf:19-23`); six `TODO(orchestrator)` markers in account-management prod tfvars.
- Commercially pre-launch: no provisioned prod tenant, no prod agent MSI, prod E2E validation gated on both; $0 MRR, 0 design partners, 1 unnamed PoV (`AgileFramework/gtm-plan/operating/briefings/2026-W31.md`).

### 2.3 VM appliance — real pipeline, feature-complete, not GA

- `cloudbuild-appliance.yaml` fires on `liarbird-v*` tags (triggers in `gke-saas/cloud_build.tf:226-292`, staging project only): builds liarbirdctl, wizard UI and docs; runs Packer `-only=qemu.liarbird` with `appliance/charts/liarbird`; smoke-boots the VM; cosign-signs; uploads to `gs://…-release-artifacts/vm-images/`. The account-management portal live-lists that bucket (read-only grant, `account-management/main.tf:245-256`).
- Engineering-complete: all 39 Epic-2 stories `done` in the sprint ledger, pen test 30 PASS / 0 FAIL, E2E passed (`AgileFramework/docs/sprint-artifacts/sprint-status.yaml:556-611`).
- Not GA per planning: the epic itself is still `in-progress`; the declared GA gate — story m3-2.47, a 72-hour soak test (`docs/epics.md:705-706`) — is a **draft absent from the sprint ledger entirely**; three E2E ACs (external OIDC, HA cluster join, ACME renewal) are deferred to un-started M4 epic `m4-epic-2-deferred: backlog`; ISSUE-347 ("no production CI produces customer-appliable bundles") is an open production blocker tagged "Self-hosted GA".
- Release channels: RC vs final is encoded in the version string, not a channel path; the real manifest comes from `cloudbuild-release.yaml`'s upload. Tags to date: `liarbird-v0.1.0`, `v0.1.1`, `v0.2.0-rc.1..rc.5` (latest 2026-05-31).
- AEMO is a named *prospect* with March 2026 indicative pricing (~$7,000/mo base; `AgileFramework/docs/aemo-indicative-pricing.md`), not a signed customer.

### 2.4 Docker Compose — dev inner loop, demo environment, documented evaluation shape

- The dev inner loop: current (2026-07-22), referenced throughout docs and tests; no CI job runs `docker compose`.
- The standing demo environment consumes it: host eb-syd2 "doubles as the demo environment" with uptime priority and runs the stack via `docker compose up -d` from `/opt/project/LiarbirdServer` (`AgileFramework/.claude/host/eb-syd2.md`).
- Customer-facing docs present it as a deployment shape: `AgileFramework/docs/technical-faq.md:25` lists "Docker Compose — single-server deployment for evaluation and small environments"; the recommended evaluation path starts with "Lab deployment — Docker Compose on a single server" (`technical-faq.md:370`); `AgileFramework/docs/auth/deployment/docker-compose.md` (2026-01-09) carries a "Production (with domain)" quick-start using `--profile internal-auth`.
- Consequence: repo defaults in `docker-compose.yml` / `.env.example` (credentials especially) reach demo and customer evaluation hosts, not just laptops.

## 3. Dev-only shapes (healthy)

- **Appliance dev builds** — `appliance/scripts/generate-dev-manifest.sh` + `release-manifest-dev.yaml` (the committed copy is a sample; the artifact is generated).

## 4. Parked vs abandoned — planning cross-check

Code recency alone misclassifies these; the AgileFramework evidence settles them.

| Shape | Code signal | Planning signal | Verdict |
|---|---|---|---|
| **BYO-Kubernetes (customer-managed)** | `values-onprem.yaml` maintained May 2026 | ADR-M3-001 (`docs/epics.md:30-60`) rejects shipping raw containers ("some customers are potential competitors"); GTM sells exactly two SKUs: Cloud SaaS and Self-Hosted VM; no M3/M4 story covers it | **Abandoned** as a customer shape. M2 Epic 3 (K8s deployment) is inherited capability; `values-onprem.yaml` was repurposed as the *appliance* overlay (see §6.1). `docs/technical-faq.md` (2025-12-22) still markets self-hosted clusters — stale |
| **Compose-production** (`docker-compose.prod.yml`) | No script/pipeline invokes it; untouched since 2026-03-18; only consumers are a file-existence test and a docs path filter | Validated Jan 2026 (`docs/validation/01-platform-setup.md` §2); ISSUE-387 (open) says it currently breaks (nginx restart loop with the zitadel include); **no retirement decision recorded anywhere** | **De facto dead, de jure alive.** Its SNI concern moved to `values-sni-ingress.yaml` in the same commit (`5360a6aa`). Needs an explicit retirement decision |
| **Air-gap Path A** (`.github/workflows/airgap-bundle.yml` + `liarbird-helm/values-airgap.yaml` + `docs/deployment/airgap/`) | Requires GitHub releases `helm-v$VERSION` and `agent-v$VERSION` — **neither tag prefix exists**; `values-airgap.yaml` sets `zitadelInit:`, `telemetry:`, `updateCheck:` — keys **no template reads** | ISSUE-409 (open, 2026-06-18): workflow fails on every push, "treat air-gap bundle health as unverified" | **Clearly stale.** Air-gap itself shipped in M2 Epic 5 (Dec 2025) and is the GTM "trump card", which makes the rot a risk, not just clutter |
| **Air-gap Path B** (`appliance/scripts/export-airgap-bundle.sh` + import + liarbirdctl update manager) | Maintained (2026-05-05, signing pipeline); schema-tested | ISSUE-347: no production CI produces customer-appliable bundles | **The real air-gap implementation; operator-run, invoker uncertain** |
| **AWS/Azure marketplace** (`appliance/marketplace/`) | Packer sources `source-aws.pkr.hcl` / `source-azure.pkr.hcl` never selected by any pipeline (only `qemu.liarbird` builds); `azure/package.sh` invoked by nothing | Stories m3-2.8/2.9/2.40 `done` — images build and boot on both clouds; submission explicitly deferred; no GTM milestone or date | **Parked, not abandoned** — but nothing scheduled. GCP Marketplace exists nowhere in code or planning |
| **Helm OCI chart publishing** (`helm-release.yml`) | One tag ever (`helm-v0.1.0-test`, 2025-12-18); pushes `oci://ghcr.io/liarbirdco/charts/liarbird` while `liarbird-helm/README.md` advertises `…/liarbird-platform` | No planning reference | **Never really exercised** |

## 5. Clearly stale — deletion candidates

Zero-consumer artifacts (all greps corroborated across repos):

- `liarbird-helm/values-gke-ingress.yaml`, `values-nginx-certmanager.yaml`, `values-local-zitadel-test.yaml` — zero external references.
- `liarbird-helm/values-gke.yaml` — superseded by the saas/sni/staging layering; `values-gke-staging.yaml:5,79` comments still falsely claim to layer on it (doc fix if kept).
- `docker-compose.posthog.yml` — zero references; own header says "NOT deployed to customers".
- `EagerBeaver/docker-compose.neo4j.yml` — legacy `docker-compose` v1 syntax; its driver `setup-neo4j.sh` is already documented as retired.
- `appliance/release-manifest-staging.yaml` — header claims Packer/liarbirdctl consumers; greps across Packer, liarbirdctl source, and all pipelines return zero.
- `appliance/liarbirdctl/k8s/liarbirdctl-service.yaml` — no consumer; liarbirdctl runs as a host systemd service.
- `.github/workflows/publish-release.yml` — offers `agent|server|helm|airgap` release types, three of which have no corresponding tags.
- Air-gap Path A set (workflow + values file + docs), per §4.
- `values-minimal.yaml`, `values-external-db.yaml`, `values-external-zitadel.yaml` — keep only as long as the docs citing them are kept. The external-Zitadel *capability* is live in staging/prod via `--set global.zitadel.external.*`, not via the file.
- Keep `.github/workflows/staging-deploy.yml.disabled` — its header is the best in-repo description of the live staging deploy path.

## 6. Defects found in passing

1. **`values-onprem.yaml` never reaches the appliance.** `appliance/packer/scripts/preload-charts.sh:31-36` copies it from the chart source dir, but Packer is pointed at `appliance/charts/liarbird` (`cloudbuild-appliance.yaml:550`), which contains only `values.yaml` and `values-ha.yaml` — the file lives in `liarbird-helm/`, and nothing copies it across. The wizard falls back with "on-prem values overlay missing" (`liarbirdctl/src/api/handlers/wizard.rs:1255-1259`). That overlay enables the liarbirdctl secret-sync + dashboard `hostNetwork` wiring — i.e. dashboard→liarbirdctl connectivity — and the enabling template (`onprem-liarbirdctl-secret-sync.yaml`) exists only in `liarbird-helm`, a chart the appliance doesn't ship. Confirm against a real appliance build log before acting.
2. **Zitadel pin three-way disagreement.** `cloudbuild-release.yaml:432,435` writes `v2.45.0` into the release manifest (which tag builds actually preload); in-tree appliance manifests say `v2.64.1`; compose runs `v2.64.5`.
3. **Fallback `appliance/release-manifest.yaml` omits `forwardingrelay`** while `cloudbuild-release.yaml:405-421` builds and ships it — a manual `_VERSION` appliance build would miss that image. The manifest is otherwise stale (`version: 0.1.0`, Feb 2026 digests) and the pipeline itself calls it "the stale in-tree template".

## 7. Stale statements worth correcting

- `cloudbuild-promote.yaml:27-33` — "the prod project does not exist yet"; contradicted by the 2026-06-28 handoff.
- `AgileFramework/docs/known-issues.md` ISSUE-313 — "SaaS production has not been provisioned (no committed prod.tfvars, no values-gke-prod.yaml)"; both now exist.
- `gke-saas/README.md:138` — `helm install … -f generated/values-gke-saas.yaml`; the `generated/` path is no longer produced (`helm-values.tf:4-10` says the overlay moved into LiarbirdServer).
- `AgileFramework/docs/technical-faq.md` — still markets BYO self-hosted Kubernetes (pre-dates ADR-M3-001).
- ADR-005 open question 3 — "prod project and prod release-artifacts bucket do not exist yet"; subsequently built.
- Engineering planning gap: nothing under `AgileFramework/docs` except GTM briefings and one validation report has moved since 2026-06-28 (~5 weeks as of this audit).

## 8. Implications

The production surface is **two shapes** (SaaS chart, appliance) plus a dev compose stack; the long tail is inert debris rather than live drift. Suggested order of work:

1. **Prune** — delete the §5 list; take explicit retirement decisions for `docker-compose.prod.yml` and air-gap Path A (cheap, shrinks every future decision's surface).
2. **Fix the §6 defects** — the `values-onprem` orphan first: it sits exactly on the appliance/SaaS chart boundary, and choosing where it lands (template moves into the appliance chart, or the file is copied in at build time) is a miniature version of the chart-convergence decision.
3. **Then** revisit bootstrap/chart convergence, now scoped to just two charts.
