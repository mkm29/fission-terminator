# Kubernetes Response Engine: Falco and Fission

## Prerequisites

### Install Falco  

```shell
kubectl create ns falco
helm upgrade --install falco \
  --set falco.jsonOutput=true --set auditLog.enabled=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.fission.function="falco-pod-delete" \
  --namespace falco \
  falcosecurity/falco
```

### Install Fission  

```shell
export FISSION_NAMESPACE="fission"
kubectl create namespace $FISSION_NAMESPACE
kubectl create -k "github.com/fission/fission/crds/v1?ref=v1.16.0"
helm repo add fission-charts https://fission.github.io/fission-charts/
helm repo update
helm install --version v1.16.0 --namespace $FISSION_NAMESPACE fission fission-charts/fission-all
```

### Install fission-cli  

#### Linux/MacOS  

```shell
curl -Lo fission https://github.com/fission/fission/releases/download/v1.16.0/fission-v1.16.0-linux-amd64 \
    && chmod +x fission && sudo mv fission /usr/local/bin/
```

## Go Function  

### Introduction  

Our function receives events from `Falco` via `Falcosidekick`, check if the triggered rule is `Terminal Shell in container`, and if so terminate the pod. The process can be summarized as:  

```
           +----------+                 +---------------+                    +----------+
           |  Falco   +-----------------> Falcosidekick +--------------------> Fission  |
           +----^-----+   sends event   +---------------+      triggers      +-----+----+
                |                                                                  |
detects a shell |                                                                  |
                |                                                                  |
           +----+-------+                                   deletes                |
           | Pwned Pod  <----------------------------------------------------------+
           +------------+
```  

### Permissions  

Let's get started! We need to give our function some permissions, so create the following resources:  

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: falco-pod-delete
  namespace: fission-function
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: falco-pod-delete-cluster-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "delete"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["*"]
- apiGroups: ["fission.io"]
  resources: ["packages"]
  verbs: ["get", "list"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: falco-pod-delete-cluster-role-binding
roleRef:
  kind: ClusterRole
  name: falco-pod-delete-cluster-role
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: falco-pod-delete
    namespace: fission-function
```

### Function Implementation  

Initialize your project:  

> `go mod init falco-pod-delete`  

Add/get project dependencies:  

> `go mod tidy`  

Create archive for package:  

> `zip -r falco-pod-delete.zip .`  

Create function as usual:  

> `fission fn create --name falco-pod-delete --env go --src falco-pod-delete.zip --entrypoint Handler`

Create route:  

> `fission route create --method GET --url /falco-pod-delete --function falco-pod-delete`  

### Test  

Let's create a test Pod: `kubectl run alpine --namespace default --image=alpine --restart='Never' -- sh -c "sleep 600"`  

Once the Pod is running, spawn a shell: `kubectl exec -i --tty alpine --namespace default -- sh -c "uptime"`  

You will get the output of the `uptime` function, however check the current status of the Pod and you will notice it has changed: `kubectl get pods`  

In my case the Pod status has changed from `running` to `completed`.  
