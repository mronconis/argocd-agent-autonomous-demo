# Argo CD Agent Autonomous Mode Demo

This repository aims to show Argo CD Agent configured in Autonomous Mode.

## Architecture Overview
```
    kind-argocd-hub               kind-argocd-agent1...
  (Control Plane Cluster)       (Workload Cluster(s))

Pod CIDR: 10.245.0.0/16        Pod CIDR: 10.246.0.0/16...
SVC CIDR: 10.97.0.0/12         SVC CIDR: 10.98.0.0/12...
┌─────────────────────┐        ┌─────────────────────┐
│ ┌─────────────────┐ │        │ ┌─────────────────┐ │
│ │   Argo CD       │ │        │ │   Argo CD       │ │
│ │ ┌─────────────┐ │ │        │ │ ┌─────────────┐ │ │
│ │ │ API Server  │ │ │◄──────┐│ │ │   App       │ │ │
│ │ │ Repository  │ │ │       ││ │ │ Controller  │ │ │
│ │ │ Redis       │ │ │       ││ │ │ Repository  │ │ │
│ │ │ Dex (SSO)   │ │ │       ││ │ │ Redis       │ │ │
│ │ └─────────────┘ │ │       ││ │ └─────────────┘ │
│ └─────────────────┘ │       ││ └─────────────────┘ │
│ ┌─────────────────┐ │       ││ ┌─────────────────┐ │
│ │   Principal     │ │◄──────┘│ │     Agent       │ │
│ │ ┌─────────────┐ │ │        │ │                 │ │
│ │ │ gRPC Server │ │ │        │ │                 │ │
│ │ │ Resource    │ │ │        │ │                 │ │
│ │ │ Proxy       │ │ │        │ │                 │ │
│ │ └─────────────┘ │ │        │ └─────────────────┘ │
│ └─────────────────┘ │        └─────────────────────┘
└─────────────────────┘
```

## Pre-requisite

### Setup Argo CD Agent CLI
```bash
wget -o argocd-agentctl https://github.com/argoproj-labs/argocd-agent/releases/download/v0.5.3/argocd-agentctl_darwin-arm64
sudo mv argocd-agentctl /usr/local/bin
sudo chmod a+x /usr/local/bin/argocd-agentctl
```

## Set environment values
```bash
export PRINCIPAL_CLUSTER_NAME="argocd-hub"
export AGENT_CLUSTER_NAME="argocd-agent1"
export AGENT_APP_NAME="agent-a"
export NAMESPACE_NAME="argocd"
export AGENT_MODE="autonomous"
export CLUSTER_USER_ID=1
export AGENT_USER_ID=2

export PRINCIPAL_POD_CIDR="10.$((244 + $CLUSTER_USER_ID)).0.0/16"
export PRINCIPAL_SVC_CIDR="10.$((96 + $CLUSTER_USER_ID)).0.0/16"
export AGENT_POD_CIDR="10.$((244 + $AGENT_USER_ID)).0.0/16"
export AGENT_SVC_CIDR="10.$((96 + $AGENT_USER_ID)).0.0/16"

export RELEASE_BRANCH="main"
export KIND_EXPERIMENTAL_PROVIDER=podman
```

## Control Plane Setup

### Create cluster
```bash
cat <<EOF | kind create cluster --name $PRINCIPAL_CLUSTER_NAME --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: $PRINCIPAL_CLUSTER_NAME
networking:
  podSubnet: "$PRINCIPAL_POD_CIDR"
  serviceSubnet: "$PRINCIPAL_SVC_CIDR"
nodes:
  - role: control-plane
  - role: worker
EOF
```

### Create namespace
```bash
kubectl create namespace $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME
```

### Install Argo CD for Control Plane
```bash
kubectl apply --server-side -n $NAMESPACE_NAME \
  -k "https://github.com/argoproj-labs/argocd-agent/install/kubernetes/argo-cd/principal?ref=$RELEASE_BRANCH" \
  --context kind-$PRINCIPAL_CLUSTER_NAME
```

### Configure Apps-in-Any-Namespace

```bash
kubectl patch configmap argocd-cmd-params-cm \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME \
  --patch '{"data":{"application.namespaces":"*"}}'

kubectl rollout restart deployment argocd-server \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME 

kubectl get configmap argocd-cmd-params-cm \
  -n "$NAMESPACE_NAME" \
  --context kind-$PRINCIPAL_CLUSTER_NAME \
  -o yaml | grep application.namespaces
```

### Initialize Certificate Authority
```bash
argocd-agentctl pki init \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --principal-namespace $NAMESPACE_NAME
```

## Install Principal

### Deploy Principal components

