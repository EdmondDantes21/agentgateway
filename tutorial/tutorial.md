# Tutorial — JWT-authenticated A2A agents behind agentgateway

Protect kagent's A2A agents with agentgateway, using Krateo `authn` as the JWT
issuer. Follow the steps in order.

## 1. Install agentgateway

Deploy the Kubernetes Gateway API CRDs.

```sh
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```

Deploy the agentgateway control-plane CRDs with Helm.

```sh
helm upgrade -i agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds \
  --create-namespace --namespace agentgateway-system \
  --version v1.3.1 \
  --set controller.image.pullPolicy=Always
```

Install the agentgateway control plane.

```sh
helm upgrade -i agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
  --namespace agentgateway-system \
  --version v1.3.1 \
  --set controller.image.pullPolicy=Always \
  --wait
```

Check the control plane is running.

```sh
kubectl get pods -n agentgateway-system
```

Expected:

```
NAME                            READY   STATUS    RESTARTS   AGE
agentgateway-5495d98459-46dpk   1/1     Running   0          19s
```

## 2. Install kagent

Install the kagent CRDs.

```sh
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --namespace kagent \
  --create-namespace
```

Set your LLM provider API key.

```sh
export GEMINI_API_KEY="your-api-key-here"
```

Install kagent.

```sh
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --namespace kagent \
  --set providers.default=gemini \
  --set providers.gemini.apiKey=$GEMINI_API_KEY
```

Optionally, port-forward the kagent UI.

```sh
kubectl port-forward -n kagent svc/kagent-ui 8080:8080
```

Delete the default agents. We create our own.

```sh
kubectl delete agents.kagent.dev -n kagent --all
```

## 3. Krateo prerequisite

This tutorial needs **Krateo 2.7** already installed. If it is not, follow the
install guide first:
<https://docs.krateo.io/2.7.0/how-to-guides/install-krateo/installing-krateo-kind>

## 4. Reinstall authn and snowplow

We reinstall `authn` and `snowplow` with the dev images and RS256 JWT wiring.

Uninstall the old ones.

```sh
helm uninstall -n krateo-system authn snowplow
```

Generate the keypair.

```sh
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

Create the Secrets.

```sh
kubectl create namespace krateo-system   # if needed

# authn — the private signing key
kubectl create secret generic jwt-sign-key -n krateo-system \
  --from-file=private.pem=./private.pem

# snowplow — authn's public verification key
kubectl create secret generic authn-jwt-public-key -n krateo-system \
  --from-file=public.pem=./public.pem
```

Install the charts.

```sh
helm install authn ./authn-chart/chart -n krateo-system \
  -f ./authn-chart/chart/values.dev.yaml

helm install snowplow ./snowplow-chart/chart -n krateo-system \
  -f ./snowplow-chart/chart/values.dev.yaml
```

Reinstall the portal chart.

```sh
helm upgrade portal marketplace/portal -n krateo-system --version 1.2.0
```

Verify authn is up and publishing its JWKS.

```sh
kubectl -n krateo-system get pods
kubectl -n krateo-system port-forward svc/authn 8082:8082
curl -s http://localhost:8082/.well-known/jwks.json | jq
```

The JWKS should list one RSA key with `"kid": "krateo-authn-key-1"`.

## 5. Create the A2A agent

Create a ModelConfig for Gemini 3.6 Flash.

```sh
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: gemini-gemini-3-6-flash
  namespace: kagent
spec:
  apiKeySecret: kagent-gemini
  apiKeySecretKey: GOOGLE_API_KEY
  gemini: {}
  model: gemini-3.6-flash
  provider: Gemini
EOF
```

Create the `k8s-a2a-agent` — an A2A agent with Kubernetes tools.

```sh
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: k8s-a2a-agent
  namespace: kagent
