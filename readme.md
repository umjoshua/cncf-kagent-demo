# kagent live demo: "which version am I actually talking to?"


(Code for the demo taken at CNCF + DevOps Malayalam Trivandrum Event - Jul 18 2026)


A minimal, deterministic setup for demoing [kagent](https://kagent.dev) root-causing
a classic GitOps footgun: a Service whose selector doesn't include the version label,
so traffic silently splits across an old and a new release.

## Repo layout

```
chart/api/       Helm chart for the demo app (Deployment + Service only)
argocd/           Argo CD Application that deploys the chart
agents/           Four kagent Agent manifests (kagent.dev/v1alpha2)
README.md         This file
```

## Prereqs

- A Kubernetes cluster with:
  - kagent + kmcp installed, including the built-in `kagent-tool-server`
    RemoteMCPServer (verify with `kubectl get remotemcpservers -n kagent`)
  - a `ModelConfig` named `default-model-config` in the `kagent` namespace
    (verify with `kubectl get modelconfigs -n kagent`)
  - Argo CD installed, with an `argocd` namespace
- `kubectl` pointed at that cluster
- `helm` (optional, only needed if you want to template/lint the chart locally)

## How the demo bug works

- `chart/api` templates a Deployment named `api-{{ .Values.version }}` - every
  version bump creates a **new** Deployment object rather than updating one in
  place (see `chart/api/templates/deployment.yaml`).
- The `api` Service selects on `app: api` only, with **no** `version` label
  (see `chart/api/templates/service.yaml`). That's the intentional bug: once
  two Deployment versions exist, the one Service load-balances across both.
- The Argo CD Application syncs with `prune: false`, so bumping `version` and
  syncing leaves the previous version's Deployment running instead of deleting
  it - that's what produces the orphaned old-version workload.

## Apply order

1. Fill in `argocd/application.yaml`'s `spec.source.repoURL` with this repo's
   git remote, then create the Application and let it sync `v1`:

   ```bash
   kubectl apply -f argocd/application.yaml
   kubectl get application api -n argocd -w
   ```

2. Bump the release: edit `chart/api/values.yaml` to `version: v2`, commit and
   push, then let Argo CD auto-sync (or `argocd app sync api`). Because
   `prune: false`, this creates `api-v2` alongside the still-running `api-v1` -
   both now match the `api` Service's selector.

   > Producing the actual leftover/orphaned old-version Deployment for the live
   > demo (i.e. performing step 2) is left to you - the manifests here just
   > make that outcome the natural result of a normal version bump.

3. Apply the four agents:

   ```bash
   kubectl apply -f agents/
   kubectl get agents -n kagent
   ```

## Running the demo

Trigger the orchestrator with only the symptom - no hints about Services,
Deployments, or Argo CD:

```bash
kagent invoke --agent release-detective --task "About half of requests to the api \
Service return VERSION=v1 even though the current release is v2. Find the root cause."
```

`release-detective` has no infrastructure tools of its own; it delegates to
`k8s-topology-agent`, `traffic-observability-agent`, and `release-gitops-agent`
and correlates their answers. Its system prompt explicitly tells it not to
guess and not to trust a single `kubectl get deploy` snapshot from any one
specialist.

## Viewing agent cards

```bash
kubectl port-forward svc/kagent-controller 8083:8083 -n kagent
```

then, in another terminal:

```bash
curl localhost:8083/api/a2a/kagent/release-detective/.well-known/agent.json
curl localhost:8083/api/a2a/kagent/k8s-topology-agent/.well-known/agent.json
curl localhost:8083/api/a2a/kagent/traffic-observability-agent/.well-known/agent.json
curl localhost:8083/api/a2a/kagent/release-gitops-agent/.well-known/agent.json
```

## Seeing the flip-flop yourself before running the agent

Once both `api-v1` and `api-v2` exist and the `api` Service is up, port-forward
the Service and hit it in a loop - you'll see the response alternate:

```bash
kubectl port-forward svc/api 8080:80 -n demo &
for i in $(seq 1 10); do curl -s localhost:8080/; sleep 0.2; done
```

## Notes on CRD verification

This demo was written against a live cluster's `agents.kagent.dev` CRD
(`kagent.dev/v1alpha2`), not from memory. Two things worth calling out:

- **No Argo CD MCP tool exists** on this cluster's `kagent-tool-server` - only
  `argo_*` tools for **Argo Rollouts** (e.g. `argo_verify_argo_rollouts_controller_install`),
  which is a different project. `release-gitops-agent` therefore uses the
  Kubernetes tools (`k8s_get_resources`, `k8s_describe_resource`,
  `k8s_get_resource_yaml`) to read `applications.argoproj.io` objects directly,
  per the fallback in the spec.
- `spec.declarative.tools[].mcpServer` requires `apiGroup: kagent.dev` and
  `kind: RemoteMCPServer` alongside `name`/`toolNames` (confirmed against
  existing in-cluster agents, e.g. `k8s-agent`) - included in all four
  manifests even though only `name` is strictly required by the schema.

Everything else (`spec.type: Declarative`, `spec.declarative.{modelConfig,
systemMessage, tools, a2aConfig.skills}`, and the `{type: Agent, agent:
{apiGroup, kind, name}}` delegation form) matched the spec as given, and all
four Agent manifests plus the Argo CD Application passed
`kubectl apply --dry-run=server` against the live CRDs.
