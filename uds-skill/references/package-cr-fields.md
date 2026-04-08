# UDS Package CR Field Reference

Full spec: [Package CR Spec](https://uds.defenseunicorns.com/reference/configuration/uds-operator/package/)
JSON Schema: [package-v1alpha1.schema.json](https://raw.githubusercontent.com/defenseunicorns/uds-core/refs/heads/main/schemas/package-v1alpha1.schema.json)

## spec.network.expose[]

| Field | Required | Description |
|-------|----------|-------------|
| `host` | yes | Hostname (e.g. `myapp` -> `myapp.uds.dev`) |
| `service` | yes | K8s Service name |
| `port` | yes | Service port |
| `targetPort` | no | Pod port (defaults to `port`) |
| `selector` | no | Pod labels for NetworkPolicy generation |
| `gateway` | no | `tenant` (default), `admin`, `passthrough`, or custom |
| `domain` | no | Domain for custom gateways only |
| `description` | no | VirtualService name suffix |
| `advancedHTTP` | no | CORS, redirects, headers, retries, timeouts |
| `match[]` | no | Request matching rules (not for passthrough) |

## spec.network.allow[]

| Field | Description |
|-------|-------------|
| `direction` | `Ingress` or `Egress` |
| `selector` | Local pod labels |
| `remoteSelector` | Remote pod labels |
| `remoteNamespace` | Remote namespace |
| `port` / `ports` | TCP ports |
| `remoteGenerated` | Preset: `KubeAPI`, `KubeNodes`, `IntraNamespace`, `CloudMetadata`, `Anywhere` |
| `remoteHost` | External hostname |
| `remoteCidr` | External CIDR |
| `serviceAccount` | Restrict by service account |

## spec.network.serviceMesh

| Field | Default | Description |
|-------|---------|-------------|
| `mode` | `ambient` | `ambient` or `sidecar` |

## spec.sso[]

| Field | Required | Description |
|-------|----------|-------------|
| `clientId` | yes | Keycloak client ID |
| `name` | no | Display name |
| `redirectUris[]` | no | Allowed post-login redirect paths |
| `enableAuthserviceSelector` | no | Pod labels to auto-protect with Authservice |
| `groups.anyOf[]` | no | Required Keycloak groups for access |
| `protocolMappers[]` | no | Custom JWT claims |
| `protocol` | no | `openid-connect` (default) or `saml` |
| `publicClient` | no | If true, no client secret required |
| `standardFlowEnabled` | no | Auth code flow |
| `serviceAccountsEnabled` | no | Client credentials grant (machine-to-machine) |
| `secretName` | no | K8s Secret name for client credentials |
| `secretTemplate` | no | Template for secret contents |
| `webOrigins[]` | no | CORS allowlist (`+` = all redirect URIs) |

## spec.monitor[]

| Field | Required | Description |
|-------|----------|-------------|
| `selector` | yes | Service labels |
| `targetPort` | yes | Pod port for scraping |
| `portName` | yes | Service port name |
| `kind` | no | `ServiceMonitor` (default) or `PodMonitor` |
| `path` | no | Scrape endpoint (default: `/metrics`) |
| `description` | no | Monitor name suffix |
| `podSelector` | no | Pod labels (defaults to `selector`) |

## ClusterConfig CR

Singleton resource (`uds-cluster-config`) for cluster-wide settings:

| Field | Description |
|-------|-------------|
| `spec.expose.domain` | Base domain for tenant services |
| `spec.expose.adminDomain` | Domain for admin gateway services |
| `spec.expose.caCert` | CA cert for private PKI domains |
| `spec.networking.kubeApiCIDR` | K8s control plane CIDR |
| `spec.networking.kubeNodeCIDRs[]` | Node CIDR ranges |
| `spec.caBundle` | Custom CA certificates |
| `spec.policy.allowAllNsExemptions` | Allow exemptions in any namespace (default: false) |
