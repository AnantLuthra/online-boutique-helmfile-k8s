# Cloud-Native Microservices Deployment Using Helm and Helmfile on Kubernetes (Linode LKE)

This project deploys the [Google Microservices Demo](https://github.com/GoogleCloudPlatform/microservices-demo) (Online Boutique) to Kubernetes. It uses reusable Helm charts and Helmfile to install the declared application stack from a single command.

## Architecture overview

The application is an end-to-end e-commerce platform made up of 10 microservices and the `redis-cart` cache. The services are written in Go, Python, Java, C#, and Node.js. Internal service-to-service communication uses Kubernetes Services and gRPC/HTTP; only the `frontend` service is exposed externally.

```
                              [ User Traffic ]
                                       │
                              (LoadBalancer: 80)
                                       ▼
                              ┌─────────────────┐
                              │ frontend (Go)   │
                              └────────┬────────┘
        ┌───────────────────┬──────────┼──────────┬──────────────────┐
        ▼                   ▼          ▼          ▼                  ▼
┌───────────────┐   ┌─────────────┐ ┌─────┐ ┌─────────────┐  ┌───────────────┐
│ productcatalog│   │ recommend-  │ │ cart│ │ checkout    │  │ adservice     │
│ service (Go)  │   │ ation (Py)  │ │ (C#)│ │ service(Go) │  │ (Java)        │
└───────┬───────┘   └──────┬──────┘ └──┬──┘ └──────┬──────┘  └───────────────┘
        │                  │           │           │
        └──────────────────┘           ▼           ├──► currencyservice (Node.js)
                                 ┌───────────┐     ├──► paymentservice (Node.js)
                                 │redis-cart │     ├──► shippingservice (Go)
                                 │ (Redis)   │     └──► emailservice (Python)
                                 └───────────┘
```

```text
User traffic
    |
    v
frontend Service (LoadBalancer, port 80)
    |
    +-- productcatalogservice, recommendationservice, cartservice
    +-- checkoutservice, adservice, currencyservice
    +-- paymentservice, shippingservice, emailservice
                         |
                         v
                    redis-cart
```


## Helm and Helmfile layout

The earlier single, consolidated Kubernetes manifest has been replaced by reusable charts. The legacy [`config.yaml`](config.yaml) remains in the repository for reference, but Helmfile is the supported way to create or remove this stack.

| Path | Purpose |
| --- | --- |
| [`charts/microservice`](charts/microservice) | Reusable Helm chart for an application microservice Deployment and Service. |
| [`charts/redis`](charts/redis) | Dedicated Helm chart for the `redis-cart` Deployment and ClusterIP Service. |
| [`values`](values) | Per-component values files containing image, ports, environment variables, replica counts, capacity settings, and Service settings. |
| [`helmfile.yaml`](helmfile.yaml) | Declares the releases and connects each release to its chart and corresponding values file. |

The microservice chart is shared by the frontend and application services; the Redis chart is separate because Redis has its own volume and TCP health-check configuration. To add or modify a component, update its file in `values/` and, when adding a release, declare it in `helmfile.yaml`.


## Key implementation decisions

- **Pinned application images:** Online Boutique images use the explicit `v0.10.7` tag for repeatable deployments.
- **Health checks:** Application containers use liveness and readiness probes; Redis uses TCP probes on port `6379`.
> **Health Check note:** Liveness and readiness probes are currently defined in `config.yaml` but not yet in the Helm charts. The frontend uses `httpGet`, while the other microservices use `grpc`. Since all microservices currently share a single Helm chart, adding a `grpc` probe would incorrectly apply it to the frontend as well. Therefore, the health checks are kept in `config.yaml` for now.
- **Capacity-aware defaults:** Microservice CPU and memory requests/limits are disabled by default so small clusters do not fail scheduling because of CPU allocation limits. Redis resource controls remain enabled.
- **Availability:** Stateless application services use two replicas. Redis remains a single `emptyDir`-backed instance, which is appropriate only for this learning environment.

## Deploy the cluster

### Prerequisites

- A running Kubernetes cluster (Linode Kubernetes Engine, Minikube, or another Kubernetes provider).
- `kubectl` configured for that cluster.
- Helm 3.
- Helmfile CLI.

Confirm that Helmfile can reach the intended cluster before making changes:

```bash
kubectl cluster-info
helmfile list
```

### Install or update all releases

From the repository root, run:

```bash
helmfile apply
```

`helmfile apply` renders the two local charts with the overrides in `values/`, shows the planned changes, and installs or upgrades the declared releases. It is safe to run again after changing a values file.

### Resource requests and limits

The `resources` block in [`charts/microservice/templates/deployment.yaml`](charts/microservice/templates/deployment.yaml) is currently commented out. Although the per-service files in `values/` contain example CPU and memory values, Helm does not render those values while that block is commented.

For a small or capacity-constrained cluster, leave the block disabled. If the cluster has sufficient capacity and you want Kubernetes scheduling guarantees and limits, uncomment the `resources`, `requests`, and `limits` lines in the microservice Deployment template, then tune `requestedCpu`, `requestedMemory`, `limitCpu`, and `limitMemory` in every applicable file under `values/` before running `helmfile apply` again. Choose values that fit the allocatable CPU and memory of your worker nodes.

### Verify the rollout

Wait until the Pods are ready:

```bash
kubectl get pods --watch
```

Inspect the Services and Helm releases:

```bash
kubectl get services
helmfile list
```

### Access the application

The frontend is configured as a Kubernetes `LoadBalancer` Service on port `80`. On a cloud Kubernetes provider, creating this Service asks the provider to provision an external load balancer. Wait for the `EXTERNAL-IP` (or hostname) to be assigned:

```bash
kubectl get service frontend --watch
```

Then open:

```text
http://<EXTERNAL-IP-OR-HOSTNAME>
```

Local Kubernetes environments may not automatically provision a load balancer; use that environment's LoadBalancer support (for example, `minikube tunnel` or `minikube service 'loadbalancer-name'`) if necessary.

## Cleanup

Remove every release declared by Helmfile:

```bash
helmfile destroy
```

Confirm that the frontend Service has been removed:

```bash
kubectl get service frontend
```

> **Important:** The frontend `LoadBalancer` Service can create a billable cloud-provider load balancer. Run `helmfile destroy` when you no longer need the environment, then verify in both Kubernetes and your cloud-provider console that the load balancer has been deleted. This helps prevent unwanted charges.

## Learning-environment considerations

- Resources are deployed to the Kubernetes `default` namespace. Use a dedicated namespace for stronger isolation in production.
- `redis:alpine` is not version-pinned; pin it to a Redis version (and ideally a digest) before production use.
- For production external traffic, also add TLS, authentication/authorization controls, and an Ingress or provider-specific load-balancer policy as appropriate.
