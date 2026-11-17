# one-agent-per-person

Companion manifests for the post **One Agent per Person**
(<https://papasy.me/posts/one-agent-per-person/>): a sandboxed Hermes agent
on Kubernetes, deployed one release per person by Argo CD.

Everything here is sanitised — replace the `<...>` placeholders and the
`example.com` hostnames before applying anything.

## What the cluster needs

- Kubernetes 1.33+ with Gateway API, and a CNI that enforces network policy
  (Cilium here)
- agent-sandbox **v1.0.10** — the `Sandbox` CRD and controller, installed
  before the agent release (sync wave 0)
- a namespace admitted as `privileged` (the combined supervisor needs it)
- an RWX class for the agent workspace and an RWO, non-NFS class for the
  gateway database
- an OpenAI-compatible model endpoint plus the `model-api-key` Secret
- an authenticating proxy in front of the dashboard — Hermes 0.19 has no
  login of its own

```sh
# Controller first, or the first sync fails with "no matches for kind Sandbox"
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/v1.0.10/sandbox-with-extensions.yaml
```

## Layout

```
gitops/
├── agent-sandbox/                      # wave 0
│   └── overlays/prod-cluster/
│       └── appset-config.yaml
└── agent-runtime/                      # wave 3
    └── overlays/
        ├── base/values-common.yaml     # <- values-common.yaml
        ├── tenant-a/
        │   ├── values.yaml             # <- tenant-a-values.yaml
        │   └── extras/                 # <- extras/
        └── prod-cluster/
            └── app.yaml                # <- app-tenant-a.yaml
```

## Use

1. Copy `values-common.yaml` to
   `agent-runtime/overlays/base/values-common.yaml`.
2. Copy `tenant-a-values.yaml` to
   `agent-runtime/overlays/tenant-a/values.yaml`.
3. Copy `extras/` to `agent-runtime/overlays/tenant-a/extras/`.
4. Copy `app-tenant-a.yaml` to
   `agent-runtime/overlays/prod-cluster/app-tenant-a.yaml`.
5. Replace the placeholders:

   | Placeholder | Set it to |
   | --- | --- |
   | `<model-name>`, `https://model.example/v1` | your OpenAI-compatible endpoint |
   | `https://api.example.com:6443` | the issuer from `kubectl get --raw /.well-known/openid-configuration` |
   | `<api-endpoint>`, `<control-plane-range>` | the API VIP and control-plane addresses (6443) |
   | `<model-endpoint>` | the model host the gateway calls |
   | `<rwx-storage-class>`, `<rwo-storage-class>` | your two storage classes |
   | `<gateway-name>`, `<gateway-namespace>`, `<listener-name>` | your Gateway API listener |
   | `<auth-namespace>` | where the forward-auth service runs |
   | `agent-a.example.com` | the tenant's dashboard hostname |

6. Commit. Argo creates the namespace, syncs the chart, and the bootstrap
   Job creates the Sandbox.

Check it:

```sh
kubectl -n agent-tenant-a get sandbox
kubectl -n agent-tenant-a get pods
```

## A second instance

Repeat steps 2-4 with `tenant-b`: a new `values.yaml`, a new `extras/`
(namespace `agent-tenant-b`), and a new child Application. Only the release
name, namespace and hostname change.

Do **not** copy `platform.serviceAccountIssuerDiscovery` out of
`values-common.yaml`: the binding it creates is cluster-scoped, and only one
release may own it.

Each release runs its own gateway pod, its own sandbox and its own volumes.

## Gotchas this code encodes

- `nemoclawHome.enabled: false` — the chart's SELinux relabel hook fails on
  enforcing nodes, so NemoClaw's drain gate can report "draining".
- `operatorClient.enabled: false` — the attach client cannot start on
  Kubernetes 1.37 / containerd 2.3 (subPath mount inside read-only `/tools`).
- `gatewayPersistence.storageClass` — OpenShell requires a non-NFS volume
  for its SQLite database.
- `exposure.type: ingress` exists to deploy the nginx edge, not an Ingress.
- The sandbox NetworkPolicy must select
  `agents.x-k8s.io/sandbox-name-hash`, not `openshell.ai/managed-by`.
- `argocd.argoproj.io/ignore-healthcheck` on the inert Ingress — otherwise
  Argo waits forever and the bootstrap Job never runs.
