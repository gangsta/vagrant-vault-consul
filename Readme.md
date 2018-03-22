# Vault with Consul Backend

## Vault Demo

* Don't forget to copy your vault tokens&keys

```
Unseal Key 1: 0L1Hmqn5NOqUjL3MHbS38HzQQ3XccPvgltuNytA1WUit                                               
Unseal Key 2: C+32LGBkVWp40Fo14/k4VyDnAS3Fag0ziQC8QO2NDDWF                                               
Unseal Key 3: WyeqLzCtf7Y4kgknTFJyS8WHfJVmerTWeeaJDQBHQ9Ef                                               
Unseal Key 4: Awfo1ajrMZPQYzT4aTgZSHCwVmDmMeMWzdJz06sOnbOM                                               
Unseal Key 5: lljRDlBdwuaH3kXIOmBX0wxLfOJPknB/NqvUTX7Jc8ue                                               

Initial Root Token: 039c9534-089d-e353-0062-1ee4c65ea3df 
```


```vim
yum install wget unzip epel-release -y
yum install jq -y
wget https://releases.hashicorp.com/vault/0.9.5/vault_0.9.5_linux_amd64.zip
unzip vault_*
cp vault /usr/local/bin/
vault
vault -autocomplete-install
vault server -dev -dev-listen-address=0.0.0.0:8200 &
export VAULT_ADDR='http://127.0.0.1:8200'
```
* copy your unseal key from

```
Unseal Key: rLZx8NjL149ItDil35PRTShgWpzIOVHyzS5K8jlYJAg=
Root Token: 6bb443f0-3d65-dbfc-f31d-3f8586de62d6
```
* unseal vault

```python
vault operator unseal
Unseal Key (will be hidden): "here your unseal key"
```

* create your first key, read, delete

```jinja2
$ vault write secret/hello value=world
Success! Data written to: secret/hello

$ vault write secret/hello value=world excited=yes
Success! Data written to: secret/hello

$ vault read secret/hello
Key                 Value
---                 -----
refresh_interval    768h
excited             yes
value               world

$ vault read -format=json secret/hello | jq -r .data.value
world

$ vault delete secret/hello
Success! Data deleted (if it existed) at: secret/hello
```

* create your first key with API

```API
export VAULT_TOKEN=6fa4128e-8bd2-fd02-0ea8-a5e020d9b766  #should be root token

curl \
    --request POST \
    --data '{"key": "Nz1QAnTdjrlSccsPho5N7SfZr6IF0XQdYLuXzXzi/kE="}' \
    http://127.0.0.1:8200/v1/sys/unseal

curl \
    -H "X-Vault-Token: f3b09679-3001-009d-2b80-9c306ab81aa6" \
    -X GET \
    http://127.0.0.1:8200/v1/secret/foo

curl \
    -H "X-Vault-Token: f3b09679-3001-009d-2b80-9c306ab81aa6" \
    -X LIST \
    http://127.0.0.1:8200/v1/secret/

curl \
    -H "X-Vault-Token: ab0ca201-4344-7e0e-b7ab-3a3e1a82ac6d" \
    -H "Content-Type: application/json" \
    -X POST \
    -d '{"value":"bar"}' \
    http://127.0.0.1:8200/v1/secret/baz
```
## Vault Ui