```bash
kubectl apply -n $NAMESPACE_NAME \
  -k "https://github.com/argoproj-labs/argocd-agent/install/kubernetes/principal?ref=$RELEASE_BRANCH" \
  --context kind-$PRINCIPAL_CLUSTER_NAME
```

### Check Principal configuration
```bash
# Check the principal authentication configuration
kubectl get configmap argocd-agent-params \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME \
  -o jsonpath='{.data.principal\.auth}'

# Should output: mtls:CN=([^,]+)
```

### Update Principal configuration
```bash
kubectl patch configmap argocd-agent-params \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME \
  --patch "{\"data\":{
    \"principal.allowed-namespaces\":\"$AGENT_APP_NAME\"
}}"

kubectl get configmap argocd-agent-params \
  -n "$NAMESPACE_NAME" \
  --context "kind-$PRINCIPAL_CLUSTER_NAME" \
  -o yaml | grep principal.allowed-namespaces

# Expected output: 
# principal.allowed-namespaces: $AGENT_APP_NAME
# ($AGENT_APP_NAME should be replaced with the string value you set for your agent application name.)
```

### Verify Principal service exposure
```bash
# NodePort
kubectl patch svc argocd-agent-principal \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME \
  --patch '{"spec":{"type":"NodePort"}}'

kubectl get svc argocd-agent-principal \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME

# Expected output:
# NAME                     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
# argocd-agent-principal   NodePort   10.106.231.64   <none>        443:32560/TCP   26s
```

## PKI Setup

Issue gRPC server certificate (address for Agent connection):
```bash
# Check NodePort
PRINCIPAL_EXTERNAL_IP=$(podman inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' argocd-hub-control-plane)
echo "<principal-external-ip>: $PRINCIPAL_EXTERNAL_IP"

PRINCIPAL_NODE_PORT=$(kubectl get svc argocd-agent-principal -n $NAMESPACE_NAME --context kind-$PRINCIPAL_CLUSTER_NAME -o jsonpath='{.spec.ports[0].nodePort}'
)
echo "<principal-node-port>: $PRINCIPAL_NODE_PORT"

# Check DNS Name
PRINCIPAL_DNS_NAME=$(kubectl get svc argocd-agent-principal -n $NAMESPACE_NAME --context kind-$PRINCIPAL_CLUSTER_NAME -o jsonpath='{.metadata.name}.{.metadata.namespace}.svc.cluster.local')
echo "<principal-dns-name>: $PRINCIPAL_DNS_NAME"

# Create pki issue
argocd-agentctl pki issue principal \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --principal-namespace $NAMESPACE_NAME \
  --ip 127.0.0.1,$PRINCIPAL_EXTERNAL_IP \
  --dns localhost,$PRINCIPAL_DNS_NAME \
  --upsert
```

Issue resource proxy certificate (address for Argo CD connection)
```bash
# Check NodePort
RESOURCE_PROXY_INTERNAL_IP=$(kubectl get svc argocd-agent-resource-proxy \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME \
  -o jsonpath='{.spec.clusterIP}')
echo "<resource-proxy-ip>: $RESOURCE_PROXY_INTERNAL_IP"

RESOURCE_PROXY_DNS_NAME=$(kubectl get svc argocd-agent-resource-proxy \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME \
  -o jsonpath='{.metadata.name}.{.metadata.namespace}.svc.cluster.local')
echo "<resource-proxy-dns>: $RESOURCE_PROXY_DNS_NAME"

# Create pki issue
argocd-agentctl pki issue resource-proxy \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --principal-namespace $NAMESPACE_NAME \
  --ip 127.0.0.1,$RESOURCE_PROXY_INTERNAL_IP \
  --dns localhost,$RESOURCE_PROXY_DNS_NAME \
  --upsert
```

### Generate JWT signing key
```bash
argocd-agentctl jwt create-key \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --principal-namespace $NAMESPACE_NAME \
  --upsert

kubectl rollout restart deployment argocd-agent-principal \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME
```

### Connection issue due to NetworkPolicy
As workaround, we remove all network policies:
```bash
kubectl delete netpol --all -n argocd --context kind-$PRINCIPAL_CLUSTER_NAME
```

### Setup UI access

Port-Forward ArgoCD:
```bash
kubectl port-forward svc/argocd-server 8443:443 \
  -n $NAMESPACE_NAME \
  --context kind-$PRINCIPAL_CLUSTER_NAME
```

### Verify Principal installation
```bash
kubectl get pods -n $NAMESPACE_NAME --context kind-$PRINCIPAL_CLUSTER_NAME | grep principal

kubectl logs -n $NAMESPACE_NAME deployment/argocd-agent-principal --context kind-$PRINCIPAL_CLUSTER_NAME

# Expected logs:
# argocd-agent-principal-785cd96ddc-sm44r            1/1     Running   0          7s
# {"level":"info","msg":"Setting loglevel to info","time":"2025-09-13T07:25:38Z"}
# ...output omit...
```

