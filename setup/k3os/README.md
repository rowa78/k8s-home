raspbian on sd-card
place SSH file on boot partition

boot

login pi/raspberry

create config.yaml



curl -sfL https://github.com/rancher/k3os/releases/download/v0.20.7-k3s1r0/k3os-rootfs-arm64.tar.gz | tar zxvf - --strip-components=1 -C /
cp myconfig.yaml /k3os/system/config.yaml
sync
reboot -f