spec:
  description: An example A2A agent that knows how to use Kubernetes tools.
  type: Declarative
  declarative:
    modelConfig: gemini-gemini-3-6-flash
    systemMessage: |-
        You are an expert Kubernetes agent that uses tools to help users.
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          kind: RemoteMCPServer
          toolNames:
          - k8s_get_resources
          - k8s_get_available_api_resources
          - k8s_apply_manifest
    a2aConfig:
      skills:
      - id: get-resources-skill
        name: Get Resources
        description: Get resources in the Kubernetes cluster
        inputModes:
        - text
        outputModes:
        - text
        tags:
        - k8s
        - resources
        examples:
        - "Get all resources in the Kubernetes cluster"
        - "Get the pods in the default namespace"
        - "Get the services in the istio-system namespace"
        - "Get the deployments in the istio-system namespace"
        - "Get the jobs in the istio-system namespace"
        - "Get the cronjobs in the istio-system namespace"
        - "Get the statefulsets in the istio-system namespace"
EOF
```

Create the `helm-a2a-agent` — an A2A agent with Helm tools, using the same model.

```sh
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: helm-a2a-agent
  namespace: kagent
spec:
  description: An example A2A agent that knows how to use Helm tools.
  type: Declarative
  declarative:
    modelConfig: gemini-gemini-3-6-flash
    systemMessage: |-
        You are an expert Helm agent that uses tools to help users.
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          kind: RemoteMCPServer
          toolNames:
          - helm_list_releases
          - helm_get_release
    a2aConfig:
      skills:
      - id: list-releases-skill
        name: List Releases
        description: List Helm releases in the Kubernetes cluster
        inputModes:
        - text
        outputModes:
        - text
        tags:
        - helm
        - releases
        examples:
        - "List all Helm releases in the cluster"
        - "List the Helm releases in the default namespace"
        - "Get the details of the kagent release"
EOF
```

## 6. Apply the gateway resources

Apply each resource in order. These mirror the files in [`../`](../).

Create the `Gateway`. The controller auto-provisions the proxy Deployment +
Service in `kagent`. All agent traffic flows through it, port 8080.

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agent-gateway
  namespace: kagent
spec:
  gatewayClassName: agentgateway
  listeners:
    - name: http
      protocol: HTTP
      port: 8080
      allowedRoutes:
        namespaces:
          from: Same
EOF
```

Create the `HTTPRoute`, forwarding every path to the `kagent-controller`
backend. The controller (not the agent pods) serves every agent's A2A endpoint
at `/api/a2a/<ns>/<agent>/...`, so all agents are reached through this one route.

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kagent-a2a
  namespace: kagent
spec:
  parentRefs:
    - name: agent-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: kagent-controller
          port: 8083
EOF
```

Create the JWT policy — **Strict** auth that fetches authn's JWKS remotely
(issuer `krateo.io`, RS256). No token → 401.

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: authn-jwt
  namespace: kagent
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agent-gateway
  traffic:
    jwtAuthentication:
      mode: Strict
      providers:
        - issuer: "krateo.io"
          jwks:
            remote:
              jwksPath: /.well-known/jwks.json
              cacheDuration: 5m
              backendRef:
                group: ""
                kind: Service
                name: authn
                namespace: krateo-system
                port: 8082
EOF
```

Create the `ReferenceGrant`, letting the policy in `kagent` read the `authn`
Service in `krateo-system` (cross-namespace refs are denied by default).

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: kagent-jwt-to-authn
  namespace: krateo-system
spec:
  from:
    - group: agentgateway.dev
      kind: AgentgatewayPolicy
      namespace: kagent
  to:
    - group: ""
      kind: Service
      name: authn
EOF
```

Create the per-user, per-agent authorization policy. On top of the JWT layer,
the `sub` claim plus the request path decide which agent a caller may reach.
Non-matching requests get 403.

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: authn-a2a-user-rbac
  namespace: kagent
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agent-gateway
  traffic:
    authorization:
      action: Allow
      policy:
        matchExpressions:
          - >-
            (request.path.startsWith("/api/a2a/kagent/k8s-a2a-agent") &&
             (jwt.sub == "admin" || jwt.sub == "cyberjoker"))
            ||
            (request.path.startsWith("/api/a2a/kagent/helm-a2a-agent") &&
             jwt.sub == "admin")
EOF
```

## 7. Get tokens

Port-forward `authn` in a separate terminal window.

```sh
kubectl port-forward -n krateo-system svc/authn 8082:8082
```

Get a token for each user.

