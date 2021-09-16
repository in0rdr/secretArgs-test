# SecretArgs Test (KV Version)

This repo holds some test files. WIP.

Install Vault on Openshift in a test namespace and add test secret:
```
helm install vault hashicorp/vault \
 --set "global.openshift=true" \
 --set "server.dev.enabled=true" \
 --set "server.dev.devroottoken=root" \
 --set "injector.enabled=false" \
 --set "csi.enabled=false"
oc exec -it vault-0 -- sh
$ vault kv put secret/test password="123"
$ vault kv put secret/test password="456"
```

Get the issuer:
```
$ kubectl proxy &
$ curl --silent http://127.0.0.1:8001/.well-known/openid-configuration | jq -r .issuer
https://kubernetes.default.svc
# fuser 8001/tcp
# kill
```

Configure the Kubernetes auth with the correct `iss`:
```
vault write auth/kubernetes/config issuer="https://kubernetes.default.svc" \
 token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
 kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
 kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Write example policy for kv2:
```
vault policy write internal-app - <<EOF
path "secret/*" {
  capabilities = ["read"]
}
EOF
```

Attach policy to `test` auth role:
```
vault write auth/kubernetes/role/test \
 bound_service_account_names=webapp-sa \
 bound_service_account_namespaces=csi-vault-test-app \
 policies=internal-app \
 ttl=20m
```

Test:
```
oc exec -it webapp -- cat /mnt/secrets-store/test-secret-from-vault
123
```