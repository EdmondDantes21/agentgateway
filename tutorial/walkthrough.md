# Impersonating Users Across kagent Agents — A Hands-On Walkthrough

Your agent has one identity.

Everyone who calls it borrows that identity. The intern asking it to list pods and the pipeline asking it to delete a namespace look exactly the same to the agent. Same name, same permissions, same blast radius.

That's fine in a demo. In production it's the thing that gets you paged at 2am.

The problem was never the agent…it was the identity around it. And the fix isn't a login at the front door — that just proves SOMEONE is there, not who, and not what they're allowed to do. The fix is to make the caller's identity survive every hop: from the user, through the gateway, into the agent, out to a tool, and on to a sub-agent.

This is the build-along. Every command here is real and in order — paste them and you'll watch the same agent behave one way for `admin` and another for `cyberjoker`, with nothing changing but the token.

Onward.

Here's what you'll build:

- A gateway in front of kagent that validates every caller's JWT.
- An identity provider (authn — Keycloak drops in the same way) that issues those tokens and publishes the keys to verify them.
- Three layers of access control, all keyed on the one token: which agent you can reach, which tool you can make it run, and which other agents it can call for you.


## Meet the two systems

Before we wire anything, know what we're wiring.

### kagent

kagent runs AI agents on Kubernetes the same way you run everything else on Kubernetes — declaratively.

An agent is a Custom Resource. So is the model it uses (`ModelConfig`), and so are the tools it's allowed to call. You write YAML, `kubectl apply` it, and a controller reconciles it into a running agent — versioned, RBAC'd, and observable like any other workload. No bespoke deployment, no snowflake.

Two protocols carry the whole story. An agent talks to its tools over MCP (the Model Context Protocol). An agent talks to OTHER agents over A2A (agent-to-agent). Both come back constantly below — the tools we gate are MCP, the delegation we gate is A2A.

And one structural detail that shapes everything: the kagent controller — NOT the agent pods — serves every agent's public A2A endpoint, at `/api/a2a/<ns>/<agent>/…`. That's why, in a moment, a single route in front of the controller reaches every agent at once.

### agentgateway

agentgateway is the data plane we put in front of kagent. It's the thing that reads tokens and enforces rules.

The obvious question: why not the API gateway you already run? Because MCP and A2A are NOT the stateless, one-request-in-one-response-out REST that ordinary gateways were built for. They're stateful, long-lived, JSON-RPC protocols. Sessions stay open. A single "list the tools" call fans out across backends and gets reassembled. Servers push events back to the client. Two different callers get shown two different tool sets on the same connection. A stateless proxy can't model any of that.

agentgateway can — it was built for exactly this. And crucially for us, it ships with the controls MCP and A2A leave out: built-in JWT authentication, and an RBAC system that writes rules against the claims inside those tokens.

That RBAC engine is the whole game. It speaks CEL — small boolean expressions — and it hands you two kinds of thing to write them against: the validated token claims (`jwt.sub`, `jwt.groups`) and facts about the request (`request.path`, `request.headers`, and for tool calls, `mcp.tool.name`). Every rule in this walkthrough is one short expression combining those.

On Kubernetes it's configured through the standard Gateway API (`Gateway`, `HTTPRoute`) plus two resources of its own: `AgentgatewayPolicy` (JWT auth, authorization, backend auth) and `AgentgatewayBackend` (an MCP tool server, among other things). You'll apply all of them below.

Two systems, one seam between them. Let's build the seam.


## Set the stage

Before any identity flows, we need the two systems and the wire between them: agentgateway (the thing that checks tokens) and kagent (the thing that runs agents).

Start with agentgateway. First the Kubernetes Gateway API CRDs it builds on:

```sh
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```

Then agentgateway's own control-plane CRDs, and the control plane itself:

```sh
helm upgrade -i agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds \
  --create-namespace --namespace agentgateway-system \
  --version v1.3.1 \
  --set controller.image.pullPolicy=Always
```

