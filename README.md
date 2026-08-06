## Multipass — Local VM Setup for k3s

Multipass is used to create lightweight Ubuntu VMs on your Mac to run a local k3s cluster that mirrors a real Kubernetes deployment.

### Install

```bash
brew install --cask multipass
```

Open the Multipass app from Applications and wait for the daemon to start before running any commands.

### Create nodes

```bash
multipass launch --name k3s-server --cpus 3 --memory 4G --disk 10G
multipass launch --name k3s-agent1 --cpus 3 --memory 4G --disk 10G
```

### Install k3s on server

```bash
multipass exec k3s-server -- bash -c "curl -sfL https://get.k3s.io | sh -"
```

### Join agent to the cluster

```bash
TOKEN=$(multipass exec k3s-server -- sudo cat /var/lib/rancher/k3s/server/node-token)
SERVER_IP=$(multipass info k3s-server | grep IPv4 | awk '{print $2}')

multipass exec k3s-agent1 -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://${SERVER_IP}:6443 K3S_TOKEN=${TOKEN} sh -"
```

### Configure kubectl to use the k3s cluster

```bash
multipass exec k3s-server -- sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/k3s-config
SERVER_IP=$(multipass info k3s-server | grep IPv4 | awk '{print $2}')
sed -i '' "s/127.0.0.1/${SERVER_IP}/" ~/.kube/k3s-config
export KUBECONFIG=~/.kube/k3s-config
```

### Verify both nodes are ready

```bash
kubectl get nodes
```

### Useful multipass commands

```bash
multipass list                        # list all VMs
multipass info k3s-server             # show VM details and IP
multipass shell k3s-server            # SSH into a VM
multipass stop k3s-server k3s-agent1  # stop VMs
multipass start k3s-server k3s-agent1 # start VMs
multipass delete k3s-server           # delete a VM
multipass purge                       # permanently remove deleted VMs
```

---

## Nginx Ingress Controller

The nginx ingress controller is the cluster-wide traffic entry point. It handles all incoming HTTP/HTTPS requests and routes them to the correct service based on ingress rules. It runs in its own namespace and serves all namespaces in the cluster.

### Install

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Verify controller is running

```bash
kubectl get pods -n ingress-nginx
kubectl get svc ingress-nginx-controller -n ingress-nginx --watch
```

On k3s the `EXTERNAL-IP` will stay `<pending>` — install MetalLB (see section below) to assign a real IP.

### Check ingress rules across namespaces

```bash
kubectl get ingress -n uber-ns-app
```

---

helm install <APPLICATION_NAME> <HELM_CHART_PATH>
helm list
helm uninstall <APPLICATION_NAME>
kubectl get deployment
kubectl get svc
kubectl get pods
kubectl describe pod <POD_ID>
kubectl logs <POD_ID>
kubectl describe configmaps <CONFIGMAP_NAME>
kubectl get configmaps <CONFIGMAP_NAME> -o yaml
docker images
docker container ls -a
docker logs <CONTAINER_NAME>

<!--
great now that the values.yml is shared across the all deployment can i just run helm install in the k8s so it will dek -->

nginx controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
 --namespace ingress-nginx \
 --create-namespace
Then watch until it gets an external IP:

kubectl get svc ingress-nginx-controller -n ingress-nginx --watch

---

## MetalLB

k3s has no cloud load balancer so `LoadBalancer` services stay `<pending>` indefinitely.
MetalLB assigns real IPs from your local network to those services, making the ingress reachable from your Mac.

### Install

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### Get VM subnet

```bash
multipass info k3s-server | grep IPv4
```

### Configure IP pool (match your VM subnet)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.252.200-192.168.252.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
EOF
```

### Verify

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx --watch
```

---

## Kubernetes Dashboard

### Install

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### Create admin user and generate token

```bash
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard

kubectl create clusterrolebinding dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:dashboard-admin

kubectl create token dashboard-admin -n kubernetes-dashboard
```

### Start proxy and open dashboard

```bash
kubectl proxy
```

Open in browser:

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Paste the token from above to log in.

---

## ghcr-secret (private image pull)

The deployments pull images from a **private** GitHub Container Registry (`ghcr.io`). Each deployment declares:

```yaml
spec:
  imagePullSecrets:
    - name: ghcr-secret
```

so the kubelet needs a `docker-registry` Secret named `ghcr-secret` in `uber-ns-app` to authenticate the pull. Without it the pods fail with `ImagePullBackOff` / `ErrImagePull`.

### 1. Log in to ghcr.io with docker

Authenticate to the registry using your GitHub username and a Personal Access Token (PAT with `read:packages` scope) as the password:

```bash
docker login ghcr.io \
  --username <your github username> \
  --password <personal access token>
```

### 2. Create the pull secret

