# Verifying JWT auth on the agent gateway

These are the exact checks run against the live cluster. The gateway Service
(`kagent/agent-gateway`, `LoadBalancer`, port 8080) has no external IP in this
kind cluster, so we port-forward it.

```sh
kubectl -n kagent port-forward svc/agent-gateway 8080:8080 &
```

## 1. No token → 401

```sh
curl -i http://localhost:8080/.well-known/agent-card.json
```

```
HTTP/1.1 401 Unauthorized
content-type: text/plain

authentication failure: no bearer token found
```

## 2. Malformed token → 401

Confirms the JWT layer is actively parsing/validating, not just checking for a
header's presence.

```sh
curl -i -H "Authorization: Bearer not.a.jwt" \
  http://localhost:8080/.well-known/agent-card.json
```

```
HTTP/1.1 401 Unauthorized
authentication failure: the token header is malformed: ...
```

## 3. Valid token → 200

In production the token comes from an `authn` login (its response includes the
JWT). For a self-contained check we mint one with `authn`'s own signing key —
same key whose public half the gateway fetches from the JWKS, so a pass proves
the whole chain (issuer + `kid` + RS256 signature against the remote JWKS).

```sh
# pull authn's private signing key
kubectl -n krateo-system get secret jwt-sign-key \
  -o jsonpath='{.data.private\.pem}' | base64 -d > /tmp/private.pem

# sign a token with the claims authn emits (kid must match the JWKS)
TOKEN=$(python3 - <<'PY'
import jwt, time
key=open("/tmp/private.pem").read()
now=int(time.time())
claims={"iss":"krateo.io","sub":"verify-user","username":"verify-user",
        "groups":["admins","devs"],"iat":now,"nbf":now,"exp":now+3600}
print(jwt.encode(claims,key,algorithm="RS256",
                 headers={"kid":"krateo-authn-key-1"}))
PY
)

curl -i -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/.well-known/agent-card.json
rm -f /tmp/private.pem
```

```
HTTP/1.1 200 OK
server: uvicorn
content-type: application/json

{"capabilities":{"streaming":true},...,"name":"k8s_a2a_agent",...}
```

## 4. Watch the proxy decisions

```sh
kubectl -n kagent logs deploy/agent-gateway -f | grep -i jwt
```

Each rejected request logs `reason=JwtAuth` with the failure cause; accepted
requests log `http.status=200`.

## Notes

- The `groups: ["admins","devs"]` claim above is what the optional
  `04-optional-groups-rbac.yaml` policy keys off. With that policy applied, a
  token lacking `admins` in `groups` gets a `403` even though the JWT itself is
  valid.
- To exercise the real login path instead, obtain a token from an `authn`
  strategy (see [`../authn/README.md`](../authn/README.md)) and drop it into the
  `Authorization: Bearer` header.
