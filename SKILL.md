---
name: uds-deployment
description: Use when deploying an application to UDS (Unicorn Delivery Service) Core, scaffolding Helm charts for Zarf packages, configuring Keycloak SSO via UDS Package CR, or setting up a local k3d dev stack with UDS
---

# UDS Deployment

Deploy any Dockerized application on UDS Core with Keycloak SSO, Istio networking, and airgap-ready packaging.

## Overview

UDS (Unicorn Delivery Service) deploys apps into a hardened Kubernetes environment with Istio service mesh, Keycloak SSO, and Zarf airgap packaging. The deployment stack is:

```
Docker Image -> Helm Chart -> Zarf Package -> UDS Bundle -> k3d/Cluster
```

The UDS Operator reads a `Package` Custom Resource to auto-configure Istio networking, Keycloak SSO clients, and network policies.

### UDS Core Applications (included out of the box)

| Application | Role |
|-------------|------|
| Istio | Service Mesh |
| Keycloak | Identity & Access Management |
| Authservice | Authorization |
| Pepr | UDS Policy Engine & Operator |
| Prometheus Stack | Monitoring |
| Grafana | Dashboards & Visualization |
| Blackbox Exporter | Endpoint Probing |
| Metrics Server | Cluster Metrics |
| Loki | Log Aggregation |
| Vector | Log Collection & Routing |
| Falco | Runtime Security |
| Velero | Backup & Restore |

## When to Use

- Deploying any app to a UDS Core environment
- Scaffolding Helm charts, Zarf packages, or UDS bundles
- Configuring Keycloak SSO for an application
- Setting up local k3d dev stack with UDS Core
- Migrating auth from any provider to Keycloak/Authservice

## Prerequisites

- Docker
- `uds` CLI (includes `zarf` via `uds zarf`)
- `k3d` (for local dev)
- An app with a working Dockerfile

## File Structure

### Minimal (getting started)

```
uds/
├── chart/                      # Helm chart for your app
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── pvc.yaml            # If stateful
│       ├── configmap.yaml
│       ├── secret.yaml
│       └── uds-package.yaml    # UDS Package CR
├── values/
│   └── common-values.yaml
├── zarf.yaml                   # Zarf package config
├── bundle/
│   └── uds-bundle.yaml         # UDS Bundle (k3d + init + core + app)
└── tasks.yaml                  # UDS task runner
```

### Full UDS Package Template (production)

