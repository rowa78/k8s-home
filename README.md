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

### install flux to cluster

install flux

``` 
kc create namespace flux-system
flux bootstrap github --owner=rowa78 --repository=k8s-home --path=./clusters/production
```
