# Setting cluster using K3D and Nginx controller 

## Prerequisites

1. Docker
1. kubectl
1. [Helm](https://helm.sh/docs/helm/helm_install/) 
1. [K3D](http://k3d.io/) ≥ 3.0.0
1. [ngrok](https://ngrok.com/) (if applicable) 

## Install

### Step 0: Create Temporary Domain

We will use `ngrok` to create a temporary domain to proxy to our port 443:

```sh
ngrok http https://localhost
```

Copy/paste the temporary domain, e.g. `92832de0.ngrok.io`

Don't close this terminal! Once you quit ngrok, the temporary domain is purged.

### Step 1: Install Kubernetes

We'll use k3d to create a quick Kubernetes installation. We are disable traefik as ingress controller. We are going to use NGINX ingress controll

```sh
k3d cluster create rancher \
  --api-port 6550 --servers 1 --agents 1 \
  --k3s-arg "--disable=traefik@server:0" \
  --port "80:80@loadbalancer" --port "443:443@loadbalancer" \
   --api-port 0.0.0.0:6550 \ 
  --wait

kubectl cluster-info

kubectl get node          # or kubectl get no
kubectl get storageclass  # or kubectl get sc
kubectl get namespace     # or kubectl get ns
kubectl get pod -A
kubectl get svc -A
```

### Step 2: Install Nginx-Ingress

Instead of Traefik, let's use nginx-ingress controller.

```sh

helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm install nginx-ingress nginx-stable/nginx-ingress

# This should respond with 404 from default backend!
curl http://localhost
```

If our nginx-ingress controller is working correctly, you should see a 404 not found message, because we haven't installed anything.

### Step 3: Install cert-manager

We'll need to install cert-manager for our Rancher installation.

```sh


# If you have installed the CRDs manually instead of with the `--set installCRDs=true` option added to your Helm install command, you should upgrade your CRD resources before upgrading the Helm chart:
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml

# Add the Jetstack Helm repository
helm2 repo add jetstack https://charts.jetstack.io

helm2 repo update

kubectl create namespace cert-manager

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.5.1


kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m


```
### Step 4: Install Rancher
Finally, let's install latest rancher. Don't forget to change `MYDOMAIN`!

```sh
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

# Change this to your domain from the ngrok tool
export MYDOMAIN=92832de0.ngrok.io #if you dont have public domain,  please use ngrox 
# if you do have a domain, please add your domain into $MYDOMAIN

kubectl create namespace cattle-system

helm install rancher rancher-latest/rancher \
	--namespace cattle-system \ 
    --version=2.6.1      \ 
    --set hostname=$MYDOMAIN" \
     --set replicas=3  \ 
      --set bootstrapPassword=bacanochevere  \ 
      --wait.           \
       --debug
       
# Wait for rancher to start
kubectl -n cattle-system get deploy rancher
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rancher   3         3         3            3           3m
```

### Step 5: Login to Rancher

That's it, open up a browser and start exploring Rancher.

```sh
open https://$MYDOMAIN/
```

## The End

Thanks for reading!

