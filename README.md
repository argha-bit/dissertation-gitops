# dissertation-gitops

A GitOps repository managing Kubernetes workloads via ArgoCD using the **App of Apps** pattern, with Gloo Edge as the API gateway and a custom **Governor** ArgoCD plugin for dynamic route management.

---

## Architecture Overview

```
GitHub (this repo)
        │
        │  Git sync
        ▼
  ┌─────────────────────────────────────────┐
  │              ArgoCD (argocd ns)         │
  │                                         │
  │   root-app ──────────────────────────┐  │
  │       │                              │  │
  │       ├── infra-apps      (secrets)  │  │
  │       ├── arctic-wolf     (app)      │  │
  │       ├── test-app        (app)      │  │
  │       ├── gloo            (helm)     │  │
  │       ├── gateway-config  (routes)   │  │
  │       └── governor-routes (plugin)   │  │
  └──────────────────────────────────────┘  │
                                            │
                  ┌─────────────────────────┘
                  ▼
         Kubernetes (my-namespace)
         ├── Gloo Edge Gateway (port 8080)
         │       └── HTTPRoute: test.local → test-app
         ├── Arctic Wolf Risk Manager (ghcr.io private image)
         ├── Test App (echo server)
         └── Governor sidecar (dynamic route management)
```

All applications run with `automated.prune: true` and `automated.selfHeal: true`, so the cluster state always converges to what is in Git.

---

## Repository Structure

```
dissertation-gitops/
├── argocd-apps/              # ArgoCD Application manifests (App of Apps)
│   ├── root-app.yaml         # Bootstrap root — manages this entire directory
│   ├── infra-apps.yaml       # Deploys infra/ (secrets, cluster-wide resources)
│   ├── arctic-wolf.yaml      # Arctic Wolf Risk Manager application
│   ├── test-app.yaml         # Echo server for testing
│   ├── gloo.yaml             # Gloo Edge gateway (Helm)
│   ├── gateway-config.yaml   # Kubernetes Gateway + HTTPRoutes
│   └── argo-plugin-arctic.yaml # Governor plugin application
│
├── apps/
│   ├── arctic-wolf-risk-manager/
│   │   └── deployment.yaml   # Deployment + Service (ghcr.io private image)
│   └── test-app/
│       └── deployment.yaml   # Deployment + Service (echo server)
│
├── gateway-config/
│   ├── gateway.yaml          # Kubernetes Gateway (gloo-gateway class, port 8080)
│   └── routes/
│       └── test-route.yaml   # HTTPRoute + RouteOption (header injection)
│
├── helm/
│   └── gloo/
│       ├── Chart.yaml        # Depends on gloo v1.20.10
│       └── values.yaml       # kubeGateway: enabled
│
├── infra/
│   ├── ghcr-secret.yaml          # Image pull secret (my-namespace)
│   └── ghcr-secret-argocd.yaml   # Image pull secret (argocd namespace)
│
├── plugin-routes/
│   └── governor-config.yaml  # Governor sidecar configuration
│
└── bootstrap/
    └── argocd-values.yaml    # Helm values used to install ArgoCD with Governor sidecar
```

---

## Bootstrap: Fresh Cluster Setup

Follow these steps in order. Steps 2–3 must happen **before** ArgoCD is installed, otherwise the Governor sidecar will fail to start.

### 1. Create the ArgoCD namespace

```bash
kubectl create namespace argocd
```

### 2. Create the GHCR image pull secret

ArgoCD needs this to pull the private Governor image from GitHub Container Registry.

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<github-pat> \
  --docker-email=<email> \
  -n argocd
```

### 3. Apply the Governor plugin ConfigMap (Only If You want to Use The-Governor as ConfigManagement Plugin into your system )

The ConfigMap must exist before the ArgoCD `repo-server` pod starts — the Governor sidecar container mounts it at startup.

```bash
kubectl apply -f bootstrap/governor-plugin-cm.yaml -n argocd
```

> **Why this order matters:** the `repo-server` pod will crash-loop if it cannot mount the `the-governor-plugin` ConfigMap. Applying it first guarantees a clean start.

### 4. Install ArgoCD with the Governor sidecar

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  -n argocd \
  --version 9.4.7 \
  -f bootstrap/argocd-values.yaml
```