```sh
helm upgrade -i agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
  --namespace agentgateway-system \
  --version v1.3.1 \
  --set controller.image.pullPolicy=Always \
  --wait
```

One pod should come up:

```sh
kubectl get pods -n agentgateway-system
```

```
NAME                            READY   STATUS    RESTARTS   AGE
agentgateway-5495d98459-46dpk   1/1     Running   0          19s
```

Now kagent. CRDs first, then the platform, wired to your model provider:

```sh
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --namespace kagent \
  --create-namespace
```

```sh
export GEMINI_API_KEY="your-api-key-here"
```

```sh
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --namespace kagent \
  --set providers.default=gemini \
  --set providers.gemini.apiKey=$GEMINI_API_KEY
```

We're using Gemini, but nothing here depends on it — OpenAI, Anthropic, Bedrock, Ollama and others work too. Just swap `providers.default` and the matching key; see the [supported providers docs](https://kagent.dev/docs/kagent/supported-providers).

kagent ships with some demo agents. Clear them out — we're building our own:

```sh
kubectl delete agents.kagent.dev -n kagent --all
```

That's the stage. Now the piece that makes the whole thing possible: identity.


## The identity provider

Everything downstream rests on one idea from public-key cryptography.

A keypair has a private half and a public half. The private key SIGNS. The public key VERIFIES. You can hand the public key to anyone — it can only check a signature, never forge one.

That asymmetry is what makes this scale safely. authn (the identity provider) is the ONLY component that ever holds the signing key; the gateway only ever needs the public half. Nothing secret gets copied into every service, and a compromised gateway can't mint tokens — it can only check them.

Generate the RSA private key:

```sh
openssl genrsa -out private.pem 2048
```

Create a Kubernetes secret holding that key:

```sh
kubectl create namespace krateo-system   # if needed

kubectl create secret generic jwt-sign-key -n krateo-system \
  --from-file=private.pem=./private.pem
```

Install the authn chart:

```sh
helm install authn oci://ghcr.io/edmonddantes21/authn -n krateo-system --version 0.0.6 \
  --set image.repository=redi21/authn \
  --set image.tag=0.0.1 \
  --set image.pullPolicy=IfNotPresent
```

### How the public half gets distributed: the JWKS

The gateway needs authn's public key, but hardcoding it into config would mean redeploying every verifier whenever the key changes. Instead, authn publishes it.

A **JWKS** (JSON Web Key Set) is just a small JSON document listing one or more public keys in a standard format. It lives at a conventional path — `/.well-known/jwks.json` — so any verifier can fetch it, cache it, and refetch it later. Each key in the set carries a **`kid`** (key ID), which is how a verifier picks the right one: a JWT's header names the `kid` it was signed with, the verifier looks that `kid` up in the set, and validates the signature against it. That indirection is what lets you rotate keys — publish the new key alongside the old one, start signing with the new `kid`, and retire the old entry once outstanding tokens have expired. No coordinated redeploy needed.

Confirm authn is publishing its:

```sh
kubectl -n krateo-system port-forward svc/authn 8082:8082  # in a separate terminal
```

```sh
curl -s http://localhost:8082/.well-known/jwks.json | jq
```

You should see exactly ONE RSA key, with `"kid": "krateo-authn-key-1"`. Every token authn signs carries that same `kid` in its header.

One last detail, and it shapes every policy below: authn's tokens carry `iss: "krateo.io"`, a `sub` (the username), and `groups` — but NO `aud` claim. So every JWT policy we write matches on the issuer and omits audiences. Remember that and you'll save yourself a confusing rejection later.

The stage is set and identity is live. Time to build something to protect.

## The two agents

Two agents, two different toolboxes. That contrast is the whole demo.

First a model config they'll share:

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

