### 1. Install and configure a K3s cluster
- 1 master node
- 2 agent nodes


1 - paket güncelleme > sudo apt update -y && sudo apt upgrade -y

2 - swap kapatma > sudo swapoff -a  ----- sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 

3 - hostname ayarları  

# Master node için:
sudo hostnamectl set-hostname k3s-master

# Worker node 1 için:
sudo hostnamectl set-hostname k3s-worker1

# Worker node 2 için:
sudo hostnamectl set-hostname k3s-worker2


4- host dosyası düzenle (her node’da aynı olmalı)
sudo tee -a /etc/hosts > /dev/null <<EOF
10.34.19.64 k3s-master
10.34.19.65 k3s-worker1
10.34.19.66 k3s-worker2
EOF

5 -  master node kurulumu 

curl -sfL https://get.k3s.io | sh -

sudo sysetmctl status k3s



6- master node token bilgileri 

sudo cat /var/lib/rancher/k3s/server/node-token


7- worker node kurulumu

curl -sfL https://get.k3s.io | K3S_URL=https://10.34.19.64:6443 K3S_TOKEN=<TOKEN> sh -


8 - cluster durumu kontrol (master node)

kubectl get nodes
kubectl get pods -A


