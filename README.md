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

## Skills demonstrated

- Choosing between env-var and file-mount configuration patterns based on what the workload actually consumes, not defaulting to one pattern everywhere
- Using `subPath` to mount a single file from a ConfigMap/Secret without overwriting the entire target directory — a common trap for anyone mounting config into a directory that also needs other files
- Correctly exposing a service both internally (`ClusterIP`) and to the host (`NodePort`) depending on which is needed

## Key technical decisions

| Decision | Why |
|---|---|
| Env vars for MongoDB, files for Mosquitto | Matches how each application actually expects its configuration — MongoDB clients read connection strings from env, Mosquitto reads a config file path. |
| `subPath` for the Mosquitto config mount | Without it, mounting a ConfigMap as a volume replaces the entire target directory, which would wipe out anything else expected to live there. |
| `NodePort` for Mongo Express, `ClusterIP` for MongoDB itself | Only the admin UI needs host access; the database itself should stay internal-only. |

## Limitations

- Local Minikube only — no cloud LoadBalancer or Ingress in this project (see `helm-lke-mongodb` / `k8s-microservices-helm` for cloud-exposed variants).
- No TLS validation performed on the Mosquitto broker beyond mounting the certificate file.

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