The Kubernetes agent. Note its tools — it can read the cluster AND apply manifests. That apply tool is the dangerous one we'll gate later:

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

And the Helm agent — same model, a Helm toolbox. Later this one becomes a sub-agent of the first:

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

Two agents, wide open. Anyone who can reach them can run anything. Let's start closing that down.


## Layer 1 — which agent can you reach?

The coarsest question first: is this user even allowed to talk to this agent?

To answer it, all traffic has to funnel through the gateway. Four resources do that: the gateway itself, one route that fronts every agent, the JWT check, and the rule that decides who reaches which agent.

The Gateway. The controller auto-provisions the proxy behind it; all agent traffic flows through port 8080:

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agent-gateway
  namespace: kagent
spec:
  gatewayClassName: agentgateway     # run this gateway with the agentgateway controller
  listeners:
    - name: http
      protocol: HTTP
      port: 8080                     # every agent request enters the cluster here
      allowedRoutes:
        namespaces:
          from: Same                 # only accept routes from this same namespace (kagent)
EOF
```

One route to rule them all. The kagent controller — NOT the agent pods — serves every agent's A2A endpoint under `/api/a2a/<ns>/<agent>/…`, so a single route to the controller reaches every agent:

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kagent-a2a
  namespace: kagent
spec:
  parentRefs:
    - name: agent-gateway            # attach this route to the gateway above
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /                 # match every path...
      backendRefs:
        - name: kagent-controller    # ...and forward to the controller, which serves
          port: 8083                 # every agent's A2A endpoint at /api/a2a/<ns>/<agent>/
EOF
```

Now the JWT check — the crypto turned into config. `mode: Strict` means no token, no entry. It fetches authn's public keys from the JWKS endpoint, matches the issuer, and (because authn emits no `aud`) sets no audiences:

```sh
kubectl apply -f - <<'EOF'
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: authn-jwt
  namespace: kagent
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agent-gateway                     # enforce this JWT check on the whole gateway
  traffic:
    jwtAuthentication:
      mode: Strict                             # no valid token -> 401, rejected before routing
      providers:
        - issuer: "krateo.io"                  # must match the token's `iss` claim exactly
          jwks:
            remote:
              jwksPath: /.well-known/jwks.json # the path where authn publishes its public keys
              cacheDuration: 5m                # re-fetch those keys every 5m (picks up key rotation)
              backendRef:                      # ...and fetch them from authn itself:
                group: ""
                kind: Service
                name: authn                    # authn's Service
                namespace: krateo-system       # in its namespace
                port: 8082
EOF
```

And here's Layer 1 itself. On top of a valid token, the `sub` claim plus the request path decide the door. `admin` reaches both agents; `cyberjoker` reaches only the Kubernetes one. Everything else is a 403:

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
      name: agent-gateway               # apply this authorization to the whole gateway
  traffic:
    authorization:
      action: Allow                      # allowlist: deny (403) unless an expression below matches
      policy:
        matchExpressions:
          # admin reaches both agents; cyberjoker only the k8s agent
          - >-
            (request.path.startsWith("/api/a2a/kagent/k8s-a2a-agent") &&
             (jwt.sub == "admin" || jwt.sub == "cyberjoker"))
            ||
            (request.path.startsWith("/api/a2a/kagent/helm-a2a-agent") &&
             jwt.sub == "admin")
EOF
```

Now get two tokens. Port-forward authn in a separate terminal, then log in as each user:

```sh
kubectl port-forward -n krateo-system svc/authn 8082:8082
```

```sh
JWT_ADMIN=$(curl -s 'http://localhost:8082/basic/login' \
  -H "Authorization: Basic $(echo -n "admin:$(kubectl get secret admin-password -n krateo-system -o jsonpath='{.data.password}' | base64 -d)" | base64)" \
  --insecure | jq -r .accessToken)