## Workload Cluster Setup

### Create cluster

```bash
cat <<EOF | kind create cluster --name $AGENT_CLUSTER_NAME --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: $AGENT_CLUSTER_NAME
networking:
  podSubnet: "$AGENT_POD_CIDR"
  serviceSubnet: "$AGENT_SVC_CIDR"
nodes:
  - role: control-plane
  - role: worker
EOF
```

### Create namespace
```bash
kubectl create namespace $NAMESPACE_NAME --context kind-$AGENT_CLUSTER_NAME
```

### Install Argo CD for Workload Cluster
```bash
kubectl apply --server-side -n $NAMESPACE_NAME \
  -k "https://github.com/argoproj-labs/argocd-agent/install/kubernetes/argo-cd/agent-$AGENT_MODE?ref=$RELEASE_BRANCH" \
  --context kind-$AGENT_CLUSTER_NAME
```

### Create Agent configuration
```bash
# Check NodePort
PRINCIPAL_EXTERNAL_IP=$(podman inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' argocd-hub-control-plane)
echo "<principal-external-ip>: $PRINCIPAL_EXTERNAL_IP"

argocd-agentctl agent create $AGENT_APP_NAME \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --principal-namespace $NAMESPACE_NAME \
  --resource-proxy-server ${PRINCIPAL_EXTERNAL_IP}:9090
```

### Issue Agent client certificate
```bash
# Issue Agent client certificate
argocd-agentctl pki issue agent $AGENT_APP_NAME \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --agent-context kind-$AGENT_CLUSTER_NAME \
  --agent-namespace $NAMESPACE_NAME \
  --upsert
```

### Propagate Certificate Authority to Agent
```bash
argocd-agentctl pki propagate \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --principal-namespace $NAMESPACE_NAME \
  --agent-context kind-$AGENT_CLUSTER_NAME \
  --agent-namespace $NAMESPACE_NAME
```

### Verify certificate installation
```bash
kubectl get secret argocd-agent-client-tls -n $NAMESPACE_NAME --context kind-$AGENT_CLUSTER_NAME

# Expected result
# NAME                      TYPE                DATA   AGE
# argocd-agent-client-tls   kubernetes.io/tls   2      7s

kubectl get secret argocd-agent-ca -n $NAMESPACE_NAME --context kind-$AGENT_CLUSTER_NAME

# Expected result
# NAME              TYPE     DATA   AGE
# argocd-agent-ca   Opaque   1      15s
```

### Create Agent namespace on Principal
```bash
kubectl create namespace $AGENT_APP_NAME --context kind-$PRINCIPAL_CLUSTER_NAME
```

### Deploy Agent
```bash
kubectl apply -n $NAMESPACE_NAME \
  -k "https://github.com/argoproj-labs/argocd-agent/install/kubernetes/agent?ref=$RELEASE_BRANCH" \
  --context kind-$AGENT_CLUSTER_NAME
```

### Configure Agent connection
```bash
# Check NodePort
PRINCIPAL_EXTERNAL_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' argocd-hub-control-plane)
echo "<principal-external-ip>: $PRINCIPAL_EXTERNAL_IP"

PRINCIPAL_NODE_PORT=$(kubectl get svc argocd-agent-principal -n $NAMESPACE_NAME --context kind-$PRINCIPAL_CLUSTER_NAME -o jsonpath='{.spec.ports[0].nodePort}'
)
echo "<principal-node-port>: $PRINCIPAL_NODE_PORT"

kubectl patch configmap argocd-agent-params \
  -n $NAMESPACE_NAME \
  --context kind-$AGENT_CLUSTER_NAME \
  --patch "{\"data\":{
    \"agent.server.address\":\"$PRINCIPAL_EXTERNAL_IP\",
    \"agent.server.port\":\"$PRINCIPAL_NODE_PORT\",
    \"agent.mode\":\"$AGENT_MODE\",
    \"agent.creds\":\"mtls:any\"
  }}"

kubectl rollout restart deployment argocd-agent-agent \
  -n $NAMESPACE_NAME \
  --context kind-$AGENT_CLUSTER_NAME
```

## Verification

### Verify Agent connection
```bash
kubectl logs -n $NAMESPACE_NAME deployment/argocd-agent-agent --context kind-$AGENT_CLUSTER_NAME

# Expected output on success:
# time="2025-10-09T09:11:22Z" level=info msg="Authentication successful" module=Connector
# time="2025-10-09T09:11:22Z" level=info msg="Connected to argocd-agent-0.0.1-alpha" module=Connector
# time="2025-10-09T09:11:23Z" level=info msg="Starting to send events to event stream" direction=send module=StreamEvent
```

