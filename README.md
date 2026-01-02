# Envoy File-based SDS Deployment

- [Envoy File-based SDS Deployment](#envoy-file-based-sds-deployment)
  - [Sequence](#sequence)
  - [Deployment Steps](#deployment-steps)
    - [1. Create kind Cluster](#1-create-kind-cluster)
    - [2. Prepare Certificates](#2-prepare-certificates)
    - [3. Create ConfigMaps](#3-create-configmaps)
    - [4. Apply Deployment](#4-apply-deployment)
  - [Zero-Downtime Certificate Rotation](#zero-downtime-certificate-rotation)
    - [1. Check Certificates](#1-check-certificates)
    - [2. Generate New Certificates](#2-generate-new-certificates)
    - [3. Update ConfigMap](#3-update-configmap)
    - [4. Check Certificates](#4-check-certificates)

Deployment procedure for certificate management on Kubernetes using file-based SDS (Secret Discovery Service) via ConfigMaps.

## Sequence

```mermaid
sequenceDiagram
    participant OS as OS / Kernel
    participant Main as Envoy Main Process
    participant Config as Config Loader
    participant Cluster as Cluster Manager
    participant Init as Init Manager
    participant SDS as SDS (FileSystem)
    participant Worker as Worker Threads

    Note over Main: Epoch 0 Initializing
    Main->>Main: Validate statically linked extensions
    Main->>Main: Initialize HTTP header map definitions

    Note over Main, Config: Start loading config (envoy.yaml)
    Config->>Main: 0 static secrets loaded
    Config->>Main: 0 clusters loaded
    Config->>Main: 1 listener loaded

    rect rgb(255, 240, 240)
        Note right of Config: Warnings Occurred
        Config-->>Main: Warning: resource_api_version not set (AUTO)
        Config-->>Main: Warning: internal_address_config not set
    end

    Main->>Init: Start Init Manager
    
    rect rgb(240, 255, 240)
        Note over Init, SDS: Resolving dependencies (SDS, etc.)
        Init->>SDS: Start loading certificate resources
        SDS-->>Init: Resource loading complete
    end

    Init->>Cluster: Instruction to initialize clusters
    Cluster-->>Init: All clusters initialized (cm init)

    Note over Main: All clusters initialized
    Main->>Init: Confirm all dependencies are ready

    rect rgb(240, 240, 255)
        Note over Main, Worker: Execution Phase
        Main->>Worker: Start worker threads
        Main->>Main: Start main dispatch loop
    end

    Note over Worker: Listening for requests (Port 10000)

```

## Deployment Steps

### 1. Create kind Cluster

```bash
kind create cluster --name envoy-sds-lab

```

### 2. Prepare Certificates

If you don't have certificates yet, generate them in the `dmmy_keys` directory.

```bash
openssl req -x509 -newkey rsa:2048 -keyout dmmy_keys/key.pem -out dmmy_keys/cert.pem -sha256 -days 365 -nodes -subj "/CN=envoy-sds-v1"

```

### 3. Create ConfigMaps

Create two ConfigMaps: one for the Envoy static configuration and another for the certificates and SDS resource definitions.

```bash
# Envoy static configuration
kubectl create configmap envoy-static-config --from-file=envoy.yaml=envoy/enovy.yaml

# SDS resources and certificates (these are updated as a set)
kubectl create configmap sds-config \
  --from-file=sds-resource.yaml=envoy/sds-resource.yaml \
  --from-file=cert.pem=dmmy_keys/cert.pem \
  --from-file=key.pem=dmmy_keys/key.pem
```

### 4. Apply Deployment

```bash
kubectl apply -f k8s/envoy-deploy.yaml
```

---

## Zero-Downtime Certificate Rotation

This procedure replaces the certificates without restarting the Pod.

### 1. Check Certificates

```bash
kubectl port-forward deployment/envoy-sds 10000:10000
```

```bash
curl -vk https://localhost:10000 2>&1 | grep "subject:"
# *  subject: CN=envoy-sds-v1
```

### 2. Generate New Certificates

```bash
openssl req -x509 -newkey rsa:2048 -keyout dmmy_keys/key.pem -out dmmy_keys/cert.pem -sha256 -days 365 -nodes -subj "/CN=envoy-sds-v2-updated"
```

### 3. Update ConfigMap

Overwrite the existing ConfigMap using `kubectl apply`.

```bash
kubectl create configmap sds-config \
  --from-file=sds-resource.yaml=envoy/sds-resource.yaml \
  --from-file=cert.pem=dmmy_keys/cert.pem \
  --from-file=key.pem=dmmy_keys/key.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 4. Check Certificates

```bash
curl -vk https://localhost:10000 2>&1 | grep "subject:"
# *  subject: CN=envoy-sds-v2-updated
```