JWT_CYBERJOKER=$(curl -s 'http://localhost:8082/basic/login' \
  -H "Authorization: Basic $(echo -n "cyberjoker:$(kubectl get secret cyberjoker-password -n krateo-system -o jsonpath='{.data.password}' | base64 -d)" | base64)" \
  --insecure | jq -r .accessToken)
```

Prove it. Port-forward the gateway in another terminal:

```sh
kubectl -n kagent port-forward svc/agent-gateway 8080:8080
```

No token? The door is shut:

```sh
curl -i http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json
# HTTP/1.1 401 Unauthorized — authentication failure: no bearer token found
```

Valid token? Open:

```sh
curl -s -i -H "Authorization: Bearer $JWT_ADMIN" \
  localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json
# HTTP/1.1 200 OK
```

And the identity actually matters. `admin` reaches BOTH agents:

```sh
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_ADMIN" \
  localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json    # 200
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_ADMIN" \
  localhost:8080/api/a2a/kagent/helm-a2a-agent/.well-known/agent-card.json   # 200
```

`cyberjoker` reaches ONLY the Kubernetes agent — the helm door is a 403:

```sh
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_CYBERJOKER" \
  localhost:8080/api/a2a/kagent/k8s-a2a-agent/.well-known/agent-card.json    # 200
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $JWT_CYBERJOKER" \
  localhost:8080/api/a2a/kagent/helm-a2a-agent/.well-known/agent-card.json   # 403
```

One token, one rule, and the two users already diverge. But a door isn't fine-grained enough. `cyberjoker` CAN reach the Kubernetes agent — and that agent owns a tool that mutates the cluster.


## The chain that keeps the token alive

Before we can gate a tool, the token has to REACH the tool. By default it doesn't, and the reason is a chain of hops where the token quietly falls off.

The tool call travels: user → gateway → controller → agent → BACK through the gateway → tool. Four settings keep the identity alive across it. Miss one and the tool hop dies in a 401:

- passthrough — the gateway STRIPS the token after validating it, by default. This keeps it.
- trusted-proxy — the controller replaces the caller with a static identity, by default. This forwards the real one.
- proxy.url — routes agent egress back through the gateway, so the tool call is visible at all.
- KAGENT_PROPAGATE_TOKEN — the agent re-attaches the token on its outbound tool call.

Set the three controller-side switches. Note `proxy.url` is GLOBAL — once set, all agent egress flows through the gateway. That fact comes back in Layer 3:

```sh
helm upgrade kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
    --namespace kagent \
    --reuse-values \
    --set controller.auth.mode=trusted-proxy \
    --set controller.auth.userIdClaim=sub \
    --set proxy.url=http://agent-gateway.kagent.svc.cluster.local:8080
kubectl -n kagent rollout restart deploy/kagent-controller
```

Now stop the gateway from stripping the token. A passthrough policy on the controller's service is what makes the controller — and then the agent — receive the `Authorization` header at all:

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
      name: kagent-controller     # target the controller Service (a backend), not the gateway
  backend:
    auth:
      passthrough: {}             # keep the Authorization header instead of stripping it post-validation
EOF
```

> A caveat, not production posture: `trusted-proxy` trusts anything that reaches the controller. In-cluster, `kagent-controller:8083` is still directly reachable — lock it down (NetworkPolicy / mTLS) for real deployments.

Tell the Kubernetes agent to forward the token onto its own tool calls:

```sh
kubectl patch agent k8s-a2a-agent -n kagent --type merge -p '{
  "spec": {
    "declarative": {
      "deployment": {
        "env": [
          {"name": "KAGENT_PROPAGATE_TOKEN", "value": "true"}
        ]
      }
    }
  }
}'
```

Last, teach the gateway to speak MCP to the tool server, and route the proxied tool calls (stamped `x-kagent-host: kagent-tools.kagent`) to it:

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend            # teaches the gateway how to speak MCP to a tool server
metadata:
  name: kagent-tools-mcp
  namespace: kagent
