# Easy kubernetes and Vault integration

This is a "quick-and-dirty" integration using hardcoded credentials in a container.  
Based on official [Integrate a Kubernetes Cluster with an External Vault](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-external-vault#deploy-application-with-hard-coded-vault-address)

Example is adjusted for Vault  namespace usecase, here: `admin/project-01/ns1`

Enable v2 kv and add test data

```
vault secrets enable -namespace=admin/project-01/ns1 -path=secret kv-v2
vault kv put -namespace=admin/project-01/ns1 secret/devwebapp/config username='giraffe' password='salsa'
```

Create policy `devwebapp` in the namespace `admin/project-01/ns1`  
```
path "secret/data/devwebapp/config" {
  capabilities = ["read"]
}
```

Create a token with the policy, token has a limited TTL:  
`vault token create -namespace=admin/project-01/ns1 -policy=devwebapp`

Use the tokein in a pod, example:  

```
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp
  labels:
    app: devwebapp
spec:
  containers:
    - name: app
      image:  hashicorp/vault:1.13.2
      env:
      - name: VAULT_ADDR
        value: "http://external-vault:8200"
      - name: VAULT_TOKEN
        value: hvs.CAES...QQyBU
      - name: VAULT_NAMESPACE
        value: "admin/project-01/ns1"
```

Check access to the secrets from the container:  
`kubectl exec -it devwebapp -- vault kv get -namespace=admin/project-01/ns1 secret/devwebapp/config`
