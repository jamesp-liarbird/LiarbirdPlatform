# Deployment shapes

The combinations the platform ships in, and which of them are settled. Every ADR is tested against these: a design that holds in one shape and not another is not finished.

A shape is two independent things — **who operates the deployment**, and **how many tenants it holds**. Carrying both on a single value is what left the third combination unnamed.

## The matrix

| | Single-tenant | Multi-tenant |
|---|---|---|
| **SaaS** — Liarbird-hosted | not a distinct shape | **Settled** |
| **Self-hosted** — customer-operated appliance | **Settled** | **Open** |

**SaaS multi-tenant** and **self-hosted single-tenant** are the two named in the standing constraints. Both are first-class; neither is a degraded mode of the other.

**Self-hosted multi-tenant** is the MSSP shape — a customer-operated appliance holding several downstream tenants. Whether it is in scope for the rebuild is unresolved, and until it is, an ADR that assumes a self-hosted deployment has exactly one tenant is resting on an open question rather than on a constraint.

**SaaS single-tenant** is not a distinct shape: a SaaS deployment holding one tenant is an early SaaS deployment, not a different product. A dedicated-instance tier would occupy this cell if one is ever offered.

## The open cell

Self-hosted multi-tenant is reachable in the proof-of-concept today — the appliance chart carries `multiTenant.enabled` and the appliance licence carries `multi_tenant_enabled` / `max_tenants`, so turning it on is a values flip and a licence rather than a code change. It is also commercially defined: AccountManagement models MSSP as a deployment model, and the MSSP console is a committed epic.

So the question is not whether to build it. It is whether the rebuild carries it, and the answer changes what "self-hosted" is allowed to assume. Evidence, and the decisions that depend on it, are in [`investigations/2026-08-06-self-hosted-multi-tenancy.md`](../investigations/2026-08-06-self-hosted-multi-tenancy.md).

## Residency

Customer data is held in Australia/New Zealand. Self-hosted is the only shape that satisfies a residency requirement elsewhere, because it places the whole stack — tenant data, control plane and identity — in the customer's own jurisdiction. On SaaS, per-tenant storage placement reaches the tenant plane but not the control plane, so a tenant under another jurisdiction's requirement is an open question rather than a region setting.

## Storage mode

Independent of the matrix above, a deployment's database is either containerised or an external customer-managed instance. External mode is only combinable with a single-tenant deployment — provisioning tenants on a customer's own instance requires `CREATEDB`, which enterprise DBAs routinely refuse.

## Related

- The proof-of-concept's tenancy modes as implemented — `liarbird-docs/internal/how-it-works/multi-tenant-isolation.md` §2
- Which packaging artifacts are live in the proof-of-concept — [`investigations/2026-07-31-deployment-shapes-audit.md`](../investigations/2026-07-31-deployment-shapes-audit.md)
- Storage topology per shape — §2.2 of [`adr/control-plane-tenant-plane-separation.md`](../adr/control-plane-tenant-plane-separation.md)