spec:
  mcp:
    targets:
      - name: kagent-tools
        static:
          backendRef:
            name: kagent-tools       # the tool-server Service to proxy to
          port: 8084
          protocol: StreamableHTTP   # the MCP transport the gateway uses to reach it
          path: /mcp
---
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
            value: /mcp                    # proxied tool calls arrive on /mcp...
          headers:
            - name: x-kagent-host
              value: kagent-tools.kagent   # ...tagged with this host by kagent
      backendRefs:
        - name: kagent-tools-mcp           # route them to the MCP backend above
          group: agentgateway.dev
          kind: AgentgatewayBackend        # (an AgentgatewayBackend, not a plain Service)
EOF
```

> If MCP calls 404 at the gateway, your build stamps a different `x-kagent-host` value. Confirm it: `kubectl -n kagent logs deploy/agent-gateway -f | grep -i x-kagent-host`.

The token now survives all the way to the tool. Let's decide what it's allowed to do there.


## Layer 2 — which tool can you make it run?

Same agent, different powers.

`cyberjoker` and `admin` both reach the Kubernetes agent. We want `admin` to apply manifests and `cyberjoker` to read the cluster but NOT mutate it. That lives one level below the door — at the tool.

First, a plumbing fix. Layer 1's rule only allowlists `/api/a2a/...` paths, so it would 403 the proxied `/mcp` calls before they reach the tool policy. Re-apply it with a clause that lets `/mcp` through at the traffic layer — the real per-tool gating happens in the backend policy next:

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
      action: Allow                      # allowlist: deny (403) unless an expression below matches
      policy:
        matchExpressions:
          # let proxied /mcp calls through here; the real per-tool gating is the backend policy next
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

Now the tool rule itself. Because the tool call comes back through the gateway, the gateway can see BOTH `jwt.sub` (who) and `mcp.tool.name` (what) — and compare them. Any tool that isn't the dangerous one is allowed; the dangerous one requires `admin`:

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
      name: kagent-tools-mcp            # attach to the MCP backend, so rules can see mcp.tool.name
  backend:
    mcp:
      authorization:
        action: Allow                   # allowlist, evaluated per tool
        policy:
          matchExpressions:
            # every tool is allowed EXCEPT k8s_apply_manifest, which requires admin
            - >-
              mcp.tool.name != "k8s_apply_manifest"
              ||
              jwt.sub == "admin"
EOF
```

Prove it. Ask the agent — through the gateway — to apply a Pod. `admin` succeeds:

```sh
curl -s -X POST http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/ \
  -H "Authorization: Bearer $JWT_ADMIN" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{"message":{"role":"user","parts":[{"kind":"text","text":"Apply this manifest: a Pod named demo-pod in the default namespace running the nginx image. USE THE TOOL YOU HAVE"}],"messageId":"apply-1","kind":"message"}}}' \
  --max-time 120
# the agent calls k8s_apply_manifest and reports the Pod was created;
# confirm with: kubectl get pod demo-pod -n default
```

`cyberjoker`, same request, gets refused at the tool:

```sh
curl -s -X POST http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/ \
  -H "Authorization: Bearer $JWT_CYBERJOKER" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{"message":{"role":"user","parts":[{"kind":"text","text":"Apply this manifest: a Pod named demo-pod in the default namespace running the nginx image. USE THE TOOL YOU HAVE"}],"messageId":"apply-2","kind":"message"}}}' \
  --max-time 120
# the agent reports it only has read-only tools and prints the YAML instead —
# the Pod is NOT created
```

The denial is quieter than you'd expect. The gateway doesn't slam a 403 — it FILTERS `k8s_apply_manifest` out of the tool list `cyberjoker`'s agent is offered. The agent can't call a tool it never sees. Watch it:

```sh
kubectl -n kagent logs deploy/agent-gateway -f | grep -i 'mcp.tool.name\|gen_ai.tool.name\|tools/'
```

