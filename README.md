# VSO Vault Implementation better implementation than sidecar!

### Add vault helm repository
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### Deploy vault server
```
helm install vault hashicorp/vault -n vault --create-namespace --set server.dev.enabled=true
```
### Deploy VSO
```
helm install vault-secrets-operator hashicorp/vault-secrets-operator -n vault-secrets-operator-system \
    --create-namespace --values vso-values.yaml
```

### Configure kubernetes auth

1. Enable kubernetes auth backend
    ```
    kubectl exec -it vault-0 -n vault -- /bin/sh
    vault auth enable kubernetes
    ```
2. Configure kubernetes endpoint
    ```
    vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
    ```

### VSO with kvv2 secrets

#### Add secrets
First, exec into the vault pod.
1. Enable kvv2 secrets engine
    ```
    vault secrets enable kv-v2
    ```
2. Add secret for the webapp
    ```
    vault kv put kv-v2/webapp/dbcred db_host=192.168.994 db_name=development username=demo password=password
    ```
3. Add policy to read the secret
    ```
    vault policy write read_only_policy - << EOF
        path "kv-v2/data/webapp/dbcred" {
        capabilities = ["read"]
        }
    EOF
    ```

#### Update application role to access kvv2 secrets

1. Create a role to enable access to the secret
   *  This is important reminder!
   *  every namespace has default Service Account upon creation of namespace
   *  this is not default service account in default namespace (bound_service_account_names)
    ```
    vault write auth/kubernetes/role/webapp \
      bound_service_account_names=default \
      bound_service_account_namespaces=development \
      policies=read_only_policy \
      audience=vault \
      ttl=24h
    ```

### VSO with dynamic database secrets

#### Deploy and Configure postgres
1. Add bitnami helm repo
    ```
    helm repo add bitnami https://charts.bitnami.com/bitnami
    ```

2. Deploy postgres
    ```
    helm install postgres bitnami/postgresql \
        --set auth.database=webapp \
        --create-namespace \
        -n postgres
    ```
3. Retrieve password for postgres user
    ```
    kubectl get secret postgres-postgresql -n postgres \
        -o jsonpath={.data.postgres-password} | base64 -d
    ```

4. Login to `webapp` database and paste password
    ```
    psql -U postgres -d webapp
    ```
5. Create user for Vault server
    ```
    CREATE ROLE vault WITH SUPERUSER LOGIN ENCRYPTED PASSWORD 'vault';
    ```

#### Configure database secrets engine
1. Enable database secrets engine
    ```
    vault secrets enable database
    ```
2. Configure postgres connection
    ```
    vault write database/config/webapp-pg-db \
        plugin_name="postgresql-database-plugin" \
        allowed_roles="webapp-role" \
        connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.postgres:5432/webapp" \
        username="vault" \
        password="vault"
    ```
3. Create database role
    ```
    vault write database/roles/webapp-role \
        db_name="webapp-pg-db" \
        creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
            GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
        default_ttl="1h" \
        max_ttl="24h"
    ```
4. Test database credential
    ```
    vault read database/creds/webapp-role
    ```
5. Create policy to read database creds for the "webapp"
    ```
    vault policy write webapp-role - <<EOF
    path "database/creds/webapp-role" {
    capabilities = ["read"]
    }
    EOF
    ```
6. Assign policy to the kubernetes role
    ```
    vault write auth/kubernetes/role/webapp \
        bound_service_account_names=default \
        bound_service_account_namespaces=webapp \
        policies=default,dbcred,webapp-role  \
        audience=vault \
        ttl=24h
    ```

### Deploy manifests
1. Create new namespace
    ```
    kubectl create ns development
    ```
2. Apply VaultAuth
    ```
    kubectl apply -f vault-auth.yaml
    ```
3. Apply VaultStaticSecret (only for kvv2 secrets)
    ```
    kubectl apply -f kv-secret.yaml
    ```
4. Apply VaultDynamicSecret (only for dynamic database secrets)
    ```
    kubectl apply -f database-creds.yaml
    ```
5. Deploy application
    ```
    kubectl apply -f deployment.yaml
    ```
