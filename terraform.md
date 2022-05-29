# Using Vault approle with Terraform

Enable and configure approle authentication  
`$ vault auth enable -description="terraform integration" -path=tf-approle approle`  
`$ vault write auth/tf-approle/role/tf-role-01 secret_id_ttl=10m token_ttl=20m token_policies=tf-pol-01`

Create policy `tf-pol-01`  
```
# kv v2 read secret
path "tf-secrets/data/private" {
  capabilities = ["read"]
}

# kv v2 add secret
path "tf-secrets/data/exchange" {
  capabilities = ["create", "update"]
}

# create token
path "auth/token/create" {
  capabilities = ["create", "update"]
}
```

Enable KV v2 and add a secret

`$ vault secrets enable -path=tf-secrets -description="secrets for terraform" kv-v2`  
`$ vault kv put tf-secrets/private key-01="super secret 01"` 

Get role-id and secret-if for the last step (secret-id valid 10m)  
`$ vault read auth/tf-approle/role/tf-role-01/role-id`  
```
Key        Value
---        -----
role_id    ae3e711c-52cf-85d1-058c-0c8f48fb5be5
```

`$ vault write -f auth/tf-approle/role/tf-role-01/secret-id` 
```
Key                   Value
---                   -----
secret_id             d3cb1df9-89a1-0159-1970-46121db84c47
secret_id_accessor    b94a471e-3ca4-4b86-5348-058c2b3bc640
secret_id_ttl         10m
```

Apply the terraform entering the provided IDs

```
variable login_approle_role_id {}
variable login_approle_secret_id {}

provider "vault" {
  auth_login {
    path = "auth/tf-approle/login"

    parameters = {
      role_id   = var.login_approle_role_id
      secret_id = var.login_approle_secret_id
    }
  }
}

# write a sectet
resource "vault_generic_secret" "example" {
  path = "tf-secrets/exchange"
  data_json = jsonencode(
    {
      "key-02" = "written with terraform again"
    }
  )
}

#read vault sectet to a local file
data "vault_generic_secret" "key" {
  path = "tf-secrets/private"
}

resource "local_file" "foo" {
    content  = data.vault_generic_secret.key.data["key-01"]
    filename = "output.txt"
}
```







