## Using GCP Secrets Engine for GCP SA keys management

Source: https://developer.hashicorp.com/vault/api-docs/secret/gcp 

Prerequisites  
1. "Root" SA for Vault configuration with permissiosn  
    "roles/iam.securityAdmin",
    "roles/iam.securityReviewer",
    "roles/iam.serviceAccountKeyAdmin",
    "roles/iam.serviceAccountTokenCreator",
    "roles/iam.serviceAccountAdmin",

2. JSON with key from (1)
3. Test SA, where we are managing keys - `vt-sa-01`

Enable secret engine and configure:  
```
vault secrets enable gcp
```
```
vault write gcp/config credentials=@root-sa.json
```
```
vault write gcp/static-account/vt-sa-01-key \
service_account_email="vt-sa-01@my-project.iam.gserviceaccount.com" \
secret_type="service_account_key"  \
token_scopes="https://www.googleapis.com/auth/cloud-platform" \
bindings=-<<EOF
resource "//cloudresourcemanager.googleapis.com/projects/my-project" {
roles = ["roles/viewer"]
}
EOF
```
Create the key:  
```
➜  ~ vault read gcp/static-account/vt-sa-01-key/key
Key                 Value
---                 -----
lease_id            gcp/static-account/vt-sa-01-key/key/wBL...PUGGFf
lease_duration      768h
lease_renewable     true
key_algorithm       KEY_ALG_RSA_2048
key_type            TYPE_GOOGLE_CREDENTIALS_FILE
private_key_data    ewogICJ0eXB...Bpcy5jb20iCn0K
```

## For token
```
vault write gcp/static-account/vt-sa-01-token \
service_account_email="vt-sa-01@my-project.iam.gserviceaccount.com" \
secret_type="access_token"  \
token_scopes="https://www.googleapis.com/auth/cloud-platform" \
bindings=-<<EOF
resource "//cloudresourcemanager.googleapis.com/projects/my-project" {
roles = ["roles/viewer"]
}
EOF
```
Create the token:  
```
➜  ~ vault read gcp/static-account/tsa-01-token/token
Key                   Value
---                   -----
expires_at_seconds    1697309729
token                 ya29.c.b0Aaek ..... 3uJ1_bXt6qo23Z
token_ttl             59m58s
```
