# Up and running Kubernetes

💠 Step-by-step guide of a custom and universal setup for plain VPSs or bare-metal servers using free and open tools.

- [Weave Net](https://www.weave.works/oss/net/)
- [MetalLB](https://metallb.universe.tf/)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Let's Encrypt TLS certificate](https://letsencrypt.org/)
- Primarily single-node, but more nodes can be added

Feel free to open Issues and Pull Requests.

## OS

Ubuntu 18+<br>
_All the steps below was made on a DigitalOcean's \$5 droplet._

**Here we go**

## Update packages

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

## Install Docker

```bash
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs)  stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

✔️ Checking

- `sudo docker run hello-world`

Source: https://docs.docker.com/install/linux/docker-ce/ubuntu/

## Install Kubernetes (kubeadm, kubelet and kubectl)

```bash
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

Source: https://kubernetes.io/docs/setup/independent/install-kubeadm/

## Initializing

```bash
kubeadm init
```

Got `[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2`? Then:

```bash
kubeadm init --ignore-preflight-errors=NumCPU
```

## Configuring

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Source: https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

## Installing a pod network add-on

Weave Net was the chosen one based on: https://chrislovecnm.com/kubernetes/cni/choosing-a-cni-provider/

```bash
sysctl net.bridge.bridge-nf-call-iptables=1
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

✔️ Checking

```bash
kubectl get pods --all-namespaces
```

Source: https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network

## Control plane node isolation

⚠️ **This is important because as a single-machine Kubernetes cluster, pods will be running on the master**

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Ingress

### MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/metallb.yaml
```

Apply the following config:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - <your-server-ip-addr>-<your-server-ip-addr>
```

(yes, your IP address twice with a hyphen between)

Source: https://metallb.universe.tf/installation/

### NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```

#### Change from NodePort to LoadBalancer (our previous installed MetalLB)

```bash
kubectl -n ingress-nginx edit svc ingress-nginx
```

Locate `type: NodePort` and change to `type: LoadBalancer`

✔️ Checking

```bash
kubectl -n ingress-nginx get svc
```

You should see your server IP address under the `EXTERNAL-IP` column of the `ingress-nginx` service. You can also make a simple request to check:

```bash
curl http://<your-server-ip>
```

You should see the Nginx default 404 page.

Source: https://kubernetes.github.io/ingress-nginx/deploy/

### Ingress resource

#### Create a test app

```bash
kubectl run web --image=gcr.io/google-samples/hello-app:1.0 --port=8080
kubectl expose deployment web --target-port=8080 --type=NodePort
```

#### Resource file

```yaml
apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: example-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
     rules:
     - host: yourdomain.tld
       http:
         paths:
         - path: /*
           backend:
             serviceName: web
             servicePort: 8080
```

✔️ Checking

```bash
curl http://<yourdomain.tld>/hello

Hello, world!
Version: 1.0.0
Hostname: web-ddb799d85-hrt6h
```

## TLS (Let's Encrypt + Certbot)

### Install Cerbot

```bash
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot
```

### Genereate the certificate

```bash
sudo certbot certonly
```

### Create the TLS secret resource

```bash
kubectl create secret tls tls-secret \
  --cert=/etc/letsencrypt/live/yourdomain.tld/fullchain.pem \
  --key=/etc/letsencrypt/live/yourdomain.tld/privkey.pem
```

### Update the Ingress resource

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts:
        - yourdomain.tld
      secretName: tls-secret
  rules:
    - host: yourdomain.tld
      http:
        paths:
          - path: /hello
            backend:
              serviceName: web
              servicePort: 8080
```

✔️ Checking

```bash
curl https://yourdomain.tld/hello

Hello, world!
Version: 1.0.0
Hostname: web-ddb799d85-hrt6h
```