`kubectl` builds the correct `kubernetes.io/dockerconfigjson` format for you:

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=zannujulius \
  --docker-password=<personal access token> \
  --namespace=uber-ns-app
```

### Verify

```bash
kubectl get secret ghcr-secret -n uber-ns-app
```

> This is a separate Secret from `uber-secret`. It only handles image pulls — it cannot be replaced by the `GIT_USERNAME`/`GIT_PAT` keys in an `Opaque` secret, because `imagePullSecrets` requires the `dockerconfigjson` type. See the Sealed Secrets section for how to seal `ghcr-secret` so it can be committed.

---

## Sealed Secrets

A plain Kubernetes `Secret` is only base64-encoded, not encrypted, so it is never safe to commit to git. The Sealed Secrets controller runs in the cluster and holds a private key. You encrypt a `Secret` with the matching public key using `kubeseal`, producing a `SealedSecret` that is safe to commit. Only the controller in this cluster can decrypt it back into a real `Secret`.

```
plain Secret  --kubeseal (public key)-->  SealedSecret  -->  git (safe)
                                                 |
                                    controller decrypts (private key)
                                                 |
                                                 v
                                     real Secret in uber-ns-app
```

The app pods never see the SealedSecret. The controller decrypts it into a normal `Secret` named `uber-secret` in `uber-ns-app`, and the deployments read it via `secretKeyRef`.

### 1. Install the controller (Helm)

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace sealed-secrets \
  --create-namespace
```

Verify the controller and its service are running:

```bash
kubectl get pods -n sealed-secrets
kubectl get svc -n sealed-secrets   # expect sealed-secrets + sealed-secrets-metrics
```

### 2. Install the kubeseal CLI (local machine)

```bash
brew install kubeseal
kubeseal --version
```

### 3. Fetch the public cert

The controller name is `sealed-secrets` and it lives in the `sealed-secrets` namespace. Save the public cert so you can seal offline (this cert is safe to commit — it only encrypts):

```bash
kubeseal --fetch-cert \
  --controller-name sealed-secrets \
  --controller-namespace sealed-secrets \
  > sealed-secrets-cert.pem
```

### 4. Write the plain Secret (do NOT commit)

`k8s/secrets.yml` — `name` and `namespace` must be literal (kubeseal cannot seal Helm `{{ }}` templating). Use `stringData` so values are written in plaintext and encoded automatically. Keys must cover every `secretKeyRef` used across the charts (`DB_PASSWORD`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `GOOGLE_MAPS_API_KEY`).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: uber-secret
  namespace: uber-ns-app
type: Opaque
stringData:
  JWT_SECRET: "..."
  JWT_REFRESH_SECRET: "..."
  DB_PASSWORD: "..."
  GOOGLE_MAPS_API_KEY: "..."
```

### 5. Seal it

```bash
kubeseal --format yaml \
  --cert sealed-secrets-cert.pem \
  < k8s/secrets.yml \
  > k8s/sealed-secret.yml
```

`k8s/sealed-secret.yml` is encrypted and safe to commit. Delete the plaintext:

```bash
rm k8s/secrets.yml
```

### 6. Apply and verify

```bash
kubectl apply -f k8s/sealed-secret.yml

kubectl get sealedsecret -n uber-ns-app
kubectl get secret uber-secret -n uber-ns-app   # controller creates this
```

When the real `uber-secret` appears, the deployments can start.

### Recreating on a fresh cluster

The private key is unique per controller install, so a `SealedSecret` sealed against an old cluster **cannot** be decrypted by a new one. To recreate:

1. Reinstall the controller (step 1).
2. Re-fetch the cert (step 3) — it is a new key.
3. Re-seal `k8s/secrets.yml` against the new cert (step 5).
4. Apply the new `k8s/sealed-secret.yml` (step 6).

(To preserve secrets across rebuilds instead, back up the controller's sealing key: `kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealing-key-backup.yaml` — keep this file secret, out of git.)

### Notes

- `--controller-namespace` is the controller's namespace (`sealed-secrets`), NOT where the app runs (`uber-ns-app`).
- A `SealedSecret` is locked to one name + namespace. Seal `uber-secret` for `uber-ns-app` or decryption fails.
- The image pull credential (`ghcr-secret`, a `docker-registry` type) is separate — seal it too if you want it in git:

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=zannujulius \
  --docker-password=ghp_xxxx \
  --namespace=uber-ns-app \
  --dry-run=client -o yaml \
| kubeseal --format yaml --cert sealed-secrets-cert.pem \
  > k8s/sealed-ghcr-secret.yml
```

---

## Naming convenstion for internal service name

- [service-name].[charts i.e postgres].[namespace].[application type. svc, deployment].[cluster].[local]
  e,g [uber-db-postgresql].[uber-ns-app].[svc].[cluster].[local]
