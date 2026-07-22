# Agentgateway JWT auth for the k8s-a2a agent

Front the kagent **A2A agent** with an **agentgateway** proxy that authenticates
every request by validating a JWT, reusing the platform's existing **`authn`**
component as the token issuer. No new IdP, no OAuth/MCP registration — just JWT
validation against the JWKS `authn` already publishes.

This is the concrete, deployed implementation of what [`../discussion.md`](../discussion.md)
described. `authn` was already migrated to RS256 + `kid` (see
[`../DEV_SETUP.md`](../DEV_SETUP.md)), so agentgateway's two hard constraints —
asymmetric signature and a `kid` header — are already satisfied.

## Architecture

```
                         AgentgatewayPolicy: authn-jwt   (Strict JWT)
                                     │ attaches to
                                     ▼
  client ──Bearer JWT──▶  Gateway (agentgateway class)  ──HTTPRoute──▶  k8s-a2a-agent:8080
                             agent-gateway, :8080                        (kagent ns)
                                     │
                                     │ remote JWKS fetch (every 5m)
                                     ▼
                          authn:8082 /.well-known/jwks.json   (krateo-system ns)
                             RS256 public key, kid=krateo-authn-key-1
```

The proxy pod/Service is auto-provisioned by the agentgateway controller
(`agentgateway-system`, v1.3.1) the moment the `Gateway` is created against the
`agentgateway` GatewayClass.

### How validation maps to what `authn` emits

| Check | Policy setting | `authn` token |
| --- | --- | --- |
| Issuer | `issuer: "krateo.io"` (exact match) | `iss: "krateo.io"` ✅ |
| Audience | `audiences` omitted → check disabled | no `aud` ✅ |
| Signature | RS256 verified against remote JWKS | RS256, `kid=krateo-authn-key-1` ✅ |
| `exp` | required by default | always emitted ✅ |
| `kid` header | must be in the JWKS | set by `authn` ✅ |
| Token location | default `Authorization: Bearer` | bearer token ✅ |

## Files

| File | Purpose |
| --- | --- |
| `00-gateway.yaml` | The `Gateway` (agentgateway class), HTTP listener on `:8080`. Provisions the proxy. |
| `01-httproute.yaml` | Routes all paths on the Gateway to the `k8s-a2a-agent` Service. |
| `02-jwt-policy.yaml` | **The JWT policy** — `Strict` mode, `krateo.io` issuer, **remote** JWKS from `authn`. |
| `03-referencegrant.yaml` | Authorizes the cross-namespace JWKS fetch (kagent → krateo-system). |
| `alt-02-jwt-policy-inline.yaml` | Alternative: **inline** JWKS (no cross-ns hop, no rotation). Use *instead of* `02` + `03`. |
| `04-optional-groups-rbac.yaml` | Optional: `Require` authorization on the `groups` claim, on top of JWT auth. |
| `VERIFY.md` | Step-by-step verification (401 without token, 200 with a valid token). |

## Deploy

Everything lives in the `kagent` namespace (the Gateway, HTTPRoute, and Policy
must be co-located — `targetRefs` has no namespace field), except the
ReferenceGrant, which must sit in the target namespace `krateo-system`.

```sh
kubectl apply -f 00-gateway.yaml \
               -f 01-httproute.yaml \
               -f 03-referencegrant.yaml \
               -f 02-jwt-policy.yaml
```

Check everything is accepted:

```sh
kubectl -n kagent get gateway agent-gateway
kubectl -n kagent get httproute k8s-a2a-agent
kubectl -n kagent get agentgatewaypolicy authn-jwt \
  -o jsonpath='{.status.ancestors[0].conditions}' | jq
```

Expect the Gateway `Programmed=True`, the HTTPRoute `ResolvedRefs=True`, and the
policy `Accepted=Valid` + `Attached`.

### Remote vs. inline JWKS

- **Remote** (`02-jwt-policy.yaml`, default): agentgateway refetches the JWKS
  from `authn` every `cacheDuration` (5m), so **key rotation is automatic**.
  Requires the ReferenceGrant because `authn` is in another namespace.
- **Inline** (`alt-02-jwt-policy-inline.yaml`): the public key is embedded in
  the policy. Fully self-contained (no ReferenceGrant), but the key must be
  re-pasted by hand if `authn` rotates it. Apply *one or the other* — both use
  the name `authn-jwt`.

## Verify

See [`VERIFY.md`](./VERIFY.md). Summary of what was confirmed against the live
cluster:

- No token → `401 authentication failure: no bearer token found`
- Malformed token → `401 ... token header is malformed`
- Valid RS256 token from `authn`'s key (kid `krateo-authn-key-1`, iss
  `krateo.io`) → `200` and the real agent card is returned.

## Cleanup

```sh
kubectl delete -f 02-jwt-policy.yaml -f 03-referencegrant.yaml \
               -f 01-httproute.yaml -f 00-gateway.yaml
```
