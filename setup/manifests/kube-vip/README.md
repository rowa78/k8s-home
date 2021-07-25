generated with:

ssh into first node and create daemonset

ctr image pull docker.io/plndr/kube-vip:v0.3.6

alias kube-vip="ctr run --rm --net-host docker.io/plndr/kube-vip:v0.3.6 vip /kube-vip"

kube-vip manifest daemonset \
    --arp \
    --interface eth0 \
    --address 192.168.0.119 \
    --controlplane \
    --leaderElection \
    --taint \
    --inCluster