### Verify Agent recognition on Principal
```bash
kubectl logs -n $NAMESPACE_NAME deployment/argocd-agent-principal --context kind-$PRINCIPAL_CLUSTER_NAME

# Expected output on success:
# time="2025-10-09T09:11:23Z" level=info msg="An agent connected to the subscription stream" client=$AGENT_APP_NAME method=Subscribe
# time="2025-10-09T09:11:23Z" level=info msg="Updated connection status to 'Successful' in Cluster: '$AGENT_APP_NAME'" component=ClusterManager
```

### List connected Agents
```bash
argocd-agentctl agent list \
  --principal-context kind-$PRINCIPAL_CLUSTER_NAME \
  --principal-namespace $NAMESPACE_NAME
```

### Verify UI access
Get inialt admin credentials
```bash
ARGOCD_ADMIN_PWD=$(kubectl get secret argocd-initial-admin-secret -o "jsonpath={.data.password}" -n $NAMESPACE_NAME --context kind-$PRINCIPAL_CLUSTER_NAME | base64 -d && echo)
echo $ARGOCD_ADMIN_PWD
```

Open your browser at https://localhost:8443, use `admin` as your username and as your password the one read by the secret.

### Test Application Synchronization

Create default project:
```bash
cat <<EOF | kubectl apply -f - --context kind-$AGENT_CLUSTER_NAME
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  destinations:
  - name: "in-cluster"
    namespace: "*"
    server: "https://kubernetes.default.svc"
  sourceNamespaces:
  - argocd
  sourceRepos:
  - "*"
EOF
```

Create test app:
```bash
cat <<EOF | kubectl apply -f - --context kind-$AGENT_CLUSTER_NAME
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: my-project
  source:
    repoURL: https://github.com/mronconis/argocd-agent-autonomous-demo
    targetRevision: HEAD
    path: app-of-apps/guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

Verify:
```bash
kubectl -nargocd get applications.argoproj.io --context kind-$AGENT_CLUSTER_NAME -o wide
# NAME             SYNC STATUS   HEALTH STATUS   REVISION                                   PROJECT
# my-app           Synced        Healthy         f26954d8f9779811c638c4ebf7844bb4171a7f15   my-project
# my-app-dev       Synced        Healthy         f26954d8f9779811c638c4ebf7844bb4171a7f15   my-project
# my-app-staging   Synced        Healthy         f26954d8f9779811c638c4ebf7844bb4171a7f15   my-project


kubectl -nargocd get applicationsets.argoproj.io --context kind-$AGENT_CLUSTER_NAME -o wide
# NAME               AGE
# guestbook-appset   4m28s

kubectl get po -A --selector app=guestbook-ui \
  --context kind-$AGENT_CLUSTER_NAME
# NAMESPACE           NAME                            READY   STATUS    RESTARTS   AGE
# guestbook-dev       guestbook-ui-6595f948db-dj99k   1/1     Running   0          5m49s
# guestbook-staging   guestbook-ui-6595f948db-grv6z   1/1     Running   0          5m49s
# guestbook           guestbook-ui-6595f948db-q8fck   1/1     Running   0          4m32s
```

## Test Control Plane API

Get OAuth bearer token:
```bash
ARGOCD_TOKEN=$(curl -q -k https://localhost:8443/api/v1/session -d "{'username': 'admin', 'password': '$ARGOCD_ADMIN_PWD'}" -H "Content-Type: application/json"|jq .token |sed -e 's/"//g')
```

Invoke API get applications list:
```bash
curl -k -X GET -H "Content-Type: application/json" -H "Authorization: Bearer $ARGOCD_TOKEN" https://localhost:8443/api/v1/applications \
  | jq
```

## Issues

1. Error on Agent: server.secretkey is missing

**Diagnose**
```bash
kubectl -nargocd logs -f argocd-application-controller-0  --context kind-$AGENT_CLUSTER_NAME

# {"level":"warning","msg":"Failed to save clusters info: server.secretkey is missing","time":"2026-02-02T15:48:32Z"
```

**Workaround**
```bash
kubectl patch secret argocd-secret -n argocd -p "{\"data\": {\"server.secretkey\": \"$(openssl rand -base64 32 | tr -d '\n' | base64 | tr -d '\n')\"}}" \
  --context kind-$AGENT_CLUSTER_NAME
kubectl rollout restart statefulset argocd-application-controller -n argocd --context kind-$AGENT_CLUSTER_NAME
```

## Resources

- https://argocd-agent.readthedocs.io/latest/getting-started/kubernetes/kind/#install-argo-cd-for-control-plane
- https://argocd-agent.readthedocs.io/latest/user-guide/applications/
- https://argocd-agent.readthedocs.io/latest/user-guide/appprojects/#autonomous-agent-mode
