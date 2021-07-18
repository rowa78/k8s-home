# Build a Kubernetes cluster using k3s via Ansible

Author: <https://github.com/itwars>

## K3s Ansible Playbook

Build a Kubernetes cluster using Ansible with k3s. The goal is easily install a Kubernetes cluster on machines running:

- [X] Debian
- [X] Ubuntu
- [X] CentOS

on processor architecture:

- [X] x64
- [X] arm64
- [X] armhf

## System requirements

Deployment environment must have Ansible 2.4.0+
Master and nodes must have passwordless SSH access

## Usage

First create a new directory based on the `sample` directory within the `inventory` directory:

```bash
cp -R inventory/sample inventory/my-cluster
```

Second, edit `inventory/my-cluster/hosts.ini` to match the system information gathered above. For example:

```bash
[master]
192.16.35.12

[node]
192.16.35.[10:11]

[k3s_cluster:children]
master
node
```

If needed, you can also edit `inventory/my-cluster/group_vars/all.yml` to match your environment.

Start provisioning of the cluster using the following command:

```bash
ansible-playbook site.yml -i inventory/my-cluster/hosts.ini
```

## Kubeconfig

To get access to your **Kubernetes** cluster just

```bash
scp debian@master_ip:~/.kube/config ~/.kube/config
```


## Rancher Install on a vm 

I like Rancher. I work with rancher at work, so i want to experiment with it in this project. The idea is to create a vm in my diskstation, setup a k3s cluster and install rancher via helm. After that i import the k3s-workloadcluster to this rancher.

- install ubuntu-server in a vm
- place your ssh-key to this server, so ansible can login without password
- add this host to the hosts.ini like that:

```
[rancher]
rancher
``` 
- edit the group_vars and host_vars  for this machine. (k3s-Options like a MARIADBSTORE)
- apply the playbook rancher.yaml
```
ansible-playbook -i inventory/k3s/hosts.ini rancher.yml
```

- add rancher helm repo

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

- add certmanager

```
# Install the CustomResourceDefinition resources separately
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.crds.yaml

# **Important:**
# If you are running Kubernetes v1.15 or below, you
# will need to add the `--validate=false` flag to your
# kubectl apply command, or else you will receive a
# validation error relating to the
# x-kubernetes-preserve-unknown-fields field in
# cert-manager’s CustomResourceDefinition resources.
# This is a benign error and occurs due to the way kubectl
# performs resource validation.

# Create the namespace for cert-manager
kubectl create namespace cert-manager
kubectl create namespace cattle-system

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.4.0  

kubectl get pods --namespace cert-manager  
```

- create a certificate for rancher. i use cloudflare for this. first add a secret with the cloudflare-api-key and a cluster-issuer

```
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-token-secret
type: Opaque
stringData:
  api-token: ${YOUR_TOKEN}
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "${SECRET_CLOUDFLARE_EMAIL}"
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - dns01:
        cloudflare:
          email: "${SECRET_CLOUDFLARE_EMAIL}"
          apiTokenSecretRef:
            name: cloudflare-token-secret
            key: api-token

```

- create a certificate for your rancher-domain in namespace cattle-system
``` 
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-rancher-ingress
  namespace: cattle-system
spec:
  secretName: tls-rancher-ingress
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "${DOMAIN}"
  dnsNames:
  - "${DOMAIN}"
```

- wait for a valid certificate

```
kubectl -n cattle-system get certificate                                                                                                                                                                                                                                                                                                      Galactica: Thu Jul 15 21:58:51 2021

NAME   READY   SECRET   AGE
tls    True    tls      10m
```

- now install rancher via helm

```
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=${DOMAIN} \
  --set replicas=1 \
  --set ingress.tls.source=secret \
  --version=2.5.9
```

- now you can login to your rancher an import the k3s cluster
