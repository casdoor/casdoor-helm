# Casdoor Helm Chart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/casdoor)](https://artifacthub.io/packages/helm/casdoor/casdoor-helm-charts)

[Casdoor](https://casdoor.org) is an open-source Identity and Access Management (IAM) platform supporting OAuth 2.0, OIDC, SAML, and LDAP. This repository provides the official Helm chart for deploying Casdoor on Kubernetes.

Official documentation: <https://casdoor.org/docs/basic/try-with-helm>

---

## Prerequisites

- Kubernetes 1.19+
- Helm v3.8+

---

## Installation

```shell
helm install casdoor oci://registry-1.docker.io/casbin/casdoor-helm-charts --version <version>
```

Check available versions on [Docker Hub](https://hub.docker.com/r/casbin/casdoor-helm-charts/tags).

To install with a custom values file:

```shell
helm install casdoor oci://registry-1.docker.io/casbin/casdoor-helm-charts \
  --version <version> \
  -f my-values.yaml
```

---

## Upgrading

```shell
helm upgrade casdoor oci://registry-1.docker.io/casbin/casdoor-helm-charts --version <version>
```

---

## Uninstalling

```shell
helm uninstall casdoor
```

---

## Configuration

Override any value from [values.yaml](charts/casdoor/values.yaml) using `--set` or a custom values file.

### Core parameters

| Parameter | Description | Default |
|---|---|---|
| `replicaCount` | Number of Casdoor pods | `1` |
| `image.repository` | Docker image repository | `casbin` |
| `image.name` | Docker image name | `casdoor` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `image.tag` | Image tag (defaults to chart appVersion) | `""` |
| `config` | Casdoor `app.conf` content (Go template) | See values.yaml |
| `configFromSecret` | Name of an existing Secret holding `app.conf` | `""` |

### Database

| Parameter | Description | Default |
|---|---|---|
| `database.driver` | Database driver: `mysql`, `postgres`, `cockroachdb`, `sqlite` | `sqlite` |
| `database.user` | Database username | `""` |
| `database.password` | Database password | `""` |
| `database.host` | Database host | `""` |
| `database.port` | Database port (empty = driver default) | `""` |
| `database.databaseName` | Database name | `casdoor` |
| `database.sslMode` | SSL mode for the connection | `disable` |

### LDAP

| Parameter | Description | Default |
|---|---|---|
| `ldap.enabled` | Enable the built-in LDAP server | `false` |
| `ldap.service.port` | LDAP service port | `389` |

### Service

| Parameter | Description | Default |
|---|---|---|
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `8000` |

### Ingress

Exposes Casdoor via a standard Kubernetes `Ingress` resource.

| Parameter | Description | Default |
|---|---|---|
| `ingress.enabled` | Enable Ingress | `false` |
| `ingress.className` | IngressClass name | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | List of `{host, paths}` entries | See values.yaml |
| `ingress.tls` | TLS configuration | `[]` |

Example:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: casdoor.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: casdoor-tls
      hosts:
        - casdoor.example.com
```

### Gateway API

Exposes Casdoor via the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) (`HTTPRoute` + optional `Gateway`). This is the modern successor to Ingress, supported by Istio, Envoy Gateway, Cilium, Kong, NGINX Gateway Fabric, and others.

**Requires:** Gateway API CRDs (`gateway.networking.k8s.io/v1`) installed in the cluster and a compatible controller running.

#### Core Gateway API parameters

| Parameter | Description | Default |
|---|---|---|
| `gatewayApi.enabled` | Enable HTTPRoute creation | `false` |
| `gatewayApi.createGateway` | Also create a `Gateway` resource | `false` |
| `gatewayApi.annotations` | Annotations for the HTTPRoute | `{}` |
| `gatewayApi.labels` | Extra labels for the HTTPRoute | `{}` |
| `gatewayApi.parentRefs` | Parent Gateway references for the HTTPRoute | `[]` |
| `gatewayApi.hostnames` | Hostnames to match (Host header) | `[]` |
| `gatewayApi.rules` | Routing rules (matches, filters, backendRefs) | PathPrefix `/` → chart Service |

#### Gateway creation parameters (`gatewayApi.gateway.*`)

Only used when `gatewayApi.createGateway=true`.

| Parameter | Description | Default |
|---|---|---|
| `gatewayApi.gateway.name` | Gateway resource name (defaults to chart fullname) | `""` |
| `gatewayApi.gateway.gatewayClassName` | **Required.** GatewayClass name (e.g. `istio`, `envoy`) | `""` |
| `gatewayApi.gateway.annotations` | Annotations for the Gateway | `{}` |
| `gatewayApi.gateway.labels` | Extra labels for the Gateway | `{}` |
| `gatewayApi.gateway.listeners` | Gateway listeners | HTTP:80 (see values.yaml) |

#### HTTPS redirect parameters (`gatewayApi.httpsRedirect.*`)

Creates a second HTTPRoute that redirects HTTP traffic to HTTPS.

| Parameter | Description | Default |
|---|---|---|
| `gatewayApi.httpsRedirect.enabled` | Enable the HTTP→HTTPS redirect HTTPRoute | `false` |
| `gatewayApi.httpsRedirect.httpListenerName` | HTTP listener `sectionName` on the Gateway | `http` |
| `gatewayApi.httpsRedirect.httpsListenerName` | HTTPS listener `sectionName` on the Gateway | `https` |
| `gatewayApi.httpsRedirect.hostnames` | Hostnames for the redirect route (falls back to `gatewayApi.hostnames`) | `[]` |
| `gatewayApi.httpsRedirect.statusCode` | Redirect response code | `301` |
| `gatewayApi.httpsRedirect.parentRefs` | Override parentRefs for the redirect route | `[]` |

#### Example: attach to an existing Gateway

```yaml
gatewayApi:
  enabled: true
  parentRefs:
    - name: my-gateway
      namespace: gateway-system
      sectionName: https
  hostnames:
    - casdoor.example.com
```

#### Example: create a new Gateway (Istio) with HTTP→HTTPS redirect

```yaml
gatewayApi:
  enabled: true
  createGateway: true
  hostnames:
    - casdoor.example.com
  gateway:
    gatewayClassName: istio
    listeners:
      - name: http
        protocol: HTTP
        port: 80
        allowedRoutes:
          namespaces:
            from: Same
      - name: https
        protocol: HTTPS
        port: 443
        tls:
          certificateRefs:
            - name: casdoor-tls
              kind: Secret
        allowedRoutes:
          namespaces:
            from: Same
  httpsRedirect:
    enabled: true
```

### Resources and scaling

| Parameter | Description | Default |
|---|---|---|
| `resources` | CPU/memory requests and limits | `{}` |
| `autoscaling.enabled` | Enable HorizontalPodAutoscaler | `false` |
| `autoscaling.minReplicas` | HPA minimum replicas | `1` |
| `autoscaling.maxReplicas` | HPA maximum replicas | `100` |
| `autoscaling.targetCPUUtilizationPercentage` | HPA CPU target | `80` |

### Scheduling

| Parameter | Description | Default |
|---|---|---|
| `nodeSelector` | Node selector labels | `{}` |
| `tolerations` | Pod tolerations | `[]` |
| `affinity` | Pod affinity/anti-affinity rules | `{}` |
| `priorityClassName` | Pod priority class | `""` |

### Extra containers and volumes

| Parameter | Description | Default |
|---|---|---|
| `initContainersEnabled` | Enable init containers | `false` |
| `initContainers` | Init container definitions (YAML string) | `""` |
| `extraContainersEnabled` | Enable sidecar containers | `false` |
| `extraContainers` | Sidecar container definitions (YAML string) | `""` |
| `extraVolumeMounts` | Extra volume mounts for the Casdoor container | `[]` |
| `extraVolumes` | Extra volumes for the pod | `[]` |
| `defaultConfigVolumeEnabled` | Mount the default config volume at `/conf` | `true` |

### Environment variables

| Parameter | Description | Default |
|---|---|---|
| `envFromSecret` | Env vars sourced from individual Secret keys | `[]` |
| `envFromConfigmap` | Env vars sourced from individual ConfigMap keys | `[]` |
| `envFrom` | Env vars from entire Secrets or ConfigMaps | `[]` |

### Probes

| Parameter | Description | Default |
|---|---|---|
| `probe.readiness.enabled` | Enable readiness probe | `true` |
| `probe.liveness.enabled` | Enable liveness probe | `true` |

---

## Repository structure

```
charts/
└── casdoor/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── configmap.yaml
        ├── deployment.yaml
        ├── gateway.yaml          # Gateway API: Gateway resource
        ├── httproute.yaml        # Gateway API: main HTTPRoute
        ├── httproute-redirect.yaml  # Gateway API: HTTP→HTTPS redirect HTTPRoute
        ├── hpa.yaml
        ├── ingress.yaml
        ├── secrets.yaml
        ├── service.yaml
        └── serviceaccount.yaml
```

---

## License

Apache 2.0 — see [LICENSE](LICENSE).
