# How To Kubernetes w/ Vault-Agent

The reason we want Vault-Agent is the client token will be written to a sink file and we will take the value at that sink file in the api pod to authenticate on the client side.

1. In the project directory
### `kubectl create secret docker-registry gitlab-access-token --docker-server=registry.gitlab.com --docker-username=nmcci --docker-password TxMX2psxzXMh_inkZ-9M`
### `helm repo add hashicorp https://helm.releases.hashicorp.com`
### `helm repo update`
### `hhelm install vault hashicorp/vault --set "server.dev.enabled=true"`
### `helm upgrade --install api vaulthelm --values vaulthelm/api.yaml`
### `helm upgrade --install client vaulthelm --values vaulthelm/client.yaml`

2. Execute a bash terminal in the vault and enable kubernetes
### `kubectl exec -it vault-0 -- /bin/sh`
### `vault auth enable kubernetes`

3. Configure the Kubernetes authentication method to use the service account token, the location of the Kubernetes host, and its certificate.
### ` vault write auth/kubernetes/config \
    issuer="https://kubernetes.default.svc.cluster.local" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
 
4. Add a Kubernetes role to access with the api service account
### `vault write auth/kubernetes/role/internal-app \
    bound_service_account_names=api \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=10h`
### `exit`

5. Now you can tunnel in to the client service to access the front-end
### `minikube service client --url`