Based on [uds-packages/template](https://github.com/uds-packages/template):

```
├── charts/                     # Supplemental Helm charts (UDS Package CR, SSO config)
├── common/
│   └── zarf.yaml               # Base Zarf package (imported by root zarf.yaml)
├── values/
│   ├── common-values.yaml      # Shared Helm values across flavors
│   └── <flavor>-values.yaml    # Flavor-specific values (upstream, registry1, etc.)
├── bundle/                     # Test bundles for local dev
├── tasks/                      # UDS CLI task definitions
├── tasks.yaml                  # Task entrypoint (like Makefile)
├── tests/                      # Integration tests
└── zarf.yaml                   # Root Zarf package with flavor components
```

Key concepts:
- **Flavors**: Different image sources (upstream, Iron Bank/registry1, Chainguard). Each flavor has its own values file and image list.
- **common/zarf.yaml**: Base package imported by root, contains charts and shared config.
- **charts/**: Supplemental charts beyond the app's own Helm chart (UDS Package CR, SSO config).

## Quick Reference

| Concept | What it does |
|---------|-------------|
| **Helm Chart** | K8s resources (Deployment, Service, PVC, ConfigMap, Secret) |
| **UDS Package CR** | Tells UDS Operator how to wire Istio + Keycloak for your app |
| **Zarf Package** | Bundles Helm chart + container images for airgap deploy |
| **UDS Bundle** | Combines k3d + Zarf init + UDS Core + your Zarf package |
| **Authservice** | Istio sidecar that handles OIDC flow with Keycloak |
| **protocolMappers** | Map Keycloak user attributes to JWT claims |

## Core Pattern: UDS Package CR

This is the most important file. It registers your app with UDS Core:

```yaml
apiVersion: uds.dev/v1alpha1
kind: Package
metadata:
  name: myapp
  namespace: {{ .Release.Namespace }}
spec:
  network:
    expose:
      - service: {{ .Release.Name }}
        selector:
          app.kubernetes.io/name: myapp
        gateway: tenant
        host: myapp          # -> myapp.uds.dev
        port: 8000
    allow:
      - direction: Egress
        remoteGenerated: Anywhere
        description: "Allow egress for external API calls"

  sso:
    - name: MyApp SSO
      clientId: uds-core-myapp
      redirectUris:
        - "https://myapp.uds.dev/login/generic_oauth"
      enableAuthserviceSelector:
        app.kubernetes.io/name: myapp
      groups:
        anyOf:
          - "/UDS Core/Admin"
      protocolMappers:
        - name: email
          protocol: "openid-connect"
          protocolMapper: "oidc-usermodel-property-mapper"
          config:
            user.attribute: "email"
            claim.name: "email"
            id.token.claim: "true"
            access.token.claim: "true"
            userinfo.token.claim: "true"
```

## Auth Pattern: Trust Istio, Decode JWT

Authservice validates the JWT at the Istio mesh layer before requests reach your app. Your app just:

1. Read `Authorization: Bearer {token}` header
2. Base64-decode the JWT payload (middle segment) -- **no signature verification needed**
3. Parse the JSON claims

The JWT is already validated by Istio. Skip signature checks entirely. Claims in the JWT are determined by the `protocolMappers` in the UDS Package CR above and user attributes configured in the Keycloak admin console (`https://keycloak.admin.uds.dev`).

## Zarf Package

```yaml
kind: ZarfPackageConfig
metadata:
  name: myapp
  version: 0.1.0

variables:
  - name: SECRET_KEY
    description: "App secret"
    default: "dev-default"
    sensitive: true

components:
  - name: myapp
    required: true
    charts:
      - name: myapp
        localPath: chart/
        namespace: myapp
        version: 0.1.0
        valuesFiles:
          - values/common-values.yaml
        variables:
          - name: SECRET_KEY
            path: secrets.secretKey
    images:
      - myapp:0.1.0
    actions:
      onDeploy:
        after:
          - wait:
              cluster:
                kind: deployment
                name: myapp
                namespace: myapp
                condition: available
```

## UDS Bundle

```yaml
kind: UDSBundle
metadata:
  name: myapp-bundle
  version: 0.0.1

packages:
  - name: uds-k3d
    repository: ghcr.io/defenseunicorns/packages/uds-k3d
    ref: 0.19.4

  - name: init
    repository: ghcr.io/zarf-dev/packages/init
    ref: v0.71.1

  - name: core
    repository: ghcr.io/defenseunicorns/packages/uds/core
    ref: 0.61.0-upstream

  - name: myapp
    path: ../
    ref: 0.1.0
```

## Bundle Overrides

Override package Helm values from the bundle level:

```yaml
packages:
  - name: core
    repository: ghcr.io/defenseunicorns/packages/uds/core
    ref: 0.61.0-upstream
    overrides:
      keycloak:                    # Zarf component name
        keycloak:                  # Helm chart name
          values:
            - path: env[0]
              value:
                name: JAVA_OPTS_KC_HEAP
                value: "-XX:MaxRAMPercentage=70"
          variables:
            - name: HEAP_OPTIONS
              path: env[0].value
              description: "Keycloak heap config"
      pepr-uds-core:
        module:
          values:
            - path: additionalIgnoredNamespaces
              value: [uds-dev-stack]
```

## Istio Gateways

| Gateway | Domain | Use for |
|---------|--------|---------|
| `tenant` | `*.uds.dev` | User-facing apps |
| `admin` | `*.admin.uds.dev` | Admin tools (Grafana, Keycloak console) |

Custom gateways can be configured for additional domains.

## Network Policies

The `network.allow` field generates Kubernetes NetworkPolicies. Common patterns:

```yaml
network:
  allow:
    # Egress to anywhere (external APIs)
    - direction: Egress
      remoteGenerated: Anywhere

    # Egress to specific in-cluster service
    - direction: Egress
      remoteNamespace: database
      remoteSelector:
        app: postgres
      port: 5432

    # Egress to Kubernetes API
    - direction: Egress
      remoteGenerated: KubeAPI

    # Ingress from another namespace
    - direction: Ingress
      remoteNamespace: other-app
      remoteSelector:
        app: worker
      port: 8080
```

## Monitoring Integration

Add Prometheus scraping via the `monitor` field in the Package CR:

```yaml
spec:
  monitor:
    - selector:
        app.kubernetes.io/name: myapp
      targetPort: 8000
      portName: http
      description: "myapp metrics"
      kind: PodMonitor        # or ServiceMonitor
```

Metrics appear automatically in Grafana at `https://grafana.admin.uds.dev`.

## SSO Secret Templating

UDS Operator auto-generates a Kubernetes Secret with the SSO client credentials. Configure via `secretName` and `secretTemplate` in the Package CR:

```yaml
sso:
  - name: MyApp SSO
    clientId: uds-core-myapp
    secretName: myapp-sso-secret        # K8s secret name (default: auto-generated)
    secretTemplate:
      OIDC_CLIENT_ID: clientField(clientId)
      OIDC_CLIENT_SECRET: clientField(secret)
      OIDC_ISSUER: https://sso.uds.dev/realms/uds
```

The secret is created in your app's namespace. Reference it in your Deployment's `envFrom` or `env`.

## Authservice vs Direct OIDC

| | Authservice (recommended) | Direct OIDC |
|--|---------------------------|-------------|
| **How** | `enableAuthserviceSelector` in Package CR | App handles OIDC flow itself |
| **App code** | Just decode JWT from header | Full OIDC client library needed |
| **Session** | Managed by Authservice (cookie) | App manages sessions |
| **Use when** | Web apps, SPAs, most cases | CLI tools, APIs without browser, device flow |

Authservice works by intercepting requests at the Istio layer. Unauthenticated users get redirected to Keycloak. After login, Authservice injects the JWT into the `Authorization` header.

## UDS Exemptions

When UDS policies block something your app legitimately needs, create an Exemption:

```yaml
apiVersion: uds.dev/v1alpha1
kind: Exemption
metadata:
  name: myapp-exemption
  namespace: uds-policy-exemptions
spec:
  exemptions:
    - matcher:
        namespace: myapp
        name: ".*"
      policies:
        - DisallowPrivileged    # Example: allow privileged containers
```

Use sparingly -- exemptions weaken security posture.

## Constraints

- **One Package CR per namespace** -- each app needs its own namespace
- **Bundle `ref` must match Zarf package `metadata.version`**
- **DO NOT use `oci://` prefix** in bundle `repository` fields
- **Redirect URIs cannot use root wildcards** (`/*`) with Authservice
- **Istio sidecar vs ambient** -- default is ambient mesh mode; set `spec.network.serviceMesh.mode: sidecar` in Package CR if needed
- **Health check endpoints required** -- Istio needs liveness/readiness probes to work properly

## Local Dev Setup

### Prerequisites

Install: [UDS CLI](https://uds.defenseunicorns.com/reference/cli/overview/) (includes `zarf` via `uds zarf`), [k3d](https://k3d.io/), [Docker](https://www.docker.com/)

### From Scratch (~15 min)

```bash
# 1. Build Docker image
docker build -t myapp:0.1.0 .

# 2. Create Zarf package (pulls image from local Docker daemon)
cd uds && uds zarf package create --confirm

# 3. Create UDS bundle (downloads UDS Core ~2GB first time)
uds create bundle/ --confirm

# 4. Deploy (creates k3d cluster + UDS Core + app + test user)
uds deploy bundle/uds-bundle-*.tar.zst --confirm
uds run common-setup:keycloak-user
```

App at `https://myapp.uds.dev`. Login with `doug / unicorn123!@#UN`.

### Redeploy App Only (cluster already running)

```bash
docker build -t myapp:0.1.0 .
cd uds
uds zarf package remove myapp --confirm || true
uds zarf package create --confirm
uds zarf package deploy zarf-package-*.tar.zst --confirm
```

### Teardown

```bash
k3d cluster delete uds
```

### Test User Setup

Include the UDS common tasks in your `tasks.yaml`:

```yaml
# tasks.yaml
includes:
  - common-setup: https://raw.githubusercontent.com/defenseunicorns/uds-common/v1.24.0/tasks/setup.yaml
```

Available tasks from `common-setup`:

| Task | What it does |
|------|-------------|
| `keycloak-user` | Creates a user in the UDS Keycloak realm |
| `keycloak-admin-user` | Creates the Keycloak admin user + stores password in K8s secret |
| `k3d-test-cluster` | Creates a k3d cluster with UDS Core Slim (lighter, for CI) |
| `k3d-full-cluster` | Creates a k3d cluster with full UDS Core |
| `print-keycloak-admin-password` | Prints the admin credentials |

`keycloak-user` accepts inputs:

| Input | Default | Description |
|-------|---------|-------------|
| `username` | `doug` | Username |
| `password` | `unicorn123!@#UN` | Password |
| `first_name` | `Doug` | First name |
| `last_name` | `Unicorn` | Last name |
| `group` | (from `$KEYCLOAK_USER_GROUP`) | Keycloak group to add user to |

Example with custom user:

```bash
uds run common-setup:keycloak-user \
  --set username=testadmin \
  --set password=MyPass123! \
  --set group="/UDS Core/Admin"
```

The task also auto-creates the Keycloak admin user if needed and disables 2FA for the test user.

For custom user attributes (roles, permissions), use the Keycloak Admin console or the [Keycloak Admin REST API](https://www.keycloak.org/docs-api/latest/rest-api/index.html).

**Important:** When updating user attributes via the API, always fetch the existing user first and merge -- a PUT with only new attributes overwrites the entire user object.

## Critical Gotchas

| Gotcha | Fix |
|--------|-----|
| Bundle `repository` has `oci://` prefix | Remove it -- UDS adds it internally |
| Authservice rejects `/*` redirect URI | Use specific path: `/login/generic_oauth` |
| Container crashes with permission denied | Istio may run container as different user -- ensure writable dirs in Dockerfile |
| Same image tag = K8s reuses old pod | Use timestamp tags, sync across chart/zarf/bundle |
| Keycloak user PUT overwrites all data | Fetch existing user, merge attributes with `jq` |
| Port-forward blocks on stale process | Kill existing with `lsof -ti:PORT | xargs kill` before starting |
| `common-setup:keycloak-user` ignores group param | Add group manually via Keycloak Admin API |
| App shows "Unknown" user name | Attributes PUT overwrote name -- must merge, not replace |
| 403 after Keycloak login | User not in required group (`/UDS Core/Admin`) |
| "You do not have access" at Keycloak | User not in the SSO client's `groups.anyOf` group |
| Zarf package uses cached old image | Delete old `.tar.zst`, rebuild with new tag |
| Bundle deploy times out on vector/loki | Resource constraints -- retry `uds deploy` on existing cluster |

## Keycloak Admin Console

- External: `https://keycloak.admin.uds.dev`
- Credentials: `admin` / password from `keycloak-admin-password` secret in `keycloak` namespace

```bash
uds zarf tools kubectl get secret keycloak-admin-password \
  -n keycloak -o jsonpath='{.data.password}' | base64 -d
```
