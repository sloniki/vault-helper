
# Cubbyhole Response Wrapping

Source  
https://learn.hashicorp.com/tutorials/vault/cubbyhole-response-wrapping  
Wraping token is short living and can be used only once 
 

### Admin
Create a KV secret  
```
$ vault secrets enable -path=secret -description="wraper-test" kv-v2
$ vault kv put secret/dev username="webapp" password="my-long-password"
```

Wrap in a token with ttl=2min  
`$ vault kv get -wrap-ttl=120 secret/dev`

Example output:  
```
Key                              Value
---                              -----
wrapping_token:                  hvs.CAGh2cy5uYzZsZEM....piR25LZXpjczA
wrapping_accessor:               3dyFk8GHlmLNvKEjxcL9TDz2
wrapping_token_ttl:              2m
wrapping_token_creation_time:    2022-04-05 21:09:08.2289 -0700 PDT
wrapping_token_creation_path:    secret/data/dev
```

### User

`$ curl -s --header "X-Vault-Token: hvs.CAGh2cy5uYzZsZEM....piR25LZXpjczA" --request POST $VAULT_ADDR/v1/sys/wrapping/unwrap | jq ".data.data"` 

Output:

```
{
  "password": "my-long-password",
  "username": "webapp"
}
```