> Heads-up: `proxy.url` is global, so `helm-a2a-agent`'s tool calls now route through the gateway too — and it wasn't given `KAGENT_PROPAGATE_TOKEN`, so under Strict they'd 401. We fix that in the next layer, where it starts to matter.

Two layers, and the same agent already behaves like two different agents. Now the one everybody forgets.


## Layer 3 — which OTHER agents can it delegate to?

Agents call agents.

Make the Helm agent a tool of the Kubernetes agent, and a Helm question sent to the Kubernetes agent gets delegated — agent to agent — to the Helm agent. The user never makes that call directly. It happens inside the cluster, on their behalf.

Your instinct says Layer 1 already covers this: `cyberjoker` can't reach the Helm agent, so the delegated call is blocked for free. It is NOT — for three concrete reasons.

ONE: a sub-agent is called at its own service root, `/`, not at the public `/api/a2a/...` path Layer 1 matches on. So Layer 1's rule never fires for the delegated call. Left alone, it matches nothing and denies EVERYONE, `admin` included.

TWO: because `proxy.url` routes agent egress through the gateway, that internal hop is a real gateway request — not an invisible pod-to-pod call. kagent stamps it with `x-kagent-host: helm-a2a-agent.kagent`, and it still carries the original user's token. THAT is the handle we authorize on.

THREE — the subtle one that bites first: before delegating, the agent fetches the sub-agent's "agent card" at `/.well-known/agent-card.json` to discover what it can do. And it fetches that card with a bare HTTP client, BEFORE it attaches the token. So the card fetch arrives with NO token. Under `mode: Strict`, the gateway 401s it, and the delegation dies with a cryptic "failed to fetch agent card" before authorization ever runs.

That third point is why you can't keep the gateway strictly strict. Here's the tension, stated plainly:

> A user running a gateway in front of remote agents wants a single, strict policy: every request must carry a valid JWT, validated at the gateway. Because the card fetch arrives with no Authorization header, that policy has to be relaxed to either exempt the card path, or fall back to per-route policies instead of one blanket rule. Both weaken the security posture and add configuration surface that only exists to accommodate this one unauthenticated request.

The fix is to loosen the JWT check from Strict to Optional. It sounds scary and isn't: a valid token is still validated, an INVALID token is still rejected, and only a request with NO token is let through — straight into the authorization layer, which becomes the real gate. The card is public discovery metadata anyway, so letting it through anonymously is fine. Let's build it.

Make the Helm agent a sub-agent. The tools list is atomic, so the patch must include BOTH the existing MCP tool AND the new agent tool, and it tweaks the system message so the agent actually delegates:

```sh
kubectl patch agent k8s-a2a-agent -n kagent --type merge --patch-file=/dev/stdin <<'EOF'
spec:
  declarative:
    systemMessage: |-
        You are an expert Kubernetes agent that uses tools to help users.
        You also have a Helm specialist available as a tool (the helm agent).
        For anything about Helm releases — listing releases, release details,
        chart status — you MUST delegate to the helm agent tool and return its
        answer. Do not answer Helm questions yourself.
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          kind: RemoteMCPServer
          toolNames:
          - k8s_get_resources
          - k8s_get_available_api_resources
          - k8s_apply_manifest
      - type: Agent
        agent:
          name: helm-a2a-agent
          namespace: kagent
EOF
```

Wait for the k8s-a2a-agent to be recreated by the kagent controller:

```sh
kubectl -n kagent rollout status deploy/k8s-a2a-agent
```

Give the Helm agent the token-forwarding env too, so once it's reached it can carry the caller's token onto its OWN tool calls (this is the fix for the Layer 2 heads-up):

```sh
kubectl patch agent helm-a2a-agent -n kagent --type merge -p \
  '{"spec":{"declarative":{"deployment":{"env":[{"name":"KAGENT_PROPAGATE_TOKEN","value":"true"}]}}}}'
kubectl -n kagent rollout status deploy/helm-a2a-agent
```

