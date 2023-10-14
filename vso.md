## Using Vault Secrets Operator on Kubernetes

Inspired by https://developer.hashicorp.com/vault/tutorials/kubernetes/vault-secrets-operator  
Used components:  
* k3s v1.26.4
* helm v3.12.3
* vault v1.15.0
* vault-secrets-operator v0.3.2

### Configure target kv secrets and policy

Configure the kv engine with target secrets, wich need to be accessed from an app on K8s.

`vault secrets enable -path=kvv2 kv-v2`  
`vault kv put kvv2/webapp username="web-user" password=":pa55word:"`

```
vault policy write webapp-ro - <<EOF
path "kvv2/data/webapp" {
   capabilities = ["read"]
}
path "kvv2/metadata/webapp" {
   capabilities = ["read"]
}
EOF
```


### Configure k8s authentication on Vault

Create a service account with a secret and binding  

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth
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
    namespace: default
```



`TOKEN_REVIEW_JWT=$(kubectl get secret vault-auth --output='go-template={{ .data.token }}' | base64 --decode)`  
`KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)`  
`KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')`  

Enable k8s authentication  
`vault auth enable -path=vso kubernetes`

Write the config    
```
vault write auth/vso/config \
      token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
      kubernetes_host="$KUBE_HOST" \
      kubernetes_ca_cert="$KUBE_CA_CERT" \
      disable_issuer_verification=true
```
Write the role  
```
vault write auth/vso/role/vso-role \
      bound_service_account_names=default \
      bound_service_account_namespaces=default \
      policies=webapp-ro \
      ttl=24h
```

Verify  

`vault list auth/vso/role`  
`vault read auth/vso/role/vso-role`

### Install and configure Vault Secrets Operator

Install in the default namespace  

`helm install vault-secrets-operator hashicorp/vault-secrets-operator`

Configure connection, authentication and secret  
```
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  namespace: default
  name: vault-connection
spec:
  # address to the Vault server.
  address: http://128.0.0.1:8200
  skipTLSVerify: true
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: vso
  kubernetes:
    role: vso-role
    serviceAccount: default
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-static-secret
spec:
  vaultAuthRef: vault-auth
  mount: kvv2
  type: kv-v2
  path:  webapp
  refreshAfter: 10s
  destination:
    create: true
    name: vso-handled

```

### Validation

Read the secret  
`kubectl get secret vso-handled -o jsonpath='{.data._raw}' | base64 --decode`  

Edit the kv secret and read again after 10 sec, it should be refreshed
