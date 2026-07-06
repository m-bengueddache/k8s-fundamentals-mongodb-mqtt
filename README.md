# Kubernetes Fundamentals — MongoDB & Mosquitto MQTT

> **FR** — Deux services avec des besoins de configuration très différents sur le même cluster Minikube : MongoDB configuré via variables d'environnement (ConfigMap/Secret), et un broker MQTT Mosquitto configuré via fichiers montés (volumes).
>
> **EN** — Two services with very different configuration needs on the same Minikube cluster: MongoDB configured through environment variables (ConfigMap/Secret), and a Mosquitto MQTT broker configured through mounted files (volumes).

![Kubernetes](https://img.shields.io/badge/Kubernetes-Minikube-326CE5?logo=kubernetes)
![MongoDB](https://img.shields.io/badge/MongoDB-latest-green?logo=mongodb)
![Mosquitto](https://img.shields.io/badge/Eclipse%20Mosquitto-MQTT-3C5280)

---

## Problem

`ConfigMap` and `Secret` can back an application two different ways — as environment variables, or as mounted files — and picking the wrong one for a given workload causes real friction: some tools only read configuration from a file on disk, others expect env vars, and a certificate simply can't be an environment variable in any sane way. This project deploys one service each way, deliberately, to make the distinction concrete rather than theoretical.

## Solution

MongoDB + Mongo Express is deployed with configuration and credentials injected as environment variables (`ConfigMap` for the connection URL, `Secret` for credentials) — the natural fit for an application that reads `process.env`. A Mosquitto MQTT broker is deployed with its configuration file and SSL certificate mounted as actual files inside the container via `volumes` + `volumeMounts` and `subPath` — the natural fit for a service that expects a config file path, not environment variables.

## The Kubernetes object model, from first principles

Before either service, the basics: pods are the smallest deployable unit, but they're never created or managed directly. A **Deployment** is the blueprint — name, image, replica count — and Kubernetes manages a **ReplicaSet** automatically underneath it to keep the requested number of pods running.

```
Deployment            (what you configure: image, replicas, updates)
  └── ReplicaSet       (managed automatically by Kubernetes)
        └── Pod        (abstraction around one or more containers)
              └── Container
```

This shows up directly in resource naming: `kubectl get deployment` shows `nginx-deployment`; `kubectl get replicaset` shows `nginx-deployment-6ff797d4c9`; `kubectl get pod` shows `nginx-deployment-6ff797d4c9-ln8lz` — each layer's name is derived from the one above it. Editing a Deployment (e.g. `kubectl edit deployment nginx-deployment` to change the image) doesn't patch the running pod in place: Kubernetes creates a *new* ReplicaSet, scales it up, and scales the old one to zero — visible directly with `kubectl get replicaset` showing the old ReplicaSet at 0 pods and a new one at the requested count.

`kubectl apply -f file.yaml` is idempotent by design: the first run reports `created`, every subsequent run (after editing the file) reports `configured` — Kubernetes diffs the manifest against the live object rather than blindly recreating it.

## Exposing a service via Ingress (NGINX + Helm)

Every service above reaches the outside world through `NodePort`, which works but means one port per service and no hostname-based routing. On this same cluster, the Kubernetes Dashboard was instead exposed through an **Ingress**, installed and wired up as follows:

1. **Install the NGINX Ingress Controller via Helm** — an Ingress resource is just a routing rule; it does nothing without a controller running in the cluster to read it:
   ```bash
   helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress
   ```
2. **Define the routing rule**, in the same namespace as the target Service:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: dashboard-ingress
     namespace: kubernetes-dashboard
   spec:
     ingressClassName: nginx
     rules:
     - host: dashboard.com
       http:
         paths:
           - path: /
             pathType: Prefix
             backend:
               service:
                 name: kubernetes-dashboard
                 port:
                   number: 80
   ```
3. **Map the hostname locally** (`127.0.0.1 dashboard.com` in `/etc/hosts`) and run `minikube tunnel` — Minikube has no real cloud load balancer, so the tunnel is what makes the Ingress Controller's address reachable from the browser at all.

The **default backend** is what the Ingress Controller falls back to for any request that matches no rule (wrong host, unmatched path) — a generic `404` by default, visible via `kubectl describe ingress`.

## Skills demonstrated

- Understanding what `kubectl apply` and a Deployment edit actually do underneath (ReplicaSet replacement, not in-place pod mutation) rather than treating `kubectl` commands as opaque
- Choosing between env-var and file-mount configuration patterns based on what the workload actually consumes, not defaulting to one pattern everywhere
- Using `subPath` to mount a single file from a ConfigMap/Secret without overwriting the entire target directory — a common trap for anyone mounting config into a directory that also needs other files
- Correctly exposing a service both internally (`ClusterIP`) and to the host (`NodePort`) depending on which is needed
- Installing an Ingress Controller via Helm and understanding that an Ingress resource without a controller reading it does nothing

## Key technical decisions

| Decision | Why |
|---|---|
| Env vars for MongoDB, files for Mosquitto | Matches how each application actually expects its configuration — MongoDB clients read connection strings from env, Mosquitto reads a config file path. |
| `subPath` for the Mosquitto config mount | Without it, mounting a ConfigMap as a volume replaces the entire target directory, which would wipe out anything else expected to live there. |
| `NodePort` for Mongo Express, `ClusterIP` for MongoDB itself | Only the admin UI needs host access; the database itself should stay internal-only. |

## Limitations

- Local Minikube only — Ingress here relies on `minikube tunnel`, not a real cloud load balancer (see `helm-lke-mongodb` / `k8s-microservices-helm` for cloud-exposed variants).
- No TLS configured on the Ingress or the Mosquitto broker beyond mounting the certificate file.

## Roadmap

- [ ] Add a small MQTT publisher/subscriber script to validate the broker end to end, not just that the pod starts

---

## FR — Détails d'implémentation

### MongoDB + Mongo Express

`ConfigMap`, `Secret`, `Deployment`, `Service` ClusterIP et NodePort. Les variables d'environnement des pods référencent des valeurs depuis un `ConfigMap` (URL de connexion) et un `Secret` (credentials). Un service `ClusterIP` expose MongoDB en interne ; un `NodePort` expose Mongo Express depuis l'hôte via `minikube service`.

### Mosquitto MQTT Broker avec Volumes

`ConfigMap` (fichier de config), `Secret` (certificat SSL), `Deployment` + `Service`. Contrairement à MongoDB, le `ConfigMap` et le `Secret` sont montés comme des fichiers dans le conteneur, pas comme des variables d'environnement. `subPath` permet de monter un fichier unique sans écraser le répertoire de destination entier.

## EN — Implementation Details

### MongoDB + Mongo Express

`ConfigMap`, `Secret`, `Deployment`, `Service` ClusterIP and NodePort. Pod environment variables reference values from a `ConfigMap` (connection URL) and a `Secret` (credentials). A `ClusterIP` service exposes MongoDB internally; a `NodePort` exposes Mongo Express from the host via `minikube service`.

### Mosquitto MQTT Broker with Volumes

`ConfigMap` (config file), `Secret` (SSL certificate), `Deployment` + `Service`. Unlike MongoDB, the `ConfigMap` and `Secret` are mounted as files in the container, not as environment variables. `subPath` mounts a single file without overwriting the entire target directory.

---

## Prerequisites

- Minikube, `kubectl`

## Project Structure

```
.
├── mongodb/     # ConfigMap, Secret, Deployment, Service (env-var configuration)
└── mosquitto/   # ConfigMap, Secret, Deployment, Service (file-mount configuration)
```