```sh
JWT_ADMIN=$(curl -s 'http://localhost:8082/basic/login' \
  -H "Authorization: Basic $(echo -n "admin:$(kubectl get secret admin-password -n krateo-system -o jsonpath='{.data.password}' | base64 -d)" | base64)" \
  --insecure | jq -r .accessToken)

JWT_CYBERJOKER=$(curl -s 'http://localhost:8082/basic/login' \
  -H "Authorization: Basic $(echo -n "cyberjoker:$(kubectl get secret cyberjoker-password -n krateo-system -o jsonpath='{.data.password}' | base64 -d)" | base64)" \
  --insecure | jq -r .accessToken)
```

## 8. Verify

Port-forward the gateway in a separate terminal window.

```sh
kubectl -n kagent port-forward svc/agent-gateway 8080:8080
```

No token → 401.

```sh
curl -i http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json
# HTTP/1.1 401 Unauthorized — authentication failure: no bearer token found
```

Valid token → 200.

```sh
curl -s -i -H "Authorization: Bearer $JWT_ADMIN" \
  localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json
# HTTP/1.1 200 OK
```

Per-user RBAC. `admin` may reach **both** agents.

```sh
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_ADMIN" \
  localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json    # 200
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_ADMIN" \
  localhost:8080/api/a2a/kagent/helm-a2a-agent/.well-known/agent-card.json   # 200
```

`cyberjoker` may reach **only** `k8s-a2a-agent`; `helm-a2a-agent` is denied.

```sh
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_CYBERJOKER" \
  localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json    # 200
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_CYBERJOKER" \
  localhost:8080/api/a2a/kagent/helm-a2a-agent/.well-known/agent-card.json   # 403
```

## 9. Gate an MCP tool per user

Now we gate at the **tool** level: `admin` may run the mutating `k8s_apply_manifest`; `cyberjoker` may talk to the same agent but not apply manifests.

This works because the token rides the whole path — `curl` → gateway (validates JWT) → controller (reads `jwt.sub`) → agent → gateway `/mcp` (re-validates, runs tool RBAC) — and four settings keep it alive across every hop:

- `backend.auth.passthrough` — stops the gateway from stripping the token after it validates it (the default), so the controller and agent actually receive it.
- `controller.auth.mode=trusted-proxy` — makes the controller trust and forward the caller's JWT instead of replacing it with a static `admin@kagent.dev`.
- `proxy.url` — routes all agent egress through the gateway.
- `KAGENT_PROPAGATE_TOKEN=true` — makes the agent re-attach the user's JWT on its outbound MCP call.

### 9.1 Route agent egress through the gateway and trust the JWT

`proxy.url` sends all agent egress through the gateway; `controller.auth.mode=trusted-proxy` makes the controller derive identity from `jwt.sub` and forward the token onward.

```sh
helm upgrade kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
    --namespace kagent \
    --reuse-values \
    --set controller.auth.mode=trusted-proxy \
    --set controller.auth.userIdClaim=sub \
    --set proxy.url=http://agent-gateway.kagent.svc.cluster.local:8080
kubectl -n kagent rollout restart deploy/kagent-controller
```

Create the token-passthrough policy so the gateway keeps the validated `Authorization` header when proxying to the `kagent-controller` Service.

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: authn-token-passthrough
  namespace: kagent
spec:
  targetRefs:
    - group: ""
      kind: Service
      name: kagent-controller
  backend:
    auth:
      passthrough: {}
EOF
```

> Sanity check: `curl` the gateway at `/api/a2a/kagent/k8s-a2a-agent/` with a valid `admin` token should return `200` — a `401` with `jwt.sub=admin` logged means passthrough isn't attached.

> Caveat: `trusted-proxy` trusts anything reaching the controller, but `kagent-controller:8083` is still directly reachable in-cluster — lock it down (NetworkPolicy/mTLS) for production.

### 9.2 Forward the user token from the agent

Patch `k8s-a2a-agent` to forward the incoming user JWT onto its outbound MCP calls (all-or-nothing per pod).

```sh
kubectl patch agent k8s-a2a-agent -n kagent --type merge -p \
  '{"spec":{"declarative":{"deployment":{"env":[{"name":"KAGENT_PROPAGATE_TOKEN","value":"true"}]}}}}'
```

### 9.3 Front the tool server with the gateway

Create the `AgentgatewayBackend` so the gateway speaks MCP (StreamableHTTP) to the `kagent-tools` Service on `:8084/mcp`.

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: kagent-tools-mcp
  namespace: kagent
spec:
  mcp:
    targets:
      - name: kagent-tools
        static:
          backendRef:
            name: kagent-tools
          port: 8084
          protocol: StreamableHTTP
          path: /mcp
EOF
```

