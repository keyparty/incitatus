# Quickstart


```
kubectl create namespace incitatus
```

## Deploy Vault
```
kubectl --namespace incitatus create -R -f vault
```

## Deploy Vault Controller
```
kubectl -n incitatus \
  create secret generic vault-controller \
  --from-literal "vault_token=3e4a5ba1-kube-422b-d1db-844979cab098"
kubectl --namespace incitatus create -R -f vault-controller
```

## Setup CA
```
kubectl -n incitatus port-forward \
  $(kubectl -n incitatus \
    get pods -l app=vault \
    -o jsonpath='{.items[0].metadata.name}') \
  8201:8200 &

export VAULT_ADDR="http://127.0.0.1:8201"
export VAULT_TOKEN="3e4a5ba1-kube-422b-d1db-844979cab098"

vault status

vault mount pki

#Generate a root certificate:
vault mount-tune -max-lease-ttl=87600h pki
vault write pki/root/generate/internal common_name=cluster.local ttl=87600h
vault write pki/config/urls issuing_certificates="http://vault.incitatus.svc.cluster.local:8200/v1/pki/ca"
vault write pki/config/urls issuing_certificates="http://vault.incitatus.svc.cluster.local:8200/v1/pki/ca" \
  crl_distribution_points="http://vault.incitatus.svc.cluster.local:8200/v1/pki/crl"


#Create the client PKI role:
vault write pki/roles/client \
  allowed_domains="cluster.local" \
  allow_subdomains="true" \
  client_flag="true" \
  max_ttl="72h" \
  server_flag="false"

#Create the server PKI role:
vault write pki/roles/server \
  allow_any_name="true" \
  allowed_domains="cluster.local" \
  allow_subdomains="true" \
  client_flag="false" \
  max_ttl="72h" \
  enforce_hostnames="false" \
  server_flag="true"

# Create a Vault Policy
vault policy-write microservice ./policy.hcl
```

## Deploy the Server Service
```
kubectl --namespace incitatus create -R -f demo-apache
```

## Deploy the client
```
kubectl --namespace incitatus create -R -f demo-client
```
