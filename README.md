# HashiCorp Vault CLI Cheatsheet

Authentication  
[AppRole](#AppRole)  
[Okta](#Okta)  
[UserPass](#UserPass)  

Policies  
...

Init storage with 3/2 keys  
`$ vault operator init -key-shares=3 -key-threshold=2`

List policies  
`$ vault policy list`

List all authentication methods  
`$ vault auth list`

## AppRole

Enable approle authentication method  
`$ vault auth enable -description="testing new-approle" -path=new-approle approle`

Create approle  
`$ vault write auth/new-approle/role/new-app token_policies=new-policy token_ttl=240m`

List approles  
`$ vault list auth/new-approle/role`

Read all approle attributes  
`$ vault read auth/new-approle/role/new-app`

Get role-id  
`$ vault read auth/new-approle/role/new-app/role-id`

Write approle secret  
`$ vault write -f auth/new-approle/role/new-app/secret-id`

Login with approle  
`$ vault write auth/new-approle/login/ role_id=84a1ab27-...-643cab249769 secret_id=33d3fdf5-...-bd5a8043b7b9`

## Okta
## Policy

Create policy from file  
`$ vault policy write new-policy new-policy.hcl`

