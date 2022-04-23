## Basic token usage workflow

Example for HCP Vault version 1.10.0+ent  

Create periodic orphan token in admin/token_case namespace with a policy attached:

```
$ vault token create -display-name=base_token -policy=kv01-all -namespace=admin/token_case -orphan -period=10h

Key                  Value
---                  -----
token                hvs.CAESIFSvXi7ZP...Yk8wSVYQ0uMC
token_accessor       80cQHE6Gb8SgV7Xrrhkwz012.bO0IV
token_duration       10h
token_renewable      true
token_policies       ["default" "kv01-all"]
identity_policies    []
policies             ["default" "kv01-all"]
```

Set the created token as $VAULT_TOKEN  

Check status:
```
$ vault token lookup
Key                 Value
---                 -----
accessor            80cQHE6Gb8SgV7Xrrhkwz012.bO0IV
creation_time       1650720098
creation_ttl        10h
display_name        token-base-token
entity_id           n/a
expire_time         2022-04-23T23:21:38.970948397Z
explicit_max_ttl    0s
id                  hvs.CAESIFSvXi7ZP...Yk8wSVYQ0uMC
issue_time          2022-04-23T13:21:38.970955167Z
meta                <nil>
namespace_path      admin/token_case/
num_uses            0
orphan              true
path                auth/token/create
period              10h
policies            [default kv01-all]
renewable           true
ttl                 9h54m5s
type                service
```

Token renew itself:  
`vault token renew`  

Create policy (example for GUI)  
```
kv01-all  

path "kv01/*"
{
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

Create payload file:  
```
$ cat payload.json
{
  "data": {
    "user": "foo",
    "pwd": "bar"
  }
}
```

Enable `kv01` engine in the namespace and post the KV secret with API:  

```
$ curl -k --header "X-Vault-Token: $VAULT_TOKEN" --request POST --data @payload.json $VAULT_ADDR/v1/admin/token_case/kv01/data/curling01 | jq
```
output  
```
{
  "request_id": "c3f0ff50-db07-d9b5-04d2-6506ad7ce3cd",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "created_time": "2022-04-23T14:05:57.452441541Z",
    "custom_metadata": null,
    "deletion_time": "",
    "destroyed": false,
    "version": 1
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```
Read the secret:  
```
$ curl -k --header "X-Vault-Token: $VAULT_TOKEN" $VAULT_ADDR/v1/admin/token_case/kv01/data/curling01 | jq
```
output
```
{
  "request_id": "5b079228-6b90-f0ed-2441-acecc1e5df01",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "pwd": "bar",
      "user": "foo"
    },
    "metadata": {
      "created_time": "2022-04-23T14:05:57.452441541Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```

Read the secret metadata:  
```
$ curl -k --header "X-Vault-Token: $VAULT_TOKEN" $VAULT_ADDR/v1/admin/token_case/kv01/metadata/curling01 | jq
```

Delete the secret:  
```
curl -k --header "X-Vault-Token: $VAULT_TOKEN" --request DELETE $VAULT_ADDR/v1/admin/token_case/kv01/metadata/curling01
```
_< no output >_

Token self lookup:  
```
$ curl -k --header "X-Vault-Token: $VAULT_TOKEN" $VAULT_ADDR/v1/admin/auth/token/lookup-self | jq
```
output
```
{
  "request_id": "66809b34-100a-ff23-6d1b-dbd401d2a748",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "accessor": "80cQHE6Gb8SgV7Xrrhkwz012.bO0IV",
    "creation_time": 1650720098,
    "creation_ttl": 36000,
    "display_name": "token-base-token",
    "entity_id": "",
    "expire_time": "2022-04-23T23:49:16.754949547Z",
    "explicit_max_ttl": 0,
    "id": "hvs.CAESIFSvXi7ZP...Yk8wSVYQ0uMC",
    "issue_time": "2022-04-23T13:21:38.970955167Z",
    "last_renewal": "2022-04-23T13:49:16.754949737Z",
    "last_renewal_time": 1650721756,
    "meta": null,
    "namespace_path": "admin/token_case/",
    "num_uses": 0,
    "orphan": true,
    "path": "auth/token/create",
    "period": 36000,
    "policies": [
      "default",
      "kv01-all"
    ],
    "renewable": true,
    "ttl": 33889,
    "type": "service"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```
Renew self  
```
$ curl -k --header "X-Vault-Token: $VAULT_TOKEN" --request POST $VAULT_ADDR/v1/admin/auth/token/renew-self | jq
```
output
```
{
  "request_id": "99689c5a-baa9-4d59-f0ad-6b2c8ef151b3",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "wrap_info": null,
  "warnings": null,
  "auth": {
    "client_token": "hvs.CAESIFSvXi7ZP...Yk8wSVYQ0uMC",
    "accessor": "80cQHE6Gb8SgV7Xrrhkwz012.bO0IV",
    "policies": [
      "default",
      "kv01-all"
    ],
    "token_policies": [
      "default",
      "kv01-all"
    ],
    "metadata": null,
    "lease_duration": 36000,
    "renewable": true,
    "entity_id": "",
    "token_type": "service",
    "orphan": true,
    "mfa_requirement": null,
    "num_uses": 0
  }
}
```