* install docker [here](https://gist.github.com/gangsta/ae8226f3eaa074e3232c992d85dce285)

```docker
docker run -d \
-p 8000:8000 \
-e VAULT_URL_DEFAULT=http://127.0.0.1:8200 \
--name vault-ui \
djenriquez/vault-ui
```
* policy for user

```vault
echo 'path "secret/*" {
  capabilities = ["create"]
}

path "secret/foo" {
  capabilities = ["read"]
}' > my-policy.hcl

vault policy fmt my-policy.hcl
```

```vault
$ vault policy write admins -<<EOF
path "secret/*" {
  capabilities = ["create", "list" , "read", ]
}

path "secret/foo" {
  capabilities = ["read"]
}

path "auth/token/lookup-self" {
    capabilities = [ "read" ]
}

path "sys/capabilities-self" {
    capabilities = [ "update" ]
}

path "sys/mounts" {
    capabilities = [ "read" ]
}

path "sys/auth" {
    capabilities = [ "read" ]
}

path "auth/token/accessors" {
    capabilities = [ "sudo", "list" ]
}

path "auth/token/lookup-accessor/*" {
    capabilities = [ "read" ]
}
EOF

vault policy list
```

* let's create some user

```hashi
vault auth enable userpass

vault write auth/userpass/users/gangsta \
    password=gangsta \
    policies=admins
```

```
vault login -no-store -method=userpass \
    username=gangsta \
    password=gangsta

curl \
    --request POST \
    --data '{"password": "gangsta"}' \
    http://http://192.168.33.60:8200/v1/auth/userpass/login/gangsta
```


* update user with `update` and `delete` policy

```hashi

$ vault policy write admins -<<EOF
path "secret/*" {
  capabilities = ["create", "list" , "read", "update", "delete"]
}

path "secret/foo" {
  capabilities = ["read"]
}

path "auth/token/lookup-self" {
    capabilities = [ "read" ]
}

path "sys/capabilities-self" {
    capabilities = [ "update" ]
}

path "sys/mounts" {
    capabilities = [ "read" ]
}

path "sys/auth" {
    capabilities = [ "read" ]
}

path "auth/token/accessors" {
    capabilities = [ "sudo", "list" ]
}

path "auth/token/lookup-accessor/*" {
    capabilities = [ "read" ]
}
EOF
```



## ACL Consul

```
curl \
    --request PUT \
    --header "X-Consul-Token: 398073a8-5091-4d9c-871a-bbbeb030d1f6" \
    --data \
'{
  "Name": "Agent Token",
  "Type": "client",
  "Rules": "node \"\" { policy = \"write\" } service \"\" { policy = \"read\" }"
}' http://127.0.0.1:8500/v1/acl/create



curl \
    --request PUT \
    --header "X-Consul-Token: 398073a8-5091-4d9c-871a-bbbeb030d1f6" \
    --data \
'{
  "ID": "anonymous",
  "Type": "client",
  "Rules": "node \"\" { policy = \"read\" }"
}' http://127.0.0.1:8500/v1/acl/update



curl \
    --request PUT \
    --header "X-Consul-Token: 398073a8-5091-4d9c-871a-bbbeb030d1f6" \
    --data \
'{
  "ID": "anonymous",
  "Type": "client",
  "Rules": "node \"\" { policy = \"read\" } service \"consul\" { policy = \"read\" }"
}' http://127.0.0.1:8500/v1/acl/update

{"ID":"anonymous"}



curl \
    --request PUT \
    --data \
'{
  "Name": "gangsta",
  "Type": "client",
  "Rules": "key \"\" { policy = \"read\" } key \"foo/\" { policy = \"write\" } key \"foo/private/\" { policy = \"deny\" } operator = \"read\""
}' http://127.0.0.1:8500/v1/acl/create?token=398073a8-5091-4d9c-871a-bbbeb030d1f6



curl \
    --header "X-Consul-Token: 398073a8-5091-4d9c-871a-bbbeb030d1f6" \
    --request PUT \
    --data '{"Name": "sample", "Type": "management"}' \
    http://127.0.0.1:8500/v1/acl/create


export VAULT_ADDR='http://127.0.0.1:8200'


vault write consul/config/access \
    address=127.0.0.1:8500 \
    token=d12526c9-3f4e-28e4-d478-ef4543868ee5


vault write consul/roles/my-role policy=$(base64 <<< 'key "" { policy = "read" }')

vault write consul/roles/readonly policy=$(base64 <<< 'key "" { policy = "read" }')


vault read consul/creds/readonly
```