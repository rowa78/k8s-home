# k8s-home

## prepare rasperry-pis

install ubuntu 20.04 lts on your pi's
login and change the default password 'ubuntu' to something you like
copy your ssh-key to the machines:

```
ssh-copy-id ubuntu@pi-0
ssh-copy-id ubuntu@pi-1
ssh-copy-id ubuntu@pi-2
ssh-copy-id ubuntu@pi-3
```

Login and change hostnames:

```
sudo hostnamectl set-hostname pi-0
sudo hostnamectl set-hostname pi-1
sudo hostnamectl set-hostname pi-2
sudo hostnamectl set-hostname pi-3
```

## setup k3s-cluster

## Setting up the cluster

Place the host-config in setup/inventory/k3s/hosts.ini site.yml

```
cd setup
ansible-playbook -i inventory/k3s/hosts.ini site.yml
```

copy kubeconfig from the cluster

```
mkdir -p $HOME/.kube
scp ubuntu@pi-0:/home/ubuntu/.kube/config $HOME/.kube/config
```

check the new cluster

```
kubectl cluster-info
```

```
Kubernetes master is running at https://pi-0:6443
CoreDNS is running at https://pi-0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://pi-0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

## Kube-vip for loadbalanced-controlpane

ssh into first node and create daemonset

kubectl apply -f https://kube-vip.io/manifests/rbac.yaml

ctr image pull docker.io/plndr/kube-vip:0.3.6

alias kube-vip="ctr run --rm --net-host docker.io/plndr/kube-vip:0.3.6 vip /kube-vip"

kube-vip manifest daemonset \
    --arp \
    --interface eth0 \
    --address 192.168.0.119 \
    --controlplane \
    --leaderElection \
    --taint \
    --inCluster | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml

check running kube-vip pods:

kubectl get pods --all-namespaces
NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
k3os-system   system-upgrade-controller-8bf4f84c4-f2ll4   1/1     Running   0          18m
kube-system   coredns-854c77959c-xpdjh                    1/1     Running   0          18m
kube-system   metrics-server-86cbb8457f-qngc6             1/1     Running   0          18m
kube-system   local-path-provisioner-5ff76fc89d-z265t     1/1     Running   0          18m
kube-system   kube-vip-ds-b27f8                           1/1     Running   0          28s
kube-system   kube-vip-ds-gfw9p                           1/1     Running   0          28s
kube-system   kube-vip-ds-5pd52                           1/1     Running   0          28s

put an dns-entry to this ip to your DNS-Server in your LAN


## Setting up direnv

Install direnv and setup some environment-variables in ./.envrc

add to .bashrc if using bash:
```
eval "$(direnv hook bash)"
```
add to .zshrc if using zsh:
```
eval "$(direnv hook zsh)"
```

Create a .envrc in project folder (and never add this file to your repo!)

```
export GITHUB_USER=
# token, that can create repositories (check all permissions under repo)
export GITHUB_TOKEN=

export BOOTSTRAP_GITHUB_REPOSITORY=https://github.com/rowa78/k8s-home

# 1Password-Token
export OP_TOKEN=
```

allow this file

```
direnv allow .envrc
```

Now your environment-variables are set.

## Setup flux

Create namespace

```
kubectl create namespace flux-system
```

### create initial config

We use the 1Password-Operator to deliver secrets to out cluster. It need's an secret ' with the 1password-credentials.json in it. create an integration in 1password and save 1password-credentials.json and the token

encode your 1password-credentials.json

```
cat 1password-credentials.json | base64
```

create a secret.yaml: 

```
apiVersion: v1
kind: Secret
metadata:
  name: op-credentials
  namespace: 1password
type: Opaque
stringData:
  1password-credentials.json: |-
    <base64 encoded 1password-credentials>
```

``` 
kubectl create namespace 1password
kubectl -n 1password apply -f secret.yaml
kubectl -n 1password create secret generic onepassword-token --from-literal=token=$OP_TOKEN

# delete the temporary secret.yaml
```

Install the 1password operator

```
helm repo add 1password https://1password.github.io/connect-helm-charts
helm -n 1password upgrade -i connect ../connect-helm-charts/charts/connect --set-file connect.credentials=/mnt/c/tmp/1password-credentials.json --values 1password-operator/values.yaml
kubectl apply -f 1password-operator/clusterrolebinding.yaml
```



### install flux to cluster

install flux

``` 
kubectl create namespace flux-system
flux bootstrap github --owner=rowa78 --repository=k8s-home --path=./clusters/production
```

### manual changed

i need to resolve dns-entries in my lan. So i changed the dns-server in configmap for CoreDNS:

```
kc -n kube-system edit configmap coredns
# change forward-line
# forward to my pi-hole
forward . 192.168.0.7
```

perhaps there is a better solution. Will look for that later.