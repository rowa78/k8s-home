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

edit the host-config in setup/hosts.yaml

```
cd setup
ansible-playbook setup-k3s.yaml
```

get kubeconfig from the cluster. It is located in file /etc/rancher/k3s/k3s.yaml. Place it in your homefolder: $HOME/.kube/config and edit the url to the loadbalanced ip provided by kube-vip. (192.168.0.119 in my case):

```
server: https://192.168.0.119:6443
```


check the new cluster

```
kubectl cluster-info
Kubernetes control plane is running at https://192.168.0.119:6443
CoreDNS is running at https://192.168.0.119:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://192.168.0.119:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

## Import Cluster to rancher





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

``` 
kubectl create namespace 1password
kubectl -n 1password create secret generic onepassword-token --from-literal=token=$OP_TOKEN
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

