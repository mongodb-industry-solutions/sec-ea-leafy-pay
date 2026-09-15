# Notes for Production

Findings from the lab deployment of MCK + Ops Manager + a 3-member TLS/auth replica set on OpenShift, exposed externally via per-member LoadBalancers and replica set horizons. Everything below works for the lab as-is — these are the items to revisit before treating this as a production-representative setup.

## Critical (security)

1. **Manifests use placeholders instead of plaintext secrets** (`add-your-strong-password`, `add-your-private-api-key`, `add-the-admin-password`, etc.) — safe to commit as-is. When moving from lab to production, populate these via a secrets manager (Vault, External Secrets Operator, Sealed Secrets, etc.) rather than hand-editing `stringData`, and as a first step when going to production, rotate/generate fresh values for the OM API key and both MongoDB user passwords rather than reusing whatever was used during lab testing.
2. **`mdb-admin` has `root` on `admin`** — appropriate for an admin user, but confirm this is the only credential with that scope. Combined with external LB access, this credential alone can fully compromise the database if leaked.

## Worth reviewing (robustness)

3. **No resource requests/limits** on any `podSpec` (operator, AppDB, replica set, Ops Manager). On a shared multi-tenant OpenShift cluster, pods without limits can get evicted under node pressure or starve neighbors. Add explicit `requests`/`limits` sized for expected load.
4. **No storage class / volume size specified** for `persistent: true` — currently relying on cluster defaults, which may allocate a small default size. Check `oc get pvc -n leafy-pay` and set explicit storage class + size for production capacity planning.
5. **Self-signed CA** (`leafy-selfsigned-ca`) — fine for this lab, but every external client needs the `ca.pem` distributed manually (via `tlsCAFile`). For production, consider a CA trusted by your client ecosystem (internal PKI or public CA) to avoid manual cert distribution.
6. **No NetworkPolicy** restricting access to the externally-exposed replica set — currently anyone with the LB hostnames + credentials can attempt a connection from anywhere. Restrict via NetworkPolicy, security groups, or a VPN/bastion instead of fully open LoadBalancers.
7. **Backup is disabled** (`backup.enabled: false` in `ops-manager/deploy-om.yaml`) — fine for a disposable lab, but a production deployment needs backups enabled and configured: point OM's backup to a blockstore/S3-compatible target, set a retention policy, enable continuous or scheduled snapshots for `replica-set`, and periodically test restores (a backup that's never been restored isn't a verified backup).

## Operational notes

- `mms.mail.*` and `mms.adminEmailAddr` are dummy `example.com` values — fine since OM email notifications aren't relied on in the lab, but real values are needed if alerting/notifications should work.
- **External exposure pattern (LB-per-member + horizons)** is the correct MCK approach for exposing a replica set outside the cluster, but has real maintenance costs at scale:
  - 3+ LoadBalancers running continuously has an ongoing cloud cost (~4 ELBs currently: 3 replica set members + 1 Ops Manager).
  - `connectivity.replicaSetHorizons` and the TLS `Certificate`'s `dnsNames` must be kept in sync manually whenever LB hostnames change (e.g., recreated LB, scaling members) — this was the source of the TLS hostname-mismatch issue hit during setup.
  - For a real automated provisioning pipeline (DBaaS-style), consider **ExternalDNS + a stable domain + wildcard certificate** (MCK's `externalAccess.externalDomain`) instead of raw ELB hostnames — this avoids the two-phase "wait for LB, then patch cert/horizons" dependency entirely and removes the need to reissue certs on every membership change.
  - If external access is only needed for internal tooling (Compass, BI tools) rather than application traffic, consider keeping the database fully internal-only and only exposing the application layer — simpler and more secure by default.
