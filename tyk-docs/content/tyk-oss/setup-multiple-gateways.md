---
title: "Running multiple instances of Tyk Gateway"
date: 2023-08-01
tags: ["OSS", "Gateways", "Kubernetes"]
description: "Running multiple instances of Tyk Gateway"
menu:
  main:
    parent: "Tyk Helm Chart"
weight: 1
---

## Introduction

Running multiple instances of Tyk Gateway in Kubernetes can be tricky, as Tyk Dashboard (a licensed component) that helps synchronise API configurations across instances of gateways is missing.

By default, Gateway stores API configurations at /mnt/tyk-gateway/apps inside the Gateway container. There are a a few challenges:
- Multiple gateways do not share app configs
- The configuration is not persistent. It got lost whenever pod restart

The same applies to Security Policies and middleware too which is stored at /mnt/tyk-gateway/policies and /mnt/tyk-gateway/middleware respectively.

This can be solved by instantiating a Persistent Volume as shared storage for the gateway instances. As each gateway is reload, they would get the API configurations from the same storage, solving the synchronisation issue between gateways. Also, the storage is persistent and can be designed to be not impacted by cluster failure, your API configurations can be maintained after pod restart.

{{< img src="/img/diagrams/multiple-gateways" alt="multiple-gateways" >}}

 
## Steps

### 1. Create PersistentVolume and PersistentVolumeClaim

Check your Kubernetes platform what kind of persistent storage is supported. Some platform supports Dynamic Storage Provisioning so you (or the Cluster Admin) do not need to create a PersistentVolume in advance. When an application requests storage through PersistentVolumeClaim, the corresponding persistent volume type would be provisioned. You can often specify type of storage through storageClassName.

Example: A PersistentVolumeClaim requesting Azure Disk storage on AKS:

`pvc-azure.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tyk-app-claim
spec:
  storageClassName: managed-csi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tyk-policies-claim
spec:
  storageClassName: managed-csi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tyk-middleware-claim
spec:
  storageClassName: managed-csi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### 2. Install Multiple Instances of Tyk Gateways

The following settings would generate 3 replicas of gateway that get Load Balanced behind the Azure Application Gateway. The gateway share the same persistent volume to store the APIs, policies, and middleware config.

```bash
kubectl apply -f pvc-azure.yaml -n $NAMESPACE

helm upgrade tyk-oss tyk-helm/tyk-oss -n $NAMESPACE --create-namespace --devel \
  --install \
  --set global.secrets.APISecret="$APISecret" \
  --set global.redis.addrs="{tyk-redis-master.$NAMESPACE.svc.cluster.local:6379}" \
  --set global.redis.pass="$(kubectl get secret --namespace $NAMESPACE tyk-redis -o jsonpath='{.data.redis-password}' | base64 -d)" \
  --set tyk-gateway.gateway.hostName=$HOSTNAME_CLOUD \
  --set tyk-gateway.gateway.image.tag=$TykVersion \
  --set global.tls.gateway=false \
  --set tyk-gateway.gateway.ingress.enabled=true \
  --set-json 'tyk-gateway.gateway.ingress.annotations={"kubernetes.io/ingress.class": "azure/application-gateway"}' \
  --set "tyk-gateway.gateway.ingress.hosts[0].host=$HOSTNAME_CLOUD" \
  --set "tyk-gateway.gateway.ingress.hosts[0].paths[0].path=/" \
  --set "tyk-gateway.gateway.ingress.hosts[0].paths[0].pathType=Exact" \
  --set tyk-gateway.gateway.control.enabled=true \
  --set tyk-gateway.gateway.replicaCount=3 \
  --set "tyk-gateway.gateway.extraVolumes[0].name=tyk-app-storage" \
  --set "tyk-gateway.gateway.extraVolumes[0].persistentVolumeClaim.claimName=tyk-app-claim" \
  --set "tyk-gateway.gateway.extraVolumes[1].name=tyk-policies-storage" \
  --set "tyk-gateway.gateway.extraVolumes[1].persistentVolumeClaim.claimName=tyk-policies-claim" \
  --set "tyk-gateway.gateway.extraVolumes[2].name=tyk-middleware-storage" \
  --set "tyk-gateway.gateway.extraVolumes[2].persistentVolumeClaim.claimName=tyk-middleware-claim" \
  --set "tyk-gateway.gateway.extraVolumeMounts[0].name=tyk-app-storage" \
  --set "tyk-gateway.gateway.extraVolumeMounts[0].mountPath=/mnt/tyk-gateway/apps" \
  --set "tyk-gateway.gateway.extraVolumeMounts[1].name=tyk-policies-storage" \
  --set "tyk-gateway.gateway.extraVolumeMounts[1].mountPath=/mnt/tyk-gateway/policies" \
  --set "tyk-gateway.gateway.extraVolumeMounts[2].name=tyk-middleware-storage" \
  --set "tyk-gateway.gateway.extraVolumeMounts[2].mountPath=/mnt/tyk-gateway/middleware"
```

### 3. Use Tyk Operator to manage your APIs and Policies

When the files in /apps or /policies folder is updated, you have to reload your gateways to pickup the change. Operator can help you to manage reload of the cluster groups automatically after each API or policies resource update.
See [Install Tyk Operator]({{<ref "/tyk-stack/tyk-operator/installing-tyk-operator">}}) to manage your APIs on how to install and use Tyk Operator.

### 4. Verification

You can see multiple gateway pods are created:

```bash
% kubectl get pods -n tyk-oss
NAME                                           READY   STATUS    RESTARTS   AGE
gateway-tyk-oss-tyk-gateway-856bc756b7-k892n   1/1     Running   0          32m
gateway-tyk-oss-tyk-gateway-856bc756b7-w45sr   1/1     Running   0          32m
gateway-tyk-oss-tyk-gateway-856bc756b7-zvkw7   1/1     Running   0          32m
```

Check the PersistentVolumeClaim, you can see that it is used by all the 3 pods:

```bash
% kubectl describe pvc tyk-app-claim -n tyk-oss
Name:          tyk-app-claim
Namespace:     tyk-oss
StorageClass:  managed-csi
Status:        Bound
Volume:        pvc-44122317-c658-40b5-b670-08016456a865
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: disk.csi.azure.com
               volume.kubernetes.io/selected-node: aks-agentpool-32347720-vmss000000
               volume.kubernetes.io/storage-provisioner: disk.csi.azure.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       gateway-tyk-oss-tyk-gateway-856bc756b7-k892n
               gateway-tyk-oss-tyk-gateway-856bc756b7-w45sr
               gateway-tyk-oss-tyk-gateway-856bc756b7-zvkw7
Events:        <none>
```

Create an API and Policy through operator:

```bash
kubectl apply -f https://raw.githubusercontent.com/TykTechnologies/tyk-operator/master/config/samples/httpbin_protected.yaml
kubectl apply -f https://raw.githubusercontent.com/TykTechnologies/tyk-operator/master/config/samples/httpbin_protected_policy.yaml
```

Now, go into each pod and examine /mnt/tyk-gateway/apps and /mnt/tyk-gateway/policies folder, all of them contain the same file:

```bash
$ ls /mnt/tyk-gateway/apps
ZGVmYXVsdC9odHRwYmlu.json  lost+found
$ ls /mnt/tyk-gateway/policies
ZGVmYXVsdC9odHRwYmlu.json  lost+found
```
