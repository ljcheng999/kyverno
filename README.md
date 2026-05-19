# kyverno (GKE Standard)

Production-grade Kyverno deployment for GKE Standard with:

- Kyverno Helm chart pinned to 3.8.0 (Kyverno 1.18.0)
- High availability replicas:
  - admission-controller: 3
  - background-controller: 2
  - cleanup-controller: 2
  - reports-controller: 2
- Atomic install/upgrade with wait and timeout support
- Pod anti-affinity + topology spread constraints across zones
- PodDisruptionBudgets for all controllers
- kyverno-policies chart installation for pod-security policy bundle in Audit mode
- Namespace PSA label mutation policy (audit/warn=restricted, no enforce label)
- Pod securityContext mutate policy for defaults on pod/containers/initContainers

## Chart Layout

- Chart definition: [Chart.yaml](Chart.yaml)
- Base config: [values.yaml](values.yaml) (common settings for all environments)
- Custom policies: [templates/mutating-policy-add-restricted-namespace-pss-labels.yaml](templates/mutating-policy-add-restricted-namespace-pss-labels.yaml), [templates/validating-policy-disallow-privileged-containers.yaml](templates/validating-policy-disallow-privileged-containers.yaml)

## Prerequisites

- Kubernetes 1.25+
- Helm 3.x
- kubectl

## Install

### Production (recommended)

```bash
helm dependency build charts/kyverno
helm upgrade --install kyverno charts/kyverno \
  --namespace kyverno --create-namespace \
  --wait --atomic --timeout 15m \
  -f charts/kyverno/values.yaml
```

## Upgrade

```bash
helm dependency build charts/kyverno
helm upgrade --install kyverno charts/kyverno \
  --namespace kyverno \
  --wait --atomic --timeout 15m \
  -f charts/kyverno/values.yaml
```

## Uninstall

```bash
helm uninstall kyverno --namespace kyverno
```




## Rollback

List revisions:

```bash
helm history kyverno -n kyverno
```

Rollback to a known good revision:

```bash
helm rollback kyverno <REVISION> -n kyverno --wait --timeout 15m
```

Re-run verification:

```bash
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-admission-controller --timeout=300s
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-background-controller --timeout=300s
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-cleanup-controller --timeout=300s
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-reports-controller --timeout=300s
kubectl get cpol add-restricted-psa-labels
kubectl get cpol add-default-pod-and-container-securitycontext
```

## Namespace Exclusion Flexibility

You can exclude any namespaces from PSA labeling and default securityContext mutation.

### PSA Labeling Exclusions (`namespacePsa.excludedNamespaces`)

The base `values.yaml` excludes **system and infrastructure namespaces** that are common across all environments (dev, SRE, and higher). These are namespaces managed by the platform team or cloud provider — not application/workload namespaces:

- **Kubernetes system:** kube-system, kube-public, kube-node-lease
- **GKE managed:** gke-managed-filestorecsi, gke-managed-networking-dra-driver, gke-managed-system, gke-managed-volumepopulator
- **Security / policy:** gatekeeper-system, kyverno, upwind
- **Service mesh:** istio-system
- **Messaging:** nats, nats-io, nats-sts
- **Observability:** dynatrace, otel, monitoring, logs
- **Infrastructure / platform:** akuity, apigee, apigee-httpbin, argo-rollouts, actions-runner-system, cnrm-system, configconnector-operator-system, crossplane, exa-logstash-operator-system, external-secrets, infrastructure, keda, kubefed, perfectscale, windmill

> **Note:** `globalExcludedNamespaces` (in values-dev.yaml / values-sre.yaml) does **NOT** affect PSA labeling. It only controls security context mutation exclusions.

To add environment-specific PSA exclusions, use `additionalExcludedNamespaces`:

```yaml
namespacePsa:
  additionalExcludedNamespaces:
    - my-custom-infra-namespace
```

### Security Context Mutation Exclusions (`podSecurityContextDefaults.excludedNamespaces`)

These policies use a separate exclusion list that **does** include `globalExcludedNamespaces`. The base `excludedNamespaces` covers the same system namespaces, and per-environment `globalExcludedNamespaces` adds all existing application namespaces to prevent breaking running workloads.

Add your own excludes:

```yaml
podSecurityContextDefaults:
  additionalExcludedNamespaces:
    - my-namespace
```

## ArgoCD App-of-Apps / ApplicationSet Integration

This chart is managed through the [argocd-app-of-apps](https://github.com/Exabeam/argocd-app-of-apps) repository from the infrastructure folder using an `ApplicationSet`:

- [infrastructure/dev/kyverno.yaml](https://github.com/Exabeam/argocd-app-of-apps/blob/main/infrastructure/dev/kyverno.yaml) - Generates Kyverno apps for dev and sre clusters

### Environment-Specific Configuration

Each environment uses the base `values.yaml` plus environment-specific overrides:


### Key Differences by Environment

| Setting | Base (values.yaml) | SRE (values-sre.yaml) | Dev (values-dev.yaml) |
|---------|---|---|---|
| Admission Controller Replicas | 3 | 3 (override) | 3 |
| Background/Cleanup/Reports Replicas | 2 | 2 (override) | 2 |
| Namespace PSA Audit Label | restricted | restricted | restricted |
| Namespace PSA Warn Label | restricted | restricted | (disabled) |
| PodDisruptionBudgets (minAvailable) | admission=2, others=1 | admission=2, others=1 | admission=2, others=1 |
| PSA Label Exclusions | 33 system/infra ns | 33 base (shared) | 33 base (shared) |
| Security Context Exclusions | 33 base | 33 base + 74 global | 33 base + 107 global |

## Troubleshooting

- Dependency fetch fails:
  - Run `helm repo add kyverno https://kyverno.github.io/kyverno/` and retry.
- Pods not spreading across zones:
  - Ensure node labels include `topology.kubernetes.io/zone` in your GKE node pools.
- Policy not mutating expected resources:
  - Check namespace exclusion lists in values files.
  - Check policy events: `kubectl get events -A | grep -i kyverno`.
- Admission timeouts:
  - Verify Kyverno admission controller replicas are healthy and webhook service is reachable.
- ArgoCD drift:
  - Ensure ArgoCD has network access to pull chart dependencies from kyverno chart repository.