Create the `HTTPRoute`, routing proxied MCP calls (path `/mcp`, stamped
`x-kagent-host: kagent-tools.kagent`) to that backend.

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kagent-tools-mcp
  namespace: kagent
spec:
  parentRefs:
    - name: agent-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /mcp
          headers:
            - name: x-kagent-host
              value: kagent-tools.kagent
      backendRefs:
        - name: kagent-tools-mcp
          group: agentgateway.dev
          kind: AgentgatewayBackend
EOF
```

> If MCP calls 404 at the gateway, the `x-kagent-host` value differs in your
> build. Confirm it: `kubectl -n kagent logs deploy/agent-gateway -f | grep -i x-kagent-host`.

### 9.4 Let proxied MCP calls past the A2A allowlist

Re-apply `authn-a2a-user-rbac` with a `/mcp` clause so it stops `403`-ing the proxied tool calls (the real per-tool gating happens in the backend policy below).

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: authn-a2a-user-rbac
  namespace: kagent
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agent-gateway
  traffic:
    authorization:
      action: Allow
      policy:
        matchExpressions:
          - >-
            request.path.startsWith("/mcp")
            ||
            (request.path.startsWith("/api/a2a/kagent/k8s-a2a-agent") &&
             (jwt.sub == "admin" || jwt.sub == "cyberjoker"))
            ||
            (request.path.startsWith("/api/a2a/kagent/helm-a2a-agent") &&
             jwt.sub == "admin")
EOF
```

### 9.5 The tool RBAC policy

Attach an MCP authorization to the backend allowing every tool except `k8s_apply_manifest`, which requires `jwt.sub == "admin"`.

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: kagent-tools-tool-rbac
  namespace: kagent
spec:
  targetRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
      name: kagent-tools-mcp
  backend:
    mcp:
      authorization:
        action: Allow
        policy:
          matchExpressions:
            - >-
              mcp.tool.name != "k8s_apply_manifest"
              ||
              jwt.sub == "admin"
EOF
```

### 9.6 Verify

Ask the agent to apply a Pod through the gateway: `admin` succeeds, `cyberjoker` is blocked.

`admin` → the agent applies the Pod.

```sh
curl -s -X POST http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/ \
  -H "Authorization: Bearer $JWT_ADMIN" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{"message":{"role":"user","parts":[{"kind":"text","text":"Apply this manifest: a Pod named demo-pod in the default namespace running the nginx image. USE THE TOOL YOU HAVE"}],"messageId":"apply-1","kind":"message"}}}' \
  --max-time 120
# the agent calls k8s_apply_manifest and reports the Pod was created;
# confirm with: kubectl get pod demo-pod -n default
```

`cyberjoker` → the same request is refused at the tool.

```sh
curl -s -X POST http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/ \
  -H "Authorization: Bearer $JWT_CYBERJOKER" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{"message":{"role":"user","parts":[{"kind":"text","text":"Apply this manifest: a Pod named demo-pod in the default namespace running the nginx image. USE THE TOOL YOU HAVE"}],"messageId":"apply-2","kind":"message"}}}' \
  --max-time 120
# the agent reports it only has read-only tools and prints the YAML instead —
# the Pod is NOT created
```

The denial is enforced by filtering `k8s_apply_manifest` out of `cyberjoker`'s `tools/list` — its `/mcp` calls all return `200`, the tool is just absent, so no `403` ever appears.

Watch the tool hop — `cyberjoker` has no `k8s_apply_manifest` line, `admin` does.

```sh
kubectl -n kagent logs deploy/agent-gateway -f | grep -i 'mcp.tool.name\|gen_ai.tool.name\|tools/'
```

> Heads-up: `proxy.url` is global, so `helm-a2a-agent`'s tool calls now `401` until you give it `KAGENT_PROPAGATE_TOKEN` too (as in 9.2); this demo focuses on `k8s-a2a-agent`.

Done. You now have two layers of control: **A2A endpoint** RBAC (who can reach an
agent) and **MCP tool** RBAC (which tools a user can make an agent run) — both
keyed on the same authn JWT.
