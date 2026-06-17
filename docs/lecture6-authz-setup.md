# Lecture 6 - Authorization (Keycloak Authz + OpenFGA)

This adds two fine-grained authorization PDPs and one demo backend:

- `infra/keycloak` - the `api-security` realm now defines roles
  (`document-viewer/editor/admin`), users `alice`/`bob`, and a confidential
  client `documents-api` with **Authorization Services** (resource `Document`,
  scopes view/edit/delete, role-based scope permissions).
- `infra/openfga` - an **OpenFGA** server (Zanzibar/ReBAC) backed by Postgres.
- `applications/authz-demo` - a Spring Boot service exposing the same documents
  API two ways: `/kc/**` decided by Keycloak, `/fga/**` decided by OpenFGA.

## Prerequisites (run once, before/while ArgoCD syncs)

ArgoCD will create the namespaces and workloads, but two external resources must
exist or the pods will crash-loop: the Postgres databases and the Vault secrets.

### 1. Postgres databases (shared instance on 192.168.50.10:5432)

Same Postgres that hosts the `keycloak` and `spa_token_demo` databases. Create
one database + role per new component:

```sql
-- OpenFGA datastore
CREATE ROLE openfga LOGIN PASSWORD 'Hx3ZxPxIMGOjJpxKq4TMdpG0mHnQ';
CREATE DATABASE openfga OWNER openfga;

-- authz-demo application database
CREATE ROLE authz_demo LOGIN PASSWORD 'grigUzcinCdrSAVSJW9jUv9PCwTZ';
CREATE DATABASE authz_demo OWNER authz_demo;
```

(Passwords are alphanumeric on purpose - the OpenFGA datastore URI embeds the
password, so URL-special characters are avoided.)

### 2. Vault secrets (consumed via ExternalSecret -> ESO)

KV v2 engine mounted at `secret/`. The ExternalSecrets read these paths:

```bash
vault kv put secret/api-security/openfga-db \
  username=openfga \
  password='Hx3ZxPxIMGOjJpxKq4TMdpG0mHnQ'

vault kv put secret/api-security/authz-demo-db \
  username=authz_demo \
  password='grigUzcinCdrSAVSJW9jUv9PCwTZ'
```

No client secret is stored: the demo backend asks Keycloak for decisions using
the UMA entitlement flow (`response_mode=decision`) with the caller's own access
token, so it does not need the `documents-api` client secret.

## How it deploys

1. The `infra` ApplicationSet picks up `infra/openfga` -> namespace `openfga`.
   The Deployment runs `openfga migrate` as an initContainer, then `openfga run`
   (playground disabled). Service `openfga.openfga.svc:8080` (HTTP) / `:8081`
   (gRPC). It is cluster-internal only - the OpenFGA API is unauthenticated, so
   it is intentionally not exposed via Ingress.
2. Keycloak re-imports the realm (`--import-realm`) with the new authz config.
3. The `applications` ApplicationSet picks up `applications/authz-demo`. The
   `authz-demo` image is built by the authz-demo repo CI, which bumps the tag in
   `applications/authz-demo/deployment.yaml`. On startup the app self-seeds the
   OpenFGA store, model, and demo tuples.

## Try it

```bash
# token for alice (document-admin) and bob (document-viewer)
KC=https://keycloak.192.168.50.10.nip.io/realms/api-security
get() { curl -sk -X POST $KC/protocol/openid-connect/token \
  -d grant_type=password -d client_id=spa-token-demo -d username=$1 -d password=$1 \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])'; }
ALICE=$(get alice); BOB=$(get bob)
BASE=https://authz-demo.192.168.50.10.nip.io

# Keycloak side (role/scope): bob may view, may not delete
curl -sk -H "Authorization: Bearer $BOB" $BASE/kc/documents            # 200
curl -sk -X DELETE -H "Authorization: Bearer $BOB" $BASE/kc/documents/1 # 403

# OpenFGA side (per-object): bob sees doc 1 (via team) + doc 2 (owner),
# but NOT doc 3 - changing the id does not help. BOLA prevented.
curl -sk -H "Authorization: Bearer $BOB" $BASE/fga/documents            # [1,2]
curl -sk -H "Authorization: Bearer $BOB" $BASE/fga/documents/3          # 403
```
