# Using External-Secrets for K8s integration

## Configure K8s cluster
Install chart
https://external-secrets.io/v0.8.1/introduction/getting-started/

```
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace \
```

Create service accounts and secrets  
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: external-secrets
---
apiVersion: v1
kind: Secret
metadata:
  name: external-secrets-token
  namespace: external-secrets
  annotations:
    kubernetes.io/service-account.name: external-secrets
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth
  namespace: external-secrets
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: external-secrets
```


## Configure Vault

In this example using vault namespace `admin/project-01/ns1`

```
TOKEN_REVIEW_JWT=$(kubectl get secret -n external-secrets vault-auth --output='go-template={{ .data.token }}' | base64 --decode)
KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
```

Create kubernetes config  
```
vault auth enable -namespace=admin/project-01/ns1 kubernetes

vault write -namespace=admin/project-01/ns1 auth/kubernetes/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT" \
        disable_issuer_verification=true
```

Create role  
``` 
vault write -namespace=admin/project-01/ns1 auth/kubernetes/role/eso-role \
bound_service_account_names=external-secrets \
bound_service_account_namespaces=external-secrets \
policies=eso ttl=24h
```
To test  login  
```
TEST_TOKEN="$(kubectl get secret -n external-secrets external-secrets-token -o jsonpath='{.data.token} {"\n"}' | base64 -d)"

vault write -namespace=admin/project-01/ns1 auth/kubernetes/login role=eso-role jwt=$TEST_TOKEN iss=https://kubernetes.default.svc.cluster.local
```

Create policy `eso` in the namespace  
```
path "kv/*" {
  capabilities = ["read"]
}
```

Create target secrets  
```
vault secrets enable -namespace=admin/project-01/ns1 -version=2 kv
vault kv put -namespace=admin/project-01/ns1 kv/secret password=extremely-secure-password
```

## Configure CRDs
Create SecretStore  

```
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault-address:8200"
      path: "kv"
      namespace: "admin/project-01/ns1"
      # Version is the Vault KV secret engine version.
      # This can be either "v1" or "v2", defaults to "v2"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-role"
```

Create ExternalSecret  
```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: vault-secret-example
  data:
  - secretKey: password
    remoteRef:
      key: secret
      property: password
```

Read the secret to test  
```
kubectl get secrets vault-secret-example -o jsonpath='{.data.password}' | base64 -d
```
