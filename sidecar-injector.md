
# Kubernetes Cluster with External Vault and Injector

Based on  
https://learn.hashicorp.com/tutorials/vault/kubernetes-external-vault?in=vault/kubernetes  
https://www.vaultproject.io/docs/platform/k8s/injector  
modified for kv v2 and proper annotations.  
Tested on k3s v1.23.0  
Access from Vault to K8s API must be provided. 

Deploy `vault-agent-injector` pointing to the external vault  
```
$ helm repo add hashicorp https://helm.releases.hashicorp.com
$ helm repo update
$ helm install vault hashicorp/vault \
    --set "injector.externalVaultAddr=http://external-vault:8200"
```

Check vault service account  
`$ kubectl describe serviceaccount vault` 

Get vault-token secret name  
`$ VAULT_HELM_SECRET_NAME=$(kubectl get secrets --output=json | jq -r '.items[].metadata | select(.name|startswith("vault-token-")).name')`

`$ kubectl describe secret $VAULT_HELM_SECRET_NAME`

Enable k8s authentication on Vault  
`$ vault auth enable kubernetes`

Get parameters for configuration  
````
$ TOKEN_REVIEW_JWT=$(kubectl get secret $VAULT_HELM_SECRET_NAME --output='go-template={{ .data.token }}' | base64 --decode)

$ KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)

$ KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
````

Create k8s configuration  
```
$ vault write auth/kubernetes/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT" \
        issuer="https://kubernetes.default.svc.cluster.local"
````

Create policy  
```
$ vault policy write devwebapp - <<EOF
path "secret/data/devwebapp/config" {
  capabilities = ["read"]
}
EOF
````

Add sa  
`$ kubectl create sa internal-app`

Create role  
```
$ vault write auth/kubernetes/role/devweb-app \
        bound_service_account_names=internal-app \
        bound_service_account_namespaces=default \
        policies=devwebapp \
        ttl=24h
````

Enable v2 kv and add test data  
```
$ vault secrets enable -path=secret kv-v2
$ vault kv put secret/devwebapp/config username='giraffe' password='salsa'
```

### To inject secrets into pod use annotations  


For init container on start  
```
annotations:
  vault.hashicorp.com/agent-inject: 'true'
  vault.hashicorp.com/role: 'devweb-app'
  vault.hashicorp.com/agent-pre-populate-only: 'true'
  # vault.hashicorp.com/tls-skip-verify: 'true'
  vault.hashicorp.com/agent-inject-secret-credentials.txt: 'secret/data/devwebapp/config'
````

For permanent sidecar container with live secrets  
```
annotations:
  vault.hashicorp.com/agent-inject: 'true'
  vault.hashicorp.com/role: 'devweb-app'
  vault.hashicorp.com/agent-pre-populate: 'false'
  # vault.hashicorp.com/tls-skip-verify: 'true'
  vault.hashicorp.com/agent-inject-secret-credentials.txt: 'secret/data/devwebapp/config'
```

Secrets will be placed in the container path  
`/vault/secrets/credentials.txt`

Add sa  
```
...
    spec:
      serviceAccountName: internal-app
      ...
```  

Deployment example

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-netshoot
  labels:
    app: nginx-netshoot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-netshoot
  template:
    metadata:
      labels:
        app: nginx-netshoot
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'devweb-app'
        vault.hashicorp.com/agent-pre-populate: 'false'
        vault.hashicorp.com/tls-skip-verify: 'true'
        vault.hashicorp.com/agent-inject-secret-credentials.txt: 'secret/data/devwebapp/config'
    spec:
      serviceAccountName: internal-app
      containers:
      - name: nginx
        image: nginx:1.20.2-alpine
        ports:
          - containerPort: 80
      - name: netshoot
        image: nicolaka/netshoot:v0.2
        command:
          - /bin/sh
          - '-ec'
          - 'while :; do echo "debug container date: `date`"; sleep 10 ; done'
```  