Now loosen the JWT check so the tokenless card fetch isn't 401'd:

```sh
kubectl patch agentgatewaypolicy authn-jwt -n kagent --type merge -p \
  '{"spec":{"traffic":{"jwtAuthentication":{"mode":"Optional"}}}}'
```

Route the delegated hop to the Helm agent. It arrives at path `/` with `x-kagent-host: helm-a2a-agent.kagent`; this route matches that header and sends it to the Helm agent's own service. The header match makes it MORE specific than the catch-all route, so it wins for the delegation while direct user traffic still goes to the controller:

```sh
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kagent-a2a-helm-subagent
  namespace: kagent
spec:
  parentRefs:
    - name: agent-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /                        # the delegated hop hits the service root, not /api/a2a/...
          headers:
            - name: x-kagent-host
              value: helm-a2a-agent.kagent  # ...but only when the hop targets the helm agent
      backendRefs:
        - name: helm-a2a-agent              # forward it to the helm agent's own Service
          port: 8080
EOF
```

Keep the token on the hop to Helm — a passthrough on the Helm agent's service, mirroring the one on the controller:

```sh
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: authn-token-passthrough-helm
  namespace: kagent
spec:
  targetRefs:
    - group: ""
      kind: Service
      name: helm-a2a-agent      # target the helm agent's Service
  backend:
    auth:
      passthrough: {}           # keep the token on the hop to helm (same trick as on the controller)
EOF
```

Now the sub-agent RBAC itself. Two new things versus the Layer 2 version: a clause that lets the sub-agent CARD PATH through anonymously (the tokenless fetch), and a clause that gates the delegated call on the `x-kagent-host` header — `admin` only. Because the JWT layer is now Optional, every other clause is pinned to a known `sub` so anonymous traffic can't ride the relaxed layer:

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
      action: Allow                      # allowlist: deny (403) unless a clause matches
      policy:
        matchExpressions:
          # clauses in order (see the table below); the card-fetch clause is first and tokenless
          - >-
            (request.path.endsWith("/.well-known/agent-card.json") &&
             request.headers["x-kagent-host"] == "helm-a2a-agent.kagent")
            ||
            (request.path.startsWith("/mcp") &&
             (jwt.sub == "admin" || jwt.sub == "cyberjoker"))
            ||
            (request.path.startsWith("/api/a2a/kagent/k8s-a2a-agent") &&
             (jwt.sub == "admin" || jwt.sub == "cyberjoker"))
            ||
            (request.path.startsWith("/api/a2a/kagent/helm-a2a-agent") &&
             jwt.sub == "admin")
            ||
            (request.headers["x-kagent-host"] == "helm-a2a-agent.kagent" &&
             jwt.sub == "admin")
