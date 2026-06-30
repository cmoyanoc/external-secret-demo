# Instalar Vault via Helm Chart

## Agregar repositorio

```bash
➜ helm repo add openshift-charts https://charts.openshift.io

```

## Listar releases disponibles

```bash
➜ helm search repo openshift-charts/vault

NAME                    CHARTVERSION  APP VERSION  DESCRIPTION
openshift-charts/vault  0.33.0        2.0.2        Official HashiCorp Vault Chart

```

## Instalar version 0.33.0

```bash
➜ helm install vault openshift-charts/vault --version 0.33.0 \
    --create-namespace \
    --namespace vault-cmc \
    --set "global.openshift=true" \
    --set "server.dev.enabled=true"
```

## Comprobar

```bash
➜ oc project vault-cmc
Already on project "vault-cmc" on server "https://api.cmc-ocp.vmware.tamlab.rdu2.redhat.com:6443".
```

```bash
➜ helm list

NAME  NAMESPACE REVISION UPDATED                                 STATUS   CHART        APP VERSION
vault vault-cmc 1        2026-06-30 12:08:43.172962563 -0400 -04 deployed vault-0.33.0 2.0.2
```

```bash
➜ oc get all

NAME                                        READY   STATUS    RESTARTS   AGE
pod/vault-0                                 1/1     Running   0          62s
pod/vault-agent-injector-559b879967-wfln6   1/1     Running   0          62s

NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/vault                      ClusterIP   172.30.106.74   <none>        8200/TCP,8201/TCP   64s
service/vault-agent-injector-svc   ClusterIP   172.30.84.116   <none>        443/TCP             64s
service/vault-internal             ClusterIP   None            <none>        8200/TCP,8201/TCP   64s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vault-agent-injector   1/1     1            1           63s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/vault-agent-injector-559b879967   1         1         1       63s

NAME                     READY   AGE
statefulset.apps/vault   1/1     63s
```

## Crear Route para acceder a la web

```bash
➜ oc expose svc vault
route.route.openshift.io/vault exposed
```

```bash
➜ oc get route
NAME    HOST/PORT                                                    PATH   SERVICES   PORT   TERMINATION   WILDCARD
vault   vault-vault-cmc.apps.cmc-ocp.vmware.tamlab.rdu2.redhat.com          vault      http                 None
```

## Configuración inicial Vault

```bash
➜ oc rsh pod/vault-0
```

```bash
sh-5.2$ vault login
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                root
token_accessor       GfX7Ihke66QYfgvDK2nPOhVv
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
sh-5.2$
```

### Activamos autentificacion Kubernetes 

```bash
sh-5.2$ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/

sh-5.2$ echo $KUBERNETES_PORT_443_TCP_ADDR
172.30.0.1

sh-5.2$ vault write auth/kubernetes/config  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
Success! Data written to: auth/kubernetes/config
```

### Creamos Secretos username y password en myapp/database

```bash
sh-5.2$ vault kv put secret/myapp/database username="mi_usuario_db" password="mi_password_super_seguro"
======= Secret Path =======
secret/data/myapp/database

======= Metadata =======
Key                Value
---                -----
created_time       2026-06-30T17:10:16.224931516Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

### Creamos politica de acceso para nuestros secrets

```bash
sh-5.2$ vault policy write eso-policy - <<EOF
> path "secret/data/myapp/database" {
>   capabilities = ["read", "list"]
> }
> EOF
Success! Uploaded policy: eso-policy
```

### Creamos role para el serviceaccount 
```bash
sh-5.2$ vault write auth/kubernetes/role/eso-role \
>     bound_service_account_names=eso-vault-sa \
>     bound_service_account_namespaces=eso-demo \
>     policies=eso-policy \
>     ttl=1h
```