The values file configures the `repo-server` with:
- An image pull secret (`ghcr-secret`) to pull the Governor image
- A sidecar container (`the-governor`) running the ArgoCD CMP server
- Volume mounts wiring the plugin config into the sidecar

### 5. Bootstrap the App of Apps

This single `kubectl apply` hands control of everything else to ArgoCD.

```bash
kubectl apply -f argocd-apps/root-app.yaml
```

ArgoCD will detect the `argocd-apps/` directory and create all child applications automatically. From this point, all changes are made via Git.

---

## ArgoCD Values: Governor Sidecar (`bootstrap/argocd-values.yaml`)
> **For More information:** Please visit `argocd` official config management plugin documentation for more details. https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/
```yaml
repoServer:
  imagePullSecrets:
    - name: ghcr-secret
  volumes:
    - name: governor-plugin-config
      configMap:
        defaultMode: 420
        name: the-governor-plugin
  extraContainers:
    - name: the-governor
      image: ghcr.io/argha-bit/the-governor:latest
      imagePullPolicy: Always
      command: [/var/run/argocd/argocd-cmp-server]
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
      volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
        - mountPath: /tmp
          name: tmp
        - mountPath: /home/argocd/cmp-server/config/plugin.yaml
          name: governor-plugin-config
          subPath: plugin.yaml
```

---

## Applications

| Application | Source | Namespace | Notes |
|-------------|--------|-----------|-------|
| **Arctic Wolf Risk Manager** | `apps/arctic-wolf-risk-manager/` | `my-namespace` | Private image — requires `ghcr-secret` |
| **Test App** | `apps/test-app/` | `my-namespace` | Public echo server (`ealen/echo-server`) |
| **Gloo Edge** | `helm/gloo/` | `my-namespace` | Kubernetes Gateway API mode (v1.20.10) |
| **Gateway Config** | `gateway-config/` | `my-namespace` | Gateway + HTTPRoutes |
| **Governor Routes** | `plugin-routes/` | `my-namespace` | Managed by `the-governor` CMP plugin |
| **Infra** | `infra/` | `default` | Cluster secrets (GHCR credentials) |

---

## Gateway & Routing

Traffic enters through the Gloo Edge gateway (Kubernetes Gateway API):

```
Host: test.local
Port: 8080
  │
  └── HTTPRoute: test-route
        Path prefix: /  →  test-app Service :8080
        Request headers injected:  x-custom-header: my-value
        Response headers injected: x-response-header: added-by-gloo
```

The `RouteOption` resource (managed by Gloo) handles header manipulation declaratively.

---

## Governor Plugin

The **Governor** is a custom ArgoCD Config Management Plugin (CMP) that runs as a sidecar inside the `repo-server` pod. It enables dynamic HTTPRoute management driven by an external configuration endpoint.

| Setting | Value |
|---------|-------|
| Config endpoint | `http://host.minikube.internal:3000/routes` |
| Webhook URL | `http://the-governor.svc.cluster.local/webhooks` |
| Service identity | `auth-svc-99` / `identity-manager` |
| Team | `platform-security` |

The sidecar is registered as an ArgoCD plugin via the `plugin.yaml` ConfigMap (`the-governor-plugin`) mounted at `/home/argocd/cmp-server/config/plugin.yaml`.

---

## Namespaces

| Namespace | What lives here |
|-----------|----------------|
| `argocd` | ArgoCD itself + GHCR pull secret for private images |
| `my-namespace` | All user workloads (apps, gateway, gloo, governor routes) |
| `default` | Infrastructure secrets (GHCR credentials for workloads) |

---

## Making Changes

All changes are made by pushing to this repository. ArgoCD polls Git and reconciles the cluster automatically.

- **Add a new application** — add a manifest under `argocd-apps/` and a corresponding directory under `apps/`
- **Update an image** — edit the `image:` field in the relevant `deployment.yaml`
- **Add a route** — add an `HTTPRoute` under `gateway-config/routes/`; gateway-config app will sync it
- **Update governor config** — edit `plugin-routes/governor-config.yaml`

ArgoCD `selfHeal` ensures any manual `kubectl` changes to the cluster are reverted back to the Git state.
