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
wget https://releases.hashicorp.com/vault/0.10.2/vault_0.10.2_linux_amd64.zip
unzip vault_*
cp vault /usr/local/bin/
vault
vault -autocomplete-install
vault server -dev -dev-listen-address=0.0.0.0:8200 &
export VAULT_ADDR='http://127.0.0.1:8200'
```
* copy your unseal key from

```
Unseal Key: R/NKw0NHknSol/gP6cdPOPEOb+9aeR7PJ94V6f0WQGo=
Root Token: f70bf6c4-739a-c5b7-76ab-a4859cf3f6e6
```
* unseal vault

```python
vault operator unseal
Unseal Key (will be hidden): "here your unseal key"
```

* create your first key, read, delete

```jinja2
$ vault kv put secret/hello foo=world
Success! Data written to: secret/hello

$ vault kv put secret/hello foo=world excited=yes
Success! Data written to: secret/hello

$ vault kv get secret/hello
Key                 Value
---                 -----
refresh_interval    768h
excited             yes
value               world

$ vault kv get -format=json secret/hello | jq -r .data.data.foo
world

$ vault kv put secret/hello foo=new-world excited=yes
Key              Value
---              -----
created_time     2018-06-19T20:51:00.236007976Z
deletion_time    n/a
destroyed        false
version          3

$ vault kv get -version=1 secret/hello
$ vault kv get -version=3 secret/hello

#Softdelete
$ vault kv delete secret/hello
Success! Data deleted (if it existed) at: secret/hello

$ vault kv metadata get secret/hello
$ vault kv undelete -versions=3 secret/hello
$ vault kv get -version=3 secret/hello

#Harddelete
$ vault kv destroy -versions=3 secret/hello
$ vault kv metadata get secret/hello
$ vault kv undelete -versions=3 secret/hello
$ vault kv get -version=3 secret/hello
```

* create your first key with API

```Curl
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

* Not actual for Vault > 0.10.0
* install docker [here](https://gist.github.com/gangsta/ae8226f3eaa074e3232c992d85dce285)
* Secrets created using Vault UI are also versioned, CLI can retriev all versions.

```docker
docker run -d \
-p 8000:8000 \
-e VAULT_URL_DEFAULT=http://127.0.0.1:8200 \
--name vault-ui \
djenriquez/vault-ui
```
## Vault User

```vault
$ vault policy write admins -<<EOF
path "secret/*" {
  capabilities = ["read"]
}

path "secret/foo/*" {
  capabilities = ["create", "read", "list"]
}
EOF
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

$ vault policy list
$ vault policy read admins
```

* let's create some user

```hashi
$ vault auth enable userpass

$ vault write auth/userpass/users/gangsta \
    password=gangsta \
    policies=admins
```

```
$ vault login -no-store -method=userpass \
    username=gangsta \
    password=gangsta

$ curl \
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
# Ha Vault

```vault
export VAULT_ADDR='http://127.0.0.1:8200'
vault operator init

vault operator unseal
vault status

vault login
Token (will be hidden): $roottoken
```

# Dynamic Secrets

* PostgreSQL Prepare

```
su postgres
createdb foocafe
psql foocafe
CREATE ROLE vault_user WITH SUPERUSER LOGIN CREATEROLE;
CREATE TABLE afterwork (id SERIAL PRIMARY KEY, title TEXT, year INTEGER);
INSERT INTO afterwork (title, year) VALUES ('HashiCorp Skane', 2018);
```
```
echo 'local   all             all                                     md5
# IPv4 local connections:
host    all             all             192.168.33.60/32        trust
host    all             all             192.168.33.71/32        trust
host    all             all             192.168.33.72/32        trust
host    all             all             192.168.33.73/32        trust
# IPv6 local connections:
host    all             all             ::1/128                 ident' > /var/lib/pgsql/9.6/data/pg_hba.conf

echo "listen_addresses = '0.0.0.0'" >> /var/lib/pgsql/9.6/data/postgresql.conf

systemctl restart postgresql-9.6
```

* Vault

```
export VAULT_ADDR='http://127.0.0.1:8200'
vault secrets enable database

vault write database/config/postgresql_foocafe \
    plugin_name=postgresql-database-plugin \
    allowed_roles="admin,junior,member" \
    connection_url="postgresql://vault_user@192.168.33.62:5432/foocafe?sslmode=disable"

vault write database/roles/admin \
    db_name=postgresql_foocafe \
    creation_statements="CREATE ROLE \"{{name}}\"  WITH SUPERUSER LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
    revocation_sql="SELECT revoke_access('{{name}}'); DROP user \"{{name}}\";"  \
    default_ttl="60s" \
    max_ttl="300s"

vault write database/roles/junior  \
    db_name=postgresql_foocafe \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT USAGE, SELECT ON SEQUENCE afterwork_id_seq TO \"{{name}}\"; \
    GRANT ALL PRIVILEGES ON TABLE "afterwork" TO \"{{name}}\"; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    revocation_sql="SELECT revoke_access('{{name}}'); DROP user \"{{name}}\";"  \
    default_ttl="60s" \
    max_ttl="300s"

vault write database/roles/member \
    db_name=postgresql_foocafe \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    revocation_sql="SELECT revoke_access('{{name}}'); DROP user \"{{name}}\";"  \
    default_ttl="1h" \
    max_ttl="24h"

vault read database/creds/admin

vault read database/creds/junior

vault read database/creds/member
```

* psql foocafe --username <Insert Generated Username> --password 
* `\dg` show database users/roles

```
ADMIN

CREATE TABLE speaker (id SERIAL PRIMARY KEY, first_name TEXT, last_name TEXT);
INSERT INTO speaker (first_name, last_name) VALUES ('Frank', 'Stella');
INSERT INTO speaker (first_name, last_name) VALUES ('Richard', 'Serra');
INSERT INTO afterwork (title, year) VALUES ('HA Secret Server', 2020);

JUNIOR

INSERT INTO afterwork (title, year) VALUES ('Foocafe', 2017);
INSERT INTO afterwork (title, year) VALUES ('Foocafe', 2017);
INSERT INTO speaker (first_name, last_name) VALUES ('Karen', 'Har-yan');      #should give you error

MEMBER

SELECT * FROM afterwork;
SELECT * FROM speaker;

INSERT INTO afterwork (title, year) VALUES ('Demo', 2018);
INSERT INTO speaker (first_name, last_name) VALUES ('Karen', 'Har-yan');
```