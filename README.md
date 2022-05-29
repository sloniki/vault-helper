# HashiCorp Vault CLI Cheatsheet

Tested on Vault 1.10  

[Administration](#Administration)  
Authentication  
[Token](#Token)  
[UserPass](#UserPass)  
[AppRole](#AppRole)  
[Okta](#Okta)  

Policy  
[Policy](#Policy)  

Secrets  
[KV](#KV)  
[Database](#Database)

## Administration 
Init storage with 3/2 keys  
`$ vault operator init -key-shares=3 -key-threshold=2`

List all authentication methods  
`$ vault auth list -detailed`

List all secrets  
`$ vault secrets list`

Install Ansible collection  
`$ pip install ansible-modules-hashivault`

## Token
List token accessors  
`$ vault list auth/token/accessors`

Create a token  
`$ vault token create -display-name=name-to-display -id=super-secret -policy=new-policy`

Lookup token  
`$ vault token lookup <TOKEN-ID>`


## UserPass  
Enable userpass engine  
`$ vault auth enable userpass`

Add a user  
`$ vault write auth/userpass/users/superman password=changeme`

List users  
`$ vault list auth/userpass/users`

## AppRole

Enable approle engine  
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

Delete approle  
`$ vault delete auth/new-approle/role/new-app`

## Okta
Octa account can be created for free on [Okta](http://developer.okta.com)  

Enable Okta authentication  
`$ vault auth enable okta`

Configure Okta account  
`$ vault write auth/okta/config base_url="okta.com" org_name="dev-XXXXXXXX" api_token="00y0RF......av5u4"`

Add Okta group  
`$ vault write -force auth/okta/groups/new-group`

Login as an okta user  
`$ vault login -method=okta username=okta-user@emai.com`

## Policy

List policies  
`$ vault policy list`

Create policy from a file  
`$ vault policy write new-policy new-policy.hcl`

Read policy  
`$ vault policy read new-policy`

## KV  

Enable KV secrets engine  
`$ vault secrets enable -path=new-secret -description="New Key-Value engine" kv`

Write a secret  
`$ vault kv put new-secret/first key-01=value-01 key-02=another-value`

List secrets  
`$ vault kv list new-secret`

Read a secret  
`$ vault kv get new-secret/first`

Delete a secret  
`$ vault kv delete new-secret/first`  

## Database  

Enable Database secret engine  
`$ vault secrets enable -path=new-db database`  

Create database configuration  
```
$ vault write new-db/config/mysql-database \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="mysql-role" \
  username="root" \
  password="secret"
  ```  
Create the role, which creates a temporary DB user  

```
$ vault write new-db/roles/mysql-role \
  db_name=mysql-database \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```  

Create a user  
`$ vault read new-db/creds/mysql-role`  

Output example  
```
Key                Value
---                -----
lease_id           new-db/creds/mysql-role/zYD2zmOrvUdOSEs3SHPQvEjh
lease_duration     1h
lease_renewable    true
password           H-qBRmuWyLEhc0Pr8I3F
username           v-root-mysql-role-mJXckL8jit5FRf
```  