EOF
```

The card clause is listed FIRST on purpose — it references no `jwt.*`, so the tokenless card fetch short-circuits to allow before any identity check runs. Here's what each clause covers:

| Clause | Traffic it authorizes |
| --- | --- |
| `…/.well-known/agent-card.json` + `x-kagent-host` | the sub-agent card fetch (tokenless — the card is public) |
| `/mcp` + known `sub` | any agent's proxied tool call (per-tool gating is the backend policy) |
| `/api/a2a/kagent/k8s-a2a-agent` | user → k8s agent (admin AND cyberjoker) |
| `/api/a2a/kagent/helm-a2a-agent` | user → helm agent directly (admin only) |
| `x-kagent-host == "helm-a2a-agent.kagent"` + `sub == admin` | k8s agent → helm agent delegation (admin only) ← the RBAC line |

Prove it. Both users hit the SAME endpoint with the SAME prompt — the only difference is what happens when the Kubernetes agent tries to delegate. `admin` succeeds:

```sh
curl -s -X POST http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/ \
  -H "Authorization: Bearer $JWT_ADMIN" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{"message":{"role":"user","parts":[{"kind":"text","text":"List all Helm releases in the cluster. Use the helm agent tool you have."}],"messageId":"helm-admin-1","kind":"message"}}}' \
  --max-time 180 | jq -r '.result.status.state, (.result.artifacts[]?.parts[]?.text // empty)'
```

```
completed
Here are all the Helm releases found in the cluster across all namespaces:

| Name | Namespace | Revision | ... | Status | Chart | App Version |
| agentgateway | agentgateway-system | 1 | ... | deployed | agentgateway-v1.3.1 | v1.3.1 |
| authn | krateo-system | 1 | ... | deployed | authn-1.0.0 | |
| kagent | kagent | 6 | ... | deployed | kagent-0.9.12 | |
| snowplow | krateo-system | 1 | ... | deployed | snowplow-1.0.0 | |
...
```

The Kubernetes agent fetched Helm's card (anonymous, allowed), delegated through the gateway (allowed — `x-kagent-host` + `admin`), Helm ran its tool, and the table came back.

`cyberjoker`, same prompt, is blocked at the delegation:

```sh
curl -s -X POST http://localhost:8080/api/a2a/kagent/k8s-a2a-agent/ \
  -H "Authorization: Bearer $JWT_CYBERJOKER" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"message/send","params":{"message":{"role":"user","parts":[{"kind":"text","text":"List all Helm releases in the cluster. Use the helm agent tool you have."}],"messageId":"helm-joker-1","kind":"message"}}}' \
  --max-time 180 | jq -r '.result.artifacts[]?.parts[]?.text // empty'
```

```
I attempted to delegate the request to the Helm agent tool (kagent__NS__helm_a2a_agent),
but the call failed with the following error:

> HTTP Error 403: Forbidden
> Client error '403 Forbidden' for url 'http://agent-gateway.kagent.svc.cluster.local:8080'

This indicates an authorization issue between the Kubernetes agent and the Helm
agent via the agent gateway.
```

`cyberjoker` reaches the Kubernetes agent fine, and even the Helm card fetch succeeds (the anonymous clause). But the delegated call carries `x-kagent-host` + `sub = cyberjoker`, matches no allow clause, and gets a hard 403. Contrast with Layer 2, where the block was a silent missing tool — here the sub-agent is simply unreachable, and the Kubernetes agent cannot borrow its Helm powers at all.

Watch it on the gateway:

```sh
kubectl -n kagent logs deploy/agent-gateway -f | grep -i 'x-kagent-host\|helm-a2a-agent\|403\|jwt.sub'
```


## How it all fits together

No single piece here is impressive on its own.

A signed token. A gateway that checks signatures. A few headers forwarded between services. Three rules comparing a claim to a path, a tool name, a header. It's the COMBINATION that does the work:

- authn mints a verifiable identity.
- Four settings keep it alive across every hop instead of letting it die at the door.
- Layer 1 decides which agents you can reach.
- Layer 2 decides which tools you can make them run.
- Layer 3 decides which other agents they can call for you.

The person who logged in is STILL the person a tool call runs as, five hops later, in a delegation they never directly made. The agent stops being a shared identity everyone borrows and becomes a faithful proxy for whoever is actually asking.

Swap authn for Keycloak and almost nothing changes — point the issuer and JWKS at your provider, add audiences if it emits an `aud`, and the rest is identical.

Where does it strain? Trusted-proxy trusts anything that reaches the controller, so keep the controller unreachable except through the gateway. Optional mode leans harder on your authorization rules than Strict would, so a sloppy rule is a real hole. And the tokenless card fetch is a rough edge that shouldn't exist — the clean fix is upstream, teaching kagent to send the token on the card fetch too, after which Strict comes right back.

But the shape is right. Identity enters once, is proven once, and travels the whole way down — and every decision about what an agent may do is just a rule about who is asking.
